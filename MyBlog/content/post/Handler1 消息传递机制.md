---
title: "Handler1-消息传递机制"
date: 2021-08-15
thumbnailImagePosition: left
thumbnailImage: handler/handler1_thumb.jpg
coverImage: handler/handler1_cover_2.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- Handler
- 2021
- August
tags:
- Android
- 源码
- framework
showSocial: false
---
Handler是一个优秀的内存共享方案。其内存管理和设计思路相当完整。
通过Handler来通知UI组件更新或者是处理对应消息。那么Handler消息机制是什么？

<!--more-->
# Handler1 消息传递机制

## 开篇问题

1.一个线程有几个handler

2.一个线程有几个looper，如何保证

3.Handler会出现内存泄漏问题吗，怎么解决内存泄漏

4.子线程总如何使用Handler

{{< image classes="fancybox center fig-100" src="/handler/handler1_1.png" thumbnail="/handler/handler1_1.png" title="App Launcher和System_Server交互">}}

android 和linux 交互会使用socket

可以说Handler、Binder、Socket是整个Android系统的灵魂。从本文开始从源码角度分析Handler


## handler使用

### 1.handler初始化

Handler构造，当前api30之后，废弃了2个，隐藏3个，实际我们可用的只有两个构造，**必须指定当前是哪个Looper**

```java
@Deprecated
public Handler() {
    this(null, false);
}

@Deprecated
public Handler(@Nullable Callback callback) {
    this(callback, false);
}
//可用的构造方法1
public Handler(@NonNull Looper looper) {
    this(looper, null, false);
}
//可用的构造方法2
public Handler(@NonNull Looper looper, @Nullable Callback callback) {
    this(looper, callback, false);
}

/*
* @hide
*/
@UnsupportedAppUsage
public Handler(boolean async) {
    this(null, async);
}

/*
* @hide
*/
public Handler(@Nullable Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
            (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                  klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
            + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

/*
* @hide
*/
@UnsupportedAppUsage
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

举例说明

```java
//可用的构造方法1
private final Handler mUIHandler = new Handler(Looper.getMainLooper()){
    @Override
    public void handleMessage(Message msg) {
        Log.d(TAG, "->>>mUIHandler);
    }
};
```

```java
//可用的构造方法2
private final Handler childHandler = new Handler(handlerThread.getLooper(), new Handler.Callback() {
    @Override
    public boolean handleMessage(@NonNull Message msg) {
        Log.d(TAG, "->>>ChildCallback|handleMessage");
        return true;
    }
});
```

其中，关于Looper.getMainLooper()和Looper.myloop()

> ### Looper.myLooper()
>
> 获取当前进程的looper对象
>
> ### Looper.getMainLooper()
>
> 获取主线程的Looper对象
>
> 通过Handler构造函数可以看出:
>  一个 Handler 中只能有一个 Looper。而一个 Looper 则可以对应多个 Handler，只要把 Looper 往 Handler 的构造方法里扔扔扔就好了。



### 2.handler发送消息

{{< image classes="fancybox center fig-100" src="/handler/handler1_2.png" thumbnail="/handler/handler1_2.png" title="Handler发送消息的方法罗列">}}

所有发送消息的方式都在上面罗列，一般常用的为直接或间接调用sendMessageDelayed方法，而其调用的sendMessageAtTime会内部判断消息队列是否为空，消息队列为空就不做处理。

并且这个msg中还会存在时间when，target还有setAsynchronous

1）其中Message的target属性在enqueMessage的时候会将msg.target = this，**即Message持有Handler对象**

2）其中在enqueMessage中的判断mAsynchronous，通常情况下都是同步消息，**这个判断为false**



### 3.handler处理消息

message回到handler中的流程

Looper.java//loop函数

msg.target.dispatchMessage(msg)--->>

Handler.java//dispatchMessage函数

分三种情况

```java
/**
 * Handle system messages here.
 */
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);//第一种情况
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;//第二种情况
            }
        }
        handleMessage(msg);//第三种情况
    }
}
```

#### 1.收发消息一体

Message通过post(Runnable)等方法进行发送走handleCallback(msg);

```java
public class MainActivity extends AppCompatActivity {
    private TextView msgTv;
    private Handler mHandler = new Handler(Looper.getMainLooper());

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        msgTv = findViewById(R.id.tv_msg);

          //消息收发一体
        new Thread(new Runnable() {
            @Override public void run() {
                String info = "第一种方式";
                mHandler.post(new Runnable() {
                    @Override public void run() {
                        msgTv.setText(info);
                    }
                });
            }
        }).start();
    }
}
```

#### 2.收发分开，实现Callback接口

在Handler构造的时候传入Handler内部抽象类，优先处理。mCallback.handleMessage(msg)这个函数会回调到构造里面Callback，最后在结尾是返回一个boolean类性值，一般为true，即外部不需要在handleMessage一次，立刻结束本次的message传递。

```java
public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    boolean handleMessage(@NonNull Message msg);
}
```

```java
public class MainActivity extends AppCompatActivity {
    private TextView msgTv;
    private Handler mHandler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
        //接收消息,刷新UI
        @Override public boolean handleMessage(@NonNull Message msg) {
            if (msg.what == 1) {
                msgTv.setText(msg.obj.toString());
            }
            //false 重写Handler类的handleMessage会被调用,  true 不会被调用
            return false;
        }
    });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        msgTv = findViewById(R.id.tv_msg);

        //发送消息
        new Thread(new Runnable() {
            @Override public void run() {
                Message message = Message.obtain();
                message.what = 1;
                message.obj = "第二种方式 --- 2"; 
                mHandler.sendMessage(message);
            }
        }).start();
    }
}
```

#### 3.重写Handler类

调用handler重新handleMessage方法，走handleMessage(msg);

```java
public class MainActivity extends AppCompatActivity {
    private TextView msgTv;
    private Handler mHandler = new Handler(Looper.getMainLooper()) {
        //接收消息,刷新UI
        @Override public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            if (msg.what == 1) {
                msgTv.setText(msg.obj.toString());
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        msgTv = findViewById(R.id.tv_msg);

        //发送消息
        new Thread(new Runnable() {
            @Override public void run() {
                Message message = Message.obtain();
                message.what = 1;
                message.obj = "第三种方式 --- 3";
                mHandler.sendMessage(message);
            }
        }).start();
    }
}
```

## Handler消息传递

### 优先级队列

根据上述可以看到Handler发送消息，最终会调用enqueMessage

```java
//handler#enqueMessage
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
                               long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

最终返回的是一个布尔值，queue.enqueMessage，其中queue是MessageQueue类型的

```java
//MessageQueue#enqueueMessage
boolean enqueueMessage(Message msg, long when) {
    ...
    synchronized (this) { 
        msg.when = when;
        //这里的mMessages是一个Message类型的链表（Message中有next属性值）
        Message p = mMessages;
        boolean needWake;
        //三个判断指的是这个链表没有Message，传入的当前p为当前第一个Message
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;//mBlocked通常为true的时候是无idle handlers，纯等待
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            //这个needWake中的后两个判断，第一个是用来判断当前MessageQueue是否开启同步屏障且传入的Message是异步的
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            //循环开始在适当的位置把传入的msg插入到mMessageQueue中
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            //用于唤醒next方法中的nativePollOnce
            nativeWake(mPtr);
        }
        ...
    }
    return true;
}
```

拆解上述方法，核心的就是其中两个判断if (p == null || when == 0 || when < p.when)，正好对应两个形式的链表的插入，从而得出正是一个按照时间顺序排列的链表。

1）如果判断if (p == null || when == 0 || when < p.when)成立，则为下图

{{< image classes="fancybox center fig-100" src="/handler/handler1_3.png" thumbnail="/handler/handler1_3.png" title="">}}

2）如果判断if (p == null || when == 0 || when < p.when)不成立，当前msg传入的when<p.when，则为下图

{{< image classes="fancybox center fig-100" src="/handler/handler1_4.png" thumbnail="/handler/handler1_4.png" title="">}}

3）如果判断if (p == null || when == 0 || when < p.when)不成立，当前p=null，则为下图

{{< image classes="fancybox center fig-100" src="/handler/handler1_5.png" thumbnail="/handler/handler1_5.png" title="">}}

那么既然有插入链表的操作，就有从MessageQueue中取出的操作，再来看next方法

```java
//MessageQueue#enqueueMessage
Message next() {
    ...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            //当需要阻塞时，防止进程持有对象的时间超过需要的时间，做一个保护作用
            Binder.flushPendingCommands();
        }
        //这个是用来阻塞的，两种方式解除
        //1.nativeWake的时候继续执行下去。即直到euqueMessage中插入了一个比nextPollTimeoutMillis小的时间的msg
        //2.nextPollTimeoutMillis==now，时间消耗完毕，开始取消阻塞
        //3.退出消息队列所在的循环Looper，即mQuitting=true
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    //这个判断是用于同步屏障中异步消息的判断，这里暂时不考虑
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            //没有IdleHandler的时候，直接continue
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }
        }
        ...
        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

拆解可知，这个链表是按照msg插入when的时间的排列的链表，符合先进先出，即为**优先级队列**。

1）当前的now<msg.when，开始阻塞线程

这里边用到了nativePollOnce，用于阻塞线程，三种方式可以取消阻塞

> 1.nativeWake的时候继续执行下去。即直到euqueMessage中插入了一个比nextPollTimeoutMillis小的时间的msg
> 2..nextPollTimeoutMillis==now，时间消耗完毕，开始取消阻塞
>
> 3.退出消息队列所在的循环Looper

2）开始获取Message，返回圈红的Message

{{< image classes="fancybox center fig-100" src="/handler/handler1_6.png" thumbnail="/handler/handler1_6.png" title="获取Message">}}

在MessageQueue里面Handler中的三个native方法，主要含义具体分析详见[Ahab](https://mp.weixin.qq.com/s/ClTE15s9qUaNsInIIwX57w)的文章

```java
//Handler中的三个native方法
private native static long nativeInit(); //返回 ptr
private native void nativePollOnce(long ptr, int timeoutMillis); //阻塞
private native static void nativeWake(long ptr); //唤醒
```



### Looper

根据上述可知Handler会通过enqueMessage来将消息根据延时的时间排列成优先级队列先进先出，那么谁去执行next让其每次一个个有序的将MessageQueue中的Message取出来呢？这时候需要用到Looper。

#### 获取next方法

听名字就知道是一个循环，主要是看里面loop方法

```java
//Looper.java#loop
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;//mQueue = new MessageQueue(quitAllowed);默认是无法退出的

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    //重置当前线程上传入的IPC的身份,确保此线程的标识是本地进程的标识
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        // Make sure the observer won't change while processing a transaction.
        //observer是用于Handler分发操作的监听，监听产生问题的trace
        //补充说明：这里面的sObserver是在LooperStateService.java里面设置下来的
        //Looper.setObserver(enable ? mStates : null);
        //其中mStates定义 LooperState mStates implement Looper.Observer
        //这里的enable是通过系统属性来区分debug.sys.Looper_state_enabled
        final Observer observer = sObserver;

        Object token = null;
        if (observer != null) {
            token = observer.messageDispatchStarting();
        }
        long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
        try {
            //将获取到的msg分发到dispatchMessage
            msg.target.dispatchMessage(msg);
            if (observer != null) {
                observer.messageDispatched(token, msg);
            }
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } catch (Exception exception) {
            if (observer != null) {
                observer.dispatchingThrewException(token, msg, exception);
            }
            throw exception;
        } finally {
            ThreadLocalWorkSource.restore(origWorkSource);
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        ...
        //msg资源池回收机制    
        msg.recycleUnchecked();
    }
}
```

拆解loop方法，其实就是做了几件事

1）重复循环获取next方法，因为next方法存在阻塞，所以queue.next()也会阻塞，且唯一退出这个方法的是就是获取一个null的Messgae

2）将获取到的msg分发到dispatchMessage，用这个方法来对消息进行处理，msg.target持有了Handler对象，那么依然会回到Handler中去使用。通过Looper使得整个消息队列形成了闭环。

3）处理完消息之后，通过msg.recycleUnchecked()，**将消息回收到消息池里**



##### 这里的消息放到消息池，是运用了设计模式的享元模式

具体看一下Message的构成

```java
//Message.java的属性
public static final Object sPoolSync = new Object();//用于消息池的上锁功能
private static Message sPool;
private static int sPoolSize = 0;//默认消息池的大小

private static final int MAX_POOL_SIZE = 50;//消息池最大容量50

private static boolean gCheckRecycle = true;//默认queueMessage的时候mQuitting=true的时候会回收
```

显然Message属性会持有一个next，spool是消息队列的头，属于链表

i) obtain

```java
//Message.java#obtain获取方法
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

方法拆解如下，类似上述的消息机制：

{{< image classes="fancybox center fig-100" src="/handler/handler1_7.png" thumbnail="/handler/handler1_7.png" title="消息池中消息的获取">}}

一般调用方式为如下所示，基本囊括了平时使用的发送消息的方法，一般我们自己使用的时候，也要多使用obtainMessage方法

```java
@NonNull
public final Message obtainMessage()
{
    return Message.obtain(this);
}
@NonNull
public final Message obtainMessage(int what)
{
    return Message.obtain(this, what);
}
@NonNull
public final Message obtainMessage(int what, @Nullable Object obj) {
    return Message.obtain(this, what, obj);
}
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```



ii) recycleUnchecked

enqueMessage调用的mQuitting中的msg.recycle()，调用到这个方法，会将sPoolSize增加。这个方法用到的都是quit方法中直接或间接调用recycleUnchecked，不需要手动去调用该方法。

```java
//Message.java#recycleUnchecked回收方法
@UnsupportedAppUsage
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

方法拆解如下，这个Message加入消息池的方式是头部添加。

{{< image classes="fancybox center fig-100" src="/handler/handler1_8.png" thumbnail="/handler/handler1_8.png" title="消息池中消息的回收">}}

这里边唯一和消息队列的区别是，消息队列是先进先出，这里是先进后出，这个消息池类似压栈的过程。



#### 退出looper方法

有两种退出Looper，quitSafely和quit，最终调用到MessageQueue中的quit方法，对应true和false

```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        //ui主线程是不允许退出的
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            //只移除所有尚未执行的消息，不移除时间戳等于当前时间的消息
            removeAllFutureMessagesLocked();
        } else {
            //暴力移除所有消息
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

拆解quit方法，主要做了以下事

1）设置标记位mQuitting=true

2）移除所有消息（安全退出和强制退出）

3）唤醒在next中的nativePollOnce，在next方法中通过if (mQuitting)返回了null，回到Looper.loop方法中。Looper.loop方法中获取的msg为null，最终也退出Looper，完成整一个退出流程。 



#### 重新新建looper方法

有退出就有重新建的方法，

```java
//Looper.java#prepare
public static void prepare() {
    prepare(true);
}
```

这里的prepare执行，是可以进行退出操作，而Ui主线程中是在ActivityThread中执行prepareMainLooper，这个looper是不可以退出的，即只能在子线程中完成可以退出的looper。

举例说明

```java
//子线程总如何使用Handler的例子
private Looper mLooper;
private void initHandler() {
    new Thread(new Runnable() {
        @SuppressLint("HandlerLeak")
        @Override
        public void run() {
            //子线程开启一个Looper
            mLooper = Looper.prepare();
            handler = new Handler(Looper.myLooper(), new Handler.Callback() {
                @Override
                public boolean handleMessage(Message msg) {
                    Log.d(TAG, "->>>handlermessage");
                    if (null != mLooper)
                    {
                        //退出looper
                        mLooper.quit();
                    }
                    return true;
                }
            });
            //执行Looper.loop，即开启消息队列的无限循环模式
            Looper.loop();
        }
    }).start();
}
```



### Handler流程图

根据Looper，Handler，Message和MessageQueue可以得知Handler流程，这个流程是典型的的生产者消费者模式。

{{< image classes="fancybox center fig-100" src="/handler/handler1_9.png" thumbnail="/handler/handler1_9.png" title="Handler流程图">}}



## 问题回答

1.一个线程有几个handler

一个，ui主线程中可以有activty，service，他们都可以使用UI主线程

2.一个线程有几个looper，如何保证

一个looper。java中prepare函数如果存在threadlocal那么会抛出异常不能new looper，所以一个threadlocal对应一个looper。**这个会在下一节重点分析**。

3.Handler会出现内存泄漏问题吗，怎么解决内存泄漏

会存在，原因是由于msg.target=this，持有了handler对象。另外，Looper对象在主线程中，整个生命周期都是存在的，MessageQueue是在Looper对象中，也就是消息队列也是存在在整个主线程中，因此Handler又是和Activity存在强引用关系。即内部类持有外部类引用。this调用ondestory并不能释放，jvm认为当前this正在被handler使用。

解决：使用静态内部类弱引Activity、清空消息队列

4.子线程如何使用Handler

子线程使用handler的例子在上面已经举例说明，不再重复。



## 参考

[[1] 小呆呆666, 万字图文，带你学懂Handler和内存屏障, 2021.](https://mp.weixin.qq.com/s/tbvzs7K5OXIiAKL_DYr20A)

[[2] zcbiner, Handler机制与生产者消费者模式, 2018.](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650243268&idx=1&sn=3f3c98ff026dd670bbc1abd2ac7276f0&chksm=886371abbf14f8bdb66ba67baad9914937f8a6ad4131a6a191f1c479093486d3d16d872374da&scene=38#wechat_redirect)

[[3] 业志陈, 一文读懂 Handler 机制, 2021.](https://mp.weixin.qq.com/s/GR8oz60osaQ5rFlro_dvqg)

[[4] Ahab, 介绍一下 Android Handler 中的 epoll 机制, 2020.](https://mp.weixin.qq.com/s/ClTE15s9qUaNsInIIwX57w)


## 猜你想看

[Handler2 Thread](https://yangyang48.github.io/2021/08/handler2-thread/)

[Handler3 同步屏障](https://yangyang48.github.io/2021/08/handler3-同步屏障/)

[Handler4 HandlerThread](https://yangyang48.github.io/2021/08/handler4-handlerthread/)

[Handler5 IntentService](https://yangyang48.github.io/2021/08/handler5-intentservice/)