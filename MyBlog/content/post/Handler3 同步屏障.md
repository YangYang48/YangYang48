---
title: "Handler3 同步屏障"
date: 2021-08-15
thumbnailImagePosition: left
thumbnailImage: handler/handler3_thumb.jpg
coverImage: handler/handler3_cover.jpg
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
学习Handler之后，通常会出现同步屏障的字样。MessageQueue管理优先级队列的过程中，如果消息存在一种“紧急”消息，
需要更高的优先级处理，这个时候就需要同步屏障。

<!--more-->
# Handler3 同步屏障

## 开篇问题

1.为什么会存在同步屏障

2.如何去除同步屏障

3.同步屏障有啥用

# 什么是同步屏障

> 同步屏障就是阻碍同步消息，只让异步消息通过  

Message分为3中：普通消息（同步消息）、屏障消息（同步屏障）和异步消息。我们通常使用的都是普通消息，而屏障消息就是在消息队列中插入一个屏障，在屏障之后的所有普通消息都会被挡着，不能被处理。不过异步消息却例外，屏障不会挡住异步消息，因此可以这样认为：屏障消息就是为了确保异步消息的优先级，设置了屏障后，只能处理其后的异步消息，同步消息会被挡住，除非撤销屏障。

注：**没有同步屏障，不区分同步消息和异步消息**，都属于一般消息按时间优先级来发送

## 如何发送异步消息

```java
Message message = Message.obtain();
message.setAsynchronous(true);
handler.sendMessage(message);
```



# 同步屏障原理

## 开启同步屏障

```java
MessageQueue.java#postSyncBarrier()
/*
 * @hide
 */
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;//类似句柄，用于管理当前的屏障的唯一标识
        //此处跟Handler.enqueueMessage区别是这里没有msg.target
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        //将屏障根据时间插入链表动作
        //开启同步屏障时间T不为0，且当前的同步消息里有时间小于T，则prev也不为null
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```



## 开始对消息同步屏障

```java
MessageQueue.java#next()
Message next() {
    ...
    for (;;) {
        ...
		//nextPollTimeoutMillis
        //1.取值为-1，一直阻塞不会超时，mMessages为空，知道下一次唤醒
        //2.取值为0，不会阻塞，当前时间和下一个消息时间相同，立即处理
        //3.取值为大于0，最长阻塞为nextPollTimeoutMillis（超时机制），一般是当前时间早于下一个消息
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //同步屏障起作用
            //判断除了第一个同步屏障之后的第一个非同步消息出现，退出循环
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                //同步消息和异步消息都需要判断是否需要设置阻塞时间
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    //用于计算上面的下一循环nativePollOnce阻塞时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    //删除节点操作
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
                //用于复位，没有消息，等待下一次唤醒
                nextPollTimeoutMillis = -1;
            }
            ...
    }
}
```

## 关闭/移除同步屏障

```java
MessageQueue.java#removeSyncBarrier()
/*
 * @hide
 */
//token为上述说明用于管理当前的屏障的唯一标识
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        //找到当前屏障
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                                            + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        //从链表中移除
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        //Message的sPool对象池回收消息
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        //移除屏障，开始唤醒
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```



# 用途

Android-Choreographer工作原理

# 问题回答

1.为什么会存在同步屏障

2.如何去除同步屏障

3.同步屏障有啥用

# 参考文献

[[1] maove, Handler机制——同步屏障, 2019.](https://blog.csdn.net/start_mao/article/details/98963744)

[[2] willwaywang6, Android筑基——可视化方式理解 Handler 的同步屏障机制, 2020](https://blog.csdn.net/willway_wang/article/details/108328820?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-14.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-14.control).

[[3] 苍耳叔叔, Android-Choreographer工作原理, 2020.](https://juejin.cn/post/6894206842277199880)



# 猜你想看

[Handler1 Looper](https://yangyang48.github.io/2021/08/handler1-looper/)

[Handler2 Thread](https://yangyang48.github.io/2021/08/handler2-thread/)

[Handler4 HandlerThread](https://yangyang48.github.io/2021/08/handler4-handlerthread/)

[Handler5 IntentService](https://yangyang48.github.io/2021/08/handler5-intentservice/)
