---
title: "Java Thread"
date: 2022-12-22
thumbnailImagePosition: left
thumbnailImage: concurrent/thread_thumb.jpg
coverImage: concurrent/thread_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- Thread
- 2022
- December
tags:
- Android
- Java
- tid
- sleep0
showSocial: false
---

除了本身的并发编程，再聊聊Thread本身的特性。

<!--more-->
# 0简介

之前的并发编程中多少提到了Java的Thread，但是关于Thread还想再聊聊自己的看法。

除了主流对Thread的用法，还有关于Thread.sleep(0)的方法使用

另外，Java端要获取Linux层的线程id的话，应该通过什么方式呢



# 1Thread.Sleep(0)

下面这段代码注释写着保护GC。这个代码是想要“触发”GC，而不是“避免”GC，或者说是“避免”时间很长的 GC。从这个角度来说，程序里面的注释其实是在撒谎或者没写完整。

```java
//这段源码来自于RocketMQ 的源码
for (int i = 0, j = 0; i < this.fileSize; i += MappedFile.OS_PAGE_SIZE, j++) {
        byteBuffer.put(i, (byte) 0);
        // force flush when flush disk type is sync
        if (type == FlushDiskType.SYNC_FLUSH) {
            if ((i / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE) >= pages) {
                flush = i;
                mappedByteBuffer.force();
            }
        }

        // prevent gc
        if (j % 1000 == 0) {
            log.info("j={}, costTime={}", j, System.currentTimeMillis() - time);
            time = System.currentTimeMillis();
            try {
                Thread.sleep(0);
            } catch (InterruptedException e) {
                log.error("Interrupted", e);
            }
        }
 }
```

原理

> - Thread.sleep(0)就是“**触发操作系统立刻重新进行一次CPU竞争**”。结果也许是当前线程仍然获得CPU控制权，也许会换成别的线程获得CPU控制权。
>
>   这样看来，这个场面就有意思了——可能有些人是PPMM，因此具有高优先级，于是她就可以经常来吃蛋糕。可能另外一个人是个丑男，而去很ws，所以优先级特别低，于是好半天了才轮到他一次（因为随着时间的推移，他会越来越饥饿，因此算出来的总优先级就会越来越高，因此总有一天会轮到他的）。而且，如果一不小心让一个大胖子得到了刀叉，因为他饭量大，可能他会霸占着蛋糕连续吃很久很久，导致旁边的人在那里咽口水。。。
>
>   而且，还可能会有这种情况出现：操作系统现在计算出来的结果，5号PPMM总优先级最高，而且高出别人一大截。因此就叫5号来吃蛋糕。5号吃了一小会儿，觉得没那么饿了，于是说“我不吃了”（挂起）。因此操作系统就会重新计算所有人的优先级。因为5号刚刚吃过，因此她的饥饿程度变小了，于是总优先级变小了；而其他人因为多等了一会儿，饥饿程度都变大了，所以总优先级也变大了。不过这时候仍然有可能5号的优先级比别的都高，只不过现在只比其他的高一点点——但她仍然是总优先级最高的啊。因此操作系统就会说：5号mm上来吃蛋糕……（5号mm心里郁闷，这不刚吃过嘛……人家要减肥……谁叫你长那么漂亮，获得了那么高的优先级）。
>
>   那么，Thread.Sleep 函数是干吗的呢？还用刚才的分蛋糕的场景来描述。上面的场景里面，5号MM在吃了一次蛋糕之后，觉得已经有8分饱了，她觉得在未来的半个小时之内都不想再来吃蛋糕了，那么她就会跟操作系统说：在未来的半个小时之内不要再叫我上来吃蛋糕了。这样，操作系统在随后的半个小时里面重新计算所有人总优先级的时候，就会忽略5号mm。Sleep函数就是干这事的，他告诉操作系统“在未来的多少毫秒内我不参与CPU竞争”。
>
> - Thread.sleep(0)**相当于插入了一个安全点。**这样就可以避免你的程序 `GC` 线程长时间等待。也就是所谓的**GC削峰**。
>
>   `Java`虚拟机在进行`GC`的时候，有着`STW`的特性。即`Stop The World`，停顿所有`Java`执行进程。
>
>   既然有`STW`的机制，那么`GC`是随时发起的吗？并是不，`Java`虚拟机利用了一个安全点机制`Safepoint`，**只有程序到达安全点的时候才能够进行`GC`。**有了安全点的设定，也就决定了用户程序执行时并**非在代码指令流的任意位置都能够停顿下来开始垃圾收集**，而是强制要求必须执行到达安全点后才能够暂停。
>
>   **可数循环**：如果循环次数比较少的话，执行时间应该不会太长，**使用`int`类型或者范围更小的数据类型作为索引值，这种是不会被放置安全点的。**循环如果没有结束。程序就无法走到安全点，就无法`GC`。
>
>   **不可数循环**：反之，如果使用`long`这样范围更大的类型作为索引的循环，就叫做不可数循环。此时循环过程中就会插入安全点。**单次循环体结束的时候，就可以进入安全点，无需等待整个循环跑完**。
>
>   
>
>   一个线程在运行 `native` 方法后，返回到 `Java` 线程后，**必须**进行一次 `safepoint` 的检测。Thread.sleep(0)正好为native方法，需要做一个安全点检测。在这里的循环中，相当于放置一个 Safepoint 呢，以达到避免 GC 线程长时间等待，相当于让可数循环变成不可数循环那样，无需等待整个循环跑完。
>
>   ```java
>   //通常num需要两个子线程结束才能给出结果
>   public class MainTest {
>   
>       public static AtomicInteger num = new AtomicInteger(0);
>   
>       public static void main(String[] args) throws InterruptedException {
>           Runnable runnable=()->{
>               for (int i = 0; i < 1000000000; i++) {
>                   num.getAndAdd(1);
>               }
>               System.out.println(Thread.currentThread().getName()+"执行结束!");
>           };
>   
>           Thread t1 = new Thread(runnable);
>           Thread t2 = new Thread(runnable);
>           t1.start();
>           t2.start();
>           Thread.sleep(1000);
>           System.out.println("num = " + num);
>       }
>   }
>   ```
>
>   1）但是把int修改成long变成不可数循环即可
>
>   ```java
>   //通常num需要两个子线程结束才能给出结果
>   public class MainTest {
>   
>       public static AtomicInteger num = new AtomicInteger(0);
>   
>       public static void main(String[] args) throws InterruptedException {
>           Runnable runnable=()->{
>               //这里把int修改成long，变成不可数循环，进入安全点无需等待循环结束
>               for (long i = 0; i < 1000000000; i++) {
>                   num.getAndAdd(1);
>               }
>               System.out.println(Thread.currentThread().getName()+"执行结束!");
>           };
>   
>           Thread t1 = new Thread(runnable);
>           Thread t2 = new Thread(runnable);
>           t1.start();
>           t2.start();
>           Thread.sleep(1000);
>           System.out.println("num = " + num);
>       }
>   }
>   ```
>
>   2）或者是把引入Thread.sleep(0)，可以达到同样效果，无需等待循环结束
>
>   ```java
>   //引入Thread.sleep(0)
>   public class MainTest {
>   
>       public static AtomicInteger num = new AtomicInteger(0);
>   
>       public static void main(String[] args) throws InterruptedException {
>           Runnable runnable=()->{
>               for (int i = 0; i < 1000000000; i++) {
>                   num.getAndAdd(1);
>                   // prevent gc
>           		if (i % 1000 == 0) {
>               		try {
>                   		Thread.sleep(0);
>               		} catch (InterruptedException e) {
>                   		e.printStackTrace();
>               		}
>           		}
>               }
>               System.out.println(Thread.currentThread().getName()+"执行结束!");
>           };
>   
>           Thread t1 = new Thread(runnable);
>           Thread t2 = new Thread(runnable);
>           t1.start();
>           t2.start();
>           Thread.sleep(1000);
>           System.out.println("num = " + num);
>       }
>   }
>   ```



# 2获取Tid

## 2.1从源码中找线索

```java
public class Thread implements Runnable {

    private volatile long nativePeer;
    private volatile String name;
	private int            priority;
    /* For autonumbering anonymous threads. */
    private static int threadInitNumber;
    private static synchronized int nextThreadNum() {
        return threadInitNumber++;
    }

    private long tid;
    private static long threadSeqNumber;
	//thread构造方法
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
    
    public Thread(String name) {
        init(null, null, name, 0);
    }
	//初始化Thread
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;
        init2(parent);
        this.stackSize = stackSize;

        //这里面的tid不是真实linux的tid，这个是按次序在java层分配的id
        tid = nextThreadID();
    }
    
}
```

### 2.1.1**startThread**

```java
//startThread创建线程工作
public synchronized void start() {
    //如果线程已启动过，抛出异常 
    if (started)
        throw new IllegalThreadStateException();
    //添加线程到 group中
    group.add(this);

    started = false;
    try {
        //调用native 函数进行真正的线程创建工作，传入2个参数，分别为线程函数栈大小及daemon书写
        nativeCreate(this, stackSize, daemon);
        started = true;
    } finally {
        try {
            //创建流程失败的逻辑处理
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
        }
    }
}
```

### 2.1.2**Thread_nativeCreate**

调用到真实的`thread`线程创建工作

```c++
//art/runtime/thread.cc
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  ...
}
```

这里面涉及到的代码比较复杂，会按照功能划分三段new Thread、pthread_create、Thread::Init

```c++
//art/runtime/thread.h
//tid保存的地方tls32_结构体中
struct PACKED(4) tls_32bit_sized_values {
    typedef uint32_t bool32_t;
    union StateAndFlags state_and_flags;
    int suspend_count GUARDED_BY(Locks::thread_suspend_count_lock_);
    int debug_suspend_count GUARDED_BY(Locks::thread_suspend_count_lock_);
    uint32_t thin_lock_thread_id;
    // System thread id.
    uint32_t tid;
    ...
} tls32_;
```

我们现在知道 `tid`是保存在 ` tls32_`结构体 中，并且其位于` Thread`对象的开头，以`Android11`源码来看， `tid` 处于第`16`个字节(位于 `state_and_flags`、`suspend_count`、`debug_suspend_count`、`think_lock_thread_id`之后)开头。 因此我们只要能够获取  `native`层 该`thread`对象的指针 就可以通过 内存偏移的方式 获取`tid`。

## 2.2获取 Java Thread对象对应的tid

```java
package com.example.nativetid;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import com.example.nativetid.databinding.ActivityMainBinding;

import java.lang.reflect.Field;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    private ActivityMainBinding binding;
    private final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        // Example of a call to a native method
        TextView tv = binding.sampleText;
        tv.setText(stringFromJNI());
        try {
            init();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    private void init() throws IllegalAccessException {
        Thread thread = new Thread("demo_thread"){
            @Override
            public void run() {
                try {
                    sleep(10 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.d(TAG, "->>>Thread对象的run方法被执行了");
            }
        };
        //线程启动
        thread.start();
        long peer = getNativePeer(thread);
        Log.d(TAG, "->>>tid = " + getTid(peer));

    }

    public static final long getNativePeer(Thread t)throws IllegalAccessException{
        try {
            Field nativePeerField = Thread.class.getDeclaredField("nativePeer");
            nativePeerField.setAccessible(true);
            Long nativePeer = (Long) nativePeerField.get(t);
            return nativePeer;
        } catch (NoSuchFieldException e) {
            throw new IllegalAccessException("failed to get nativePeer value");
        } catch (IllegalAccessException e) {
            throw e;
        }
    }
    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
    public static native int getTid(long nativePeer);
}
```

笔者以红米手机测试ok

{{< image classes="fancybox center fig-100" src="/concurrent/thread_1.png" thumbnail="/concurrent/thread_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/concurrent/thread_2.png" thumbnail="/concurrent/thread_2.png" title="">}}



# 本文源码

下载点击[这里](https://github.com/YangYang48/project/tree/master/Thread)

# 参考

[[1] 卓修武_, Android虚拟机线程启动过程解析, 获取Java线程真实线程Id的方式, 2022.](https://juejin.cn/post/7138690370694545415)

[[2] why技术, 没有二十年功力，写不出这一行“看似无用”的代码！, 2022.](https://mp.weixin.qq.com/s/he9Y3saAIYIMaU4XAHDPPw)