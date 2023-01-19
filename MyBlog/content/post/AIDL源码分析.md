---
title: "AIDL源码分析"
date: 2021-11-14
thumbnailImagePosition: left
thumbnailImage: aidl/aidl2_thumb.jpg
coverImage: aidl/aidl2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- AIDL
- 2021
- November
tags:
- Android
- IPC
- 源码
showSocial: false
---

aidl一种android接口描述语言，本文主要是对.aidl文件自动生成的.java文件的具体源码进行分析，描述AIDL生成的java类细节。

<!--more-->
# AIDL实例分析

## 1.客户端的操作

1) 获取aidl的对象

2) 通过获取的对象调用方法（1,2，3...）

```kotlin
mContext?.bindService(aidlIntent, mAidlConnect, Context.BIND_AUTO_CREATE)

private var mAidlConnect: ServiceConnection = object:ServiceConnection{
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        //1.获取aidl的对象
        mAidlClient = IDataUpdate.Stub.asInterface(service)
        initView()
    }

    override fun onServiceDisconnected(name: ComponentName?) {
        Log.d(TAG, "->>>mAidl disConnect")
        mAidlClient = null
    }
}

private fun initView() {
    //2.通过获取的对象调用方法（1,2，3...）
    var ret = mAidlClient?.StartUp()
    for (i in 0  .. 100)
    {
        mAidlClient?.DialogUpdate(i)
    }
}
```

## 2.服务端的操作

3) 实现aidl.stub类（继承或者内部匿名的方式）

4) 回调中实现方法（1,2,3...）

```kotlin
//1.实现aidl.stub类（继承或者内部匿名的方式）
private val dataUpdateBinder: IDataUpdate.Stub  = object : IDataUpdate.Stub(){
    //2.回调中实现方法（1,2,3...）
    override fun StartUp(): Int {
        return 0
    }

    override fun DialogUpdate(progress: Int) {
        if (!DialogFlag) return
        dataUpdate(progress)
    }

}
```

服务端和客户端之间是通过IDataUpdate.java来做到通信的



# AIDL生成的java文件

首先列出这个java文件的组成，如下图所示，可以看到IDateUpdate.java中除了需要实现的两个方法DialogUpdate和StartUp之外，还有两个类，都实现了IDateUpdate.java的接口。

{{< image classes="fancybox center fig-100" src="/aidl/aidl2_3.png" thumbnail="/aidl/aidl2_3.png" title="IDateUpdate.java组成">}}

## 1.Default类

Default类里面就是一个空操作，这里不展开描述

```java
/** Default implementation for IDataUpdate. */
public static class Default implements com.example.aidl.IDataUpdate
{
    @Override public int StartUp() throws android.os.RemoteException{return 0;}
    @Override public void DialogUpdate(int progress) throws android.os.RemoteException{}
    @Override public android.os.IBinder asBinder() {return null;}
}
```

## 2.Stub类

这里最重要的是Stub类，实现了IDateUpdate.java的接口之外，还继承了Binder，并且可以从IDateUpdate.java整体的结构中看到内部还有一个Proxy.java，同样也实现了IDateUpdate.java的接口。

```java
public static abstract class Stub extends android.os.Binder implements com.example.aidl.IDataUpdate
{
    private static final java.lang.String DESCRIPTOR = "com.example.aidl.IDataUpdate";
    static final int TRANSACTION_StartUp = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_DialogUpdate = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    public Stub(){this.attachInterface(this, DESCRIPTOR);}
    public static com.example.aidl.IDataUpdate asInterface(android.os.IBinder obj){...}
    @Override public android.os.IBinder asBinder(){return this;}
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException{...}
    //内部类，实现了IDataUpdate接口
    private static class Proxy implements com.example.aidl.IDataUpdate{...}
    public static boolean setDefaultImpl(com.example.aidl.IDataUpdate impl) {...}
    public static com.example.aidl.IDataUpdate getDefaultImpl() {return Stub.Proxy.sDefaultImpl;}
}
```



# AIDL源码流程

## 1.从客户端开始，首先获取aidl的对象

### 1）判断是否是跨进程

如果是就通过Proxy代理类处理，不然的话就无须Binder。

```java
//获取aidl对象，这里的obj是从onServiceConnected传过来的IBinder对象
public static com.example.aidl.IDataUpdate asInterface(android.os.IBinder obj)
{
    if ((obj==null)) {
        return null;
    }
    //查看是否是跨进程,如果是本地就无须进行Binder
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin!=null)&&(iin instanceof com.example.aidl.IDataUpdate))) {
        return ((com.example.aidl.IDataUpdate)iin);
    }
    //属于跨进程，就需要通过Proxy类代理
    return new com.example.aidl.IDataUpdate.Stub.Proxy(obj);
}
```

### 2）queryLocalInterface

这里主要用来检查是否为跨进程，**注意两点**1.传入值descriptor 2.mOwner，这个是Binder的私有属性，后面服务端会阐述作用。

```java
//Binder.java#queryLocalInterface
//这里注意两点1。传入值descriptor 2.mOwner，这个是Binder的私有属性
public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
    if (mDescriptor != null && mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}
```

### 3）完成检查之后，调用Proxy的构造

这个构造实际上相当于是Stub对象在内部Proxy类进行**注入**操作。

```java
private android.os.IBinder mRemote;
Proxy(android.os.IBinder remote)
{
    mRemote = remote;
}
```

最终返回的aidl的对象是Proxy中的mRemote



## 2.通过获取的对象调用方法（1,2，3...）

### 1）客户端用IBinder调用方法

因为实际上的对象就是Proxy类的IBinder，即跨进程的调用方法事件交给代理类。（即客户端调用，在Proxy实现）

代理类中有两个包，_data和  _reply。可以分别看做是行李包和纪念品包。行李包的特点是从客户端发起向服务端发送的数据，纪念品包是从服务端发起向客户端发送的数据。

```java
@Override public int StartUp() throws android.os.RemoteException
{
    //android.os.Parcel.obtain这个用到了线程池的操作，跟Message的获取类似，属于享元模式
    //行李包
    android.os.Parcel _data = android.os.Parcel.obtain();
    //纪念品包
    android.os.Parcel _reply = android.os.Parcel.obtain();
    int _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        boolean _status = mRemote.transact(Stub.TRANSACTION_StartUp, _data, _reply, 0);
        if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().StartUp();
        }
        _reply.readException();
        _result = _reply.readInt();
    }
    finally {
        //清空、回收parcel对象的内存
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
@Override public void DialogUpdate(int progress) throws android.os.RemoteException
{
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeInt(progress);
        boolean _status = mRemote.transact(Stub.TRANSACTION_DialogUpdate, _data, _reply, 0);
        if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().DialogUpdate(progress);
            return;
        }
        _reply.readException();
    }
    finally {
        _reply.recycle();
        _data.recycle();
    }
}
```

然后调用到transact会挂起客户端线程，0是同步，1是异步，也可以用android.os.IBinder.FLAG_ONEWAY表示。默认AIDL为同步操作，需要等待_result的返回值，但是异步操作是不需要 _reply的。调用结束之后，每次都会清空、回收parcel对象的内存。



> **不要在服务端oneway接口中处理耗时操作**，一旦用于高频调用，服务端又处理耗时，再偶尔碰上cpu负荷高，很可能会发生其他关键调用**偶现失败的隐蔽问题，而且很难复现**，隐患就一直在那儿了。具体原因分析[请点击](https://www.jianshu.com/p/4c8d346185cb)。



### 2）transact方法

通过Proxy中的transact方法调用，最终调用到Binder.java里面的transact方法，具体的类似Parcel中data操作的setDataPosition的方法含义详见下表，不具体展开说明。

表1 Parcel一些常用的方法

|    方法名称     | 作用                                                         |
| :-------------: | :----------------------------------------------------------- |
|     obtain      | 获得一个新的parcel ，相当于new一个对象                       |
|    dataSize     | 得到当前parcel对象的实际存储空间                             |
|  dataCapacity   | 得到当前parcel对象的已分配的存储空间, >=dataSize()值 (以空间换时间) |
|   dataPostion   | 获得当前parcel对象的偏移量(类似于文件流指针的偏移量)         |
| setDataPosition | 设置偏移量                                                   |
|     recyle      | 清空、回收parcel对象的内存                                   |
|    writeInt     | 写入一个整数                                                 |
|   writeFloat    | 写入一个浮点数                                               |
|   writeDouble   | 写入一个双精度数                                             |
|   writeString   | 写入一个字符串                                               |
| writeException  | Parcel队头写入“无异常“                                       |
|  readException  | 在Parcel队头读取，若读取值为异常，则抛出该异常；否则，程序正常运行 |



可以看到transact方法会调用到onTransact方法，因为这个this是从Proxy传过来，而Proxy的操作的IBinder对象实际是Stub中的IBinder对象，即最终调用到的onTransact，实际上是在Stub中回调的onTransact的方法。

```java
//Binder.java#transact
public final boolean transact(int code, @NonNull Parcel data, @Nullable Parcel reply,
                              int flags) throws RemoteException {
    if (false) Log.v("Binder", "Transact: " + code + " to " + this);

    if (data != null) {
        data.setDataPosition(0);
    }
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```



### 3）onTransact

```java
//IDataUpdate.java中的Stub类#onTransact
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
    java.lang.String descriptor = DESCRIPTOR;
    switch (code)
    {
        case INTERFACE_TRANSACTION:
            {
                reply.writeString(descriptor);
                return true;
            }
        case TRANSACTION_StartUp:
            {
                data.enforceInterface(descriptor);
                int _result = this.StartUp();
                reply.writeNoException();
                reply.writeInt(_result);
                return true;
            }
        case TRANSACTION_DialogUpdate:
            {
                data.enforceInterface(descriptor);
                int _arg0;
                _arg0 = data.readInt();
                this.DialogUpdate(_arg0);
                reply.writeNoException();
                return true;
            }
        default:
            {
                return super.onTransact(code, data, reply, flags);
            }
    }
}
```

可以看到通过Proxy对IBinder操作，传过来的是一系列的code，对应的code进行处理，通常会返回true，在Stub类里面找不到对应的code，就回到Binder.java中onTransact去查找对应的code。

而code里边的处理是和服务端紧密联系的，并且如果客户端需要有返回值这里就会会返回。

通过上面的代码可以得知实际上调用到了int _result = this.StartUp()和this.DialogUpdate(arg0)，**其中这里面的this就是Stub对象**




## 3.服务端的操作

实现aidl.stub类 ----继承或者内部匿名的方式

这里的例子是服务端通过内部匿名类的形式实现，具体看上面的服务端实现操作。

```kotlin
private val dataUpdateBinder: IDataUpdate.Stub  = object : IDataUpdate.Stub(){}
```

这个实际上做了两件事创建了一个匿名内部类，并且调用了IDataUpdate.Stub()的构造方法

```java
//IDataUpdate.java中的Stub类#Stub
private static final java.lang.String DESCRIPTOR = "com.example.aidl.IDataUpdate";
/** Construct the stub at attach it to the interface. */
public Stub()
{
    this.attachInterface(this, DESCRIPTOR);
}
```

可以看到实际上构造方法，传入了两个参数到attachInterface方法中，一个是Stub对象本身，另一个定义的AIDL类名。

```java
//Binder.java#attachInterface
public void attachInterface(@Nullable IInterface owner, @Nullable String descriptor) {
    mOwner = owner;
    mDescriptor = descriptor;
}
```

可以看到实际上这里传入的mOwner是Stub对象，mDescriptor是定义的AIDL类名。

结合上述queryLocalInterface中遗留下来的1.传入值descriptor 2.mOwner，

现在可以知道其实queryLocalInterface查询做了两件事：

1.优先得知mOwner是否为空，如果为空，则不是两个进程

2.比对AIDL类名是否相同（因此AIDL文件需要对应**放在相同目录**）



## 4.回调中实现方法（1,2,3...）

这个对象正好是IDataUpdate.Stub对象，跟上面在Stub类中onTransact的处理为Stub对象吻合，即可以知道从Stub类中调用的onTransact方法，真正的实现是服务端。

```java
private val dataUpdateBinder: IDataUpdate.Stub  = object : IDataUpdate.Stub(){
    override fun StartUp(): Int {
        //真正实现的方法1
        return 0
    }

    override fun DialogUpdate(progress: Int) {
        //真正实现的方法2
        return

    }
}
```



# 总结

1.UML调用关系，下图所示

{{< image classes="fancybox center fig-100" src="/aidl/aidl2_2.png" thumbnail="/aidl/aidl2_2.png" title="AIDL UML图">}}

2.根据客户端和服务端的调用，如下图所示。

客户端和服务端之间为AIDL，AIDL的内部调用在红色边框内部，也是简洁明了。

实际上客户端和服务端的调用就是操作Parcel数据，这个是一种**共享内存**，即客户端和服务端可以抽象成虚线所示的调用关系。绿色部分为客户端进程，蓝色部分为服务端进程，红色部分为AIDL的流程部分。

{{< image classes="fancybox center fig-100" src="/aidl/aidl2_4.png" thumbnail="/aidl/aidl2_4.png" title="AIDL客户端和服务端的调用图">}}

3.ConnectService调用

另外，这里没有过多阐述连接Service的过程，这里简单通过时序图来表示了调用关系，具体深入详解ConnectService后续更新

{{< image classes="fancybox center fig-100" src="/aidl/aidl2_1.png" thumbnail="/aidl/aidl2_1.png" title="ConnectService调用时序图">}}



# 猜你喜欢

[AIDL的使用](https://yangyang48.github.io/2021/08/aidl%E4%BD%BF%E7%94%A8/)



# 参考

[[1] qinjuning, Android中Parcel的分析以及使用, 2011.](https://blog.csdn.net/qinjuning/article/details/6785517)

[[2] 杰杰_88, AIDL oneway 方法的隐患, 2020.](https://www.jianshu.com/p/4c8d346185cb)

[[3] kururunga, Android AIDL源码分析, 2017.](https://blog.csdn.net/kururunga/article/details/61418959)

[[4] 不会写代码的丝丽, AIDL源码分析, 2017.](https://blog.csdn.net/kururunga/article/details/61418959)

[[5] anlian523, AIDL oneway 以及in、out、inout参数的理解, 2019.](https://blog.csdn.net/anlian523/article/details/98476033)

