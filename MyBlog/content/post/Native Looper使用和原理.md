---
title: "Native Looper使用和原理"
date: 2023-06-24
thumbnailImagePosition: left
thumbnailImage: handler/Looper/Looper_thumb.jpg
coverImage: handler/Looper/Looper_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- Native Looper
- 2023
- June
tags:
- Android
- 源码
- Handler
- epoll
- Thread
- pthread_once
showSocial: false
---

Android中大量的用到了消息机制，而最终消息机制都离不开native Looper。

<!--more-->
# 0简介

实际上，除了本身Handler的消息机制外，很多进程也会用到这些消息机制，比如SurfaceFlinger进程。所以说Native的Looper机制就显得更加重要。下面以例子分析Looper原理。

# 1举例说明

选择源码编译或者NDK编译皆可，这里笔者选择源码编译，编译路径为/external/

```c++
//external/native_looper_test/main.h
#ifndef NATIVE_LOOPER_TEST_MAIN_H
#define NATIVE_LOOPER_TEST_MAIN_H
#include <utils/Looper.h>
#include <utils/Timers.h>
#include <utils/Log.h>
#include <unistd.h>
#include <time.h>
#include <utils/threads.h>

//使用Android智能指针或者日志需要使用这个命名空间
using namespace android;
using namespace std;

class MyMessageHandler : public MessageHandler {
public:
    Vector<Message> messages;

    virtual void handleMessage(const Message& message);
    virtual ~MyMessageHandler(){}
};

struct MyLooperThread : public Thread {
public:
    MyLooperThread(Looper *looper)
        : mLooper(looper) {
    }
	
    virtual bool threadLoop();

protected:
    virtual ~MyLooperThread() {}

private:
    Looper *mLooper;
};

class MyPipe {
public:
    int sendFd;
    int receiveFd;
    
    MyPipe();
    ~MyPipe();
    status_t writeSignal();
    status_t readSignal();
};

#endif
```

定义完成了头文件，还有源文件

```c++
//external/native_looper_test/main.cpp
#ifdef LOG_TAG
#undef LOG_TAG
#define LOG_TAG "native_looper_test"
#endif

#include "main.h"

void MyMessageHandler::handleMessage(const Message& message) {
    ALOGD("[Thread=%d] %s message.what=%d \n", gettid(), __func__, message.what);
    messages.push(message);
}

//使用Android源码中的Thread，run就会调用threadLoop，如果返回true不断循环调用threadLoop，直到返回false，不再调用
bool MyLooperThread::threadLoop() {
    if(mLooper == NULL)
        return false;
    //调用pollOnce里面也会循环处理，底层使用了epoll机制，-1的时候表示一直等下去，直到有事件返回
    int32_t ret = mLooper->pollOnce(-1);
    switch (ret) {
        case Looper::POLL_WAKE:
        case Looper::POLL_CALLBACK:
            return true;
        case Looper::POLL_ERROR:
            ALOGE("Looper::POLL_ERROR");
            return true;
        case Looper::POLL_TIMEOUT:
            // timeout (should not happen)
            return true;
        default:
            // should not happen
            ALOGE("Looper::pollOnce() returned unknown status %d", ret);
            return true;
    }
}

MyPipe::MyPipe() {
    int fds[2];
    //这里调用真正的底层linux管道初始化
    ::pipe(fds);

    receiveFd = fds[0];
    sendFd = fds[1];
}

MyPipe::~MyPipe() {
    if (sendFd != -1) {
        ::close(sendFd);
    }

    if (receiveFd != -1) {
        ::close(receiveFd);
    }
}

status_t MyPipe::writeSignal() {
    ssize_t nWritten = ::write(sendFd, "1", 1);
    return nWritten == 1 ? 0 : -errno;
}

status_t MyPipe::readSignal() {
    char buf[1];
    ssize_t nRead = ::read(receiveFd, buf, 1);
    return nRead == 1 ? 0 : nRead == 0 ? -EPIPE : -errno;
}

//回调类
class MyCallbackHandler {
public:
    MyCallbackHandler() : callbackCount(0) {}
    void setCallback(const sp<Looper>& looper, int fd, int events) {
        //往native looper注册fd，回调为staticHandler，参数为MyCallbackHandler.this
        looper->addFd(fd, 0, events, staticHandler, this);
    }

protected:
    int handler(int fd, int events) {
        callbackCount++;
        ALOGD("[Thread=%d] %s fd=%d, events=%d, callbackCount=%d\n", gettid(), __func__, fd, events, callbackCount);
        return 0;
    }

private:
    static int staticHandler(int fd, int events, void* data) {
        return static_cast<MyCallbackHandler*>(data)->handler(fd, events);
    }
    int callbackCount;
};

int main(int argc, char ** argv)
{
    //测试消息机制
    // Looper的轮询处理工作在新线程中
    sp<Looper> mLooper = new Looper(true);
    sp<MyLooperThread> mLooperThread = new MyLooperThread(mLooper.get());
    mLooperThread->run("MyLooperThread");

    // 测试消息的发送与处理
    sp<MyMessageHandler> handler = new MyMessageHandler();
    ALOGD("[Thread=%d] sendMessage message.what=%d \n", gettid(), 1);
    mLooper->sendMessage(handler, Message(1));
    ALOGD("[Thread=%d] sendMessage message.what=%d \n", gettid(), 2);
    mLooper->sendMessage(handler, Message(2));
    sleep(1);
    
    // 测试监测fd与回调callback
    MyPipe pipe;
    MyCallbackHandler mCallbackHandler;
    mCallbackHandler.setCallback(mLooper, pipe.receiveFd, Looper::EVENT_INPUT);
    ALOGD("[Thread=%d] writeSignal 1\n", gettid());
    pipe.writeSignal(); // would cause FD to be considered signalled
    sleep(1);
    mCallbackHandler.setCallback(mLooper, pipe.receiveFd, Looper::EVENT_INPUT);
    ALOGD("[Thread=%d] writeSignal 2\n", gettid());
    pipe.writeSignal();
    
    sleep(1);
    mLooperThread->requestExit();
    mLooper.clear();
}
```

这里出现了继承Thread，这个是system源码中的线程，系统中很多都会用到这个线程使用。

> 源码所示中并没有实现run方法，说明使用的是Thread父类的run方法
>
> 可以看到父类确实有run方法
>
> ```c++
> //system/core/libutils/include/utils/Thread.h
> class Thread : virtual public RefBase
> {
> public:
>     //这里默认传入的参数是true
>     explicit            Thread(bool canCallJava = true);
>     virtual             ~Thread();
>     virtual status_t    run(    const char* name,
>                                 int32_t priority = PRIORITY_DEFAULT,
>                                 size_t stack = 0);
>     virtual void        requestExit();
>     virtual status_t    readyToRun();
>             status_t    requestExitAndWait();
>             status_t    join();
>             bool        isRunning() const;
>             pid_t       getTid() const;
> 
> protected:
>             bool        exitPending() const;
>     
> private:
>     // 1)循环:如果threadLoop()返回true，如果requestExit()没有被调用，它将被再次调用。
> 	// 2)一次:如果threadLoop()返回false，线程返回后退出。
>     virtual bool        threadLoop() = 0;
> 
> private:
>     Thread& operator=(const Thread&);
>     static  int             _threadLoop(void* user);
>     const   bool            mCanCallJava;
>             thread_id_t     mThread;
>     mutable Mutex           mLock;
>             Condition       mThreadExitedCondition;
>             status_t        mStatus;
>     volatile bool           mExitPending;
>     volatile bool           mRunning;
>             sp<Thread>      mHoldSelf;
>             pid_t           mTid;
> 
> };
> ```
>
> 真实调用run方法
>
> ```c++
> //system/core/libutils/Threads.cpp
> status_t Thread::run(const char* name, int32_t priority, size_t stack)
> {
>     LOG_ALWAYS_FATAL_IF(name == nullptr, "thread name not provided to Thread::run");
> 
>     Mutex::Autolock _l(mLock);
> 	...
>     bool res;
>     if (mCanCallJava) {
>         //这里传入的是_threadLoop为函数指针
>         //this=MyLooperThread
>         //name="MyLooperThread"
>         //priority=PRIORITY_DEFAULT=0
>         //stack=0
>         //mThread=thread_id_t(-1)
>         res = createThreadEtc(_threadLoop,
>                 this, name, priority, stack, &mThread);
>     } else {
>         res = androidCreateRawThreadEtc(_threadLoop,
>                 this, name, priority, stack, &mThread);
>     }
>     ...
>     return OK;
> }
> ```
>
> 传入的参数mCanCallJava为true
>
> ```c++
> //system/core/libutils/include/utils/AndroidThreads.h
> inline bool createThreadEtc(thread_func_t entryFunction,
>                             void *userData,
>                             const char* threadName = "android:unnamed_thread",
>                             int32_t threadPriority = PRIORITY_DEFAULT,
>                             size_t threadStackSize = 0,
>                             thread_id_t *threadId = nullptr)
> {
>     //这里通过返回直来判断创建线程是否成功
>     return androidCreateThreadEtc(entryFunction, userData, threadName,
>         threadPriority, threadStackSize, threadId) ? true : false;
> }
> ```
>
> 调用到androidCreateThreadEtc
>
> ```c++
> //system/core/libutils/Threads.cpp
> static android_create_thread_fn gCreateThreadFn = androidCreateRawThreadEtc;
> 
> int androidCreateThreadEtc(android_thread_func_t entryFunction,
>                             void *userData,
>                             const char* threadName,
>                             int32_t threadPriority,
>                             size_t threadStackSize,
>                             android_thread_id_t *threadId)
> {
>     return gCreateThreadFn(entryFunction, userData, threadName,
>         threadPriority, threadStackSize, threadId);
> }
> ```
>
> 调用到androidCreateRawThreadEtc
>
> ```c++
> //system/core/libutils/Threads.cpp
> int androidCreateRawThreadEtc(android_thread_func_t entryFunction,
>                                void *userData,
>                                const char* threadName __android_unused,
>                                int32_t threadPriority,
>                                size_t threadStackSize,
>                                android_thread_id_t *threadId)
> {
>     //设置线程分离
>     pthread_attr_t attr;
>     pthread_attr_init(&attr);
>     pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
> 
>     errno = 0;
>     pthread_t thread;
>     //真是创建线程，linux方法
>     int result = pthread_create(&thread, &attr,
>                     (android_pthread_entry)entryFunction, userData);
>     pthread_attr_destroy(&attr);
>     if (result != 0) {
>         ALOGE("androidCreateRawThreadEtc failed (entry=%p, res=%d, %s)\n"
>              "(android threadPriority=%d)",
>             entryFunction, result, strerror(errno), threadPriority);
>         return 0;
>     }
> 
>     if (threadId != nullptr) {
>         *threadId = (android_thread_id_t)thread; // XXX: this is not portable
>     }
>     return 1;
> }
> ```
>
> 最终线程开启，分离线程，会走到entryFunction对应的函数指针_threadLoop，传入的参数为userData，就是传入的MyLooperThread指针
>
> ```c++
> //system/core/libutils/Threads.cpp
> int Thread::_threadLoop(void* user)
> {
>     //这里的self实际上指的是MyLooperThread
>     Thread* const self = static_cast<Thread*>(user);
> 
>     sp<Thread> strong(self->mHoldSelf);
>     wp<Thread> weak(strong);
>     self->mHoldSelf.clear()
> 
>     bool first = true;
> 
>     do {
>         bool result;
>         if (first) {
>             first = false;
>             self->mStatus = self->readyToRun();
>             result = (self->mStatus == OK);
> 
>             if (result && !self->exitPending()) {
>                 result = self->threadLoop();
>             }
>         } else {
>             result = self->threadLoop();
>         }
>         ...
>     } while(strong != nullptr);
> 
>     return 0;
> }
> ```
>
> 1. _threadLoop()这个方法就是Thread的最大秘密，它是一个while循环。创建线程时，会sp和wp一次线程本身
> 2. 如果是第一次执行会运行线程的readyToRun()方法，再执行threadLoop()，否则，直接运行threadLoop()
> 3. threadLoop()方法有返回值，如果threadLoop()返回false的时候，线程会做清理工作，然后退出while循环，结束运行
>
> 因此线程Thread中的threadLoop()能够循环处理数据就到此做了说明。Thread被创 建，Thread中的run被调用，__threadLoop()被调用，readyToRun()被调用，然后循环调用threadLoop()。并且 在threadLoop()返回false时，可以退出循环
>
> {{< image classes="fancybox center fig-100" src="/handler/Looper/Looper_5.png" thumbnail="/handler/Looper/Looper_5.png" title="">}}
>
> 总结：**threadLoop()方法有返回值，如果threadLoop()返回false的时候，线程会做清理工作，然后退出while循环，结束运行。**

最终运行起来的部分日志

```txt
06-23 17:00:44.250   1548   1548 D native_looper_test: [Thread=1548] sendMessage message.what=1
06-23 17:00:44.250   1548   1548 D native_looper_test: [Thread=1548] sendMessage message.what=2
06-23 17:00:44.250   1548   1548 D native_looper_test: [Thread=1548] handleMessage message.what=1
06-23 17:00:44.250   1548   1548 D native_looper_test: [Thread=1548] handleMessage message.what=2
06-23 17:00:45.250   1548   1548 D native_looper_test: [Thread=1548] writeSignal 1
06-23 17:00:45.250   1548   1548 D native_looper_test: [Thread=1548] handler fd=6, events=1, callbackCount=1
06-23 17:00:46.250   1548   1548 D native_looper_test: [Thread=1548] writeSignal 2
06-23 17:00:46.250   1548   1548 D native_looper_test: [Thread=1548] handler fd=6, events=2, callbackCount=1
```

# 2源码解析

## 2.1关于Looper的数据结构

直接查看下面的图



{{< image classes="fancybox center fig-100" src="/handler/Looper/Looper_1.png" thumbnail="/handler/Looper/Looper_1.png" title="">}}

| 名称                                  | 含义                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| `Message `                            | 消息的载体，代表了一个事件，通过一个`what`字段来标记是什么事件 |
| `MessageHandler/WeakMessageHandler`   | 消息处理的接口(基类), 子类通过实现`handleMessage`来实现特定`Message`的处理逻辑。`WeakMessageHandler`包含了一个`MessageHandler`的弱指针 |
| `LooperCallback/SimpleLooperCallback` | 用于`Looper`回调，实际上就是保存一个`Looper_callbackFunc`指针的包装基类。在`Looper::addFd()`方法添加监测的`fd`时来设置回调 |

## 2.2创建Looper

### 2.2.1直接构造

```c++
//system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks)
    //参数allowNonCallbacks表明是否可以在Looper_addFd时不提供callback
    : mAllowNonCallbacks(allowNonCallbacks),
      mSendingMessage(false),
      mPolling(false),
      mEpollRebuildRequired(false),
      mNextRequestSeq(WAKE_EVENT_FD_SEQ + 1),
      mResponseIndex(0),
      mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```

Looper的构造函数里主要做两件事情

- 调用eventfd(0, EFD_NONBLOCK)返回mWakeEventFd，用于唤醒epoll_wait()
-  调用rebuildEpollLocked() 创建epoll 文件描述符，并将mWakeEventFd加入到epoll监听队列中

> 这里出现了eventfd，简单介绍一下这个函数
>
> ```c
> #include <sys/eventfd.h>
> int eventfd(unsigned int initval, int flags);
> ```
>
> - initval
>
>   eventfd()创建了一个“eventfd对象”，它可以被用户空间应用程序用作事件等待/通知机制，也可以被内核用于将事件通知用户空间应用程序。该对象包含一个由内核维护的无符号64位整数(uint64_t)计数器。该计数器使用参数initval中指定的值初始化
>
>   **eventfd 是一个计数相关的fd**。计数不为零是有**可读事件**发生，`read` 之后计数会清零，`write` 则会递增计数器，因此通常初始化为0。
>
> - flags
>
>   **EFD_CLOEXEC**
>   在新的文件描述符上设置关闭执行(FD_CLOEXEC)标志。
>
>   **EFD_NONBLOCK**
>   在新打开的文件描述上设置O_NONBLOCK文件状态标志。使用这个标志可以节省对fcntl(2)的额外调用来达到相同的结果。
>
>   **EFD_SEMAPHORE**
>   为从新的文件描述符读取提供类似信号量的语义。
>
> - 返回值
>
>   作为它的返回值，eventfd()返回一个新的文件描述符，可以用来引用eventfd对象。

```c++
//system/core/libutils/Looper.cpp
constexpr uint64_t WAKE_EVENT_FD_SEQ = 1;

void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    if (mEpollFd >= 0) {
        mEpollFd.reset();
    }

    // 分配新的epoll实例并注册WakeEventFd
    mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));
	//添加了第一个fd到mEpollFd中
    epoll_event wakeEvent = createEpollEvent(EPOLLIN, WAKE_EVENT_FD_SEQ);
    int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &wakeEvent);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                        strerror(errno));
	//std::unordered_map<SequenceNumber, Request> mRequests;
    for (const auto& [seq, request] : mRequests) {
        epoll_event eventItem = createEpollEvent(request.getEpollEvents(), seq);

        int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, request.fd, &eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                  request.fd, strerror(errno));
        }
    }
}

epoll_event createEpollEvent(uint32_t events, uint64_t seq) {
    return {.events = events, .data = {.u64 = seq}};
}
```

这里涉及到IO多路复用的epoll机制

> **关于epoll机制**
>
> 关于具体epoll，请参考https://blog.csdn.net/bandaoyu/article/details/89531493
>
> - epoll高效的核心是：
>
>   1、用户态和内核太共享内存mmap
>
>   2、数据到来采用事件通知机制（而不需要轮询）
>
> - epoll的接口
>
>   epoll的接口非常简单，一共就三个函数：
>
> 1. **epoll_create系统调用**
>
>    ```c
>    int epoll_create(int size);
>    ```
>
>    epoll_create返回一个句柄，之后 epoll的使用都将依靠这个句柄来标识。参数 size是告诉 epoll所要处理的大致事件数目。不再使用 epoll时，必须调用 close关闭这个句柄。
>
>    注意：size参数只是告诉内核这个 epoll对象会处理的事件大致数目，而不是能够处理的事件的最大个数。在 Linux最新的一些内核版本的实现中，这个 size参数没有任何意义。
>
>
> 2. **epoll_ctl系统调用**
>
>    ```c
>    int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
>    ```
>
>    epoll_ctl向 epoll对象中添加、修改或者删除感兴趣的事件，返回0表示成功，否则返回–1，此时需要根据errno错误码判断错误类型。epoll_wait方法返回的事件必然是通过 epoll_ctl添加到 epoll中的。
>
>    | 名称    | 解释                                                         |
>    | ------- | ------------------------------------------------------------ |
>    | `epfd`  | `epoll_create`返回的句柄                                     |
>    | `op`    | `EPOLL_CTL_ADD`：注册新的`fd`到`epfd`中<br/>`EPOLL_CTL_MOD`：修改已经注册的`fd`的监听事件<br/>`EPOLL_CTL_DEL`：从`epfd`中删除一个`fd` |
>    | `fd`    | 需要监听的`socket`句柄`fd`                                   |
>    | `event` | 告诉内核需要监听什么事的结构体                               |
>
>    > struct epoll_event结构如下所示
>    >
>    > ```c
>    > epoll_data_t;
>    > 
>    > struct epoll_event {
>    >  __uint32_t events; /* Epoll events */
>    >  epoll_data_t data; /* User data variable */
>    > };
>    > ```
>    >
>    > events表示时间，具体时间再定义的列表中选取
>    >
>    > | 名称           | 解释                                                         |
>    > | -------------- | ------------------------------------------------------------ |
>    > | `EPOLLIN `     | 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）         |
>    > | `EPOLLOUT`     | 表示对应的文件描述符可以写                                   |
>    > | `EPOLLPRI`     | 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来 |
>    > | `EPOLLERR`     | 表示对应的文件描述符发生错误                                 |
>    > | `EPOLLHUP`     | 表示对应的文件描述符被挂断                                   |
>    > | `EPOLLONESHOT` | 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个`socket`的话，需要再次把这个`socket`加入到`EPOLL`队列里 |
>    > | `EPOLLET`      | 将`EPOLL`设为边缘触发`(Edge Triggered)`模式，这是相对于水平触发`(Level Triggered)`来说的 |
>    >
>    > data成员是一个epoll_data联合，其定义如下
>    >
>    > ```c
>    > typedef union epoll_data {
>    >  void *ptr;
>    >  int fd;
>    >  uint32_t u32;
>    >  uint64_t u64;
>    > } epoll_data_t;
>    > ```
>
> 
>
> 3. **int epoll_wait系统调用**
>
>    ```c
>    int epoll_wait(int epfd,struct epoll_event* events,int maxevents,int timeout);
>    ```
>
>    收集在 epoll监控的事件中已经发生的事件，如果 epoll中没有任何一个事件发生，则最多等待timeout毫秒后返回。epoll_wait的返回值表示当前发生的事件个数，如果返回0，则表示本次调用中没有事件发生，如果返回–1，则表示出现错误，需要检查 errno错误码判断错误类型。
>
>    | 解释                                                         | 名称        |
>    | ------------------------------------------------------------ | ----------- |
>    | `epoll`的描述符                                              | `epfd`      |
>    | 分配好的 `epoll_event`结构体数组，`epoll`将会把发生的事件复制到 `events`数组中（`events`不可以是空指针，内核只负责把数据复制到这个` events`数组中，不会去帮助我们在用户态中分配内存。内核这种做法效率很高） | `events`    |
>    | 表示本次可以返回的最大事件数目，通常 `maxevents`参数与预分配的`events`数组的大小是相等的 | `maxevents` |
>    | 表示在没有检测到事件发生时最多等待的时间（单位为毫秒）。一般如果网络主循环是单独的线程的话，可以用-1来等，这样可以保证一些效率，如果是和主逻辑在同一个线程的话，则可以用0来保证主循环的效率<br />0的时候表示马上返回<br />-1的时候表示一直等下去，直到有事件返回<br />为任意正整数的时候表示等这么长的时间，如果一直没有事件，则返回 | `timeout`   |



### 2.2.2通过prepare创建

```c++
//system/core/libutils/Looper.cpp
sp<Looper> Looper::prepare(int opts) {
    //PREPARE_ALLOW_NON_CALLBACKS = 1
    bool allowNonCallbacks = opts & PREPARE_ALLOW_NON_CALLBACKS;
    sp<Looper> looper = Looper::getForThread();
    if (looper == nullptr) {
        looper = sp<Looper>::make(allowNonCallbacks);
        Looper::setForThread(looper);
    }
    if (looper->getAllowNonCallbacks() != allowNonCallbacks) {
        ALOGW("Looper already prepared for this thread with a different value for the "
                "LOOPER_PREPARE_ALLOW_NON_CALLBACKS option.");
    }
    return looper;
}
```

这里的创建跟Java层的Handler类型，也是先判断当前线程是否存在Looper，没有Looper就和当前线程绑定一个Looper

```c++
//system/core/libutils/Looper.cpp
static pthread_once_t gTLSOnce = PTHREAD_ONCE_INIT;
static pthread_key_t gTLSKey = 0;
void Looper::initTLSKey() {
    int error = pthread_key_create(&gTLSKey, threadDestructor);
    LOG_ALWAYS_FATAL_IF(error != 0, "Could not allocate TLS key: %s", strerror(error));
}
//现成结束的时候，会调用到对应的threadDestructor，并释放其中对应的Looper指针
void Looper::threadDestructor(void *st) {
    Looper* const self = static_cast<Looper*>(st);
    if (self != nullptr) {
        self->decStrong((void*)threadDestructor);
    }
}

sp<Looper> Looper::getForThread() {
    //这个initTLSKey有且仅会执行一次
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");
	//从这里可以得知pthread_key_create对应传入的函数指针的void*为Looper*
    Looper* looper = (Looper*)pthread_getspecific(gTLSKey);
    return sp<Looper>::fromExisting(looper);
}
```



> 1）关于pthread_once解析，**第二个参数的函数指针会被执行有且仅有一次**
>
> ```c
> int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));
> ```
>
> pthread_once的作用为，在给定once_control的情况下，在整个程序中仅调用一次init_routine（在多线程中具体哪一个线程执行不一定）。如果再次调用，pthread_once将不会调用init_routine。
>
> 该函数执行成功返回0，执行失败返回错误码。
>
> 2）另外这里涉及到线程存储的用法，具体可以点击[这里](https://blog.csdn.net/qq_42956653/article/details/126129532)
>
> 调用 pthread_key_create() 来创建一个类型为 pthread_key_t 类型的变量。
>
> ```c
> int pthread_key_create(pthread_key_t *key, void (*destr_function) (void*));
> ```
>
> 该函数有两个参数，第一个参数就是上面声明的 pthread_key_t 变量，第二个参数是一个清理函数，用来在线程释放该线程存储的时候被调用。该函数指针可以设成 NULL ，这样系统将调用默认的清理函数。**那么在线程执行完毕退出时，已key指向的内容为入参调用destr_function(),释放分配的缓冲区以及其他数据**。
>
> **key是全局变量，这个全局变量可以认为是线程内部的私有空间。不论哪个线程调用了pthread_key_create，所创建的key都是所有的线程都可以访问，每个线程根据自己的需求往key中set不同的值，这就形成了同名而不同值，即同key不同value，一键多值。**
>
> 当线程中需要存储特殊值的时候，可以调用 pthread_setspcific() 。该函数有两个参数，第一个为前面声明的 pthread_key_t 变量，第二个为 void* 变量，这样你可以存储任何类型的值。
>
> ```c
> int pthread_setspecific(pthread_key_t key, const void *pointer);
> ```
>
> 该接口将指针pointer的值(指针值而非其指向的内容)与key相关联，用pthread_setspecific为一个键指定新的线程数据时，线程必须释放原有的数据用以回收空间。**这个pointer参数会正好对应上面pthread_key_create函数中第二个函数指针参数的void***。
>
> ```c
> void * pthread_getspecific(pthread_key_t key);
> ```
>
> 读函数就是将与key关联的数据val读出来，数据类型为void * ，所有可以指向任何类型的数据。通过上面我们得知，数据存在一个32 * 32的二维数组中，所以访问的时候，也是通过计算key值得到数据的位置再返回其内容的

设置线程

```c++
//system/core/libutils/Looper.cpp
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS

    if (looper != nullptr) {
        looper->incStrong((void*)threadDestructor);
    }

    pthread_setspecific(gTLSKey, looper.get());

    if (old != nullptr) {
        old->decStrong((void*)threadDestructor);
    }
}
```

这个函数的意义，就是以gTLSKey为key，以传入的Looper的sp对象为value进行数据绑定，即完成**当前线程和Looper对象的绑定**，然后当线程执行完毕退出的时候，会调用对应的threadDestructor函数去释放对应绑定的Looper的sp对象。

## 2.3发送消息

```c++
//system/core/libutils/Looper.cpp
//1.直接发消息
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now, handler, message);
}

//2.延时发消息
void Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
                                const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now + uptimeDelay, handler, message);
}
```

最终都会调用到sendMessageAtTime

```c++
//system/core/libutils/Looper.cpp
//这里的Vector不是stl中的vector，这个是Android自定义的数组，并且是排序好的
Vector<MessageEnvelope> mMessageEnvelopes;
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { // acquire lock
        AutoMutex _l(mLock);
		
        size_t messageCount = mMessageEnvelopes.size();
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }
		//实际上会把消息根据时间优先级去插入到对应的数组中去
        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

		//如果loop当前正在发送消息，那么我们可以跳过对wake()的调用
        //对应后面的轮询机制的操作，循环机制已经在处理消息的时候，不需要再次唤醒
        if (mSendingMessage) {
            return;
        }
    } // release lock

    //只有当我们在头部排队新消息时，才会唤醒投票循环
    if (i == 0) {
        wake();
    }
}
```

根据uptime在mMessageEnvelopes遍历，找到合适的位置，并将message 封装成MessageEnvlope，插入找到的位置上。
然后决定是否要唤醒Looper：

1. 如果Looper此时正在派发message，则不需要wakeup Looper。因为这一次looper处理完消息之后，会重新估算下一次epoll_wait() 的wakeup时间。
2. 如果是插在消息队列的头部，则需要立即wakeup Looper

这里的wake的作用，暂时按下不表，在下面的轮询机制会得到解释

## 2.4注册监听fd的方法

根据上面demo实际上调用的是int addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data);

```c++
//system/core/libutils/Looper.cpp
enum {
    POLL_WAKE = -1,
    POLL_CALLBACK = -2,
    POLL_TIMEOUT = -3,
    POLL_ERROR = -4,
};
//这个mRequests不做排序，可以认为是k-v的键值对
std::unordered_map<SequenceNumber, Request> mRequests;
//这里将自定义的demo中的callback封装成SimpleLooperCallback的callback
int Looper::addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data) {
    sp<SimpleLooperCallback> looperCallback;
    if (callback) {
        looperCallback = sp<SimpleLooperCallback>::make(callback);
    }
    return addFd(fd, ident, events, looperCallback, data);
}

int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
    //可能传入的callback为nullptr
    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }

        if (ident < 0) {
            ALOGE("Invalid attempt to set NULL callback with ident < 0.");
            return -1;
        }
    } else {
        //从传入的0变成传入的-2
        ident = POLL_CALLBACK;
    }

    { // acquire lock
        AutoMutex _l(mLock);
        // There is a sequence number reserved for the WakeEventFd.
        if (mNextRequestSeq == WAKE_EVENT_FD_SEQ) mNextRequestSeq++;
        const SequenceNumber seq = mNextRequestSeq++;

        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.callback = callback;
        request.data = data;
		//通过传入的events和序列号来获取fd相关的eventItem
        epoll_event eventItem = createEpollEvent(request.getEpollEvents(), seq);
        auto seq_it = mSequenceNumberByFd.find(fd);
        if (seq_it == mSequenceNumberByFd.end()) {
            //在缓存中没有找到，就添加一个新的eventItem
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, fd, &eventItem);
            if (epollResult < 0) {
                ALOGE("Error adding epoll events for fd %d: %s", fd, strerror(errno));
                return -1;
            }
            //并且将封装好的request和序列号seq（第一个fd的seq为3）封装成新的mRequests
            mRequests.emplace(seq, request);
            mSequenceNumberByFd.emplace(fd, seq);
        } else {
            //如果在缓存中找到对应的eventItem，则修改对应的value值
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_MOD, fd, &eventItem);
            ...
            //并且更新对应的seq
            const SequenceNumber oldSeq = seq_it->second;
            mRequests.erase(oldSeq);
            mRequests.emplace(seq, request);
            seq_it->second = seq;
        }
    } // release lock
    return 1;
}
```



## 2.5轮询机制

看到上面无论是发送消息还是注册监听fd，还没有和我们的demo产生关联，这个时候需要我们的轮询机制来让整个Looper动起来。

demo中启动对应的MyLooperThread现成，根据前文可知threadLoop()返回值为true，说明本身就会一直调用threadLoop。

其中threadLoop中

```c++
//external/native_looper_test/main.cpp
//使用Android源码中的Thread，run就会调用threadLoop，如果返回true不断循环调用threadLoop，直到返回false，不再调用
bool MyLooperThread::threadLoop() {
    if(mLooper == NULL)
        return false;
    //调用pollOnce里面也会循环处理，底层使用了epoll机制，-1的时候表示一直等下去，直到有事件返回
    int32_t ret = mLooper->pollOnce(-1);
    switch (ret) {
        case Looper::POLL_WAKE:
        case Looper::POLL_CALLBACK:
            return true;
        case Looper::POLL_ERROR:
            ALOGE("Looper::POLL_ERROR");
            return true;
        case Looper::POLL_TIMEOUT:
            // timeout (should not happen)
            return true;
        default:
            // should not happen
            ALOGE("Looper::pollOnce() returned unknown status %d", ret);
            return true;
    }
}
```

pollOnce(-1)

```c++
//system/core/libutils/Looper.cpp
enum {
    POLL_WAKE = -1,
    POLL_CALLBACK = -2,
    POLL_TIMEOUT = -3,
    POLL_ERROR = -4,
};

int pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData);
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, nullptr, nullptr, nullptr);
}
//传入的只有timeoutMillis=-1，其余为nullptr
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        //这里开始判断mResponses，不过这里只看ident大于等于0，而上面有回调的会变成-2就不走这里
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != nullptr) *outFd = fd;
                if (outEvents != nullptr) *outEvents = events;
                if (outData != nullptr) *outData = data;
                return ident;
            }
        }
		//每次的轮询会返回一个值，这个值通常为POLL_WAKE、POLL_CALLBACK、POLL_TIMEOUT、POLL_ERROR
        if (result != 0) {
            if (outFd != nullptr) *outFd = 0;
            if (outEvents != nullptr) *outEvents = 0;
            if (outData != nullptr) *outData = nullptr;
            return result;
        }
		//开始轮询
        result = pollInner(timeoutMillis);
    }
}
```

真正开始轮询的地方为pollInner

```c++
//system/core/libutils/Looper.cpp
static const int EPOLL_MAX_EVENTS = 16;
Vector<Response> mResponses;//这个是排序的mResponses，根据mResponses中的seq排序

int Looper::pollInner(int timeoutMillis) {
    //重新调整时间，这里传入的是消息队列等待才会调整，不然消息队列时间没到就会一直等待
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
            && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }
    // Poll.
    //默认为唤醒，唤醒的含义是消息队列过来，用于从子线程给主线程发送消息的
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mPolling = true;
	//这里最大为16
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    //出去唤醒句柄，实际上其他文件相关的fd注册，一次性不能超过15
    //1.根据之前的epoll机制了解到timeoutMillis=-1，一直阻塞直到有唤醒或者注册的fd发生变化（变化包括读写）
    //2.只有消息队列时间还没到会调整时间，所以这里timeoutMillis调整过后不会一直阻塞，会循环调用
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();
	
    //如果epoll出现问题，本轮轮询结束
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    //如果返回0，说明传入的timeoutMillis时间到了，走到这里必须timeoutMillis>0
    //由于不会一直阻塞，所以隔段时间就会返回一次超时
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }

    // 这里开始处理事件了
    for (int i = 0; i < eventCount; i++) {
        const SequenceNumber seq = eventItems[i].data.u64;
        uint32_t epollEvents = eventItems[i].events;
        //如果是唤醒的事件响应了，那么执行awoken
        if (seq == WAKE_EVENT_FD_SEQ) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            //如果不为唤醒事件，在demo中就是想Looper添加监听的fd
            const auto& request_it = mRequests.find(seq);
            if (request_it != mRequests.end()) {
                const auto& request = request_it->second;
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //将对应的时间request取出放入到mResponses
                mResponses.push({.seq = seq, .events = events, .request = request});
            }
        }
    }
Done: ;
    //最终都会走到这里
    mNextMessageUptime = LLONG_MAX;
    //这块属于消息机制发送处理
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        //根据当前时间和排序的队列时间判断，如果队列中的时间更早那么处理发送的消息
        //如果时间没到等下轮消息轮询
        if (messageEnvelope.uptime <= now) {
            // we reacquire our lock.
            { // obtain handler
                //这里的handler就是传入的外部MessageHandler，即为MyMessageHandler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                //先移除队列中的消息，在处理消息
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
				//处理消息，如果是刚加入的队列中的元素也会判断时间执行里面内容
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            //消息机制也是返回一个POLL_CALLBACK
            result = POLL_CALLBACK;
        }
    }

    // Release lock.
    mLock.unlock();

    // 回调对应的fd监听对应的函数
    //将刚刚的mResponses中的元素取出
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        //如果存在回调，那么处理这个response
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            //回调到对应的handleEvent，先处理再移除对应的response
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                AutoMutex _l(mLock);
                removeSequenceNumberLocked(response.seq);
            }
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

这个方法特别长

1. **调整时间**

   mNextMessageUptime 是 消息队列 mMessageEnvelopes 中最近一个即将要被处理的message的时间点。所以需要根据mNextMessageUptime 与 调用者传下来的timeoutMillis 比较计算出一个最小的timeout。**通常是只有消息队列时间还没到的情况**。

2. epoll_wait

   处理epoll_wait() 返回的epoll events.判断epoll event 是哪个fd上发生的事件。如果是mWakeEventFd，则执行awoken(). awoken() 只是将数据read出来，然后继续往下处理了。其目的也就是使epoll_wait() 从阻塞中返回。

3. **监听队列的fd准备**

   如果是通过Looper.addFd() 接口加入到epoll监听队列的fd，并不是立马处理，而是先push到mResponses，后面再处理。

4. **处理消息队列和监听队列的fd**

   处理消息队列 mMessageEnvelopes 中的Message.如果还没有到处理时间，就更新一下mNextMessageUptime

   处理刚才放入mResponses中的 事件.只处理ident 为POLL_CALLBACK的事件。其他事件在pollOnce中处理

其中handleEvent

```c++
//system/core/libutils/Looper.cpp
int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    //这里的mCallback就是竹签addFd中的回调
    //回调会返回fd，event事件还有传入的data，这个data就是demo中的this指针，指代MyCallbackHandler
    return mCallback(fd, events, data);
}
```

另外解释上面的还有一个问题wake

```c++
//system/core/libutils/Looper.cpp
void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            LOG_ALWAYS_FATAL("Could not write wake signal to fd %d (returned %zd): %s",
                             mWakeEventFd.get(), nWrite, strerror(errno));
        }
    }
}
```

现在看就明白含义了，就是发生一个写操作，让注册到epoll中的mWakeEventFd发生变化，最终破除epoll_wait的阻塞，直接调用到后面的处理过程。

对应唤醒的事件响应了，那么执行awoken

```c++
//system/core/libutils/Looper.cpp
void Looper::awoken() {
    uint64_t counter;
    TEMP_FAILURE_RETRY(read(mWakeEventFd.get(), &counter, sizeof(uint64_t)));
}
```

发生一个读操作，让之前的写操作清除。

## 2.6图解

### 2.6.1发送消息

{{< image classes="fancybox center fig-100" src="/handler/Looper/Looper_2.png" thumbnail="/handler/Looper/Looper_2.png" title="">}}

### 2.6.2注册监听fd

{{< image classes="fancybox center fig-100" src="/handler/Looper/Looper_3.png" thumbnail="/handler/Looper/Looper_3.png" title="">}}

# 3补充

解释一下为什么上述demo的fd=6

{{< image classes="fancybox center fig-100" src="/handler/Looper/Looper_4.png" thumbnail="/handler/Looper/Looper_4.png" title="">}}

# 4总结

通过上述直到native Looper的机制之后，可以更加顺利的阅读其他类型的源码，或者可以在自己的demo中使用现成的消息机制。

# 下载

点击[这里](https://github.com/YangYang48/project/tree/master/Looper)

# 参考文献

[[1]  二的次方. Android Native -- Message/Handler/Looper机制（应用篇）, 2021.](https://www.cnblogs.com/roger-yu/p/15100416.html)

[[2] 二的次方,Android Native -- Message/Handler/Looper机制,2021](https://www.cnblogs.com/roger-yu/p/15099541.html).

[[3] bandaoyu, 【epoll】epoll使用详解（精髓）--研读和修正, 2019.](https://blog.csdn.net/bandaoyu/article/details/89531493)

[[4] 安德路, Android Thread之threadLoop方法, 2018.](https://blog.csdn.net/ch853199769/article/details/79917188)

[[5] 煎鱼（EDDYCJY）, Linux fd 系列 — eventfd 是什么？, 2021.](https://blog.csdn.net/EDDYCJY/article/details/118980819)

[[6] donaldsy, pthread_once使用 -- 仅初始化一次, 2021.](https://blog.51cto.com/u_3826358/3832670)

[[7] cheems~, Linux线程私有数据Thread-specific Data(TSD) 详解, 2022.](https://blog.csdn.net/qq_42956653/article/details/126129532)
