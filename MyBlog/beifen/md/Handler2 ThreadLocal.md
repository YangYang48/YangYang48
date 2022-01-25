# Handler2 ThreadLocal

## 开篇问题

1.子线程维护的Looper，消息队列无消息的时候，处理方法是什么

2.如何保证Handler的内部线程安全

3.Handler中message的线程

4.Handler中Looper死循环会导致应用卡死吗



MessageQueue中的addIdHandler





# 问题回答

1.子线程维护的Looper，消息队列无消息的时候，处理方法是什么

2.如何保证Handler的内部线程安全

3.Handler中message的线程

4.Handler中Looper死循环会导致应用卡死吗

handler中msg。target=this 

messagequeue会持有message，msg中持有了一个handler，handler中持有了this，



怎么解决内存泄漏

软引用加static，百度



子线程创建handler如何解决？



主线程为什么可以new handler？

looper初始化prepare

其中activitythread。java中main函数Looper。prepareMainLooper（），Looper。loop（）

子线程使用handler，looper必须prepare然后再Loop，才能使用handler

> ams是以activitythread



子线程维护一个Looper，消息队列无消息的时候处理方案是什么，有什么作用

Looper。prepare，new handler，Looper.loop，Loop。quitSafely

在Looper。loop中有一个for（；；）死循环中Message msg=queue。next（），使得Looper一直阻塞不会退出

Looper里面有一个quitSafely（）调用messagequeue quit函数中有mQuitting=true，removeAllFutureMessageLocked释放内存，nativeWake会通知Message next（）函数中nativePollOnce（），让他取消阻塞执行下去。

其中，messagequeue这个类里面的Message next（）里面也是死循环，一旦执行下去之后发现mQuiiting为true，return null

也就是Looper。loop中的死循环

if (msg == null) return;loop收到null直接返回



quitSafely作用，

1.释放内存，获取不到Looper。且让Looper退出

2.释放线程

主线程使用Loop。quitSafely，会抛出异常不让你调用

销毁的过程只是置零，并不会把东西一个一个移除



子线程-》主线程

保证不会出现队列的混乱，保证线程安全

涉及了多线程并发编程

1.安全性

Messagequeue里面的enqueueMessage放消息里面有synchronize

next取消息synchronize，保证取消息

2.handler delay消失时间准确吗？安全性

synchronize修饰一个方法，静态方法，代码块（object）和代码块（this）区别



我们使用message该怎么创建

obtain

内存共享，内存复用，内存抖动

享元设计模式

synchronize（spoolsync）

Message m=spool

Message属性会持有一个next，spool是消息队列的头，属于链表



使用handler delay消息队列会有什么变化

这个消息队列为空，不会立刻执行，计算等待时间

添加一个message，立刻在messagequeue里面的enquemessage函数nativeWake来唤醒Message next（）函数中nativePollOnce（），接着出发Message next（）函数中nextPollTimeoutMills变量的值，等下次next循环会在nativePollOnce中体现睡眠时间，wait



Looper死循环为什么不会导致应用卡死

主线程泡在Looper。loop

app都有一个虚拟机，都有一个main函数

（ams问题APP启动流程）launcher-》application-》zygote-》虚拟机-》ActivityThread



activity，service都有一个loop 以消息的方式存在



主线程唤醒的方式

1）输入事件2）往looper里面添加消息



ANR（屏幕按下）广播10秒内没有执行

主线程中Looper。loop next会休眠

类似微信banner，消息都会第二天通知，ui主线程休眠



应用卡死跟Looper是没有关系的，应用卡死是跟输入无响应有关

1.Looper。loop中的queue。next是在休眠，并不是卡死（ANR）

2.屏幕上的操作即为输入事件，输入完之后

msg。target。dispatchMessage





# 问题回答

上面源码基本就分析到这边了，咱们看看能根据这些知识点，能提一些什么问题呢？



**1、先来个自己想的问题：Handler中主线程的消息队列是否有数量上限？为什么？**



这问题整的有点鸡贼，可能会让你想到，是否有上限这方面？而不是直接想到到上限数量是多少？



解答：Handler主线程的消息队列肯定是有上限的，每个线程只能实例化一个Looper实例（上面讲了，Looper.prepare只能使用一次），不然会抛异常，消息队列是存在Looper()中的，且仅维护一个消息队列



重点：每个线程只能实例化一次Looper()实例、消息队列存在Looper中



拓展：MessageQueue类，其实都是在维护mMessage，只需要维护这个头结点，就能维护整个消息链表



**2、Handler中有Loop死循环，为什么没有卡死？为什么没有发生ANR？**



先说下ANR：5秒内无法响应屏幕触摸事件或键盘输入事件；广播的onReceive()函数时10秒没有处理完成；前台服务20秒内，后台服务在200秒内没有执行完毕；ContentProvider的publish在10s内没进行完。所以大致上Loop死循环和ANR联系不大，问了个正确的废话，所以触发事件后，耗时操作还是要放在子线程处理，handler将数据通讯到主线程，进行相关处理。



线程实质上是一段可运行的代码片，运行完之后，线程就会自动销毁。当然，我们肯定不希望主线程被over，所以整一个死循环让线程保活。



为什么没被卡死：在事件分发里面分析了，在获取消息的next()方法中，如果没有消息，会触发nativePollOnce方法进入线程休眠状态，释放CPU资源，MessageQueue中有个原生方法nativeWake方法，可以解除nativePollOnce的休眠状态，ok，咱们在这俩个方法的基础上来给出答案



- 当消息队列中消息为空时，触发MessageQueue中的nativePollOnce方法，线程休眠，释放CPU资源
- 消息插入消息队列，会触发nativeWake唤醒方法，解除主线程的休眠状态

- 当插入消息到消息队列中，为消息队列头结点的时候，会触发唤醒方法
- 当插入消息到消息队列中，在头结点之后，链中位置的时候，不会触发唤醒方法

- 综上：消息队列为空，会阻塞主线程，释放资源；消息队列为空，插入消息时候，会触发唤醒机制

- 这套逻辑能保证主线程最大程度利用CPU资源，且能及时休眠自身，不会造成资源浪费

- 本质上，主线程的运行，整体上都是以事件（Message）为驱动的


**3、为什么不建议在子线程中更新UI？

**

多线程操作，在UI的绘制方法表示这不安全，不稳定。



假设一种场景：我会需要对一个圆进行改变，A线程将圆增大俩倍，B改变圆颜色。A线程增加了圆三分之一体积的时候，B线程此时，读取了圆此时的数据，进行改变颜色的操作；最后的结果，可能会导致，大小颜色都不对。。。



**4、可以让自己发送的消息优先被执行吗？原理是什么？**



这个问题，我感觉只能说：在有同步屏障的情况下是可以的。



同步屏障作用：在含有同步屏障的消息队列，会及时的屏蔽消息队列中所有同步消息的分发，放行异步消息的分发。



在含有同步屏障的情况，我可以将自己的消息设置为异步消息，可以起到优先被执行的效果。



**5、子线程和子线程使用Handler进行通信，存在什么弊端？**



子线程和子线程使用Handler通信，某个接受消息的子线程肯定使用实例化handler，肯定会有Looper操作，Looper.loop()内部含有一个死循环，会导致线程的代码块无法被执行完，该线程始终存在。



如果在完成通信操作，我们一般可以使用：mHandler.getLooper().quit() 来结束分发操作



说明下quit()方法本质上是清空消息队列，让该线程进入休眠，确保所有事务都处理完，可以使用thread.interrupt()中断线程



**6、Handler中的阻塞唤醒机制？**



这个阻塞唤醒机制是基于 Linux 的 I/O 多路复用机制 epoll 实现的，它可以同时监控多个文件描述符，当某个文件描述符就绪时，会通知对应程序进行读/写操作.



MessageQueue 创建时会调用到 nativeInit，创建新的 epoll 描述符，然后进行一些初始化并监听相应的文件描述符，调用了epoll_wait方法后，会进入阻塞状态；nativeWake触发对操作符的 write 方法，监听该操作符被回调，结束阻塞状态



详细请查看：同步屏障？阻塞唤醒？和我一起重读 Handler 源码（https://xiaozhuanlan.com/topic/0843791256）



**7、什么是IdleHandler？什么条件下触发IdleHandler？**



IdleHandler的本质就是接口，为了在消息分发空闲的时候，能处理一些事情而设计出来的



具体条件：消息队列为空的时候、发送延时消息的时候



**8、消息处理完后，是直接销毁吗？还是被回收？如果被回收，有最大容量吗？**



Handler存在消息池的概念，处理完的消息会被重置数据，采用头插法进入消息池，取的话也直接取头结点，这样会节省时间



消息池最大容量为50，达到最大容量后，不再接受消息进入



**9、不当的使用Handler，为什么会出现内存泄漏？怎么解决？**



先说明下，Looper对象在主线程中，整个生命周期都是存在的，MessageQueue是在Looper对象中，也就是消息队列也是存在在整个主线程中；我们知道Message是需要持有Handler实例的，Handler又是和Activity存在强引用关系



存在某种场景：我们关闭当前Activity的时候，当前Activity发送的Message，在消息队列还未被处理，Looper间接持有当前activity引用，因为俩者直接是强引用，无法断开，会导致当前Activity无法被回收



思路：断开俩者之间的引用、处理完分发的消息，消息被处理后，之间的引用会被重置断开



解决：使用静态内部类弱引Activity、清空消息队列

# 参考





# 猜你想看

Handler1 Looper

Handler3 同步屏障

Handler4 HandlerThread

Handler5 IntentService