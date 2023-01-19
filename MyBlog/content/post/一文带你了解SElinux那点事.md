---
title: "一文带你了解SElinux那点事"
date: 2023-01-07T5:00:00
thumbnailImagePosition: left
thumbnailImage: SElinux/selinux_thumb.jpg
coverImage: SElinux/selinux_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- SElinux
- 2023
- January
tags:
- DAC
- MAC
- TE
- avc
- cts
- 源码
- makefile
- 正则表达式
showSocial: false
---
SELinux(Security Enhanced Linux)是一个Linux内核模块，也是一个安全子系统。用于最大限度地减小系统中服务进程可访问的资源（最小权限原则）。
<!--more-->
# 1介绍

SELinux(Security Enhanced Linux是一个Linux内核模块，也是Linux的一个安全子系统
SELinux主要作用: 最大限度地减小系统中服务进程可访问的资源（最小权限原则）

SELinux存在的意义是什么？
更安全吧，即使某个进程被黑了或者直接被获取了root权限，也没有拥有所有权限，没有在te策略文件里面声明的权限都不允许

前面提到了即使被root了，也不一定有权限操作，但是我们在板子上su获取root之后，不是拥有权限setenforce吗，直接关了SELinux不就完了？
在userdebug版本中，为了方便调试，Google原生是有给su足够的权限去操作很多事情的，基本上root就是无敌的存在，所以我们可以直接用setenforce命令去关闭SELinux

user版本中，权限收紧，即使是su获取了root权限，有很多命令都是没有权限执行的，也就是说user版本的系统打开SELinux之后才是最安全的


## 1.1DAC 和 MAC

正如很多文章都会提到的两个概念：DAC 和 MAC

DAC：Discretionary Access Control 自主访问控制

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_1.png" thumbnail="/SElinux/selinux_1.png" title="">}}

进程理论上所拥有的权限与执行它的用户的权限相同，只要获得root权限，几乎无所不能

比如上面的图片，user是没有权限写入test.txt的

但是可以获取root权限，接着就能写入了

MAC：Mandatory Access Control 强制访问控制

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_2.png" thumbnail="/SElinux/selinux_2.png" title="">}}

相比DAC，这里进程和文件都被打上了安全上下文，user在写入test.txt的时候，除了要拥有写权限，同时也要在规则库里面声明权限，才能正常写入。

任何进程想在 SELinux 系统上干任何事情，都必须在安全策略文件中赋予权限，凡是没有出现在安全策略文件中的权限都不拥有；即使你是root，也不一定拥有权限操作，这个并不能做到防御一切攻击，但是能将损失降到最小


## 1.2SElinux规则

### 1.2.1安全上下文（Security Context）

完整的 Security Context 字符串为：

```bash
user:role:type[:range]
```

SELinux给每一个文件、进程、属性、服务都给打上一个标签，叫安全上下文（Security Context）

用下面的命令可以分别查看对应的安全上下文

- id -Z 查看当前进程的scontext

```bash
picasso:/ $ id -Z
context=u:r:shell:s0
 
picasso:/ # id -Z
context=u:r:su:s0
```

- ls -Z 查看文件的scontext

```bash
picasso:/ # ps -AZ | grep surface
u:r:surfaceflinger:s0          system        2930     1   67104  12856 SyS_epoll_wait      0 S surfaceflinger
```

- getprop -Z 查看属性的scontext

```bash
picasso:/ # getprop -Z | grep selinux
[ro.boot.selinux]: [u:object_r:default_prop:s0]
[ro.boottime.init.selinux]: [u:object_r:boottime_prop:s0]
[selinux.restorecon_recursive]: [u:object_r:restorecon_prop:s0]
```

- user：用户，Android里面就一个 u
- role：角色，进程用 r ，文件用 object_r
- type ：类型，对进程而言也通常叫域，都是一个概念
- range，也就是上面的s0，是Multi-Level Security（MLS）的等级，这个有个概念就行了，MLS 将系统的进程和文件进行了分级，不同级别的资源需要对应级别的进程才能访问

安全上下文其实就是一个字符串，我们最需要关注的就是type，MAC基本管理单位是TEAC（Type Enforcement Accesc Control）

### 1.2.2TE （Type Enforcement）

#### 1.2.2.1 SELinux策略规则语句格式

```bash
rules domains types:classes permissions;
```

- rules 包括的规则有四个：allow、neverallow、dontaudit、auditallow

  allow ： 允许主体对客体进行操作

  neverallow ：拒绝主体对客体进行操作

  dontaudit ： 表示不记录某条违反规则的决策信息

  auditallow ：记录某项决策信息，通常 SElinux 只记录失败的信息，应用这条规则后会记录成功的决策信息

- domains 一个进程或一组进程的标签。也称为域类型，因为它只是指进程的类型

- types 一个对象（例如：文件、属性）或一组对象的标签

- classes 要访问的对象（例如：文件、属性）的类型

- permissions 要执行的操作（例如：读、写）

规则库以白名单的形式存在，要求所有允许的操作都必须要用allow规则来定义，不然都会被禁止操作

domains、types、classes、permissions都可以是单独的一个，也可以是一个集合，用花括号括起来

其中的domains和types，既可以指定一个type，也可以指定一组type（也就是attribute），这些定义在te文件里面

classes的定义在/android/system/sepolicy/private/security_classes

permissions的定义在/android/system/sepolicy/private/access_vectors

上面这些规则我们用最多的就是allow和neverallow，搞清楚这两个之后其他的规则是一样的道理

#### 1.2.2.2type和attribute

**1、type的定义规则**

```bash
type type_name [alias alias_set] [, attribute_set] ;
```

比如system_app.te里面定义：

```bash
type system_app, domain;
```

type的名字是system_app

alias定义的是别名，在Android里面这个基本没有用到，见不到它的身影

逗号后面的是属性（attribute），也就是说domain是一个属性，type可以关联多个属性，后面用逗号分割，比如：

```bash
type traced_probes, domain, coredomain, mlstrustedsubject;
```

表示traced_probes这个type关联上了domain、coredomain、mlstrustedsubject这三个属性

如果在type定义的时候没有关联属性，后面也可以通过typeattribute关键字来关联

```bash
typeattribute type_name attribute_name;
```

**2、attribute的定义规则**

```bash
attribute attribute_name;
```

代码里面属性的定义在：

/android/system/sepolicy/public/attributes

```bash
# All types used for processes.
attribute domain;
 
# All types used for files that can exist on a labeled fs.
# Do not use for pseudo file types.
# On change, update CHECK_FC_ASSERT_ATTRS
# definition in tools/checkfc.c.
attribute file_type;
 
# All types used for domain entry points.
attribute exec_type;
 
# All types used for /data files.
attribute data_file_type;
expandattribute data_file_type false;
# All types in /data, not in /data/vendor
attribute core_data_file_type;
expandattribute core_data_file_type false;
# All types in /vendor
attribute vendor_file_type;
......
```

**3、type和attribute的区别**

关于type和attribute，一开始接触，很难理解这两个的区别

它们两个有很多类似的点，但是又不大一样：

- type和attribute都是不能重复定义的，它们位于同一个命令空间，也就意味着定义一个type之后，就不能定义一个同名的attribute

```bash
下面的都会定义会报错提示，重复定义

type system_app, domain;
attribute system_app;
 
 
type system_app, domain;
type system_app, coredomain, mlstrustedsubject;
```

- type就是一个类型；attribute可以找到所有关联了这个属性的type，可以把它当成一个集合、组、标签

attribute其实就是为了方便书写te规则，我们可以用attribute找到所有关联了属性的type集合，而不用每一个都写出来



#### 1.2.2.3例子

上面的概念比较抽象，通常我们能比较快理解的是如何在te文件里使用allow语句添加权限，但是对于neverallow语句所表达的意思就很难理解了

下面我用例子来解释（**提到的权限只是示例，并不一定是代码里面的，也有可能违反neverallow规则**）

1、假如有几个文件，以及它们的安全上下文：

```bash
picasso:/ # ls -Z /data/test.txt
u:object_r:system_data_file:s0 /data/test.txt
 
picasso:/ # ls -Z /data/vendor/log_test.txt
u:object_r:vendor_data_file:s0 /data/vendor/log_test.txt
 
picasso:/ # ls -Z /data/media/media_test.txt
u:object_r:media_rw_data_file:s0 /data/media/media_test.txt
```

**2、有如下几个脚本，内容都一样**

/system/bin/testA.sh

/vendor/bin/testB.sh

/vendor/bin/testC.sh

```bash
echo "hello world" > /data/test.txt
echo "hello world" > /data/vendor/log_test.txt
echo "hello world" > /data/media/media_test.txt
```

**3、init.rc里面的定义如下：**

```rc
service testA /system/bin/testA.sh
    user root
    group root
    disabled
    oneshot
    seclabel u:r:testA:s0
 
service testB /vendor/bin/testB.sh
    user root
    group root
    disabled
    oneshot
    seclabel u:r:testB:s0
 
service testC /vendor/bin/testC.sh
    user root
    group root
    disabled
    oneshot
    seclabel u:r:testC:s0
```

**4、假设系统里面定义的type和attribute只有下面这些：**

```bash
type system_server, coredomain, domain;
type testA, domain;
type testB, domain;
type performanced, domain, coredomain;
type kernel, domain, data_between_core_and_vendor_violators;
type testC, domain;
type init, domain, data_between_core_and_vendor_violators;
 
type system_data_file, file_type, data_file_type, core_data_file_type;
type vendor_data_file, file_type, data_file_type;
type media_rw_data_file, file_type, data_file_type, core_data_file_type;
 
attribute domain;
attribute coredomain;
attribute data_between_core_and_vendor_violators;
attribute file_type;
attribute data_file_type;
attribute core_data_file_type;
```

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_10.png" thumbnail="/SElinux/selinux_10.png" title="">}}

正如前面提到的概念：

```bash
1、属性domain是type集合 ： { system_server testA testB performanced kernel testC init }
2、属性coredomain是type集合 ： { system_server performanced }
3、属性data_between_core_and_vendor_violators是type集合 ： { kernel init }
4、属性file_type是type集合 ：{ system_data_file vendor_data_file media_rw_data_file }
5、属性data_file_type是type集合 ： { system_data_file vendor_data_file media_rw_data_file }
6、属性core_data_file_type是type集合 ： { system_data_file media_rw_data_file }
```

脚本testA.sh运行起来之后，脚本的进程所在的安全上下文是u:r:testA:s0

如果要想写文件 /data/test.txt，就要给它足够的权限才行：

```te
allow testA system_data_file:file { read write open create };
```

这句话的意思是type为testA的进程，拥有对type为system_data_file的文件（file）的read write open create权限

此时写第二个文件 /data/vendor/log_test.txt 就没有权限了，因为文件log_test.txt的type是vendor_data_file

同样的得添加权限

```te
allow testA vendor_data_file:file { read write open create };
```

同理，/data/media/media_test.txt也是一样的道理

```te
allow testA media_rw_data_file:file { read write open create };
```

上面的allow语句大家很容易理解

再比如：

```te
allow coredomain data_file_type:file { read write open create };
```

这里coredomain和data_file_type是一个属性（一个type集合，而不仅仅是一个type了），等价于

```te
allow { system_server performanced } { system_data_file vendor_data_file media_rw_data_file }:file { read write open create };
```

当然，Android系统里面关联data_file_type这个属性的不止这三个，那么data_file_type其实就是所有关联了这个属性的type的集合

domain是所有进程type的总和，因为所有的进程type都要关联属性domain，不然脚本都没有办法运行起来



**5、neverallow规则**

再看看neverallow规则，其实和allow规则是一样格式的

```te
假如：
neverallow {
  domain
  -coredomain
  -data_between_core_and_vendor_violators
} {
  core_data_file_type
}:file { create open };
```

减号 - 代表去掉一个type或者一组type

还有几个：

*号表示所有内容

~号表示取反，剩下的所有内容

```te
{ domain -coredomain -data_between_core_and_vendor_violators } 等于
{ system_server testA testB performanced kernel testC init } - { system_server performanced } - { kernel init }
也就是： { testA testB testC }
```

上面的neverallow语句等价于：

```te
neverallow { testA testB testC } { system_data_file vendor_data_file media_rw_data_file }:file { create open };
```

整理成这样之后，就会发现和前面提到的规则是一模一样的：

- rules : neverallow
- domains : { testA testB testC }
- types : { system_data_file vendor_data_file media_rw_data_file }
- classes : file
- permissions : { create open }

neverallow语句的意思是：不允许testA testB testC这些type的进程，对type为system_data_file vendor_data_file media_rw_data_file的file，有create和open权限

如果我们添加了这样的权限，将会报错:

```te
allow testA system_data_file:file create；
```

之所以说neverallow和allow的书写规则是一样的，是因为如果把neverallow换成allow之后，就是允许给testA testB testC添加对类型为core_data_file_type的file的create和open权限

neverallow就是反过来，不允许你给它们这些权限

**6、题外话**

我们在添加脚本的时候，通常对应的te会有类似下面的三句：

```te
type testA, domain;
type testA_exec, exec_type, vendor_file_type, file_type;
 
init_daemon_domain(testA)
```

第一个是testA.sh脚本运行起来之后，进程的type，它的安全上下文是 u:r:testA:s0

第二个是testA.sh这个文件的type，它的安全上下文是 u:object_r:testA_exec:s0

第三个是域转移，我们这些脚本都是init进程启动的，如果没有域转移，那么脚本运行起来它的安全上下文就是 u:r:init:s0了，显然没有细分出来

那这里的意思就是type为init的进程，在运行type为testA_exec的脚本时，会把域切换到testA来

init_daemon_domain的定义在/system/sepolicy/public/te_macros

```te
#####################################
# init_daemon_domain(domain)
# Set up a transition from init to the daemon domain
# upon executing its binary.
define(`init_daemon_domain', `
domain_auto_trans(init, $1_exec, $1)
tmpfs_domain($1)
')
```



## 1.3SELinux 配置

### 1.3.1开关SELinux

SELinux的状态有两个

- enforcing 权限拒绝事件会被记录下来并强制执行
- permissive 权限拒绝事件会被记录下来，但不会被强制执行
  （当然有些还会有disable作为第三个状态，就是彻底关闭的状态）

进入系统后，可以使用setenforce命令来开关

```bash
#关闭：
picasso:/ $ setenforce 0
#打开：
picasso:/ $ setenforce 1
```

但如果要开机阶段也是关闭的状态，那就要修改到bootargs的参数了

```bash
#打开
androidboot.selinux=enforcing
 
#关闭
androidboot.selinux=permissive
```

这里面还需要注意一点，一个宏： ALLOW_PERMISSIVE_SELINUX

/android/system/core/init/Android.mk

```makefile
ifneq (,$(filter userdebug eng,$(TARGET_BUILD_VARIANT)))
init_options += \
    -DALLOW_LOCAL_PROP_OVERRIDE=1 \
    -DALLOW_PERMISSIVE_SELINUX=1 \
    -DREBOOT_BOOTLOADER_ON_PANIC=1 \
    -DWORLD_WRITABLE_KMSG=1 \
    -DDUMP_ON_UMOUNT_FAILURE=1
else
init_options += \
    -DALLOW_LOCAL_PROP_OVERRIDE=0 \
    -DALLOW_PERMISSIVE_SELINUX=0 \
    -DREBOOT_BOOTLOADER_ON_PANIC=0 \
    -DWORLD_WRITABLE_KMSG=0 \
    -DDUMP_ON_UMOUNT_FAILURE=0
endif
```

/android/system/core/init/selinux.cpp

```c++
EnforcingStatus StatusFromCmdline() {
    EnforcingStatus status = SELINUX_ENFORCING;
 
    import_kernel_cmdline(false,
                          [&](const std::string& key, const std::string& value, bool in_qemu) {
                              if (key == "androidboot.selinux" && value == "permissive") {
                                  status = SELINUX_PERMISSIVE;
                              }
                          });
 
    return status;
}
 
bool IsEnforcing() {
    if (ALLOW_PERMISSIVE_SELINUX) {
        return StatusFromCmdline() == SELINUX_ENFORCING;
    }
    return true;
}
```

对于user版本，是没有办法通过bootargs的androidboot.selinux来关闭的，一直都是enforcing状态的

上面的方法只适用于eng和userdebug版本，在user版本上，权限收缩管控得非常严格

不仅改bootargs没有作用，进入系统后，即使你是root用户，setenforce也没有权限操作

```bash
picasso:/ $ setenforce 0
setenforce: Couldn't set enforcing status to '0': Permission denied
[ 4427.467459@3] type=1404 audit(1623583196.490:218): enforcing=1 old_enforcing=0 auid=4294967295 ses=4294967295
[ 4427.477347@3] type=1400 audit(1623583199.930:219): avc: denied { setenforce } for pid=4412 comm="setenforce" scontext=u:r:shell:s0 tcontext=u:object_r:kernel:s0 tclass=security permissive=0
```

所以user版本要想关闭SELinux，只能修改代码逻辑了

## 1.3.2策略文件配置

Google原生的te在/android/system/sepolicy目录下

设备制造商客制化部分的策略文件te可以通过变量**BOARD_SEPOLICY_DIRS**来指定，是对/android/system/sepolicy/vendor目录的拓展

这是个很关键的变量，通过这个变量就可以找到你要添加的te在哪个路径

比如：

```makefile
//对应设备目录/vendor/etc/selinux
BOARD_SEPOLICY_DIRS += device/google/marlin/sepolicy
BOARD_SEPOLICY_DIRS += device/google/marlin/sepolicy/verizon
```

> 事实上除了上述之外，芯片厂商可能还会定制其他分区，比如product分区
>
> ```makefile
> //对应设备目录/product/etc/selinux
> PRODUCT_PRIVATE_SEPOLICY_DIRS += device/google/marlin/sepolicy/private
> PRODUCT_PUBLIC_SEPOLICY_DIRS += device/google/marlin/sepolicy/public
> ```



还有两个变量也是对策略文件拓展，这个大家会比较陌生

```makefile
BOARD_PLAT_PUBLIC_SEPOLICY_DIR    #对PLAT_PUBLIC_POLICY的拓展  即对/android/system/sepolicy/public目录的拓展
BOARD_PLAT_PRIVATE_SEPOLICY_DIR   #对PLAT_PRIVATE_POLICY的拓展 即对/android/system/sepolicy/private目录的拓展
```

> **private**代表目录下面的文件，只对BOARD平台策略可见/system/sepolicy，通常为这个目录
>
> **public**代表目录下面所有的策略都可见，非平台的目录在/device下面，可以看到/system/sepolicy/public目录下面定义的策略

## 1.4判断是否为SELinux导致的权限问题

判断是否为SELinux导致的权限问题很简单

- 打开SELinux情况下必现问题
- 关闭SELinux之后如果问题消失，那基本上就可以认定是权限问题导致的

接着重要的就是抓取avc报错信息，添加权限

**kernel log**

```log
picasso:/ $ dmesg | grep avc
[ 8839.922581@3] type=1400 audit(1623587612.374:455): avc: denied { setattr } for pid=4570 comm="chmod" name="video" dev="mmcblk0p21" ino=384 scontext=u:r:system_app:s0 tcontext=u:object_r:system_data_file:s0 tclass=dir permissive=0
```

**Andriod log**

```log
picasso:/ $ logcat | grep avc
01-01 08:00:09.652  3259  3259 I init    : type=1400 audit(0.0:17): avc: denied { entrypoint } for path="/vendor/bin/testA.sh" dev="mmcblk0p16" ino=705 scontext=u:r:testA:s0 tcontext=u:object_r:vendor_file:s0 tclass=file permissive=1
```

如果是SELinux权限问题，那必然是100%必现的

前面提到的会出现第一次开机有权限问题，后面开机就没有问题了

找了很久的原因，发现是其他脚本会操作SELinux的开关，没有复现的情况刚好被关掉了

# 2开机初始化与策略文件编译过程

## 2.1SELinux开机初始化

### 2.1.1init.cpp

代码路径：android/system/core/init/init.cpp

main函数：

```c++
if (is_first_stage) {  // 第一阶段初始化
    ...
 
    // Set up SELinux, loading the SELinux policy.
    SelinuxSetupKernelLogging();
    SelinuxInitialize();
 
    ...
}
 
 
// Now set up SELinux for second stage.
SelinuxSetupKernelLogging();
SelabelInitialize();
SelinuxRestoreContext();
```

### 2.1.2selinux.cpp

代码路径：android/system/core/init/selinux.cpp

#### 2.1.2.1SelinuxSetupKernelLogging

设置log callback，可以调用selinux_log把log写入到kmsg里面

#### 2.1.2.2SelinuxInitialize

selinux 初始化，从二进制策略文件里面读取策略，加载到内核

```c++
void SelinuxInitialize() {
    Timer t;
 
    LOG(INFO) << "Loading SELinux policy";
    if (!LoadPolicy()) { // 加载SELinux策略
        LOG(FATAL) << "Unable to load SELinux policy";
    }
 
    bool kernel_enforcing = (security_getenforce() == 1); // 从kernel获取SELinux的状态，和getenforce的实现一样
    bool is_enforcing = IsEnforcing();  // 从bootargs里面获取
    if (kernel_enforcing != is_enforcing) {
        if (security_setenforce(is_enforcing)) { // 如果kernel里面的SELinux状态和bootargs的不一致，要设置成bootargs里面传过来的值
            PLOG(FATAL) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");
        }
    }
 
    if (auto result = WriteFile("/sys/fs/selinux/checkreqprot", "0"); !result) {  // 由内核强制执行检查保护
        LOG(FATAL) << "Unable to write to /sys/fs/selinux/checkreqprot: " << result.error();
    }
 
    // init's first stage can't set properties, so pass the time to the second stage.
    setenv("INIT_SELINUX_TOOK", std::to_string(t.duration().count()).c_str(), 1);
}
 
bool LoadPolicy() { // 从Android8.0之后，因为Project Treble，system和vendor策略分离，所以Android P上走的是LoadSplitPolicy
    return IsSplitPolicyDevice() ? LoadSplitPolicy() : LoadMonolithicPolicy();
}
```

IsEnforcing 从bootargs里面的androidboot.selinux取值; security_getenforce 从kernel取值，这个值和getenforce拿到的是一样的

#### 2.1.2.3 checkreqprot

设置"checkreqprot"标记的初始值。

"0"表示由内核强制执行检查保护(包括其中隐含的所有执行保护)

"1"表示由应用程序自己主动请求执行检查保护

默认值由内核在编译时确定，也可以在运行时通过/sys/fs/selinux/checkreqprot修改

#### 2.1.2.4 LoadPolicy

policy的分割是从Android 8.0之后开始的，

Android低版本用的是LoadMonolithicPolicy，8.0之后调用的是LoadSplitPolicy

sepolicy分离

> #public - policy exported on which non-platform policy developers may write
> #additional policy. types and attributes are versioned and included in
> #delivered non-platform policy, which is to be combined with platform policy. 导出的策略，非平台策略开发人员可以在其上编写附加策略
> 类型和属性被版本化并包含在交付的非平台策略中，该策略将与平台策略相结合
> 使用BOARD_PLAT_PUBLIC_SEPOLICY_DIR来添加拓展

types和attributes生成在vendor分区的会带上版本，比如bootanim_30_0，实现在cil_android_attributize

> #private - platform-only policy required for platform functionality but which
> #is not exported to vendor policy developers and as such may not be assumed
> #to exist. 平台功能所需的纯平台策略，但不会导出到供应商策略开发人员，因此可能不存在。

> #vendor - vendor-only policy required for vendor functionality. This policy can
> #reference the public policy but cannot reference the private policy. This
> #policy is for components which are produced from the core/non-vendor tree and
> #placed into a vendor partition. 供应商功能所需的供应商专用策略。此策略可以引用公共策略，但不能引用私有策略。此策略适用于从核心/非供应商树生成并放置到供应商分区中的组件。

> #mapping - This contains policy statements which map the attributes
> #exposed in the public policy of previous versions to the concrete types used
> #in this policy to ensure that policy targeting attributes from public
> #policy from an older platform version continues to work. 它包含策略语句，这些语句将以前版本的公共策略中公开的属性映射到此策略中使用的具体类型，以确保来自较旧平台版本的公共策略的策略目标属性继续工作。

#### 2.1.2.5LoadSplitPolicy

```c++
bool LoadSplitPolicy() {
    std::string precompiled_sepolicy_file;
    if (FindPrecompiledSplitPolicy(&precompiled_sepolicy_file)) { // 先找odm分区，再找vendor分区
        unique_fd fd(open(precompiled_sepolicy_file.c_str(), O_RDONLY | O_CLOEXEC | O_BINARY));  // open file
        if (fd != -1) {
            if (selinux_android_load_policy_from_fd(fd, precompiled_sepolicy_file.c_str()) < 0) { // 加载到kernel
                LOG(ERROR) << "Failed to load SELinux policy from " << precompiled_sepolicy_file;
                return false;
            }
            return true;
        }
    }
    // No suitable precompiled policy could be loaded
 
    LOG(INFO) << "Compiling SELinux policy";
 
    // Determine the highest policy language version supported by the kernel
    set_selinuxmnt("/sys/fs/selinux");
    int max_policy_version = security_policyvers();
    if (max_policy_version == -1) {
        PLOG(ERROR) << "Failed to determine highest policy version supported by kernel";
        return false;
    }
 
    // We store the output of the compilation on /dev because this is the most convenient tmpfs
    // storage mount available this early in the boot sequence.
    char compiled_sepolicy[] = "/dev/sepolicy.XXXXXX";
    unique_fd compiled_sepolicy_fd(mkostemp(compiled_sepolicy, O_CLOEXEC));
    if (compiled_sepolicy_fd < 0) {
        PLOG(ERROR) << "Failed to create temporary file " << compiled_sepolicy;
        return false;
    }
 
    // Determine which mapping file to include
    std::string vend_plat_vers;
    if (!GetVendorMappingVersion(&vend_plat_vers)) {
        return false;
    }
    std::string mapping_file("/system/etc/selinux/mapping/" + vend_plat_vers + ".cil");
 
    // vendor_sepolicy.cil and plat_pub_versioned.cil are the new design to replace
    // nonplat_sepolicy.cil.
    std::string plat_pub_versioned_cil_file("/vendor/etc/selinux/plat_pub_versioned.cil");
    std::string vendor_policy_cil_file("/vendor/etc/selinux/vendor_sepolicy.cil");
 
    if (access(vendor_policy_cil_file.c_str(), F_OK) == -1) {
        // For backward compatibility.
        // TODO: remove this after no device is using nonplat_sepolicy.cil.
        vendor_policy_cil_file = "/vendor/etc/selinux/nonplat_sepolicy.cil";
        plat_pub_versioned_cil_file.clear();
    } else if (access(plat_pub_versioned_cil_file.c_str(), F_OK) == -1) {
        LOG(ERROR) << "Missing " << plat_pub_versioned_cil_file;
        return false;
    }
 
    // odm_sepolicy.cil is default but optional.
    std::string odm_policy_cil_file("/odm/etc/selinux/odm_sepolicy.cil");
    if (access(odm_policy_cil_file.c_str(), F_OK) == -1) {
        odm_policy_cil_file.clear();
    }
    const std::string version_as_string = std::to_string(max_policy_version);
 
    // clang-format off
    std::vector<const char*> compile_args {
        "/system/bin/secilc",
        plat_policy_cil_file,
        "-m", "-M", "true", "-G", "-N",
        // Target the highest policy language version supported by the kernel
        "-c", version_as_string.c_str(),
        mapping_file.c_str(),
        "-o", compiled_sepolicy,
        // We don't care about file_contexts output by the compiler
        "-f", "/sys/fs/selinux/null",  // /dev/null is not yet available
    };
    // clang-format on
 
    if (!plat_pub_versioned_cil_file.empty()) {
        compile_args.push_back(plat_pub_versioned_cil_file.c_str());
    }
    if (!vendor_policy_cil_file.empty()) {
        compile_args.push_back(vendor_policy_cil_file.c_str());
    }
    if (!odm_policy_cil_file.empty()) {
        compile_args.push_back(odm_policy_cil_file.c_str());
    }
    compile_args.push_back(nullptr);
 
    if (!ForkExecveAndWaitForCompletion(compile_args[0], (char**)compile_args.data())) {
        unlink(compiled_sepolicy);
        return false;
    }
    unlink(compiled_sepolicy);
 
    LOG(INFO) << "Loading compiled SELinux policy";
    if (selinux_android_load_policy_from_fd(compiled_sepolicy_fd, compiled_sepolicy) < 0) {
        LOG(ERROR) << "Failed to load SELinux policy from " << compiled_sepolicy;
        return false;
    }
 
    return true;
}
```

LoadSplitPolicy 加载selinux 策略时，首先从预先编译好的二进制文件precompiled_sepolicy读取

如果没有找到的话，再用secilc命令编译所有的cil，重新得到一个二进制策略文件

#### 2.1.2.6FindPrecompiledSplitPolicy

这个函数会查找两个地方，/odm/etc/selinux和/vendor/etc/selinux

- /vendor/etc/selinux/precompiled_sepolicy
- /odm/etc/selinux/precompiled_sepolicy

如果有odm分区，precompiled_sepolicy会放在odm分区，否则在vendor分区，因此读取的时候也是先找odm再找vendor

```c++
bool FindPrecompiledSplitPolicy(std::string* file) {
    file->clear();
    // If there is an odm partition, precompiled_sepolicy will be in
    // odm/etc/selinux. Otherwise it will be in vendor/etc/selinux.
    static constexpr const char vendor_precompiled_sepolicy[] =
        "/vendor/etc/selinux/precompiled_sepolicy";
    static constexpr const char odm_precompiled_sepolicy[] =
        "/odm/etc/selinux/precompiled_sepolicy";
    if (access(odm_precompiled_sepolicy, R_OK) == 0) {
        *file = odm_precompiled_sepolicy;
    } else if (access(vendor_precompiled_sepolicy, R_OK) == 0) {
        *file = vendor_precompiled_sepolicy;
    } else {
        PLOG(INFO) << "No precompiled sepolicy";
        return false;
    }
 
        // 接下来下面会对哈希值做校验，分别是：
    // /system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256
    // /vendor/etc/selinux/precompiled_sepolicy.plat_and_mapping.sha256
    std::string actual_plat_id;
    if (!ReadFirstLine("/system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256", &actual_plat_id)) {
        PLOG(INFO) << "Failed to read "
                      "/system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256";
        return false;
    }
 
    std::string precompiled_plat_id;
    std::string precompiled_sha256 = *file + ".plat_and_mapping.sha256";
    if (!ReadFirstLine(precompiled_sha256.c_str(), &precompiled_plat_id)) {
        PLOG(INFO) << "Failed to read " << precompiled_sha256;
        file->clear();
        return false;
    }
    if ((actual_plat_id.empty()) || (actual_plat_id != precompiled_plat_id)) {
        file->clear();
        return false;
    }
    return true;
}
```

最后面会对哈希值做一个校验，这两个文件都是编译阶段生成的

后面我们局部编译SELinux策略文件时，一定要注意只替换precompiled_sepolicy就行了

千万不要同时替换precompiled_sepolicy 和 precompiled_sepolicy.plat_and_mapping.sha256，而忘记了/system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256，这个会导致你替换的策略文件不生效

## 2.2SELinux Policy编译

### 2.2.1二进制策略文件的生成过程

本节涉及的代码集中在：

- android/system/sepolicy/Android.mk
- android/external/selinux/secilc
- android/external/selinux/checkpolicy
- android/external/selinux/libselinux
- android/external/selinux/libsepol

其中Android mk中的一些变量：

```makefile
PLAT_PUBLIC_POLICY=system/sepolicy/public
PLAT_PRIVATE_POLICY=system/sepolicy/private
PLAT_VENDOR_POLICY=system/sepolicy/vendor
REQD_MASK_POLICY=system/sepolicy/reqd_mask
BOARD_PLAT_PUBLIC_SEPOLICY_DIR    #对PLAT_PUBLIC_POLICY的拓展
BOARD_PLAT_PRIVATE_SEPOLICY_DIR   #对PLAT_PRIVATE_POLICY的拓展
 
BOARD_SEPOLICY_DIRS  #设备制造商客制化部分的策略文件
```

### 2.2.1.1 precompiled_sepolicy是如何生成的？

先看总的流程：

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_3.png" thumbnail="/SElinux/selinux_3.png" title="">}}

1）如果定义了BOARD_ODM_SEPOLICY_DIRS，那么还会有一个odm_sepolicy.cil，当然前提还要有odm分区

```makefile
ifdef BOARD_ODM_SEPOLICY_DIRS
    all_cil_files += $(built_odm_cil)
endif
```

在 android/system/sepolicy/Android.mk 的注释里面，提到了SELinux策略文件编译成二进制策略文件的过程

> 这里面主要包含设置的几种类型文件，context和xml文件，其中xml是加了密的文件，通常会放到设备的类似/system/etc/selinux，/vendor/etc/selinux，/product/etc/selinux等。
>
> |    context名称    |             含义              |
> | :---------------: | :---------------------------: |
> |   file_contexts   | 系统中所有file_contexts上下文 |
> |  genfs_contexts   |     虚拟文件的安全上下文      |
> |  seapp_contexts   |         app安全上下文         |
> | property_contexts |        属性安全上下文         |
> | service_contexts  |      服务文件安全上下文       |





2）第二步，是针对30.0.cil、plat_pub_versioned.cil、vendor_sepolicy.cil，而plat_sepolicy.cil不需要

并且，只有在system/sepolicy/public、system/sepolicy/private下定义的types和attributes才会有加上这个版本后缀

调用过程 (build_sepolicy -> ) version_policy -> cil_android_attributize

```text
# build process for device:
# 1) convert policies to CIL:
#    - private + public platform policy to CIL
#    - mapping file to CIL (should already be in CIL form)
#    - non-platform public policy to CIL
#    - non-platform public + private policy to CIL
# 2) attributize policy
#    - run script which takes non-platform public and non-platform combined
#      private + public policy and produces attributized and versioned
#      non-platform policy
# 3) combine policy files
#    - combine mapping, platform and non-platform policy.
#    - compile output binary policy file
```

> ACP描述的是一个Android专用的cp命令，在生成system.img镜像文件的过程中是需要用到的。普通的cp命令在不同的平台（Mac
> OSX、MinGW/Cygwin和Linux）的实现略有差异，并且可能会导致一些问题，于是Android编译系统就重写了自己的cp命令，使得它在不同平台下执行具有统一的行为，并且解决普通cp命令可能会出现的问题。例如，在Linux平台上，当我们把一个文件从NFS文件系统拷贝到本地文件系统时，普通的cp命令总是会认为在NFS文件系统上的文件比在本地文件系统上的文件要新，因为前者的时间戳精度是微秒，而后者的时间戳精度不是微秒。Android专用的cp命令源码可以参考build/tools/acp目录。具体可以点击[这里](https://blog.csdn.net/kc58236582/article/details/49795865)。

3）分析Android.mk文件： android/system/sepolicy/Android.mk

BOARD_USES_ODMIMAGE 在 BoardConfig.mk 里面定义

这个是决定是否有odm分区，如果带odm分区，那么路径就在odm分区

```makefile
LOCAL_MODULE := precompiled_sepolicy
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
LOCAL_PROPRIETARY_MODULE := true
 
ifeq ($(BOARD_USES_ODMIMAGE),true)
LOCAL_MODULE_PATH := $(TARGET_OUT_ODM)/etc/selinux
else
LOCAL_MODULE_PATH := $(TARGET_OUT_VENDOR)/etc/selinux
endif
```

首先它依赖的是这些cil文件（CIL : Common Intermediate Language，通用中间语言）

- /system/etc/selinux/plat_sepolicy.cil
- /system/etc/selinux/mapping/30.0.cil
- /vendor/etc/selinux/plat_pub_versioned.cil
- /vendor/etc/selinux/vendor_sepolicy.cil

4）接着是通过secilc命令来生成precompiled_sepolicy

```makefile
built_device=xxxx
built_plat_cil=out/target/product/${built_device}/obj/ETC/plat_sepolicy.cil_intermediates/plat_sepolicy.cil
built_mapping_cil=out/target/product/${built_device}/obj/ETC/30.0.cil_intermediates/30.0.cil
built_plat_pub_vers_cil=out/target/product/${built_device}/obj/ETC/plat_pub_versioned.cil_intermediates/plat_pub_versioned.cil
built_vendor_cil=out/target/product/${built_device}/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_sepolicy.cil
 
secilc -m -M true -G -c 30 ${built_plat_cil} ${built_mapping_cil} ${built_plat_pub_vers_cil} ${built_vendor_cil} -o precompiled_sepolicy -f /dev/null
```

secilc的源码在：android/external/selinux/secilc

secilc是用来编译cil中间文件的，将其生成二进制文件的

代码中有文档，其中有一张图片对于理解secilc非常重要

android/external/selinux/secilc/docs/cil_design.jpeg

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_4.png" thumbnail="/SElinux/selinux_4.png" title="">}}

> The parse tree consists of open parenthesis nodes and nodes for each symbol or quoted strings. An open parenthesis is the parent of the symbols or quoted strings that follow it, until a closed parenthesis is reached.Close parenthesis and comments do not make it into the parse tree. 
>
> 解析树由开放的括号节点和每个符号或带引号的字符串的节点组成。开括号是紧随其后的符号或带引号的字符串的父代，直到达到闭括号为止。闭括号和注释不将其放入语法分析树中。
>
> The cil_build_ast function walks the parse tree and creates the ast. The first node in each list is checked for keywords,and ast nodes are created containing the data structures corresponding to the keyword. Only declarations are handled in this step.Delared symbols are add to symtabs for each node flavor, and in local or global namespace symtabs.Strings for symbols that are references(not declarations) are copied into the ast, and will be resolved later.The parse tree should be destroyed following this step. cil_build_ast
> 
> 函数遍历解析树并创建ast。检查每个列表中的第一个节点是否包含关键字，并创建包含与关键字相对应的数据结构的ast节点。
> 在此步骤中仅处理声明。对于每个节点类型，将符号添加到符号表中，并在本地或全局名称空间中将符号表添加。将引用（非声明）符号的字符串复制到ast中，稍后将对其进行解析。此步骤之后，应该销毁解析树。

5）每一个cil是如何生成的
拿其中一个来看看：plat_sepolicy.cil

```makefile
include $(CLEAR_VARS)

LOCAL_MODULE := plat_sepolicy.cil
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_OUT)/etc/selinux

include $(BUILD_SYSTEM)/base_rules.mk

plat_policy.conf := $(intermediates)/plat_policy.conf
$(plat_policy.conf): PRIVATE_MLS_SENS := $(MLS_SENS)
$(plat_policy.conf): PRIVATE_MLS_CATS := $(MLS_CATS)
$(plat_policy.conf): PRIVATE_TARGET_BUILD_VARIANT := $(TARGET_BUILD_VARIANT)
$(plat_policy.conf): PRIVATE_TGT_ARCH := $(my_target_arch)
$(plat_policy.conf): PRIVATE_TGT_WITH_ASAN := $(with_asan)
$(plat_policy.conf): PRIVATE_ADDITIONAL_M4DEFS := $(LOCAL_ADDITIONAL_M4DEFS)
$(plat_policy.conf): PRIVATE_SEPOLICY_SPLIT := $(PRODUCT_SEPOLICY_SPLIT)
$(plat_policy.conf): PRIVATE_COMPATIBLE_PROPERTY := $(PRODUCT_COMPATIBLE_PROPERTY)
$(plat_policy.conf): $(call build_policy, $(sepolicy_build_files), \
$(PLAT_PUBLIC_POLICY) $(PLAT_PRIVATE_POLICY))
	$(transform-policy-to-conf)
    $(hide) sed '/dontaudit/d' $@ > $@.dontaudit

$(LOCAL_BUILT_MODULE): PRIVATE_ADDITIONAL_CIL_FILES := \
  $(call build_policy, $(sepolicy_build_cil_workaround_files), $(PLAT_PRIVATE_POLICY))
$(LOCAL_BUILT_MODULE): PRIVATE_NEVERALLOW_ARG := $(NEVERALLOW_ARG)
$(LOCAL_BUILT_MODULE): $(plat_policy.conf) $(HOST_OUT_EXECUTABLES)/checkpolicy \
  $(HOST_OUT_EXECUTABLES)/secilc \
  $(call build_policy, $(sepolicy_build_cil_workaround_files), $(PLAT_PRIVATE_POLICY)) \
  $(built_sepolicy_neverallows)
	@mkdir -p $(dir $@)
	$(hide) $(CHECKPOLICY_ASAN_OPTIONS) $(HOST_OUT_EXECUTABLES)/checkpolicy -M -C -c \
        $(POLICYVERS) -o $@ $<
    $(hide) cat $(PRIVATE_ADDITIONAL_CIL_FILES) >> $@
    $(hide) $(HOST_OUT_EXECUTABLES)/secilc -m -M true -G -c $(POLICYVERS) $(PRIVATE_NEVERALLOW_ARG) $@ -o /dev/null -f /dev/null

built_plat_cil := $(LOCAL_BUILT_MODULE)
plat_policy.conf :=
```

sepolicy_build_files 包含了所有的te文件，这个是我们最熟悉的部分

还包括其他必需的文件，如宏定义global_macros 、 te_macros等

```te
sepolicy_build_files := security_classes \
                        initial_sids \
                        access_vectors \
                        global_macros \
                        neverallow_macros \
                        mls_macros \
                        mls_decl \
                        mls \
                        policy_capabilities \
                        te_macros \
                        attributes \
                        ioctl_defines \
                        ioctl_macros \
                        *.te \
                        roles_decl \
                        roles \
                        users \
                        initial_sid_contexts \
                        fs_use \
                        genfs_contexts \
                        port_contexts
```

然后调用transform-policy-to-conf将策略文件转成conf文件，生成plat_policy.conf

```makefile
define transform-policy-to-conf
@mkdir -p $(dir $@)
$(hide) m4 $(PRIVATE_ADDITIONAL_M4DEFS) \
    -D mls_num_sens=$(PRIVATE_MLS_SENS) -D mls_num_cats=$(PRIVATE_MLS_CATS) \
    -D target_build_variant=$(PRIVATE_TARGET_BUILD_VARIANT) \
    -D target_with_dexpreopt=$(WITH_DEXPREOPT) \
    -D target_arch=$(PRIVATE_TGT_ARCH) \
    -D target_with_asan=$(PRIVATE_TGT_WITH_ASAN) \
    -D target_full_treble=$(PRIVATE_SEPOLICY_SPLIT) \
    -D target_compatible_property=$(PRIVATE_COMPATIBLE_PROPERTY) \
    $(PRIVATE_TGT_RECOVERY) \
    -s $^ > $@
endef
.KATI_READONLY := transform-policy-to-conf
```



> 关于mk相关的一些语法
>
> ```makefile
> #目标，:前面的变量
> $@ 
> #所有依赖，:后面的所有变量
> $^
> #第一个依赖，:的第一个变量
> $<
> ```
>
> Linux中的sed使用
>
> ```shell
> sed '/^\s*dontaudit.*;/d' $@ | sed '/^\s*dontaudit/,/;/d' > $@.dontaudit
> # 分成两部分
> #Example1: sed '/^\s*dontaudit.*;/d' $@
> #Example2: sed '/^\s*dontaudit/,/;/d' > $@.dontaudit
> #其中sed '/string/d' ,表示到匹配到string的字符串的行数删除
> # sed '/string1/,/string2/d',表示匹配到string1开头，string2结尾的这一段字符串全部删除
> # \s 在正则中表示一个空白字符（可能是空格、制表符、其他空白）
> # \s* 在正则中表示匹配0个或多个空白字符，会尽可能多的匹配
> # ^\s* 在正则中表示匹配0个或多个非空白字符，会尽可能多的匹配
> # Example1表示在$@文件中去除包含dontaudit字符的行数，这里主要是dontaudit前后的括号且占单行
> # Example2表示在$@文件中去除包含dontaudit字符的行数，这里主要是dontaudit前后的括号且占多行
> ```
>

把上面的变量替换之后就是

```makefile
policy_files := $(call build_policy, $(sepolicy_build_files), \
  $(PLAT_PUBLIC_POLICY) $(PLAT_PRIVATE_POLICY))
plat_policy.conf := out/target/product/xxx/obj/ETC/plat_sepolicy.cil_intermediates/plat_policy.conf
$(plat_policy.conf): PRIVATE_MLS_SENS := 1
$(plat_policy.conf): PRIVATE_MLS_CATS := 1024
$(plat_policy.conf): PRIVATE_TARGET_BUILD_VARIANT := userdebug
$(plat_policy.conf): PRIVATE_TGT_ARCH := arm
$(plat_policy.conf): PRIVATE_TGT_WITH_ASAN := false
$(plat_policy.conf): PRIVATE_TGT_WITH_NATIVE_COVERAGE := false
$(plat_policy.conf): PRIVATE_ADDITIONAL_M4DEFS :=
$(plat_policy.conf): PRIVATE_SEPOLICY_SPLIT := true
$(plat_policy.conf): PRIVATE_COMPATIBLE_PROPERTY := true
$(plat_policy.conf): PRIVATE_TREBLE_SYSPROP_NEVERALLOW := true
$(plat_policy.conf): PRIVATE_POLICY_FILES := $(policy_files)
$(plat_policy.conf): $(policy_files) $(M4)
	$(transform-policy-to-conf)
	$(hide) sed '/^\s*dontaudit.*;/d' plat_policy.conf | sed '/^\s*dontaudit/,/;/d' > plat_policy.conf.dontaudit

$(LOCAL_BUILT_MODULE): PRIVATE_ADDITIONAL_CIL_FILES := \
  system/sepolicy/private/technical_debt.cil
$(LOCAL_BUILT_MODULE): PRIVATE_NEVERALLOW_ARG :=
$(LOCAL_BUILT_MODULE): out/target/product/xxx/obj/ETC/plat_sepolicy.cil_intermediates/plat_policy.conf out/target/product/xxx/obj/EXECUTABLES/secilc_intermediates/checkpolicy \
 out/target/product/xxx/obj/EXECUTABLES/secilc_intermediates/secilc \
  system/sepolicy/private/technical_debt.cil \
  out/target/product/xxx/obj/FAKE/sepolicy_neverallows_intermediates/sepolicy_neverallows
	@mkdir -p $(dir plat_sepolicy.cil)
	$(hide) detect_leaks=0 out/target/product/xxx/obj/EXECUTABLES/checkpolicy -M -C -c \
		30 -o plat_sepolicy.cil.tmp out/target/product/xxx/obj/ETC/plat_sepolicy.cil_intermediates/plat_policy.conf
	$(hide) cat system/sepolicy/private/technical_debt.cil >> plat_sepolicy.cil.tmp
	$(hide) out/target/product/xxx/obj/EXECUTABLES/secilc_intermediates/secilc -m -M true -G -c 30 plat_sepolicy.cil.tmp -o /dev/null -f /dev/null
	$(hide) mv plat_sepolicy.cil.tmp plat_sepolicy.cil

built_plat_cil := out/target/product/xxx/obj/ETC/plat_sepolicy.cil_intermediates/plat_sepolicy.cil
plat_policy.conf :=
```

其中`$(transform-policy-to-conf)`又是一个宏，展开显示

```makefile
@mkdir -p $(dir $@)
$(hide) $(M4) --fatal-warnings  \
	-D mls_num_sens=1 -D mls_num_cats=1024 \
	-D target_build_variant=userdebug \
	-D target_with_dexpreopt=true \
	-D target_arch=arm \
	-D target_with_asan=false \
	-D target_with_native_coverage=false \
	-D target_full_treble=true \
	-D target_compatible_property=true \
	-D target_treble_sysprop_neverallow=true \
	-D target_exclude_build_test= \
	-D target_requires_insecure_execmem_for_swiftshader= \
	-s $(policy_files) > plat_policy.conf
```

其他的cil也是类似的过程，会有一些差异，但是跟着mk来分析就行了

#### 2.2.1.2案例-策略文件编译生成cil

##### 2.2.1.2.1原始的testA.te

```te
type testA, domain, binder_in_vendor_violators, vendor_executes_system_violators;
type testA_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(testA)

allow testA picasso_device:chr_file rw_file_perms;
allow testA testA_exec:file { execute_no_trans };
allow testA vendor_toolbox_exec:file { execute_no_trans };
allow testA vendor_data_file:dir create_dir_perms;
allow testA vendor_data_file:file create_file_perms;
allow testA vendor_file:file { execute_no_trans };
allow testA mnt_media_rw_file:dir { search };
allow testA vfat:file { getattr read open };
allow testA vfat:dir { search };
allow testA block_device:dir { search };
allow testA activity_service:service_manager { find };
allow testA system_server:binder { call transfer };
binder_use(testA)
allow testA shell_exec:file { read execute };
allow testA system_file:file { execute_no_trans };
```

##### 2.2.1.2.2conf文件

经过m4宏处理器处理后，所有的宏都被替换展开了

```te
#line 1 "device/xxxx/common/sepolicy/testA.te"
type testA, domain, binder_in_vendor_violators, vendor_executes_system_violators;
type testA_exec, exec_type, vendor_file_type, file_type;
 

#line 4
 
#line 4
# Allow the necessary permissions.
#line 4
 
#line 4
# Old domain may exec the file and transition to the new domain.
#line 4
allow init testA_exec:file { getattr open read execute map };
#line 4
allow init testA:process transition;
#line 4
# New domain is entered by executing the file.
#line 4
allow testA testA_exec:file { entrypoint open read execute getattr map };
#line 4
# New domain can send SIGCHLD to its caller.
#line 4
 
#line 4
# Enable AT_SECURE, i.e. libc secure mode.
#line 4
dontaudit init testA:process noatsecure;
#line 4
# XXX dontaudit candidate but requires further study.
#line 4
allow init testA:process { siginh rlimitinh };
#line 4
 
#line 4
# Make the transition occur by default.
#line 4
type_transition init testA_exec:process testA;
#line 4
 
#line 4
 
#line 4
type testA_tmpfs, file_type;
#line 4
type_transition testA tmpfs:file testA_tmpfs;
#line 4
allow testA testA_tmpfs:file { read write getattr map };
#line 4
allow testA tmpfs:dir { getattr search };
#line 4
 
#line 4
 
 
allow testA picasso_device:chr_file { { getattr open read ioctl lock map } { open append write lock map } };
allow testA testA_exec:file { execute_no_trans };
allow testA vendor_toolbox_exec:file { execute_no_trans };
allow testA vendor_data_file:dir { create reparent rename rmdir setattr { { open getattr read search ioctl lock } { open search write add_name remove_name lock } } };
allow testA vendor_data_file:file { create rename setattr unlink { { getattr open read ioctl lock map } { open append write lock map } } };
allow testA vendor_file:file { execute_no_trans };
allow testA mnt_media_rw_file:dir { search };
allow testA vfat:file { getattr read open };
allow testA vfat:dir { search };
allow testA block_device:dir { search };
 
 
allow testA activity_service:service_manager { find };
allow testA system_server:binder { call transfer };
 
#line 22
# Call the servicemanager and transfer references to it.
#line 22
allow testA servicemanager:binder { call transfer };
#line 22
# servicemanager performs getpidcon on clients.
#line 22
allow servicemanager testA:dir search;
#line 22
allow servicemanager testA:file { read open };
#line 22
allow servicemanager testA:process getattr;
#line 22
# rw access to /dev/binder and /dev/ashmem is presently granted to
#line 22
# all domains in domain.te.
#line 22
 
allow testA shell_exec:file { read execute };
allow testA system_file:file { execute_no_trans };
```

##### 2.2.1.2.3 cil文件

接着使用checkpolicy命令，将conf文件转成cil文件：

```makefile
ASAN_OPTIONS=detect_leaks=0 checkpolicy -M -C -c 30 -o plat_sepolicy.cil plat_policy.conf
```

这个过程中，checkpolicy还会检查是否策略文件是否存在问题，比如是否违反neverallow，语法错误，方案自定义策略文件访问平台私有策略等等；check_assertions检查是否有违反neverallow的规则。

checkpolicy源码在：android/external/selinux/checkpolicy

违反neverallow：

```bash
[  6% 3/47] build out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
FAILED: out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
/bin/bash -c "(rm -f out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows ) && (ASAN_OPTIONS=detect_leaks=0 out/host/linux-x86/bin/checkpolicy -M -c           30 -o out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf )"
libsepol.report_failure: neverallow on line 671 of system/sepolicy/public/domain.te (or line 10586 of policy.conf) violated by allow testA servicemanager:binder { call transfer };
libsepol.report_failure: neverallow on line 638 of system/sepolicy/public/domain.te (or line 10522 of policy.conf) violated by allow testA activity_service:service_manager { find };
libsepol.check_assertions: 2 neverallow failures occurred
Error while expanding policy
out/host/linux-x86/bin/checkpolicy:  loading policy configuration from out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf
ninja: build stopped: subcommand failed.
01:32:50 ninja failed with: exit status 1
```

语法错误：

```bash
[  6% 3/47] build out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
FAILED: out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
/bin/bash -c "(rm -f out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows ) && (ASAN_OPTIONS=detect_leaks=0 out/host/linux-x86/bin/checkpolicy -M -c           30 -o out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf )"
device/xxxxxx/common/sepolicy/testA.te:11:ERROR 'syntax error' at token 'allow' on line 47452:
allow testA system_server { call transfer };
allow testA activity_service:service_manager { find }
checkpolicy:  error(s) encountered while parsing configuration
out/host/linux-x86/bin/checkpolicy:  loading policy configuration from out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf
ninja: build stopped: subcommand failed.
10:17:55 ninja failed with: exit status 1
```

方案自定义策略文件访问平台私有策略：

```bash
FAILED: out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_sepolicy.cil
/bin/bash -c "out/host/linux-x86/bin/build_sepolicy -a out/host/linux-x86/bin build_cil                 -i out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_policy.conf -m out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/reqd_policy_mask.cil -c ASAN_OPTIONS=detect_leaks=0          -b out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/plat_pub_policy.cil -d out/target/product/xxxx/obj/ETC/plat_sepolicy.cil_intermediates/plat_sepolicy.cil out/target/product/xxxx/obj/ETC/plat_pub_versioned.cil_intermediates/plat_pub_versioned.cil out/target/product/xxxx/obj/ETC/30.0.cil_intermediates/30.0.cil -f out/target/product/xxxx/obj/ETC/plat_pub_versioned.cil_intermediates/plat_pub_versioned.cil                 -t 30.0 -p 30 -o out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_sepolicy.cil"
device/xxxxxx/common/sepolicy/testA.te:18:ERROR 'unknown type storaged' at token ';' on line 34318:
allow hal_memtrack_default storaged:dir { search };
#allow hal_memtrack_default rootdaemon:dir { search };
checkpolicy:  error(s) encountered while parsing configuration
out/host/linux-x86/bin/checkpolicy:  loading policy configuration from out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_policy.conf
build_sepolicy - failed to run command: 'ASAN_OPTIONS=detect_leaks=0 out/host/linux-x86/bin/checkpolicy -C -M -c 30 -o out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_policy_raw.cil out/target/product/xxxx/obj/ETC/vendor_sepolicy.cil_intermediates/vendor_policy.conf' (ret:1)
ninja: build stopped: subcommand failed.
10:24:07 ninja failed with: exit status 1
```

**转出来cil之后：**

```
(type testA)
(roletype object_r testA)
(type testA_exec)
(roletype object_r testA_exec)
(type testA_tmpfs)
(roletype object_r testA_tmpfs)
 
(typeattributeset file_type (...... testA_exec testA_tmpfs ......))
(typeattributeset exec_type (...... testA_exec ......))
(typeattributeset vendor_file_type (...... testA_exec ......))
(typeattributeset binder_in_vendor_violators (testA apkinstalldata xbug))
(typeattributeset vendor_executes_system_violators (testA apkinstalldata))
(typeattributeset domain (...... testA ......))
 
(allow init_30_0 testA_exec (file (read getattr map execute open)))
(allow init_30_0 testA (process (transition)))
(allow testA testA_exec (file (read getattr map execute entrypoint open)))
(dontaudit init_30_0 testA (process (noatsecure)))
(allow init_30_0 testA (process (siginh rlimitinh)))
(typetransition init_30_0 testA_exec process testA)
(typetransition testA tmpfs_30_0 file testA_tmpfs)
(allow testA testA_tmpfs (file (read write getattr map)))
(allow testA tmpfs_30_0 (dir (getattr search)))
(allow testA picasso_device_30_0 (chr_file (ioctl read write getattr lock append map open)))
(allow testA testA_exec (file (execute_no_trans)))
(allow testA vendor_toolbox_exec_30_0 (file (execute_no_trans)))
(allow testA vendor_data_file_30_0 (dir (ioctl read write create getattr setattr lock rename add_name remove_name reparent search rmdir open)))
(allow testA vendor_data_file_30_0 (file (ioctl read write create getattr setattr lock append map unlink rename open)))
(allow testA vendor_file_30_0 (file (execute_no_trans)))
(allow testA mnt_media_rw_file_30_0 (dir (search)))
(allow testA vfat_30_0 (file (read getattr open)))
(allow testA vfat_30_0 (dir (search)))
(allow testA block_device_30_0 (dir (search)))
(allow testA property_socket_30_0 (sock_file (write)))
(allow testA init_30_0 (unix_stream_socket (connectto)))
(allow testA activity_service_30_0 (service_manager (find)))
(allow testA system_server_30_0 (binder (call transfer)))
(allow testA servicemanager_30_0 (binder (call transfer)))
(allow servicemanager_30_0 testA (dir (search)))
(allow servicemanager_30_0 testA (file (read open)))
(allow servicemanager_30_0 testA (process (getattr)))
(allow testA shell_exec_30_0 (file (read execute)))
(allow testA system_file_30_0 (file (execute_no_trans)))
```

##### 2.2.1.2.4检查
在最后会用secilc编译一次cil文件，检查是否有问题

经历过上面的一个过程，编译出所有需要的cil，最后就会调用secilc命令编译所有的cil文件，生成二进制策略文件precompiled_sepolicy

简单的说，这个过程就是：

te -> conf -> cil -> precompiled_sepolicy

cil这个在很早的Android版本，比如5.1的时候是还没有的，是后面的版本才添加的

### 2.2.2将策略文件加载到kernel

接着回到加载selinux 策略的初始化过程

#### 2.2.2.1selinux_android_load_policy_from_fd

android/external/selinux/libselinux/src/android/android_platform.c

```c++
int selinux_android_load_policy_from_fd(int fd, const char *description)
{
    int rc;
    struct stat sb;
    void *map = NULL;
    static int load_successful = 0;
 
    /*
        * Since updating policy at runtime has been abolished
        * we just check whether a policy has been loaded before
        * and return if this is the case.
        * There is no point in reloading policy.
        */
    if (load_successful){
        selinux_log(SELINUX_WARNING, "SELinux: Attempted reload of SELinux policy!/n");
        return 0;
    }
 
    set_selinuxmnt(SELINUXMNT);  //设置SELinux mount的位置  /sys/fs/selinux
    if (fstat(fd, &sb) < 0) {
        selinux_log(SELINUX_ERROR, "SELinux:  Could not stat %s:  %s\n",
                description, strerror(errno));
        return -1;
    }
    map = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (map == MAP_FAILED) {
        selinux_log(SELINUX_ERROR, "SELinux:  Could not map %s:  %s\n",
                description, strerror(errno));
        return -1;
    }
 
    rc = security_load_policy(map, sb.st_size); //加载sepolicy
    if (rc < 0) {
        selinux_log(SELINUX_ERROR, "SELinux:  Could not load policy:  %s\n",
                strerror(errno));
        munmap(map, sb.st_size);
        return -1;
    }
 
    munmap(map, sb.st_size);
    selinux_log(SELINUX_INFO, "SELinux: Loaded policy from %s\n", description);
    load_successful = 1;
    return 0;
}
```

#### 2.2.2.2security_load_policy（libselinux）

android/external/selinux/libselinux/src/load_policy.c

```c++
int security_load_policy(void *data, size_t len)
{
    char path[PATH_MAX];
    int fd, ret;
 
    if (!selinux_mnt) {
        errno = ENOENT;
        return -1;
    }
 
    snprintf(path, sizeof path, "%s/load", selinux_mnt);
    fd = open(path, O_RDWR | O_CLOEXEC);
    if (fd < 0)
        return -1;
 
    ret = write(fd, data, len);
    close(fd);
    if (ret < 0)
        return -1;
    return 0;
}
```

这里是把文件内容读取出来后，直接写入节点：/sys/fs/selinux/load

#### 2.2.2.3/sys/fs/selinux/load

接着往下就要看kernel的代码了

android/common/security/selinux/selinuxfs.c

```c++
static struct tree_descr selinux_files[] = {
        [SEL_LOAD] = {"load", &sel_load_ops, S_IRUSR|S_IWUSR},
        [SEL_ENFORCE] = {"enforce", &sel_enforce_ops, S_IRUGO|S_IWUSR},
        [SEL_CONTEXT] = {"context", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_ACCESS] = {"access", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_CREATE] = {"create", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_RELABEL] = {"relabel", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_USER] = {"user", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_POLICYVERS] = {"policyvers", &sel_policyvers_ops, S_IRUGO},
        [SEL_COMMIT_BOOLS] = {"commit_pending_bools", &sel_commit_bools_ops, S_IWUSR},
        [SEL_MLS] = {"mls", &sel_mls_ops, S_IRUGO},
        [SEL_DISABLE] = {"disable", &sel_disable_ops, S_IWUSR},
        [SEL_MEMBER] = {"member", &transaction_ops, S_IRUGO|S_IWUGO},
        [SEL_CHECKREQPROT] = {"checkreqprot", &sel_checkreqprot_ops, S_IRUGO|S_IWUSR},
        [SEL_REJECT_UNKNOWN] = {"reject_unknown", &sel_handle_unknown_ops, S_IRUGO},
        [SEL_DENY_UNKNOWN] = {"deny_unknown", &sel_handle_unknown_ops, S_IRUGO},
        [SEL_STATUS] = {"status", &sel_handle_status_ops, S_IRUGO},
        [SEL_POLICY] = {"policy", &sel_policy_ops, S_IRUGO},
        [SEL_VALIDATE_TRANS] = {"validatetrans", &sel_transition_ops,
                    S_IWUGO},
        /* last one */ {""}
    };
 
static const struct file_operations sel_load_ops = {
    .write  = sel_write_load,
    .llseek = generic_file_llseek,
};
```

接着是 sel_write_load

```c++
static ssize_t sel_write_load(struct file *file, const char __user *buf,
                size_t count, loff_t *ppos)
 
{
    ......
 
    length = security_load_policy(data, count);
    if (length)
        goto out;
 
    ......
}
```

#### 2.2.2.4security_load_policy（kernel）

android/common/security/selinux/ss/services.c

```c++
if (!ss_initialized) {
        avtab_cache_init();
        rc = policydb_read(&policydb, fp);
        if (rc) {
            avtab_cache_destroy();
            goto out;
        }
 
        policydb.len = len;
        rc = selinux_set_mapping(&policydb, secclass_map,
                     ¤t_mapping,
                     ¤t_mapping_size);
        if (rc) {
            policydb_destroy(&policydb);
            avtab_cache_destroy();
            goto out;
        }
 
        rc = policydb_load_isids(&policydb, &sidtab);
        if (rc) {
            policydb_destroy(&policydb);
            avtab_cache_destroy();
            goto out;
        }
 
        security_load_policycaps();
        ss_initialized = 1;
        seqno = ++latest_granting;
        selinux_complete_init();
        avc_ss_reset(seqno);
        selnl_notify_policyload(seqno);
        selinux_status_update_policyload(seqno);
        selinux_netlbl_cache_invalidate();
        selinux_xfrm_notify_policyload();
        goto out;
    }
```

通过policydb_read把数据读取保存到policydb这个数据结构里面

policydb是一个结构体

定义在：android/common/security/selinux/ss/policydb.h

```c++
struct policydb {
    int mls_enabled;
 
    /* symbol tables */
    struct symtab symtab[SYM_NUM];
#define p_commons symtab[SYM_COMMONS]
#define p_classes symtab[SYM_CLASSES]
#define p_roles symtab[SYM_ROLES]
#define p_types symtab[SYM_TYPES]
#define p_users symtab[SYM_USERS]
#define p_bools symtab[SYM_BOOLS]
#define p_levels symtab[SYM_LEVELS]
#define p_cats symtab[SYM_CATS]
 
    /* symbol names indexed by (value - 1) */
    struct flex_array *sym_val_to_name[SYM_NUM];
 
    /* class, role, and user attributes indexed by (value - 1) */
    struct class_datum **class_val_to_struct;
    struct role_datum **role_val_to_struct;
    struct user_datum **user_val_to_struct;
    struct flex_array *type_val_to_struct_array;
 
    /* type enforcement access vectors and transitions */
    struct avtab te_avtab;
 
    /* role transitions */
    struct role_trans *role_tr;
 
    /* file transitions with the last path component */
    /* quickly exclude lookups when parent ttype has no rules */
    struct ebitmap filename_trans_ttypes;
    /* actual set of filename_trans rules */
    struct hashtab *filename_trans;
 
    /* bools indexed by (value - 1) */
    struct cond_bool_datum **bool_val_to_struct;
    /* type enforcement conditional access vectors and transitions */
    struct avtab te_cond_avtab;
    /* linked list indexing te_cond_avtab by conditional */
    struct cond_node *cond_list;
 
    /* role allows */
    struct role_allow *role_allow;
 
    /* security contexts of initial SIDs, unlabeled file systems,
       TCP or UDP port numbers, network interfaces and nodes */
    struct ocontext *ocontexts[OCON_NUM];
 
    /* security contexts for files in filesystems that cannot support
       a persistent label mapping or use another
       fixed labeling behavior. */
    struct genfs *genfs;
 
    /* range transitions table (range_trans_key -> mls_range) */
    struct hashtab *range_tr;
 
    /* type -> attribute reverse mapping */
    struct flex_array *type_attr_map_array;
 
    struct ebitmap policycaps;
 
    struct ebitmap permissive_map;
 
    /* length of this policy when it was loaded */
    size_t len;
 
    unsigned int policyvers;
 
    unsigned int reject_unknown : 1;
    unsigned int allow_unknown : 1;
 
    u16 process_class;
    u32 process_trans_perms;
};
```

将安全策略从用户空间加载到 SELinux LSM 模块中去了，这个数据结构很复杂

te规则最后会保存在

```c++
/* type enforcement access vectors and transitions */
struct avtab te_avtab;
```

> 另外还有服务、属性等初始化，有兴趣可以结合大佬的博客一起看看：
>
> - [SEAndroid安全机制中的文件安全上下文关联分析](https://blog.csdn.net/Luoshengyang/article/details/37749383)
> - [SEAndroid安全机制中的进程安全上下文关联分析](https://blog.csdn.net/luoshengyang/article/details/38054645)
> - [SEAndroid安全机制对Android属性访问的保护分析](https://blog.csdn.net/Luoshengyang/article/details/38102011)
> - [SEAndroid安全机制对Binder IPC的保护分析](https://blog.csdn.net/Luoshengyang/article/details/38326729)

还记得最开头有提到，在android/device/xxxxxx/common/sepolicy和android/system/sepolicy都要添加 xxx.te，而有些方案却不用？

分析mk之后，不难发现，是因为去掉了这个：

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_5.png" thumbnail="/SElinux/selinux_5.png" title="">}}

然后检查对比的动作没有做了

## 2.3总结

到这里，总结一下：

1、从Android8.0之后，加载的是 /(vendor|odm)/etc/selinux/precompiled_sepolicy ，以前的版本用的是 /sepolicy，如下图

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_6.png" thumbnail="/SElinux/selinux_6.png" title="">}}

2、有些平台上有odm分区，因此用的是 /odm/etc/selinux/precompiled_sepolicy，这也就是为什么前面提到的编译替换/vendor/etc/selinux/和/system/etc/selinux/没有效果的原因

# 3权限检查原理与调试

我们在处理SELinux权限问题的时候，avc denied信息是最关键的，那么avc denied 的打印信息要怎么看、里面的内容每一个字段是什么意思？kernel log的avc denied信息是哪里打印出来的？

以前有个想法，我们系统的操作、命令都是有限的，那其实脚本写完之后需要什么权限都是确定的，那可不可能更快的加好te规则呢？

假设有如下脚本以及te规则

- /vendor/bin/testA.sh

```shell
#!/vendor/bin/sh

echo "hello world" > /data/vendor/test.txt
```

- /android/device/xxxx/common/sepolicy/file_contexts

```te
/vendor/bin/testA.sh     u:object_r:testA_exec:s0
```

- /android/device/xxxx/common/sepolicy/testA.te

```te
type testA, domain;
type testA_exec, exec_type, vendor_file_type, file_type;
 
init_daemon_domain(testA)
 
allow testA picasso_device:chr_file { write open getattr ioctl };
allow testA vendor_toolbox_exec:file { execute_no_trans };
allow testA vendor_data_file:dir { add_name };
```

执行脚本后，会有如下报错信息：

```shell
[ 5424.996583@0]- type=1400 audit(1577887433.668:59): avc: denied { write } for pid=4021 comm="testA.sh"
name="vendor" dev="mmcblk0p20" ino=7395 scontext=u:r:testA:s0 tcontext=u:object_r:vendor_data_file:s0 tclass=dir permissive=0
```

## 3.1权限检测原理

这部分的代码集中在两个部分

- kernel的代码，里面的security/selinux
- /android/external/selinux

一开始，/data/vendor/test.txt是不存在的；testA.sh脚本运行起来之后，它的安全上下文是u:r:testA:s0，type为testA的进程，要拥有对type为vendor_data_file的目录(dir)的写(write)权限，才能创建一个不存在的文件

权限检测其实就是针对进程访问对象的两个权限：拥有的权限和需要的权限，将两者进行计算得出结果，即允许还是拒绝

### 3.1.1拥有的权限

te规则定义的权限保存在哪里？

testA.te里面它拥有add_name权限，前面的文章有提到te策略文件的编译过程，这个规则最终会生成在二进制策略文件precompiled_sepolicy里面，而policydb这个数据结构是用来组织起所有数据的

其中te规则保存在这个成员里面

```c++
/* type enforcement access vectors and transitions */
avtab_t te_avtab;
```

这里面的数据结构比较复杂，里面主要需要理解几个数据结构：

- 位图ebitmap
- 链表
- 哈希表hashtab
- 符号表symtab

这里面的内容太多了，我只是简单分析了其中te规则一部分代码，有兴趣可以结合参考文章《Linux多安全策略和动态安全策 略框架模块代码分析报告》（见参考文章）来阅读源码深入了解

下面分析的一些结论，如有问题请指出：

testA这个type的进程，对type为vendor_data_file的dir(目录)所拥有的权限是用一个整数u32来表示的：

它的值是：

```c++
0x00120010
```

下面简单看看这个值是怎么生成的？

```te
(allow testA_30_0 vendor_data_file_30_0 (dir (add_name)))
```

接着看看 **/android/external/selinux/libsepol/cil/src/cil_binary.c** 里面的__cil_perms_to_datum函数

```c++
int __cil_perms_to_datum(struct cil_list *perms, class_datum_t *sepol_class, uint32_t *datum)
{
    int rc = SEPOL_ERR;
    char *key = NULL;
    struct cil_list_item *curr_perm;
    struct cil_perm *cil_perm;
    uint32_t data = 0;
 
    cil_list_for_each(curr_perm, perms) {
        cil_perm = curr_perm->data;
        key = cil_perm->datum.fqn;
 
        rc = __perm_str_to_datum(key, sepol_class, &data);
        if (rc != SEPOL_OK) {
            goto exit;
        }
    }
 
    *datum = data;
 
    return SEPOL_OK;
 
exit:
    return rc;
}
 
 
int __perm_str_to_datum(char *perm_str, class_datum_t *sepol_class, uint32_t *datum)
{
    int rc;
    perm_datum_t *sepol_perm;
    common_datum_t *sepol_common;
 
    sepol_perm = hashtab_search(sepol_class->permissions.table, perm_str);
    if (sepol_perm == NULL) {
        sepol_common = sepol_class->comdatum;
        sepol_perm = hashtab_search(sepol_common->permissions.table, perm_str);
        if (sepol_perm == NULL) {
            cil_log(CIL_ERR, "Failed to find datum for perm %s\n", perm_str);
            rc = SEPOL_ERR;
            goto exit;
        }
    }
    *datum |= 1 << (sepol_perm->s.value - 1);
 
    return SEPOL_OK;
 
exit:
    return rc;
}
```

datum 里面保存的值就是0x00120010

看最关键的一句话：

```c++
*datum |= 1 << (sepol_perm->s.value - 1);
```

上面的操作其实就是将对应的位置1，代表它拥有的权限，那每一个位对应的权限是什么呢？
这个要看：

**/android/system/sepolicy/private/access_vectors**

```te
common file
{
    ioctl
    read
    write
    create
    getattr
    setattr
    lock
    relabelfrom
    relabelto
    append
    map
    unlink
    link
    rename
    execute
    quotaon
    mounton
}
 
class dir
inherits file
{
    add_name
    remove_name
    reparent
    search
    rmdir
    open
    audit_access
    execmod
}
```

sepol_perm->s.value可以理解成是解析过程中某个权限在access_vectors定义的class里面的index值，它会用符号表和对应的字符串关联起来，比如add_name在集合里面的是第18个，对应的值是0x00020000

我们这里的是dir，将其合并下：

```c++
{
    ioctl,read,write,create,getattr,setattr,lock,relabelfrom,relabelto,append,map,unlink,
    link,rename,execute,quotaon,mounton,add_name,remove_name,reparent,search,rmdir,open,
    audit_access,execmod
}
```

上面那句话的操作就是将datum的第18位置成1

那么最后计算出来的0x00120010就是：

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_7.png" thumbnail="/SElinux/selinux_7.png" title="">}}

翻译过来，是这个规则

```te
allow testA vendor_data_file:dir { search add_name getattr };
```

奇怪的是，这里怎么会多出来两个权限呢，我都没有加上去

原因是来自这条规则：

这个在domain.te里面

```te
allow domain vendor_data_file:dir { getattr search };
```

### 3.1.2需要的权限

要分析操作vendor_data_file:dir所需要的权限，借助打印堆栈信息来分析：

```bash
[ 5424.899991@1]d CPU: 1 PID: 4119 Comm: testA.sh Tainted: P           O    4.9.113 #14
[ 5424.907561@1]d Hardware name: Generic DT based system
[ 5424.912595@1]d [bc3c7a84+  16][<c020dc10>] show_stack+0x20/0x24
[ 5424.918447@1]d [bc3c7aa4+  32][<c05443ec>] dump_stack+0x90/0xac
[ 5424.924311@1]d [bc3c7aec+  72][<c04d5954>] slow_avc_audit+0x88/0xa8
[ 5424.930515@1]d [bc3c7b34+  72][<c04da498>] audit_inode_permission+0x80/0x88
[ 5424.937417@1]d [bc3c7bac+ 120][<c04daf20>] selinux_inode_permission+0x2d0/0x314
[ 5424.944664@1]d [bc3c7bcc+  32][<c04d1f44>] security_inode_permission+0x4c/0x68
[ 5424.951824@1]d [bc3c7bec+  32][<c03a5bc8>] __inode_permission2+0x50/0xf0
[ 5424.958455@1]d [bc3c7bfc+  16][<c03a5cb0>] inode_permission2+0x20/0x54
[ 5424.964930@1]d [bc3c7c84+ 136][<c03a9048>] path_openat+0xb30/0x118c
[ 5424.971137@1]d [bc3c7d2c+ 168][<c03aabe4>] do_filp_open+0x7c/0xe0
[ 5424.977179@1]d [bc3c7d7c+  80][<c0397b34>] do_sys_open+0x124/0x238
[ 5424.983303@1]d [bc3c7d8c+  16][<c0397c90>] SyS_openat+0x1c/0x20
[ 5424.989165@1]d [00000000+   0][<c02085c0>] ret_fast_syscall+0x0/0x48
 
用addr2line看看对应的代码调用逻辑
1  show_stack c020dc10
show_stack
kernelcode/arch/arm/kernel/traps.c:263
 
2  dump_stack c05443ec
dump_stack
kernelcode/lib/dump_stack.c:53
 
3  slow_avc_audit c04d5954
slow_avc_audit
kernelcode/security/selinux/avc.c:775
 
4  audit_inode_permission c04da498
audit_inode_permission
kernelcode/security/selinux/hooks.c:3005
 
5  selinux_inode_permission c04daf20
selinux_inode_permission
kernelcode/security/selinux/hooks.c:3058
 
6  security_inode_permission c04d1f44
security_inode_permission
kernelcode/security/security.c:611 (discriminator 5)
 
7  __inode_permission2 c03a5bc8
__inode_permission2
kernelcode/fs/namei.c:436
 
8  inode_permission2 c03a5cb0
inode_permission2
kernelcode/fs/namei.c:486
 
9  path_openat c03a9048
may_o_create
kernelcode/fs/namei.c:3018
 
10  do_filp_open c03aabe4
do_filp_open
kernelcode/fs/namei.c:3569
 
11  do_sys_open c0397b34
do_sys_open
kernelcode/fs/open.c:1073
 
12  SyS_openat c0397c90
SyS_openat
kernelcode/fs/open.c:1093
 
13  ret_fast_syscall c02085c0
ret_fast_syscall
kernelcode/arch/arm/kernel/entry-common.S:37
```

堆栈打印添加在kernel代码里面：

kernelcode$ git diff security/selinux/avc.c

```bash
diff --git a/security/selinux/avc.c b/security/selinux/avc.c
index e60c79d..79cea20 100644
--- a/security/selinux/avc.c
+++ b/security/selinux/avc.c
@@ -771,7 +771,7 @@ noinline int slow_avc_audit(u32 ssid, u32 tsid, u16 tclass,
        sad.result = result;
  
        a->selinux_audit_data = &sad;
-
+       dump_stack();
        common_lsm_audit(a, avc_audit_pre_callback, avc_audit_post_callback);
        return 0;
 }
```

跟着堆栈简单看一遍代码，不难发现，需要的权限是从这里来的

may_o_create

kernelcode/fs/namei.c:3018

```c++
static int may_o_create(const struct path *dir, struct dentry *dentry, umode_t mode)
{
    struct user_namespace *s_user_ns;
    int error = security_path_mknod(dir, dentry, mode, 0);
    if (error)
        return error;
 
    s_user_ns = dir->dentry->d_sb->s_user_ns;
    if (!kuid_has_mapping(s_user_ns, current_fsuid()) ||
        !kgid_has_mapping(s_user_ns, current_fsgid()))
        return -EOVERFLOW;
 
    error = inode_permission2(dir->mnt, dir->dentry->d_inode, MAY_WRITE | MAY_EXEC);
    if (error)
        return error;
 
    return security_inode_create(dir->dentry->d_inode, dentry, mode);
}
```

inode_permission2的第三个参数 MAY_WRITE | MAY_EXEC

会一直传递到 kernelcode/security/selinux/hooks.c

selinux_inode_permission这个函数的第二参数mask

```c++
static int selinux_inode_permission(struct inode *inode, int mask)
{
    const struct cred *cred = current_cred();
    u32 perms;
    bool from_access;
    unsigned flags = mask & MAY_NOT_BLOCK;
    struct inode_security_struct *isec;
    u32 sid;
    struct av_decision avd;
    int rc, rc2;
    u32 audited, denied;
 
    from_access = mask & MAY_ACCESS;
  //mask = MAY_WRITE | MAY_EXEC;
    mask &= (MAY_READ|MAY_WRITE|MAY_EXEC|MAY_APPEND);
 
    /* No permission to check.  Existence test. */
    if (!mask)
        return 0;
 
    validate_creds(cred);
 
    if (unlikely(IS_PRIVATE(inode)))
        return 0;
 
  //mask = MAY_WRITE | MAY_EXEC;
  //perms = DIR__WRITE | DIR__SEARCH
  //perms = 0x00400004
    perms = file_mask_to_av(inode->i_mode, mask);
 
    sid = cred_sid(cred);
    isec = inode_security_rcu(inode, flags & MAY_NOT_BLOCK);
    if (IS_ERR(isec))
        return PTR_ERR(isec);
 
  //主要是获取av_decision信息，里面包含了权限信息，先从avc cache里面获取，如果没有就从policydb里面计算得出
    rc = avc_has_perm_noaudit(sid, isec->sid, isec->sclass, perms, 0, &avd);
  //这个就是审计、裁决的动作了，计算拥有的权限avd和需要的权限perms，得出缺失的权限
    audited = avc_audit_required(perms, &avd, rc,
                     from_access ? FILE__AUDIT_ACCESS : 0,
                     &denied);
  //大部分情况下，如果没有权限问题，到这儿就结束了
    if (likely(!audited))
        return rc;
  //再往下就要开始收集信息，打印avc denied信息了
    rc2 = audit_inode_permission(inode, perms, audited, denied, rc, flags);
    if (rc2)
        return rc2;
    return rc;
}
```

这里面file_mask_to_av计算出来的perms就是我们现在操作dir所需要的权限

DIR__WRITE 、 DIR__SEARCH的定义是av_permissions.h头文件里面

不过不是这个 /android/external/selinux/libselinux/include/selinux/av_permissions.h

而是编译生成，路径在：/android/out/target/product/xxxxx/obj/KERNEL_OBJ/security/selinux/av_permissions.h

```c++
#define DIR__WRITE                                0x00000004UL
#define DIR__SEARCH                               0x00400000UL
```

也就是perms = 0x00400004

实际上，这里面这个权限也是每一位代表了一个权限，和上面的不同的是，这个是根据classmap.h里面定义的值来计算的

**kernelcode/security/selinux/include/classmap.h**

```c++
#define COMMON_FILE_SOCK_PERMS "ioctl", "read", "write", "create", \
    "getattr", "setattr", "lock", "relabelfrom", "relabelto", "append"
 
#define COMMON_FILE_PERMS COMMON_FILE_SOCK_PERMS, "unlink", "link", \
    "rename", "execute", "quotaon", "mounton", "audit_access", \
    "open", "execmod"
 
#define COMMON_SOCK_PERMS COMMON_FILE_SOCK_PERMS, "bind", "connect", \
    "listen", "accept", "getopt", "setopt", "shutdown", "recvfrom",  \
    "sendto", "name_bind"
 
#define COMMON_IPC_PERMS "create", "destroy", "getattr", "setattr", "read", \
        "write", "associate", "unix_read", "unix_write"
 
#define COMMON_CAP_PERMS  "chown", "dac_override", "dac_read_search", \
        "fowner", "fsetid", "kill", "setgid", "setuid", "setpcap", \
        "linux_immutable", "net_bind_service", "net_broadcast", \
        "net_admin", "net_raw", "ipc_lock", "ipc_owner", "sys_module", \
        "sys_rawio", "sys_chroot", "sys_ptrace", "sys_pacct", "sys_admin", \
        "sys_boot", "sys_nice", "sys_resource", "sys_time", \
        "sys_tty_config", "mknod", "lease", "audit_write", \
        "audit_control", "setfcap"
 
#define COMMON_CAP2_PERMS  "mac_override", "mac_admin", "syslog", \
        "wake_alarm", "block_suspend", "audit_read"
 
/*
 * Note: The name for any socket class should be suffixed by "socket",
 *   and doesn't contain more than one substr of "socket".
 */
struct security_class_mapping secclass_map[] = {
    { "security",
      { "compute_av", "compute_create", "compute_member",
        "check_context", "load_policy", "compute_relabel",
        "compute_user", "setenforce", "setbool", "setsecparam",
        "setcheckreqprot", "read_policy", "validate_trans", NULL } },
    { "process",
      { "fork", "transition", "sigchld", "sigkill",
        "sigstop", "signull", "signal", "ptrace", "getsched", "setsched",
        "getsession", "getpgid", "setpgid", "getcap", "setcap", "share",
        "getattr", "setexec", "setfscreate", "noatsecure", "siginh",
        "setrlimit", "rlimitinh", "dyntransition", "setcurrent",
        "execmem", "execstack", "execheap", "setkeycreate",
        "setsockcreate", NULL } },
    { "system",
      { "ipc_info", "syslog_read", "syslog_mod",
        "syslog_picasso", "module_request", "module_load", NULL } },
    { "capability",
      { COMMON_CAP_PERMS, NULL } },
    { "filesystem",
      { "mount", "remount", "unmount", "getattr",
        "relabelfrom", "relabelto", "associate", "quotamod",
        "quotaget", NULL } },
    { "file",
      { COMMON_FILE_PERMS,
        "execute_no_trans", "entrypoint", NULL } },
    { "dir",
      { COMMON_FILE_PERMS, "add_name", "remove_name",
        "reparent", "search", "rmdir", NULL } },
    { "fd", { "use", NULL } },
    { "lnk_file",
      { COMMON_FILE_PERMS, NULL } },
    { "chr_file",
      { COMMON_FILE_PERMS, NULL } },
    { "blk_file",
      { COMMON_FILE_PERMS, NULL } },
    { "sock_file",
      { COMMON_FILE_PERMS, NULL } },
    { "fifo_file",
      { COMMON_FILE_PERMS, NULL } },
    { "socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "tcp_socket",
      { COMMON_SOCK_PERMS,
        "node_bind", "name_connect",
        NULL } },
    { "udp_socket",
      { COMMON_SOCK_PERMS,
        "node_bind", NULL } },
    { "rawip_socket",
      { COMMON_SOCK_PERMS,
        "node_bind", NULL } },
    { "node",
      { "recvfrom", "sendto", NULL } },
    { "netif",
      { "ingress", "egress", NULL } },
    { "netlink_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "packet_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "key_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "unix_stream_socket",
      { COMMON_SOCK_PERMS, "connectto", NULL } },
    { "unix_dgram_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "sem",
      { COMMON_IPC_PERMS, NULL } },
    { "msg", { "send", "receive", NULL } },
    { "msgq",
      { COMMON_IPC_PERMS, "enqueue", NULL } },
    { "shm",
      { COMMON_IPC_PERMS, "lock", NULL } },
    { "ipc",
      { COMMON_IPC_PERMS, NULL } },
    { "netlink_route_socket",
      { COMMON_SOCK_PERMS,
        "nlmsg_read", "nlmsg_write", NULL } },
    { "netlink_tcpdiag_socket",
      { COMMON_SOCK_PERMS,
        "nlmsg_read", "nlmsg_write", NULL } },
    { "netlink_nflog_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_xfrm_socket",
      { COMMON_SOCK_PERMS,
        "nlmsg_read", "nlmsg_write", NULL } },
    { "netlink_selinux_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_iscsi_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_audit_socket",
      { COMMON_SOCK_PERMS,
        "nlmsg_read", "nlmsg_write", "nlmsg_relay", "nlmsg_readpriv",
        "nlmsg_tty_audit", NULL } },
    { "netlink_fib_lookup_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_connector_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_netfilter_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_dnrt_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "association",
      { "sendto", "recvfrom", "setcontext", "polmatch", NULL } },
    { "netlink_kobject_uevent_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_generic_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_scsitransport_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_rdma_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "netlink_crypto_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "appletalk_socket",
      { COMMON_SOCK_PERMS, NULL } },
    { "packet",
      { "send", "recv", "relabelto", "forward_in", "forward_out", NULL } },
    { "key",
      { "view", "read", "write", "search", "link", "setattr", "create",
        NULL } },
    { "dccp_socket",
      { COMMON_SOCK_PERMS,
        "node_bind", "name_connect", NULL } },
    { "memprotect", { "mmap_zero", NULL } },
    { "peer", { "recv", NULL } },
    { "capability2",
      { COMMON_CAP2_PERMS, NULL } },
    { "kernel_service", { "use_as_override", "create_files_as", NULL } },
    { "tun_socket",
      { COMMON_SOCK_PERMS, "attach_queue", NULL } },
    { "binder", { "impersonate", "call", "set_context_mgr", "transfer",
              NULL } },
    { "cap_userns",
      { COMMON_CAP_PERMS, NULL } },
    { "cap2_userns",
      { COMMON_CAP2_PERMS, NULL } },
    { "bpf",
      {"map_create", "map_read", "map_write", "prog_load", "prog_run"} },
    { NULL }
  };
```

把dir所对应的展开一下：

```c++
{ "dir",
  { "ioctl", "read", "write", "create", "getattr", "setattr",
        "lock", "relabelfrom", "relabelto", "append", "unlink", "link",
    "rename", "execute", "quotaon", "mounton", "audit_access",
    "open", "execmod","add_name", "remove_name",
    "reparent", "search", "rmdir", NULL }
},
```

其实这里对应的就是权限：“write"和"search”

也就是说，这一次权限检查，要检查的权限是对type为vendor_data_file的目录(dir)的"write"和"search"权限

其他的其实也是一样的道理，kernel里面的代码规定了进程在访问某些对象的时候所需要的权限

再比如：

```bash
mkdir /data/testdir
 
[ 2336.839823@1] type=1400 audit(1230741525.976:909): avc: denied { getattr } for pid=18561 comm="mkdir"
path="/data/testdir" dev="mmcblk0p21" ino=1149 scontext=u:r:testA:s0 tcontext=u:object_r:system_data_file:s0 tclass=dir permissive=0
 
[ 2336.754088@3] [bc6e5b7c+  16][<c020e3c0>] show_stack+0x20/0x24
[ 2336.759889@3] [bc6e5b9c+  32][<c05daf0c>] dump_stack+0x90/0xac
[ 2336.765696@3] [bc6e5bf4+  88][<c055e978>] slow_avc_audit+0x108/0x128
[ 2336.772019@3] [bc6e5c74+ 128][<c055f040>] avc_has_perm+0x13c/0x170
[ 2336.778172@3] [bc6e5c8c+  24][<c0560334>] inode_has_perm+0x48/0x5c
[ 2336.784326@3] [bc6e5cb4+  40][<c0562b00>] selinux_inode_getattr+0x6c/0x74
[ 2336.791085@3] [bc6e5cd4+  32][<c055ab40>] security_inode_getattr+0x4c/0x68
[ 2336.797934@3] [bc6e5cec+  24][<c03a1ff4>] vfs_getattr+0x20/0x38
[ 2336.803826@3] [bc6e5d24+  56][<c03a20d4>] vfs_fstatat+0x70/0xb0
[ 2336.809718@3] [bc6e5d8c+ 104][<c03a2804>] SyS_fstatat64+0x24/0x40
[ 2336.815787@3] [00000000+   0][<c0208680>] ret_fast_syscall+0x0/0x48
 
inode_has_perm
kernelcode/security/selinux/hooks.c:1705
 
selinux_inode_getattr
kernelcode/security/selinux/hooks.c:3089
 
security_inode_getattr
kernelcode/security/security.c:631 (discriminator 5)
 
vfs_getattr
kernelcode/fs/stat.c:70
 
vfs_fstatat
kernelcode/fs/stat.c:110
 
SYSC_fstatat64
kernelcode/fs/stat.c:440
```

首先检查的需要的权限是：getattr

```c++
static int selinux_inode_getattr(const struct path *path)
{
    return path_has_perm(current_cred(), path, FILE__GETATTR);
}
 
#define FILE__GETATTR                             0x00000010UL
```



### 3.1.3裁决

前面的分析，我们知道了需要的权限是0x00400004，拥有的权限是0x00120010

那代码是怎么来判断权限是否都ok呢？

先看看权限是怎么从policydb里面读取，然后转换的

```c++
inline int avc_has_perm_noaudit(u32 ssid, u32 tsid,
             u16 tclass, u32 requested,
             unsigned flags,
             struct av_decision *avd)
{
    ......
    node = avc_lookup(ssid, tsid, tclass);
    if (unlikely(!node))
        node = avc_compute_av(ssid, tsid, tclass, avd, &xp_node);
    else
        memcpy(avd, &node->ae.avd, sizeof(*avd));
    ......
}
```

首先从avc cache里面找看看有没有缓存值，这个是为了提高运行效率的，系统里面成千上万条规则，如果每一次判断权限都要从文件里面读取然后计算结果，这将会是一个巨大的工作量

所以采用了内存里面缓存之前计算的结果，如果没有的话那就调用avc_compute_av函数计算一次

接着调用security_compute_av

kernelcode/security/selinux/ss/services.c

```c++
void security_compute_av(u32 ssid,
             u32 tsid,
             u16 orig_tclass,
             struct av_decision *avd,
             struct extended_perms *xperms)
{
    ......
    context_struct_compute_av(scontext, tcontext, tclass, avd, xperms);
    map_decision(orig_tclass, avd, policydb.allow_unknown);
  ......
}
```

函数比较长，这里只看最重要的两句

context_struct_compute_av是从policydb里面读取出来的权限值，在这里就是0x00120010

map_decision会将权限值做一个映射关系，如果认真对比classmap.h和access_vectors里面dir权限的顺序，会发现其实不是完全一样的

```c++
static void map_decision(u16 tclass, struct av_decision *avd,
             int allow_unknown)
{
    if (tclass < current_mapping_size) {
        unsigned i, n = current_mapping[tclass].num_perms;
        u32 result;
 
        for (i = 0, result = 0; i < n; i++) {
            if (avd->allowed & current_mapping[tclass].perms[i])
                result |= 1<<i;
            if (allow_unknown && !current_mapping[tclass].perms[i])
                result |= 1<<i;
        }
        avd->allowed = result;
 
        ......
    }
}
 
//current_mapping 初始化的时候，是从classmap.h的secclass_map读取的
rc = selinux_set_mapping(&policydb, secclass_map,
                     ¤t_mapping,
                     ¤t_mapping_size);
```

这里会将0x00120010转成0x00480010

和上面一样的道理：

```bash
0x00480010
 
二进制表示：
0000 0000 0100 1000 0000 0000 0001 0000
 
这里的每一个位代表着不同的权限，1是有权限，0是没有权限
```

那么0x00480010对应的权限：

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_8.png" thumbnail="/SElinux/selinux_8.png" title="">}}

和前面的权限是一致的，这里没有搞清楚为什么要有一个映射关系，感觉直接搞成一样的顺序不是更方便吗？

那接下来要比较的权限就是0x00480010（拥有的权限）和0x00400004（需要的权限）

找到0x00400004（需要的权限）为1 但是 0x00480010（拥有的权限）不为 1的那一位，就是缺少权限的

计算方法可以看看函数：

```c++
static inline u32 avc_audit_required(u32 requested,
                  struct av_decision *avd,
                  int result,
                  u32 auditdeny,
                  u32 *deniedp)
{
    u32 denied, audited;
  //最重要的就是这个计算
    denied = requested & ~avd->allowed;
    ......
}
```

那么最后计算的结果就是

0x00400004 & ~0x00480010 = 0x00000004

根据前面的描述，这个权限就是 “write”

所以会看到avc denied信息里面会提示你缺少write权限

```bash
[ 5424.996583@0]- type=1400 audit(1577887433.668:59): avc: denied { write } for pid=4021 comm="testA.sh"
name="vendor" dev="mmcblk0p20" ino=7395 scontext=u:r:testA:s0 tcontext=u:object_r:vendor_data_file:s0 tclass=dir permissive=0
```

上面提到的只是一个简单的例子，其他的权限也可以用类似的方式分析

我们用一张图来总结下：

{{< image classes="fancybox center fig-100" src="/SElinux/selinux_9.png" thumbnail="/SElinux/selinux_9.png" title="">}}



### 3.1.4其他

下面聊聊servicemanager里面一些权限的检查，这个是Android才有的，具体流程可以点击[这里](http://gityuan.com/2015/11/07/binder-start-sm/)

我们在这直接看到svcmgr_handler

比较常看见的两个权限问题，find和add

```c++
case SVC_MGR_CHECK_SERVICE:
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }
    handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
    if (!handle)
        break;
    bio_put_ref(reply, handle);
    return 0;
 
case SVC_MGR_ADD_SERVICE:
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }
    handle = bio_get_ref(msg);
    allow_isolated = bio_get_uint32(msg) ? 1 : 0;
    dumpsys_priority = bio_get_uint32(msg);
    if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated, dumpsys_priority,
                       txn->sender_pid))
        return -1;
    break;
```

这里面的调用栈都会跑到kernel里面，和前面的分析都差不多的

```c++
do_add_service -> svc_can_register -> check_mac_perms_from_lookup -> check_mac_perms ->
selinux_check_access -> avc_has_perm -> avc_has_perm_noaudit
```

重点的这里的权限需要哪些，这个是在service_manager.c里面指定的

比如add 和 find ：

```c++
static int svc_can_find(const uint16_t *name, size_t name_len, pid_t spid, uid_t uid)
{
    const char *perm = "find";
    return check_mac_perms_from_lookup(spid, uid, perm, str8(name, name_len)) ? 1 : 0;
}
 
static int svc_can_register(const uint16_t *name, size_t name_len, pid_t spid, uid_t uid)
{
    const char *perm = "add";
 
    if (multiuser_get_app_id(uid) >= AID_APP) {
        return 0; /* Don't allow apps to register services */
    }
 
    return check_mac_perms_from_lookup(spid, uid, perm, str8(name, name_len)) ? 1 : 0;
}
```

不过这里的打印不再是kernel log里面的了，而是logcat信息里面，那它的打印是怎么来的？

```bash
06-21 13:12:58.631  2832  2832 E SELinux : avc:  denied  { find } for service=activity pid=4138 uid=0 scontext=u:r:testA:s0 tcontext=u:object_r:activity_service:s0 tclass=service_manager permissive=1
```

在前面提到的avc_has_perm函数里面，还有一句

```c++
rc2 = avc_audit(ssid, tsid, tclass, requested, &avd, rc, auditdata, 0);
```

/android/external/selinux/libselinux/src/avc.c

```c++
void avc_audit(security_id_t ssid, security_id_t tsid,
           security_class_t tclass, access_vector_t requested,
           struct av_decision *avd, int result, void *a)
{
    access_vector_t denied, audited;
 
    denied = requested & ~avd->allowed;
    if (denied)
        audited = denied & avd->auditdeny;
    else if (!requested || result)
        audited = denied = requested;
    else
        audited = requested & avd->auditallow;
    if (!audited)
        return;
#if 0
    if (!check_avc_ratelimit())
        return;
#endif
    /* prevent overlapping buffer writes */
    avc_get_lock(avc_log_lock);
    snprintf(avc_audit_buf, AVC_AUDIT_BUFSIZE,
         "%s:  %s ", avc_prefix, (denied || !requested) ? "denied" : "granted");
    avc_dump_av(tclass, audited);
    log_append(avc_audit_buf, " for ");
 
    /* get any extra information printed by the callback */
    avc_suppl_audit(a, tclass, avc_audit_buf + strlen(avc_audit_buf),
            AVC_AUDIT_BUFSIZE - strlen(avc_audit_buf));
 
    log_append(avc_audit_buf, " ");
    avc_dump_query(ssid, tsid, tclass);
 
    if (denied)
        log_append(avc_audit_buf, " permissive=%u", result ? 0 : 1);
 
    log_append(avc_audit_buf, "\n");
    avc_log(SELINUX_AVC, "%s", avc_audit_buf);
 
    avc_release_lock(avc_log_lock);
}
```

最后调用的是avc_log打印的，而这个其实这里是回调service_manager.c设置的callback函数

```c++
cb.func_audit = audit_callback;
selinux_set_callback(SELINUX_CB_AUDIT, cb);
cb.func_log = selinux_log_callback;
selinux_set_callback(SELINUX_CB_LOG, cb);
 
 
int selinux_log_callback(int type, const char *fmt, ...)
{
    va_list ap;
    int priority;
    char *strp;
 
    switch(type) {
    case SELINUX_WARNING:
        priority = ANDROID_LOG_WARN;
        break;
    case SELINUX_INFO:
        priority = ANDROID_LOG_INFO;
        break;
    default:
        priority = ANDROID_LOG_ERROR;
        break;
    }
 
    va_start(ap, fmt);
    if (vasprintf(&strp, fmt, ap) != -1) {
        LOG_PRI(priority, "SELinux", "%s", strp);
        LOG_EVENT_STRING(AUDITD_LOG_TAG, strp);
        free(strp);
    }
    va_end(ap);
    return 0;
}
```

关于Android里面的属性，也有相应的权限检查机制，有兴趣的可以看看相关代码~

## 3.2avc denied信息分析

kernel log中的avc denied来自于common_lsm_audit，还有函数avc_audit_pre_callback、avc_audit_post_callback，把avc denied的每一段信息拼接起来

最后是用printk打印到kernel log里面

kernelcode/security/selinux/avc.c

```c++
/* This is the slow part of avc audit with big stack footprint */
noinline int slow_avc_audit(u32 ssid, u32 tsid, u16 tclass,
        u32 requested, u32 audited, u32 denied, int result,
        struct common_audit_data *a,
        unsigned flags)
{
    ......
    common_lsm_audit(a, avc_audit_pre_callback, avc_audit_post_callback);
    return 0;
}
```

下面针对几个关键的信息分析下

- { write }
  缺失的权限，这个是本文讨论的核心部分

打印来自于函数 avc_dump_av (kernelcode/security/selinux/avc.c)

```c++
/**
 * avc_dump_av - Display an access vector in human-readable form.
 * @tclass: target security class
 * @av: access vector
 */
static void avc_dump_av(struct audit_buffer *ab, u16 tclass, u32 av)
{
    const char **perms;
    int i, perm;
 
    if (av == 0) {
        audit_log_format(ab, " null");
        return;
    }
 
    BUG_ON(!tclass || tclass >= ARRAY_SIZE(secclass_map));
    perms = secclass_map[tclass-1].perms;
 
    audit_log_format(ab, " {");
    i = 0;
    perm = 1;
    while (i < (sizeof(av) * 8)) {
        if ((perm & av) && perms[i]) {
            audit_log_format(ab, " %s", perms[i]);
            av &= ~perm;
        }
        i++;
        perm <<= 1;
    }
 
    if (av)
        audit_log_format(ab, " 0x%x", av);
 
    audit_log_format(ab, " }");
}
```

分析avc_dump_av代码逻辑，其实就是在secclass_map数组里面找到权限对应的字符串，打印出来方便分析

和前面部分提到的内容是相关的

for pid=4021

进程ID

comm=“testA.sh”

进程的名称，但是这个字符串设计的是有长度限制的，所以经常看见一些apk的包名这里会打印不完整

```bash
[ 5786.420602@3] type=1400 audit(1624264582.595:426): avc: denied { open } for pid=4686 comm="testA.launcher3" path="/proc/vmstat" dev="proc" ino=4026532002 scontext=u:r:untrusted_app:s0:c46,c256,c512,c768 tcontext=u:object_r:proc_vmstat:s0 tclass=file permissive=1
 
实际上进程名是com.testA.launcher3
```

- scontext=u:r:testA:s0
  主体的安全上下文，这里就是脚本所在的进程testA.sh

- tcontext=u:object_r:vendor_data_file:s0

  被操作对象的安全上下文，这里是/data/vendor/目录的安全上下文，默认在这个目录下面创建的文件的安全上下文，都会继承这个父目录的安全上下文

- tclass=dir
  种类，dir是目录，file文件，这个也比较好理解

所以后面写te规则的时候，其实就是

```c++
allow scontext tcontext:tclass perms;
 
也就是：
 
allow testA vendor_data_file:dir { write };
```

有个工具叫audit2allow，可以将报错信息自动生成te规则，其实讲实话我并不建议使用

首先，格式有要求，动不动转失败，还有转漏的；

其次，你都不清楚需要的权限是什么，就一顿操作，把所有报错信息都加上去了，可能加的一些权限和问题没有关联，结果还违反neverallow规则，改完以后根本追溯不了

我建议直接看log信息添加权限信息，结合SELinux的知识，添加的权限有理有据，也可以提升自己的能力



## 3.3调试

### 3.3.1快速编译和替换

#### 3.3.1.1编译和替换

| 修改的内容        | 编译命令                                           | 替换方式                                                     |
| ----------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| *.te              | make precompiled_sepolicy 或者 make selinux_policy | /vendor/etc/selinux/precompiled_sepolicy 或者 /odm/etc/selinux/precompiled_sepolicy |
| property_contexts | 可以直接板子上验证                                 | 直接到板子上改 /vendor/etc/selinux/vendor_property_contexts、 /system/etc/selinux/plat_property_contexts |
| file_contexts     | 可以直接板子上验证                                 | 直接到板子上改 /vendor/etc/selinux/vendor_file_contexts、 /system/etc/selinux/plat_file_contexts |
| service_contexts  | 可以直接板子上验证                                 | 直接到板子上改<br/>/vendor/etc/selinux/vendor_hwservice_contexts、<br/>/system/etc/selinux/plat_hwservice_contexts、<br/>/system/etc/selinux/plat_service_contexts |

有很多文章提到在Android 9.0上，编译策略文件要用make sepolicy然后替换/vendor/etc/selinux /system/etc/selinux 两个目录，其实这里是有问题的

1、make sepolicy之后，/vendor/etc/selinux /system/etc/selinux 目录下的产物是不会发生变化，因此替换整个目录是没有作用的（如果lunch的时候target发生变化，那也只是第一次会重新生成，后面再修改te也不会变）

2、system/sepolicy目录下, mma -j编译后，替换是可以的，因为这个相当于把android/system/sepolicy/Android.mk 的所有module都编译了一遍

讲讲由来：

从Android 8.0之后，因为Google project treble的影响，二进制策略文件不再是根目录的 /sepolicy （也就是make sepolicy 编译生成的产物 root/sepolicy）

而是 vendor/etc/selinux/precompiled_sepolicy （或者是odm/etc/selinux/precompiled_sepolicy）

相应的，编译的命令也有变化，修改了*.te，更正确的应该是使用make precompiled_sepolicy

生成 vendor/etc/selinux/precompiled_sepolicy 替换到板子即可 (如果存在odm分区，生成在 odm/etc/selinux/precompiled_sepolicy，替换到板子的路径也是对应的)

那make sepolicy和make precompiled_sepolicy 有什么区别？

编译出来的产物其实内容是一样的，但是生成路径不一样

之所以说有误解，是因为我们没有搞清楚真正要替换的是什么

make selinux_policy 我比较常使用，因为这个编译范围把需要的都编译了，在对应目录下的文件都会更新

这里还需要注意一下，修改/vendor 、 /system下面文件的安全上下文（在/vendor/etc/selinux/vendor_file_contexts、/system/etc/selinux/plat_file_contexts）时，

可能出现修改了没有生效的问题，要注意用restorecon命令恢复一下安全上下文，这些只读分区的文件是一开始编译的时候确定下来的，

所以会发现有些时候没有改成功，最关键的就是出现这些问题的时候，多用ls -Z、getprop -Z看看安全上下文

比如：

```bash
picasso:/ # ls -lZ /system/xbin/testB
-rwxr-xr-x 1 root shell u:object_r:system_file:s0 20380 2009-01-01 00:00 /system/xbin/testB
 
修改了file_context
 
picasso:/ # restorecon -R /system/xbin/testB
SELinux: Loaded file_contexts
picasso:/ #
picasso:/ # ls -lZ /system/xbin/testB
-rwxr-xr-x 1 root shell u:object_r:testB_exec:s0 20380 2009-01-01 00:00 /system/xbin/testB
```



#### 3.3.1.2load_policy

上面的方式在替换之后需要重启，开机会重新load新的二进制策略文件

还有一个命令也很方便：

load_policy

我们可以直接加载二进制策略文件

```c++
load_policy  /storage/xxxx/precompiled_sepolicy
```

不过只在本次开机才有效，重启就没有效果了，所以如果是开机过程的SELinux权限问题，还是得乖乖的替换文件



### 3.3.2尽量使用宏

对于一些文件和目录的操作，我们可以尽量使用宏

比如：

```te
allow testC vendor_data_file:dir { write add_name create getattr read setattr open remove_name rmdir };
allow testC vendor_data_file:file { create read write open ioctl getattr lock append unlink setattr };
```

这里脚本的目的是要创建目录和文件

如果按我们现在的方式，等着系统一条一条的报权限问题，那这个过程会发现你得反复添加编译再替换到板子，很浪费时间

但其实可以直接添加下面的权限：

```te
allow testC vendor_data_file:dir create_dir_perms;
allow testC vendor_data_file:file create_file_perms;
```

create_dir_perms和create_file_perms的定义在/android/system/sepolicy/public/global_macros

如果只是读取，那就直接用r_file_perms、r_dir_perms

使用宏会比较快的解决文件和目录相关的操作，这个也是遇到比较多的一种

还有一个宏定义的地方：

/android/system/sepolicy/public/te_macros

对于属性，通常用的是这两个

```c++
get_prop(testC, system_prop)
set_prop(testC, system_prop)
```

这里是有包含关系的，set_prop宏里面是包含了get_prop的，所以其实写了set_prop就不用写get_prop了

对于binder，这两个用的比较多

```c++
binder_use
binder_call
```

遇到一些权限问题的时候，可以先看看te_macros里面的，搜一搜，看看注释，会有很大帮助的

当然，还可以自定义宏，可以少写很多重复的规则，比如下面这个：

```te
define(`testD_common_file_dir_r_perms', `
allow testD $1:file r_file_perms;
allow testD $1:dir r_dir_perms;
')
 
然后使用：
testD_common_file_dir_r_perms(init)
testD_common_file_dir_r_perms(kernel)
testD_common_file_dir_r_perms(logd)
testD_common_file_dir_r_perms(vold)
testD_common_file_dir_r_perms(audioserver)
testD_common_file_dir_r_perms(lmkd)
testD_common_file_dir_r_perms(tee)
testD_common_file_dir_r_perms(surfaceflinger)
testD_common_file_dir_r_perms(hdmicecd)
testD_common_file_dir_r_perms(hwservicemanager)
testD_common_file_dir_r_perms(netd)
testD_common_file_dir_r_perms(sdcardfs)
testD_common_file_dir_r_perms(servicemanager)
```



### 3.3.3使用属性

如果出现下面这种一直报的同一类型和权限的
比如service_manager { find }

```te
allow testD activity_service:service_manager { find };
allow testD package_service:service_manager { find };
allow testD window_service:service_manager { find };
allow testD meminfo_service:service_manager { find };
allow testD cpuinfo_service:service_manager { find };
allow testD mount_service:service_manager { find };
allow testD surfaceflinger_service:service_manager { find };
allow testD audioserver_service:service_manager { find };
allow testD secure_element_service:service_manager { find };
allow testD contexthub_service:service_manager { find };
allow testD netd_listener_service:service_manager { find };
allow testD connmetrics_service:service_manager { find };
allow testD bluetooth_manager_service:service_manager { find };
allow testD imms_service:service_manager { find };
allow testD cameraproxy_service:service_manager { find };
allow testD slice_service:service_manager { find };
allow testD media_projection_service:service_manager { find };
allow testD crossprofileapps_service:service_manager { find };
allow testD launcherapps_service:service_manager { find };
allow testD shortcut_service:service_manager { find };
allow testD media_router_service:service_manager { find };
allow testD hdmi_control_service:service_manager { find };
allow testD media_session_service:service_manager { find };
allow testD restrictions_service:service_manager { find };
allow testD graphicsstats_service:service_manager { find };
allow testD dreams_service:service_manager { find };
allow testD commontime_management_service:service_manager { find };
allow testD network_time_update_service:service_manager { find };
allow testD diskstats_service:service_manager { find };
allow testD voiceinteraction_service:service_manager { find };
allow testD appwidget_service:service_manager { find };
allow testD backup_service:service_manager { find };
allow testD trust_service:service_manager { find };
allow testD voiceinteraction_service:service_manager { find };
allow testD jobscheduler_service:service_manager { find };
allow testD hardware_properties_service:service_manager { find };
allow testD serial_service:service_manager { find };
allow testD usb_service:service_manager { find };
allow testD DockObserver_service:service_manager { find };
allow testD audio_service:service_manager { find };
allow testD wallpaper_service:service_manager { find };
allow testD search_service:service_manager { find };
allow testD country_detector_service:service_manager { find };
allow testD location_service:service_manager { find };
allow testD devicestoragemonitor_service:service_manager { find };
allow testD notification_service:service_manager { find };
allow testD updatelock_service:service_manager { find };
allow testD system_update_service:service_manager { find };
allow testD servicediscovery_service:service_manager { find };
allow testD connectivity_service:service_manager { find };
allow testD ethernet_service:service_manager { find };
allow testD wifip2p_service:service_manager { find };
allow testD wifiscanner_service:service_manager { find };
allow testD wifi_service:service_manager { find };
allow testD netpolicy_service:service_manager { find };
allow testD netstats_service:service_manager { find };
allow testD network_score_service:service_manager { find };
allow testD textclassification_service:service_manager { find };
allow testD textservices_service:service_manager { find };
allow testD ipsec_service:service_manager { find };
allow testD network_management_service:service_manager { find };
allow testD clipboard_service:service_manager { find };
allow testD statusbar_service:service_manager { find };
allow testD device_policy_service:service_manager { find };
allow testD deviceidle_service:service_manager { find };
allow testD uimode_service:service_manager { find };
allow testD lock_settings_service:service_manager { find };
allow testD storagestats_service:service_manager { find };
allow testD accessibility_service:service_manager { find };
allow testD input_method_service:service_manager { find };
allow testD pinner_service:service_manager { find };
allow testD network_watchlist_service:service_manager { find };
allow testD vr_manager_service:service_manager { find };
allow testD input_service:service_manager { find };
```

可以找到这些type的共同点，也就是attribute

上面的可以简化为下面的，减去的哪些type是因为违反了neverallow规则的

```te
allow testD { service_manager_type -stats_service -incident_service -vold_service -netd_service -installd_service -dumpstate_service }:service_manager { find };
```



### 3.3.4setools工具

这个工具挺好用的，可以分析二进制策略文件，比如用于某些情况下判断权限是否加上了，可以点击[下载setools3 工具](https://github.com/TresysTechnology/setools3)

```bash
$ sesearch -A precompiled_sepolicy | grep kernel
allow kernel app_data_file:file read;
allow kernel asec_image_file:file read;
allow kernel binder_device:chr_file { append getattr ioctl lock map open read write };
allow kernel debugfs_mali:dir search;
allow kernel device:blk_file { create getattr setattr unlink };
allow kernel device:chr_file { create getattr setattr unlink };
allow kernel device:dir { add_name append create getattr ioctl lock map open read remove_name rmdir search write };
allow kernel file_contexts_file:file { getattr ioctl lock map open read };
allow kernel hwbinder_device:chr_file { append getattr ioctl lock map open read write };
allow kernel init:process { rlimitinh share siginh transition };
allow kernel init_exec:file { execute getattr map open read relabelto };
allow kernel kernel:cap_userns { sys_boot sys_nice sys_resource };
allow kernel kernel:capability { mknod sys_boot sys_nice sys_resource };
allow kernel kernel:dir { getattr ioctl lock open read search };
allow kernel kernel:fd use;
allow kernel kernel:fifo_file { append getattr ioctl lock map open read write };
allow kernel kernel:file { append getattr ioctl lock map open read write };
allow kernel kernel:lnk_file { getattr ioctl lock map open read };
allow kernel kernel:process { fork getattr getcap getpgid getsched getsession setcap setpgid setrlimit setsched sigchld sigkill signal signull sigstop };
allow kernel kernel:security setcheckreqprot;
......
```

# 4CTS neverallow处理

## 4.1一些权限的解决方案

### 4.1.1区分vendor域和system域

按照以往的经验，我们在Android P上添加客制化的脚本，还是喜欢用 #!/system/bin/sh

但是自从Android 8.0的Treble Project之后，vendor和system已经开始分离了，vendor下面的脚本要使用 #!/vendor/bin/sh，这样能减少很多权限问题

举个例子：

`/vendor/bin/testA.sh`

```shell
#!/system/bin/sh
 
...
```

然后运行的时候会报下面的权限问题

```bash
[  973.048826@1]- type=1400 audit(1619667483.432:167): avc: denied { read execute } for pid=4368 comm="testA.sh"
path="/system/bin/sh" dev="mmcblk0p17" ino=1073 scontext=u:r:testA:s0 tcontext=u:object_r:shell_exec:s0 tclass=file permissive=0
```

这个时候如果直接在te里面添加权限：

```te
allow testA shell_exec:file { read execute };
```

添加完之后就会发现违反了neverallow规则

```bash
FAILED: out/target/product/xxxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
/bin/bash -c "(rm -f out/target/product/xxxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows ) && (ASAN_OPTIONS=detect_leaks=0 out/host/linux-x86/bin/checkpolicy -M -c               30 -o out/target/product/xxxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows out/target/product/xxxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf )"
libsepol.report_failure: neverallow on line 1047 of system/sepolicy/public/domain.te (or line 11499 of policy.conf) violated by allow testA shell_exec:file { execute };
libsepol.check_assertions: 1 neverallow failures occurred
Error while expanding policy
out/host/linux-x86/bin/checkpolicy:  loading policy configuration from out/target/product/xxxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf
ninja: build stopped: subcommand failed.
11:42:19 ninja failed with: exit status 1
```

原因在domain.te的注释里面说的很清楚了，vendor的组件不允许直接运行system的file

```shell
full_treble_only(`
    # Do not allow vendor components to execute files from system
    # except for the ones whitelist here.
    neverallow {
```

修改脚本为：

```shell
#!/vendor/bin/sh
 
...
```

可以解决上述问题

但是改完之后，有些在/system/bin/ 下的命令就用不了，比如pm、am、logcat这些Android才有的命令，而不是在toybox里面的

如果实在要用，可以这样操作：

```shell
#!/vendor/bin/sh
 
am broadcast -a com.test.broadcast.toast -e Message "hello world" > /dev/picasso

直接运行也会报错
picasso:/ # /vendor/bin/testA.sh[3]: am: not found
```

可以修改成：

```shell
/system/bin/cmd activity broadcast -a com.test.broadcast.toast -e Message "hello world" > /dev/picasso
```

te里面要补充关联上属性： vendor_executes_system_violators

```shell
type testA, domain, vendor_executes_system_violators;
```



### 4.1.2setenforce的可行性

这个肯定是没有办法解决的，如果随便一个脚本都可以拥有关闭SELinux的权限，那还得了

在userdebug和eng版本上，可以先这样debug搞：

添加permissive testA;

对testA type以permissive状态运行

不过只能在eng和userdebug版本下才能用

```shell
type testA, coredomain, domain, mlstrustedsubject;
type testA_exec, exec_type, file_type;
 
permissive testA;

init_daemon_domain(testA)
```

要过CTS，这个权限可是天敌啊，主要就是有些功能想做的事情太多，权限太大，搞得非得要临时关闭SELinux来处理

跟着代码实现，不难发现setenforce命令的操作其实就是写节点/sys/fs/selinux/enforce

结合前面提到的权限检查原理，很快可以发现，这个权限就是我们无法绕过的根本原因了

```c++
static ssize_t sel_write_enforce(struct file *file, const char __user *buf,
                 size_t count, loff_t *ppos)
 
{
    ......
        length = task_has_security(current, SECURITY__SETENFORCE);
        if (length)
            goto out;
    ......
}
```

这个权限检查就是后面报avc denied的地方了

```te
allow testA kernel:security { setenforce };
```

对应的neverallow规则如下：

```te
# Only init prior to switching context should be able to set enforcing mode.
# init starts in kernel domain and switches to init domain via setcon in
# the init.rc, so the setenforce occurs while still in kernel. After
# switching domains, there is never any need to setenforce again by init.
neverallow * kernel:security setenforce;
neverallow { domain -kernel } kernel:security setcheckreqprot;
```

这儿有个不是办法的办法：就是不影响现有命令的基础上，新增了一个节点，然后直接不做权限检查

```bash
diff --git a/security/selinux/selinuxfs.c b/security/selinux/selinuxfs.c
index 72c145dd799f..b2332f55415c 100644
--- a/security/selinux/selinuxfs.c
+++ b/security/selinux/selinuxfs.c
@@ -100,6 +100,7 @@ enum sel_inos {
        SEL_ROOT_INO = 2,
        SEL_LOAD,       /* load policy */
        SEL_ENFORCE,    /* get or set enforcing status */
+       SEL_ENABLE,     /* get or set enforcing status */
        SEL_CONTEXT,    /* validate context */
        SEL_ACCESS,     /* compute access decision */
        SEL_CREATE,     /* compute create labeling decision */
@@ -193,6 +194,57 @@ static const struct file_operations sel_enforce_ops = {
        .llseek         = generic_file_llseek,
 };
  
+#ifdef CONFIG_SECURITY_SELINUX_DEVELOP
+static ssize_t sel_write_enable(struct file *file, const char __user *buf,
+                size_t count, loff_t *ppos)
+
+{
+    char *page = NULL;
+    ssize_t length;
+    int new_value;
+
+    if (count >= PAGE_SIZE)
+        return -ENOMEM;
+
+    /* No partial writes. */
+    if (*ppos != 0)
+        return -EINVAL;
+
+    page = memdup_user_nul(buf, count);
+    if (IS_ERR(page))
+        return PTR_ERR(page);
+
+    length = -EINVAL;
+    if (sscanf(page, "%d", &new_value) != 1)
+        goto out;
+
+    if (new_value != selinux_enforcing) {
+        audit_log(current->audit_context, GFP_KERNEL, AUDIT_MAC_STATUS,
+            "enforcing=%d old_enforcing=%d auid=%u ses=%u",
+            new_value, selinux_enforcing,
+            from_kuid(&init_user_ns, audit_get_loginuid(current)),
+            audit_get_sessionid(current));
+        selinux_enforcing = new_value;
+        if (selinux_enforcing)
+            avc_ss_reset(0);
+        selnl_notify_setenforce(selinux_enforcing);
+        selinux_status_update_setenforce(selinux_enforcing);
+    }
+    length = count;
+out:
+    kfree(page);
+    return length;
+}
+#else
+#define sel_write_enable NULL
+#endif
+
+static const struct file_operations sel_enable_ops = {
+    .read       = sel_read_enforce,
+    .write      = sel_write_enable,
+    .llseek     = generic_file_llseek,
+};
+
 static ssize_t sel_read_handle_unknown(struct file *filp, char __user *buf,
                                        size_t count, loff_t *ppos)
 {
@@ -1790,6 +1842,7 @@ static int sel_fill_super(struct super_block *sb, void *data, int silent)
        static struct tree_descr selinux_files[] = {
                [SEL_LOAD] = {"load", &sel_load_ops, S_IRUSR|S_IWUSR},
                [SEL_ENFORCE] = {"enforce", &sel_enforce_ops, S_IRUGO|S_IWUSR},
+               [SEL_ENABLE] = {"enable", &sel_enable_ops, S_IRUGO|S_IWUSR},
                [SEL_CONTEXT] = {"context", &transaction_ops, S_IRUGO|S_IWUGO},
                [SEL_ACCESS] = {"access", &transaction_ops, S_IRUGO|S_IWUGO},
                [SEL_CREATE] = {"create", &transaction_ops, S_IRUGO|S_IWUGO},
```

然后开关就用：

关SELinux

```shell
echo 0 > /sys/fs/selinux/enable
```

开SELinux

```shell
echo 1 > /sys/fs/selinux/enable
```

权限还得补一下：

```te
allow testA selinuxfs:file rw_file_perms;
selinux_check_access(testA)
```

相比setenforce，差异就是去掉了权限检查，开了一个后门，比较bug的存在

### 4.1.3dac_override 解决办法

谷歌官方文档的一段话总结的很精辟

> **授予dac_override权能**
>
> `dac-override`拒绝事件意味着违规进程正在尝试使用错误的unix user/group/world权限访问某个文件。正确的解决方案几乎从不授予`dac-override`权限，而是更改相应文件或进程的unix权限。有些网域（例如`init`、`vold`和`installd`)确实需要能够蓄换unix文件权限才能访问其他进程的文件。如需查看更深入的讲解，请访问Dan Walsh的博客。

关于解决selinux dac_override/dac_read_search问题，有一个固定思路，可以点击[这里](https://blog.csdn.net/Donald_Zhuang/article/details/108786482)

比如修改group的：

未修改前：

```shell
service testA /system/bin/testA.sh -start
    user root
    group root
    disabled
    oneshot
    seclabel u:r:testA:s0
```

修改后：

```shell
service testA /system/bin/testA.sh -start
    user root
    group root system
    disabled
    oneshot
    seclabel u:r:testA:s0
```

大部分情况下，我们可以直接ls -l 查看要访问的文件或者目录的user 和 group

但是有些情况下，我们很难搞清楚访问的文件或者目录所在的group是哪个，这些信息在avc denied里面是不会有的

可以在kernel中补充这些代码，打印出来

patch:

```bash
kernelcode$ git diff .
diff --git a/fs/namei.c b/fs/namei.c
index a5a05d3..9b8f0da 100644
--- a/fs/namei.c
+++ b/fs/namei.c
@@ -39,6 +39,8 @@
 #include <linux/init_task.h>
 #include <asm/uaccess.h>
  
+#include <linux/printk.h>
+
 #include "internal.h"
 #include "mount.h"
  
@@ -341,8 +343,12 @@ int generic_permission(struct inode *inode, int mask)
  
        if (S_ISDIR(inode->i_mode)) {
                /* DACs are overridable for directories */
-               if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
-                       return 0;
+        if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE)) {
+            return 0;
+        } else {
+            pr_err("dac_override(dir) denied for cap=%3o inode=%lu with uid=%u gid=%u\n",
+                mask&0777, inode->i_ino, inode->i_uid.val, inode->i_gid.val);
+        }
                if (!(mask & MAY_WRITE))
                        if (capable_wrt_inode_uidgid(inode,
                                                     CAP_DAC_READ_SEARCH))
@@ -354,9 +360,17 @@ int generic_permission(struct inode *inode, int mask)
         * Executable DACs are overridable when there is
         * at least one exec bit set.
         */
-       if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
-               if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
-                       return 0;
+       // if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
+       //      if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
+    /* IXUGO = (S_IXUSR|S_IXGRP|S_IXOTH) */
+    if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO)) {
+        if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE)) {
+            return 0;
+        } else {
+            pr_err("dac_override(file) denied for cap=%3o inode=%lu with uid=%u gid=%u\n",
+                mask&0777, inode->i_ino, inode->i_uid.val, inode->i_gid.val);
+        }
+    }
  

diff --git a/security/selinux/avc.c b/security/selinux/avc.c
index e60c79d..9b1dfad 100644
--- a/security/selinux/avc.c
+++ b/security/selinux/avc.c
@@ -734,7 +734,11 @@ static void avc_audit_post_callback(struct audit_buffer *ab, void *a)
        if (ad->selinux_audit_data->denied) {
                audit_log_format(ab, " permissive=%u",
                                 ad->selinux_audit_data->result ? 0 : 1);
-       }
+        if (ad->u.cap == 1) {
+            audit_log_format(ab, " cap=dac_override, dumping stack");
+            WARN_ON(ad->u.cap == 1);
+        }
+    }
 }
  
 /* This is the slow part of avc audit with big stack footprint */
```

打印信息如下：

```bash
[   44.952920@0]- ------------[ cut here ]------------
[   44.952983@0]- WARNING: CPU: 0 PID: 3901 at :739 avc_audit_post_callback+0x180/0x184
[   44.961323@0]- Modules linked in: hardware_dmx(O) jpegenc(O) amvdec_av1(O) amvdec_mavs(O) encoder(O) amvdec_avs2(O) amvdec_vp9(O) amvdec_vc1(O) amvdec_real(O) amvdec_mmpeg4(O) amvdec_mpeg4(O) amvdec_mmpeg12(O) amvdec_mpeg12(O) amvdec_mmjpeg(O) amvdec_mjpeg(O) amvdec_h265(O) amvdec_h264mvc(O) amvdec_mh264(O) amvdec_h264(O) amvdec_avs(O) stream_input(O) decoder_common(O) video_framerate_adapter(O) firmware(O) media_clock(O) tb_detect(PO) mali(O) dnlp_alg(P)
[   45.000226@0]d CPU: 0 PID: 3901 Comm: cmd Tainted: P        W  O    4.9.113 #20
[   45.007459@0]d Hardware name: Generic DT based system
[   45.012491@0]d [bc4afac4+  16][<c020dc10>] show_stack+0x20/0x24
[   45.018343@0]d [bc4afae4+  32][<c054394c>] dump_stack+0x90/0xac
[   45.024201@0]d [bc4afb14+  48][<c0228724>] __warn+0x104/0x11c
[   45.029897@0]d [bc4afb2c+  24][<c022880c>] warn_slowpath_null+0x30/0x38
[   45.036456@0]d [bc4afb5c+  48][<c04d4fec>] avc_audit_post_callback+0x180/0x184
[   45.043606@0]d [bc4afbb4+  88][<c04f20e0>] common_lsm_audit+0x260/0x7c0
[   45.050165@0]d [bc4afbfc+  72][<c04d5994>] slow_avc_audit+0x9c/0xb0
[   45.056376@0]d [bc4afc64+ 104][<c04da364>] cred_has_capability+0x118/0x124
[   45.063184@0]d [bc4afc74+  16][<c04da3f0>] selinux_capable+0x38/0x3c
[   45.069483@0]d [bc4afc9c+  40][<c04d1370>] security_capable+0x4c/0x68
[   45.075870@0]d [bc4afcac+  16][<c0233d58>] ns_capable_common+0x7c/0x90
[   45.082336@0]d [bc4afcc4+  24][<c0233e00>] capable_wrt_inode_uidgid+0x28/0x54
[   45.089409@0]d [bc4afcf4+  48][<c03a5abc>] generic_permission+0x100/0x208
[   45.096138@0]d [bc4afd14+  32][<c0414df0>] kernfs_iop_permission+0x58/0x64
[   45.102951@0]d [bc4afd34+  32][<c03a5c84>] __inode_permission2+0xc0/0xf0
[   45.109589@0]d [bc4afd44+  16][<c03a5cfc>] inode_permission2+0x20/0x54
[   45.116057@0]d [bc4afd8c+  72][<c0396fb4>] SyS_faccessat+0xec/0x220
[   45.122273@0]d [00000000+   0][<c02085c0>] ret_fast_syscall+0x0/0x48
[   45.128879@0]s DI: DI: tasklet schedule cost 128ms.
[   45.133835@0]- ---[ end trace 40474d395a0d6c86 ]---
[   45.138462@0]- dac_override(file) denied for cap= 22 inode=5 with uid=1000 gid=1000
[   45.138640@1]- type=1400 audit(1620051338.180:75): avc: denied { dac_override } for pid=3900 comm="cmd" capability=1 scontext=u:r:testA:s0 tcontext=u:r:testA:s0 tclass=capability permissive=0 cap=dac_override, dumping stack
```

打印的gid在 system/core/include/private/android_filesystem_config.h 里面查找，就知道init.rc里面要加什么了

### 4.1.4echo打印到终端

```bash
#echo "test" > /dev/picasso
allow testA picasso_device:chr_file rw_file_perms;
```



### 4.1.5读写U盘内容

读U盘：

```te
allow testA mnt_media_rw_file:dir { search read open };
allow testA vfat:file r_file_perms;
allow testA vfat:dir r_dir_perms;
```

要写和创建的话，权限给大一点就行了：

```te
allow testA mnt_media_rw_file:dir { search read open };
allow testA vfat:dir create_dir_perms;
allow testA vfat:file create_file_perms;
```



### 4.1.6am 命令

```te
allow testA activity_service:service_manager { find };
allow testA system_server:binder { call transfer };
binder_use(testA)
binder_call(system_server, testA)
```

pm命令同理

### 4.1.7ioctl

```c++
[   57.670632] type=1400 audit(1623998437.166:9): avc: denied { ioctl } for pid=4285 comm="Thread-43" path="socket:[30690]" dev="sockfs" ino=30690 ioctlcmd=0x8927 scontext=u:r:untrusted_app_25:s0:c512,c768 tcontext=u:r:untrusted_app_25:s0:c512,c768 tclass=tcp_socket permissive=0
```

ioctl有少许不一样，Andrid o之后有所变化

> `ioctl`
>
> Google在Android Q上增强了对ioctl的审查，除保持对ioctl的审查/授权之外，对具体的ioctlcmd也需要进一步地审查/授权。具体可以点击[这里](https://www.cnblogs.com/fanglongxiang/p/13306088.html)解决方案。
>
> ```c++
> //kernel/security/selinux/hooks.c
> ...
> error = ioctl_has_perm(cred, file, FILE__IOCTL, (u16) cmd);
> ```
>
> 对于这样的一条avc log：
>
> ```bash
> avc: denied { ioctl } for comm="BootAnimation" path="/proc/ged" dev="proc" ino=4026532249 ioctlcmd=0x6700 scontext=u:r:bootanim:s0 tcontext=u:object_r:proc_ged:s0 tclass=file permissive=0
> ```
>
> **在Android Q上，则需要：**
>
> ```te
> allow bootanim proc_ged:file ioctl;            
> allowxperm bootanim proc_ged:file ioctl 0x6700;
> ```
>
> 0x6700 即log中的ioctlcmd。
>
> **0x6700这个值一般在sepolicy目录下的ioctl_defines文件中配置。所以在Q上，上述log最终**：
>
> ```te
> ioctl_defines#
> define(`GED_BRIDGE_IO_LOG_BUF_GET', `0x6700')
>  
> bootanim.te#
> allow bootanim proc_ged:file ioctl;
> allowxperm bootanim proc_ged:file ioctl GED_BRIDGE_IO_LOG_BUF_GET;
> ```



```te
allow untrusted_app_25 self:udp_socket ioctl;
allowxperm untrusted_app_25 self:udp_socket ioctl SIOCGIFHWADDR;

两句都要添加上，缺一不可
```



## 4.2常见违反neverallow规则的解决方案

### 4.2.1指定文件、属性、服务的安全上下文，避免权限放大

这一类neverallow规则，其实搞明白为什么要这么做，是很容易解决的

有一个系统属性`persist.sys.test`

```bash
picasso:/data # getprop -Z persist.sys.test
u:object_r:system_prop:s0
```

假如你想要设置这个属性，加了一个权限

```te
set_prop(testA, system_prop)
```

结果一编译就违反neverallow规则了，这是为什么？

因为没有特别指定，`persist.sys.`开头的属性，它的安全上下问题都是`u:object_r:system_prop:s0`

```shell
//android/system/sepolicy/private/property_contexts
 
persist.sys.            u:object_r:system_prop:s0
```

一旦你给了这个权限，意味着你可以设置的属性不仅仅是persist.sys.test了，还可以是persist.sys.A、persist.sys.B 等等

这就是权限放大，neverallow规则的限定，就是为了不给你这么玩

我们要做的就是，指定文件、属性、服务的安全上下文，收缩权限

下面有一些例子：

#### 4.2.1.1testNeverallowRules131 解决方案

```bash
<Test result="fail" name="testNeverallowRules131">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow { domain -kernel -init -recovery } block_device:blk_file { open read write };
libsepol.report_failure: neverallow violated by allow testA block_device:blk_file { read write open };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

testA是因为要操作 `/dev/block/testblock`

```bash
picasso:/ # ls -lZ /dev/block/testblock
brw------- 1 root root u:object_r:block_device:s0 179,   5 1970-01-01 08:00 /dev/block/testblock
```

给这个testblock block指定type，不然就会有权限放大的问题

```te
//sepolicy/device.te
type testblock_device, dev_type;
 
sepolicy/file_contexts
/dev/block/testblock                 u:object_r:testblock_device:s0
```

接着allow语句修改为：

```te
allow testA testblock_device:blk_file { read write open };
```

#### 4.2.1.2testNeverallowRules198 解决方案

```bash
<Test result="fail" name="testNeverallowRules198">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {      domain      -coredomain      -appdomain       -data_between_core_and_vendor_violators       -socket_between_core_and_vendor_violators      -vendor_init    } {      coredomain_socket      core_data_file_type      unlabeled     }:sock_file ~{ append getattr ioctl read write };
libsepol.report_failure: neverallow violated by allow testA property_socket:sock_file { open };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

查脚本，看设置的是哪个属性，然后把这个属性的安全上下文指定到我们自定义的上面去

```te
//sepolicy/property_contexts
sys.testA.finish            u:object_r:test_prop:s0
 
sepolicy/testA.te
set_prop(testA, test_prop)
```



#### 4.2.1.3testNeverallowRules154 解决方案

```bash
<Test result="fail" name="testNeverallowRules154">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow * default_android_service:service_manager add;
libsepol.report_failure: neverallow violated by allow testA default_android_service:service_manager { add };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

首先这个违反的neverallow规则如下，禁止所有的进程添加default_android_service类型的service_manager

```te
neverallow * default_android_service:service_manager add;
```

先看看之前的解决方案：

sepolicy/service_contexts

```te
添加
test.service                               u:object_r:test_service:s0
```

service.te：

```te
添加
type test_service,    app_api_service, system_server_service, service_manager_type;
```

testA.te 里面如下：

```te
allow testA default_android_service:service_manager { add  };
```

然后再把neverallow规则干掉，神挡杀神，佛挡杀佛

```te
neverallow { domain -testA } default_android_service:service_manager add;
```

看到这里，比较疑惑的点是service_contexts和service.te，和普通文件一样，service也是可以给它定义一个新的type

但是定义了这个test_service type之后，没有生效？

从报错的信息来看，test.service的type依然还是default_android_service，这是为何？

```bash
06-10 08:50:30.172  3062  3062 E SELinux : avc:  denied  { add } for service=test.service pid=4707 uid=0 scontext=u:r:testA:s0 tcontext=u:object_r:default_android_service:s0 tclass=service_manager permissive=0
```

这时，我们查看 /system/etc/selinux/plat_service_contexts 里面的内容，发现我们添加的test.service 压根就不在里面，难怪不会生效

在板卡上直接修改添加，重启之后，就会发现报错已经变成了这样了：

```bash
06-10 08:55:15.748  3062  3062 E SELinux : avc:  denied  { add } for service=test.service pid=4732 uid=0 scontext=u:r:testA:s0 tcontext=u:object_r:test_service:s0 tclass=service_manager permissive=0
```

接下来再添加权限就行了，这个时候就不会违反neverallow规则

透过现象看本质，为什么会导致这个问题呢？

原因要从编译出plat_service_contexts那里说起，回到`/android/system/sepolicy/Android.mk`

```makefile
include $(CLEAR_VARS)
 
LOCAL_MODULE := plat_service_contexts
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
ifeq ($(PRODUCT_SEPOLICY_SPLIT),true)
LOCAL_MODULE_PATH := $(TARGET_OUT)/etc/selinux
else
LOCAL_MODULE_PATH := $(TARGET_ROOT_OUT)
endif
 
include $(BUILD_SYSTEM)/base_rules.mk
 
plat_svcfiles := $(call build_policy, service_contexts, $(PLAT_PRIVATE_POLICY))
 
plat_service_contexts.tmp := $(intermediates)/plat_service_contexts.tmp
$(plat_service_contexts.tmp): PRIVATE_SVC_FILES := $(plat_svcfiles)
$(plat_service_contexts.tmp): PRIVATE_ADDITIONAL_M4DEFS := $(LOCAL_ADDITIONAL_M4DEFS)
$(plat_service_contexts.tmp): $(plat_svcfiles)
    @mkdir -p $(dir $@)
    $(hide) m4 -s $(PRIVATE_ADDITIONAL_M4DEFS) $(PRIVATE_SVC_FILES) > $@
 
$(LOCAL_BUILT_MODULE): PRIVATE_SEPOLICY := $(built_sepolicy)
$(LOCAL_BUILT_MODULE): $(plat_service_contexts.tmp) $(built_sepolicy) $(HOST_OUT_EXECUTABLES)/checkfc $(ACP)
    @mkdir -p $(dir $@)
    sed -e 's/#.*$$//' -e '/^$$/d' $< > $@
    $(HOST_OUT_EXECUTABLES)/checkfc -s $(PRIVATE_SEPOLICY) $@
 
built_plat_svc := $(LOCAL_BUILT_MODULE)
plat_svcfiles :=
plat_service_contexts.tmp :=
```

mk里面搜索的目录是PLAT_PRIVATE_POLICY，前面有提过，这个目录是/android/system/sepolicy/private 以及 BOARD_PLAT_PRIVATE_SEPOLICY_DIR 所组成的,所以device厂商下面的sepolicy/service_contexts 没有用到

解决办法很简单：

```c++
首先指定拓展的私有目录 BOARD_PLAT_PRIVATE_SEPOLICY_DIR
BOARD_PLAT_PRIVATE_SEPOLICY_DIR += \
    xxxx/sepolicy/private
 
然后添加：
xxxx/sepolicy/private/service_contexts
 
内容为：
test.service                               u:object_r:test_service:s0
```

编译selinux_policy，就可以看到这个context被追加到plat_service_contexts了

还有，在这个地方添加的service_contexts，并不是所有情况都没有作用

看看这两段mk：

```makefile
# Include precompiled policy, unless told otherwise
ifneq ($(PRODUCT_PRECOMPILED_SEPOLICY),false)
LOCAL_REQUIRED_MODULES += precompiled_sepolicy precompiled_sepolicy.plat_and_mapping.sha256
endif
else
# The following files are only allowed for non-Treble devices.
LOCAL_REQUIRED_MODULES += \
    sepolicy \
    vendor_service_contexts
endif
```

vendor_service_contexts

```makefile
include $(CLEAR_VARS)
 
LOCAL_MODULE := vendor_service_contexts
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_ROOT_OUT)
 
include $(BUILD_SYSTEM)/base_rules.mk
 
vendor_svcfiles := $(call build_policy, service_contexts, $(PLAT_VENDOR_POLICY) $(BOARD_VENDOR_SEPOLICY_DIRS) $(REQD_MASK_POLICY))
 
vendor_service_contexts.tmp := $(intermediates)/vendor_service_contexts.tmp
$(vendor_service_contexts.tmp): PRIVATE_SVC_FILES := $(vendor_svcfiles)
$(vendor_service_contexts.tmp): PRIVATE_ADDITIONAL_M4DEFS := $(LOCAL_ADDITIONAL_M4DEFS)
$(vendor_service_contexts.tmp): $(vendor_svcfiles)
    @mkdir -p $(dir $@)
    $(hide) m4 -s $(PRIVATE_ADDITIONAL_M4DEFS) $(PRIVATE_SVC_FILES) > $@
 
$(LOCAL_BUILT_MODULE): PRIVATE_SEPOLICY := $(built_sepolicy)
$(LOCAL_BUILT_MODULE): $(vendor_service_contexts.tmp) $(built_sepolicy) $(HOST_OUT_EXECUTABLES)/checkfc $(ACP)
    @mkdir -p $(dir $@)
    sed -e 's/#.*$$//' -e '/^$$/d' $< > $@
    $(hide) $(HOST_OUT_EXECUTABLES)/checkfc -s $(PRIVATE_SEPOLICY) $@
 
built_vendor_svc := $(LOCAL_BUILT_MODULE)
vendor_svcfiles :=
vendor_service_contexts.tmp :=
 
endif
```

注释很清楚，也就是说在非treble的设备上，上面路径的service_contexts会编译到vendor_service_contexts里面；如果要生效，就得把它加到private的拓展目录

#### 4.2.1.4testNeverallowRules252 解决方案

```bash
<Test result="fail" name="testNeverallowRules252">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow { domain -init -vendor_init -system_server -dumpstate } debugfs:file { { append create link unlink relabelfrom rename setattr write } open read ioctl lock };
libsepol.report_failure: neverallow violated by allow mediaserver debugfs:file { read open };
libsepol.report_failure: neverallow violated by allow hal_memtrack_default debugfs:file { read open };
libsepol.check_assertions: 2 neverallow failures occurred
```

报错信息如下：

```bash
[  818.411706@1] type=1400 audit(1623295203.581:48): avc: denied { read } for pid=3147 comm="HwBinder:3147_3" name="test_en" dev="debugfs" ino=377 scontext=u:r:mediaserver:s0 tcontext=u:object_r:debugfs:s0 tclass=file permissive=0
```

拿其中mediaserver的来分析，查找修改记录，很快就可以得知，这个是在读写节点 /sys/kernel/debug/test/test_en 的时候出现的权限问题

和上面的处理思路一样，我们要给这个节点指定一个安全上下文，不然就是权限放大的问题

但是和普通的文件不同，这个不是添加在file_contexts里面了，而是要添加在 genfs_contexts

file_contexts 里面的指定的文件，都是编译之后就有了的，而像 /sys/kernel/debug 、 /proc 这些是系统起来之后动态生成的，这些的节点文件的安全上下文就要在genfs_contexts里面指定的

```te
genfscon debugfs /sys/kernel/debug/test/test_en u:object_r:debugfs_test:s0
```

这里很容易犯一个错误，就是直接把完整路径写到第三个字段里面去了

实际上，debugfs 指代的就是/sys/kernel/debug这个路径，后面要写的是相对于debugfs 的路径

因此，正确的写法是这样的：

```te
genfscon debugfs /test/test_en u:object_r:debugfs_test:s0
```

> `特别注意`
>
> 这里根据笔者经验，特别是/proc或者是/sys/fs目录下面，存在很多软连接，上面的例如/test/test_en这类必须为**真实路径**，否则设置te文件的权限可能无效

当然，file.te里面还得定义debugfs_test这个type

```te
type debugfs_test, fs_type, sysfs_type, debugfs_type;
```

修改之后是要编译precompiled_sepolicy的，而不是像file_contexts可以直接在板子上修改，重启之后报错如下：

```bash
[  818.411706@1] type=1400 audit(1623295203.581:48): avc: denied { read } for pid=3147 comm="HwBinder:3147_3" name="test_en" dev="debugfs" ino=377 scontext=u:r:mediaserver:s0 tcontext=u:object_r:debugfs_afc:s0 tclass=file permissive=0
```

再往下只要添加对应的权限就行了

#### 4.2.1.5exec_type

还有一个例子也是很重要的

```bash
[   10.583284] type=1400 audit(10.256:8): avc: denied { entrypoint } for pid=3233 comm="init" path="/vendor/bin/testA.sh" dev="mmcblk0p16" ino=712 scontext=u:r:testA:s0 tcontext=u:object_r:vendor_file:s0 tclass=file permissive=0
```

添加权限

```te
allow testA vendor_file:file entrypoint;
```

结果违反了neverallow规则，然后继续干掉规则呗，这还不简单

```bash
FAILED: out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows
/bin/bash -c "(rm -f out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows ) && (ASAN_OPTIONS=detect_leaks=0 out/host/linux-x86/bin/checkpolicy -M -c           30 -o out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/sepolicy_neverallows out/target/product/xxxx/obj/ETC/sepolicy_neverallows_intermediates/policy.conf )"
libsepol.report_failure: neverallow on line 376 of system/sepolicy/public/domain.te (or line 10221 of policy.conf) violated by allow testA vendor_file:file { entrypoint };
libsepol.check_assertions: 1 neverallow failures occurred
```

domain.te里面的neverallow规则：

```te
# Ensure that all entrypoint executables are in exec_type or postinstall_file.
neverallow * { file_type -exec_type -postinstall_file }:file entrypoint;
```

正确的处理方式和前面提到的是一样的，也是同个道理

```te
//sepolicy/file_contexts
 
/vendor/bin/testA.sh        u:object_r:testA_exec:s0
```

这就是为什么我们之前添加脚本的时候，每次都要加这样一句话的原因

### 4.2.2利用attributes绕开neverallow规则

这一类处理方式并不是从根源上解决，而是利用了规则的定义绕开了而已

8.0以前的不会有这样的一些规则，归根到底还是treble工程

#### 4.2.2.1testNeverallowRules184 解决方案

```bash
<Test result="fail" name="testNeverallowRules184">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {      domain      -coredomain      -appdomain      -binder_in_vendor_violators     } binder_device:chr_file { { getattr open read ioctl lock map } { open append write lock map } };
libsepol.report_failure: neverallow violated by allow testA binder_device:chr_file { ioctl read write open };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

利用属性 `binder_in_vendor_violators` 规避这个问题

```te
type testA, domain;
修改为
type testA, domain, binder_in_vendor_violators;
```

原因很简单，`binder_in_vendor_violators`是一个属性，查看上面的neverallow规则就可以明白

#### 4.2.2.3testNeverallowRules185 解决方案

```bash
<Test result="fail" name="testNeverallowRules185">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {      domain      -coredomain      -appdomain       -binder_in_vendor_violators     } service_manager_type:service_manager find;
libsepol.report_failure: neverallow violated by allow testA activity_service:service_manager { find };
libsepol.check_assertions: 1 neverallow failures occurred
```

和上面的一样，利用属性 `binder_in_vendor_violators` 规避这个问题

```te
type testA, domain;
修改为
type testA, domain, binder_in_vendor_violators;
```

这些属性之所以会存在，说明了Google并没有做到最完美的system和vendor分离，所有有这么一个像是后门的东西

还有一些：

- vendor_executes_system_violators
- data_between_core_and_vendor_violators
- system_writes_vendor_properties_violators
  

### 4.2.3vendor的脚本，在有权限的目录读写，不要操作system的

这类问题很明显，system和vendor分离之后才有的，没有啥好讲的，多看看报错信息和domain.te里面的注释，就能搞明白了

#### 4.2.3.1testNeverallowRules217 解决方案

```bash
<Test result="fail" name="testNeverallowRules217">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {          domain          -coredomain          -appdomain          -vendor_executes_system_violators          -vendor_init      } {          exec_type          -vendor_file_type          -crash_dump_exec          -netutils_wrapper_exec      }:file { entrypoint execute execute_no_trans };
libsepol.report_failure: neverallow violated by allow testA shell_exec:file { execute };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

首先，如果脚本在/vendor/bin下面，先将sh改过来

```shell
#!/system/bin/sh
 
修改为：
 
#!/vendor/bin/sh
```

以上能解决大部分问题，但是如果有些命令只有在/system/bin下面才有的，那这个时候就没有办法了，这个shell_exec:file 一定要加

此时，只能通过vendor_executes_system_violators 来规避问题了

```shell
type testA, domain, binder_in_vendor_violators;
 
修改为：
 
type testA, domain, binder_in_vendor_violators, vendor_executes_system_violators;
```

#### 4.2.3.2testNeverallowRules203 解决方案

```bash
<Test result="fail" name="testNeverallowRules203">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {      domain      -appdomain       -coredomain      -data_between_core_and_vendor_violators       -vendor_init    } {      core_data_file_type                        -zoneinfo_data_file    }:{ { chr_file blk_file } { file lnk_file sock_file fifo_file } } ~{ append getattr ioctl read write map };
libsepol.report_failure: neverallow violated by allow testA system_data_file:file { open };
libsepol.check_assertions: 1 neverallow failures occurred
 
</StackTrace>
        </Failure>
      </Test>
```

仔细看domain.te里面的注释，就能明白了

```te
# vendor domains may only access files in /data/vendor, never core_data_file_types
```

把脚本里面的路径修改一下

```shell
TEMP_PATH="/data/test"
 
修改为：
 
TEMP_PATH="/data/vendor/test"
```

当然，有些neverallow规则还是可以利用属性绕过的（像data_between_core_and_vendor_violators），不过能正面解决就直接解决了

### 4.2.4dac_override 问题

这个在前面以及提到过了，阅读一下参考文章就能明白了

解决这个问题从来不是添加SELinux处理的，必须要去解决unix user/group/others权限问题

### 4.2.5读写/data数据

这里讨论下另一个解决方案：

```bash
<Test result="fail" name="testNeverallowRules237">
        <Failure message="junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:">
          <StackTrace>junit.framework.AssertionFailedError: The following errors were encountered when validating the SELinuxneverallow rule:
neverallow {   domain   -system_server   -system_app   -init   -installd    -vold_prepare_subdirs     } system_data_file:file { append create link unlink relabelfrom rename setattr write };
libsepol.report_failure: neverallow violated by allow testA system_data_file:file { write create setattr unlink };
libsepol.check_assertions: 1 neverallow failures occurred
```

再次看到这个很好奇，不是已经在上面的处理方法中解决了吗，修改读写的路径，改成有权限的地方就行了

这里再提供另一种解决思路，对于那些我们不方便修改代码的，这是一个福音

以下面的这个权限为例：

```te
allow testA system_data_file:file { write create unlink };
```

这个权限添加的目的是为了读写、删除这个文件： /data/test.txt

问题就在于进程testA他没有直接权限读写啊

按照以前的思路：干掉规则 -testA

```te
neverallow {
  domain
  -system_server
  -system_app
  -init
  -installd # for relabelfrom and unlink, check for this in explicit neverallow
  -vold_prepare_subdirs # For unlink
  -testA
  with_asan(`-asan_extract')
} system_data_file:file no_w_file_perms;
```

经过上面的处理经验，一个简单的方案出来了，把/data/test.txt 修改成/data/vendor/test.txt

然后权限拉满，就可以正常创建和删除了

```te
allow testA vendor_data_file:dir create_dir_perms;
allow testA vendor_data_file:file create_file_perms;
```

**下面讨论的是另一种解决方案：**

data根目录，它所对应的安全上下文为：u:object_r:system_data_file:s0

```shell
drwxrwx--x  40 system system u:object_r:system_data_file:s0            4096 2021-06-10 16:09 data
```

如果创建一个文件 /data/test.txt， 它的安全上下文是继承于父目录的，也就是u:object_r:system_data_file:s0

所以在上面的报错会看到是对system_data_file类型的目录缺少写的权限

但是，**如果有一种方式可以实现创建文件或者子目录的时候，不要继承父目录的scontext，改成一个拥有权限的type，那不就可以了嘛？**

没错，还真的有这种操作，这个叫做type transition ， **类型转移**

在前面的基础文档里面已经有介绍了，我们直切主题，用宏 file_type_auto_trans

```te
file_type_auto_trans(testA, system_data_file, vendor_data_file)
```

```te
#####################################
# file_type_auto_trans(domain, dir_type, file_type)
# Automatically label new files with file_type when
# they are created by domain in directories labeled dir_type.
#
define(`file_type_auto_trans', `
# Allow the necessary permissions.
file_type_trans($1, $2, $3)
# Make the transition occur by default.
type_transition $1 $2:dir $3;
type_transition $1 $2:notdevfile_class_set $3;
')
 
意思就是，testA（第一个参数domain）这个type的进程，在system_data_file（第二个参数dir_type）类型的目录下面，创建文件时，指定它的type为vendor_data_file（第三个参数file_type）
```

同样的，这里同样要给vendor_data_file加权限

这一句将会发生如下变化：

```bash
picasso:/ # ls -lZ /data/test.txt
-rw------- 1 root system u:object_r:system_data_file:s0 5320 2021-06-10 17:06 /data/test.txt
 
变成
 
picasso:/ # ls -lZ /data/test.txt
-rw------- 1 root system u:object_r:vendor_data_file:s0 5320 2021-06-10 17:06 /data/test.txt
```

接着还有删除文件时需要下面的权限，这个正常添加就行了，下面的权限不会违反neverallow规则

```bash
[   67.886702@2] type=1400 audit(1623315964.145:18): avc: denied { remove_name } for pid=3142 comm="HwBinder:3142_2" name="test.txt" dev="mmcblk0p21" ino=1263 scontext=u:r:testA:s0 tcontext=u:object_r:system_data_file:s0 tclass=dir permissive=0
```

这种方式不一定适用于app，因为app里面还有其他neverallow规则会限制

它不是针对某个进程，而是针对的是一个域，这样会导致所有在这个域里面的进程都会创建出新的type的文件

## 4.3一些思考

### 4.3.1修改属性的neverallow规则，还能测试通过

在/android/system/sepolicy/public/property.te 这一段，本来以为会有报错的，因为违反了neverallow规则

```te
compatible_property_only(`
  # Neverallow coredomain to set vendor properties
  neverallow {
    coredomain
    -init
    -system_writes_vendor_properties_violators
    -system_server
    -platform_app
    -testA
  } {
    property_type
    -audio_prop
    -bluetooth_a2dp_offload_prop
    -bluetooth_prop
    -bootloader_boot_reason_prop
......
    -test_prop
  }:property_service set;
')
```

但实际上并没有报错，找到CTS测试的源码：

```c++
@RestrictedBuildTest
  public void testNeverallowRules436() throws Exception {
    String neverallowRule = "neverallow {      coredomain      -init      -system_writes_vendor_properties_violators    } {      property_type      -audio_prop      -bluetooth_a2dp_offload_prop      -bluetooth_prop      -bootloader_boot_reason_prop      -boottime_prop      -config_prop      -cppreopt_prop      -ctl_bootanim_prop      -ctl_bugreport_prop      -ctl_picasso_prop      -ctl_default_prop      -ctl_dumpstate_prop      -ctl_fuse_prop      -ctl_interface_restart_prop      -ctl_interface_start_prop      -ctl_interface_stop_prop      -ctl_mdnsd_prop      -ctl_restart_prop      -ctl_rildaemon_prop      -ctl_sigstop_prop      -ctl_start_prop      -ctl_stop_prop      -dalvik_prop      -debug_prop      -debuggerd_prop      -default_prop      -device_logging_prop      -dhcp_prop      -dumpstate_options_prop      -dumpstate_prop      -exported2_config_prop      -exported2_default_prop      -exported2_radio_prop      -exported2_system_prop      -exported2_vold_prop      -exported3_default_prop      -exported3_radio_prop      -exported3_system_prop      -exported_bluetooth_prop      -exported_config_prop      -exported_dalvik_prop      -exported_default_prop      -exported_dumpstate_prop      -exported_ffs_prop      -exported_fingerprint_prop      -exported_overlay_prop      -exported_pm_prop      -exported_radio_prop      -exported_secure_prop      -exported_system_prop      -exported_system_radio_prop      -exported_vold_prop      -exported_wifi_prop      -extended_core_property_type      -ffs_prop      -fingerprint_prop      -firstboot_prop      -hwservicemanager_prop      -last_boot_reason_prop      -log_prop      -log_tag_prop      -logd_prop      -logpersistd_logging_prop      -lowpan_prop      -mmc_prop      -net_dns_prop      -net_radio_prop      -netd_stable_secret_prop      -nfc_prop      -overlay_prop      -pan_result_prop      -persist_debug_prop      -persistent_properties_ready_prop      -pm_prop      -powerctl_prop      -radio_prop      -restorecon_prop      -safemode_prop      -serialno_prop      -shell_prop      -system_boot_reason_prop      -system_prop      -system_radio_prop      -test_boot_reason_prop      -traced_enabled_prop      -vendor_default_prop      -vendor_security_patch_level_prop      -vold_prop      -wifi_log_prop      -wifi_prop    }:property_service set;";
    boolean fullTrebleOnly = false;
    boolean compatiblePropertyOnly = true;
 
    if ((fullTrebleOnly) && (!isFullTrebleDevice()))
    {
      return;
    }
    if ((compatiblePropertyOnly) && (!isCompatiblePropertyEnforcedDevice()))
    {
      return;
    }
 
    File policyFile = (isSepolicySplit()) && (this.mVendorSepolicyVersion < 28) ?
      this.deviceSystemPolicyFile :
      this.devicePolicyFile;
 
    ProcessBuilder pb = new ProcessBuilder(new String[] { this.sepolicyAnalyze.getAbsolutePath(), policyFile
      .getAbsolutePath(), "neverallow", "-w", "-n", neverallowRule });
 
    pb.redirectOutput(ProcessBuilder.Redirect.PIPE);
    pb.redirectErrorStream(true);
    Process p = pb.start();
    p.waitFor();
    BufferedReader result = new BufferedReader(new InputStreamReader(p.getInputStream()));
 
    StringBuilder errorString = new StringBuilder();
    String line;
    while ((line = result.readLine()) != null) {
      errorString.append(line);
      errorString.append("\n");
    }
    assertTrue("The following errors were encountered when validating the SELinuxneverallow rule:\n" + neverallowRule + "\n" + errorString, errorString
      .length() == 0);
  }
```

代码是在这里被return了，也就是没有做检查，所以测试结果是pass的

```c++
if ((compatiblePropertyOnly) && (!isCompatiblePropertyEnforcedDevice()))
{
  return;
}
 
 
//isCompatiblePropertyEnforcedDevice拿的是系统属性，从板子里面看了看，这个值是false
public static boolean isCompatiblePropertyEnforcedDevice(ITestDevice device)
    throws DeviceNotAvailableException
{
  return PropertyUtil.propertyEquals(device, "ro.actionable_compatible_property.enabled", "true");
}
```

简单追了一下这里面的流程，首先compatible_property_only是一个宏，定义如下

```te
# Compatible property only
# SELinux rules which apply only to devices with compatible property
#
define(`compatible_property_only', ifelse(target_compatible_property, `true', $1,
ifelse(target_compatible_property, `cts',
# BEGIN_COMPATIBLE_PROPERTY_ONLY -- this marker is used by CTS -- do not modify
$1
# END_COMPATIBLE_PROPERTY_ONLY -- this marker is used by CTS -- do not modify
, )))
```

值来源于target_compatible_property，这个值是前面提到的transform-policy-to-conf里面的变量传递进来的

```makefile
-D target_compatible_property=$(PRIVATE_COMPATIBLE_PROPERTY)
```

继续，值是从PRODUCT_COMPATIBLE_PROPERTY来的

```makefile
PRIVATE_COMPATIBLE_PROPERTY := $(PRODUCT_COMPATIBLE_PROPERTY)
```

往下

```makefile
#android/build/make/core/config.mk
 
# Boolean variable determining if the whitelist for compatible properties is enabled
PRODUCT_COMPATIBLE_PROPERTY := false
ifneq ($(PRODUCT_COMPATIBLE_PROPERTY_OVERRIDE),)
  PRODUCT_COMPATIBLE_PROPERTY := $(PRODUCT_COMPATIBLE_PROPERTY_OVERRIDE)
else ifeq ($(PRODUCT_SHIPPING_API_LEVEL),)
  #$(warning no product shipping level defined)
else ifneq ($(call math_lt,27,$(PRODUCT_SHIPPING_API_LEVEL)),)
  PRODUCT_COMPATIBLE_PROPERTY := true
endif
```

PRODUCT_COMPATIBLE_PROPERTY 的值默认为false，如果有PRODUCT_COMPATIBLE_PROPERTY_OVERRIDE，则用这个

然后是PRODUCT_SHIPPING_API_LEVEL

也就是说，PRODUCT_COMPATIBLE_PROPERTY 最后的值为false

所以前面的neverallow规则不会生效

再看看属性`ro.actionable_compatible_property.enabled`是怎么来的？

```makefile
# Sets ro.actionable_compatible_property.enabled to know on runtime whether the whitelist
# of actionable compatible properties is enabled or not.
ifeq ($(PRODUCT_ACTIONABLE_COMPATIBLE_PROPERTY_DISABLE),true)
ADDITIONAL_DEFAULT_PROPERTIES += ro.actionable_compatible_property.enabled=false
else
ADDITIONAL_DEFAULT_PROPERTIES += ro.actionable_compatible_property.enabled=${PRODUCT_COMPATIBLE_PROPERTY}
endif
```

`PRODUCT_ACTIONABLE_COMPATIBLE_PROPERTY_DISABLE` 是空的没有定义，也是最后生成的值是false

追查了代码，发现原来以前BoardConfig.mk里面是有定义的

```c++
PRODUCT_SHIPPING_API_LEVEL := 28
```

但是后面居然把这个去掉了？

PRODUCT_SHIPPING_API_LEVEL 这个会影响到一些neverallow规则

还会影响到这个属性`ro.product.first_api_level`的生成

### 4.3.2抽离业务场景需求的公共权限

或许我们会有一个疑问，就是为什么不把所有自己的脚本都用同一个安全上下文呢？

这的确是可行的方案，可以更加便捷的完成功能需求，但是从权限管控的初衷来看，分开了之后能够达到权限最小化；

在上面的思路基础上，引发了另一个想法：

定义一个自定义的属性，然后添加脚本的时候，关联上这个属性，就拥有了对应的权限了

比如：

```te
attribute testdomain;
```

testdomain.te里面定义一些公共的权限，比如：

```te
allow testdomain picasso_device:chr_file rw_file_perms;
allow testdomain vendor_toolbox_exec:file { execute_no_trans };
allow testdomain vendor_data_file:dir create_dir_perms;
allow testdomain vendor_data_file:file create_file_perms;
```

然后在对应的type中关联属性

```te
type testA, domain, testdomain;
```

**当然目前属于想法阶段，还没有具体实施，理论上可行**

### 4.3.3三方apk的权限

对于第三方apk的权限问题，比如：

目前这个在系统端没有找到解决方案，只能从apk实现上入手了

```bash
[   97.450511@1] type=1400 audit(1623116538.872:18): avc: denied { create } for pid=3953 comm="testB" name="shpex98m.tmp" scontext=u:r:testB:s0 tcontext=u:object_r:system_data_file:s0 tclass=file permissive=0
```

这个是绕不过neverallow规则的

```bash
[   45.533219@3] type=1400 audit(1623985073.895:7): avc: denied { execute } for pid=4052 comm="bootloader" path="/data/data/com.xxxx.xxx/files/test/test.so" dev="mmcblk0p21" ino=1085 scontext=u:r:system_app:s0 tcontext=u:object_r:system_app_data_file:s0 tclass=file permissive=0
```

到这里，基本上就介绍完了，总结来说，多看看代码，报错信息以及te文件里面的注释，能学到很多东西

能不动到原生的部分，就不要修改

# 5名词解释

| 英文名次 |                             含义                             |
| :-------: | :----------------------------------------------------------: |
|  `AVC`   | Access Vector Cache访问向量缓存<br />用于当一个subject要访问object时，selinux执行检查AVC中，subject和object的权限被缓存 |
|  `CTS`   | Compatibility Test Suite（兼容性测试）<br />Android设备只有满足规定并且通过CTS，才能获得Android的商标和享受Android Market的权限 |
|  `DAC`   |        DAC：Discretionary Access Control 自主访问控制        |
|  `LSM`   | Linux Security Modules (LSM) 是一种 Linux 内核子系统<br />旨在将内核以模块形式集成到各种安全模块中 |
|  `MAC`   |          MAC：Mandatory Access Control 强制访问控制          |
|  `MLS`   | Multi-Level Security,多层次的安全<br />系统的进程和文件进行了分级，不同级别的资源需要对应级别的进程才能访问 |
| `SElinux` | SELinux(Security-Enhanced Linux) 安全增强型linux子系统<br />在这种访问控制体系的限制下，进程只能访问那些在他的任务中所需要文件 |
|   `TE`   | TE （Type Enforcement）强制类型<br />基本管理单位是TEAC（Type Enforcement Accesc Control），源码中存在TE文件，这些文件是用于制定策略规则的 |

# 转载说明

本文非原创，主要是转载**headwind**的文章内容，在加上稍微的修改。

# 参考

[[1] nianxing123, 【安卓】SELinux走过的坑, 2022.](https://juejin.cn/post/7126884870999474184)

[[2] 沉默的过客, SELinux策略语言–客体类别和许可, 2017.](https://blog.csdn.net/u014135607/article/details/78697587)

[[3] 罗升阳, SEAndroid安全机制框架分析, 2014.](https://blog.csdn.net/Luoshengyang/article/details/37613135)

[[4] headwind_, Android P SELinux (一) 基础概念, 2021.](https://headwind.blog.csdn.net/article/details/119704755)

[[5] headwind_, Android P SELinux (二) 开机初始化与策略文件编译过程, 2021.](https://headwind.blog.csdn.net/article/details/119707417)

[[6] headwind_, Android P SELinux (三) 权限检查原理与调试, 2022.](https://headwind.blog.csdn.net/article/details/119709982)

[[7] headwind_, Android P SELinux (四) CTS neverallow处理总结, 2022.](https://headwind.blog.csdn.net/article/details/119711869)

[[8] IT先森, Android SELinux开发入门指南之SELinux基础知识, 2020.](https://blog.csdn.net/tkwxty/article/details/104538447)

[[9] IT先森, 正确姿势临时和永久开启关闭Android的SELinux, 2020.](https://blog.csdn.net/tkwxty/article/details/103938287)

[[10] 冯疯子, 谈谈Android 安全策略SElinux, 2020.](https://blog.csdn.net/fhaitao900310/article/details/108868103)

[[11] 五菱宏光8PLUS, Android 添加 SELinux权限, 2022.](https://juejin.cn/post/7051154021200953357)

[[12] Android开发实践, Android安全策略Selinux小结, 2019.](https://juejin.cn/post/6844903813195759624)

[[13] 砸漏, 详解Android Selinux 权限及问题, 2020.](https://cloud.tencent.com/developer/article/1730613)

[[14] sheldon_blogs, Android : SELinux 简析&修改 , 2017.](https://www.cnblogs.com/blogs-of-lxl/p/7515023.html)

[[15] 程序员大本营, SELinux简介, 2018.](https://www.pianshen.com/article/1843146239/)

[[16] markvz, SELinux简介, 2018.](https://blog.csdn.net/weixin_42082222/article/details/85239018)

[[17] 安德路, android sepolicy 最新小结, 2018.](https://blog.csdn.net/ch853199769/article/details/82498725)

[[18] miroku_it, SELinux策略配置语言, 2009.]([miroku_it](https://blog.csdn.net/miroku_it))

[[19] 岁月斑驳7, android 8.1 安全机制 — SEAndroid & SELinux, 2018.](https://blog.csdn.net/qq_19923217/article/details/81240027)

[[20] _dowork, SELinux audit2allow命令使用, 2019.](https://blog.csdn.net/q1183345443/article/details/90438283)

[[21] Donald_Zhuang, selinux dac_override/dac_read_search问题处理思路, 2020.](https://blog.csdn.net/Donald_Zhuang/article/details/108786482)

[[22] linux内核之旅, Linux多安全策略和动态安全策略框架模块代码分析报告(14), 2017.](https://m.sohu.com/a/141709117_467784/?pvid=000115_3w_a)

[[23] Boyliang1987, SEAndroid策略, 2013.](https://blog.csdn.net/l173864930/article/details/17194899)

[[24] KINGYT, 精致全景图 | linux内核输出的日志去哪里了, 2021.](https://mp.weixin.qq.com/s/mdDLw6AIp9ws9LTaHg64pg)

[[25] 小武, kmsg日志的存储与读取, 2021.](https://zhuanlan.zhihu.com/p/406631271?utm_id=0)

[[26] jimbo_lee, 常用正则表达式大全 -- 正则表达式基本语法, 2011.](https://blog.csdn.net/jimbo_lee/article/details/51593513)

[[27] Sherry Baron, Android.mk(持续补充中哦！！！), 2021.](https://blog.csdn.net/qq_41663117/article/details/117411597)

[[28] 魏波., Makefile_03：Makefile介绍（作用、例子、原理）, 2022.](https://weibo01.blog.csdn.net/article/details/115582646)

[[29] kc专栏, Android.mk 小细节（LOCAL_CFLAGS 、BUILD_PREBUILT）, 2015.](https://blog.csdn.net/kc58236582/article/details/49795865)

[[30] hjxu2016, Makefile入门二、理解$@、$^和$<, 2019.](https://blog.csdn.net/hjxu2016/article/details/101699484)

[[31] 「已注销」, Android CTS中neverallow规则生成过程, 2019.](https://blog.csdn.net/liwugang43210/article/details/103754878)

[[32] 安德路, Android sepolicy简要记, 2018.](https://blog.csdn.net/ch853199769/article/details/82501078)

[[33] 安德路, android sepolicy 最新小结, 2018.](https://blog.csdn.net/ch853199769/article/details/82498725)

[[34] 铁桶小分队, SEAndroid的MLS相关知识以及配置方法, 2019.](https://blog.csdn.net/keheinash/article/details/99692988)

[[35] wsrspirit, 从【SELINUX】策略中学习【LSM】编写规则, 2014.](https://blog.csdn.net/WSRspirit/article/details/26398883)

[[36] CSlunatic, Linux Security模块, 2014.](https://www.cnblogs.com/cslunatic/p/3709356.html)

[[37] MrHare, SEAndroid kernel层源码解析1——从hook点到策略点, 2017.](https://blog.csdn.net/mrhare/article/details/71215391)

[[38] MrHare, SEAndroid kernel 源码解析2--策略执行, 2017.](https://blog.csdn.net/MrHare/article/details/71405139)

[[39] InsightAndroid, selinux常见neverallow项解决方法与常用命令, 2020.](https://blog.csdn.net/k663514387/article/details/107983037)

# 其他参考

## 1**学习本文内容最好先阅读下面SElinux文章**

[[1] 阿拉神农, 深入理解SELinux SEAndroid（第一部分）, 2014.](https://blog.csdn.net/innost/article/details/19299937)

[[2] 阿拉神农, 深入理解SELinux SEAndroid之二, 2014.](https://blog.csdn.net/innost/article/details/19641487)

[[3] 阿拉神农, 深入理解SELinux SEAndroid（最后部分）, 2014.](https://blog.csdn.net/innost/article/details/19767621)

[[4] sven, SELinux 安全上下文, 2015.](http://lishiwen4.github.io/android/selinux-security-context)

[[5] bruk_spp, selinux源码分析, 2020.](https://blog.csdn.net/bruk_spp/article/details/107283935)

[[6] 岁月斑驳7, android 8.1 安全机制 — SEAndroid & SELinux, 2018.](https://blog.csdn.net/qq_19923217/article/details/81240027)

[[7] 罗升阳, SEAndroid安全机制对Binder IPC的保护分析, 2014.](https://blog.csdn.net/Luoshengyang/article/details/38326729)

[[8] 罗升阳, SEAndroid安全机制对Android属性访问的保护分析, 2014.](https://blog.csdn.net/Luoshengyang/article/details/38102011)

[[9] 罗升阳, SEAndroid安全机制中的进程安全上下文关联分析, 2014.](https://blog.csdn.net/luoshengyang/article/details/38054645)

[[10] 罗升阳, SEAndroid安全机制中的文件安全上下文关联分析, 2014.](https://blog.csdn.net/Luoshengyang/article/details/37749383)

[[11] 罗升阳, SEAndroid安全机制框架分析, 2014.](https://blog.csdn.net/Luoshengyang/article/details/37613135)

[[12] 罗升阳, SEAndroid安全机制简要介绍和学习计划, 2014.](https://blog.csdn.net/Luoshengyang/article/details/35392905)

## 2**文章通用参考汇总**

[[1] 高桐@BILL, Android8.0 SELinux详解, 2018.](https://blog.csdn.net/huangyabin001/article/details/79290382)

[[2] 戈壁老王, Android Treble 简介, 2020.](https://segmentfault.com/a/1190000021550665)

[[3] Gunder, SeLinux权限配置心得总结, 2021.](https://blog.csdn.net/u013357557/article/details/113893751)

[[4] 流浪_归家, SELinux avc 权限问题修改, 2020.](https://www.cnblogs.com/fanglongxiang/p/13306088.html)

[[5] 缥缈孤鸿影_love, Android SELinux avc dennied权限问题解决方法, 2017.](https://blog.csdn.net/tung214/article/details/72734086)

[[6] 岁月斑驳7, Android 8.1 非系统进程设置系统域属性问题, 2018.](https://blog.csdn.net/qq_19923217/article/details/83820902)

[[7] 李晓刚, Android Selinux, 2019.](https://lixiaogang03.github.io/2019/03/28/Android-Selinux/)

[[8] Andy.Lee , 详解Android Selinux 权限及问题, 2017.](https://www.jb51.net/article/128214.htm)

[[9] 布道师Peter, Selinux 权限问题解决方案, 2020.](https://peter.blog.csdn.net/article/details/111570144)

[[10] DJLZPP, 访问文件的SELINUX权限添加, 2020.](https://blog.csdn.net/qq_34211365/article/details/106056873)

[[11] 花一样的阿衰, SElinux权限配置, 2020.](https://blog.csdn.net/qq_33242956/article/details/101421189)

[[12] 飞车侠, i.MX8 系列 | Android APP 访问硬件驱动可以这样做！, 2020.](https://www.wpgdadatong.com/cn/blog/detail?BID=B2206)

[[13] 秋少, Android系统开发入门-7.添加java层系统服务, 2019.](http://qiushao.net/2019/12/20/Android系统开发入门/7-添加java系统服务/)

[[14] gaon5, Android : 为系统服务添加 SELinux 权限 (Android 9.0), 2018.](http://www.mamicode.com/info-detail-2532042.html)

[[15] 李晓刚, Android R Selinux, 2020.](https://lixiaogang03.github.io/2020/11/12/Android-R-Selinux/)

[[16] 守望尼罗河畔的初心, Android7.1 Selinux使用, 2017.](https://blog.csdn.net/zmnqazqaz/article/details/76130202)

[[17] CedarDiao, Android9 Sepolicy规则基础 - MTK平台, 2020.](https://blog.csdn.net/diaoxuesong/article/details/104572033)

[[18] 锄禾豆, Android 9 SELinux, 2018.](https://www.jianshu.com/p/e95cd0c17adc)

[[19] JasonWan, SELinux在Android中的应用, 2019.](http://sniffer.site/2019/12/07/Selinux在Android中的应用/)

[[20] gandalf, Android（十）SEAndroid初探, 2019.](http://www.gandalf.site/2019/02/androidseandroid.html)

[[21] sheldon, Android : SELinux 简析&修改, 2017.](https://www.cnblogs.com/blogs-of-lxl/p/7515023.html)

[[22] werther zhang, SEAndroid规则介绍, 2017.](https://wertherzhang.com/seandroid规则介绍/)

[[23] xuefeng_apple, android启动脚本-selinux(实践), 2020.](android启动脚本-selinux(实践))

[[24] 恶魔殿下_HIM, Android的安全机制（SEANDROID）, 2016.](https://www.jianshu.com/p/a3572eee341c)

[[25] kc专栏, SeLinux语法规则, 2015.](https://blog.csdn.net/kc58236582/article/details/46723333)

[[26] 沉默的过客, SELinux策略语言–客体类别和许可, 2017.](https://blog.csdn.net/u014135607/article/details/78697587)

[[27] 林多, SELinux MAC安全机制简介, 2019.](https://blog.csdn.net/zxc024000/article/details/88389657)

[[28] SinoTech, Android R Selinux, 2018.](为 Android 8.0 添加开机启动脚本)

[[29] MickCaptain, Android 8.1添加系统服务,sepolicy相关配置, 2019.](https://www.jianshu.com/p/5fe2e6d730b8?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[[30] SinoTech, 在 Android 8.0 中绕过 hwbinder 实现跨模块对 audio HAL 调用, 2017.](https://zhuanlan.zhihu.com/p/30406415)

[[31] xbalien, SEAndroid策略, 2014.](https://blog.csdn.net/xbalien29/article/details/19505575)

[[32] tww85, 对Android 平台下SElinux的理解及遇到过的相关问题解决方法总结, 2016.](https://blog.csdn.net/tww85/article/details/52451606)

[[33] wydong, SELinux与SEAndroid, 2018.](https://segmentfault.com/a/1190000012780150)

[[34] 段小苏学习之路, Android有关selinux详解, 2019.](https://blog.csdn.net/duan_xiaosu/article/details/103251903)

[[35] 不上班行不行, SEAndroid系统架构总体分析, 2018.](https://www.cnblogs.com/startkey/articles/9931669.html)

[[36] 亚洲第一蓝胖子, android中SELINUX规则分析和语法简介, 2018.](https://blog.csdn.net/QWE123ZXCZ/article/details/84955034)

[[37] Zpeg, selinux - Android编写sepolicy, 2020.](https://blog.csdn.net/qq_28351465/article/details/107431232)

[[39] 光明之门, SELinux&SEAndroid简介, 2017.](https://blog.csdn.net/doubleghost2010/article/details/70238037)

[[39] IT先森,  Android SELinux开发入门指南之正确姿势解决访问data目录权限问题, 2020.](https://blog.csdn.net/tkwxty/article/details/104039681)

[[40] 渴望成长的菜鸟, Android O selinux违反Neverallow解决办法, 2018.](https://blog.csdn.net/yxw0609131056/article/details/79784490)

[[41] kc专栏, Android6.0 selinux没有对某个文件的权限（又neverAllow）处理方法, 2016.](https://blog.csdn.net/kc58236582/article/details/50537782)

[[42] 维基百科, SELinux/Quick introduction, 2021.](https://wiki.gentoo.org/wiki/SELinux/Quick_introduction)

[[43] foxleezh, (连载)Android 8.0 : 系统启动流程之init进程(一), 2018.](https://juejin.cn/post/6844903568747544584)

[[44] xiaozhuangzi, selinux label的初始化过程, 2019.](https://blog.51cto.com/u_2559640/2410131)

[[45] Ryan, Selinux中的APP分类, 2018.](http://huzhengyu.com/2018/06/08/APP-Category-In-Selinux/)

[[46] zhudaozhuan, SELinux app权限配置, 2016.](https://blog.csdn.net/zhudaozhuan/article/details/50964832)

[[47] 砸漏, Android 7.0 SEAndroid app权限配置方法, 2020.](https://cloud.tencent.com/developer/article/1742749)

[[48] 子曰小玖, Selinux梳理总结, 2019.](https://blog.csdn.net/wxh0000mm/article/details/94456071)

[[49] 夜风~, linux内核中likely与unlikely, 2018.](https://blog.csdn.net/u014470361/article/details/81193023)

[[50] Decisiveness, Linux内核中的IS_ERR()、PTR_ERR()和ERR_PTR(), 2015.](https://blog.csdn.net/Decisiveness/article/details/45049875)

[[51] 扬眉, Flex(scanner)/Bison(parser)词法语法分析工作原理, 2022.](https://zhuanlan.zhihu.com/p/120812270)

[[52] up哥小号, Linux 编程中的API函数和系统调用的关系, 2012.](http://blog.chinaunix.net/uid-28362602-id-3424404.html)

[[53] wenyubo_2008, Linux 中open系统调用实现原理, 2012.](http://blog.chinaunix.net/uid-25968088-id-3426026.html)

[[54] feixin620, linux syscall 详解, 2017.](https://blog.csdn.net/feixin620/article/details/78416560)

[[55] Xiaowei He, Linux 进程安全上下文 struct cred, 2022.](http://onestraw.github.io/linux/linux-struct-cred/)

[[56] Morphad, linux cred管理, 2013.](https://blog.csdn.net/Morphad/article/details/9089601)

[[57] 庾志辉, openVswitch（OVS）源代码分析之工作流程（哈希桶结构体的解释）, 2014.](https://blog.csdn.net/YuZhiHui_No1/article/details/39939241)

[[58] 庾志辉, openVswitch（OVS）源代码的分析技巧（哈希桶结构体为例）, 2014.](https://blog.csdn.net/yuzhihui_no1/article/details/39558815)

[[59] Happybevis, 神奇的Selinux Restore Rule, 2018.](https://happybevis.github.io/2018/05/02/The-Magic-Selinux-Restore-Rule/)



## 3《Linux强制访问控制机制模块代码分析报告》

[[1] linux内核之旅, Linux强制访问控制机制模块代码分析报告（一）, 2017.](https://www.sohu.com/a/135967369_467784)

[[2] linux内核之旅, Linux强制访问控制机制模块代码分析报告（二）, 2017.](https://www.sohu.com/a/136011893_467784)

[[3] linux内核之旅, Linux强制访问控制机制模块代码分析报告（三）, 2017.](https://www.sohu.com/a/136288365_467784)

[[4] linux内核之旅, Linux强制访问控制机制模块代码分析报告（四）, 2017.](https://www.sohu.com/a/136482060_467784)

[[5] linux内核之旅, Linux强制访问控制机制模块代码分析报告（五）, 2017.](https://www.sohu.com/a/136639046_467784)

[[6] linux内核之旅, Linux强制访问控制机制模块代码分析报告（六）, 2017.](http://www.safebase.cn/article-231024-1.html)

[[7] linux内核之旅, Linux强制访问控制机制模块代码分析报告（七）, 2017.](https://www.sohu.com/a/137347468_467784)

[[8] linux内核之旅, Linux强制访问控制机制模块代码分析报告（八）, 2017.](https://www.sohu.com/a/137512129_467784)

[[9] linux内核之旅, Linux强制访问控制机制模块代码分析报告（九）, 2017.](https://www.sohu.com/a/137888523_467784)

[[10] linux内核之旅, Linux强制访问控制机制模块代码分析报告（十）, 2017.](https://www.sohu.com/a/137888531_467784)

[[11] linux内核之旅, Linux强制访问控制机制模块代码分析报告（十一）, 2017.](https://www.sohu.com/a/138185447_467784)



## 4《Linux多安全策略和动态安全策 略框架模块代码分析报告》

[[1] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(1) , 2017.](https://www.sohu.com/a/138580970_467784)

[[2] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(2) , 2017.](http://www.safebase.cn/article-231245-1.html)

[[3] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(3) , 2017.](https://www.sohu.com/a/138804853_467784)

[[4] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(4) , 2017.](https://www.sohu.com/a/138976820_467784)

[[5] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(5) , 2017.](http://www.voidcn.com/article/p-sfquuwtm-wk.html)

[[6] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(6) , 2017.](https://www.sohu.com/a/139413489_467784)

[[7] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(7) , 2017.](https://www.sohu.com/a/139709740_467784)

[[8] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(8) , 2017.](https://www.sohu.com/a/139973785_467784)

[[9] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(9) , 2017.](http://www.voidcn.com/article/p-xhablags-wk.html)

[[10] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(10) , 2017.](http://www.voidcn.com/article/p-uwdtubpm-wk.html)

[[11] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(11) , 2017.](https://www.sohu.com/a/140884205_467784)

[[12] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(12) , 2017.](http://www.voidcn.com/article/p-tkroeamc-wk.html)

[[13] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(13) , 2017.](https://www.sohu.com/a/141414625_467784)

[[14] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(14) , 2017.](https://www.sohu.com/a/141709117_467784)

[[15] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(15) , 2017.](https://m.sohu.com/a/142034082_467784/)

[[16] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(16) , 2017.](http://www.voidcn.com/article/p-ksphumbr-wk.html)

[[17] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(17) , 2017.](https://www.sohu.com/a/142438688_467784)

[[18] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(18) , 2017.](http://www.voidcn.com/article/p-blaxfzwg-wk.html)

[[19] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(19) , 2017.](http://www.voidcn.com/article/p-gfifderx-wk.html)

[[20] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(20) , 2017.](https://m.sohu.com/a/143323077_467784/)

[[21] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(21) , 2017.](http://www.voidcn.com/article/p-eoijaouz-wk.html)

[[22] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(22) , 2017.](https://www.sohu.com/a/143971000_467784)

[[23] linux内核之旅, Linux多安全策略和动态安全策 略框架模块代码分析报告(23) , 2017.](http://www.voidcn.com/article/p-zfzucopl-wk.html)

## 5fs_use_xattr

[[1] Boyliang1987, SEAndroid策略, 2013.](https://blog.csdn.net/l173864930/article/details/17194899)

[[2] 铁桶小分队, 预设置只读文件系统squashfs上的文件的扩展属性的方法, 2017.](https://blog.csdn.net/keheinash/article/details/78507063)

## 6dac_override

[[1] pwl999, Linux DAC 权限管理详解, 2020.](https://blog.csdn.net/pwl999/article/details/110878563)

[[2] Open Devices Corner, Debugging DAC_OVERRIDE, 2019.](https://sx.ix5.org/info/post/debugging-dac_override/)

[[3] 悟空小饭, Linux Capabilites 机制详细介绍, 2018.](https://gohalo.me/post/linux-capabilities-introduce.html)

[[4] Donald_Zhuang, selinux dac_override/dac_read_search问题处理思路, 2020.](https://blog.csdn.net/Donald_Zhuang/article/details/108786482)

[[5] shichaog, Selinux 权限策略定制, 2016.](https://blog.csdn.net/shichaog/article/details/53728893)

[[6] 大江之舞, SELinux: capabilites{dac_override} 权限, 2015.](https://blog.csdn.net/bhghost/article/details/47312861)



## 7android 用户组

[[1] CarolineVampire, Android用户和用户组的定义, 2014.](https://blog.csdn.net/yangxuehui1990/article/details/41891263)

[[2] `__2017__`, Android添加用户组及自定义App权限, 2016.](https://blog.csdn.net/u013686019/article/details/52753727/)

[[3] Kian_G, Android P SElinux权限调试, 2019.](https://blog.csdn.net/pen_cil/article/details/89434349)