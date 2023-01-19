---
title: "Android智能指针解析"
date: 2022-11-27
thumbnailImagePosition: left
thumbnailImage: AndroidSmartPtr/asp_thumb.jpg
coverImage: AndroidSmartPtr/asp_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 智能指针
- 2022
- November
tags:
- C++
- Java
- sp
- wp
- Binder
- aidl
- 运算符重载
showSocial: false
---

Android源码中有大量的智能指针，它们经常使用，通常一个类前会修饰sp或者wp，这种修饰即为智能指针。

<!--more-->
# Android智能指针解析

# 1简介

智能指针`C++11`就存在，是为了媲美`java`的`JVM`，完全不用管对象的回收问题。

`Android`的智能指针，巧妙的运用`C++`的基础特性，实现对象的自动释放。

# 2智能指针原理

## 2.1作用域

### 2.1.1栈空间

```c++
//demo1
class Yangyang
{};
//yangyang1的作用域开始
void demo1()
{
    Yangyang yangyang1;
    //yangyang2的作用域开始
    {
        Yangyang yangyang2;
    }
    //yangyang2的作用域结束
}
//yangyang1的作用域结束
```

可以看出局部变量的作用域实际上是最近一对括号包含的范围。

### 2.1.2堆空间

```c++
//demo2
class Yangyang
{};

void demo2()
{
    Yangyang stackYangyang;
    //pHeapYangyang是栈空间的变量，但是指向的是堆空间变量
    Yangyang* pHeapYangyang = new Yangyang;
    //指向堆空间变量的栈变量需要手动释放
    delete pHeapYangyang;
    //将指向堆空间的栈变量指向nullptr，防止造成野指针
    pHeapYangyang = nullptr;
}
```



# 3源码解析

源码里主要用到了`sp`（强引用指针）和`wp`（弱引用指针）的智能指针，当然还有其他类似`LightRefBase`（本文不做介绍）。

`Android`设计了强引用`sp`和弱引用`wp`，故实际对象的释放，可分为强引用控制和弱引用控制。前者是，指的是强引用数mStrong为0时，释放实际对象；后者是，则指的是弱引用数mWeak为0时，才释放实际对象。

{{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_1.png" thumbnail="/AndroidSmartPtr/asp_1.png" title="">}}

## 3.1举例说明

```c++
//demo3
class Yangyang : public RefBase
{
    Yangyang() : RefBase()
    {
        //可以显示调用RefBase构造
        cout << "Yangyang Constructor" << endl;
    }
    virtual ~Yangyang()
    {
        //声明virtual，可以从Yangyang派生
        cout << "Yangyang Destructor" << endl;
    }
};

void demo3()
{
    //new一个对象，pYangyang指向的是一个堆空间
    Yangyang* pYangyang = new Yangyang();
    //限定sp的作用域,利用sp的栈变量，spYangyang
    {
        sp<Yangyang> spYangyang(pYangyang);
        //限定wp的作用域，利用wp的栈变量，wpYangyang
        {
            wp<Yangyang> wpYangyang(pYangyang);
        }
        //wpYangyang作用域结束
    }
    //spYangyang作用域结束
}
```

结果显示

```c++
Yangyang::-----demo3--
Yangyang::Yangyang Constructor this(0x21127080)
Yangyang:No refs,strong count(0x10000000),weak count(0)
Yangyang:in sp scope Yangyang:onFirstRef,object(0x21127080)
Yangyang::After strong ref,strong count(1), weak count(1)
Yangyang:in wp scope Yangyang::After weak ref,strong count(1),weak count(2)
Yangyang:out wp scope Yangyang:release weak ref,strong count(1),weak count(1)
Yangyang::out sp scope Yangyang:onLastStrongRef,id(0x21712848)
Yangyang:Yangyang Destructor  this(0x21127080)
Yangyang::------
```

## 3.2RefBase

`RefBase`构造

```c++
//system/core/libutils/RefBase.cpp
//INITIAL_STRONG_VALUE 0x10000000
//这个传入的base就是实际对象pYangyang
//mFlags表示是否强引用，0是强引用，1是弱引用
explicit weakref_impl(RefBase* base)
: mStrong(INITIAL_STRONG_VALUE)
    , mWeak(0)
    , mBase(base)
    , mFlags(0)
{
}

//这里的this指的就是Yangyang的实例化对象pYangyang
RefBase::RefBase()
    : mRefs(new weakref_impl(this))
{
}
```

## 3.3sp构造

`sp`构造

```c++
//system/core/libutils/include/utils/StrongPointer.h
inline sp() : m_ptr(nullptr) { }

//这里的T就是Yangyang类，other就是pYangyang
//这里的this，实际上是spYangyang栈变量的地址
template<typename T>
sp<T>::sp(T* other)
    : m_ptr(other) {
    if (other) {
        check_not_on_stack(other);
        other->incStrong(this);
    }
}
```

因为`pYangyang`的`Yangyang`类，继承`RefBase`，所以可以找到基类`RefBase`对应的函数`incStrong`

```c++
//system/core/libutils/RefBase.cpp
//这里的mRefs实际上是new weakref_impl(pYangyang)
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    //1.弱引用增加1
    refs->incWeak(id);
    
    refs->addStrongRef(id);
    //2-1.fetch_add操作之后，refs->mStrong=0x10000001，c=0x10000000
    const int32_t c = refs->mStrong.fetch_add(1, std::memory_order_relaxed);
    ...
	//2-2.fetch_sub操作之后，refs->mStrong=1，c=0x10000001
    int32_t old __unused = refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE, std::memory_order_relaxed);

    //3调用onFirstRef，实际上调用的是pYangyang->onFirstRef
    refs->mBase->onFirstRef();
}
```



这里存在一些原子操作的函数`/system/core/libcutils/include/cutils/atomic.h`

|        函数名        |                        含义                        |
| :------------------: | :------------------------------------------------: |
| `android_atomic_inc` |    自增函数，返回值为旧值<br />`*ptr =*ptr + 1`    |
| `android_atomic_dec` |    自减函数，返回值为旧值<br />`*ptr =*ptr - 1`    |
| `android_atomic_add` | 加函数，返回值为旧值<br />`*ptr =*ptr + increment` |
| `android_atomic_and` |  位与函数，返回值为旧值<br />`*ptr =*ptr & value`  |
| `android_atomic_or`  |  位或函数，返回值为旧值<br />`*ptr =*ptr | value`  |
|     `fetch_add`      |  自增函数，返回值为旧值。<br />`*this =*this + 1`  |
|     `fetch_sub`      |  自减函数，返回值为旧值。<br />`*this =*this - 1`  |

`incWeak`

```c++
//system/core/libutils/RefBase.cpp
//这里的this指针是上面的mRefs，类型为weakref_impl* const
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->addWeakRef(id);
    //fetch_add之后，impl->mWeak=1，c=0
    const int32_t c __unused = impl->mWeak.fetch_add(1,
            std::memory_order_relaxed);
    ALOG_ASSERT(c >= 0, "incWeak called on %p after last weak ref", this);
}
```

因此，`sp`构造完成后，`mRefs`的弱引用数变为1，强引用数也变为1；第一次强引用时，回调`onFirstRef()`。

## 3.4wp构造

弱引用构造

```c++
//system/core/libutils/include/utils/RefBase.h
//这里的T就是Yangyang类，other就是pYangyang
//这里的this，实际上是wpYangyang栈变量的地址
template<typename T>
wp<T>::wp(T* other)
    : m_ptr(other)
{
    m_refs = other ? m_refs = other->createWeak(this) : nullptr;
}
```

这里的`demo`不存在会创建对象失败的情况，所以这里的`wp`中的`m_refs`为`other->createWeak(this)`，即`pYangyang->createWeak(&wpYangyang)`

```c++
//system/core/libutils/RefBase.cpp
//这里的返回值会返回当前的mRefs，是new weakref_impl(pYangyang)
RefBase::weakref_type* RefBase::createWeak(const void* id) const
{
    //这里的mRefs就是new weakref_impl(pYangyang)
    mRefs->incWeak(id);
    return mRefs;
}
```

`incWeak`在3.3中已经出现过了，作用是原子操作，让`mRefs->mWeak`增加1

调用`incWeak`，最终的影响是弱引用数+1。当前的`demo3`中，强引用数为1，弱引用数为2。

> `RefBase`构造的时候，`mFlags`默认为0，即`OBJECT_LIFETIME_STRONG`，强引用控制。设置为`OBJECT LIFETIME WEAK`时，为弱引用控制。可以通过`extendObjectLifetime`函数修改
>
> ```c++
> //system/core/libutils/RefBase.cpp
> //如果仅仅是弱引用，需要mFlags修改成1
> void RefBase::extendObjectLifetime(int32_t mode)
> {
>     // Must be happens-before ordered with respect to construction or any
>     // operation that could destroy the object.
>     mRefs->mFlags.fetch_or(mode, std::memory_order_relaxed);
> }
> ```

## 3.5wp析构

当`wp`作用域结束，弱引用开始析构

```c++
//system/core/libutils/include/utils/RefBase.h
//这里的m_ptr是pYangyang，因为还存在所以可以操作
//这里的m_refs是new weakref_impl(pYangyang)
//这里的this，实际上是wpYangyang栈变量的地址
template<typename T>
wp<T>::~wp()
{
    if (m_ptr) m_refs->decWeak(this);
}
```

由于`m_refs`的变量是基类，`m_refs`所指向的指针是派生类，这里的`decWeak`不是`virtual`标识，只能调用基类的`decWeak`函数

```c++
//system/core/libutils/RefBase.cpp
void RefBase::weakref_type::decWeak(const void* id)
{
    //这里的static_cast就是子父类的向下转换
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->removeWeakRef(id);
    //fetch_sub,mRefs->mWeak=0，c=1
    const int32_t c = impl->mWeak.fetch_sub(1, std::memory_order_release);
    //如果同时出现sp和wp的时候，mWeak的计数会是2，这里的c返回旧值也是2，直接返回
    if (c != 1) return;
    atomic_thread_fence(std::memory_order_acquire);

    int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
    //如果是强引用
    if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
        //如果强引用的mStrong元素时0x10000000,那么强引用错误
        if (impl->mStrong.load(std::memory_order_relaxed)
                == INITIAL_STRONG_VALUE) {
            ALOGW("RefBase: Object at %p lost last weak reference "
                    "before it had a strong reference", impl->mBase);
        } else {
            //如果强引用mStrong是其他值，释放impl，这里实际上释放的是mRefs
            //这里不能直接释放pYangyang，可能计数还没完全减掉
            delete impl;
        }
    } else {
        //如果只是弱引用，调用onLastWeakRef，这个impl->mBase实际上是pYangyang，即pYangyang->onLastWeakRef
        //然后把弱引用中的impl->mBase释放，即释放的是pYangyang，会调用pYangyang的析构函数
        impl->mBase->onLastWeakRef(id);
        delete impl->mBase;
    }
}
```

wp析构，情况比较复杂

- 弱引用数减1
- 最后一次弱引用时，如果是强引用控制，释放`mRfs`
- 若没有强引用，释放实际对象`pYangyang`

`demo3`来看，此时，强引用数为1，弱引用数为1，并没有任何释放



## 3.6sp析构

当`sp`作用域结束，强引用开始析构

```c++
//system/core/libutils/include/utils/StrongPointer.h
//针对sp析构，m_ptr实际上是pYangyang,所以调用的是pYangyang基类RefBase的decStrong函数
template<typename T>
sp<T>::~sp() {
    if (m_ptr)
        m_ptr->decStrong(this);
}
```

`decStrong`

```c++
//system/core/libutils/RefBase.cpp
//这里的mRefs
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id);
    //做减操作，mStrong=0，c=1
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release);

    if (c == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        //如果是强引用，调用onLastStrongRef，这个impl->mBase实际上是pYangyang，即pYangyang->onLastStrongRef
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        //如果是强引用
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            //这里的this指的是m_ptr，指向的是pYangyang，释放pYangyang，并调用其析构
            delete this;
        }
    }
	//做减操作，mWeak=0，c=1
    refs->decWeak(id);
}
```

## 3.7RefBase析构

析构完实际对象pYangyang之后，基类也同步析构

```c++
//system/core/libutils/RefBase.cpp
RefBase::~RefBase()
{
    int32_t flags = mRefs->mFlags.load(std::memory_order_relaxed);
	//如果是弱引用
    if ((flags & OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) {
        //弱引用计算为0，那么就把mRefs释放
        if (mRefs->mWeak.load(std::memory_order_relaxed) == 0) {
            delete mRefs;
        }
    } else if (mRefs->mStrong.load(std::memory_order_relaxed) == INITIAL_STRONG_VALUE) {
        //如果mStrong是0x10000000，会有RefBase相关的堆栈
        ALOGD("RefBase: Explicit destruction, weak count = %d (in %p)", mRefs->mWeak.load(), this);
        CallStack::logStack(LOG_TAG);
    }
    //清除mRefs。对未完成的wp无效
    const_cast<weakref_impl*&>(mRefs) = nullptr;
}
```

{{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_2.png" thumbnail="/AndroidSmartPtr/asp_2.png" title="">}}

## 3.8总结

1）仅有sp的时候（绿色是构造，红色是析构）

{{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_3.png" thumbnail="/AndroidSmartPtr/asp_3.png" title="">}}

2）仅有wp的时候（绿色是构造，红色是析构）

{{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_4.png" thumbnail="/AndroidSmartPtr/asp_4.png" title="">}}

3）有sp和wp的时候（绿色是构造，红色是析构）

{{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_7.png" thumbnail="/AndroidSmartPtr/asp_7.png" title="">}}

- 如果是demo3的场景，先出现析构wp，再出现析构sp场景

  ```c++
  void demo3()
  {
      //new一个对象，pYangyang指向的是一个堆空间
      Yangyang* pYangyang = new Yangyang();
      //限定sp的作用域,利用sp的栈变量，spYangyang
      {
          sp<Yangyang> spYangyang(pYangyang);
          //限定wp的作用域，利用wp的栈变量，wpYangyang
          {
              wp<Yangyang> wpYangyang(pYangyang);
          }
          //wpYangyang作用域结束
      }
      //spYangyang作用域结束
  }
  ```

  {{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_8.png" thumbnail="/AndroidSmartPtr/asp_8.png" title="">}}

- 如果是先析构sp，再出现析构wp的场景

  ```c++
  void demo4()
  {
      //new一个对象，pYangyang指向的是一个堆空间
      Yangyang* pYangyang = new Yangyang();
      //限wp的作用域,利用wp的栈变量，wpYangyang
      {
          wp<Yangyang> wpYangyang(pYangyang);
          //限定sp的作用域，利用sp的栈变量，spYangyang
          {
              sp<Yangyang> spYangyang(pYangyang);
          }
          //spYangyang作用域结束
      }
      //wpYangyang作用域结束
  }
  ```

  {{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_9.png" thumbnail="/AndroidSmartPtr/asp_9.png" title="">}}

# 4智能指针使用

源码中大量使用`sp`和`wp`，而且用于很广泛，也无需考虑是否足要去释放

```c++
//frameworks/av/services/camera/libcameraservice/CameraService.cpp
void CameraService::updateProxyDeviceState(int newState,
        const String8& cameraId, int facing, const String16& clientName, int apiLevel) {
    sp<ICameraServiceProxy> proxyBinder = getCameraServiceProxy();
    if (proxyBinder == nullptr) return;
    String16 id(cameraId);
    proxyBinder->notifyCameraState(id, newState, facing, clientName, apiLevel);
}

sp<ICameraServiceProxy> CameraService::getCameraServiceProxy() {
#ifndef __BRILLO__
    Mutex::Autolock al(sProxyMutex);
    if (sCameraServiceProxy == nullptr) {
        //这里拿到ServiceManager的代理对象，即defaultServiceManager返回的是BpServiceManager
        sp<IServiceManager> sm = defaultServiceManager();
        //这里的checkService和getService类似，查找注册服务，前者是非阻塞，后者是阻塞的
        sp<IBinder> binder = sm->checkService(String16("media.camera.proxy"));
        if (binder != nullptr) {
            //interface_cast会调用ICameraServiceProxy::asInterface(binder);
            //asInterface又是在IInterface.h中定义的一个宏
            //DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)
            sCameraServiceProxy = interface_cast<ICameraServiceProxy>(binder);
        }
    }
#endif
    return sCameraServiceProxy;
}
```

上面以`CameraService`中的`ICameraServiceProxy`为例，这里的`proxyBinder`是一个栈变量，但是后面的`proxyBinder->notifyCameraState`，就直接将其当做一个指针使用。

这其实是正确的的，并不是乱写，一切都有迹可循

1）`ICameraServiceProxy`间接继承于`RefBase`，其中`ICameraServiceProxy`继承`IInterface`，`IInterface`继承`RefBase`

```c++
//path:out/soong/.intermediates/frameworks/av/camera/libcamera_client/xxx/gen/aidl/android/hardware/
//file:ICameraServiceProxy.h
//file:BpCameraServiceProxy.h
//file:BnCameraServiceProxy.h
//其中这里是ICameraServiceProxy.h
//这里声明了一个宏，正好对应上面的ICameraServiceProxy::asInterface
//这个宏展开
/*
//这个传入的obj可知道是代理客户端的IBinder
::android::sp<ICameraServiceProxy> ICameraServiceProxy::asInterface(
        const ::android::sp<::android::IBinder>& obj)
{
    ::android::sp<ICameraServiceProxy> intr;
    if (obj != nullptr) {
        intr = static_cast<ICameraServiceProxy*>(
        	//这里代理客户端的queryLocalInterface返回结果为nullptr
            obj->queryLocalInterface(
                    ICameraServiceProxy::descriptor).get());
        if (intr == nullptr) {
            intr = new BpCameraServiceProxy(obj);
        }
    }
    return intr;
}
*/
DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE("CameraServiceProxy", "android.hardware.ICameraServiceProxy")
class ICameraServiceProxy : public ::android::IInterface
{...};
```



2）`proxyBinder->notifyCameraState`

```c++
//system/core/libutils/include/utils/StrongPointer.h
//proxyBinder->notifyCameraState转化为proxyBinder.operator->()->notifyCameraState,proxyBinder->()返回的是m_ptr，m_ptr指向ICameraServiceProxy的实例化对象，即ICameraServiceProxy的实例化对象的notifyCameraState调用
inline T*       operator-> () const    { return m_ptr;  }
```

> 关于为什么关于'->'运算符的重载会有.operator->()的解释，具体可以点击[这里](https://blog.csdn.net/zzl_python/article/details/83243353)
>
> 这里就涉及到C++标准对`->`运算符的语义解释
>
> 因为运算符的结合律不同，解引用（*）是**向右结合**的，而 成员访问运算符（`.`和`->`）是**向左结合**的。这是不同符号的结合律不同的例子，C++ 里符号相同的时候也会有用法不同导致结合律不同的例子。例如：
>
> |  运算符   |        向左结合         |   向右结合    |
> | :-------: | :---------------------: | :-----------: |
> |           | `lhs op rhs  /  lhs op` |   `op rhs`    |
> | `++` `--` |      后缀递增/递减      | 前缀递增/递减 |
> |  `+` `-`  |     二进制加法/减法     | 一元加号/减号 |
> |    `*`    |       二进制乘法        |    解引用     |
> |    `&`    |         按位与          |    取地址     |
> |   `()`    |        函数调用         |   类型转换    |
>
> 当你重载一个左结合律的操作符时（如`+ - * / ()`等），往往这个操作符是个二元操作符，对于`lhs op rhs`，就会调用`lhs.operator op(rhs)`或`operator op(lhs, rhs)`，我们只需要按照这个函数签名来重载操作符就行了。
>
> 但是，如果这个操作符是一元操作符怎么办？这就要分情况讨论了：
>
> 1）如果这个符号有一种以上的用法（比如`++`或`--`），对于`lhs op`（左结合）和`op rhs`（右结合），我们得想个办法区分不同的用法啊，所以就会出现用于占位的函数参数（`int`）：
>
> - 对于`lhs op`调用`lhs.operator op(int)`
> - 对于`op rhs`调用`rhs.operator op()`
>
> 2）如果这个符号只有一种用法（比如`->`），对于`lhs op`，那就不需要用于占位的函数参数了，可以直接写成类似右结合的函数签名，即
>
> - 对于`lhs op`调用`lhs.operator op()`
>
> 这是由 C++ 标准规定的，对于`ptr->mem`根据`ptr`类型的不同，操作符`->`的解释也不同：
>
> - 当`ptr`的类型是内置指针类型时，等价于`(*ptr).mem`
> - 当`ptr`的类型是类时，等价于`ptr.operator->()->mem`
>
> 你会发现这是一个递归的解释，对于`ptr->mem`会递归成：
>
> ```c++
> (*(ptr.operator->().operator->().….operator->())).mem
> ```
>
> 操作符`->`是一元的，而操作符`.`是二元的。操作符`->`最终是通过操作符`.`来访问成员的，而`.`这个操作符是不允许重载的，只能由编译器实现。

3）服务端实现，可以发现`sp`调用的客户端可以通过`Binder`调用到这里，即`C++`端为客户端，`Java`端为服务端

```java
//frameworks/base/services/core/java/com/android/server/camera/CameraServiceProxy.java
//ICameraServiceProxy.Stub()这个是匿名内部类的方式继承来最终实现
private final ICameraServiceProxy.Stub mCameraServiceProxy = new ICameraServiceProxy.Stub() {
    @Override
    public void pingForUserUpdate() {
        if (Binder.getCallingUid() != Process.CAMERASERVER_UID) {
            Slog.e(TAG, "Calling UID: " + Binder.getCallingUid() + " doesn't match expected " +
                   " camera service UID!");
            return;
        }
        notifySwitchWithRetries(RETRY_TIMES);
    }

    @Override
    public void notifyCameraState(String cameraId, int newCameraState, int facing,
                                  String clientName, int apiLevel) {
        if (Binder.getCallingUid() != Process.CAMERASERVER_UID) {
            Slog.e(TAG, "Calling UID: " + Binder.getCallingUid() + " doesn't match expected " +
                   " camera service UID!");
            return;
        }
        String state = cameraStateToString(newCameraState);
        String facingStr = cameraFacingToString(facing);
        if (DEBUG) Slog.v(TAG, "Camera " + cameraId + " facing " + facingStr + " state now " +
                          state + " for client " + clientName + " API Level " + apiLevel);

        updateActivityCount(cameraId, newCameraState, facing, clientName, apiLevel);
    }
};
```

> 补充说明：`C++`端代理调用和`Java`端代理调用
>
> - `Java`端代理调用
>
>   通过`AIDL`的方式，系统会生成一个`ICameraServiceProxy.java`文件，其中类图关系如下。
>
>   {{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_6.png" thumbnail="/AndroidSmartPtr/asp_6.png" title="">}}
>
>   调用方式
>
>   ```java
>   //1）调用asInterface
>   ICameraServiceProxy.Stub.asInterface(binder);
>   //2)判断是否是跨进程通信，如果不是返回null，如果是实例化一个代理类对象
>   mCameraServiceProxy = new ICameraServiceProxy.Stub.Proxy(binder);
>   //3)根据对象调用
>   mCameraServiceProxy.pingForUserUpdate();
>   //4)找到Proxy中的对应的方法，传递数据则用该方法中的transact
>   ICameraServiceProxy.Stub.Proxy.pingForUserUpdate#transact;
>   //5)找到Stub中的onTransact中的方法
>   ICameraServiceProxy.Stub.Proxy.pingForUserUpdate#onTransact;
>   //6)服务端实现ICameraServiceProxy.Stub接口，找到onTransact对应的pingForUserUpdate方法
>   new ICameraServiceProxy.Stub(){...@Override public void pingForUserUpdate() {} }
>   ```
>
>   
>
> 
>
> - `C++`端代理调用
>
>   通过`AIDL`的方式，系统会生成对应的四个文件，分别为`ICameraServiceProxy.h`,`BpCameraServiceProxy.h`,`BnCameraServiceProxy.h`,`ICameraServiceProxy.cpp`，具体的类图关系如下所示。
>
>   {{< image classes="fancybox center fig-100" src="/AndroidSmartPtr/asp_5.png" thumbnail="/AndroidSmartPtr/asp_5.png" title="">}}
>
>   调用方式
>
>   ```c++
>   //1)defaultServiceManager返回一个BpServiceManager
>   sp<IServiceManager> sm = defaultServiceManager();
>   //2)找到对应service的IBinder，这里的IBinder是代理客户端的
>   sp<IBinder> binder = sm->checkService(String16("media.camera.proxy"));
>   //3)实例化一个代理类
>   //interface_cast<ICameraServiceProxy>(binder)转化为ICameraServiceProxy::asInterface(binder)
>   //asInterface调用到BpBinder中的queryLocalInterface，返回nullptr
>   ///asInterface最后返回代理客户端的BpCameraServiceProxy实例化对象
>   sCameraServiceProxy = interface_cast<ICameraServiceProxy>(binder);
>   //4)根据对象调用
>   sCameraServiceProxy->pingForUserUpdate();
>   //5)调用到BpCameraServiceProxy.h中的函数，传递数据则用该方法中的transact
>   BpCameraServiceProxy::transact;
>   //6)BnCameraServiceProxy.h中的中的onTransact中的方法
>   BnCameraServiceProxy::onTransact;
>   //7)服务端找到onTransact中的pingForUserUpdate函数实现
>   BnCameraServiceProxy::pingForUserUpdate();
>   ```
>
> 关于`Binder`流程中，`defaultServiceManager`返回的是`BpServiceManager`，可以点击[这里](https://blog.csdn.net/yiranfeng/article/details/105210611)，本文不过多介绍。



# 源码下载

[sp_wp源码](https://github.com/YangYang48/project/tree/master/sp_wp)



