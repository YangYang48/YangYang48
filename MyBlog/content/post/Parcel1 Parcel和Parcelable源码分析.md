---
title: "Parcel1 Parcel和Parcelable源码分析"
date: 2022-02-20
thumbnailImagePosition: left
thumbnailImage: Parcel/Parcel1_thumb.jpg
coverImage: Parcel/Parcel1_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- Parcel
- Parcelable
- 2022
- February 
tags:
- java
- Android
- 源码
showSocial: false
---

Intent数据会作为Parcel被存储在Binder事务缓冲区中的对象进行传输。Parcel作为Android底层IPC通信的基础，熟悉Parcel作为了解Binder的第一步。

<!--more-->
{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_1.png" thumbnail="/Parcel/Parcel1_1.png" title="Parcel在Binder通信中的传输图">}}

(上图：Android通信过程，其中绿色为相同进程的内部通信，红色为不同进程的跨进程通信，即Binder通信)



# Parcel源码分析

如果数据传递是基本类型，直接使用`Parcel`传递。其中在App中的`Parcel`只是一个幌子，具体是通过native指针的共享内存来调用最终的实现。关于`Parcel`的使用，请移步到[手写Parcel的C++层及其使用]()。这里使用的是SDK API30，Android11源码分析。不同版本源码部分会有所区别，但原理是不变的。

首先，对App的Parcel.java接口分为三部分(基本类型、泛型、二进制的序列化)

|              基本类型              |              基本类型的数组               |
| :--------------------------------: | :---------------------------------------: |
|          `writeInt(int)`           |          `writeIntArray(int[])`           |
|        `writeFloat(float)`         |        `writeFloatArray(float[])`         |
|       `writeDouble(double)`        |       `writeDoubleArray(double[])`        |
|         `writeLong(long)`          |         `writeLongArray(long[])`          |
|       `writeString(String)`        |       `writeStringArray(String[])`        |
|         `writeChar(char)`          |         `writeCharArray(char[])`          |
|         `writeByte(byte)`          |         `writeByteArray(byte[])`          |
|      `writeBoolean(boolean)`       |      `writeBooleanArray(boolean[])`       |
|        `writeValue(Object)`        |        `writeValueArray(Object[])`        |
|                泛型                |                 泛型数组                  |
|         `writeList(List)`          |         `writeTypedList(List<T>)`         |
|          `writeMap(Map)`           | `writeArrayMap(ArrayMap<String, Object>)` |
|           二进制的序列化           |            二进制的序列化数组             |
| `writeParcelable(Parcelable, int)` |     `writeParcelableArray(T[], int)`      |
| `writeSerializable(Serializable)`  |                     /                     |



## 1.首先获取Parcel

```java
//Parcel.java#obtain
//这里面的sOwnedPool跟Message中的类似，这里判断有使用的就用使用过的，都没使用过再实例化一个新的
public static Parcel obtain() {
    final Parcel[] pool = sOwnedPool;
    synchronized (pool) {
        Parcel p;
        for (int i=0; i<POOL_SIZE; i++) {
            p = pool[i];
            if (p != null) {
                pool[i] = null;
                if (DEBUG_RECYCLE) {
                    p.mStack = new RuntimeException();
                }
                p.mReadWriteHelper = ReadWriteHelper.DEFAULT;
                return p;
            }
        }
    }
    //第一次创建会走到这里
    return new Parcel(0);
}
```

可以看到实例化之后，传入一个0的参数，这个参数实际上是C++端共享内存的**地址初始化**

```java
//Parcel.java#Parcel
private Parcel(long nativePtr) {
    if (DEBUG_RECYCLE) {
        mStack = new RuntimeException();
    }
    init(nativePtr);
}
```

Parcel继续走到`init`中

```java
//Parcel.java#init
//如果已经存在mNativePtr且不为0，那么继续使用原先的
//这里是第一次初始化，所以会调用nativeCreate，这个是走到jni层了
private long mNativePtr; // used by native code

private void init(long nativePtr) {
    if (nativePtr != 0) {
        mNativePtr = nativePtr;
        mOwnsNativeParcelObject = false;
    } else {
        mNativePtr = nativeCreate();
        mOwnsNativeParcelObject = true;
    }
}

private static native long nativeCreate();
```

jni这边会调用到android_os_Parcel.cpp，这个就是设计模式中的外观者模式，真正实现的是Binder中的Parcel.cpp。并且在jni会返回一个实例化对象的地址，正好对应传入的`mNativePtr`。

```c++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_create
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {
 {"nativeCreate",              "()J", (void*)android_os_Parcel_create},
};

static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    //这个Parcel是在Binder目录下面的Parcel.cpp
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jlong>(parcel);
}
```

下面真正的Parcel实例化的实现

特别是`mData`,`mDataSize`,`mDataCapacity`,`mDataPos`这四个值，以下简称四个值

```c++
//frameworks/native/libs/binder/Parcel.cpp#Parcel、initState
//下面不仅是一个简单的构造，还初始化一些值，后面write和read会用到
//特别是mData，mDataSize，mDataCapacity，mDataPos这四个值
Parcel::Parcel()
{
    LOG_ALLOC("Parcel %p: constructing", this);
    initState();
}

void Parcel::initState()
{
    LOG_ALLOC("Parcel %p: initState", this);
    mError = NO_ERROR;
    mData = nullptr;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    mOwner = nullptr;
    ...
}
```

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_2.png" thumbnail="/Parcel/Parcel1_2.png" title="obtain中四个值的分布图">}}



## 2.Parcel的写入

java层调用writeInt(2022)/writeString(“MyParcel”)/writeDouble(2.25)

接下来的序列化都是采用**大端**模式。

### writeInt(2022)

```java
//Parcel.java#writeInt
public final void writeInt(int val) {
    nativeWriteInt(mNativePtr, val);
}

@FastNative
private static native void nativeWriteInt(long nativePtr, int val);
```

这里直接调用到native层，可以发现这里用到了初始化的`mNativePtr`

>  使用 [`@FastNative`](https://android.googlesource.com/platform/libcore/+/master/dalvik/src/main/java/dalvik/annotation/optimization/FastNative.java) 注解可实现对 Java 原生接口 (JNI) 更快速的原生调用。`@FastNative` 注解支持非静态方法。如果某种方法将 `jobject` 作为参数或返回值进行访问，请使用此注解。`@FastNative` 可以使原生方法的性能提升高达 3 倍。



```c++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_writeInt
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {
// @FastNative
{"nativeWriteInt",            "(JI)V", (void*)android_os_Parcel_writeInt},
};

static void android_os_Parcel_writeInt(JNIEnv* env, jclass clazz, jlong nativePtr, jint val) {
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeInt32(val);
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

调用到Parcel.cpp中的`writeInt32`，最终调用到模板类

```c++
//frameworks/native/libs/binder/Parcel.cpp#writeInt32、writeAligned
status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}

//初次进入模板类的流程如下
//1.判断当前的mDataPos和传入大小是否是比已存在的容量小
//2.扩容当前的容量
//3.扩容完成之后，跳转到restart_write中写入，进入finishWrite对数据移位
template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));
	//首先判断
    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }
	//开始扩容
    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}
```

这个是用于判断扩容的函数

```c++
//frameworks/native/libs/binder/Parcel.cpp#growData
//这里的#define SIZE_MAX ((size_t)(-1)) 
//可以得知扩容方式newSize = ((mDataSize+len)*3)/2;
status_t Parcel::growData(size_t len)
{
    if (len > INT32_MAX) {
        return BAD_VALUE;
    }

    if (len > SIZE_MAX - mDataSize) return NO_MEMORY; // overflow
    if (mDataSize + len > SIZE_MAX / 3) return NO_MEMORY; // overflow
    size_t newSize = ((mDataSize+len)*3)/2;
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(newSize);
}
```

设置了扩容方式之后，开始分配内存

```c++
//frameworks/native/libs/binder/Parcel.cpp#continueWrite
//这里分配内存分两种情况
//1.如果从未扩容，直接malloc分配内存,并对四个值赋值
//2.如果本身存在数据，那么就realloc方式扩容,并对四个值赋值
status_t Parcel::continueWrite(size_t desired)
{
    ...
    else if (mData) {
        ...
        //如果本身存在数据，那么就realloc方式扩容,并对四个值赋值
        if (desired > mDataCapacity) {
            uint8_t* data = (uint8_t*)realloc(mData, desired);
            if (data) {
                pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
                gParcelGlobalAllocSize += desired;
                gParcelGlobalAllocSize -= mDataCapacity;
                pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);
                mData = data;
                mDataCapacity = desired;
            } 
        }
        ...
    } else {
        //如果从未扩容，第一次会进入这里，直接malloc分配内存,并对四个值赋值
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }
        pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocCount++;
        pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);

        mData = data;
        mDataSize = mDataPos = 0;
        mDataCapacity = desired;
    }

    return NO_ERROR;
}
```

扩容完成写入数据之后，进行数据移位

```c++
//frameworks/native/libs/binder/Parcel.cpp#finishWrite
//可以发现这里的逻辑跟bytebuffer原理类似，都是通过mData,mDataPos，mDataSize，mDataCapacity操作的
status_t Parcel::finishWrite(size_t len)
{
    if (len > INT32_MAX) {
        return BAD_VALUE;
    }

    //这里开始数据移位
    mDataPos += len;
    if (mDataPos > mDataSize) {
        mDataSize = mDataPos;
    }
    return NO_ERROR;
}
```

这里用**绿色**标识`Int`类型的数据存储。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_3.png" thumbnail="/Parcel/Parcel1_3.png" title="writeInt(2022)中四个值的分布图">}}



### writeString("MyParcel")

同理可知道`writeString`跟`writeInt`类似

```java
//Parcel.java#writeString
public final void writeString(@Nullable String val) {
    writeString16(val);
}
```

这个String的写入比较特殊，会经过`mReadWriteHelper`来转一下，最终调用`writeString16NoHelper`

```java
//Parcel.java#writeString16NoHelper
public void writeString16NoHelper(@Nullable String val) {
    nativeWriteString16(mNativePtr, val);
}

@FastNative
private static native void nativeWriteString16(long nativePtr, String val);
```

这里直接调用到native层，可以发现这里也用到了初始化的`mNativePtr`

这里有一个`GetStringCritical`，得到的是一个指向JVM内部字符串的直接指针，提高JVM返回源字符串直接指针的可能性，具体可看[这里](https://blog.csdn.net/xyang81/article/details/42066665)。

```c++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_writeString16
//这里需要传入两个值，一个为字符串的地址，一个是字符串的length长度
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {
// @FastNative
    {"nativeWriteString16",       "(JLjava/lang/String;)V", (void*)android_os_Parcel_writeString16},
}

static void android_os_Parcel_writeString16(JNIEnv* env, jclass clazz, jlong nativePtr, jstring val)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        status_t err = NO_MEMORY;
        if (val) {
            //通过val，获取到str
            const jchar* str = env->GetStringCritical(val, 0);
            if (str) {
                //传入两个值，一个值为str字符串，另一个为string内容的length
                err = parcel->writeString16(
                    reinterpret_cast<const char16_t*>(str),
                    env->GetStringLength(val));
                env->ReleaseStringCritical(val, str);
            }
        } else {
            err = parcel->writeString16(NULL, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

调用到`writeString16`

```c++
//frameworks/native/libs/binder/Parcel.cpp#writeString16
//writeString写入分成两个部分
//1.写入String类型的长度，这个跟上述writeInt(2022)完全一致
//2.写入String类型的内容（考虑到String需要最后加‘\0’，且为String16类型，单个字符占两个字节）
status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == nullptr) return writeInt32(-1);
    //1.writeInt跟上述一致
    status_t err = writeInt32(len);
    if (err == NO_ERROR) {
        //2.len的大小为原来的两倍
        len *= sizeof(char16_t);
        uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
        if (data) {
            memcpy(data, str, len);
            //将返回已分配好内存的地址返回，字符串最后设置为‘\0’
            //这个设置0的原因，writeInplace中paded分配长度可能会大于传入的len，触发大段覆盖逻辑
            *reinterpret_cast<char16_t*>(data+len) = 0;
            return NO_ERROR;
        }
        err = mError;
    }
    return err;
}
```

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_4.png" thumbnail="/Parcel/Parcel1_4.png" title="writeString(\"MyParcel\")中长度部分四个值的分布图">}}

`writeInt`这里不再阐述，直接看`writeInplace`

```c++
//frameworks/native/libs/binder/Parcel.cpp#writeInplace
//这里的len为字符串的(length+1)*2
//这里的String用到了计算所需长度特殊的函数pad_size
void* Parcel::writeInplace(size_t len)
{
    //通过paded方式计算所需空间
    const size_t padded = pad_size(len);
    if (mDataPos+padded < mDataPos) {
        return nullptr;
    }

    if ((mDataPos+padded) <= mDataCapacity) {
restart_write:
        uint8_t* const data = mData+mDataPos;
        //大段覆盖逻辑：判断paded的长度和传入的len长度，如果不相等，那么最后四个字节就按照打断方式赋值
        if (padded != len) {
            //这里定义了大端和小端，我们用的是大端
#if BYTE_ORDER == BIG_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0xffffff00, 0xffff0000, 0xff000000
            };
#endif
#if BYTE_ORDER == LITTLE_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0x00ffffff, 0x0000ffff, 0x000000ff
            };
#endif
            *reinterpret_cast<uint32_t*>(data+padded-4) &= mask[padded-len];
        }

        finishWrite(padded);
        return data;
    }
    status_t err = growData(padded);
    if (err == NO_ERROR) goto restart_write;
    return nullptr;
}
```

这里还有一个`padded`的长度，这个是`writeString`里面独有的

```c++
//frameworks/native/libs/binder/Parcel.cpp#pad_size
#define PAD_SIZE_UNSAFE(s) (((s)+3)&~3)

static size_t pad_size(size_t s) {
    if (s > (std::numeric_limits<size_t>::max() - 3)) {
        LOG_ALWAYS_FATAL("pad size too big %zu", s);
    }
    return PAD_SIZE_UNSAFE(s);
}
```

可以知道，传入的“MyParcel”，`paded`大小为20，`len`为18，不相等，触发**大段覆盖**逻辑。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_5.png" thumbnail="/Parcel/Parcel1_5.png" title="writeString(\"MyParcel\")中数据部分四个值的分布图">}}

这里用**黄红两色**标识`String`类型的数据存储。



### writeDouble(2.25)

同理可知道`writeDouble`跟`writeInt`类似

```java
//Parcel.java#writeDouble
public final void writeDouble(double val) {
    nativeWriteDouble(mNativePtr, val);
}

@FastNative
private static native void nativeWriteDouble(long nativePtr, double val);
```

这里直接调用到native层，可以发现这里也用到了初始化的`mNativePtr`

```C++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_writeDouble
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {    
// @FastNative
    {"nativeWriteDouble",         "(JD)V", (void*)android_os_Parcel_writeDouble},
}

static void android_os_Parcel_writeDouble(JNIEnv* env, jclass clazz, jlong nativePtr, jdouble val)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeDouble(val);
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

调用到`writeDouble`。显然这个和writeInt类似，都进入到模板函数中

```c++
//frameworks/native/libs/binder/Parcel.cpp#pad_size
status_t Parcel::writeDouble(double val)
{
    return writeAligned(val);
}
```

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_6.png" thumbnail="/Parcel/Parcel1_6.png" title="writeDouble(2.25)中四个值的分布图">}}

这里用**蓝色**标识`Double`类型的数据存储。

可以发现三次`write`操作，实际上存储是一段连续的空间，并且对空间的数进行划分存储。



## 3.Parcel的读出

java层调用readInt()/readString()/readDouble

### readInt()

```java
//Parcel.java#readInt
public final int readInt() {
    return nativeReadInt(mNativePtr);
}

@CriticalNative
private static native int nativeReadInt(long nativePtr);
```

用初始化的`mNativePtr`调用`nativeReadInt`

```c++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_readInt
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {
// @CriticalNative
    {"nativeReadInt",             "(J)I", (void*)android_os_Parcel_readInt},
};

static jint android_os_Parcel_readInt(jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        return parcel->readInt32();
    }
    return 0;
}
```

跟`writeInt32`同理，最终`readInt32`调用到模板函数`readAligned`中

```c++
//frameworks/native/libs/binder/Parcel.cpp#readInt32、readAligned
int32_t Parcel::readInt32() const
{
    return readAligned<int32_t>();
}

template<class T>
T Parcel::readAligned() const {
    T result;
    if (readAligned(&result) != NO_ERROR) {
        result = 0;
    }

    return result;
}

template<class T>
status_t Parcel::readAligned(T *pArg) const {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(T)) <= mDataSize) {
        if (mObjectsSize > 0) {
            //这个是在内存中的数据有效性校验，如果传入的值跟内存的所需长度不一致就会异常
            status_t err = validateReadData(mDataPos + sizeof(T));
            if(err != NO_ERROR) {
                // Still increment the data position by the expected length
                mDataPos += sizeof(T);
                return err;
            }
        }
        //这里开始读取数据
        const void* data = mData+mDataPos;
        mDataPos += sizeof(T);
        *pArg =  *reinterpret_cast<const T*>(data);
        return NO_ERROR;
    } else {
        return NOT_ENOUGH_DATA;
    }
}
```

但是发现如果在write之后直接read，pos和data所指的位置在最后，是不正确的。

所需在write和read之间必须调用`setDataPosition(0);`，让其重置到数据的第一位。

```java
//Parcel.java#setDataPosition
public final void setDataPosition(int pos) {
    nativeSetDataPosition(mNativePtr, pos);
}

@CriticalNative
private static native void nativeSetDataPosition(long nativePtr, int pos);
```

用初始化的`mNativePtr`调用`nativeReadInt`

```c++
//frameworks/base/core/jni/android_os_Parcel.cpp#android_os_Parcel_setDataPosition
//动态注册省略
static const JNINativeMethod gParcelMethods[] = {
// @CriticalNative
    {"nativeSetDataPosition",     "(JI)V", (void*)android_os_Parcel_setDataPosition},
}

static void android_os_Parcel_setDataPosition(jlong nativePtr, jint pos)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        parcel->setDataPosition(pos);
    }
}
```

可以看到最终使得pos重置为0

```c++
//frameworks/native/libs/binder/Parcel.cpp#setDataPosition
void Parcel::setDataPosition(size_t pos) const
{
    if (pos > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        LOG_ALWAYS_FATAL("pos too big: %zu", pos);
    }

    mDataPos = pos;
    mNextObjectHint = 0;
    mObjectsSorted = false;
}
```



重置完成之后，开始读取Int值就正常了。数值为2022。

`readString`和`readDouble`也是如此，这里不在展开详细解释了。

可以发现读取也是一段连续的空间，并且对存储的空间划分区域取出。这就是为什么写入之后，必须也按**顺序读取**的原因。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_12.png" thumbnail="/Parcel/Parcel1_12.png" title="Parcel源码分析原理图">}}



# Parcelable.java源码分析

如果传递的不是基本类型是对象，那么需要用到的是`Parcelable`的实现类。

然而Parcelable.java最终也是通过Parcel.java来实现的。

下面通过简单的两个`Activity`页面的跳转`Intent`传递的`Bundle`值来说明`Parcalable`原理，具体请移步到[手写Parcel的C++层及其使用]()。



## 数据写入

### 1.putExtra

这里直接从`Activity`中调用`putExtra`

```java
//MainActivity.java#onCreate
//我们从这里开始调用
intent.putExtra("P1", bean1);//这里的bean1的javaBean为(2022,"MyParcel",2.25)
intent.putExtra("P2", bean2);//这里的bean2的javaBean为(2022,"AndroidSourceCode",2.25)
```

调用到`Intent`中的`putExtra`

```java
//Intent.java#putExtra
//可以看到最终调用Bundle里面的putParcelable方法
public @NonNull Intent putExtra(String name, @Nullable Parcelable value) {
    if (mExtras == null) {
        mExtras = new Bundle();
    }
    mExtras.putParcelable(name, value);
    return this;
}
```

很显然，传递进去的值都是通过`Bundle`来传递的

```java
//Bundle.java#putParcelable
//这里面有两个比较重要的一个是unparcel
//这个unparcel先按下不表，作用是mParcelledData这个Parcel中解包,把数据取出存入mMap中
//另一个是mMap,这个用于将刚刚放入Bundle中的数据，通过键值对存到mMap中
public void putParcelable(@Nullable String key, @Nullable Parcelable value) {
    unparcel();
    mMap.put(key, value);
    mFlags &= ~FLAG_HAS_FDS_KNOWN;
}
```

上面的流程是将数据存入到`Bundle`中的`mMap`中。



### 2.AMS的回调

在Activity启动过程中，由AMS完成进程通信，期间，将调用**Intent.writeToParcel()**将所有必要数据进行序列化，并完成传输。(具体AMS如何调到Intent的writeToParcel，接下来的AMS篇章会解释)

```java
//Intent.java#writeToParcel
public void writeToParcel(Parcel out, int flags) {
    out.writeString8(mAction);
    Uri.writeToParcel(out, mData);
    out.writeString8(mType);
    out.writeString8(mIdentifier);
    out.writeInt(mFlags);
    out.writeString8(mPackage);
    ComponentName.writeToParcel(mComponent, out);

    if (mSourceBounds != null) {
        out.writeInt(1);
        mSourceBounds.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }

    if (mCategories != null) {
        final int N = mCategories.size();
        out.writeInt(N);
        for (int i=0; i<N; i++) {
            out.writeString8(mCategories.valueAt(i));
        }
    } else {
        out.writeInt(0);
    }

    if (mSelector != null) {
        out.writeInt(1);
        mSelector.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }

    if (mClipData != null) {
        out.writeInt(1);
        mClipData.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }
    out.writeInt(mContentUserHint);
    out.writeBundle(mExtras);
}
```

可以看到实际上通过Parcel写入很多信息，包括动作，数据，类型，认证等

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_9.png" thumbnail="/Parcel/Parcel1_9.png" title="AMS中写入的数据结构图">}}



这里只对Bundle进行展开分析，其他的感兴趣的宝宝可以自助分析

```java
//Bundle.java#writeBundle、writeToParcel
public final void writeBundle(@Nullable Bundle val) {
    if (val == null) {
        writeInt(-1);
        return;
    }
    val.writeToParcel(this, 0);
}

@Override
public void writeToParcel(Parcel parcel, int flags) {
    final boolean oldAllowFds = parcel.pushAllowFds((mFlags & FLAG_ALLOW_FDS) != 0);
    try {
        super.writeToParcelInner(parcel, flags);
    } finally {
        parcel.restoreAllowFds(oldAllowFds);
    }
}
```

最终调用到了`Bundle`基类中的`writeToParcelInner`

```java
//BaseBundle.java#writeToParcelInner
//这个方法内容比较多，拆分成几部分
//1.unparcel，先按下不表，数据压入到mMap
//2.第一次进入mParcelledData没有数据，直接往下走不退出，且将unparcel中的mMap数据给到map
//3.不是第一次进入，mParcelledData有数据，Parcel数据拼接给mParcelledData
void writeToParcelInner(Parcel parcel, int flags) {
    if (parcel.hasReadWriteHelper()) {
        unparcel();
    }
    final ArrayMap<String, Object> map;
    synchronized (this) {
        if (mParcelledData != null) {
            if (mParcelledData == NoImagePreloadHolder.EMPTY_PARCEL) {
                parcel.writeInt(0);
            } else {
                //mParcelledData有数据，Parcel数据拼接给mParcelledData
                int length = mParcelledData.dataSize();
                parcel.writeInt(length);
                parcel.writeInt(mParcelledByNative ? BUNDLE_MAGIC_NATIVE : BUNDLE_MAGIC);
                parcel.appendFrom(mParcelledData, 0, length);
            }
            return;
        }
        //第一次进入mParcelledData没有数据，直接往下走不退出
        map = mMap;
    }

    if (map == null || map.size() <= 0) {
        parcel.writeInt(0);
        return;
    }
    //开始写入一些信息，-1（map的长度），魔数和map数据
    int lengthPos = parcel.dataPosition();
    parcel.writeInt(-1); // dummy, will hold length
    parcel.writeInt(BUNDLE_MAGIC);

    int startPos = parcel.dataPosition();
    parcel.writeArrayMapInternal(map);
    int endPos = parcel.dataPosition();
	//写完map数据后，再将原来是-1的值改成map的长度
    parcel.setDataPosition(lengthPos);
    int length = endPos - startPos;
    parcel.writeInt(length);
    parcel.setDataPosition(endPos);
}
```

然后对map具体进一步写入分析，把写入的每一步细化出来

```java
//Parcel.java#writeArrayMapInternal、writeValue
void writeArrayMapInternal(@Nullable ArrayMap<String, Object> val) {
    if (val == null) {
        writeInt(-1);
        return;
    }
    //统计val的map中有多少个元素，然后循环遍历key和value
    final int N = val.size();
    writeInt(N);
    int startPos;
    for (int i=0; i<N; i++) {
        if (DEBUG_ARRAY_MAP) startPos = dataPosition();
        writeString(val.keyAt(i));
        writeValue(val.valueAt(i));
    }
}

public final void writeValue(@Nullable Object v) {
    if (v == null) {
        writeInt(VAL_NULL);
    }
    ...
    else if (v instanceof Parcelable) {
        //写入value的类型
        writeInt(VAL_PARCELABLE);
        writeParcelable((Parcelable) v, 0);
    }
    ...
}
```

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_7.png" thumbnail="/Parcel/Parcel1_7.png" title="Bundle的数据结构图">}}

传入到`v`，实际上就是实现`Parcelable`的Bean类的对象

```java
//Parcel.java#writeParcelable、writeParcelableCreator
public final void writeParcelable(@Nullable Parcelable p, int parcelableFlags) {
    if (p == null) {
        writeString(null);
        return;
    }
    writeParcelableCreator(p);
    p.writeToParcel(this, parcelableFlags);
}

public final void writeParcelableCreator(@NonNull Parcelable p) {
    //写入全类名
    String name = p.getClass().getName();
    writeString(name);
}
```

最终调用到了Bean类中的`writeToParcel`

```java
//Bean#writeToParcel
@Override
public void writeToParcel(Parcel dest, int flags) {
    Log.d(TAG, "->>>ParcelTest|writeToParcel|dest = " + dest + " flags = " + flags);
    dest.writeInt(age);
    dest.writeString(name);
    dest.writeInt(count);
}
```

这里的bean1的javaBean为`(2022,"MyParcel",2.25)`和bean2的javaBean为`(2022,"AndroidSourceCode",2.25)`

那么实际的数为下图所示。绿色代表`Int`类型，浅绿色代表`Int`的魔数数据，黄红双色为`String`类，蓝色代表`Double`类型，具体可根据Parcel源码解析结合Parcal写入分析。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_8.png" thumbnail="/Parcel/Parcel1_8.png" title="实例的Bundle的数据结构图">}}

`Parcelable`写入的流程如下图所示。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_10.png" thumbnail="/Parcel/Parcel1_10.png" title="Parcelable写入的流程图">}}



## 数据读出

从新的`Activity`中读取

`getIntent().getParcelableExtra("P1");`和`getIntent().getParcelableExtra("P2");`

```java
//Intent.java#getParcelableExtra
public @Nullable <T extends Parcelable> T getParcelableExtra(String name) {
    return mExtras == null ? null : mExtras.<T>getParcelable(name);
}
```

很显然，先前传递进去的值都是通过`Bundle`来获取的

```java
//Bundle.java
//这里分成两部分
//1.unparcel，这个是读取过程中最重要的，将数据放入到mMap中
//2.通过mMap中的key获取对应value的Parcelable数据
@Nullable
public <T extends Parcelable> T getParcelable(@Nullable String key) {
    unparcel();
    Object o = mMap.get(key);
    if (o == null) {
        return null;
    }
    try {
        return (T) o;
    } catch (ClassCastException e) {
        typeWarning(key, o, "Parcelable", e);
        return null;
    }
}
```

主要分析这个unparcel

```java
//BaseBundle.java#unparcel
//显然是通过mParcelledData数据来处理这个过程，这个数据在Bundle基类中的writeToParcelInner也存在
@UnsupportedAppUsage
/* package */ void unparcel() {
    synchronized (this) {
        final Parcel source = mParcelledData;
        if (source != null) {
            initializeFromParcelLocked(source, /*recycleParcel=*/ true, mParcelledByNative);
        } else {
            if (DEBUG) {
                Log.d(TAG, "unparcel "
                      + Integer.toHexString(System.identityHashCode(this))
                      + ": no parcelled data");
            }
        }
    }
}
```

通过传入的`mParcelledData`来分析这个读取过程

```java
//BaseBundle.java
//这个内容比较多，分成几个部分
//1.parcelledData的有效性判断
//2.清空mMap数据，并开始重新写入
//3.ParcelData的读取Map内容过程，将数据读取到map中，赋值给mMap
private void initializeFromParcelLocked(@NonNull Parcel parcelledData, boolean recycleParcel,
                                        boolean parcelledByNative) {
    ...
    if (isEmptyParcel(parcelledData)) {
        ...
            if (mMap == null) {
                mMap = new ArrayMap<>(1);
            } else {
                mMap.erase();
            }
        mParcelledData = null;
        mParcelledByNative = false;
        return;
    }
	//先读取二进制中存入的map的size
    final int count = parcelledData.readInt();
    ...
    if (count < 0) {
        return;
    }
    //这里操作为清空mMap数据，并开始重新写入
    ArrayMap<String, Object> map = mMap;
    if (map == null) {
        map = new ArrayMap<>(count);
    } else {
        map.erase();
        map.ensureCapacity(count);
    }
    
    try {
        if (parcelledByNative) {
            parcelledData.readArrayMapSafelyInternal(map, count, mClassLoader);
        } else {
            //这个是ParcelData的读取Map内容过程
            parcelledData.readArrayMapInternal(map, count, mClassLoader);
        }
    } catch (BadParcelableException e) {
        if (sShouldDefuse) {
            Log.w(TAG, "Failed to parse Bundle, but defusing quietly", e);
            map.erase();
        } else {
            throw e;
        }
    } finally {
        //读取中的数据，重新写入到空的mMap中
        mMap = map;
        //这个是用来将上述的parcelledData保存到Parcel的sOwnedPool中
        if (recycleParcel) {
            recycleParcel(parcelledData);
        }
        mParcelledData = null;
        mParcelledByNative = false;
    }
    ...
}
```

读取`parcelledData`中的数据，这里传入的`map`是空的

```java
//Parcel.java#readArrayMapInternal
//这里是真正意义上读取二进制的map数据中的key和value
void readArrayMapInternal(@NonNull ArrayMap outVal, int N,
                          @Nullable ClassLoader loader) {
    int startPos;
    while (N > 0) {
        if (DEBUG_ARRAY_MAP) startPos = dataPosition();
        String key = readString();
        Object value = readValue(loader);
        //开始通过读取key，value保存到outVal中
        outVal.append(key, value);
        N--;
    }
    outVal.validate();
}
```

对`readValue`进一步分析，这个`value`实际就是`Parcelable`

```java
//Parcel.java#readValue
@Nullable
public final Object readValue(@Nullable ClassLoader loader) {
    int type = readInt();

    switch (type) {
        case VAL_NULL:
            return null;

        case VAL_PARCELABLE:
            return readParcelable(loader);
        ...
}
```

这里对`Parcelable`读取操作

```java
//Parcel.java#readParcelable
public final <T extends Parcelable> T readParcelable(@Nullable ClassLoader loader) {
    //通过类加载器获取到Parcelable实现的Bean类中的creator
    Parcelable.Creator<?> creator = readParcelableCreator(loader);
    if (creator == null) {
        return null;
    }
    if (creator instanceof Parcelable.ClassLoaderCreator<?>) {
        Parcelable.ClassLoaderCreator<?> classLoaderCreator =
            (Parcelable.ClassLoaderCreator<?>) creator;
        return (T) classLoaderCreator.createFromParcel(this, loader);
    }
    //调用到Bean内部类的createFromParcel
    return (T) creator.createFromParcel(this);
}
```

这边会使用到一次java反射，并对传入的类加载器进入处理

```java
//Parcel.java#readParcelableCreator
//第一次进入做了一次反射操作，获取creator
//1.第一次进去，map中通过全类名获取不到对应的Bean的creator，反射获取，存入到map中
//2.非第一次进入，通过全类名可以获取到对应的Bean的creator
@Nullable
public final Parcelable.Creator<?> readParcelableCreator(@Nullable ClassLoader loader) {
    //获取全类名
    String name = readString();
    if (name == null) {
        return null;
    }
    Parcelable.Creator<?> creator;
    HashMap<String, Parcelable.Creator<?>> map;
    synchronized (mCreators) {
        map = mCreators.get(loader);
        if (map == null) {
            map = new HashMap<>();
            mCreators.put(loader, map);
        }
        creator = map.get(name);
    }
    //非第一次进入,通过在map中全类名获取到具体的creator
    if (creator != null) {
        return creator;
    }
    //第一次进入获取不到具体的creator
    try {
        ClassLoader parcelableClassLoader =
            (loader == null ? getClass().getClassLoader() : loader);
        //通过全类名，类加载器，获取对应的Class对象，即Bean对象
        Class<?> parcelableClass = Class.forName(name, false /* initialize */,
                                                 parcelableClassLoader);
        if (!Parcelable.class.isAssignableFrom(parcelableClass)) {
            throw new BadParcelableException("Parcelable protocol requires subclassing "
                                             + "from Parcelable on class " + name);
        }
        //获取Bean对象中的静态内部类CREATOR字段信息
        Field f = parcelableClass.getField("CREATOR");
        //getModifiers的解释详见如下
        if ((f.getModifiers() & Modifier.STATIC) == 0) {
            throw new BadParcelableException("Parcelable protocol requires "
                                             + "the CREATOR object to be static on class " + name);
        }
        //获取静态内部类的类型，并判断是否和Bean中的类型一致
        Class<?> creatorType = f.getType();
        if (!Parcelable.Creator.class.isAssignableFrom(creatorType)) {
            throw new BadParcelableException("Parcelable protocol requires a "
                                             + "Parcelable.Creator object called "
                                             + "CREATOR on class " + name);
        }
        //获取静态对象的Field属性值，静态方法可以传递null
        creator = (Parcelable.Creator<?>) f.get(null);
    } catch (IllegalAccessException e) {
        Log.e(TAG, "Illegal access when unmarshalling: " + name, e);
        throw new BadParcelableException(
            "IllegalAccessException when unmarshalling: " + name);
    } catch (ClassNotFoundException e) {
        Log.e(TAG, "Class not found when unmarshalling: " + name, e);
        throw new BadParcelableException(
            "ClassNotFoundException when unmarshalling: " + name);
    } catch (NoSuchFieldException e) {
        throw new BadParcelableException("Parcelable protocol requires a "
                                         + "Parcelable.Creator object called "
                                         + "CREATOR on class " + name);
    }
    if (creator == null) {
        throw new BadParcelableException("Parcelable protocol requires a "
                                         + "non-null Parcelable.Creator object called "
                                         + "CREATOR on class " + name);
    }
    //得到所想要的creator之后，将其存入值map中，下一次可以通过全类名去获取到
    synchronized (mCreators) {
        map.put(name, creator);
    }

    return creator;
}
```



> Java反射机制的常用方法
>
> **1.getModifiers**
>
> Field的getModifiers()方法返回int类型值表示该字段的修饰符
>
> ```
> PUBLIC: 1
> PRIVATE: 2
> PROTECTED: 4
> STATIC: 8
> FINAL: 16
> SYNCHRONIZED: 32
> VOLATILE: 64
> TRANSIENT: 128
> NATIVE: 256
> INTERFACE: 512
> ABSTRACT: 1024
> STRICT: 2048
> ```
>
> 
>
> **2.isAssignableFrom**
>
> - isAssignableFrom()方法是从类继承的角度去判断，instanceof关键字是从实例继承的角度去判断。
> - isAssignableFrom()方法是判断是否为某个类的父类，instanceof关键字是判断是否某个类的子类。
>
> ```java
> 父类.class.isAssignableFrom(子类.class)
> 
> 子类实例 instanceof 父类类型
> ```
>
> 
>
> **3.Field.get(null)**
>
> 关于Field.get(null)，详见[这里](https://blog.csdn.net/moakun/article/details/80577194)
>
> 定义了嵌套接口，那么接口默认修饰为public static



最终调用到了`creator.createFromParcel`，这个就是我们实现的Bean

```java
//Bean.java
public static final Creator<ParcelTest> CREATOR = new Creator<ParcelTest>() {
    @Override
    public ParcelTest createFromParcel(Parcel in) {
        //最终会调用到这里，这里会调用Parcel的有参构造，完成真正意义的Bean内容读取
        Log.d(TAG, "->>>ParcelTest|createFromParcel|in = " + in);
        return new ParcelTest(in);
    }

    @Override
    public ParcelTest[] newArray(int size) {
        Log.d(TAG, "->>>ParcelTest|newArray|size = " + size);
        return new ParcelTest[size];
    }
};

protected ParcelTest(Parcel in) {
    Log.d(TAG, "->>>ParcelTest|in = " + in);
    age = in.readInt();
    name = in.readString();
    count = in.readInt();
}
```

读取流程如下图所示。

{{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_11.png" thumbnail="/Parcel/Parcel1_11.png" title="Parcelable读取流程图">}}



> 补充：
>
> Q:关于`Bundle`中的`mMap`是哪里来的?
>
> A:来自`readArrayMapInternal`中的`outVal`。
>
> Bundle中的mMap是继承自BaseBundle，BaseBundle中的mMap通过readArrayMapInternal传入空的outval。
>
> Parcel通过mParcelableData的数据，读取key和value拼接放置到outVal中。其中value是通过Parcel读取readParcel，通过反射真正读取到Bean的三种类型数据。如下图所示。
>
> {{< image classes="fancybox center fig-100" src="/Parcel/Parcel1_13.png" thumbnail="/Parcel/Parcel1_13.png" title="mMap数据跟踪图">}}



# 总结：

1.Parcel

实际上我们调用的java层的Parcel不是真正的实现，通过jni获取到的`mNativePtr`共享内存最终调用到`Binder`下面的Parcel.cpp。

这个cpp中真正意义上的实现并且可以写入读取一段连续的内存，因此我们需要在读取按照写入的顺序依次读取。



2.Parcelable

- `unparcel`反序列化过程，将二进制的连续内存转化存入为`mMap`中（`mParcelableData`--->`mMap`）
- `parcel`是二进制序列化过程，将`mMap`中的数据转化为二进制的连续内存（`mMap`--->`mParcelableData`）

`Parcel`不仅仅是序列化，确切是二进制的序列化。因为序列化的种类很多，`Parcel`和`Serializable`是二进制序列化。反序列化过程，java会出现一次反射，序列化过程中不需要反射。



3.嵌套的Bean对象

如果javaBean中对象嵌套了另一个javaBean，如果BeanInnerBean类里面有其他对象（比如实体类DataBean）的话，那么DataBean也需要实现Parcelable接口，用法与上面的Bean类一样。

writeToParcel里面需要写上：

```java
@Override
public void writeToParcel(Parcel dest, int flags) {
    ...
    dest.writeParcelable(dataBean, 0);
}
```

BeanInnerBean构造函数中

```java
protected BeanInnerBean(Parcel in) {
    ...
    dataBean = in.readParcelable(DataBean.class.getClassLoader());
}
```



# 猜你想看

[bytebuffer使用](https://yangyang48.github.io/2022/02/bytebuffer%E4%BD%BF%E7%94%A8/)

[java序列化]()

[jni6 手写Parcel的C++层及其使用]()



# 特别感谢

微博用户[S小吠](https://weibo.com/u/1864580434)提供的冰墩墩的壁纸



# 参考

[[1] 韦东锏, Parcel的源码上手, 2021.](https://zhuanlan.zhihu.com/p/402790867)

[[2] MxsQ, Parcelable 是如何实现的？, 2020.](https://juejin.im/user/3491704658736942)

[[3] Boyikia, 理解 Parcel 和 Parcelable, 2019.](https://blog.csdn.net/zaojian0848/article/details/102503836)

[[4] 铁憨憨的学习记录, Java反射机制getModifiers()方法的作用, 2018.](https://blog.csdn.net/qq_40434646/article/details/82351488)

[[5] zero9988, 简要分析Android中的Intent,Bundle,Parcel中的数据传递, 2017.](https://blog.csdn.net/zero9988/article/details/73459022)

[[6] freshui, Parcel数据传输过程，简要分析Binder流程, 2017.](https://blog.csdn.net/freshui/article/details/55051268)

[[7] 水蓝城城主, java.nio.ByteBuffer用法小结, 2019.](https://blog.csdn.net/mrliuzhao/article/details/89453082)

[[8] zhangbijun1230, Android 8.0学习（31）---Android 8.0 中的 ART 功能改进, 2018.](https://blog.csdn.net/zhangbijun1230/article/details/80562747)

[[9] xyang0917, JNI/NDK开发指南（四）——字符串处理, 2014.](https://blog.csdn.net/xyang81/article/details/42066665)

[[10] 茅坤宝骏氹, java反射的field.get(null), 2018.](https://blog.csdn.net/moakun/article/details/80577194)

[[11] wkCaeser_, java中isAssignableFrom()方法与instanceof关键字用法及通过反射配合注解为字段设置默认值, 2018.](https://blog.csdn.net/qq_36666651/article/details/81215221)