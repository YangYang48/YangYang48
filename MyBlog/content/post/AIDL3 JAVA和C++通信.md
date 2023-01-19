---
title: "AIDL3 JAVA和C++通信"
date: 2022-03-08
thumbnailImagePosition: left
thumbnailImage: aidl/aidl3_thumb.png
coverImage: aidl/aidl31_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- AIDL
- 2022
- March 
tags:
- Android
- IPC
- Binder
showSocial: false
---


祝各位女神3.8快乐~最近在阅读Android源码的过程中再次遇到AIDL。和以往不同，这次是Java层和c++层的相互调用，跟以往App端的两个Java进程的IPC通信有区别。

<!--more-->
笔者在先前已经简单的分析了AIDL，但是AIDL远不止这么JAVA-JAVA通信这么简单，事实上Android系统中可以通过binder，实现双向IPC通信，可以是JAVA - JAVA 、 JAVA - C++ 、 C++ - C++ ，都可以通过Binder进行IPC双向通信。本文主要通过JAVA-C++的通信更加深入了解AIDL。



# 1.Binder简介

Binder作为Android系统提供的一种IPC机制，这是Android系统中最重要的组成，也是最难理解的一块知识点，错综复杂。

Binder是Android整个系统的核心，考虑复杂性，暂时对其具体实现过程简单提下，具体需要在Binder篇章深入研究。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_1.png" thumbnail="/aidl/aidl3_1.png" title="Binder系统架构图">}}

由于AIDL可以是C++层也可以是Java层，那么上述Binder通信的变体就会出现下面这几种。

1）两个App进程通信

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_2.png" thumbnail="/aidl/aidl3_2.png" title="两个App进程通信图">}}



2）一个App进程，一个Native进程通信

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_3.png" thumbnail="/aidl/aidl3_3.png" title="App和Native进程通信图">}}

3）两个Native进程通信

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_4.png" thumbnail="/aidl/aidl3_4.png" title="两个Native进程通信图">}}

结合代表性，本文主要对第二种情况的通信展开描述。

# 2.Binder核心类介绍和原理

## 2.1Binder框架

### 2.1.1Binder由四部分组成

Binder驱动：Binder的核心，实现底层操作

ServiceManager：守护进程，Binder服务的管理者

服务端：Binder的提供者Bn

客户端：Binder的使用者Bp

### 2.1.2Binder设计分层

Binder实现通过libbinder库实现。Binder源码点击[这里](https://github.com/YangYang48/project/tree/master/libbinder/frameworks/native/libs/binder)。

/frameworks/native/libs/binder

| /frameworks/native/libs/binder                               |
| ------------------------------------------------------------ |
| /frameworks/native/libs/binder/Binder.cpp                    |
| /frameworks/native/libs/binder/BpBinder.cpp                  |
| /frameworks/native/libs/binder/IInterface.cpp                |
| /frameworks/native/libs/binder/IPCThreadState.cpp            |
| /frameworks/native/libs/binder/ProcessState.cpp              |
| /frameworks/native/libs/binder/Parcel.cpp                    |
| /frameworks/native/libs/binder/include/binder/Binder.h       |
| /frameworks/native/libs/binder/include/binder/BpBinder.h     |
| /frameworks/native/libs/binder/include/binder/IBinder.h      |
| /frameworks/native/libs/binder/include/binder/IInterface.h   |
| /frameworks/native/libs/binder/include/binder/IPCThreadState.h |
| /frameworks/native/libs/binder/include/binder/ProcessState.h |
| /frameworks/native/libs/binder/include/binder/Parcel.h       |

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_5.png" thumbnail="/aidl/aidl3_5.png" title="">}}

## 2.2Binder核心类

IIterface类，BpInterface类，BnInterface类，BBinder类，BpBinder类，IBinder类

### 2.2.1各类之间的继承

各类的继承关系

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_6.png" thumbnail="/aidl/aidl3_6.png" title="各类的继承关系图">}}

简化成客户端和服务端的关系

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_7.png" thumbnail="/aidl/aidl3_7.png" title="客户端和服务端的关系图">}}

### 2.2.2服务端

以下为Android11源码分析

BnInterface类，**开发Binder服务必须要继承的类**

```c++
//frameworks/native/libs/binder/include/binder/IInterface.h#BnInterface
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};
```

BBinder类，服务类的核心。这里的IBinder类前面的clang::lto_visibility_public详见[这里](https://blog.csdn.net/superlee1125/article/details/114403368)。

```c++
//frameworks/native/libs/binder/include/binder/IBinder.h#transact
class [[clang::lto_visibility_public]] IBinder : public virtual RefBase{
    virtual status_t        transact(   uint32_t code,
                                 const Parcel& data,
                                 Parcel* reply,
                                 uint32_t flags = 0) = 0;
};
```

最终实现是在Binder

```c++
//frameworks/native/libs/binder/Binder.cpp#transact
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            err = pingBinder();
            break;
        case EXTENSION_TRANSACTION:
            err = reply->writeStrongBinder(getExtension());
            break;
        case DEBUG_PID_TRANSACTION:
            err = reply->writeInt32(getDebugPid());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }

    // In case this is being transacted on in the same process.
    if (reply != nullptr) {
        reply->setDataPosition(0);
    }

    return err;
}
```

其中BBinder负责和更底层的IPCThread交互，IPCThread会调用BBinder的transact方法，transact方法会调用到继承类中的onTransact方法实现交互。

因此，我们自己只需要对对BBinder的继承类重写onTransact方法，既可以实现数据与驱动交互。

```c++
//frameworks/native/libs/binder/Binder.cpp#onTransact
status_t BBinder::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t /*flags*/)
{
    switch (code) {
        case INTERFACE_TRANSACTION:
            reply->writeString16(getInterfaceDescriptor());
            return NO_ERROR;

        case DUMP_TRANSACTION: {
            int fd = data.readFileDescriptor();
            int argc = data.readInt32();
            Vector<String16> args;
            for (int i = 0; i < argc && data.dataAvail() > 0; i++) {
               args.add(data.readString16());
            }
            return dump(fd, args);
        }

        case SHELL_COMMAND_TRANSACTION: {
            int in = data.readFileDescriptor();
            int out = data.readFileDescriptor();
            int err = data.readFileDescriptor();
            int argc = data.readInt32();
            Vector<String16> args;
            for (int i = 0; i < argc && data.dataAvail() > 0; i++) {
               args.add(data.readString16());
            }
            sp<IShellCallback> shellCallback = IShellCallback::asInterface(
                    data.readStrongBinder());
            sp<IResultReceiver> resultReceiver = IResultReceiver::asInterface(
                    data.readStrongBinder());

            // XXX can't add virtuals until binaries are updated.
            //return shellCommand(in, out, err, args, resultReceiver);
            (void)in;
            (void)out;
            (void)err;

            if (resultReceiver != nullptr) {
                resultReceiver->send(INVALID_OPERATION);
            }

            return NO_ERROR;
        }

        case SYSPROPS_TRANSACTION: {
            report_sysprop_change();
            return NO_ERROR;
        }

        default:
            return UNKNOWN_TRANSACTION;
    }
}
```

### 2.2.3客户端

BpBinder

这个是客户端的核心

```c++
//frameworks/native/libs/binder/include/binder/IInterface.h#BpInterface
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
    explicit                    BpInterface(const sp<IBinder>& remote);

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};
```

与BnBinder中BnInterface类继承BBinder不同的是，BpInterface类不是继承BpBinder的，而是继承BpRefBase类的。这个可以更加灵活地进行应用开发，BpBinder更改对客户端实现无影响。

```c++
//frameworks/native/libs/binder/include/binder/Binder.h#BpRefBase
class BpRefBase : public virtual RefBase
{
protected:
    explicit                BpRefBase(const sp<IBinder>& o);
    virtual                 ~BpRefBase();
    virtual void            onFirstRef();
    virtual void            onLastStrongRef(const void* id);
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);

    inline  IBinder*        remote()                { return mRemote; }
    inline  IBinder*        remote() const          { return mRemote; }

private:
                            BpRefBase(const BpRefBase& o);
    BpRefBase&              operator=(const BpRefBase& o);

    IBinder* const          mRemote;
    RefBase::weakref_type*  mRefs;
    std::atomic<int32_t>    mState;
};
```

BpBinder

负责与底层IPCThread交互，**代理类**中的函数实现会最终调用IBinder的transact，transact调用IPCThread函数向驱动传递数据。

```c++
//frameworks/native/libs/binder/include/binder/BpBinder.h#transact
class BpBinder : public IBinder{
   virtual status_t    transact(   uint32_t code,
                             const Parcel& data,
                             Parcel* reply,
                             uint32_t flags = 0) final; 
};
```

最终实现在BpBinder.cpp

```c++
//frameworks/native/libs/binder/BpBinder.cpp#transact
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        ...
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}
```

Binder代理类

asInterface，实际上这个是宏定义出来的

```c++
//frameworks/native/libs/binder/include/binder/IInterface.h#DECLARE_META_INTERFACE
#define DECLARE_META_INTERFACE(INTERFACE)                               \
public:                                                                 \
    static const ::android::String16 descriptor;                        \
    static ::android::sp<I##INTERFACE> asInterface(                     \
            const ::android::sp<::android::IBinder>& obj);              \
    virtual const ::android::String16& getInterfaceDescriptor() const;  \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();                                            \
    static bool setDefaultImpl(std::unique_ptr<I##INTERFACE> impl);     \
    static const std::unique_ptr<I##INTERFACE>& getDefaultImpl();       \
private:                                                                \
    static std::unique_ptr<I##INTERFACE> default_impl;                  \
public:                                                                 \
```

具体实现

```c++
//frameworks/native/libs/binder/include/binder/IInterface.h#DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE
#define DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)\
    ...
    ::android::sp<I##INTERFACE> I##INTERFACE::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<I##INTERFACE> intr;                               \
        if (obj != nullptr) {                                           \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == nullptr) {                                      \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
```

考虑到上述queryLocalInterface的入参，可以是BBinder，也可以是BpBinder，那么BnInterface中的实现，返回的是BnInterface实例它本身。

```c++
//frameworks/native/libs/binder/include/binder/IInterface.h#BnInterface<INTERFACE>::queryLocalInterface
template<typename INTERFACE>
inline sp<IInterface> BnInterface<INTERFACE>::queryLocalInterface(
        const String16& _descriptor)
{
    if (_descriptor == INTERFACE::descriptor) return this;
    return nullptr;
}
```

BpInterface的本身没有queryLocalInterface方法，它的构造BpInterface(const sp<IBinder>& remote);中的参数是IBinder类型，最终调用到的是IBinder中的queryLocalInterface方法。也就是BpInterface间接调用BpBinder，BpBinder的实现是返回nullptr，那么一定会调用new Bp##INTERFACE(obj);这里。**客户端才能生成真正的代理类，服务端生成不了**。

```c++
//frameworks/native/libs/binder/Binder.cpp#IBinder::queryLocalInterface
sp<IInterface>  IBinder::queryLocalInterface(const String16& /*descriptor*/)
{
    return nullptr;
}
```

### 2.2.4总结

BBinder是服务端的核心，BpBinder是客户端的核心，两者都是IBinder派生出来的，其中服务端需要实现BnInterface，客户端需要实现BpInterface。

## 2.3Binder死亡通知

检测一个服务，是不是宕机或者已经挂死，可以通过死亡通知来检测。

## 2.3.1linkToDeath

IBinder提供了一个死亡通知接口linkToDeath。

其有三个参数，recipient为接收对象，必须自定义实现。

```c++
//frameworks/native/libs/binder/include/binder/IBinder.h#linkToDeath
class [[clang::lto_visibility_public]] IBinder : public virtual RefBase{
    virtual status_t        linkToDeath(const sp<DeathRecipient>& recipient,
                                        void* cookie = nullptr,
                                        uint32_t flags = 0) = 0;
};
```

这个类DeathRecipient就是接口，用于回调，需要重载binderDied

```c++
//frameworks/native/libs/binder/include/binder/IBinder.h#DeathRecipient
class DeathRecipient : public virtual RefBase
{
    public:
    virtual void binderDied(const wp<IBinder>& who) = 0;
};
```

考虑到IBinder，既可以是BBinder，也可以是BpBinder。他们又有分别的实现。

BBinder

```c++
//frameworks/native/libs/binder/Binder.cpp#linkToDeath
status_t BBinder::linkToDeath(
    const sp<DeathRecipient>& /*recipient*/, void* /*cookie*/,
    uint32_t /*flags*/)
{
    return INVALID_OPERATION;
}
```

BpBinder

```c++
//frameworks/native/libs/binder/BpBinder.cpp#linkToDeath
status_t BpBinder::linkToDeath(
    const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    Obituary ob;
    ob.recipient = recipient;
    ob.cookie = cookie;
    ob.flags = flags;

    LOG_ALWAYS_FATAL_IF(recipient == nullptr,
                        "linkToDeath(): recipient must be non-NULL");

    {
        AutoMutex _l(mLock);

        if (!mObitsSent) {
            if (!mObituaries) {
                mObituaries = new Vector<Obituary>;
                if (!mObituaries) {
                    return NO_MEMORY;
                }
                ALOGV("Requesting death notification: %p handle %d\n", this, mHandle);
                getWeakRefs()->incWeak(this);
                IPCThreadState* self = IPCThreadState::self();
                self->requestDeathNotification(mHandle, this);
                self->flushCommands();
            }
            ssize_t res = mObituaries->add(ob);
            return res >= (ssize_t)NO_ERROR ? (status_t)NO_ERROR : res;
        }
    }

    return DEAD_OBJECT;
}
```

因此，只能在客户端接收死亡通知。

### executeCommand

那么有了死亡通知的接口，那么谁来发送死亡通知呢？

这个死亡通知实际上是由驱动推送的，驱动检测某个服务死亡，会根据设置向应用推送消息。

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp#executeCommand
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    ...
    case BR_DEAD_BINDER:
        {
            BpBinder *proxy = (BpBinder*)mIn.readPointer();
            proxy->sendObituary();
            mOut.writeInt32(BC_DEAD_BINDER_DONE);
            mOut.writePointer((uintptr_t)proxy);
        } break;
    ...
    default:
        ALOGE("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break;
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }

    return result;
}
```

调用到BpBinder中的sendObituary方法

```c++
//frameworks/native/libs/binder/BpBinder.cpp#sendObituary
void BpBinder::sendObituary()
{
    ...
    if (obits != nullptr) {
        const size_t N = obits->size();
        for (size_t i=0; i<N; i++) {
            reportOneDeath(obits->itemAt(i));
        }

        delete obits;
    }
}
```

而reportOneDeath最终会调用到上述死亡通知篇章中的回调函数binderDied，这也是为什么该函数必须重载。

```c++
//frameworks/native/libs/binder/BpBinder.cpp#reportOneDeath
void BpBinder::reportOneDeath(const Obituary& obit)
{
    sp<DeathRecipient> recipient = obit.recipient.promote();
    ALOGV("Reporting death to recipient: %p\n", recipient.get());
    if (recipient == nullptr) return;

    recipient->binderDied(this);
}
```



# 3.AIDL实战

1.接口扩展

```c++
//ITestService.h
#include <IInterface.h>
#define TEST_GET 1
#define TEST_SET 2

namespace android{
Class ITestService : public IInterface{
public:
    DECLARE_META_INTERFACE(TestService);
    virtual int get() = 0;
    virtual void set(int value) = 0;
};
}
```

2.服务端

```c++
//BnTestService.h
namespace android{
class BnTestService : public BnInterface<ITestService>{
public:
    BnTestService();
public:
    int cnt = 0;
    virtual int get();
    virtual void set(int value);
    virtual status_t onTransact(uint32_t code,
                                const Parcel& data,
                                Parcel* reply,
                                uint32_t flags = 0);
};
}
```

3.客户端

```c++
#BpTestService.h
namespace android{
class BpTestService : public BpInterface<ITestService>{
public:
    BpTestService::BpTestService(const sp<IBinder>& impl) : BpInterface<ITestService>(impl);
public:
    virtual int get();
    virtual void set(int value);
};
}
```

3.Bn端的代码实现

```c++
//BnTestService.cpp
#include "BnTestService.h"
BnTestService::BnTestService()
{
}

status_t BnTestService::onTransact(uint32_t code,
                                const Parcel& data,
                                Parcel* reply,
                                uint32_t flags = 0)
{
    switch() {
        case TEST_GET: {
           //数据有效性检测 
           CHECK_INTERFACE(ITestService, data, reply);
           set(data->readInt());
           return NO_ERROR;
        } break;
        case TEST_SET: {
           //数据有效性检测 
           CHECK_INTERFACE(ITestService, data, reply);
           reply->writeInt(get());
           return NO_ERROR;
        } break;
        default:
            return IBinder::onTransact(code, data, reply, flag);
    }
}

int BnTestService::get()
{
    return cnt;
}

void BnTestService::set(int cnt)
{
    this->cnt = cnt;
}
```

4.Bp端的代码实现

```c++
namespace android{
IMPLEMENT_META_INTERFACE(ITestService, "android.test.ITestService");

BpTestService::BpTestService(const sp<IBinder>& impl) : BpInterface<ITestService>(impl)
{
}

int BpTestService::get()
{
    Parcel data, reply;
    remote()->transact(TEST_GET, data, &reply);
    return reply.readInt();
}

void BpTestService::set(int cnt)
{
    Parcel data, reply;
    data.writeInt(cnt);
    remote()->transact(TEST_SET, data, &reply);
}     
}
```

Bp中的remote()，就是mRemote，而mRemote是在Bp构造的时候传入的impl，即为BpBinder的一个实例。

```c++
//frameworks/native/libs/binder/include/binder/Binder.h#BpRefBase::remote
inline  IBinder*        remote()                { return mRemote; }
```



# 4.系统源码实例分析

事实上，源码中有大量的使用这类AIDL的方式。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_8.png" thumbnail="/aidl/aidl3_8.png" title="Android Camera AIDL图">}}

举一个比较常见的代码，Camera源码中Camera App调用到CameraService里面的功能，这个时候用得就是AIDL。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_12.png" thumbnail="/aidl/aidl3_12.png" title="Camera架构图">}}

## 4.1 addListener

这里调用了cameraService中的addListener方法，但实际上最终实现是在CameraService.cpp中的addListener函数。

```java
//base/core/java/android/hardware/camera2/CameraManager.java#connectCameraServiceLocked
private void connectCameraServiceLocked() {
    if (mCameraService != null || sCameraServiceDisabled) return;
    Log.i(TAG, "Connecting to camera service");

    IBinder cameraServiceBinder = ServiceManager.getService(CAMERA_SERVICE_BINDER_NAME);
    ICameraService cameraService = ICameraService.Stub.asInterface(cameraServiceBinder);
    ...
    try {
        CameraStatus[] cameraStatuses = cameraService.addListener(this);
        for (CameraStatus c : cameraStatuses) {
            onStatusChangedLocked(c.status, c.cameraId);

            if (c.unavailablePhysicalCameras != null) {
                for (String unavailPhysicalCamera : c.unavailablePhysicalCameras) {
                    onPhysicalCameraStatusChangedLocked(
                        ICameraServiceListener.STATUS_NOT_PRESENT,
                        c.cameraId, unavailPhysicalCamera);
                }
            }
        }
        mCameraService = cameraService;
    }
    ...
}
```

## 4.2 系统AIDL组成结构

### 4.2.1 AIDL定义

```aidl
//frameworks/av/camera/aidl/android/hardware/ICameraService.aidl
interface ICameraService {
    /**
     * Add listener for changes to camera device and flashlight state.
     *
     * Also returns the set of currently-known camera IDs and state of each device.
     * Adding a listener will trigger the torch status listener to fire for all
     * devices that have a flash unit.
     */
    CameraStatus[] addListener(ICameraServiceListener listener);
    ...
}
```

### 4.2.2 生成的AIDL的java接口

这部分需要编译frameworks之后才能在/out目录下生成对应的java文件。这个java文件跟在as定义之后生成的java文件类似。

```java
///out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/dotdot/av/camera.aidl/amdroid/hardware/ICameraService.java#ICmaeraService
public interface ICmaeraService extends android.os.IInterface
{
    ...
    private static class Proxy implements android.hardware.ICameraService
    {
        private android.os.IBinder mRemote;
        Proxy(android.os.IBinder remote)
        {
            mRemote = remote;
        }
        
        @override
        public android.hardware.CameraStatus[] addListener(android.hardware.ICameraServiceListener listener)
            throws android.os.RemoteException
        {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            android.hardware.CameraStatus[] _result;
            try
            {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeStrongBinder((((listener != null)) ? (listener.asBinder()) : (null)));
                mRemote.transact(Stub.TRANSACTION_addListener, _data, _reply, 0);
                _reply.readException();
                _result = _reply.createTypedArray(android.hardware.CameraStatus.CREATOR);
            }
            finally
            {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }
    }
    ...
}
```

### 4.2.3 生成的AIDL的c++接口

这里的接口主要有三个，分别是Bp，Bn和I(BpCameraService.h，BnCameraService.h，ICameraService.h)。

这里有两种对应的路径，分别对应32位(android_arm_armv7-a)和64位(android_arm64_armv8-a)，笔者这里以64位举例说明。

ICameraService.h

```c++
///out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm64_armv8_xxx/gen/aidl/android/hardware/ICameraService.h
Class ICameraService : public ::android::IInterface
{
public:
    DECLARE_META_INTERFACE(CameraService);
    virtual ::android::binder::Status addListener(const ::android::sp<::android::hardware::ICameraServiceListener>& listener, ::std::vector<::android::hardware::CameraStatus>* _aidl_return) = 0;
    enum Call{
        ADDLISTENER = ::android::IBinder::FIRST_CALL_TRANSACTION + 5;
    };
    ...
};
```

BpCameraService.h

```c++
///out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm64_armv8_xxx/gen/aidl/android/hardware/BpCameraService.h
class BpCameraService : public ::android::BnInterface<ICameraService>
{
public:
    explicit BpCameraService(const ::android::sp<::android::IBinder>& _aidl_impl;
    virtual ~BpCameraService() = default;
    ::android::binder::Status addListener(const ::android::sp<::android::hardware::ICameraServiceListener>& listener, ::std::vector<::android::hardware::CameraStatus>* _aidl_return) override;
    ...
};
```

BnCameraService.h

```c++
///out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm64_armv8_xxx/gen/aidl/android/hardware/BnCameraService.h
class BnCameraService : public ::android::BnInterface<ICameraService>
{
public:
    ::android::status_t BnCameraService::onTransact(uint32_t _aidl_code, 
                                                    const ::android::Parcel& _aidl_data,
                                                    ::android::Parcel* _aidl_reply,
                                                    uint32_t _aidl_flags = 0) override;
};
```



### 4.2.4生成的AIDL的c++实现

```c++
///out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm64_armv8_xxx/gen/aidl/frameworks/av/camera/aidl/android/hardware/ICameraService.cpp
::android::status_t BnCameraService::onTransact(uint32_t _aidl_code, 
                                                const ::android::Parcel& _aidl_data,
                                                ::android::Parcel* _aidl_reply,
                                                uint32_t _aidl_flags){
    ::android::status_t _aidl_ret_status = ::android::OK;
    switch (_aidl_code){
        case Call::ADDLISTENER:
            {
                ::android::sp<::android::hardware::ICameraServiceListener> in_listener;
                ::std::vector<::android::hardware::CameraStatus> _aidl_return;
                if (!(_aidl_data.checkInterface(this))){
                    _aidl_ret_status = ::android::BAD_TYPE;
                    break;
                }
                _aidl_ret_status = _aidl_data.readStrongBinder(&in_listener);
                if (((_aidl_ret_status) != (::android::OK))){
                    break;
                }

                ::android::Binder::Status _aidl_status(addListener(in_listener, &_aidl_return));
                _aidl_ret_status = _aidl_status.writeToParcel(_aidl_reply);

                if (((_aidl_ret_status) != (::android::OK))){
                    break;
                }
                if (!_aidl_status.isOk()){
                    break;
                }
                _aidl_ret_status = _aidl_reply->writeParcelableVector(_aidl_return
                                                                     );
                if (((_aidl_ret_status) != (::android::OK))){
                    break;
                }
            }break; 
    }
}
```



### 4.2.5最终的服务实现

最终会调用到cpp继承的库libcameraservice，这里的libcameraservice中有很多目录。这里面会分别对应api接口和device接口，而调用这些借口的正是CameraService，所以我们这个libcameraservice这个库可以看做成由CameraService作为外观者模式，其余函数用于真正实现的库。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_9.png" thumbnail="/aidl/aidl3_9.png" title="libcameraservice树状结构图">}}

```c++
//frameworks/av/services/camera/libcameraservice/CameraService.h
class CameraService :
    public BinderService<CameraService>,
    public virtual ::android::hardware::BnCameraService,
    public virtual IBinder::DeathRecipient,
    public virtual CameraProviderManager::StatusListener
    {
        virtual binder::Status    addListener(const sp<hardware::ICameraServiceListener>& listener,
                                              /*out*/
                                              std::vector<hardware::CameraStatus>* cameraStatuses);
        ...
};
```

最终实现是在这里CameraService.cpp，具体里面的实现内容就不展开描述，主要是梳理aidl的流程。

```c++
//frameworks/av/services/camera/libcameraservice/CameraService.cpp
Status CameraService::addListener(const sp<ICameraServiceListener>& listener,
        /*out*/
        std::vector<hardware::CameraStatus> *cameraStatuses) {
    return addListenerHelper(listener, cameraStatuses);
}
```

## 4.3总结

系统源码Camrea Frameworks中的框架的调用，是从上层的app应用调用到sdk jar包中，这个jar包调用到CameraManager，而CameraManager内部去通过aidl方式获取CameraService的实例，并通过这个实例调用addListener方法，最终这个方法调用到libcameraservice中的CameraService.cpp。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_10.png" thumbnail="/aidl/aidl3_10.png" title="Camera aidl继承关系图">}}

除此之外，更加细节的调用如下图所示。

{{< image classes="fancybox center fig-100" src="/aidl/aidl3_11.png" thumbnail="/aidl/aidl3_11.png" title="Camera aidl调用关系">}}

# 参考

[[1] gcwl2016, Android系统中通过binder(AIDL)进行跨层IPC通信, 2019.](https://blog.csdn.net/gcwl2016/article/details/88691049)

[[2] i加加, （五十七）Android O WiFi的扫描流程梳理续——梳理java与c++之间的aidl-cpp通信, 2018.](https://jiatai.blog.csdn.net/article/details/80945447)

[[3] 飞同小可,android系统开发binder调用（C++和java相互调用), 2018.](https://blog.csdn.net/po__oq/article/details/80985658)

[[4] aaajj, c++层使用和编译aidl文件例子, 2019.](https://blog.csdn.net/aaajj/article/details/87924692)

[[5] SherlockCharlie, Android8.0 Binder之面向系统服务(二), 2018.](https://blog.csdn.net/u013928208/article/details/81190552)

[[6] 躬行之, Android进阶之AIDL的使用详解, 2018.]()

[[7] 袁辉辉, Binder系列—开篇, 2015.](http://gityuan.com/2015/10/31/binder-prepare/)

[[8] rockmanlc, IBinder类前面的clang::lto_visibility_public, 2021.](https://blog.csdn.net/superlee1125/article/details/114403368)

[[9] syfchao, Android基础框架Binder应用开发（一)-BnBp服务实现, 2021.](https://www.bilibili.com/video/BV1dN411d7V7)

[[10] 程序员Android, Camera2 / HAL3 架构了解下,2020](https://blog.csdn.net/wjky2014/article/details/108722509)

[[11] zhuyong006, Camera 初始化(Open)一（FrameWork -> Hal）,2019](https://blog.csdn.net/zhuyong006/article/details/102519200)