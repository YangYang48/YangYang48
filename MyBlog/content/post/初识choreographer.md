---
title: "初识choreographer"
date: 2022-05-04
thumbnailImagePosition: left
thumbnailImage: choreographer/choreographer_thumb.jpg
coverImage: choreographer/choreographer_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- choreographer
- 2022
- May
tags:
- Surfaceflinger
- Android
- performance
- Handler
- ThreadLocal
- 同步屏障
- 源码
showSocial: false
---
屏幕的刷新率一般为60fps，当然最新的手机刷新率更高，这一切都是surfacelinger的安排，通过一个叫Choreographer来监控和保证应用的帧率为固定的1/60s。

<!--more-->
# 1choreographer概述

我们直接看源码中的翻译解释

用于协调动画、输入和绘图的时间。编舞者从显示子系统接收定时脉冲（例如垂直同步），然后安排工作作为渲染下一显示帧的一部分。应用程序通常使用动画框架或视图层次结构中的更高级别抽象间接与编舞者交互。以下是您可以使用更高级别 API 执行的操作的一些示例。

> 1. 要定期显示要与显示帧渲染同步处理的动画，请使用 {@link android.animation.ValueAnimator#start}。
> 2. 要显示{@link Runnable} 以在下一个显示帧开始时调用一次，请使用 {@link View#postOnAnimation}
> 3. 要显示{@link Runnable} 以在延迟后的下一个显示帧开始时调用一次，请使用 {@link View#postOnAnimationDelayed}
> 4. 要在下一个显示帧开始时发布一次对 {@link View#invalidate()} 的调用，请使用 {@link View#postInvalidateOnAnimation()} 或 {@link View#postInvalidateOnAnimation(int,int, int, int)}。
> 5. 如果您的应用程序在不同的线程中进行渲染，可能使用 GL，或者根本不使用动画框架或视图层次结构，并且您希望确保它与显示适当同步，则使用 {@link choreography#postFrameCallback}。每个 {@link Looper} 线程都有自己的choreographer。其他线程可以发布回调以在编排器上运行，但它们将在编排器所属的 {@link Looper} 上运行。

关于上面的几点现在看会有点懵，笔者这里先按下不表，继续往下看。

## 1.1choreography由来

界面的显示大体会经过CPU的计算-> GPU合成栅格化->显示设备显示。我们知道Android设备的刷新频率一般都是60HZ也就是一秒60次，如果一次绘制在约16毫喵内完成时没有问题的。但是一旦出现不协调的地方就会出问题如下图

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_21.png" thumbnail="/choreographer/choreographyer_21.png" title="">}}

- 第一个周期，cpu计算、GPU操作、显示设备显示没有问题。
- 第二个周期，显示上一个周期绘制准备好的图像；CPU计算是在周期将要结束的时候才开始计算
- 第三个周期，由于进入第二个周期的时候CPU动手计算晚点了，致使进入第三个周期的时候本应该显示的图像没有准备好，导致整个第三个周期内还是显示上一个周期的图像，这样看上去会卡，掉帧！google工程师们管整个叫Jank延误军机。



垂直同步简单的说，就是让CPU计算别没有计划没有规律而是在每个周期开始的时候开始计算，这样就有条不紊的有序进行了（如下图）。因此在android4.1及以后的版本中加入了Choreographer这个类，Choreographer收到VSync信号才开始绘制，保证绘制拥有完整的16.7ms，避免绘制的随机性。

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_22.png" thumbnail="/choreographer/choreographyer_22.png" title="">}}

## 1.2choreography作用

这个原理可以简单的理解为是Handler的消息机制的变体。只不过这里用到了更多的异步消息的同步屏障处理。

Choreographer 的引入，主要是配合 Vsync ，给上层 App 的渲染**提供一个稳定的 Message 处理的时机，也就是 Vsync 到来的时候 ，系统通过对 Vsync 信号周期的调整，来控制每一帧绘制操作的时机. 至于为什么 Vsync 周期选择是 16.7ms (60 fps)** ，是因为目前大部分手机的屏幕都是 60Hz 的刷新率，也就是 16.7ms 刷新一次，系统为了配合屏幕的刷新频率，将 Vsync 的周期也设置为 16.7 ms，每隔 16.7 ms ，Vsync 信号到来唤醒 Choreographer 来做 App 的绘制操作 ，如果每个 Vsync 周期应用都能渲染完成，那么应用的 fps 就是 60 ，给用户的感觉就是非常流畅，这就是引入 Choreographer 的主要作用。

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_1.png" thumbnail="/choreographer/choreographyer_1.png" title="">}}

# 2源码分析

本文主要结合android11的源码分析。

## 2.1Choreographer 的初始化

首先是choreographyer的启动。在Activity启动过程，执行完onResume后，会调用Activity.makeVisible()，然后再调用到addView()， 层层调用会进入如下方法

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java#ViewRootImpl
public ViewRootImpl(Context context, Display display, IWindowSession session,
                    boolean useSfChoreographer) {
    ...
    //开始初始化mChoreographer，默认的传入的useSfChoreographer都为false
    mChoreographer = useSfChoreographer
        ? Choreographer.getSfInstance() : Choreographer.getInstance();
    ...
}
```

从ViewRootImpl.java开始构造Choreographer。当前所在线程为UI线程，也就是常说的主线程。但是实际上UI也可以在子线程更新，所以这里有一个是否为主线程的looper判断。

```java
//frameworks/base/core/java/android/view/Choreographer.java#getInstance
public static Choreographer getInstance() {
    return sThreadInstance.get();
}

//通过Threadlocal来获取choreographer的单例，标记为VSYNC_SOURCE_APP
//useSfChoreographer传入的值为true的时候，标记为VSYNC_SOURCE_SURFACE_FLINGER
private static final ThreadLocal<Choreographer> sThreadInstance =
    new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        //获取当前线程的looper
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        //通常looper都是主线程的looper，但也存在子线程更新UI的情况
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
    }
};
```

### 2.1.1Choreographer单例构造

这个Choreographer最重要的三个实例化的属性，mHandler，mDisplayEventReceiver和mCallbackQueues

```java
//frameworks/base/core/java/android/view/Choreographer.java#Choreographer
private static final boolean USE_VSYNC = SystemProperties.getBoolean(
    "debug.choreographer.vsync", true);

private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    //[2.1.2]
    mHandler = new FrameHandler(looper);
    //这里的USE_VSYNC默认就是开启的，除非手动通过设置属性将Choreographer关闭。
    //关闭方式setprop debug.choreographer.vsync false
    //[2.1.3]
    mDisplayEventReceiver = USE_VSYNC
        ? new FrameDisplayEventReceiver(looper, vsyncSource)
        : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    //[2.1.4]
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
    // b/68769804: For low FPS experiments.
    setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

> - mLastFrameTimeNanos：是指上一次帧绘制时间点；
> - mFrameIntervalNanos：帧间时长，一般等于16.7ms.

### 2.1.2mHandler

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameHandler
mHandler = new FrameHandler(looper);

//实际上就是内部的一个Handler，用于处理内部事件，这里的事件分成三项
//FrameHandler的事件
//MSG_DO_FRAME
//MSG_DO_SCHEDULE_VSYNC
//MSG_DO_SCHEDULE_CALLBACK
private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        //初始化handler
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME:
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK:
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
```

### 2.1.3mDisplayEventReceiver

这个可以看到里面有几个属性，分别为mHavePendingVsync，mTimestampNanos和mFrame，另外有两个回调onVsync和run。具体在后面展开关于DisplayEventReceiver描述。

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameDisplayEventReceiver
//这里的vsyncSource为VSYNC_SOURCE_APP或者为VSYNC_SOURCE_SURFACE_FLINGER
mDisplayEventReceiver = new FrameDisplayEventReceiver(looper, vsyncSource);

private final class FrameDisplayEventReceiver extends DisplayEventReceiver
    implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;

    //【见小节2.2Choreographer中Vsync的注册】
    public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
        super(looper, vsyncSource, CONFIG_CHANGED_EVENT_SUPPRESS);
    }

    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        long now = System.nanoTime();
        if (timestampNanos > now) {
            Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                  + " ms in the future!  Check that graphics HAL is generating vsync "
                  + "timestamps using the correct timebase.");
            timestampNanos = now;
        }

        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be "
                  + "one at a time.");
        } else {
            mHavePendingVsync = true;
        }

        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

### 2.1.4mCallbackQueues

这个是一个数据结构为CallbackQueue类型的数组，元素大小为5，分别代表CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL和CALLBACK_COMMIT。每个元素都有CallbackRecord类型的按时间戳排列的优先级队列mHead。

mCallbackQueues数据结构图，可以通过下面代码可知是按时间的优先级排序的链表队列

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_16.png" thumbnail="/choreographer/choreographyer_16.png" title="">}}

1）**提取**extractDueCallbacksLocked

此方法就是为了得到执行时间小于当前时间的Callback链表集合，得到之后会将链表断成两份，前面一份为需要马上执行的Callback，后面一份为还未到执行时间的Callback，这样保证了下一次绘制请求的Callback能够放在后面一个链表中，最早也在下一个Vsync到才执行



{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_18.png" thumbnail="/choreographer/choreographyer_18.png" title="">}}

2）**插入**addCallbackLocked

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_17.png" thumbnail="/choreographer/choreographyer_17.png" title="">}}

3）**消费**removeCallbacksLocked

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_19.png" thumbnail="/choreographer/choreographyer_19.png" title="">}}

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameDisplayEventReceiver
public static final int CALLBACK_INPUT = 0;
public static final int CALLBACK_ANIMATION = 1;
public static final int CALLBACK_INSETS_ANIMATION = 2;
public static final int CALLBACK_TRAVERSAL = 3;
public static final int CALLBACK_COMMIT = 4;
private static final int CALLBACK_LAST = CALLBACK_COMMIT;

mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];

//CallbackRecord是一个数据类型，主要用于存储三个属性，dueTime，action和token
//action为Runnable或者 FrameCallback两者的子类
//token会在后面体现出来，主要是用于区分两个api的调用，分别为FRAME_CALLBACK_TOKEN和其他，
//其中postFrameCallback调用的api定义token为FRAME_CALLBACK_TOKEN
//其中postCallback调用的api定义token为其他
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    @UnsupportedAppUsage
    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}

private final class CallbackQueue {
    //单个元素内部有一个CallbackRecord类型的mHead队列
    private CallbackRecord mHead;

    public boolean hasDueCallbacksLocked(long now) {
        return mHead != null && mHead.dueTime <= now;
    }
    //将对应元素中维护的mHead队列元素取出来
    //将小于等于传入时间戳的mHead队列元素取出来
    public CallbackRecord extractDueCallbacksLocked(long now) {
        CallbackRecord callbacks = mHead;
        //如果将元素取出来消费，必须使得mHead队列头部的时间要早于需要时间
        if (callbacks == null || callbacks.dueTime > now) {
            return null;
        }

        CallbackRecord last = callbacks;
        CallbackRecord next = last.next;
        while (next != null) {
            if (next.dueTime > now) {
                last.next = null;
                break;
            }
            last = next;
            next = next.next;
        }
        mHead = next;
        return callbacks;
    }

    //按照元素种类添加对应的内部mHead队列元素
    public void addCallbackLocked(long dueTime, Object action, Object token) {
        CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
        CallbackRecord entry = mHead;
        //如果之前没有mHead队列，那么传入的单个元素callback就为新的mHead队列的队头
        if (entry == null) {
            mHead = callback;
            return;
        }
        //将单个元素callback插入到mHead队列前面
        if (dueTime < entry.dueTime) {
            callback.next = entry;
            mHead = callback;
            return;
        }
        //将单个元素callback按照时间优先级插入到mHead队列中
        while (entry.next != null) {
            if (dueTime < entry.next.dueTime) {
                callback.next = entry.next;
                break;
            }
            entry = entry.next;
        }
        entry.next = callback;
    }

    //去除对应mHead对应action和token的CallbackRecord元素
    public void removeCallbacksLocked(Object action, Object token) {
        CallbackRecord predecessor = null;
        for (CallbackRecord callback = mHead; callback != null;) {
            final CallbackRecord next = callback.next;
            if ((action == null || callback.action == action)
                && (token == null || callback.token == token)) {
                if (predecessor != null) {
                    predecessor.next = next;
                } else {
                    mHead = next;
                }
                recycleCallbackLocked(callback);
            } else {
                predecessor = callback;
            }
            callback = next;
        }
    }
}
```

## 2.2Choreographer中Vsync的注册

承接上述的[2.1.3]，主要是对DisplayEventReceiver进行一个初始化，并且对Vysnc信号进行一个注册

### 2.2.1DisplayEventReceiver的构造

```java
//frameworks/base/core/java/android/view/DisplayEventReceiver.java#DisplayEventReceiver
public DisplayEventReceiver(Looper looper, int vsyncSource, int configChanged) {
    if (looper == null) {
        throw new IllegalArgumentException("looper must not be null");
    }

    mMessageQueue = looper.getQueue();
    //[2.2.2]
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
            vsyncSource, configChanged);

    mCloseGuard.open("dispose");
}
```

### 2.2.2nativeInit

```c++
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp#nativeInit
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj, jint vsyncSource, jint configChanged) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    ...
    //[2.2.3]
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue, vsyncSource, configChanged);
    //[2.2.5]
    status_t status = receiver->initialize();

    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
```

### 2.2.3NativeDisplayEventReceiver

```c++
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp#NativeDisplayEventReceiver
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<MessageQueue>& messageQueue, jint vsyncSource,
        jint configChanged) :
        DisplayEventDispatcher(messageQueue->getLooper(),
                static_cast<ISurfaceComposer::VsyncSource>(vsyncSource),
                static_cast<ISurfaceComposer::ConfigChanged>(configChanged)),
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}
```

在初始化NativeDisplayEventReceiver的同时，由于也初始化了基类的构造

```c++
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp#DisplayEventDispatcher
DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper,
                                               ISurfaceComposer::VsyncSource vsyncSource,
                                               ISurfaceComposer::ConfigChanged configChanged)
        //初始化mLooper，mReceiver和类成员变量mWaitingForVsync，这个值在后续的processPendingEvents会用到
        //mReceiver的初始化[2.2.4]
      : mLooper(looper), mReceiver(vsyncSource, configChanged), mWaitingForVsync(false) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}
```

### 2.2.4mReceiver初始化

这里面的逻辑比较清晰，首先是sf的构造，其次是mEventConnection的初始化，最后是mEventConnection的stealReceiveChannel

```c++
//frameworks/native/libs/gui/DisplayEventReceiver.cpp#DisplayEventReceiver
DisplayEventReceiver::DisplayEventReceiver(ISurfaceComposer::VsyncSource vsyncSource,
                                           ISurfaceComposer::ConfigChanged configChanged) {
    //这里有两个一个是获取Composer服务，一个是sf的构造
    //ComposerService::getComposerService(),[2.2.4.1]
    //sf的构造，[2.2.4.3]
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr) {
        //[2.2.4.4]
        mEventConnection = sf->createDisplayEventConnection(vsyncSource, configChanged);
        if (mEventConnection != nullptr) {
            //获取到一个BitTube的智能指针变量mDataChannel
            mDataChannel = std::make_unique<gui::BitTube>();
            //[2.2.4.12]
            mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}
```

#### 2.2.4.1getComposerService

getComposerService函数定义在ComposerService.h头文件中，内部持有ISurfaceComposer的成员变量

```c++
//frameworks/native/libs/gui/include/private/gui/ComposerService.h#ComposerService
class ISurfaceComposer;

class ComposerService : public Singleton<ComposerService>
{
    sp<ISurfaceComposer> mComposerService;
    sp<IBinder::DeathRecipient> mDeathObserver;
    Mutex mLock;

    ComposerService();
    void connectLocked();
    void composerServiceDied();
    friend class Singleton<ComposerService>;
public:
    static sp<ISurfaceComposer> getComposerService();
};
```

这里获取的ComposerService是一个单例

```c++
//frameworks/native/libs/gui/SurfaceComposerClient.cpp#getComposerService
/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    Mutex::Autolock _l(instance.mLock);
    if (instance.mComposerService == nullptr) {
        //这里有一个服务获取[2.2.4.2]
        ComposerService::getInstance().connectLocked();
        assert(instance.mComposerService != nullptr);
        ALOGD("ComposerService reconnected");
    }
    return instance.mComposerService;
}
```

#### 2.2.4.2connectLocked

获取链接的远端服务，这里可以看出通过ServiceManager来获取对应服务，这个服务是SurfaceFlinger，并且设置了对应的死亡回调

> 在SurfaceFlinger服务初始化的时候，就会把对应的name添加到ServiceManager中，所有后续能够直接通过getService中的名字直接获取到对应的服务。

```c++
//frameworks/native/libs/gui/SurfaceComposerClient.cpp#connectLocked
void ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    assert(mComposerService != nullptr);

    //设置死亡回调
    class DeathObserver : public IBinder::DeathRecipient {
        ComposerService& mComposerService;
        virtual void binderDied(const wp<IBinder>& who) {
            ALOGW("ComposerService remote (surfaceflinger) died [%p]",
                  who.unsafe_get());
            mComposerService.composerServiceDied();
        }
     public:
        explicit DeathObserver(ComposerService& mgr) : mComposerService(mgr) { }
    };

    mDeathObserver = new DeathObserver(*const_cast<ComposerService*>(this));
    IInterface::asBinder(mComposerService)->linkToDeath(mDeathObserver);
}
```

#### 2.2.4.3sf的构造

这里有一个显示的构造，实际上是传入的就是impl，这里的impl就是上面getservice获取到的SurfaceFlinger实例。关于explicit关键字，可以点击[这里](https://blog.csdn.net/yu132563/article/details/80103693)。

```c++
//frameworks/native/libs/gui/ISurfaceComposer.cpp#BpSurfaceComposer#BpSurfaceComposer
class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
{
public:
    explicit BpSurfaceComposer(const sp<IBinder>& impl)
        : BpInterface<ISurfaceComposer>(impl)
    {
    }
    ...
};
```

#### 2.2.4.4createDisplayEventConnection

```c++
//frameworks/native/libs/gui/ISurfaceComposer.cpp#BpSurfaceComposer#createDisplayEventConnection
class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
{
    virtual sp<IDisplayEventConnection> createDisplayEventConnection(VsyncSource vsyncSource,
                                                                     ConfigChanged configChanged) {
        Parcel data, reply;
        sp<IDisplayEventConnection> result;
        int err = data.writeInterfaceToken(
            ISurfaceComposer::getInterfaceDescriptor());
        if (err != NO_ERROR) {
            return result;
        }
        data.writeInt32(static_cast<int32_t>(vsyncSource));
        data.writeInt32(static_cast<int32_t>(configChanged));
        //开始进行binder操作，操作内容为CREATE_DISPLAY_EVENT_CONNECTION，[2.2.4.5]
        err = remote()->transact(
            BnSurfaceComposer::CREATE_DISPLAY_EVENT_CONNECTION,
            data, &reply);
        if (err != NO_ERROR) {
            ALOGE("ISurfaceComposer::createDisplayEventConnection: error performing "
                  "transaction: %s (%d)", strerror(-err), -err);
            return result;
        }
        result = interface_cast<IDisplayEventConnection>(reply.readStrongBinder());
        return result;
    }
};
```

#### 2.2.4.5transact

通过binder通信，会调到ontrasact

```c++
//frameworks/native/libs/gui/ISurfaceComposer.cpp#BnSurfaceComposer#onTransact
status_t BnSurfaceComposer::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
         ...
         case CREATE_DISPLAY_EVENT_CONNECTION: {
            CHECK_INTERFACE(ISurfaceComposer, data, reply);
            auto vsyncSource = static_cast<ISurfaceComposer::VsyncSource>(data.readInt32());
            auto configChanged = static_cast<ISurfaceComposer::ConfigChanged>(data.readInt32());
            //其实这边还比较复杂，用一次用到了binder，这里的connection也是IDisplayEventConnection初始化
            //createDisplayEventConnection,[2.2.4.6]
            //connection,[2.2.4.11]
            sp<IDisplayEventConnection> connection(
                    createDisplayEventConnection(vsyncSource, configChanged));
            reply->writeStrongBinder(IInterface::asBinder(connection));
            return NO_ERROR;
        }
        ...
    }
}
```

可以得知目前传入的值，vsyncSource值为eVsyncSourceApp，configChanged值为CONFIG_CHANGED_EVENT_SUPPRESS

```c++
//frameworks/native/libs/gui/include/gui/ISurfaceComposer.h
enum VsyncSource {
    eVsyncSourceApp = 0,
    eVsyncSourceSurfaceFlinger = 1
};

enum ConfigChanged { eConfigChangedSuppress = 0, eConfigChangedDispatch = 1 };
```

#### 2.2.4.6createDisplayEventConnection

由于直到服务端是SurfaceFlinger，即SurfaceFlinger是继承BnSurfaceComposer的，所以最终调用到SurfaceFlinger中，这个函数主要是为了初始化EventThreadConnection

```c++
//frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp#createDisplayEventConnection
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        ISurfaceComposer::VsyncSource vsyncSource, ISurfaceComposer::ConfigChanged configChanged) {
    const auto& handle =
            vsyncSource == eVsyncSourceSurfaceFlinger ? mSfConnectionHandle : mAppConnectionHandle;
    //[2.2.4.7]
    return mScheduler->createDisplayEventConnection(handle, configChanged);
}
```

#### 2.2.4.7createDisplayEventConnection

通过Scheduler来管理EventThreadConnection和EventThread的实例

```c++
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp#createDisplayEventConnection
sp<IDisplayEventConnection> Scheduler::createDisplayEventConnection(
        ConnectionHandle handle, ISurfaceComposer::ConfigChanged configChanged) {
    RETURN_IF_INVALID_HANDLE(handle, nullptr);
    //[2.2.4.8]
    return createConnectionInternal(mConnections[handle].thread.get(), configChanged);
}
```

这里的mConnections是在头文件Scheduler.h中定义的

```c++
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h#Scheduler
class Scheduler : public IPhaseOffsetControl {
    ...
    struct Connection {
        sp<EventThreadConnection> connection;
        std::unique_ptr<EventThread> thread;
    };
	std::unordered_map<ConnectionHandle, Connection> mConnections;
    ...
};
```

#### 2.2.4.8createConnectionInternal

```c++
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp#createConnectionInternal
sp<EventThreadConnection> Scheduler::createConnectionInternal(
        EventThread* eventThread, ISurfaceComposer::ConfigChanged configChanged) {
    //[2.2.4.9]
    return eventThread->createEventConnection([&] { resync(); }, configChanged);
}
```

#### 2.2.4.9eventThread->createEventConnection

最终调用到eventThread->createEventConnection，这里还是用了内部匿名方式，用于初始化EventThreadConnection，其中这里的this指的是SurfaceFinger初始化EventThread的实例，**对于EventThread的实例化会在SurfaceFlinger篇章阐述**

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#createEventConnection
sp<EventThreadConnection> EventThread::createEventConnection(
        ResyncCallback resyncCallback, ISurfaceComposer::ConfigChanged configChanged) const {
    //[2.2.4.10]
    return new EventThreadConnection(const_cast<EventThread*>(this), std::move(resyncCallback),
                                     configChanged);
}
```

#### 2.2.4.10EventThreadConnection

构造一个EventThreadConnection

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#EventThreadConnection
EventThreadConnection::EventThreadConnection(EventThread* eventThread,
                                             ResyncCallback resyncCallback,
                                             ISurfaceComposer::ConfigChanged configChanged)
      : resyncCallback(std::move(resyncCallback)),
        mConfigChanged(configChanged),
        mEventThread(eventThread),
        mChannel(gui::BitTube::DefaultSize) {}
```



#### 2.2.4.11connection

同2.2.4.3类似，这里有一个显示的构造，实际上是传入的就是impl，这里的impl就是上面mScheduler管理的的EventThreadConnection的实例。

```c++
//frameworks/native/libs/gui/IDisplayEventConnection.cpp#BpDisplayEventConnection#BpDisplayEventConnection
class BpDisplayEventConnection : public SafeBpInterface<IDisplayEventConnection> {
public:
    explicit BpDisplayEventConnection(const sp<IBinder>& impl)
          : SafeBpInterface<IDisplayEventConnection>(impl, "BpDisplayEventConnection") {}
};
```



#### 2.2.4.12stealReceiveChannel

直接调用基类，是纯虚函数，找到对应的子类，BpDisplayEventConnection

```c++
//frameworks/native/libs/gui/include/gui/IDisplayEventConnection.h
class IDisplayEventConnection : public IInterface {
    virtual status_t stealReceiveChannel(gui::BitTube* outChannel) = 0;
};
```

BpDisplayEventConnection里面的函数会进行跨进程通信找到对应stealReceiveChannel函数

```c++
//frameworks/native/libs/gui/IDisplayEventConnection.cpp#BpDisplayEventConnection#stealReceiveChannel
class BpDisplayEventConnection : public SafeBpInterface<IDisplayEventConnection> {
    status_t stealReceiveChannel(gui::BitTube* outChannel) override {
        return callRemote<decltype(
            &IDisplayEventConnection::stealReceiveChannel)>(Tag::STEAL_RECEIVE_CHANNEL,
                                                            outChannel);
    }
};
```

最终调用到EventThreadConnection，继承BnDisplayEventConnection

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.h#EventThreadConnection
class EventThreadConnection : public BnDisplayEventConnection {
public:
    EventThreadConnection(EventThread*, ResyncCallback,
                          ISurfaceComposer::ConfigChanged configChanged);
    virtual ~EventThreadConnection();
    status_t stealReceiveChannel(gui::BitTube* outChannel) override;
    ...
};
```

stealReceiveChannel

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#stealReceiveChannel
status_t EventThreadConnection::stealReceiveChannel(gui::BitTube* outChannel) {
    outChannel->setReceiveFd(mChannel.moveReceiveFd());
    return NO_ERROR;
}
```



### 2.2.5initialize

承接上面的2.2.4之后，开始初始化，由于派生类NativeDisplayEventReceiver没有初始化函数，所以直接使用子类的初始化函数

```c++
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp#initialize
status_t DisplayEventDispatcher::initialize() {
    //这个mReceiver是DisplayEventReceiver的实例，这里用于初始化的检验，是否已经完成初始化了
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }

    if (mLooper != nullptr) {
        //这个方法比较关键，通过mReceiver的getFd获取到BitTube，用于操控底层的socket
        //设置回调为this，这里的this就是DisplayEventDispatcher的实例
        int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT, this, NULL);
        if (rc < 0) {
            return UNKNOWN_ERROR;
        }
    }

    return OK;
}
```

### 2.2.6 Choreographer中Vsync的注册图解

这块逻辑比较复杂，通过画图的方式在梳理上述流程会更加清晰明了

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_5.png" thumbnail="/choreographer/choreographyer_5.png" title="">}}

## 2.3Choreographer处理上一帧

处理一帧的逻辑是从回调开始，为了讲清楚来龙去脉，还得从native层的回调开始

实际上是本质是**双向通信管道的socketpair的receive过程**，并且获取到的回调信号事件进行**分发处理**

### 2.3.1回调处理

在上述Choreographer中Vsync的注册的2.2.5中，设置了DisplayEventDispatcher的实例的回调，那么这里就是开始的地方

```c++
//system/core/libutils/Looper.cpp#pollInner
int Looper::pollInner(int timeoutMillis) {
    ...
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            ...
            //[2.3.2]
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            ...
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

### 2.3.2DisplayEventDispatcher.handleEvent

这里的callback实际上就是上面注册Vysnc信号传入的DisplayEventDispatcher实例的this

所以监听了mReceiveFd之后，当mSendFd写入数据时就能收到消息，并回调DisplayEventDispatcher的handleEvent函数

```c++
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp#handleEvent
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    ...
    // Drain all pending events, keep the last vsync.
    nsecs_t vsyncTimestamp;
    PhysicalDisplayId vsyncDisplayId;
    uint32_t vsyncCount;
    //processPendingEvents，[2.3.3]
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64
              ", displayId=%" ANDROID_PHYSICAL_DISPLAY_ID_FORMAT ", count=%d",
              this, ns2ms(vsyncTimestamp), vsyncDisplayId, vsyncCount);
        //一旦处理了延时事件之后，设置mWaitingForVsync=false，在
        mWaitingForVsync = false;
        //dispatchVsync用于处理Vysnc信号事件，[2.3.6]
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }

    return 1; // keep the callback
}
```

### 2.3.3processPendingEvents

```c++
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp#processPendingEvents
bool DisplayEventDispatcher::processPendingEvents(nsecs_t* outTimestamp,
                                                  PhysicalDisplayId* outDisplayId,
                                                  uint32_t* outCount) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    //这个循环或重复获取getEvents，直到没有为止[2.3.4]
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        ALOGV("dispatcher %p ~ Read %d events.", this, int(n));
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    //之后的 vsync 事件只会覆盖之前的信息
                    gotVsync = true;
                    *outTimestamp = ev.header.timestamp;
                    *outDisplayId = ev.header.displayId;
                    *outCount = ev.vsync.count;
                    break;
                ...
            }
        }
    }
    ...
    return gotVsync;
}
```

### 2.3.4getEvents

最终会调用到BitTube，使用**socketpair的receive过程**

```c++
//frameworks/native/libs/gui/DisplayEventReceiver.cpp#getEvents
ssize_t DisplayEventReceiver::getEvents(DisplayEventReceiver::Event* events,
        size_t count) {
    return DisplayEventReceiver::getEvents(mDataChannel.get(), events, count);
}

ssize_t DisplayEventReceiver::getEvents(gui::BitTube* dataChannel,
        Event* events, size_t count)
{
    //[2.3.5]
    return gui::BitTube::recvObjects(dataChannel, events, count);
}
```

### 2.3.5recvObjects

最终是调用到BitTube，使用**双向通信管道的socketpair的receive过程**

> 关于socketpair
>
> socket在内核中有一个发送缓冲区和一个接收缓冲区，send函数负责将数据存入socket的发送缓冲区，recv函数负责从socket的接收缓冲区中将数据拷贝到用户空间，recv和send不一定是一一对应，也就是说并不是send一次，就一定recv一次接收完，有可能send一次，recv多次才能接收完，也可能send多次，一次recv就接收完了
>
> 简单理解就是send发数据，recv就能够收到数据

```c++
//frameworks/native/libs/gui/BitTube.cpp#recvObjects
ssize_t BitTube::recvObjects(BitTube* tube, void* events, size_t count, size_t objSize) {
    char* vaddr = reinterpret_cast<char*>(events);
    //vaddr为缓冲区的地址，count * objSize为缓冲区的大小
    ssize_t size = tube->read(vaddr, count * objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

### 2.3.6DisplayEventDispatcher.dispatchVsync

这里的dispatchVsync实际上调用的是子类NativeDisplayEventReceiver中的dispatchVsync函数

```c++
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp#dispatchVsync
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, PhysicalDisplayId displayId,
                                               uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();

    ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        //这里jni开始回调到java层的DisplayEventReceiver#dispatchVsync，[2.3.7]
        env->CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, displayId, count);
        ALOGV("receiver %p ~ Returned from vsync handler.", this);
    }

    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```

### 2.3.7DisplayEventReceiver.dispatchVsync

从这里开始，事件的分发来到了java层

```java
//frameworks/base/core/java/android/view/DisplayEventReceiver.java#dispatchVsync
private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame) {
    //[2.3.8]
    onVsync(timestampNanos, physicalDisplayId, frame);
}
```

### 2.3.8DisplayEventReceiver.onVsync

由于FrameDisplayEventReceiver继承了DisplayEventReceiver，所以实际上onVsync调用的是FrameDisplayEventReceiver中的方法

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameDisplayEventReceiver#onVsync
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
    implements Runnable {
    ...
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        long now = System.nanoTime();
        if (timestampNanos > now) {
            Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                  + " ms in the future!  Check that graphics HAL is generating vsync "
                  + "timestamps using the correct timebase.");
            timestampNanos = now;
        }

        //如果存在多个延时信号，那么当前时间只能处理一个同步信号Vsync
        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be "
                  + "one at a time.");
        } else {
            mHavePendingVsync = true;
        }

        mTimestampNanos = timestampNanos;
        mFrame = frame;
        //handler处理，使用的是Choreographer内部的mHandler，并且使用同步屏障，优先处理当前的事件
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }
    
    //在timestampNanos/TimeUtils.NANOS_PER_MS时间的时候开始处理doFrame
    @Override
    public void run() {
        mHavePendingVsync = false;
        //[2.3.9]
        doFrame(mTimestampNanos, mFrame);
    }
}
```

### 2.3.9Choreographer.doFrame

这个接下来就是doFrame方法了，这个doFrame是Choreographer内部的，**而不是FrameCallback中的doFrame**

```java
//frameworks/base/core/java/android/view/Choreographer.java#doFrame
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }
        ...
        //计算掉帧逻辑,[2.3.9.2]   
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                      + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            frameTimeNanos = startNanos - lastFrameOffset;
        }

        if (frameTimeNanos < mLastFrameTimeNanos) {
            scheduleVsyncLocked();
            return;
        }

        if (mFPSDivisor > 1) {
            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                scheduleVsyncLocked();
                return;
            }
        }
        //[2.3.9.3]
        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        //[2.3.9.1]
        mFrameScheduled = false;
        mLastFrameTimeNanos = frameTimeNanos;
    }

    //这里面的Trace.traceBegin和traceEnd，通过androidsdk的systrace可以捕获这段信息，然后再将对应的html展示出来
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
        //[2.3.9.3]
        mFrameInfo.markInputHandlingStart();
        //[2.3.9.4]
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        //[2.3.9.3]
        mFrameInfo.markAnimationsStart();
        //[2.3.9.4]
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
        //[2.3.9.3]
        mFrameInfo.markPerformTraversalsStart();
        //[2.3.9.4]
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```

下面的doFrame方法还是比较复杂的，拆解开来主要关注几点

#### 2.3.9.1变量mFrameScheduled

mFrameScheduled必须是true才能进入处理分发事件，而且同一时间只能处理一个分发事件，分发完成时间也不会马上复位。

这个会跟后面的下一帧的Vsync请求联系，直到2.4.9的scheduleFrameLocked请求事件之后，才会让上面的分发再一次得到处理，**mFrameScheduled就是Choreographer.java里面的精髓所在**，是控制完成16.7ms的关键。

#### 2.3.9.2计算掉帧逻辑

```java
//frameworks/base/core/java/android/view/Choreographer.java#doFrame
final long startNanos;
    synchronized (mLock) {
        ...
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
        }
        ...
    }
}
```

#### 2.3.9.3记录帧绘制信息

```java
//frameworks/base/core/java/android/view/Choreographer.java#doFrame
FrameInfo mFrameInfo = new FrameInfo();

mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
mFrameInfo.markInputHandlingStart();
mFrameInfo.markAnimationsStart();
mFrameInfo.markPerformTraversalsStart();
```

其中mFrameInfo，内部维护了一个长度为9的long类型的数组，每个数组元素都有具体含义，**作用是记录doCallbacks方法时间**

```java
//frameworks/base/graphics/java/android/graphics/FrameInfo.java
public final class FrameInfo {

    public long[] frameInfo = new long[9];
    // Various flags set to provide extra metadata about the current frame
    private static final int FLAGS = 0;
    // Is this the first-draw following a window layout?
    public static final long FLAG_WINDOW_LAYOUT_CHANGED = 1;
    // A renderer associated with just a Surface, not with a ViewRootImpl instance.
    public static final long FLAG_SURFACE_CANVAS = 1 << 2;

    @LongDef(flag = true, value = {
            FLAG_WINDOW_LAYOUT_CHANGED, FLAG_SURFACE_CANVAS })
    @Retention(RetentionPolicy.SOURCE)
    public @interface FrameInfoFlags {}
    // The intended vsync time, unadjusted by jitter
    private static final int INTENDED_VSYNC = 1;
    // Jitter-adjusted vsync time, this is what was used as input into the
    // animation & drawing system
    private static final int VSYNC = 2;
    // The time of the oldest input event
    private static final int OLDEST_INPUT_EVENT = 3;
    // The time of the newest input event
    private static final int NEWEST_INPUT_EVENT = 4;
    // When input event handling started
    private static final int HANDLE_INPUT_START = 5;
    // When animation evaluations started
    private static final int ANIMATION_START = 6;
    // When ViewRootImpl#performTraversals() started
    private static final int PERFORM_TRAVERSALS_START = 7;
    // When View:draw() started
    private static final int DRAW_START = 8;

    public void setVsync(long intendedVsync, long usedVsync) {
        frameInfo[INTENDED_VSYNC] = intendedVsync;
        frameInfo[VSYNC] = usedVsync;
        frameInfo[OLDEST_INPUT_EVENT] = Long.MAX_VALUE;
        frameInfo[NEWEST_INPUT_EVENT] = 0;
        frameInfo[FLAGS] = 0;
    }

    public void updateInputEventTime(long inputEventTime, long inputEventOldestTime) {
        if (inputEventOldestTime < frameInfo[OLDEST_INPUT_EVENT]) {
            frameInfo[OLDEST_INPUT_EVENT] = inputEventOldestTime;
        }
        if (inputEventTime > frameInfo[NEWEST_INPUT_EVENT]) {
            frameInfo[NEWEST_INPUT_EVENT] = inputEventTime;
        }
    }

    public void markInputHandlingStart() {
        frameInfo[HANDLE_INPUT_START] = System.nanoTime();
    }

    public void markAnimationsStart() {
        frameInfo[ANIMATION_START] = System.nanoTime();
    }

    public void markPerformTraversalsStart() {
        frameInfo[PERFORM_TRAVERSALS_START] = System.nanoTime();
    }

    public void markDrawStart() {
        frameInfo[DRAW_START] = System.nanoTime();
    }

    public void addFlags(@FrameInfoFlags long flags) {
        frameInfo[FLAGS] |= flags;
    }
}
```

#### 2.3.9.4doCallbacks

执行doCallbacks方法，获取mCallbackQueues中的对应mHead队列中的元素

```java
//frameworks/base/core/java/android/view/Choreographer.java#doFrame
doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
```

这里就调用到了doCallbacks方法

```java
//frameworks/base/core/java/android/view/Choreographer.java#doCallbacks
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        final long now = System.nanoTime();
        //这里开始获取mCallbackQueues中的对应mHead队列中的元素组
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
            now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        //获取到CallbackRecord类型的队列后，设置状态为true
        mCallbacksRunning = true;
        if (callbackType == Choreographer.CALLBACK_COMMIT) {
            final long jitterNanos = now - frameTimeNanos;
            Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
            if (jitterNanos >= 2 * mFrameIntervalNanos) {
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                    + mFrameIntervalNanos;
                frameTimeNanos = now - lastFrameOffset;
                mLastFrameTimeNanos = frameTimeNanos;
            }
        }
    }
    //主要耗时在这里，包括trace记录也是从这里开始
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            //[2.3.9.5]
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
            //CallbackRecord类型的队列Running完成之后，设置状态为false
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

这里面有一个recycleCallbackLocked方法，是为了让其置为空

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_20.png" thumbnail="/choreographer/choreographyer_20.png" title="">}}

```java
//frameworks/base/core/java/android/view/Choreographer.java#recycleCallbackLocked
private void recycleCallbackLocked(CallbackRecord callback) {
    callback.action = null;
    callback.token = null;
    callback.next = mCallbackPool;
    mCallbackPool = callback;
}
```

#### 2.3.9.5CallbackRecord.run

最重要的就是run方法，回调到CallbackRecord的run方法，具体关于run方法的图解，可以直接跳转到2.4.25关于Choreographer的流程

```java
//frameworks/base/core/java/android/view/Choreographer.java#CallbackRecord#run
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    @UnsupportedAppUsage
    public void run(long frameTimeNanos) {
        //这里根据token判断传入的api方式
        //其中postFrameCallback调用的api定义token为FRAME_CALLBACK_TOKEN,执行doFrame
        //其中postCallback调用的api定义token为其他，run
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
```

### 2.3.10Choreographer处理上一帧图解

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_8.png" thumbnail="/choreographer/choreographyer_8.png" title="">}}

## 2.4下一帧的Vsync请求

这里的Vsync信号是由SurfaceFlinger中创建HWC触发的，**具体关于Vysnc会在单独的章节阐述这个流程**。

下一帧的Vsync的请求的处理，实际上是本质是**双向通信管道的socketpair的send过程**。

>  VSync信号
>
>  VSync并不是一个实际的信号，可以认为是一条通知，EventThread收到通知之后自己构造一个Event，然后发送出去VSync类型事件构造好了之后通过条件变量唤醒陷入wait状态的EventThread内部线程，它会从mPendingEvents中拿到最头部事件调用dispatchEvent将事件分发给感兴趣的监听者，感兴趣的监听者即是向EventThread请求了Vsync的EventThreadConnection。dispatchEvent的实现很简单，最终就是通过gui::BitTube的sendObjects函数向mSendFd中写入数据，另一端监听了mReceiveFd的进程就能够收到消息，知道Vsync到来了，然后完成绘制工作

我们下面以Animation为例，使用的是postFrameCallback，定义token为FRAME_CALLBACK_TOKEN

### 2.4.1ObjectAnimator.start

```java
//frameworks/base/core/java/android/animation/ObjectAnimator.java#start
@Override
public void start() {
    AnimationHandler.getInstance().autoCancelBasedOn(this);
    //[2.4.2]
    super.start();
}
```

### 2.4.2ValueAnimator.start

```java
//frameworks/base/core/java/android/animation/ValueAnimator.java#start
@Override
public void start() {
    start(false);
}

private void start(boolean playBackwards) {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    ...
    //[2.4.3]
    addAnimationCallback(0);
    ...
}
```

### 2.4.3ValueAnimator.addAnimationCallback

```java
//frameworks/base/core/java/android/animation/ValueAnimator.java#addAnimationCallback
private void addAnimationCallback(long delay) {
    if (!mSelfPulse) {
        return;
    }
    //[2.4.4]
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```

### 2.4.4AnimationHandler.addAnimationFrameCallback

```java
//frameworks/base/core/java/android/animation/AnimationHandler.java#addAnimationFrameCallback
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        //继续下一帧的获取[2.4.5]
        getProvider().postFrameCallback(mFrameCallback);
    }
    ...
}

//最终回调
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    //Choreographer回调，处理当前动画的
    public void doFrame(long frameTimeNanos) {
        //处理当前动画的
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            //继续下一帧的获取[2.4.5]
            getProvider().postFrameCallback(this);
        }
    }
};
```

### 2.4.5getProvider().postFrameCallback

这里的provider实际上AnimationHandler内部维护了一个AnimationFrameCallbackProvider用于申请下一个Vsync信号

```java
//frameworks/base/core/java/android/animation/AnimationHandler.java#postFrameCallback
//内部维护了一个Choreographer单例，直接调用
private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

    final Choreographer mChoreographer = Choreographer.getInstance();

    @Override
    public void postFrameCallback(Choreographer.FrameCallback callback) {
        //[2.4.6]
        mChoreographer.postFrameCallback(callback);
    }
    ...
}
```

### 2.4.6postFrameCallback

```java
//frameworks/base/core/java/android/view/Choreographer.java#postFrameCallback
public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
                                callback, FRAME_CALLBACK_TOKEN, delayMillis);
}

// callbackType为动画，action为mAnimationFrameCallback
// token为FRAME_CALLBACK_TOKEN，delayMillis=0
private void postCallbackDelayedInternal(int callbackType,
                                         Object action, Object token, long delayMillis) {
    //默认DEBUG_FRAMES是false，如果需要打印手动设置DEBUG_FRAMES为true
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
              + ", action=" + action + ", token=" + token
              + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //添加到mCallbackQueues队列
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        //如果传进里面的时间比现在的时间少，那么就立即进行
        if (dueTime <= now) {
            //[2.4.9]
            scheduleFrameLocked(now);
        //如果传进里面的时间比现在的时间多，那么就handler延时处理事件
        } else {
            //[2.4.7]
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

### 2.4.7mHandler消息处理事件MSG_DO_SCHEDULE_CALLBACK

mHandler处理消息，消息内容为MSG_DO_SCHEDULE_CALLBACK

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameHandler#MSG_DO_SCHEDULE_CALLBACK
private final class FrameHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ...
            case MSG_DO_SCHEDULE_CALLBACK:
                //[2.4.8]
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
```

### 2.4.8doScheduleCallback

```java
//frameworks/base/core/java/android/view/Choreographer.java#MSG_DO_SCHEDULE_CALLBACK
void doScheduleCallback(int callbackType) {
    synchronized (mLock) {
        if (!mFrameScheduled) {
            final long now = SystemClock.uptimeMillis();
            if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                //[2.4.9]
                scheduleFrameLocked(now);
            }
        }
    }
}
```

### 2.4.9scheduleFrameLocked

```java
//frameworks/base/core/java/android/view/Choreographer.java#scheduleFrameLocked
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        //默认USE_VSYNC为true，使用的是垂直同步
        if (USE_VSYNC) {
            if (isRunningOnLooperThreadLocked()) {
                //[2.4.12]
                scheduleVsyncLocked();
            } else {
                //[2.4.10]
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
            }
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

### 2.4.10mHandler消息处理事件MSG_DO_SCHEDULE_VSYNC

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameHandler#MSG_DO_SCHEDULE_VSYNC
private final class FrameHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ...
            case MSG_DO_SCHEDULE_VSYNC:
                //[2.4.11]
                doScheduleVsync();
                break;
        }
    }
}
```

### 2.4.11doScheduleVsync

```java
//frameworks/base/core/java/android/view/Choreographer.java#doScheduleVsync
void doScheduleVsync() {
    synchronized (mLock) {
        if (mFrameScheduled) {
            //[2.4.12]
            scheduleVsyncLocked();
        }
    }
}
```

### 2.4.12scheduleVsyncLocked

```java
//frameworks/base/core/java/android/view/Choreographer.java#scheduleVsyncLocked
private void scheduleVsyncLocked() {
    //[2.4.13]
    mDisplayEventReceiver.scheduleVsync();
}
```

### 2.4.13scheduleVsync

这里开始从java层调用到native层往下

```java
//frameworks/base/core/java/android/view/DisplayEventReceiver.java#scheduleVsync
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
              + "receiver has already been disposed.");
    } else {
        //[2.4.14]
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

### 2.4.14nativeScheduleVsync

```c++
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp#nativeScheduleVsync
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    //[2.4.15]
    status_t status = receiver->scheduleVsync();
    ...
}
```

### 2.4.15scheduleVsync

由于NativeDisplayEventReceiver是继承基类DisplayEventDispatcher，所以这里调用到了基类的函数

```c++
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp#NativeDisplayEventReceiver
class NativeDisplayEventReceiver : public DisplayEventDispatcher {
public:
    NativeDisplayEventReceiver(JNIEnv* env,
            jobject receiverWeak, const sp<MessageQueue>& messageQueue, jint vsyncSource,
            jint configChanged);

    void dispose();

protected:
    virtual ~NativeDisplayEventReceiver();

private:
    jobject mReceiverWeakGlobal;
    sp<MessageQueue> mMessageQueue;

    void dispatchVsync(nsecs_t timestamp, PhysicalDisplayId displayId, uint32_t count) override;
    void dispatchHotplug(nsecs_t timestamp, PhysicalDisplayId displayId, bool connected) override;
    void dispatchConfigChanged(nsecs_t timestamp, PhysicalDisplayId displayId,
                               int32_t configId, nsecs_t vsyncPeriod) override;
};
```

scheduleVsync

```c++
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp#scheduleVsync
status_t DisplayEventDispatcher::scheduleVsync() {
    //这个的mWaitingForVsync表示同一时间只能处理一次Vysnc信号
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);
        nsecs_t vsyncTimestamp;
        PhysicalDisplayId vsyncDisplayId;
        uint32_t vsyncCount;
        //这个processPendingEvents函数会在后面解释，是双向通信管道的socketpair的receive过程，用于获取vysnc信号
        //这里的作用实际上是规定时间内，发送太多次vynsc，没来的及处理的error日志信息
        //processPendingEvents，处理Vysnc信号事件的[2.3.2]，主要是处理延时信号
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "", this,
                  ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }
        //[2.4.16]
        status_t status = mReceiver.requestNextVsync();

        mWaitingForVsync = true;
    }
    return OK;
}
```

### 2.4.16DisplayEventReceiver.requestNextVsync

该方法的作用请求下一次Vsync信息处理

```c++
//frameworks/native/libs/gui/DisplayEventReceiver.cpp#requestNextVsync
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != nullptr) {
        //[2.4.17]
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}
```

### 2.4.17BpDisplayEventConnection.requestNextVsync

由于对应的IDisplayEventConnection的定义是requestNextVsync是纯虚函数，所以最终调用是在BpDisplayEventConnection中。显然这里面有一个binder过程，这里面的SafeBpInterface是IInterface这个基类的封装类，对其做了一次封装用于这里的场景

```c++
//frameworks/native/libs/gui/IDisplayEventConnection.cpp#requestNextVsync
class BpDisplayEventConnection : public SafeBpInterface<IDisplayEventConnection> {
    void requestNextVsync() override {
        //[2.4.18]
        callRemoteAsync<decltype(&IDisplayEventConnection::requestNextVsync)>(
            Tag::REQUEST_NEXT_VSYNC);
    }
};
```

IDisplayEventConnection初始化，其中传入的impl就是对应的服务端

```c++
//frameworks/native/libs/gui/IDisplayEventConnection.cpp#BpDisplayEventConnection
class BpDisplayEventConnection : public SafeBpInterface<IDisplayEventConnection> {
public:
    explicit BpDisplayEventConnection(const sp<IBinder>& impl)
          : SafeBpInterface<IDisplayEventConnection>(impl, "BpDisplayEventConnection") {}
    ...
}
```

### 2.4.18EventThreadConnection.requestNextVsync

事实上传入的impl就是EventThreadConnection的实例

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.h
class EventThreadConnection : public BnDisplayEventConnection {
public:
    EventThreadConnection(EventThread*, ResyncCallback,
                          ISurfaceComposer::ConfigChanged configChanged);
    virtual ~EventThreadConnection();

    virtual status_t postEvent(const DisplayEventReceiver::Event& event);

    status_t stealReceiveChannel(gui::BitTube* outChannel) override;
    status_t setVsyncRate(uint32_t rate) override;
    void requestNextVsync() override; // asynchronous
    void requestLatestConfig() override; // asynchronous

    const ResyncCallback resyncCallback;
    VSyncRequest vsyncRequest = VSyncRequest::None;
    ISurfaceComposer::ConfigChanged mConfigChanged =
            ISurfaceComposer::ConfigChanged::eConfigChangedSuppress;
    bool mForcedConfigChangeDispatch = false;

private:
    virtual void onFirstRef();
    EventThread* const mEventThread;
    gui::BitTube mChannel;
};
```

通过binder，最终调用到服务端实现。可以看到requestNextVsync的最后实际上就是用于一个信号的唤醒，notify_all

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#requestNextVsync
void EventThreadConnection::requestNextVsync() {
    ATRACE_NAME("requestNextVsync");
    mEventThread->requestNextVsync(this);
}

void EventThread::requestNextVsync(const sp<EventThreadConnection>& connection) {
    if (connection->resyncCallback) {
        connection->resyncCallback();
    }

    std::lock_guard<std::mutex> lock(mMutex);

    if (connection->vsyncRequest == VSyncRequest::None) {
        connection->vsyncRequest = VSyncRequest::Single;
        //[2.4.19]
        mCondition.notify_all();
    }
}
```

### 2.4.19EventThread::threadMain#wait_for

notify_all这里是用于唤醒，有唤醒那么等待。这里有threadMain的函数，是由SurfaceFlinger初始化的时候启动的线程函数。这个函数主要是用于收集和处理唤醒事件

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#EventThread#requestNextVsync
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;

    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;

        // Determine next event to dispatch.
        if (!mPendingEvents.empty()) {
            //std::deque<DisplayEventReceiver::Event> mPendingEvents GUARDED_BY(mMutex);
            //GUARDED_BY 是线程的安全注解
            //Event队列不为空，取出队列中的元素，这里指的是之前唤醒的Vysnc事件
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
			
            switch (event->header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                    if (event->hotplug.connected && !mVSyncState) {
                        mVSyncState.emplace(event->header.displayId);
                    } else if (!event->hotplug.connected && mVSyncState &&
                               mVSyncState->displayId == event->header.displayId) {
                        mVSyncState.reset();
                    }
                    break;
				//回调mInterceptVSyncsCallback处理
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    if (mInterceptVSyncsCallback) {
                        mInterceptVSyncsCallback(event->header.timestamp);
                    }
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_CONFIG_CHANGED:
                    mLastConfigChangeEvent = *event;
                    break;
            }
        }

        bool vsyncRequested = false;

        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) {
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;

                if (event && shouldConsumeEvent(*event, connection)) {
                    if (event->header.type == DisplayEventReceiver::DISPLAY_EVENT_CONFIG_CHANGED &&
                        connection->mForcedConfigChangeDispatch) {
                        connection->mForcedConfigChangeDispatch = false;
                    }
                    consumers.push_back(connection);
                }

                ++it;
            } else {
                it = mDisplayEventConnections.erase(it);
            }
        }

        //处理分发的事件
        if (!consumers.empty()) {
            //[2.4.20]
            dispatchEvent(*event, consumers);
            consumers.clear();
        }
        //开始下一帧的申请
        State nextState;
        if (mVSyncState && vsyncRequested) {
            nextState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            ALOGW_IF(!mVSyncState, "Ignoring VSYNC request while display is disconnected");
            nextState = State::Idle;
        }

        if (mState != nextState) {
            if (mState == State::VSync) {
                mVSyncSource->setVSyncEnabled(false);
            } else if (nextState == State::VSync) {
                mVSyncSource->setVSyncEnabled(true);
            }

            mState = nextState;
        }

        if (event) {
            continue;
        }

        //继续等待，直到开始有唤醒
        if (mState == State::Idle) {
            mCondition.wait(lock);
        } else {
            const std::chrono::nanoseconds timeout =
                    mState == State::SyntheticVSync ? 16ms : 1000ms;
            //上面传递过来的notify_all用于这里的唤醒
            if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
                if (mState == State::VSync) {
                    ...
                    auto pos = debugInfo.find('\n');
                    while (pos != std::string::npos) {
                        ALOGW("%s", debugInfo.substr(0, pos).c_str());
                        debugInfo = debugInfo.substr(pos + 1);
                        pos = debugInfo.find('\n');
                    }
                }

                const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
                const auto expectedVSyncTime = now + timeout.count();
                //唤醒之后，首先把唤醒的这个信号的信息，存储在Event队列中
                mPendingEvents.push_back(makeVSync(mVSyncState->displayId, now,
                                                   ++mVSyncState->count, expectedVSyncTime));
            }
        }
    }
}
```

### 2.4.20dispatchEvent

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#EventThread#dispatchEvent
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        //开始处理这里的事件[2.4.21]
        switch (consumer->postEvent(event)) {
            case NO_ERROR:
                break;

            case -EAGAIN:
                // TODO: Try again if pipe is full.
                ALOGW("Failed dispatching %s for %s", toString(event).c_str(),
                      toString(*consumer).c_str());
                break;

            default:
                // Treat EPIPE and other errors as fatal.
                removeDisplayEventConnectionLocked(consumer);
        }
    }
}
```

### 2.4.21postEvent

```c++
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp#EventThreadConnection#postEvent
status_t EventThreadConnection::postEvent(const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

postEvent，可以发现实际上最终调用到的就是sendObjects

```c++
//frameworks/native/libs/gui/DisplayEventReceiver.cpp#sendEvents
ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    //[2.4.22]
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

### 2.4.22sendObjects

这里的BitTube实际上就是封装的socketpair

```c++
//frameworks/native/libs/gui/include/private/gui/BitTube.h#sendObjects
template <typename T>
static ssize_t sendObjects(BitTube* tube, T const* events, size_t count) {
    //[2.4.23]
    return sendObjects(tube, events, count, sizeof(T));
}
```

### 2.4.23sendObjects

走到这里，实际上就已经不难了，原理就是socket通信

```c++
//frameworks/native/libs/gui/BitTube.cpp#sendObjects
ssize_t BitTube::sendObjects(BitTube* tube, void const* events, size_t count, size_t objSize) {
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count * objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

### 2.4.34下一帧的Vsync请求流程图如下所示

java流程

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_2.png" thumbnail="/choreographer/choreographyer_2.png" title="">}}

c++流程

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_3.png" thumbnail="/choreographer/choreographyer_3.png" title="">}}

总的流程

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_4.png" thumbnail="/choreographer/choreographyer_4.png" title="">}}



### 2.4.25关于Choreographer的流程

结合2.3的Choreographer处理上一帧流程和2.4的下一帧的Vsync请求流程，具体可以看到下图

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_9.png" thumbnail="/choreographer/choreographyer_9.png" title="">}}

承接2.3.9.5所示的run方法，不同的action（action为Runnable或者 FrameCallback两者的子类）和callbackType（CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL和CALLBACK_COMMIT），最终回调用不同的方法。

#### 2.4.25.1FrameCallback.doFrame

 Choreographer的FrameCallback用的callbackType只能为CALLBACK_ANIMATION，这里会会调到FrameCallback子类的doFrame方法

```java
//frameworks/base/core/java/android/view/Choreographer.javapostFrameCallbackDelayed
public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
                                callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_10.png" thumbnail="/choreographer/choreographyer_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_11.png" thumbnail="/choreographer/choreographyer_11.png" title="">}}

表2.1postFrameCallbackDelayed子类

| 子类名                            | 回调名                                  |
| --------------------------------- | --------------------------------------- |
| AnimationHandler.java             | mFrameCallback                          |
| ViewRootImpl.java                 | mRenderProfiler                         |
| TextView.java                     | mTickCallback                           |
|                                   | mStartCallback                          |
| SfVsyncFrameCallbackProvider.java | callback                                |
| BackdropFrameRenderer.java        | BackdropFrameRenderer::this             |
| TiledImageView.java               | mFrameCallback                          |
| SurfaceAnimationRunner.java       | SurfaceAnimationRunner::startAnimations |
| WindowAnimator.java               | mAnimationFrameCallback                 |
| WindowTracing.java                | mFrameCallback                          |
| BubbleStackView.java              | new Choreographer.FrameCallback()       |
| FrameProtoTracer.java             | FrameProtoTracer::this                  |

举例说明

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_12.png" thumbnail="/choreographer/choreographyer_12.png" title="">}}

#### 2.5.25.2Runnable.run

这里的5种类型的run，比如这就是进入了事件分发的机制。事实上，callbackType为CALLBACK_ANIMATION更多一点，这里会会调到Runnable子类的run方法

1）callbackType为**CALLBACK_INPUT**时

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java#doConsumeBatchedInput
final class ConsumeBatchedInputRunnable implements Runnable {
    @Override
    public void run() {
        mConsumeBatchedInputScheduled = false;
        if (doConsumeBatchedInput(mChoreographer.getFrameTimeNanos())) {
            //我们想要继续并安排在下一帧消费延时输入事件。
            //如果不先消费这一批次的输入事件，我们会等到有更多的输入事件挂起，并且可能会因过程中发生的其他事情而饿死
            scheduleConsumeBatchedInput();
        }
    }
}

boolean doConsumeBatchedInput(long frameTimeNanos) {
    final boolean consumedBatches;
    if (mInputEventReceiver != null) {
        consumedBatches = mInputEventReceiver.consumeBatchedInputEvents(frameTimeNanos);
    } else {
        consumedBatches = false;
    }
    doProcessInputEvents();
    return consumedBatches;
}

void scheduleConsumeBatchedInput() {
    if (!mConsumeBatchedInputScheduled && !mConsumeBatchedInputImmediatelyScheduled) {
        mConsumeBatchedInputScheduled = true;
        mChoreographer.postCallback(Choreographer.CALLBACK_INPUT,
                                    mConsumedBatchedInputRunnable, null);
    }
}
```

input最终会调用到DecorView 的 dispatchTouchEvent，这就到了我们熟悉的 Input 事件分发，这里不展开描述事件分发机制。

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_13.png" thumbnail="/choreographer/choreographyer_13.png" title="">}}

2）callbackType为**CALLBACK_ANIMATION**时

```java
//frameworks/base/core/java/android/view/View.java
//由于postOnAnimation的参数是Runnable的动作，子类或者是匿名内部类来进行回调就可以收到回调的Vysnc信号事件
public void postOnAnimation(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        //调用postCallback的api，使用的是runnable形式
        attachInfo.mViewRootImpl.mChoreographer.postCallback(
            Choreographer.CALLBACK_ANIMATION, action, null);
    } else {
        // Postpone the runnable until we know
        // on which thread it needs to run.
        getRunQueue().post(action);
    }
}
```

由于上面的action，是由其他类去注入的，因此会回调用到 View.postOnAnimation的操作，如下表所示

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_6.png" thumbnail="/choreographer/choreographyer_6.png" title="">}}

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_7.png" thumbnail="/choreographer/choreographyer_7.png" title="">}}

举例如下，RecyclerView.java

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_14.png" thumbnail="/choreographer/choreographyer_14.png" title="">}}

3）callbackType为**CALLBACK_TRAVERSAL**时

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java#scheduleTraversals
//这个scheduleTraversals方法设置Runnable的回调
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

//Vsync信号分发事件回调到这里的run方法
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
//这里做真正的measure、layout和draw操作，这里就是具体的绘制三部曲流程了
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
        //这里做真正的measure、layout和draw操作，这里就是具体的绘制三部曲流程了
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

举例说明

{{< image classes="fancybox center fig-100" src="/choreographer/choreographyer_15.png" thumbnail="/choreographer/choreographyer_15.png" title="">}}

## 3.5 Choreographer应用

许多性能监控的手段都是利用 Choreographer 来做的，除了自带的掉帧计算，Choreographer 提供的 FrameCallback 和 FrameInfo 都给 App 暴露了接口，让 App 开发者可以通过这些方法监控自身 App 的性能，其中常用的方法如下：

1. 利用 FrameCallback 的 doFrame 回调
2. 利用 FrameInfo 进行监控
   1. 使用 ：adb shell dumpsys gfxinfo framestats
   2. 示例 ：adb shell dumpsys gfxinfo <PACKAGE> framestats
3. 利用 SurfaceFlinger 进行监控
   1. 使用 ：adb shell dumpsys SurfaceFlinger –latency
   2. 示例 ：adb shell dumpsys SurfaceFlinger –latency <PACKAGE>/<ACTIVITY>



### 3.5.1FrameCallback 接口

利用 FrameCallback 的 doFrame 回调用于监控应用的帧率，类似开源项目TinyDancer 就是使用了这个方法来计算 FPS (https://github.com/friendlyrobotnyc/TinyDancer)

```java
//frameworks/base/core/java/android/view/Choreographer.java#FrameCallback
public interface FrameCallback {
    public void doFrame(long frameTimeNanos);
}
```

开始自定义处理帧逻辑

```java
Choreographer.getInstance().postFrameCallback(new Choreographer.postFrameCallback(){
    @override
    public void doFrame(long now)
    {
        //计算上一次now和这一次now的时间间隔
        mGap = now - mNext;
        mNext = now;
    }
});

public void postFrameCallback(FrameCallback callback, long delayMillis) {
    ......
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

### 3.5.2FrameInfo 进行监控

截取部分数据

```txt
picasso:/ $ dumpsys gfxinfo com.miui.home framestats
Applications Graphics Acceleration Info:
Uptime: 18524925 Realtime: 30993029

** Graphics info for pid 24987 [com.miui.home] **

Stats since: 15566429111717ns
Total frames rendered: 976
Janky frames: 179 (18.34%)
50th percentile: 5ms
90th percentile: 11ms
95th percentile: 19ms
99th percentile: 150ms
Number Missed Vsync: 31
Number High input latency: 710
Number Slow UI thread: 49
Number Slow bitmap uploads: 19
Number Slow issue draw commands: 45
Number Frame deadline missed: 77
HISTOGRAM: 5ms=667 6ms=65 7ms=49 8ms=35 9ms=30 10ms=30 11ms=13 12ms=8 13ms=10 14ms=2 15ms=6 16ms=4 17ms=2 18ms=4 19ms=3 20ms=1 21ms=2 22ms=1 23ms=4 24ms=1 25ms=2 26ms=0 27ms=0 28ms=1 29ms=1 30ms=0 31ms=0 32ms=2 34ms=0 36ms=0 38ms=0 40ms=3 42ms=2 44ms=2 46ms=1 48ms=0 53ms=4 57ms=0 61ms=0 65ms=1 69ms=0 73ms=1 77ms=0 81ms=2 85ms=0 89ms=0 93ms=2 97ms=1 101ms=3 105ms=0 109ms=0 113ms=0 117ms=0 121ms=0 125ms=0 129ms=0 133ms=1 150ms=3 200ms=1 250ms=2 300ms=2 350ms=1 400ms=0 450ms=0 500ms=0 550ms=0 600ms=0 650ms=0 700ms=0 750ms=0 800ms=0 850ms=0 900ms=0 950ms=0 1000ms=1 1050ms=0 1100ms=0 1150ms=0 1200ms=0 1250ms=0 1300ms=0 1350ms=0 1400ms=0 1450ms=0 1500ms=0 1550ms=0 1600ms=0 1650ms=0 1700ms=0 1750ms=0 1800ms=0 1850ms=0 1900ms=0 1950ms=0 2000ms=0 2050ms=0 2100ms=0 2150ms=0 2200ms=0 2250ms=0 2300ms=0 2350ms=0 2400ms=0 2450ms=0 2500ms=0 2550ms=0 2600ms=0 2650ms=0 2700ms=0 2750ms=0 2800ms=0 2850ms=0 2900ms=0 2950ms=0 3000ms=0 3050ms=0 3100ms=0 3150ms=0 3200ms=0 3250ms=0 3300ms=0 3350ms=0 3400ms=0 3450ms=0 3500ms=0 3550ms=0 3600ms=0 3650ms=0 3700ms=0 3750ms=0 3800ms=0 3850ms=0 3900ms=0 3950ms=0 4000ms=0 4050ms=0 4100ms=0 4150ms=0 4200ms=0 4250ms=0 4300ms=0 4350ms=0 4400ms=0 4450ms=0 4500ms=0 4550ms=0 4600ms=0 4650ms=0 4700ms=0 4750ms=0 4800ms=0 4850ms=0 4900ms=0 4950ms=0
```

具体可以点击[这里](https://www.cnblogs.com/zhengna/p/10032078.html)

### 3.5.3SurfaceFlinger 进行监控

截取部分数据

```txt
picasso:/ $ dumpsys SurfaceFlinger -latency com.miui.home/com.miui.home.launcher.Launcher
Build configuration: [sf PRESENT_TIME_OFFSET=0 FORCE_HWC_FOR_RBG_TO_YUV=1 MAX_VIRT_DISPLAY_DIM=4096 RUNNING_WITHOUT_SYNC_FRAMEWORK=0 NUM_FRAMEBUFFER_SURFACE_BUFFERS=3] [libui] [libgui]

Display identification data:
Display 19260668995500161 (HWC display 0): port=129 pnpId=QCM displayName="xiaomi 37 02 "

Wide-Color information:
Device has wide color built-in display: 1
Device uses color management: 1
DisplayColorSetting: Unknown 258
Display 19260668995500161 color modes:
    ColorMode::NATIVE (0)
    ColorMode::SRGB (7)
    ColorMode::DISPLAY_P3 (9)
    Current color mode: ColorMode::SRGB (7)

Sync configuration: [using: EGL_ANDROID_native_fence_sync EGL_KHR_wait_sync]

VSYNC configuration:
         app phase:   1000000 ns                 SF phase:   1000000 ns
   early app phase:   1000000 ns           early SF phase:   1000000 ns
GL early app phase:   1000000 ns        GL early SF phase:   1000000 ns
    present offset:         0 ns             VSYNC period:  20000000 ns

Scheduler enabled.+  Smart 90 for video detection: off
```

具体可以点击[这里](https://zhuanlan.zhihu.com/p/435304317)

# 3总结

**Choreographer**：编舞者，指对CPU/GPU绘制的指导，收到VSync信号才开始绘制，保证绘制拥有完整的16.7ms。通常应用层不会直接使用Choreographer，但是存在**Vsync + TripleBuffer + Choreographer** 的机制，其就可以提供一个稳定的帧率输出机制，让软件层和硬件层可以以共同的频率一起工作。主要是作为一种承上启下的作用，实质就是通过上层封装，间接去操作socketpair双向通信管道。



承上

负责接收和处理 **App** 的各种更新消息和回调，等到 Vsync 事件回调(通过 FrameDisplayEventReceiver.onVsync )统一处理。比如集中五种处理 Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 measure、layout、draw 等操作) 等，判断卡顿掉帧情况，记录 CallBack 耗时等

启下

负责请求和申请Vsync的下一帧信号。请求 Vsync(FrameDisplayEventReceiver.scheduleVsync)。

- 尽量避免在执行动画渲染的前后在主线程放入耗时操作，否则会造成卡顿感，影响用户体验；
- 可通过Choreographer.getInstance().postFrameCallback()来监听帧率情况；
- 每调用一次scheduleFrameLocked()，则mFrameScheduled=true，可进入doFrame()方法体内部，执行完doFrame()并设置mFrameScheduled=false；
- doCallbacks回调方法有4个类别：INPUT（输入事件），ANIMATION（动画），TRAVERSAL（窗口刷新），COMMIT（完成后的提交操作）。

# 4补充

1.关于mFrameScheduled的bool值

```java
//doFrame是处理上一帧的数据，每隔一个Vsync信号，即每隔16.7ms，只处理一次，当然在卡顿的时候并不能保证16.7ms一个信号周期
//frameworks/base/core/java/android/view/Choreographer.java#doFrame
void doFrame(long frameTimeNanos, int frame) {
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }
        ...
        mFrameScheduled = false;
    }
}

//这里的scheduleFrameLocked，使得每次申请下一帧都保持最新的一次事件
//frameworks/base/core/java/android/view/Choreographer.java#scheduleFrameLocked
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        ...
    }
}
```

2.这里面涉及到的部分时间梳理

会在Vsync信号篇章中具体展开。

# 5源码下载

源码下载，点击[这里](https://github.com/YangYang48/project/tree/master/choreographer/)



# 参考

[[1] Gracker, Android 基于 Choreographer 的渲染机制详解, 2019.](https://www.androidperformance.com/2019/10/22/Android-Choreographer/#/APM-%E4%B8%8E-Choreographer)

[[2] 袁辉辉, Choreographer原理, 2017.](http://gityuan.com/2017/02/25/choreographer/)

[[3] lbtrace, Android应用与SurfaceFlinger建立连接的过程, 2019.](https://www.jianshu.com/p/304f56f5d486)

[[4] 苍耳叔叔, Android-Choreographer原理, 2020.](https://ljd1996.github.io/2020/09/07/Android-Choreographer%E5%8E%9F%E7%90%86/)

[[5] bug樱樱, 一看就会！Android屏幕刷新机制最新讲解—VSync、Choreographer 全面理解！, 2020.](https://www.jianshu.com/p/0d8fbee0869d)

[[6] Drummor, Android 怎么就不卡了呢之Choreographer, 2019.](https://juejin.cn/post/6844903818044375053)

[[7] 丹枫无迹, Choreographer全解析, 2021.](http://www.360doc.com/content/21/1214/20/65839755_1008716414.shtml)

[[8] DeltaTech, Android Choreographer 源码分析, 2016.](https://www.jianshu.com/p/996bca12eb1d)

[[9] 静默加载, ViewRootImpl的独白，我不是一个View(布局篇), 2017.](https://blog.csdn.net/stven_king/article/details/78775166)

[[10] 大苏, Android 屏幕刷新机制, 2018.](https://www.cnblogs.com/dasusu/p/8311324.html)

[[11] 梅花十三儿, [Android禅修之路] 解读Vsync(一), 2022.](https://blog.csdn.net/Android062005/article/details/123090139)

[[12] Innerpeace_yu, C++中explicit的用法, 2018.](https://blog.csdn.net/yu132563/article/details/80103693)

[[13] zhengna, Android UI性能测试——使用 Gfxinfo 衡量性能, 2019.](https://www.cnblogs.com/zhengna/p/10032078.html)

[[14] 麻辣小龙虾, 安卓帧率计算方案和背后原理剖析, 2021.](https://zhuanlan.zhihu.com/p/435304317)

[[15] DJLZPP, AndroidQ UI刷新机制详解, 2020.](https://blog.csdn.net/qq_34211365/article/details/105093648)

[[16] DJLZPP, AndroidQ 应用层Vsync信号的注册与接收（下）, 2020.](https://blog.csdn.net/qq_34211365/article/details/105155801)