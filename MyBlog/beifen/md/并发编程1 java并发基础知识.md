并发编程1 java并发基础知识

# 1新启线程的方式

Thread源码启线程的方式

```java
/*有两种方法可以创建新的执行线程。 一是要
  * 将一个类声明为 {@code Thread} 的子类。 这个
  * 子类应该覆盖类的 {@code run} 方法
  * {@code 线程}。 然后子类的实例可以是
  * 分配并启动。 例如，一个计算素数的线程
  * 大于规定值可以写成如下：
 */
class PrimeThread extends Thread {
    long minPrime;
    PrimeThread(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
        &nbsp;.&nbsp;.&nbsp;.
    }
}

PrimeThread p = new PrimeThread(143);
p.start();

class PrimeRun implements Runnable {
    long minPrime;
    PrimeRun(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
        &nbsp;.&nbsp;.&nbsp;.
    }
}

PrimeRun p = new PrimeRun(143);
new Thread(p).start();
```



callable不算第三种

```java
//callable的demo
//注意点，这里的泛型需要使用原始类型
static class PrimCallable implements Callable<Long>{
    long minPrime;

    public PrimCallable(long minPrime) {
        this.minPrime = minPrime;
    }


    @Override
    public Long call() throws Exception {
        System.out.println("->>>PrimCallable is starting!");
        return minPrime;
    }
} 

//Thread 启动第二种方式的变体，实际上是包装runnable，交给线程去执行
PrimCallable pCallable = new PrimCallable(143);
FutureTask<Long> futureTask = new FutureTask<>(pCallable);
new Thread(futureTask).start();
//获取返回值
System.out.println(futureTask.get());
```



交给线程去执行的时候，实际上是包装runnable，交给线程去执行。

本质上实际上是实现了runnable接口

```java
public class FutureTask<V> implements RunnableFuture<V> {}
public interface RunnableFuture<V> extends Runnable, Future<V> 
```

![concurrent1_1](D:\hugo\MyBlog\static\concurrent\concurrent1_1.png)



# 2线程的生命周期

![concurrent1_2](D:\hugo\MyBlog\static\concurrent\concurrent1_2.png)

通过jdk文档查看Thread状态，一共分为6种状态。分别为BLOCKED（阻塞），NEW（初始）、RUNNABLE（运行）、TERMINATED（终止）、TIMED_WAITING（等待超时）和WAITING（等待）。

![concurrent1_3](D:\hugo\MyBlog\static\concurrent\concurrent1_3.png)

NEW--->RUNNABLE

start之后还分成两个阶段，running和ready，从running到ready会经过时间轮转和上下文切换，等待cpu调度。Ready调用yield操作等待操作系统分配时间片，时间片轮转又回到running，这个阶段在juc里面称为RUNNABLE状态。java里面running和ready合二为一，统称运行态，而操作系统做了区分，分成两种形式。

RUNNABLE--->TERMINATED

表示该run方法执行完成，或者run方法抛出异常。

RUNNABLE<--->WAITING

表示运行状态和等待状态可以相互切换。运行态转到等待状态，使当前调用该方法的线程暂停执行一段时间，让其他线程有机会执行。等待状态转到运行态，notify或者显示锁unpark皆可以。

RUNNABLE<--->TIMED_WAITING

表示运行状态和等待超时状态可以相互切换。运行态转到等待超时状态，该状态不同于WAITING，它可以在指定的时间后自行返回。等待超时状态转到运行态，在指定时候后，等待超时状态结束，可以看到LockSupport.unpark或者notify。

RUNNABLE<--->BLOCKED

显示锁加锁去锁都不算阻塞状态，**阻塞状态只有在synchronized才算**。等待该线程拿到对应的synchronized的锁之后，退出阻塞状态，进入运行状态。



# 3.死锁

比较通俗的描述死锁：

1.拥有多个操作者，操作者大于等于2人，争夺多个资源，资源数大于等于2项，且操作者数量大于等于资源数量。

2.多个操作者争夺资源的顺序不一样。

3.多个操作者拿到资源不放手。----可以用lock尝试拿锁可以打破死锁



> 死锁产生的4个必要条件？
> 产生死锁的必要条件：
>
> 互斥条件：进程要求对所分配的资源进行排它性控制，即在一段时间内某资源仅为一进程所占用。
> 请求和保持条件：当进程因请求资源而阻塞时，对已获得的资源保持不放。
> 不剥夺条件：进程已获得的资源在未使用完之前，不能剥夺，只能在使用完时由自己释放。
> 环路等待条件：在发生死锁时，必然存在一个进程--资源的环形链。



避免死锁

只要打破四个必要条件之一就能有效预防死锁的发生。

1.打破互斥条件：改造独占性资源为虚拟资源，增加资源数，让多个操作者不在发生争夺。

2.打破不可抢占条件：当操作者占有一争夺资源后又尝试争夺其他资源而无法满足，操作者争夺资源选择放手。用lock尝试拿锁代替synchronized。

3.打破占有且申请条件：采用资源预先分配策略，操作者要不就争夺所有资源，要不就等待，不能单独争夺某一个资源。

4.打破循环等待条件：多个操作者争夺资源的顺序发生变化，都保持顺序一致。

```java
//对象锁
public static Object obj1 = new Object();//第一个锁
public static Object obj2 = new Object();//第二个锁
//可重入锁
private static Lock obj3 = new ReentrantLock();//第一个锁
private static Lock obj4 = new ReentrantLock();//第二个锁

//1.破坏死锁的第一种方式 增加资源数
static void OperatorOneScramble() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    System.out.println(threadName+" get first-1");
    System.out.println(threadName+" get second-1");
    Thread.sleep(100);
}

//2.破坏死锁的第二种方式 通过尝试拿资源的方式
static void OperatorOneScramble() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    Random r = new Random();
    while(true)
    {
        if (obj3.tryLock()){
            System.out.println(threadName+" get first");
            try{
                if (obj4.tryLock()){
                    try {
                        System.out.println(threadName+" get second");
                        System.out.println(threadName+" success done!");
                        break;
                    }finally {
                        System.out.println(threadName+" obj4.unlock");
                        obj4.unlock();
                    }
                }
            }finally {
                obj3.unlock();
            }
        }
        //循环里面休眠一小段时间的原因,防止活锁
        Thread.sleep(r.nextInt(3));
    }
}

//3.破坏死锁的第三种方式 原子性的方式拿资源
static void OperatorOneScramble() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    synchronized (obj1){
        System.out.println(threadName+" get first");
        System.out.println(threadName+" get second");
        Thread.sleep(100);
    }
}

//4.破坏死锁的第四种方式 按照顺序obj1-->obj2的方式拿资源
static void OperatorOneScramble() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    synchronized (obj1){
        System.out.println(threadName+" get first");
        Thread.sleep(100);
        synchronized (obj2){
            System.out.println(threadName+" get second");
        }
    }
}
```

由于上面只展示部分，具体看详见[本文源码](https://github.com/YangYang48/project/blob/master/MyConcurrent/concurrent/src/main/java/com/example/concurrent/DealDeadLock.java)。

上面方式二中出现sleep，这个原因是**避免出现活锁，增加时间，没有做实际业务**。



## 线程饥饿

饿死（starvation） 是一个线程长时间得不到需要的资源而不能执行的现象。 有人饿死并不代表着出现了死锁。很有可能系统还能很好的进行。

> 通俗的来讲就像皇帝晚上选妃一样。皇帝优先选择好感度高的妃子，好感度低的妃子就不会被选中。如果皇帝长期只选择几个好感度最高的妃子，其他的妃子就会被冷落。
>
> 这里的皇帝就是cpu，妃子就是对应的线程，好感度就是对cpu而言，线程的优先级别，cpu总让优先级高的线程执行，从而优先级低的线程一直没有处理，就会处于饥饿状态。



如何避免有三种方案：

1. 一是保证资源充足
2. 二是公平地分配资源
3. 三就是避免持有锁的线程长时间执行，切换其他线程会把剩余的时间片给那些饥饿度高的来执行

  

### **活锁**

虽然不会像死锁那样因为获取不到资源而阻塞，也不会像饥饿那样得不到处理器时间而无可奈何，活锁仍旧可以让程序无法执行下去。

> 就像一座独木桥上，AB两人都要到对面去。AB看到对方来了，A示意让B先走，B示意让A走。结果等了一会，AB都认为对方不走，那自己先走。结果刚走，发现对面想继续走。然后AB都互相让一步，发现对面没有过来。接着AB就都走上一步，发现对面想继续走。这样循环往复，各自在这座独木桥重复。
>
> 这里的AB就是为两个线程，走过独木桥才能执行对应的任务，很显然两个线程很忙碌却有没有执行什么任务，两个线程在尝试拿锁的机制中，发生多个线程之间互相谦让，不断发生同一个线程总是拿到同一把锁，在尝试拿另一把锁时因为拿不到，而将本来已经持有的锁释放的过程，这就是活锁。

解决方法：

每个线程休眠随机数，错开拿锁的时间。在循环开始引入一个时间差，这样一来再一次互相让步的过程后，下一次总有一方会比另一方先走这个阻塞，执行对应的任务。



# 4ThreadLocal

线程本地变量。ThreadLocal为每个线程提供了变量的副本，使得每一个线程访问到的是不同的对象，隔离了线程之间对数据的共享访问。内部有一个TheadLocalMap，用来保存每一个变量的副本。关于ThreadLocal更多的内容，可以看之前[ThreadLocal之初出茅庐](https://yangyang48.github.io/2021/11/threadlocal%E4%B9%8B%E5%88%9D%E5%87%BA%E8%8C%85%E5%BA%90/)。



# 参考文献

[[1]  AddoilDan. 死锁面试题（什么是死锁，产生死锁的原因及必要条件）, 2018.](https://blog.csdn.net/hd12370/article/details/82814348)

[[2] pradosoul,Thread学习之二（线程的生命周期以及常见方法）,2015](https://my.oschina.net/u/1757476/blog/420169).

[[3] 大叶子不小,线程生命周期的几种状态,2020](https://blog.csdn.net/qq_32907195/article/details/108906059?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-108906059.pc_agg_new_rank&utm_term=%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E5%8C%85%E6%8B%AC%E5%93%AA%E5%87%A0%E7%A7%8D%E7%8A%B6%E6%80%81&spm=1000.2123.3001.4430).

[[4] 小孩子,活跃性（死锁、饥饿、活锁）,2018](https://mp.weixin.qq.com/s/eZcv847R5tQ6b2V1CjzuJw).

