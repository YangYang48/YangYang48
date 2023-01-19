---
title: "ThreadLocal之子父线程的恩爱情仇"
date: 2022-04-10
thumbnailImagePosition: left
thumbnailImage: ThreadLocal/threadlocal2_thumb.jpg
coverImage: ThreadLocal/threadlocal2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- InheritableThreadLocal
- 2022
- April
tags:
- Android
- 源码
- framework
- Thread
- ThreadLocal
showSocial: false
---
我们知道Thread中会维护两个ThreadLocalMap，这个时候如果同时存在子父线程，子线程该如何获取父线程ThreadLocal的值

<!--more-->
# 1Java的子父线程

子父线程，可以简单的人为，子线程就是一个线程A中的一个线程B，那么父线程就是线程A。

如下举一个小例子，可以得知子线程和父线程拥有不同的生命周期，可以是主线程退出，子线程依然在运行。

```java
public class ThreadTest {
    public static void main(String[] args) throws InterruptedException {
        Thread parentThread = new Thread(() -> {

            new Thread(() -> {
                System.out.println("son Thread exit");
            }, "子线程").start();
            System.out.println("parentThread Thread exit");
        }, "父线程");
        parentThread.start();
        System.out.println("main exit");
    }
}
```

{{< image classes="fancybox center fig-100" src="/ThreadLocal/threadlocal2_3.png" thumbnail="/ThreadLocal/threadlocal2_3.png" title="">}}

> 原理说明
>
> 如果main方法中没有创建其他线程，那么当main方法返回时JVM就会结束Java应用程序。但如果main方法中创建了其他线程，那么JVM就要在主线程和其他线程之间轮流切换，保证每个线程都有机会使用CPU资源，main方法返回(主线程结束)JVM也不会结束，要一直等到该程序所有线程全部结束才结束Java程序(另外一种情况是：程序中调用了Runtime类的exit方法，并且安全管理器允许退出操作发生。这时JVM也会结束该程序)。





# 2子线程获取父类的ThreadLocal

## 2.1demo打印

如果想在as中运行纯java文件，可以点击[这里](https://blog.csdn.net/tectrol/article/details/104393319)。

如果使用Lamda表达式，推荐使用JavaVersion.VERSION_1_8。

```java
public class ThreadTest {
    public static void main(String[] args) throws InterruptedException {
        Thread parentThread = new Thread(() -> {
            ThreadLocal<Integer> tLocal = new ThreadLocal<>();
            tLocal.set(1);
            InheritableThreadLocal<Integer> iTLocal = new InheritableThreadLocal<>();
            iTLocal.set(2);
            System.out.println("->>>threadLocal=" + tLocal.get());
            System.out.println("->>>inheritableThreadLocal=" + iTLocal.get());

            new Thread(() -> {
                System.out.println("threadLocal=" + tLocal.get());
                System.out.println("inheritableThreadLocal=" + iTLocal.get());
            }, "子线程").start();
        }, "父线程");
        parentThread.start();
    }
}
```

结果

{{< image classes="fancybox center fig-100" src="/ThreadLocal/threadlocal2_1.png" thumbnail="/ThreadLocal/threadlocal2_1.png" title="">}}

# 3源码分析

这里笔者看的是AndroidSDK中的Api30，和实际的JDK1.8会有区别，下面会有说明。

查看Thread源码，里面定义了两个ThreadLocalMap。关于Thread，ThreadLocal和ThreadLocalMap关系，可以点击[这里](https://yangyang48.github.io/2021/11/threadlocal%E4%B9%8B%E5%88%9D%E5%87%BA%E8%8C%85%E5%BA%90/)。

```java
//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

## 3.1父线程

首先，还是看父线程。父线程只有两个操作

### 3.1.1.创建两个ThreadLocal

创建两个ThreadLocal，分别是tLocal和iTLocal

```java
ThreadLocal<Integer> tLocal = new ThreadLocal<>();
InheritableThreadLocal<Integer> iTLocal = new InheritableThreadLocal<>();
```

### 3.1.2ThreadLocal的set方法

对这两个ThreadLocal做了一个set操作

```java
tLocal.set(1);
iTLocal.set(2);
```

查看tLocal的set方法，可以知道ThreadLocal会和Thread绑定在一起

```java
//ThreadLocal.java#set
public void set(T value) {
    //获取当前的线程
    Thread t = Thread.currentThread();
    //查找当前线程的map表，这个表中会存放键值对，key是ThreadLocal，value为ThreadLocal需要存入的值
    ThreadLocal.ThreadLocalMap map = this.getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        //因为map会不存在，这里会走到，去创建表
        this.createMap(t, value);
    }
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

这里的创建其实就是创建一个16个长度的Entry组成的map，因为这里有魔数的存在，所以是从2开始排列。

```java
//Thread.java#ThreadLocalMap
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

同理，iTLocal的set方法也是类似。区别在于有两个函数是复写了。

```java
//InheritableThreadLocal.java#getMap和createMap
ThreadLocalMap getMap(Thread t) {
    return t.inheritableThreadLocals;
}

void createMap(Thread t, T firstValue) {
    t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
}
```

创建完成之后如下图所示。

{{< image classes="fancybox center fig-100" src="/ThreadLocal/threadlocal2_4.png" thumbnail="/ThreadLocal/threadlocal2_4.png" title="">}}

## 3.2子线程

### 3.2.1子线程Thread构造

```java
//Thread.java#Thread
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
    init(g, target, name, stackSize, null);
}

private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    this.name = name;

    Thread parent = currentThread();
    if (g == null) {
        g = parent.getThreadGroup();

    }

    g.addUnstarted();

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();

    this.target = target;
    //最终会走到这里，其中parent的类型是Thread，那么这里的parent就是子线程还在创建过程中的父线程
    init2(parent);

    this.stackSize = stackSize;

    tid = nextThreadID();
}
```

子线程Thread初始化，有一个关于和父线程的判断。

如果父线程中存在一个inheritableThreadLocals属性且不为null，那么子线程中的inheritableThreadLocals创建起来。

```java
//Thread.java#init2
private void init2(Thread parent) {
    this.contextClassLoader = parent.getContextClassLoader();
    this.inheritedAccessControlContext = AccessController.getContext();
    if (parent.inheritableThreadLocals != null) {
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(
            parent.inheritableThreadLocals);
    }
}
```

> 因为笔者用得是AndroidSDK的Api30，实际上这里的构造和JDK1.8有一定差距，主要看判断处
>
> ```java
> public Thread(ThreadGroup group, Runnable target, String name, long stackSize) {
>     this(group, target, name, stackSize, (AccessControlContext)null, true);
> }
> 
> //这个Thread的构造，可以直接定义inheritThreadLocals的值，是否进行后续的copy操作
> public Thread(ThreadGroup group, Runnable target, String name, long stackSize, boolean inheritThreadLocals) {
>     this(group, target, name, stackSize, (AccessControlContext)null, inheritThreadLocals);
> }
> 
> private Thread(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc, boolean inheritThreadLocals) {
>     this.daemon = false;
>     this.stillborn = false;
>     this.threadLocals = null;
>     this.inheritableThreadLocals = null;
>     this.blockerLock = new Object();
>     if (name == null) {
>         throw new NullPointerException("name cannot be null");
>     } else {
>         this.name = name;
>         Thread parent = currentThread();
>         ...
>         this.inheritedAccessControlContext = acc != null ? acc : AccessController.getContext();
>         this.target = target;
>         this.setPriority(this.priority);
>         //这里的判断增加了inheritThreadLocals，这个inheritThreadLocals值是从上面传进来的true。
>         //JDK源码设置的好处就是，可以从外部自定义，我可以传入false，不进行下面的操作，扩展性更强
>         if (inheritThreadLocals && parent.inheritableThreadLocals != null) {
>             this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
>         }
> 
>         this.stackSize = stackSize;
>         this.tid = nextThreadID();
>     }
> }
> ```
>
> JDK源码设置的好处就是，可以从外部Thread构造的时候决定是否需要创建inheritableThreadLocals。

开始创建Map

```java
//ThreadLocal.java#createInheritedMap
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

传入Map的拷贝构造

```java
//ThreadLocal.java#ThreadLocalMap
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

构造完成之后如下图所示

{{< image classes="fancybox center fig-100" src="/ThreadLocal/threadlocal2_5.png" thumbnail="/ThreadLocal/threadlocal2_5.png" title="">}}

### 3.2.2子线程中的ThreadLocal获取

首先是tLocal的get方法

```java
//ThreadLocal.java#get
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //因为没有Map，所以会走到这里
    return setInitialValue();
}
```

因为没有默认的threadLocals表，那么就自己创建一个表，返回的是null

```java
//ThreadLocal.java#setInitialValue
private T setInitialValue() {
    //初始化一个value值，默认是null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        //因为createMap在上面复述过，这里不再展开，意思就是创建一个键值对，其中key是tLocal，value为null
        createMap(t, value);
    return value;
}
```

其次是iTLocal的get方法，跟上面类似。唯一的区别是子线程中是存在inheritableThreadLocals表的。

```java
//ThreadLocal.java#get
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            //因为存在Map，且里面的value的值也是不为null的，那么可以返回这个value值
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

最终如下图所示

{{< image classes="fancybox center fig-100" src="/ThreadLocal/threadlocal2_2.png" thumbnail="/ThreadLocal/threadlocal2_2.png" title="">}}

# 4总结

1. 父线程中的一个ThreadLocalMap类型的inheritableThreadLocals属性，保存一对键值。key为定义的一个InheritableThreadLocal类型的iTLocal，value为定义的Integer类型的2。
2. 子线程的创建过程，会先判断是否存在父线程的inheritableThreadLocals这个属性，如果存在那么copy一份到子线程对应的inheritableThreadLocals属性中。其中JDK增加了额外的判断，是否在父线程中创建了InheritableThreadLocal类型的实例，创建了才能去copy，如果没有创建那么不做拷贝动作。
3. 子线程运行中获取当前子线程中的InheritableThreadLocal类型的实例的对应的value值。直接查询当前子线程的ThreadLocalMap类型的inheritableThreadLocals属性中的键值对。其中键值对的key为iTLocal，对应的value为2。



# 本文源码

[这点这里查看](https://github.com/YangYang48/project/tree/master/MyThreadLocal)



# 猜你喜欢

[ThreadLocal之初出茅庐](https://yangyang48.github.io/2021/11/threadlocal%E4%B9%8B%E5%88%9D%E5%87%BA%E8%8C%85%E5%BA%90/)



# 参考

[[1] 添码星空, AS编写运行测试纯java代码，带main()函数, 2022.](https://blog.csdn.net/tectrol/article/details/104393319)

[[2] NCSTA, Java中的父线程与子线程, 2018.](https://blog.csdn.net/u011535508/article/details/83515702)

[[3] 只会一点java, ThreadLocal终极源码剖析-一篇足矣！, 2017.](https://www.cnblogs.com/dennyzhangdd/p/7978455.html)

[[4] 刘望舒, 京东一面：子线程如何获取父线程ThreadLocal的值, 2022.](https://mp.weixin.qq.com/s/RBAIHnfzDXXKEc9m1eP5ZA)
