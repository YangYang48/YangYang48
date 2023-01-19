---
title: "HIDL介绍和使用"
date: 2022-03-27
thumbnailImagePosition: left
thumbnailImage: hidl/hidl_thumb.jpg
coverImage: hidl/hidl_cover.jpeg
metaAlignment: center
coverMeta: out
draft: true
categories:
- HIDL
- 2022
- March 
tags:
- Binder
- Android
- IPC
- 源码
showSocial: false
---


AIDL并不是唯一的Binder的应用，除了AIDL还有一个经常在HAL层代码中出现的HIDL。本文就是通过HIDL展开描述，主要对于AIDL的介绍和使用。

<!--more-->
一个同样基于Binder通信的一个架构

{{< image classes="fancybox center fig-100" src="/hidl/hidl_1.png" thumbnail="/hidl/hidl_1.png" title="">}}

# 1HIDL

## 1.1HIDL介绍

HIDL 是用于指定 HAL 与其用户之间接口的一个接口描述语言（Interface Description Language），它允许将指定的类型与函数调用收集到接口（Interface）和包（Package）中。更广泛地说，HIDL 是一个可以让那些独立编译的代码库（Libraries）之间进行通信的系统。 HIDL 实际上是用于进行进程间通信（Inter-process Communication，IPC）的。进程间的通信可以称为 Binder 化（Binderized）。对于必须连接到进程的库，也可以使用 passthough 模式（但在Java中不支持）。 HIDL 将指定的数据结构与方法签名组织到接口中，这些接口又会被收集到包中以供使用。它的语法与 C++、JAVA 是类似的，不过关键字集合不尽相同。其注释风格与 JAVA 是一致的。




## 1.2AIDL和HIDL区别

三种Binder介绍以及之间的联系

在Android 8.0开始，Android引入了Treble的机制，为了方便Android系统的快速移植、升级，提升系统稳定性，Binder机制被拓展成了"/dev/binder", "/dev/hwbinder","/dev/vndbinder"。最明显的就是binder库发生变化，Binder和VndBinder共同使用libbinder， HwBinder使用libhwbinder。

|             libbinder              |         libhwbinder         |
| :--------------------------------: | :-------------------------: |
| 前缀/frameworks/native/libs/binder |   前缀/system/libhwbinder   |
|             Binder.cpp             |         Binder.cpp          |
|            BpBinder.cpp            |       BpHwBinder.cpp        |
|           IIterface.cpp            |        IIterface.cpp        |
|         IPCThreadState.cpp         |     IPCThreadState.cpp      |
|          ProcessState.cpp          |      ProcessState.cpp       |
|             Parcel.cpp             |         Parcel.cpp          |
|     **前缀+=/include/binder**      | **前缀+=/include/hwbinder** |
|              Binder.h              |          Binder.h           |
|             BpBinder.h             |        BpHwBinder.h         |
|             IBinder.h              |          IBinder.h          |
|            IIterface.h             |         IIterface.h         |
|          IPCThreadState.h          |      IPCThreadState.h       |
|           ProcessState.h           |       ProcessState.h        |
|              Parcel.h              |          Parcel.h           |

另外servicemanager的差异如下

|               servicemanager               |  vndservicemanager   |       hwservicemanager       |
| :----------------------------------------: | :------------------: | :--------------------------: |
| 前缀/frameworks/native/cmds/servicemanager | 见servicemanager前缀 | 前缀/system/hwservicemanager |
|             servicemanager.rc              | vndservicemanager.rc |     hwservicemanager.rc      |
|             service_manager.c              |  service_manager.c   |      ServiceManager.cpp      |
|                  binder.c                  |       binder.c       |       HidlService.cpp        |
|                  binder.h                  |       binder.h       |        HidlService.h         |



### 1.2.1 dev/binder

这个是我们最熟悉的Binder，App开发中，ActivityManagerService用的都是这个，Java继承BpBinder，C++中继承BBinder，然后通过servicemanager进程注册实名Binder，然后通过已经创建好的Binder接口传递匿名Binder对象，拿到BinderProxy或者BpBinder以后，就可以Binder通信了。

### 1.2.2 dev/vndbinder

其实dev/vndbinde和dev/binder使用方式基本一样而且是共用一套Binder SDK，也是Java继承Bpinder，C++中继承BBinder，但是通过vndservicemanager进程注册实名Binder，然后通过已经创建好的Binder接口传递匿名Binder对象，拿到BinderProxy或者BpBinder以后，就可以Binder通信了。如何在使用同一套Binder SDK的代码，最后访问的设备节点变成dev/vndbinder，servicemanager变成vndservicemanager。

其实和简单，知识进程初始化的时候执行下面这个代码

```c++
ProcessState::initWithDriver("/dev/vndbinder");
```

注：**dev/binder和dev/vndbinder无法在一个进程中同时使用**



### 1.2.3 dev/hwbinder

那么dev/hwbinder是如何解决与dev/binder或dev/vndbinder之间的共存问题？有人肯定想到了，很简单，我们把所有Binder SDK复制一套新的Hw Binder SDK，改名成dev/hwbinder，HwBinder，HwBbinder，hwservicemanager，HwProcessState，这样子不就可以和dev/binder或dev/vndbinder共存了嘛？其实android团队就会类似这样子干的。

他们的目标有两个：

1.不能那么自由，强制所有供应商按照android官方定义的hal接口来实现
2.不能增加供应商开发人员的学习成本，学习一套复杂的Hw Binder SDK

为了达成上述的两个目标，android团队想出了HIDL这个方案。

### 1.2.4 总结

Binder和VndBinder 共用一套libbinder、servicemanager的代码，使用AIDL接口，两者不能共存。

HwBinder采用了单独的libhwbinder、hwservicemanager的代码，单独管理，使用HIDL接口。

三者之间互不干扰，在方便系统升级的同时，又提升了系统的安全和稳定性。


# 2HIDL实战

## 2.1定义接口文件

进入`hardware/interfaces/`目录下建立新的接口文件

首先建立对应的文件夹

```shell
mkdir -p hardware/interfaces/yangyang/1.0/defaul
```

接着创建接口描述文件IYangyang.hal：

```shell
vi hardware/interfaces/yangyang/1.0/IYangyang.hal
```

IYangyang.hal内容：

```go
//hardware/interfaces/yangyang/1.0/IYangyang.hal
package android.hardware.yangyang@1.0;

interface IYangyang{
    helloWorld(string name) generates (string result);
};
```


 
## 2.2使用工具，根据接口文件生成代码

系统定义的所有的.hal接口，都是通过hidl-gen工具转换成对应的代码。hidl-gen源码路径：system/tools/hidl，是在ubuntu上可执行的二进制文件。

{{< image classes="fancybox center fig-100" src="/hidl/hidl_2.png" thumbnail="/hidl/hidl_2.png" title="">}}

> aosp的部分翻译
>
> hidl-gen
> 完整的文档可以在这里找到：https://source.android.com/devices/architecture/hidl/
>
> hidl-gen 是 HIDL（HAL 接口设计语言）的编译器，它为 RPC 机制生成 C++ 和 Java 端点。该编译器使用的主要用户空间库可以在 system/libhidl 中找到。
>
> 1. 构建
> m hidl-gen
>
> 2. 运行
>     请注意，预期将由构建系统调用的 hidl-gen 选项在帮助菜单中标记为“内部”。
>
>   hidl-gen -h
>
>   hidl-gen -o output -L c++-impl -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport android.hardware.nfc@1.0
>
>   hidl-gen -o output -L c++-impl android.hardware.nfc@1.0
>   hidl-gen -o output -L vts android.hardware.nfc@1.0
>   hidl-gen -L hash android.hardware.nfc@1.0
>
>   供应商项目的示例命令
>
>   hidl-gen -L c++-impl -r vendor.foo:vendor/foo/interfaces vendor.foo.nfc@1.0
>   有关如何生成 HIDL makefile（使用 -Landroidbp 选项）的示例，请参阅 update-makefiles-helper.sh 和 update-all-google-makefiles.sh。
>
>   
>
> 使用方法：hidl-gen -o output-path -L language (-r interface-root) fqname
>
> 例子：
>
> ```groovy
> hidl-gen -o hardware/interfaces/yangyang/1.0/default/ -Lc++-impl -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport android.hardware.yangyang@1.0
> ```
>
> 参数说明：
>
> -L： 语言类型，包括c++, c++-headers, c++-sources, export-header, c++-impl, java, java-constants, vts, makefile, androidbp, androidbp-impl, hash等。hidl-gen可根据传入的语言类型产生不同的文件。
> fqname： 完全限定名称的输入文件。比如本例中android.hardware.yangyang@1.0，要求在源码目录下必须有hardware/interfaces/ yangyang/1.0/目录。
>
> 对于单个文件来说，格式如下：package@version::fileName，比如android.hardware. yangyang@1.0::types.Feature。对于目录来说。格式如下package@version，比如android.hardware. yangyang@1.0。
> -r： 格式package:path，可选，对fqname对应的文件来说，用来指定包名和文件所在的目录到Android系统源码根目录的路径。如果没有制定，前缀默认是：android.hardware，目录是Android源码的根目录。
> -o：存放hidl-gen产生的中间文件的路径。
> {{< image classes="fancybox center fig-100" src="/hidl/hidl_n.png" thumbnail="/hidl/hidl_n.png" title="">}}



Google提供了一些工具来帮助制作HIDL的框架：

```shell
make hidl-gen 
```

源码中编译生成hidl-gen。

注意：编译前需要执行全编译的环境变量加载（source build/envsetup.sh && lunch ,具体可以点击[这里](https://blog.csdn.net/GetNextWindow/article/details/48160569?spm=1001.2101.3001.6650.12&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-12.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-12.pc_relevant_default&utm_relevant_index=18)）

使用hidl-gen工具生成代码

```shell
## 设置两个临时变量PACKAGE和LOC
$ PACKAGE=android.hardware.yangyang@1.0
$ LOC=hardware/interfaces/yangyang/1.0/default/
# 生成两个文件，一个c++实现文件和一个头文件
$ hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport $PACKAGE
# 生成一个bp文件
$ hidl-gen -o $LOC -Landroidbp-impl -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport $PACKAGE
```

执行完后，会生成三个文件：

{{< image classes="fancybox center fig-100" src="/hidl/hidl_3.png" thumbnail="/hidl/hidl_3.png" title="">}}

## 2.3完善接口函数

Yangyang.h

```cpp
//hardware/interfaces/yangyang/1.0/default/Yangyang.h
#ifndef ANDROID_HARDWARE_YANGYANG_V1_0_YANGYANG_H
#define ANDROID_HARDWARE_YANGYANG_V1_0_YANGYANG_H

#include <android/hardware/yangyang/1.0/IYangyang.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace yangyang {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct Yangyang : public IYangyang {
    // Methods from IYangyang follow.
    Return<void> helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.

};

// FIXME: most likely delete, this is only for passthrough implementations
// extern "C" IYangyang* HIDL_FETCH_IYangyang(const char* name);

}  // namespace implementation
}  // namespace V1_0
}  // namespace yangyang
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_YANGYANG_V1_0_YANGYANG_H
```

去掉extern注释，采用直通模式。

```sql
// extern "C" IYangyang* HIDL_FETCH_IYangyang(const char* name);
```

Yangyang.cpp

```cpp
//hardware/interfaces/yangyang/1.0/default/Yangyang.cpp
#include "Yangyang.h"

namespace android {
namespace hardware {
namespace yangyang {
namespace V1_0 {
namespace implementation {

// Methods from IYangyang follow.
Return<void> Yangyang::helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) {
    char buf[100];
    ::memset(buf, 0x00, 100);
    ::snprintf(buf, 100, "->>> Yangyang::helloWorld | Hello World, %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    return Void();
}


// Methods from ::android::hidl::base::V1_0::IBase follow.

//IYangyang* HIDL_FETCH_IYangyang(const char* /* name */) {
//    return new Yangyang();
//}

}  // namespace implementation
}  // namespace V1_0
}  // namespace yangyang
}  // namespace hardware
}  // namespace android
```

去掉注释．

```perl
//IYangyang* HIDL_FETCH_IYangyang(const char* /* name */) {
//    return new Yangyang();
//}
```

这些都是工具生成的代码，接下来我们来完善下接口．

主要修改的就是Yangyang.cpp代码`helloWorld`函数．

```php
Return<void> Yangyang::helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) {
    char buf[100];
    ::memset(buf, 0x00, 100);
    ::snprintf(buf, 100, "->>> Yangyang::helloWorld | Hello World, %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    return Void();
}
```

接着使用脚本来更新Makefile，下面这个文件用于更新 HAL 和 VTS 模块的 Android 生成文件的脚本。执行完成之后，会在自定义的package目录生成Android.bp和Android.mk

```shell
./hardware/interfaces/update-makefiles.sh
```

{{< image classes="fancybox center fig-100" src="/hidl/hidl_4.png" thumbnail="/hidl/hidl_4.png" title="">}}

## 2.4编译

因为已经生成了mk和bp文件，可以通过编译来生成android.hardware.yangyang@1.0-impl.so

```shell
$ ./hardware/interfaces/update-makefiles.sh
$ mmm hardware/interfaces/yangyang/1.0
```

可以通过mk和bp文件得知，实际上bp文件会生成对应的五个头文件，一个实现cpp文件和一个动态库android.hardware.yangyang@1.0.so。关于Android.bp文件语法和使用，可以点击[这里](https://blog.csdn.net/tkwxty/article/details/104395820)。

```go
//hardware/interfaces/yangyang/1.0/Android.bp
//在本例中会生成一个实现的cpp类，android/hardware/yangyang/1.0/YangyangAll.cpp
//在本例中会生成五个头文件，
//"android/hardware/yangyang/1.0/IYangyang.h",
//"android/hardware/yangyang/1.0/IHwYangyang.h",
//"android/hardware/yangyang/1.0/BnHwYangyang.h",
//"android/hardware/yangyang/1.0/BpHwYangyang.h",
//"android/hardware/yangyang/1.0/BsYangyang.h",
// 除此之外，还会生成一个android.hardware.yangyang@1.0.so的动态库
// This file is autogenerated by hidl-gen. Do not edit manually.

filegroup {
    name: "android.hardware.yangyang@1.0_hal",
    srcs: [
        "IYangyang.hal",
    ],
}

genrule {
    name: "android.hardware.yangyang@1.0_genc++",
    tools: ["hidl-gen"],
    cmd: "$(location hidl-gen) -o $(genDir) -Lc++-sources -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport android.hardware.yangyang@1.0",
    srcs: [
        ":android.hardware.yangyang@1.0_hal",
    ],
    out: [
        "android/hardware/yangyang/1.0/YangyangAll.cpp",
    ],
}

genrule {
    name: "android.hardware.yangyang@1.0_genc++_headers",
    tools: ["hidl-gen"],
    cmd: "$(location hidl-gen) -o $(genDir) -Lc++-headers -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport android.hardware.yangyang@1.0",
    srcs: [
        ":android.hardware.yangyang@1.0_hal",
    ],
    out: [
        "android/hardware/yangyang/1.0/IYangyang.h",
        "android/hardware/yangyang/1.0/IHwYangyang.h",
        "android/hardware/yangyang/1.0/BnHwYangyang.h",
        "android/hardware/yangyang/1.0/BpHwYangyang.h",
        "android/hardware/yangyang/1.0/BsYangyang.h",
    ],
}

cc_library {
    //这里的so跟下面的impl.so区别在于，这个更多的是接口，没有实现
    name: "android.hardware.yangyang@1.0",
    defaults: ["hidl-module-defaults"],
    generated_sources: ["android.hardware.yangyang@1.0_genc++"],
    generated_headers: ["android.hardware.yangyang@1.0_genc++_headers"],
    export_generated_headers: ["android.hardware.yangyang@1.0_genc++_headers"],
    vendor_available: true,
    vndk: {
        enabled: true,
    },
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libhwbinder",
        "liblog",
        "libutils",
        "libcutils",
    ],
    export_shared_lib_headers: [
        "libhidlbase",
        "libhidltransport",
        "libhwbinder",
        "libutils",
    ],
}
```

mk文件会生成对应的java文件

```makefile
#hardware/interfaces/yangyang/1.0/Android.mk
#在本例中会生成编译出来的jar包
# android.hardware.yangyang-V1.0-java.jar和android.hardware.yangyang-V1.0-java-static.jar
#在本例中会生成java的hidl接口，类似aidl的java接口类
#  (intermediates)/android/hardware/yangyang/V1_0/IYangyang.java
# This file is autogenerated by hidl-gen. Do not edit manually.

LOCAL_PATH := $(call my-dir)

################################################################################

include $(CLEAR_VARS)
LOCAL_MODULE := android.hardware.yangyang-V1.0-java
LOCAL_MODULE_CLASS := JAVA_LIBRARIES

intermediates := $(call local-generated-sources-dir, COMMON)

HIDL := $(HOST_OUT_EXECUTABLES)/hidl-gen$(HOST_EXECUTABLE_SUFFIX)

LOCAL_JAVA_LIBRARIES := \
    android.hidl.base-V1.0-java \

#
# Build IYangyang.hal
#
GEN := $(intermediates)/android/hardware/yangyang/V1_0/IYangyang.java
$(GEN): $(HIDL)
$(GEN): PRIVATE_HIDL := $(HIDL)
$(GEN): PRIVATE_DEPS := $(LOCAL_PATH)/IYangyang.hal
$(GEN): PRIVATE_OUTPUT_DIR := $(intermediates)
$(GEN): PRIVATE_CUSTOM_TOOL = \
        $(PRIVATE_HIDL) -o $(PRIVATE_OUTPUT_DIR) \
        -Ljava \
        -randroid.hardware:hardware/interfaces \
        -randroid.hidl:system/libhidl/transport \
        android.hardware.yangyang@1.0::IYangyang

$(GEN): $(LOCAL_PATH)/IYangyang.hal
	$(transform-generated-source)
LOCAL_GENERATED_SOURCES += $(GEN)
include $(BUILD_JAVA_LIBRARY)

################################################################################

include $(CLEAR_VARS)
LOCAL_MODULE := android.hardware.yangyang-V1.0-java-static
LOCAL_MODULE_CLASS := JAVA_LIBRARIES

intermediates := $(call local-generated-sources-dir, COMMON)

HIDL := $(HOST_OUT_EXECUTABLES)/hidl-gen$(HOST_EXECUTABLE_SUFFIX)

LOCAL_STATIC_JAVA_LIBRARIES := \
    android.hidl.base-V1.0-java-static \

#
# Build IYangyang.hal
#
GEN := $(intermediates)/android/hardware/yangyang/V1_0/IYangyang.java
$(GEN): $(HIDL)
$(GEN): PRIVATE_HIDL := $(HIDL)
$(GEN): PRIVATE_DEPS := $(LOCAL_PATH)/IYangyang.hal
$(GEN): PRIVATE_OUTPUT_DIR := $(intermediates)
$(GEN): PRIVATE_CUSTOM_TOOL = \
        $(PRIVATE_HIDL) -o $(PRIVATE_OUTPUT_DIR) \
        -Ljava \
        -randroid.hardware:hardware/interfaces \
        -randroid.hidl:system/libhidl/transport \
        android.hardware.yangyang@1.0::IYangyang

$(GEN): $(LOCAL_PATH)/IYangyang.hal
	$(transform-generated-source)
LOCAL_GENERATED_SOURCES += $(GEN)
include $(BUILD_STATIC_JAVA_LIBRARY)

include $(call all-makefiles-under,$(LOCAL_PATH))
```

## 2.5构建binder service

### 2.5.1 添加rc，service.cpp

虽然有了库，但是我们还需要构建binder通信服务，创建一个rc文件(rc文件是被init进程访问的)

```shell
$vim hardware/interfaces/yangyang/1.0/default/android.hardware.yangyang@1.0-service.rc
```

android.hardware.yangyang@1.0-service.rc

```shell
# hardware/interfaces/yangyang/1.0/default/android.hardware.yangyang@1.0-service.rc
service yangyang_service_hal /vendor/bin/hw/android.hardware.yangyang@1.0-service
    class hal
    user system
    group system
```

设定yangyang_service_hal服务，让rc文件启动的时候运行yangyang_service_hal的进程。

进程的入口就是下面的service.cpp文件的main函数

```shell
vim hardware/interfaces/yangyang/1.0/default/service.cpp
```

具体内容

```cpp
//hardware/interfaces/yangyang/1.0/default/service.cpp
#define LOG_TAG "android.hardware.yangyang@1.0-service"
#include <android/hardware/yangyang/1.0/IYangyang.h>
#include <hidl/LegacySupport.h>
using android::hardware::yangyang::V1_0::IYangyang;
using android::hardware::defaultPassthroughServiceImplementation;

int main(){
    printf("hidl yangyang service is starting\n");
    return defaultPassthroughServiceImplementation<IYangyang>();

}
```

在Android.bp中添加yangyang_service_hal服务的编译：

```shell
vim hardware/interfaces/yangyang/1.0/default/Android.bp 
```

Android.bp中添加

```go
//hardware/interfaces/yangyang/1.0/default/Android.bp 
cc_library_shared {
    //通过之前编译的android.hardware.yangyang@1.0.so，Yangyang.cpp用来生成实现的so
    //这个实现的so为android.hardware.yangyang@1.0-impl.so
    name: "android.hardware.yangyang@1.0-impl",
    relative_install_path: "hw",
    proprietary: true,
    srcs: [
        "Yangyang.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.yangyang@1.0",
    ],
}

cc_binary {
    //这个是一个全局唯一的名称，代表编译出来的是android.hardware.yangyang@1.0-service进程
    name: "android.hardware.yangyang@1.0-service",
    relative_install_path: "hw",
    proprietary: true,
    //这个是service启动的rc文件
    init_rc: ["android.hardware.yangyang@1.0-service.rc"],
    //这个是service的源文件
    srcs: ["service.cpp"],

    shared_libs: [
        "liblog",
        "libcutils",
        "libdl",
        "libbase",
        "libutils",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "android.hardware.yangyang@1.0",
    ],
}
```

最后的目录结构

{{< image classes="fancybox center fig-100" src="/hidl/hidl_5.png" thumbnail="/hidl/hidl_5.png" title="">}}

### 2.5.2注册hidl到manifest中

为了能够定义之后让后续测试的客服端访问到，需要把这个hidl服务注册到manifest中。这个目的是为了在init进程的时候，去自动解析rc文件，找到对应的hidl的服务。当然也可以直接在设备中修改，笔者设备是红米手机，目录位于/vendor/manifest.xml中

{{< image classes="fancybox center fig-100" src="/hidl/hidl_9.png" thumbnail="/hidl/hidl_9.png" title="">}}



```xml
<!-- 增加hidl的一个服务 -->
<hal format="hidl">
    <name>android.hardware.yangyang</name>
    <transport>hwbinder</transport>
    <impl level="generic"></impl>
    <version>1.0</version>
    <interface>
        <name>IYangyang</name>
        <instance>default</instance>
    </interface>
</hal>    
```

## 2.6最终编译

编译如下

```shell
$ mmm hardware/interfaces/yangyang/1.0/default/
```

编译后生成的文件

```tex
/vendor/lib64/hw/android.hardware.yangyang@1.0-impl.so
/vendor/etc/init/android.hardware.yangyang@1.0-service.rc 
/vendor/bin/hw/android.hardware.yangyang@1.0-service
/system/lib64/android.hardware.yangyang@1.0.so 
```

## 2.7验证服务端代码

笔者这里用到了两种方法来验证。

第一种方法主要是在同目录下创造一个test目录，用这个客户端去测试这个服务端。

第二种方法主要用jar包，然后用源码编译app，最终通过手动click，用toast空间显示是否获取服务端成功。

### 2.7.1c++验证

#### 2.7.1.1增加编译文件和测试源文件

{{< image classes="fancybox center fig-100" src="/hidl/hidl_6.png" thumbnail="/hidl/hidl_6.png" title="">}}

这个test目录增加了两个文件，一个编译文件Android.bp和测试文件yangyangTest.cpp

```go
//hardware/interfaces/yangyang/1.0/test/Android.bp
cc_binary {
    relative_install_path: "hw",
    defaults: ["hidl_defaults"],
    //生成一个yangyang_client进程，源文件为yangyangTest.cpp
    name: "yangyang_client",
    proprietary: true,
    srcs: ["yangyangTest.cpp"],

    shared_libs: [
        "liblog",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.yangyang@1.0",
    ],

}
```

测试实现文件

```c++
//hardware/interfaces/yangyang/1.0/test/yangyangTest.cpp
#include <android/hardware/yangyang/1.0/IYangyang.h>
#include <hidl/Status.h>
#include <hidl/LegacySupport.h>
#include <utils/misc.h>
#include <hidl/HidlSupport.h>
#include <stdio.h>

using ::android::hardware::hidl_string;
using ::android::sp;
using android::hardware::yangyang::V1_0::IYangyang;

int main(){
    android::sp<IYangyang> service = IYangyang::getService();
    if (service == nullptr){
        printf("->>> yangyang48 | Failed to get service\n");
        return -1;
    }

    service->helloWorld("I am Yangyang48", [&](hidl_string result){
        printf("%s\n", result.c_str());
    });
    return 0;
}
```



#### 2.7.1.2编译

```shell
# 执行第一条命令是为了更新hardware/interfaces/yangyang/目录下的Android.bp文件，自动增加1.0/test的目录
$ ./hardware/interfaces/update-makefiles.sh
$ mmm hardware/interfaces/yangyang/1.0/default/
```

编译完成之后，将文件导入到

```tex
/vendor/bin/hw/yangyang_client
```

#### 2.7.1.3测试方法

首先运行服务端进程，进程启动后会出现日志hidl yangyang service is starting

```shell
./android.hardware.yangyang@1.0-service
```

然后启动测试端进程，这个时候服务端就会和客户端交互，打印成功的日志->>> Yangyang::helloWorld | Hello World, I am Yangyang48。

```shell
./yangyang_client
```



### 2.7.2jar包验证

#### 2.7.2.1更改mk并编译jar包

可以知道，包目录下的mk文件实际上跟java相关，除了编译java的interface之外，还会编译对应的jar包。为了编译出classes.jar，需要修改hardware/interfaces/yangyang/1.0目录的Android.mk

```makefile
#hardware/interfaces/yangyang/1.0/Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
# 增加一个LOCAL_JACK_ENABLED disable的方式，这样编译的时候就不走jack编译了
LOCAL_JACK_ENABLED := disabled
LOCAL_MODULE := android.hardware.yangyang-V1.0-java
LOCAL_MODULE_CLASS := JAVA_LIBRARIES
...
```

#### 2.7.2.2jar包集成到Android源码中

具体目录如下所示。

{{< image classes="fancybox center fig-100" src="/hidl/hidl_7.png" thumbnail="/hidl/hidl_7.png" title="">}}

我们这边只看一个主界面MainActivity，注册AndroidManifest.xml和mk的编译脚本

MainActivity.java，**笔者在主界面中用的还是v7的包，其实也可以用androidx的包**

```java
//packages/apps/HIDLdemo/src/com/example/hidldemo/MainActivity.java
package com.example.hidldemo;

import android.hardware.yangyang.V1_0.IYangyang;
import android.os.Bundle;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {
    IYangyang iYangyangService;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        try {
            iYangyangService = IYangyang.getService(); //获取服务
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    public void hidlTest(View view){
        if (iYangyangService != null){
            Log.d("MainActivity", "Yangyang48 | service is connect.");
            String s = null;
            try {
                s = iYangyangService.helloWorld("I am Yangyang48_java");//调用HAL层接口
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            Log.d("MainActivity", s);
            Toast.makeText(this, s, Toast.LENGTH_LONG).show();
        }
    }
}
```

AndroidManifest.xml，这个跟默认的app写法一致

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.hidldemo">
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Android.mk

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
#定义app的apk名
LOCAL_PACKAGE_NAME := HIDLdemo
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_MODULE_TAGS :=optional
LOCAL_STATIC_JAVA_LIBRARIES := android-support-v4
# 这里可以加入androidx的包，需要在同目录下也同样导入
LOCAL_STATIC_JAVA_LIBRARIES += android-support-v7-appcompat
# 增加yangyang的jar包，这个步骤不能缺
LOCAL_STATIC_JAVA_LIBRARIES += android.hardware.yangyang-V1.0-java-static

LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
LOCAL_RESOURCE_DIR += prebuilts/sdk/current/support/v7/appcompat/res

LOCAL_CERTIFICATE := platform
LOCAL_AAPT_FLAGS := --auto-add-overlay
include $(BUILD_PACKAGE)
```



# 3总结

HIDL主要通过hwbinder注册，通过hwservicemanager来管理，通过一个客户端和一个服务端，总的来实现一个hidl的系统。本文主要简述关于HIDL的使用方式。通过自定义一个HIDL服务端，通过c++和java两种方式的客户端，去调用到服务端的最终实现。

当然HIDL的精髓不只是一个简单的小demo就可以描述清楚，这里笔者做一个抛砖引玉的作用，后续会持续更新关于HIDL方面的文章。

hwbinder的关系图如下图所示。

{{< image classes="fancybox center fig-100" src="/hidl/hidl_8.png" thumbnail="/hidl/hidl_8.png" title="">}}

# 4下载

本文源码下载，点击[这里](https://github.com/YangYang48/project/tree/master/MyHIDL)



# 参考

[[1] android官方文档, 使用Binder IPC, 2020.](https://source.android.google.cn/devices/architecture/hidl/binder-ipc)

[[2] 木叶风神, Android HIDL学习（2） ---- HelloWorld, 2018.](https://www.jianshu.com/p/ca6823b897b5)

[[3] IngresGe, Android10.0 Binder通信原理(一)Binder、HwBinder、VndBinder概要 --- 注册回调, 2020.](https://blog.csdn.net/yiranfeng/article/details/105175578)

[[4] IngresGe, Native层HIDL服务的注册原理-Android10.0 HwBinder通信原理(六), 2020.](https://blog.csdn.net/yiranfeng/article/details/107966454)

[[5] IngresGe, JAVA层HIDL服务的注册原理-Android10.0 HwBinder通信原理(八), 2020.](https://blog.csdn.net/yiranfeng/article/details/108037698)

[[6] liujun3512159, Android HIDL学习(3) --- 注册回调, 2022.](https://blog.csdn.net/liujun3512159/article/details/123103406)

[[7] 嵌入式Linux,, binder，hwbinder，vndbinder之间的关系, 2020.](https://linus.blog.csdn.net/article/details/104289913)

[[8] 私房菜,Android HIDL 中的数据类型, 2019.](https://justinwei.blog.csdn.net/article/details/86531179)

[[9] liujun3512159,HIDL实战笔记, 2022.](https://blog.csdn.net/liujun3512159/article/details/123103620)

[[10] 开发小院,CameraProvider进程启动流程, 2020.](http://www.javashuo.com/article/p-ghdvyhzt-mk.html)

[[11] 私房菜,Android HIDL 中的函数, 2019.](https://blog.csdn.net/shift_wwx/article/details/86531137)

[[12] Gunder,HIDL最全编译流程, 2018.](https://blog.csdn.net/u013357557/article/details/84561652)

[[13] Gunder,HIDL概述, 2018.](https://blog.csdn.net/u013357557/article/details/84561457)

[[14] nginux,source /build/envsetup.sh和lunch）, 2015.](https://blog.csdn.net/GetNextWindow/article/details/48160569?spm=1001.2101.3001.6650.12&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-12.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-12.pc_relevant_default&utm_relevant_index=18)

[[15] IT先森,Android.bp入门指南之浅析Android.bp语法, 2020.](https://blog.csdn.net/tkwxty/article/details/104395820)
