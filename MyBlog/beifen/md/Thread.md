Thread

java三种线程，严格意义上来讲它只有两种，callable不算,callable执行线程实质就是通过包装成FutureTask，并交给一个线程去执行的。本质上还是Runable就是第二种，FutureTask是实现了RunableFuture的，那RunableFuture接口继承了两个接口Runable和Future。

```java
//Thread.java
* There are two ways to create a new thread of execution. One is to
* declare a class to be a subclass of <code>Thread</code>. This
* subclass should override the <code>run</code> method of class
* <code>Thread</code>. An instance of the subclass can then be
* allocated and started. For example, a thread that computes primes
* larger than a stated value could be written as follows:
* <hr><blockquote><pre>
*     class PrimeThread extends Thread {
*         long minPrime;
*         PrimeThread(long minPrime) {
*             this.minPrime = minPrime;
*         }
*
*         public void run() {
*             // compute primes larger than minPrime
*             &nbsp;.&nbsp;.&nbsp;.
*         }
*     }
* </pre></blockquote><hr>
* <p>
* The following code would then create a thread and start it running:
* <blockquote><pre>
*     PrimeThread p = new PrimeThread(143);
*     p.start();
* </pre></blockquote>
* <p>
* The other way to create a thread is to declare a class that
* implements the <code>Runnable</code> interface. That class then
* implements the <code>run</code> method. An instance of the class can
* then be allocated, passed as an argument when creating
* <code>Thread</code>, and started. The same example in this other
* style looks like the following:
* <hr><blockquote><pre>
*     class PrimeRun implements Runnable {
*         long minPrime;
*         PrimeRun(long minPrime) {
*             this.minPrime = minPrime;
*         }
*
*         public void run() {
*             // compute primes larger than minPrime
*             &nbsp;.&nbsp;.&nbsp;.
*         }
*     }
* </pre></blockquote><hr>
* <p>
* The following code would then create a thread and start it running:
* <blockquote><pre>
*     PrimeRun p = new PrimeRun(143);
*     new Thread(p).start();
* </pre></blockquote>
* <p>
```



```java
//第一种，实实在在的线程
private static class StudentThread extends Thread{
    @override
    public void run(){
        super.run();
        System.out.println("do work Thread");
    }
}
```



```java
//2.任务----》》Thread.start
public static class PersonThread implements Runnable{
    @override
    public void run()
    {
        system.out.println("do work RUnnale");
    }
}
```



```java
//3.任务 有返回值 return
private static class WorkThread implements Callable<String>{
    @override
    public String call() throws Exception{
        System.out.println("do work");
        Thread.sleep(2000);
        return "run success";
    }
}
```



```java
//1.
StudentThread thread = new StudentThread();
thread.start();//.start()才能证明这个是线程会走到private native void start0();.run()只是函数调用
//2.
PersonThread personthread = new PersonThread();
new Thread(personthread).start();

//3.有返回值
WorkThread workthread = new WorkThread();
FutureTask<String> futuretask = new FutureTask<>(workthread);
new Thread(futuretask).start();
System.out.println(futuretask.get());

```



停止线程

stop已经过时了，强烈不推荐使用

1）无法停止

 ![image-20211021224301243](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211021224301243.png)

2）和谐停止，通过标记位就结束

![image-20211021224548532](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211021224548532.png)

3）和谐的停止，通过当前的Thread获取interrupt

![image-20211021224842690](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211021224842690.png)





# #Todo 如何控制线程的执行顺序

t1.join();

t2.start();

先t1后t2

join来控制，让t2获取执行权力，能够做到顺序执行

![image-20211123230650786](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211123230650786.png)

结合下图，t1获取执行权，t2一直处于就绪状态，直到t1结束，执行权落到t2，才执行t2，所以t1和t2是顺序执行，没有抢占现象



#todo 在Java中能不能指定cpu去执行某个线程

不能，唯一能去干预的就是c语言调用内核api去指定才行 

#todo 在项目开发中，你会考虑java线程优先级吗

不会考虑，因为线程的优先级很依赖与系统的平台，所以这个优先级无法对号入座，无法做到你想象中的优先级，属于不稳定，有风险，java优先级有十级，操作系统优先级只有2-3级，那么对应不上

#todo sleep和wait有什么区别

sleep是休眠，等休眠时间已过，才有执行权的资格，（并不代表马上执行，什么时候执行取决于操作系统调度）

wait是等待，需要人家唤醒，唤醒后，才有执行的资格，并不代表马上执行，什么时候执行取决于操作系统调度）

#todo 在java中能不能强制中断线程的执行

不能，虽然有stop但是已经停止使用，因为暴力的方式很危险，下载图片5kb，只下载了4kb。我们可以使用interrupt来处理线程的停止，但是注意interrupt只是协作式的方式，并不能保证绝对中断，并不是抢占式的

#todo 如何让出当前线程的执行权

yield方式，只有jdk某些实现才能看到，是让出执行权

#todo sleep，wait到底哪个函数才会清除中断标记

**sleep在抛出异常的时候，捕获异常之前，就已经清除**

sleep的时候会清除中断标记interpret，即sleep会将线程挂起用于清楚中断标记，所以通常这么写

```java
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    //中断标志已经被清除了
    // 手动中断本线程，将本线程打上中断信号。
    Thread.currentThread().interrupt();
}
// Thread.currentThread().isInterrupted():是否被中断了（是否有中断标志）
if(!Thread.currentThread().isInterrupted()) {
    //如果没有被中断，则处理业务
    doSomething();
}
```





![image-20211021225846394](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211021225846394.png)

守护线程

t.setDameon(true);

t.start();

//主线程，是为了等Thread t 10s，主线程结束，守护线程，会让子线程也结束

Thread.sleep(10000);



锁(类锁、对象锁、显示锁)

synchronized**都是内置锁**，lock可以是**显示锁**（interface接口，需要实现）

obj不加static就是一般的对象锁，如果加了static就是类锁

![image-20211129220105755](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211129220105755.png)

单例模式dcl用到的就是**类锁**



![image-20211129220345974](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211129220345974.png)

```java
//对象锁--内置锁
//synchronized的作用 
public class SynTest {
	private long count = 0;
	private Object obj = new Object();//作为一个锁
	
	public void inCount(){
		synchronized(obj)//使用对象锁
		{
			count++;
		}
	}
	//对象锁，内置锁
	public synchronized void inCount1(){
		count++;
	}
	
    //count进行累加
    public void inCount2(){
        synchronized(this){//this为对象锁
            count++;
        }
    }
    
    public static class Count extends Thread{
        private SynTest simplOper;
        public Count(SynTest simplOper) {this.simplOper = simplOper;}
        
        @override
        public void run(){
            for(int i = 0; i < 10000; i++)
            {
                simplOper.inCount();
            }
        }
    }
    
    public static void main(String[] args)
    {
        SynTest simplOer = new SynTest();
        
        Count count1 = new Count(simplOer);
        Count count2 = new Count(simplOer);
        count1.start();
        count2.start();
    }
}

 
```



类锁

```java
public class SynClzAndTest{
    private static class SynClass extends Thread{
        @override
        public void run(){
            System.out.println("TestClass is running");
            synClass();
        }
    }
    //类锁,内置锁 SynClass.class
    private static synchronized void synClass(){
        SleepTools.second(1);
        System.out.println("synClass is going");
        SleepTools.second(1);
        System.out.println("synClass is end");
    }
    //加了static就是特殊的类锁 obj对象类锁
    private static Object obj = new Object();
    
    private void synStaticObject(){
        synchronized(obj)
        {
            SleepTools.second(1);
            System.out.println("synClass is going");
            SleepTools.second(1);
            System.out.println("synClass is end");
        }
    }
}
```

![image-20211025220938193](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211025220938193.png)

重入锁定义：递归调用自己，锁可以释放出来

as继承关系**Ctrl+H**查看，

![image-20211129222100603](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211129222100603.png)

![image-20211129221522506](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211129221522506.png)



规范的写可重入锁，必须加上final

![image-20211129222327511](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211129222327511.png)

重入锁，ReentrantLock

```java
//synchronized,默认为可重入锁，自己调用自己，锁可以释放出来
//synchronized是不能内部中断
//synchronized没法尝试等10s后再去拿锁
//synchronized非公平锁
public synchronized void incr2()
{
    count++;
    incr2();
}
//非公平锁
private Lock lock = new ReentrantLock();
public void incr(){
    lock.lock();
    try{
        count++;//加finally为了防止try语句未执行
    }finally{//finally意思为是否异常，都会执行
        lock.unlock();
    }
}
```

synchronized

1. 如果没有实现可重入，会自己把自己锁死
2. 中间不能中断
3. 没法尝试等待10s后再去拿锁





**lock和synchronized两个都是非公平锁**

其中线程的挂起状态到可执行状态，会**有上下文切换20000时间周期**

非公平锁和公平锁效率那个高？

非公平锁效率高

private Lock lock = new ReentrantLock(true);//**公平锁**



防止死锁

两次synchronized会容易死锁



wait或者nitify为什么需要syn包装，否则会报错

wait等待区域：

1.获取对象锁

2.检测条件等待，内部逻辑

3.syn(this) wait();//wait后持有的this锁，会被释放

notify通知区域：

1.获取对象锁

2.syn(this) notify();或者notifyAll();



内置锁和显示锁执行速率

显示锁的读写锁效率会比内置锁快16倍以上

内置锁，synchronized来读写，大概17s左右

![image-20211130223057505](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211130223057505.png)

显示读写锁锁，800ms左右

![image-20211130224244605](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211130224244605.png)

![image-20211130223203614](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211130223203614.png)

```text
儿时的游戏：（等待 与 唤醒） 
　　有一群小朋友一起玩一个游戏，这个游戏可能大家都玩过，大家一起划拳，划拳输得最惨的那个小朋友去抓人(这个小朋友取名为 CPU)，被抓的很多人取名为线程，有很多线程，如果其中一个小朋友(例如:Thread-3) 被木头了(wait();)  就站着不准动了(冻结状态)，Thread-3小朋友站着不动(冻结状态) CPU小朋友就不会去抓Thread-3小朋友，因为Thread-3小朋友(释放CPU执行资格)   需要其他小朋友(例如:Thread-5) 去啪一下Thread-3小朋友(notify();),  此时Thread-3小朋友就可以跑了(被唤醒) 此时Thread-3小朋友具备被抓的资格(具备CPU执行资格)，能否被抓得到 要看CPU小朋友的随机抓人法（CPU切换线程的随机性）
```







![image-20211030105710760](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030105710760.png)

当前线程等待cpu轮转调度，被分配了时间片，即从ready状态转到Running状态。

进入阻塞（被迫进入）只有一种状态，synchronized，其他显示锁或其他都是等待或者等待超时



死锁条件：

1.资源数小于等于操作者数，操作者M》=2，资源》=2

2.争夺资源顺序不对

3.拿到资源不放手

1.不互斥条件

2.请求保持

3.不剥夺

4.环路等待



去除死锁，一是都按顺序来，而是使用显示锁，并且用尝试加锁的方式，通过休眠循环来获取。

![image-20211030111625768](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030111625768.png)

不加休眠就是活锁。

拿到锁释放锁拿到锁释放锁，反复循环，但是并没有处理到业务，看起来比较忙碌。



CAS compare and swap

![image-20211030122107122](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030122107122.png)

实现原子操作使用锁

原子性：不可再分，要不一个都没执行要不全部执行

关键字Synchronized也是原子性



Synchronized是悲观锁，总有刁民想害郑，其他会被阻塞，上下文切换



CAS是乐观锁，可能没有人修改，性能要高，并发编程会更好

自旋--jdk死循环



cas问题：

ABA问题：设置版本戳![image-20211030123412933](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030123412933.png)



![image-20211030123444415](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030123444415.png)

开销问题：还是用加锁机制

只能保证一个共享变量原子操作：用一个新类装几个变量

![image-20211030124937689](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211030124937689.png)

