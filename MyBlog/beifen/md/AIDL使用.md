# AIDL使用



> AIDL（Android 接口定义语言），可以使用它定义一个app作为Client端与另一个app作为Server端通信（IPC）的编程接口，在 Android 中，进程之间无法共享内存（用户空间），不同进程之间的通信一般使用 AIDL 来处理。

aidl用通俗的话来阐述，就是定义了一个跨进程的接口，使得客户端app调用，服务端app来实现此接口。

# aidl使用流程

- 创建aidl接口
- 创建客户端app，用于调用aidl接口
- 创建服务端app，用于实现aidl接口

本文举一个比较常见的例子，仅供参考（Android的系统升级ota包校验的数据更新）。

目前有一个需求，跨进程的数据来更新UI

需要一个app可以直接展示画面，动态接收数据后来更新UI

需要另一个app可以拿到原始更新数据，并且发送动态数据



## 创建aidl接口

定义一个aidl接口

```kotlin
// IDataUpdate.aidl
package com.example.aidl;
/**
 * 这是一个aidl定义接口
 * 这个aidl接口主要用来发送数据
 */
interface IDataUpdate {
    //是否开启数据发送，0是不准备发送，1是准备发送
    int StartUp();
    void DialogUpdate(int progress);
}
```

默认的aidl是和java目录为同级目录，但是也可以认为修改aidl目录，笔者修改在java下面，/java/com.example.aidl

如果像笔者一样更换路径需要注意的点

> 1.app/build.gradle中修改sourceSets，如下所示
>
> ```java
> sourceSets {
>     main {
>         aidl.srcDirs = ['src/main/']
>     }
> }
> ```
>
> 2.aidl的client端和aidl的server端需要保证相同的路径
>
> ![aidl2](D:\hugo\MyBlog\static\aidl\aidl2.png)

笔者使用的as可以在定义完aidl文件之后，通过rebuild会自动生成一个同名的java文件

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 */
package com.example.aidl;
/**
 * 这是一个aidl定义接口
 * 这个aidl接口主要用来发送数据
 */
public interface IDataUpdate extends android.os.IInterface
{
  /** Default implementation for IDataUpdate. */
  public static class Default implements com.example.aidl.IDataUpdate
  {
    //是否开启数据发送，0是不准备发送，1是准备发送

    @Override public int StartUp() throws android.os.RemoteException
    {
      return 0;
    }
    @Override public void DialogUpdate(int progress) throws android.os.RemoteException
    {
    }
    @Override
    public android.os.IBinder asBinder() {
      return null;
    }
  }
  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements com.example.aidl.IDataUpdate
  {
    private static final java.lang.String DESCRIPTOR = "com.example.aidl.IDataUpdate";
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
    /**
     * Cast an IBinder object into an com.example.aidl.IDataUpdate interface,
     * generating a proxy if needed.
     */
    public static com.example.aidl.IDataUpdate asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.example.aidl.IDataUpdate))) {
        return ((com.example.aidl.IDataUpdate)iin);
      }
      return new com.example.aidl.IDataUpdate.Stub.Proxy(obj);
    }
    @Override public android.os.IBinder asBinder()
    {
      return this;
    }
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
    private static class Proxy implements com.example.aidl.IDataUpdate
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      //是否开启数据发送，0是不准备发送，1是准备发送

      @Override public int StartUp() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
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
      public static com.example.aidl.IDataUpdate sDefaultImpl;
    }
    static final int TRANSACTION_StartUp = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_DialogUpdate = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    public static boolean setDefaultImpl(com.example.aidl.IDataUpdate impl) {
      // Only one user of this interface can use this function
      // at a time. This is a heuristic to detect if two different
      // users in the same process use this function.
      if (Stub.Proxy.sDefaultImpl != null) {
        throw new IllegalStateException("setDefaultImpl() called twice");
      }
      if (impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }
    public static com.example.aidl.IDataUpdate getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }
  //是否开启数据发送，0是不准备发送，1是准备发送

  public int StartUp() throws android.os.RemoteException;
  public void DialogUpdate(int progress) throws android.os.RemoteException;
}
```

那么第一步的创建aidl就已经完成了

# 创建客户端app

跟aidl相关的就是绑定客户端的service

```kotlin
//主要用于绑定aidlservice，记录服务端的包名和service名
var aidlIntent: Intent = Intent()       aidlIntent.setClassName("com.example.myaidlserver", "com.example.myaidlserver.DataUpdateService")
mContext?.bindService(aidlIntent, mAidlConnect, Context.BIND_AUTO_CREATE)
```

通过成功连接服务端回调，完成aidl的绑定

```kotlin
//这个内部匿名类通过连接服务回调，通知客户端是否可以开始使用aidl
private var mAidlConnect: ServiceConnection = object:ServiceConnection{
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        Log.d(TAG, "->>>mAidlConnect")
        mAidlClient = IDataUpdate.Stub.asInterface(service)
        //开始使用aidl
        Log.d(TAG, "->>>init view")
        initView()
    }

    override fun onServiceDisconnected(name: ComponentName?) {
        Log.d(TAG, "->>>mAidl disConnect")
        //结束使用aidl
        mAidlClient = null
    }
}
```

使用aidl

```kotlin
//通过一个简单的调用来使用aidl
var ret = mAidlClient?.StartUp()
Log.d(TAG, "->>>sleep 2s")
Thread.sleep(2000)
if (1 == ret)
{
    for (i in 0  .. 100)
    {
        Log.d(TAG, "->>>send data update , date is $i")
        mAidlClient?.DialogUpdate(i)
        Thread.sleep(200)//控制发送时间间隔200ms一次
    }
}else
{
    Log.d(TAG, "->>>init failed")
}
```



# 创建服务端app

跟aidl相关的就是继承一个service用于实现aidl的接口

最后在实现onBind方法中返回aidl接口的对象

```kotlin
//一个内部匿名类实现aidl的接口
private val dataUpdateBinder: IDataUpdate.Stub  = object : IDataUpdate.Stub(){
    override fun StartUp(): Int {
        Log.d(TAG, "->>>begin to start")
        initView()
        if (DialogFlag)
        {
            Log.d(TAG, "->>>initDialog successful")
            return 1
        }
        return 0
    }

    override fun DialogUpdate(progress: Int) {
        if (!DialogFlag) return
        Log.d(TAG, "->>>data has update to $progress")
        dataUpdate(progress)
    }
}
```

```kotlin
//返回aidl对象
override fun onBind(intent: Intent): IBinder {
	return dataUpdateBinder
}
```



# 调式

调试过程中可能会出现的问题

service无法被调用

> 可能出现原因如下：
>
> 首先在manifest中确认service是否具备外部可调用，设置android:exported="true"
>
> 其次Android8不允许创建后台服务的情况下，推荐使用startForegroundService，并且还要注意Notification需要一个ChannelID
>
> ```kotlin
> try {
>             val CHANNEL_ONE_ID = "com.example.myaidlclient"
>             val CHANNEL_ONE_NAME = "Channel One"
>             var notificationChannel: NotificationChannel? = null
>             notificationChannel = NotificationChannel(
>                 CHANNEL_ONE_ID,
>                 CHANNEL_ONE_NAME, NotificationManager.IMPORTANCE_DEFAULT
>             )
>             val manager = (getSystemService(NOTIFICATION_SERVICE) as NotificationManager)!!
>             manager!!.createNotificationChannel(notificationChannel)
>             startForeground(1, NotificationCompat.Builder(this, CHANNEL_ONE_ID).build())
>         } catch (e: Exception) {
>             Log.e(TAG, e.message!!)
>         }
> ```
>
> 最后排查系统是否存在一个用于回收没有动作或者休眠的service的进程

服务端使用了一个Service和一个Activity的形式，数据更新到Service，无法直接通过UI展示，笔者在这里添加了一个内部接口来实现在Activity的SeekBar UI更新。

1. 首先将服务端app打开，保持展示UI

2. 通过adb命令，拉起客户端app进程

   ```java
   adb shell am start-foreground-service -n com.example.myaidlclient/.BootService
   ```

3. 查看客户端进程是否启动，启动后可直观观察服务端UI展示

具体效果如下

![aidl1](D:\hugo\MyBlog\static\aidl\aidl1.jpg)



[本文所有实例代码下载](https://github.com/YangYang48/project/tree/master/MyAIDLClient)

# 参考

[[1] 官方文档, Android 接口定义语言 (AIDL), 2019.](https://developer.android.google.cn/guide/components/aidl.html?hl=zh-cn#java)

[[2] 躬行之, Android进阶之AIDL的使用详解, 2018.](https://segmentfault.com/a/1190000015001951)

[[3] handsome黄, Service由浅到深——AIDL的使用方式, 2017.](https://www.cnblogs.com/huangjialin/p/7738104.html)

[[4] As新晋小白, [Android] AIDL详解, 2020.](https://blog.csdn.net/weixin_38423311/article/details/108677594)

[[5] 我是午饭, Android中AIDL的使用详解, 2016.](https://www.jianshu.com/p/d1fac6ccee98)

