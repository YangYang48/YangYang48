# Binder

binder是什么

- 进程间通讯机制
- 也是一个驱动
- Binder.java--->实现IBinder---跨进程的能力



机制：Binder是一种进程间通讯的机制

驱动：Binder是一个虚拟物理设备驱动

应用层：Binder是一个能发起通讯的Java类



一个app可以多个进程

进程的内存大小有限制

getprop dalvik.vm.heapsize



因此

突破进程内存限制，如图库占用内存过多，需要新开一个进程

功能稳定性：独立通信进程保持长连接的稳定性

规避系统内存泄漏：独立的webview进程阻隔内存泄漏

隔离风险：对于不稳定的功能放入独立进程，避免主进程崩溃

优点

内存----一个app，6g

getprop dalvik.vm.heapsize



风险隔离---每一个进程单独一个app



## Binder有什么优势

Linux进程间通讯

管道、信号量、socket、共享内存等等

![image-20210824225850112](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210824225850112.png)

binder小于共享内存，优于其他IPC进程通讯问题

Binder为每个app分配UID，同时支持实名（类似坐公交车可以找到站点就可以通信）和匿名（类似滴滴打车必须在滴滴系统中下单才会有对应的通信）



**系统服务**---实名

**个人服务**---匿名

ServiceManager注册过的就是实名

其他进程获取不到自定义的服务



## Binder是如何做到一次拷贝的

进程间和线程间通信不同，内存机制不同，线程共享内存，进程间是隔离的

进程间 通讯

进程之间的内存是隔离的

我们平时说的内存都是虚拟的，连续的，通过规则映射到物理内存中

内核空间都是映射到物理内存，关系类似多个地球仪--》一个地球，

![image-20210824231145522](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210824231145522.png)

## mmap原理讲解

Linux通过一个虚拟内存区域与一个磁盘对象关联起来，以初始化这个虚拟内存区域的内容，这个过程称为内存映射

mmap作用：让一块虚拟内存指向（映射）一块一致的物理内存

所以服务端的用户空间和内核空间可以映射到同一个物理内存，类似两个exe快捷方式和一个exe文件

## Binder机制如何跨进程

![image-20211007130355578](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007130355578.png)

数据发送方通过copy_from_user,将用户空间数据拷贝到内核空间，内核空间的虚拟内存和数据接收方的虚拟内存存在一个物理内存共享（映射），这个时候就不需要copy_to_user去复制了，数据接收方就可以拿到这个数据。

## 描述AIDL生成的java类细节

aidl：android接口描述语言

1.客户端bindservice之后查看是否是跨进程

![image-20211007154538414](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007154538414.png)



2.拿到代理对象的iPersonManager的对象或者服务

跨进程的事交给代理类，代理类中有两个包，分别是行李包和纪念品包，然后调用到transact会挂起客户端线程，0是同步，1是异步

![image-20211007155048613](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007155048613.png)

3.客户端通过Binder进入服务端，进入stub中的onTransact回调,通过code值回去调用相对应的方法，最后通过这个方法，会找到服务端的具体方法的实现。如果存在返回，会通过reply去返回。





## 四大组件底层通信机制

//广播是如何实现跨进程的

bindservice和onserviceconnect之间的联系

![image-20211007160322780](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007160322780.png)

## 为什么Intent不能传递大数据

binder通信数据传输大小是有共享内存决定的

![image-20211007162445442](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007162445442.png)

processstate.cpp

BINDER_VM_SIZE，同步极限大小为1M-8K

![image-20211007162530194](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007162530194.png)

kernel binder.c

bindmap（从mmap调用下来），异步处理为512K-4k

![image-20211007163720293](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211007163720293.png)
