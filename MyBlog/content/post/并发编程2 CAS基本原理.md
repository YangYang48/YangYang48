---
title: "并发编程2 CAS基本原理"
date: 2021-12-12
thumbnailImagePosition: left
thumbnailImage: concurrent/concurrent2_thumb.jpg
coverImage: concurrent/concurrent2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- CAS
- 2021
- December
tags:
- JDK
- Concurrent
- Thread
showSocial: false
---
CAS(compare and Swap)是由硬件实现的。CAS可以将read- modify - write这类的操作转换为原子操作。jdk1.5之后引入CAS利用CPU原语保证线程操作的原子性。

<!--more-->
# CAS原理

CAS（V, A, B），V为内存地址、A为预期原值，B为新值。

{{< image classes="fancybox center fig-100" src="/concurrent/concurrent2_1.png" thumbnail="/concurrent/concurrent2_1.png" title="CAS(V, A, B)原理图">}}

> 我们假设内存中的原数据V，旧的预期值A，需要修改的新值B。
>
> 1. 比较 A 与 V 是否相等。（比较）
> 2. 如果比较相等，将 B 写入 V。（交换）
> 3. 如果不相等，更新预期值A并循环CAS，重复上两步。



CAS的实现主要在juc里面的atomic包中，其中atomic都是原子操作变量类。

{{< image classes="fancybox center fig-100" src="/concurrent/concurrent2_2.png" thumbnail="/concurrent/concurrent2_2.png" title="juc里面的atomic包">}}



这里一AtomicInteger为例，找到对应的

```java
//package java.util.concurrent.atomic;#AtomicInteger
public final boolean compareAndSet(int expectedValue, int newValue) {
    return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
}
```

这里的compareAndSetInt会调用到Unsafe类中的compareAndSetInt

```java
//package jdk.internal.misc;#Unsafe
@HotSpotIntrinsicCandidate
public final int getAndSetInt(Object o, long offset, int newValue) {
    int v;
    do {
        v = this.getIntVolatile(o, offset);
        //这里的while循环里面，v为内存地址是旧值，newValue为新值，
        //这里的旧值为期望的旧值，不符合条件重新赋值v。如果有其他线程有新值操作，这里的v会更新成newValue，不再是原来的旧值了。
    } while(!this.weakCompareAndSetInt(o, offset, v, newValue));

    return v;
}
//这里的weakCompareAndSetInt会调用到compareAndSetInt
@HotSpotIntrinsicCandidate
public final native boolean compareAndSetInt(Object var1, long var2, int var4, int var5);
```

这里面会间接调用到ndk中的cmpxchg函数

```c++
//linux_x86底层实现\hotspot\src\os_cpu\linux_x86\vm\atomic_linux_x86.inline.hpp
inline jint   Atomic::cmpxchg  (jint   exchange_value, volatile jint*   dest, jint   compare_value) {
 int mp = os::is_MP();
 __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
          : "=a" (exchange_value)
          : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
          : "cc", "memory");
 return exchange_value;
}
```

其中cmpxchg是汇编指令，作用是比较并交换操作数。

- `__asm__` 的意思是这个是一段内嵌汇编代码。
- 这里的 `volatile`和 JAVA 不同，告诉编译器对访问该变量的代码就不再进行优化。
- `LOCK_IF_MP(%4)` 的意思就比较简单，就是如果操作系统是多核的，那就增加一个 LOCK。
- `cmpxchgl` 就是汇编版的“比较并交换”。但是我们知道比较并交换，有三个步骤，不是原子的。这里在多核情况下加一个 LOCK，由CPU硬件保证他的原子性。

说白了就是juc中的CAS原子性实际上是使用cpu硬件提供的Lock信号，保证了原子性。



# CAS实现原子操作的三大问题

## 1.ABA问题

因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。

> 出差前，A在桌子上放了一杯水。结果被B误喝了，B喝了一口发现不是自己水杯，之后拿饮水机又给接满了。A出差回来发现水没人动过，直接喝桌上的水，但桌上的水并不是原来的水了，实际上已经被人喝过了。
>
> 这里的水发生的变化，从a状态（A出差前的状态）-->b（被B误喝的水）-->a（被B重新接满的水）。
>
> 虽然第一个a和第二个a的值没有变化，但本质已经发生变化。
>
> 如果线程将CAS修饰的变量将a变化到b在变化到a，其他线程会认为没有修改。



### 解决ABA问题

解决ABA问题需要加上`版本戳`，每个线程修改这个值的时候增加版本戳。

{{< image classes="fancybox center fig-100" src="/concurrent/concurrent2_3.png" thumbnail="/concurrent/concurrent2_3.png" title="解决ABA问题的类">}}

利用版本戳的形式记录了每次改变以后的版本号，这样的话就不会存在ABA问题了。其中AtomicMarkable只关心动没动过，而AtomicStampedReference不仅关心动没动过，还关注动过几次。



## 2.循环时间长开销问题

当比较内存中变量值和期待的旧值，如果不相等就会自旋。如果这个过程不停地自旋，会给CPU带来非常大的执行开销。

{{< image classes="fancybox center fig-100" src="/concurrent/concurrent2_4.png" thumbnail="/concurrent/concurrent2_4.png" title="">}}

### 解决开销问题

由于线程多且循环次数较多，这个时候不能再用轻量级的CAS指令，只能加锁解决。



## 3.只能保证一个共享变量的原子性

当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性。



### 解决一个共享变量的原子性问题

这个时候会将多个需要改变的共享变量封装成一个类，通过修改这个类的对象中的属性，达到修改一个共享变量原子性的问题。juc里面这个AutomicReference解决，保证一个共享变量的原子性。将多个变量组合成一个java对象，最终修改这个对象，从而达到保证一个共享变量的原子性。



# CAS与Synchronized区别

synchronized是悲观锁，总有刁民想害朕，先下手为强，不允许加锁过程中其中线程进入。加锁线程阻塞会引起上下文切换，会比CAS效率低。



CAS是乐观锁，姿势不对起来重睡，允许多个线程进入CAS循环，发现比较值已经修改过了就进行CAS轮询，直到比较变量值和期望一样，交换为新值。因为不需要阻塞，可以不上锁。

表1 CAS机制和synchronized区别

|          |                           CAS机制                            |                         synchronized                         |
| -------- | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 适用场景 | 乐观认为并发不高，不需要阻塞，可以不上锁。线程数较少、等待时间短可以采用自旋锁进行CAS尝试拿锁，较于synchronized高效。 | 悲观认为并发很高，需要阻塞，需要上锁。一般用于线程数较大、等待时间长，占用CPU较高的场景。 |
| 特点     | 不断比较更新，直到成功<br />比较简单的并行加锁太大材小用，轻量级推荐使用CAS指令 |    语言层面的优化，锁粗化、偏向锁、轻量锁等等；可读性高。    |
| 缺点     | 1.ABA问题<br />2.循环时间长开销问题<br />3.只能保证一个共享变量的原子性 |        加锁线程阻塞会引起上下文切换，会比CAS效率低。         |



JUC中的CAS操作

表2 相关原子操作类的使用

| 方法          | AtomicInteger                                                | AtomicIntegerArray                                           |
| :------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| addAndGet     | `addAndGet(int delta)`以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果。 | `addAndGet(int i, int delta)`以原子方式将输入值与数组中索引i的元素相加。 |
| compareAndSet | `compareAndSet(int expect, int update)`如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。 | `compareAndSet(int i, int expect, int update)`如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update值。 |
| getAndSet     | `getAndSet(int newValue)`以原子方式设置为newValue的值，并返回 | `getAndSet(int i, int newValue)`以原子方式设置数组中索引i为newValue的值，并返回 |

注意：数组value通过构造方法传递进去，然后AtomicIntegerArray会将当前数组**复制**一份，所以当AtomicIntegerArray对内部的数组元素进行修改时，不会影响传入的数组。



# 参考文献

[[1] VincentYew. Java CAS底层实现原理实例详解, 2020.](https://www.jb51.net/article/178206.htm)

[[2] 犀利豆,JAVA 中的 CAS,2018](https://segmentfault.com/a/1190000013127775).

[[3] 动力节点,Java CAS多线程,2020](http://www.bjpowernode.com/javathread/1211.html).

[[4] wanhf11,Java CAS 和 synchronized 和 Lock,2018](https://blog.csdn.net/qq_17612199/article/details/80385737).

