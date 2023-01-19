---
title: "dumpsys简介 以media.camera为例"
date: 2022-04-17
thumbnailImagePosition: left
thumbnailImage: dumpsys/dumpsys_thumb.jpg
coverImage: dumpsys/dumpsys_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- dumpsys
- media.camera
- 2022
- April 
tags:
- java
- C++
- Android
- 源码
showSocial: false
---

dumpsys是一种在Android设备上运行的工具，可提供有关系统服务的信息。在相机开发的过程中，关于一些相机的配置可以直接dumpsyss media.camera来查看。

<!--more-->
# 1dumpsys

dumpsys是一种在Android设备上运行的工具，可提供有关系统服务的信息。可以通过ADB命令行调用dumpsys获取在连接的设备上运行的系统服务的诊断输出。

## 1.1dumpsys 常见指令

### 1.1.1dumpsys --help

dumpsys工具的帮助文本

```shell
dumpsys --help
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_2.png" thumbnail="/dumpsys/dumpsys_2.png" title="">}}

> 对上面的AEGS打印整理
>
> | 选项                | 说明                                                         |
> | ------------------- | ------------------------------------------------------------ |
> | -t timeout          | 指定超时期限（秒）。如果未指定，默认值为 10 秒。             |
> | -T TIMEOUT_MS       | 指定超时期限（毫秒）。如果未指定，默认值为 10 秒。           |
> | --help              | 输出 `dumpsys` 工具的帮助文本                                |
> | -l                  | 输出可与 `dumpsys` 配合使用的系统服务的完整列表              |
> | --skip services     | 指定不希望包含在输出中的 services                            |
> | service [arguments] | 指定希望输出的 service。某些服务可能允许您传递可选 arguments，可以通过将 `-h` 选项与服务名称一起传递来了解这些可选参数<br />dumpsys activity -h |
> | -c                  | 指定某些服务时，附加此选项能以计算机可读的格式输出数据       |
> | -h                  | 对于某些服务，附加此选项可查看该服务的帮助文本和其他选项     |
> | --proto             | 过滤支持以proto格式dump数据的服务，默认是全部打印，proto用于种结构化数据进行序列化 |
> | --priority LEVEL    | 根据指定的优先级过滤服务，LEVEL只能是三种里面选择，CRITICAL \|  HIGH \| NORMAL，分别是紧急，高和普通三种选项 |

###  1.1.2dumpsys -l

输出所有支持dumpsys工具的系统服务的完整列表

```shell
dumpsys -l
```

> 内容比较多，笔者这里划分成了常用服务和不常用服务用于区分。
>
> | 服务名              | 功能                                            |
> | ------------------- | ----------------------------------------------- |
> | activity            | AMS相关信息，可以打印一些activity的信息         |
> | adb                 | adb调试相关，adb的状态和一些信息                |
> | appops              | app使用情况                                     |
> | audio               | 查看声音信息                                    |
> | cpuinfo             | CPU统计相关                                     |
> | input               | IMS，可以打印一些输入相关信息                   |
> | media.audio_flinger | 音频管理器                                      |
> | media.audio_policy  | 音频配置                                        |
> | media.camera        | 相机信息，具体会在后面详述                      |
> | meminfo             | 内存相关，用于查看当前的内存使用情况            |
> | netstats            | 网络接口状态信息，监控TCP/IP网络的情况          |
> | permission          | 权限相关                                        |
> | package             | PMS相关，打印一些package的信息                  |
> | SurfaceFlinger      | 图像相关                                        |
> | window              | WMS相关，主要是布局相关，可以查看栈顶的activity |
> | wifi                | 网络相关，历史连接信心等等                      |
>
>  不常用的服务，除了本身的系统之外，一些芯片产商和手机产商会有自定义的服务
>
> | accessibility        | account               | activity_task             | alarm                  | android.security.keystore                 |
> | -------------------- | --------------------- | ------------------------- | ---------------------- | ----------------------------------------- |
> | app_binding          | app_prediction        | appwidget                 | autofill               | ashmem_device_service                     |
> | backup               | battery               | batteryproperties         | batterystats           | binder_calls_stats                        |
> | biometric            | bluetooth_manager     | bsgamepad                 | bugreport              |                                           |
> | carrier_config       | clipboard             | color_display             | companiondevice        | com.goodix.FingerprintService             |
> | connectivity         | connmetrics           | consumer_ir               | content                | com.qualcomm.location.izat.IzatService    |
> | content_suggestions  | contexthub            | crossprofileapps          | country_detector       |                                           |
> | DockObserver         | dbinfo                | device_config             | device_identifiers     | devicestoragemonitor                      |
> | device_policy        | deviceidle            | diskstats                 | display                | drm.drmManager                            |
> | dpmservice           | dreams                | dropbox                   | dynamic_system         |                                           |
> | ethernet             | extphone              | external_vibrator_service |                        |                                           |
> | fingerprint          |                       |                           |                        |                                           |
> | gfxinfo              | gpu                   | graphicsstats             |                        |                                           |
> | hardware_properties  |                       |                           |                        |                                           |
> | idmap                | imms                  | input_method              | inputflinger           | incidentcompanion                         |
> | ions                 | ipsec                 | ircs                      | isms                   | iphonesubinfo                             |
> | isub                 |                       |                           |                        |                                           |
> | jobscheduler         |                       |                           |                        |                                           |
> | launcherapps         | location              | locationpolicy            | lock_settings          | looper_stats                              |
> | media.aaudio         | media.camera.proxy    | media.drm                 | media.extractor        | media.resource_manager                    |
> | media.metrics        | media.player          | media_projection          | media_router           | media.sound_trigger_hw                    |
> | media_session        | midi                  | mount                     | media_resource_monitor |                                           |
> | miui.fdpp            | miui.face.FaceService | miui.sedc                 | miui.shell             | miui.contentcatcher.ContentCatcherService |
> | miui.whetstone.klo   | miui.whetstone.mcd    | miuiboosterservice        | miui.mqsas.MQSService  | miui.mqsas.IMQSNative                     |
> | miui.whetstone.power | MiuiBackup            | MiuiInit                  |                        |                                           |
> | netd_listener        | netpolicy             | netstats                  | network_score          | network_management                        |
> | network_stack        | network_watchlist     | nfc                       | notification           | network_time_update_service               |
> | oem_lock             | otadexopt             | overlay                   |                        |                                           |
> | ProcessManager       | package_native        | perfshielder              | phone                  | persistent_data_block                     |
> | pinner               | power                 | print                     | processinfo            | procstats                                 |
> | qspmsvc              |                       |                           |                        |                                           |
> | recovery             | restrictions          | role                      | rollback               | runtime                                   |
> | search               | secure_element        | security                  | scheduling_policy      | sec_key_att_app_id_provider               |
> | sensor_privacy       | sensorservice         | serial                    | settings               | servicediscovery                          |
> | shortcut             | sip                   | slice                     | simphonebook           | soundtrigger                              |
> | stats                | statscompanion        | statusbar                 | storaged               | storaged_pri                              |
> | storagestats         | system_update         |                           |                        |                                           |
> | telecom              | testharness           | textclassification        | textservices           | telephony.registry                        |
> | thermalservice       | time_detector         | trust                     |                        |                                           |
> | uimode               | updatelock            | uri_grants                | usagestats             | usb                                       |
> | user                 |                       |                           |                        |                                           |
> | vibrator             | voiceinteraction      | vendor.perfservice        | vendor.audio.vrservice |                                           |
> | wallpaper            | webviewupdate         | wifi                      | wifiaware              | whetstone.activity                        |
> | wificond             | wifip2p               | wifirtt                   | wifiscanner            |                                           |
> | xiaomi.joyose        |                       |                           |                        |                                           |



### 1.1.3dumpsys -t

指定dumpsys工具的超时期限

举一个例子，dumpsys中meminfo的dump输出会比较耗时。

```shell
dumpsys -t 2 meminfo
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_1.png" thumbnail="/dumpsys/dumpsys_1.png" title="">}}

### 1.1.4dumpsys --skip

指定不希望包含在输出中的 services

笔者这里去除所有，只留下adb的服务，这样一下就能发现

```shell
dumpsys --skip DockObserver   MiuiBackup   MiuiInit   ProcessManager   SurfaceFlinger   accessibility   account   activity   activity_task    alarm   android.security.keystore   app_binding   app_prediction   appops   appwidget   ashmem_device_service   audio   autofill   backup   battery   batteryproperties   batterystats   binder_calls_stats   biometric   bluetooth_manager   bsgamepad   bugreport   carrier_config clipboard   color_display   com.goodix.FingerprintService   com.qualcomm.location.izat.IzatService   companiondevice   connectivity   connmetrics   consumer_ir   content   content_suggestions   contexthub   country_detector   cpuinfo   crossprofileapps   dbinfo device_config   device_identifiers   device_policy   deviceidle   devicestoragemonitor  diskstats   display   dpmservice   dreams   drm.drmManager   dropbox   dynamic_system  ethernet   external_vibrator_service   extphone   fingerprint   gfxinfo   gpu   graphicsstats   hardware_properties   idmap   imms   incidentcompanion   input   input_method  inputflinger   ions   iphonesubinfo   ipsec   ircs   isms   isub   jobscheduler   launcherapps   location   locationpolicy   lock_settings   looper_stats   media.aaudio   media.audio_flinger   media.audio_policy   media.camera   media.camera.proxy   media.drm media.extractor   media.metrics   media.player   media.resource_manager   media.sound_trigger_hw   media_projection   media_resource_monitor   media_router   media_session   meminfo   midi   miui.contentcatcher.ContentCatcherService   miui.face.FaceService   miui.fdpp   miui.mqsas.IMQSNative   miui.mqsas.MQSService   miui.sedc   miui.shell   miui.whetstone.klo   miui.whetstone.mcd   miui.whetstone.power   miuiboosterservice   mount   netd_listener   netpolicy   netstats   network_management   network_score   network_stack  network_time_update_service   network_watchlist   nfc   notification   oem_lock   otadexopt   overlay   package   package_native   perfshielder   permission   persistent_data_block   phone   pinner   power   print   processinfo   procstats   qspmsvc   recovery restrictions   role   rollback   runtime   scheduling_policy   search   sec_key_att_app_id_provider   secure_element   security   sensor_privacy   sensorservice   serial   servicediscovery   settings   shortcut   simphonebook   sip   slice   soundtrigger   stats  statscompanion   statusbar   storaged   storaged_pri   storagestats   system_update telecom   telephony.registry   testharness   textclassification   textservices   thermalservice   time_detector   trust   uimode   updatelock   uri_grants   usagestats   usb user   vendor.audio.vrservice   vendor.perfservice   vibrator   voiceinteraction   wallpaper   webviewupdate   whetstone.activity   wifi   wifiaware   wificond   wifip2p   wifirtt   wifiscanner   window   xiaomi.joyose
```

这里截取部分输出信息，可以看到如果被skip包含进去的service，后面会被追加skipped的标志位，然后输出的时候就会被自动过滤掉。然后接着会输出未被skipped标志的所有服务，这里只有adb服务，那么下面就会输出dump这个adb的服务内容。

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_3.png" thumbnail="/dumpsys/dumpsys_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_4.png" thumbnail="/dumpsys/dumpsys_4.png" title="">}}





### 1.1.5dumpsys --priority 

根据指定的优先级过滤服务，LEVEL只能是三种里面选择

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_8.png" thumbnail="/dumpsys/dumpsys_8.png" title="">}}

这里默认有三种等级的服务，分别有CRITICAL |  HIGH | NORMAL。这里只能打印显示三种优先级的服务，如果需要自定义添加删除，需要在对应的服务添加的时候去修改。这里暂且按下不表，在后面的源码过程中会比较清楚。

### 1.1.6dumpsys SERVICE

指定希望输出的 service。某些服务可能允许您传递可选 arguments

这里包含两部分，一部分是指定service，另一部分除了指定service之外，还可以增加参数

有些是支持参数的，可以通过-h查询

```shell
dumpsys activity -h
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_6.png" thumbnail="/dumpsys/dumpsys_6.png" title="">}}

还有一些是不支持的，比如adb

```shell
dumpsys adb -h
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_5.png" thumbnail="/dumpsys/dumpsys_5.png" title="">}}

### 1.1.7dumpsys --proto

过滤支持以proto格式dump数据的服务

可以看到比如window这个服务是支持proto的，那么我可以直接以这种方式输出，proto可以认为是一种序列化方式。

关于是否支持proto，可以通过直接在一个服务中-h，先查询是否可以有--proto的选项，有的话那就支持。

```
dumpsys window --proto
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_7.png" thumbnail="/dumpsys/dumpsys_7.png" title="">}}

打印出来是乱码，这是因为输出默认就是protocol的格式，是一种二进制文件，需要特定的方式还原到可以阅读的方式，具体dump出来的proto数据，如何解析可以查看[这里](https://www.jianshu.com/p/1cc7615f030f?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)。

## 1.2实例

例如常见的指令

```shell
dumpsys --help
dumpsys adb
dumpsys window -h
dumpsys activity --proto
dumpsys --skip meminfo
dumpsys --priority HIGH -l
```



# 2dumpsys源码分析

dumpsys是Android自带的强大debug工具，它其实是一个进程，类似源码目录下的external目录下的一些工程文件用于自定义进程，apk等等。本文以android 11源码做分析。

目录结构如下所示

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_10.png" thumbnail="/dumpsys/dumpsys_10.png" title="">}}

因为这个dumpsys是一个进程没被定义在bp文件中，dumpsys是被编译成了一个可执行的文件，并放在了系统中，路径在/system/bin目录下面

```makefile
//frameworks/native/cmds/dumpsys/Android.bp
//
// Executable
//

cc_binary {
    name: "dumpsys",

    defaults: ["dumpsys_defaults"],

    srcs: [
        "main.cpp",
    ],
}
```

通过bp文件调用到main.cpp的函数入口，main函数

```c++
//frameworks/native/cmds/dumpsys/main.cpp#main
int main(int argc, char* const argv[]) {
    signal(SIGPIPE, SIG_IGN);
    //获取大管家serviceManager
    sp<IServiceManager> sm = defaultServiceManager();
    fflush(stdout);
    if (sm == nullptr) {
        ALOGE("Unable to get default service manager!");
        std::cerr << "dumpsys: Unable to get default service manager!" << std::endl;
        return 20;
    }
    //最终调用到了这里的main
    Dumpsys dumpsys(sm.get());
    return dumpsys.main(argc, argv);
}
```

最终会调用到这里，因为代码比较冗长，这里**笔者人为的分成三段**。

## 2.1Dumpsys的Option处理

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::main#1
int Dumpsys::main(int argc, char* const argv[]) {
    Vector<String16> services;
    Vector<String16> args;
    String16 priorityType;
    Vector<String16> skippedServices;
    Vector<String16> protoServices;
    bool showListOnly = false;
    bool skipServices = false;
    bool asProto = false;
    Type type = Type::DUMP;
    int timeoutArgMs = 10000;
    int priorityFlags = IServiceManager::DUMP_FLAG_PRIORITY_ALL;
    static struct option longOptions[] = {{"pid", no_argument, 0, 0},
                                          {"priority", required_argument, 0, 0},
                                          {"proto", no_argument, 0, 0},
                                          {"skip", no_argument, 0, 0},
                                          {"help", no_argument, 0, 0},
                                          {0, 0, 0, 0}};
    optind = 1;
    while (1) {
        int c;
        int optionIndex = 0;
		//这个函数比较重要，需要结合longOptions分析
        c = getopt_long(argc, argv, "+t:T:l", longOptions, &optionIndex);

        if (c == -1) {
            break;
        }

        switch (c) {
        case 0:
            if (!strcmp(longOptions[optionIndex].name, "skip")) {
                skipServices = true;
            } else if (!strcmp(longOptions[optionIndex].name, "proto")) {
                asProto = true;
            } else if (!strcmp(longOptions[optionIndex].name, "help")) {
                usage();
                return 0;
            } else if (!strcmp(longOptions[optionIndex].name, "priority")) {
                priorityType = String16(String8(optarg));
                if (!ConvertPriorityTypeToBitmask(priorityType, priorityFlags)) {
                    fprintf(stderr, "\n");
                    usage();
                    return -1;
                }
            } else if (!strcmp(longOptions[optionIndex].name, "pid")) {
                type = Type::PID;
            }
            break;

        case 't':
            {
                char* endptr;
                timeoutArgMs = strtol(optarg, &endptr, 10);
                timeoutArgMs = timeoutArgMs * 1000;
                if (*endptr != '\0' || timeoutArgMs <= 0) {
                    fprintf(stderr, "Error: invalid timeout(seconds) number: '%s'\n", optarg);
                    return -1;
                }
            }
            break;

        case 'T':
            {
                char* endptr;
                timeoutArgMs = strtol(optarg, &endptr, 10);
                if (*endptr != '\0' || timeoutArgMs <= 0) {
                    fprintf(stderr, "Error: invalid timeout(milliseconds) number: '%s'\n", optarg);
                    return -1;
                }
            }
            break;

        case 'l':
            showListOnly = true;
            break;

        default:
            fprintf(stderr, "\n");
            usage();
            return -1;
        }
    }
    ...
}
```

> 上述有几个陌生的函数，这里稍微提一下
>
> getopt_long
>
> ```c++
> int getopt_long(int argc, char * const argv[], const char *optstring, const struct option *longopts, int *longindex);
> ```
>
> 1、**argc和argv**和main函数的两个参数一致
>
> 2、**optstring:** 表示短选项字符串。这里用到了"+t:T:l"，实际上是由三个函数，t和T都带一个参数，l后面不带参数，
>
> ```tex
> 形式如“a:b::cd:“，分别表示程序支持的命令行短选项有-a、-b、-c、-d，冒号含义如下：
> (1)只有一个字符，不带冒号——只表示选项， 如-c 
> (2)一个字符，后接一个冒号——表示选项后面带一个参数，如-a 100
> (3)一个字符，后接两个冒号——表示选项后面带一个可选参数，即参数可有可无， 如果带参数，则选项与参数直接不能有空格
>     形式应该如-b200
> ```
> 3、**longopts**：表示长选项结构体，这里可自定义参数
>
> ```c++
> struct option 
> {  
>     const char *name;  
>     int         has_arg;  
>     int        *flag;  
>     int         val;  
> };
> //本文自定的内容如下
> static struct option longOptions[] = {{"pid", no_argument, 0, 0},
>                                       {"priority", required_argument, 0, 0},
>                                       {"proto", no_argument, 0, 0},
>                                       {"skip", no_argument, 0, 0},
>                                       {"help", no_argument, 0, 0},
>                                       {0, 0, 0, 0}};
> ```
>
> 这里的option结构体有四个成员
>
> 1. name:表示选项的名称,比如pid,priority,proto等。
>
> 2. has_arg:表示选项后面是否携带参数。该参数有三个不同值，如下：
>
>    a:  no_argument(或者是0)时   ——参数后面不跟参数值，eg: --help
>
>    b: required_argument(或者是1)时 ——参数输入格式为：--参数 值 或者 --参数=值。eg:--dir=/home
>
>    c: optional_argument(或者是2)时  ——参数输入格式只能为：--参数=值
>
> 3. flag:这个参数有两个意思，空或者非空。
>
>    a:**如果参数为空NULL，那么当选中某个长选项的时候，getopt_long将返回val值**。
>
>    eg，可执行程序 --help，getopt_long的返回值为h. 
>
>    b:如果参数不为空，那么当选中某个长选项的时候，getopt_long将返回0，并且将flag指针参数指向val值。
>
>    eg: 可执行程序 --http-proxy=127.0.0.1:80 那么getopt_long返回值为0，并且flag指向的值为1。
>
>    ```c++
>    static struct option longOpts[] = {
>        { "daemon", no_argument, NULL, 'D' },
>        { "dir", required_argument, NULL, 'd' },
>        { "http-proxy", required_argument, &flag, 1 },
>        { "version", no_argument, NULL, 'v' },
>        { "help", no_argument, NULL, 'h' },
>        { 0, 0, 0, 0 }
>    };
>    ```
>
> 4. val：表示指定函数找到该选项时的返回值，或者当flag非空时指定flag指向的数据的值val。
>
> 4、**longindex：longindex**非空，它指向的变量将记录当前找到参数符合longopts里的第几个元素的描述，即是longopts的下标值
>
> 
>
> 补充注意点：
>
> - 如果 optstring 的第一个字符是 **'+'** 或设置了环境变量 POSIXLY_CORRECT，则一旦遇到**非选项参数**，选项处理就会停止
> - 如果 optstring 的第一个字符是 '-'，则每个非选项 argv 元素都被处理为就好像**它是具有字符代码 1 的选项的参数**一样
> - 如果 getopt() 不能识别选项字符，它会向 stderr 打印一条错误消息，将该字符存储在 optopt 中，然后返回“？”
> - 如果 optstring 的第一个字符（在上述任何可选的 '+' 或 '-' 之后）是冒号 (':')，则 getopt() 返回 ':' 而不是 '?'，表示缺少选项参数。
> - 如果检测到错误，并且 optstring 的第一个字符不是冒号，并且外部变量 opterr 不为零（这是默认值），则 getopt() 会打印一条错误消息。
> - 更多详细的内容，可通过man getopt_long在linux c函数中查看。



### 2.1.1 getopt_long

那么本文的getopt_long(argc, argv, "+t:T:l", longOptions, &optionIndex);含义就是

总共会有四种返回值，分别是t，T，l和0。因为自定义的longOpts里面的val都为0，所以返回值为0的又会有5种情况，分别对应结构体的5个name，"pid"，"priority"，"proto"，"skip"，"help"。

### 2.1.2 ConvertPriorityTypeToBitmask

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#ConvertPriorityTypeToBitmask
static bool ConvertPriorityTypeToBitmask(const String16& type, int& bitmask) {
    if (type == PriorityDumper::PRIORITY_ARG_CRITICAL) {
        bitmask = IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL;
        return true;
    }
    if (type == PriorityDumper::PRIORITY_ARG_HIGH) {
        bitmask = IServiceManager::DUMP_FLAG_PRIORITY_HIGH;
        return true;
    }
    if (type == PriorityDumper::PRIORITY_ARG_NORMAL) {
        bitmask = IServiceManager::DUMP_FLAG_PRIORITY_NORMAL;
        return true;
    }
    return false;
}
```

这个函数用来打印优先级的服务，比如根据上述1.1.5过程中的图片，我们只能打印优先级的服务

如果我们需要自己添加删除服务优先级的话，也是可以的。比如对于activity的修改，首先打开AMS。可以看到只要对应修改DUMP_FLAG_PRIORITY_的值即可。

```c++
//framworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void setSystemProcess() {
    try {
        //这里的Context.ACTIVITY_SERVICE里面有CRITICAL和NORMAL，如果需要在HIGH中添加，在添加一个DUMP_FLAG_PRIORITY_HIGH即可
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                                  DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        //对应的meminfo，默认是在优先级HIGH中的
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                                  DUMP_FLAG_PRIORITY_HIGH);
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                                      /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ServiceManager.addService("cacheinfo", new CacheBinder(this));
        ...
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
            "Unable to find android system package", e);
    }

    ...
}

```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_9.png" thumbnail="/dumpsys/dumpsys_9.png" title="">}}



## 2.2Dumpsys services的处理

这里做了service的操作，用于标记是否为skipped，标记过skipped的服务，会在最后打印过程中列出，并且显示(skipped)

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::main#2
int Dumpsys::main(int argc, char* const argv[]) {
    ...
    for (int i = optind; i < argc; i++) {
        if (skipServices) {
            //如果是skip关键字后的服务，那么就存到这个数组中
            skippedServices.add(String16(argv[i]));
        } else {
            if (i == optind) {
                //这个optind实际上是保存的service，类似dumpsys activity
                services.add(String16(argv[i]));
            } else {
                const String16 arg(argv[i]);
                args.add(arg);
				//这里面的PriorityDumper::PROTO_ARG为 u"--proto"
                //用于判断单个服务后面跟的参数是proto类型
                //比如dumpsys activity --proto就会走到这里
                if (!asProto && !arg.compare(String16(PriorityDumper::PROTO_ARG))) {
                    asProto = true;
                }
            }
        }
    }

    if ((skipServices && skippedServices.empty()) ||
            (showListOnly && (!services.empty() || !skippedServices.empty()))) {
        usage();
        return -1;
    }

    if (services.empty() || showListOnly) {
        //有两种情况会把所有的服务添加到services数组，一个进行skip操作，另外一种-l或者是什么参数都不加
        services = listServices(priorityFlags, asProto);
        setServiceArgs(args, asProto, priorityFlags);
    }

    const size_t N = services.size();
    if (N > 1) {
        // first print a list of the current services
        std::cout << "Currently running services:" << std::endl;

        for (size_t i=0; i<N; i++) {
            //校验是否service存在，如果没有这个服务就会有提示不存在
            sp<IBinder> service = sm_->checkService(services[i]);
            //这里主要判断是都需要加(skipped)标记
            if (service != nullptr) {
                bool skipped = IsSkipped(skippedServices, services[i]);
                std::cout << "  " << services[i] << (skipped ? " (skipped)" : "") << std::endl;
            }
        }
    }

    if (showListOnly) {
        return 0;
    }
    ...
}
```

这里的逻辑比较简单，主要是判断是否存在--skip的存在，并将一系列的skip的services保存起来。

有两种情况，一种只有skip标志位，还有一种是service+可加参数组合

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_11.png" thumbnail="/dumpsys/dumpsys_11.png" title="">}}

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_12.png" thumbnail="/dumpsys/dumpsys_12.png" title="">}}

### 2.2.1listServices

这个函数的作用，**是将priorityFilterFlags类型的服务添加到services数组中，对做一个排序**，这里排序用到了STL的仿函数，仿函数为sort_func。

如果这个函数的filterByProto是true，那么这些服务可以进行proto操作，会将原本存在services数组中的服务数组转移到另外一个叫protoServices的数组中。

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::listServices
Vector<String16> Dumpsys::listServices(int priorityFilterFlags, bool filterByProto) const {
    Vector<String16> services = sm_->listServices(priorityFilterFlags);
    services.sort(sort_func);
    if (filterByProto) {
        Vector<String16> protoServices = sm_->listServices(IServiceManager::DUMP_FLAG_PROTO);
        protoServices.sort(sort_func);
        Vector<String16> intersection;
        std::set_intersection(services.begin(), services.end(), protoServices.begin(),
                              protoServices.end(), std::back_inserter(intersection));
        services = std::move(intersection);
    }
    return services;
}

//仿函数
static int sort_func(const String16* lhs, const String16* rhs)
{
    return lhs->compare(*rhs);
}
```

无proto操作

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_13.png" thumbnail="/dumpsys/dumpsys_13.png" title="">}}

proto操作

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_14.png" thumbnail="/dumpsys/dumpsys_14.png" title="">}}

这个函数调用listServices(priorityFlags, asProto)，入参为priorityFlags和asProto，默认priorityFlags为DUMP_FLAG_PRIORITY_ALL。其中的DUMP_FLAG_PRIORITY_ALL就是15，也就是包含所有服务的意思。asProto是判断这个服务是否可以进行Proto序列化的操作输出。

```c++
//frameworks/native/libs/binder/include/binder/IServiceManager.h        
static const int DUMP_FLAG_PRIORITY_CRITICAL = 1 << 0;									//1
static const int DUMP_FLAG_PRIORITY_HIGH = 1 << 1;										//2
static const int DUMP_FLAG_PRIORITY_NORMAL = 1 << 2;									//4
static const int DUMP_FLAG_PRIORITY_DEFAULT = 1 << 3;									//8
static const int DUMP_FLAG_PRIORITY_ALL = DUMP_FLAG_PRIORITY_CRITICAL |
    DUMP_FLAG_PRIORITY_HIGH | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PRIORITY_DEFAULT;	//15
static const int DUMP_FLAG_PROTO = 1 << 4;												//16
```





## 2.3dumpsys services的实现

通过上面的三个vector数组和标志位，再加上对services的检查，开始针对对应的services进行dumpsys数据。默认不加参数也是dumpsys所有的services。

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::main#3
int Dumpsys::main(int argc, char* const argv[]) {
    ...
    for (size_t i = 0; i < N; i++) {
        const String16& serviceName = services[i];
        if (IsSkipped(skippedServices, serviceName)) continue;

        //检查是否状态值为OK
        if (startDumpThread(type, serviceName, args) == OK) {
            bool addSeparator = (N > 1);
            if (addSeparator) {
                writeDumpHeader(STDOUT_FILENO, serviceName, priorityFlags);
            }
            std::chrono::duration<double> elapsedDuration;
            size_t bytesWritten = 0;
            
            status_t status =
                writeDump(STDOUT_FILENO, serviceName, std::chrono::milliseconds(timeoutArgMs),
                          asProto, elapsedDuration, bytesWritten);

            if (status == TIMED_OUT) {
                std::cout << std::endl
                     << "*** SERVICE '" << serviceName << "' DUMP TIMEOUT (" << timeoutArgMs
                     << "ms) EXPIRED ***" << std::endl
                     << std::endl;
            }

            if (addSeparator) {
                writeDumpFooter(STDOUT_FILENO, serviceName, elapsedDuration);
            }
            bool dumpComplete = (status == OK);
            stopDumpThread(dumpComplete);
        }
    }
    return 0;
}
```

### 2.3.1startDumpThread

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::startDumpThread
status_t Dumpsys::startDumpThread(Type type, const String16& serviceName,
                                  const Vector<String16>& args) {
    //如果输入的只有一个service，那么这里需要去进行服务校验
    sp<IBinder> service = sm_->checkService(serviceName);
    if (service == nullptr) {
        std::cerr << "Can't find service: " << serviceName << std::endl;
        return NAME_NOT_FOUND;
    }
    
    
    int sfd[2];
    if (pipe(sfd) != 0) {
        std::cerr << "Failed to create pipe to dump service info for " << serviceName << ": "
             << strerror(errno) << std::endl;
        return -errno;
    }

    redirectFd_ = unique_fd(sfd[0]);
    unique_fd remote_end(sfd[1]);
    sfd[0] = sfd[1] = -1;

    //用异步回调的方式调用到真是的service->dump
    activeThread_ = std::thread([=, remote_end{std::move(remote_end)}]() mutable {
        status_t err = 0;

        switch (type) {
        case Type::DUMP:
            //这里是默认的dump
            err = service->dump(remote_end.get(), args);
            break;
        case Type::PID:
            err = dumpPidToFd(service, remote_end);
            break;
        default:
            std::cerr << "Unknown dump type" << static_cast<int>(type) << std::endl;
            return;
        }

        if (err != OK) {
            std::cerr << "Error dumping service info status_t: " << statusToString(err) << " "
                 << serviceName << std::endl;
        }
    });
    return OK;
}
```

> 这里用到了管道pipe函数，管道是一种把两个进程之间的标准输入和标准输出连接起来的机制，从而提供一种让多个进程间通信的方法。如下图所示。
>
> {{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_15.png" thumbnail="/dumpsys/dumpsys_15.png" title="">}}
>
> 特别如下：
>
> - **数据只能从管道的一端写入，从另一端读出**
> - 写入管道的数据遵循先入先从出的规则
> - **管道不是普通的文件，不属于某个文件系统，其只存在于内存中**
> - 管道在内存中对应一个缓冲区，不同的系统其大小不一定相同
> - **从管道读数据是一次性操作，数据一旦被读走，它就从管道中丢弃，释放空间以便写更多的数据**
>
> 而本文中的通过新启动一个线程，将fd数组的fd[0]去做读的操作，fd[1]去做write操作，如下unique_fd参数中的读写操作。
>
> ```c++
> //system/core/base/include/android-base/unique_fd.h
> template <typename Closer>
> inline bool Pipe(unique_fd_impl<Closer>* read, unique_fd_impl<Closer>* write,
>                  int flags = O_CLOEXEC) {
>     ...
>     read->reset(pipefd[0]);
>     write->reset(pipefd[1]);
>     return true;
> }
> ```
>
> {{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_18.png" thumbnail="/dumpsys/dumpsys_18.png" title="">}}
>
> 写入操作的fd[1]就是remote_end，会传入到对应service::dump中去，把dump的数据写入到管道中。
>
> ```c++
> //这里是默认的dump
> err = service->dump(remote_end.get(), args);
> ```

关于如何获取ServiceManager和Service这个后续会在Binder篇章中详细解析，这个先按下不表。

由前面的原理可知， 先要查询sm->checkService(“media.camera”)，这里得到的是CameraService，那么也就意味着上述命令等价于调用CameraService.dump()。 同理其他的命令也是类似的方式。

### 2.3.2writeDump

这个函数比较复杂，涉及到linux的poll机制。跟上面的写入管道操作对应，这里是从管道中读出，`redirectFd_`对应是fd[0]，这里的`redirectFd_`是全局变量，所以在startDumpThread的异步操作写入到管道中，这里的writeDump可以从管道中读出数据。

```c++
//frameworks/native/cmds/dumpsys/dumpsys.cpp#Dumpsys::writeDump
status_t Dumpsys::writeDump(int fd, const String16& serviceName, std::chrono::milliseconds timeout,
                            bool asProto, std::chrono::duration<double>& elapsedDuration,
                            size_t& bytesWritten) const {
    status_t status = OK;
    size_t totalBytes = 0;
    auto start = std::chrono::steady_clock::now();
    auto end = start + timeout;

    int serviceDumpFd = redirectFd_.get();
    if (serviceDumpFd == -1) {
        return INVALID_OPERATION;
    }

    struct pollfd pfd = {.fd = serviceDumpFd, .events = POLLIN};

    while (true) {
        // Wrap this in a lambda so that TEMP_FAILURE_RETRY recalculates the timeout.
        auto time_left_ms = [end]() {
            auto now = std::chrono::steady_clock::now();
            auto diff = std::chrono::duration_cast<std::chrono::milliseconds>(end - now);
            return std::max(diff.count(), 0LL);
        };

        int rc = TEMP_FAILURE_RETRY(poll(&pfd, 1, time_left_ms()));
        if (rc < 0) {
            std::cerr << "Error in poll while dumping service " << serviceName << " : "
                 << strerror(errno) << std::endl;
            status = -errno;
            break;
        } else if (rc == 0) {
            status = TIMED_OUT;
            break;
        }

        char buf[4096];
        //从管道中读出数据，每次管道输出4096个字节大小
        rc = TEMP_FAILURE_RETRY(read(redirectFd_.get(), buf, sizeof(buf)));
        if (rc < 0) {
            std::cerr << "Failed to read while dumping service " << serviceName << ": "
                 << strerror(errno) << std::endl;
            status = -errno;
            break;
        } else if (rc == 0) {
            // EOF.
            break;
        }

        if (!WriteFully(fd, buf, rc)) {
            std::cerr << "Failed to write while dumping service " << serviceName << ": "
                 << strerror(errno) << std::endl;
            status = -errno;
            break;
        }
        totalBytes += rc;
    }

    if ((status == TIMED_OUT) && (!asProto)) {
        std::string msg = StringPrintf("\n*** SERVICE '%s' DUMP TIMEOUT (%llums) EXPIRED ***\n\n",
                                       String8(serviceName).string(), timeout.count());
        WriteStringToFd(msg, fd);
    }

    elapsedDuration = std::chrono::steady_clock::now() - start;
    bytesWritten = totalBytes;
    return status;
}
```



# 3dumpsys media.camera

## 3.1打印结果

这里开始解释一下名词，和详细的信息，因为笔者没有高通对接的Camera源码，也只能通过例子结合网上说明去解释。

这里的打印主要分成这六种信息，下面对几种信息简单概述，具体可在文末可以下载具体的dump信息。

### 3.1.1Service global info

```tex
Number of camera devices: 10
Number of normal camera devices: 6
    Device 0 maps to "0"
    Device 1 maps to "1"
    Device 2 maps to "2"
    Device 3 maps to "3"
    Device 4 maps to "8"
    Device 5 maps to "9"
Active Camera Clients:
[
(Camera ID: 0, Cost: 99, PID: 14259, Score: -900, State: 0User Id: 0, Client Package Name: com.android.camera, Conflicting Client Devices: {6, })
]
Allowed user IDs: 0, 999
```

上面的含义是总共有十个摄像头可支持，但是目前能用的是6个，其中当前用的摄像头是设备0。

这里解释一下，以红米K30说明。其中的device 0是后摄，device 1是前摄，device 2是微距，device 3是专业模式，device 6是人像模式。

### 3.1.2EventLog

```tex
04-15 22:35:16 : CONNECT device 0 client for package com.android.camera (PID 14259)
04-15 16:12:04 : DISCONNECT device 0 client for package com.eg.android.AlipayGphone (PID 30996)
04-15 16:12:00 : CONNECT device 0 client for package com.eg.android.AlipayGphone (PID 30996)
04-15 14:55:08 : DISCONNECT device 0 client for package com.zte.softda (PID 24784)
04-15 14:55:06 : CONNECT device 0 client for package com.zte.softda (PID 24784)
04-15 10:17:27 : DISCONNECT device 0 client for package com.tencent.mm (PID 13858)
04-15 10:17:24 : CONNECT device 0 client for package com.tencent.mm (PID 13858)
...
```

正常能够容纳100条，历史使用摄像头记录。包括时间，哪种设备连接摄像头，以及连接哪个device的摄像头情况。

### 3.1.3Camera device dynamic info

```tex
  Camera1 API shim is using parameters:
        CameraParameters::dump: mMap.size = 59
        antibanding: auto
        antibanding-values: off,50hz,60hz,auto
        auto-exposure-lock: false
        auto-exposure-lock-supported: true
        auto-whitebalance-lock: false
        auto-whitebalance-lock-supported: true
        effect: none
        effect-values: none,mono,negative,solarize,sepia,posterize,aqua,blackboard,whiteboard
        exposure-compensation: 0
        exposure-compensation-step: 0.166667
        flash-mode: off
        flash-mode-values: off,auto,on,torch
        focal-length: 5.43
        focus-areas: (0,0,0,0,0)
        focus-distances: Infinity,Infinity,Infinity
        focus-mode: auto
        focus-mode-values: infinity,auto,macro,continuous-video,continuous-picture
        horizontal-view-angle: 68.5295
        jpeg-quality: 90
        jpeg-thumbnail-height: 154
        jpeg-thumbnail-quality: 90
        jpeg-thumbnail-size-values: 0x0,176x144,205x154,240x144,256x144,240x160,256x154,246x184,240x240,320x240
        jpeg-thumbnail-width: 205
        max-exposure-compensation: 24
        max-num-detected-faces-hw: 10
        max-num-detected-faces-sw: 0
        max-num-focus-areas: 1
        max-num-metering-areas: 1
        max-zoom: 99
        metering-areas: (0,0,0,0,0)
        min-exposure-compensation: -24
        picture-format: jpeg
        picture-format-values: jpeg
        picture-size: 4624x3472
        picture-size-values: 4624x3472,4624x2600,4624x2080,3472x3472,4096x2160,3840x2160,3264x2448,2592x1944,2592x1940,2400x1080,2160x1080,1920x1080,1600x1200,1440x1080,1280x960,1560x720,1440x720,1280x720,864x480,800x600,800x480,800x400,720x480,640x480,640x360,
        preferred-preview-size-for-video: 1920x1080
        preview-format: yuv420sp
        preview-format-values: yuv420p,yuv420sp,
        preview-fps-range: 14000,30000
        preview-fps-range-values: (15000,15000),(14000,20000),(20000,20000),(14000,30000),(30000,30000)
        preview-frame-rate: 30
        preview-frame-rate-values: 15,20,30
        preview-size: 1920x1080
        preview-size-values: 1920x1080,1600x1200,1440x1080,1280x960,1560x720,1440x720,1280x720,864x480,800x600,800x480,800x400,720x480,640x480,640x360,352x288,320x240,176x144
        recording-hint: false
        rotation: 0
        smooth-zoom-supported: false
        vertical-view-angle: 54.1821
        video-frame-format: android-opaque
        video-size: 1920x1080
        video-size-values: 4096x2160,3840x2160,2592x1944,2592x1940,2400x1080,2160x1080,1920x1080,1600x1200,1440x1080,1280x960,1560x720,1440x720,1280x720,864x480,800x600,800x480,800x400,720x480,640x480,640x360,352x288,320x240,176x144
        video-snapshot-supported: false
        video-stabilization: false
        video-stabilization-supported: true
        whitebalance: auto
        whitebalance-values: auto,incandescent,fluorescent,warm-fluorescent,daylight,cloudy-daylight,twilight,shade,
        zoom: 0
        zoom-ratios: 100,109,118,127,136,145,154,163,172,181,190,200,209,218,227,236,245,254,263,272,281,290,299,309,318,327,336,345,354,363,372,381,390,399,409,418,427,436,445,454,463,472,481,490,499,509,518,527,536,545,554,563,572,581,590,599,609,618,627,636,
        zoom-supported: true
        ...
```

主要记录摄像头的一些基本信息，包括图像格式，分辨率，缩放大小，帧率，旋转角度等等。

其中还包含图像的histogram图，从0-255，分别对应四个通道的颜色，BGGR，这个需要按照sensor的Bayer pattern layout

```tex
[363760 84623 85493 96276 ]
[98224 98574 103085 106812 ]
[106479 109762 108894 109744 ]
[110569 109656 107188 105870 ]
[103437 101058 98766 95246 ]
[93007 90480 87229 83466 ]
[80494 77153 73958 70434 ]
[66901 64467 60989 57368 ]
[54882 51618 49085 45944 ]
[43575 40909 38256 36107 ]
[34225 31701 29517 27736 ]
[25599 24098 22416 20808 ]
[19247 18045 16468 15184 ]
[14115 13101 11864 10886 ]
[10162 9292 8389 7766 ]
[7243 6516 5858 5336 ]
[4904 4508 4042 3583 ]
[3239 2890 2661 2466 ]
[2239 1977 1764 1637 ]
[1420 1269 1135 1002 ]
[881 825 686 591 ]
[561 475 411 419 ]
[339 293 244 219 ]
[200 160 160 146 ]
[122 109 92 72 ]
[63 56 61 40 ]
[35 28 28 24 ]
[17 18 21 17 ]
[9 8 12 2 ]
[4 5 8 2 ]
[5 2 3 1 ]
[1 0 1 1 ]
[1 0 1 3 ]
[1 0 0 1 ]
[0 2 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
[0 0 0 0 ]
```

会有两个通道的输出

```tex
Stream[0]: Output
      Consumer name: SurfaceTexture-1-14259-1
      State: 4
      Dims: 1440 x 1080, format 0x22, dataspace 0x0
      Max size: 0
      Combined usage: 133376, max HAL buffers: 8
      Frames produced: 43, last timestamp: 1430362607248159 ns
      Total buffers: 10, currently dequeued: 6
      DequeueBuffer latency histogram: (49) samples
        5     10     15     20     25     30     35     40     45    inf (max ms)
     95.92   4.08   0.00   0.00   0.00   0.00   0.00   0.00   0.00   0.00 (%)
Stream[1]: Output
      Consumer name: ImageReader-4624x3472f23m10-14259-1
      State: 4
      Dims: 4624 x 3472, format 0x23, dataspace 0x8c20000
      Max size: 0
      Combined usage: 131075, max HAL buffers: 8
      Frames produced: 0, last timestamp: 0 ns
      Total buffers: 18, currently dequeued: 0
```



### 3.1.4CameraProviderManager

```tex
== Camera Provider HAL legacy/0 (v2.5, remote) static info: 10 devices: ==
== Camera HAL device device@3.5/legacy/0 (v3.5) static information: ==
  Resource cost: 99
  Conflicting devices:
    6
  API1 info:
    Has a flash unit: true
    Facing: Back
    Orientation: 90
  API2 camera characteristics:
    Dumping camera metadata array: 141 / 141 entries, 18344 / 18344 bytes of extra data.

      Version: 1, Flags: 00000000
      android.colorCorrection.availableAberrationModes (00004): byte[3]
        [0 1 2 ]
      android.control.aeAvailableAntibandingModes (10012): byte[4]
        [0 1 2 3 ]
      android.control.aeAvailableModes (10013): byte[4]
        [0 1 2 3 ]
      android.control.aeAvailableTargetFpsRanges (10014): int32[10]
        [15 15 14 20 ]
        [20 20 14 30 ]
        [30 30 ]
      android.control.aeCompensationRange (10015): int32[2]
        [-24 24 ]
      android.control.aeCompensationStep (10016): rational[1]
        [(1 / 6) ]
      android.control.afAvailableModes (10017): byte[5]
        [0 1 2 3 4 ]
      android.control.availableEffects (10018): byte[9]
        [0 1 2 3 4 5 8 7 6 ]
      android.control.availableSceneModes (10019): byte[16]
        [0 1 2 3 4 5 6 7 8 9 10 12 13 14 15 18 ]
        ...
```

这里的CameraProvider和Camera Hal是在一起出现的，这里用的是Provider2.5版本，Hal3.5版本，从/legacy/0到/legacy/9。主要是用于isp参数的配置还有通过预设一些参数和操作，某些平台可以拍raw图，qcom平台不太熟悉，可能也会有这种操作。

```
//查看相机支持的preview size和picture size
HAL_PIXEL_FORMAT_RAW16 = 32;
HAL_PIXEL_FORMAT_BLOB = 33;
HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED = 34;
HAL_PIXEL_FORMAT_YCbCr_420_888 = 35;
HAL_PIXEL_FORMAT_RAW_OPAQUE = 36;
HAL_PIXEL_FORMAT_RAW10 = 37;
HAL_PIXEL_FORMAT_RAW12 = 38;
```

这里可以看出,preview和capture最大尺寸都是4624*3472，从拍照的图片信息也同样确认是正确的。

```tex
[34 4624 3472 OUTPUT ]//最大预览尺寸
[34 4624 3472 INPUT ]
[35 4624 3472 OUTPUT ]
[35 4624 3472 INPUT ]
[33 4624 3472 OUTPUT ]//最大图片尺寸
```

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_16.png" thumbnail="/dumpsys/dumpsys_16.png" title="">}}

RAW的尺寸设置可以更大，9248*6944的大小

```tex
android.scaler.availableRawSizes (d0008): int32[14]
[9248 6944 4624 3472 ]
[1152 1720 4624 2600 ]
[1152 1296 2312 1300 ]
[2312 1300 ]
```

实际帧率范围

```tex
android.control.aeTargetFpsRange (10005): int32[2]
[14 30 ]
```



### 3.1.5GlobalVendorTagDescriptor

```tex
== Vendor tags: ==
  Dumping vendor tag descriptors for vendor with id 3854507339
  Dumping configured vendor tag descriptors: 617 entries
    0x80000000 (private_property) with type 0 (byte) defined in section org.codeaurora.qcamera3.internal_private
    0x80010000 (exposure_metering_mode) with type 1 (int32) defined in section org.codeaurora.qcamera3.exposure_metering
    0x80010001 (available_modes) with type 1 (int32) defined in section org.codeaurora.qcamera3.exposure_metering
    0x80020000 (aec_speed) with type 2 (float) defined in section org.codeaurora.qcamera3.aec_convergence_speed
    0x80030000 (strength) with type 1 (int32) defined in section org.codeaurora.qcamera3.sharpness
    0x80030001 (range) with type 1 (int32) defined in section org.codeaurora.qcamera3.sharpness
    0x80040000 (level) with type 1 (int32) defined in section org.codeaurora.qcamera3.contrast
```

这里就是entry的描述，大概有617条，用于描述一些vendor的细节，这里不展开

### 3.1.6CameraTraces

```tex
== Camera error traces (0): ==
  No camera traces collected.
```

通常来说，这里没有systrace，也没有出现异常，所以不会出现trace，是正常现象。

## 3.2从dumpsys到CameraService

这里主要分成几步，从CameraSever启动，启动CameraService服务并添加到servicemanager，启动CameraServiceProxy服务和CameraService服务的AIDL使用。其实说白了，也是一种Binder通信的实例，关于CameraService的AIDL通信，可以点击[这里](https://yangyang48.github.io/2022/03/aidl3-java%E5%92%8Cc-%E9%80%9A%E4%BF%A1/)。

总的顺序图如下所示，其中进程后面的括号是实际开发板中的进程号，进程号越小越早启动。

{{< image classes="fancybox center fig-100" src="/dumpsys/dumpsys_17.png" thumbnail="/dumpsys/dumpsys_17.png" title="">}}

### 3.2.1CameraSever启动

关于CameraService启动流程，这里展开描述一下。

首先分成几个过程，涉及到android系统的开机流程，这里只描述CameraService部分。

#### 3.2.1.1init进程会初始化属性服务，并且解析.rc文件

开机之后，Linux内核启动1号进程，对应的源文件是system/core/init/main.cpp。而这个文件源文件，会调用到system/core/init/init.cpp。这个init.cpp非常关键，会去解析**init.rc**配置文件和其他文件。我们这边只看解析部分。

```c++
//system/core/init/init.cpp
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```

可以看到这里有4个目录"/system/etc/init","/product/etc/init","/odm/etc/init","/vendor/etc/init"。等init.rc解析完成后，会来解析4个目录中的rc文件，用来执行相关的动作。

#### 3.2.1.2解析cameraserver.rc文件，并启动cameraserver

下面的Android.bp可以看到init_rc: ["cameraserver.rc"]，说明rc文件编译到/system/etc/init中。而上面的开机流程的init.cpp中会去解析这个目录，那么这个cameraserver.rc这个目录就会被解析出来。那么实际上，解析完成之后，就会启动"cameraserver.rc"。

```makefile
#frameworks/av/camera/cameraserver/Android.bp
cc_binary {
    name: "cameraserver",

    srcs: ["main_cameraserver.cpp"],

    header_libs: [
        "libmedia_headers",
    ],

    shared_libs: [
        "libcameraservice",
        "liblog",
        "libutils",
        "libui",
        "libgui",
        "libbinder",
        "libhidlbase",
        "android.hardware.camera.common@1.0",
        "android.hardware.camera.provider@2.4",
        "android.hardware.camera.provider@2.5",
        "android.hardware.camera.provider@2.6",
        "android.hardware.camera.device@1.0",
        "android.hardware.camera.device@3.2",
        "android.hardware.camera.device@3.4",
    ],
    compile_multilib: "32",
    cflags: [
        "-Wall",
        "-Wextra",
        "-Werror",
        "-Wno-unused-parameter",
    ],
    #rc文件编译到/system/etc/init中
    init_rc: ["cameraserver.rc"],

    vintf_fragments: [
        "manifest_android.frameworks.cameraservice.service@2.1.xml",
    ],
}
```

可以看到这个rc文件解析之后，被开启cameraserver，这个cameraserver实际上是一个进程，位于/system/bin/cameraserver。这里的cameraserver的rc文件，**里面没有设置oneshot，说明当service退出后会自重启**，没有设置disable，说明这个服务会随着它的类启动而自动启动，具体rc语法点击[这里](https://blog.csdn.net/eliot_shao/article/details/53163522)。

```rc
#frameworks/av/camera/cameraserver/cameraserver.rc 
service cameraserver /system/bin/cameraserver
    class main
     user cameraserver
     group audio camera input drmrpc
     ioprio rt 4
     task_profiles CameraServiceCapacity MaxPerformance
     rlimit rtprio 10 10
```

#### 3.2.1.3CameraService::instantiate

最终会进入到进程的的main函数中，这个main_cameraserver.cpp中有CameraService::instantiate()，这个是后续添加服务到ServiceManager中。

```c++
//frameworks/av/camera/cameraserver/main_cameraserver.cpp
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    hardware::configureRpcThreadpool(5, /*willjoin*/ false);

    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p", sm.get());
    CameraService::instantiate();
    ALOGI("ServiceManager: %p done instantiate", sm.get());
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

来看CameraService类的定义(截取部分)：

```c++
//frameworks/av/services/camera/libcameraservice/CameraService.h
class CameraService :
    public BinderService<CameraService>,
    public virtual ::android::hardware::BnCameraService,
    public virtual IBinder::DeathRecipient,
    public virtual CameraProviderManager::StatusListener
{
    friend class BinderService<CameraService>;
    friend class CameraClient;
    friend class CameraOfflineSessionClient;
public:
    class Client;
    class BasicClient;
    class OfflineClient;
    
    // Implementation of BinderService<T>
    static char const* getServiceName() { return "media.camera"; }

                        CameraService();
    virtual             ~CameraService();

    virtual status_t    dump(int fd, const Vector<String16>& args);
    ...
}
```

可以看到主要是getService和dump这两个类成员函数会用到，其他的函数暂时先不管。



### 3.2.2启动CameraService服务并添加到servicemanager

dumpsys那些信息的打印就是上述的dump函数负责实现的。

instantiate()来自 CameraService的父类BinderService，BinderService是一个模版类。最终这个会真正启动CameraService，对应的对应的binder是"media.camera"。

```c++
//frameworks/native/libs/binder/include/binder/BinderService.h
template<typename SERVICE>
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false,
                            int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
        sp<IServiceManager> sm(defaultServiceManager());
        return sm->addService(String16(SERVICE::getServiceName()), new SERVICE(), allowIsolated,
                              dumpFlags);
    }

    static void publishAndJoinThreadPool(
            bool allowIsolated = false,
            int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
        publish(allowIsolated, dumpFlags);
        joinThreadPool();
    }

    static void instantiate() { publish(); }

    static status_t shutdown() { return NO_ERROR; }

private:
    static void joinThreadPool() {
        sp<ProcessState> ps(ProcessState::self());
        ps->startThreadPool();
        ps->giveThreadPoolName();
        IPCThreadState::self()->joinThreadPool();
    }
};
```

用 CameraService替换模版之后：

```c++
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        //这里面涉及到两个过程：
        //1.new CameraService会去真正启动CameraService
        //2.sm->addService将实例化的CameraService添加到servicemanager中，标记binder为"media.camera",其他进程可以在sm中唱过这个"media.camera"来获取这个服务，从而可以实现跨进程的Binder通信
        return sm->addService(
                String16(CameraService::getServiceName()),
                new CameraService(), allowIsolated);
    }
 
    static void instantiate() { publish(); }
};
```


这样，通过IServiceManager#addService()服务 “media.camera”就被添加到了系统中。

### 3.2.3启动CameraServiceProxy服务

#### 3.2.3.1启动system_server进程

在上述解析init.rc配置文件和其他.rc文件之后，会启动zygote进程，而这个zygote进程会启动system_server

```java
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
   
    //启动system_server的命令行参数
    String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                    + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",
    };
    ZygoteArguments parsedArgs = null;

    int pid;

        pid = Zygote.forkSystemServer(
                parsedArgs.mUid, parsedArgs.mGid,
                parsedArgs.mGids,
                parsedArgs.mRuntimeFlags,
                null,
                parsedArgs.mPermittedCapabilities,
                parsedArgs.mEffectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        //这里真正启动SystemServer
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

system_server进程的执行过程，回调用到run方法

```java
//frameworks/base/services/java/com/android/server/SystemServer.java#run
public static void main(String[] args) {
    new SystemServer().run();
}
```

这个run方法有几个功能。创建 SystemServiceManager(用于管理后面服务的生命周期)、**启动引导服务、启动核心服、启动其他服务**等

```java
private void run() {
    ...
    // 加载libandroid_servers.so
    System.loadLibrary("android_servers");

    // 创建 SystemContext
    createSystemContext();
        
    // 创建 SystemServiceManager
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);       SystemServerInitThreadPool.start();
    ... 
    // Start services.
    try {
        t.traceBegin("StartServices");
        //启动引导服务
        startBootstrapServices(t);
        //启动核心服务
        startCoreServices(t);
        //启动其他服务
        startOtherServices(t);
    } catch (Throwable ex) {
        ...
    } finally {
        t.traceEnd(); // StartServices
    }
    ... 
}
```

#### 3.2.3.2启动CameraServiceProxy服务

因为上述system_server会创建其他服务，而这个CameraService就在其他服务中。这里截取小段，可知会真实启动CameraServiceProxy，对应的binder是"media.camera.proxy"

```java
//framworks/base/services/java/com/android/server/SystemServer.java#
if (!disableCameraService) {
    t.traceBegin("StartCameraServiceProxy");
    mSystemServiceManager.startService(CameraServiceProxy.class);
    t.traceEnd();
}
```



### 3.2.4CameraService服务的AIDL使用

CameraManager类获取"media.camera"服务，通过它和摄像头打交道，通过AIDL将App的相机进程和CameraService进程互相通信。这里的getService，就是获取CameraService进程，因为之前的getname已经将"media.camera"和CameraService绑定，那么这里就可以直接调用到CameraService。App相机进程在servicemanager中查询"media.camera"这个标记binder的服务，查询之后和CameraService建立Binder通信，通过App进程中jar包Api，对应调用到CameraService中相应的实现。

```java
//以camera2为例
//frameworks/base/core/java/android/hardware/camera2/CameraManager.java
private static final String CAMERA_SERVICE_BINDER_NAME = "media.camera";
private final ICameraService mCameraService;
private void connectCameraServiceLocked() {
    Log.i(TAG, "Connecting to camera service");

    IBinder cameraServiceBinder = ServiceManager.getService(CAMERA_SERVICE_BINDER_NAME);

    ICameraService cameraService = ICameraService.Stub.asInterface(cameraServiceBinder);

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

    } catch(ServiceSpecificException e) {
        // Unexpected failure
        throw new IllegalStateException("Failed to register a camera service listener", e);
    } catch (RemoteException e) {
        // Camera service is now down, leave mCameraService as null
    }
}
```



## 3.3CameraService::dump

最终调用到CameraService这个动态库中，实际上是调用到CameraService.cpp

这里主要dump几点Service global info、EventLog、Camera device dynamic info、CameraProviderManager、GlobalVendorTagDescriptor、CameraTraces，这里的6项跟上述3.1的打印信息一一对应。

```c++
//frameworks/av/services/camera/libcameraservice/CameraService.cpp
status_t CameraService::dump(int fd, const Vector<String16>& args) {
    ATRACE_CALL();

    if (checkCallingPermission(sDumpPermission) == false) {
        dprintf(fd, "Permission Denial: can't dump CameraService from pid=%d, uid=%d\n",
                CameraThreadState::getCallingPid(),
                CameraThreadState::getCallingUid());
        return NO_ERROR;
    }
    bool locked = tryLock(mServiceLock);
    // failed to lock - CameraService is probably deadlocked
    if (!locked) {
        dprintf(fd, "!! CameraService may be deadlocked !!\n");
    }

    if (!mInitialized) {
        dprintf(fd, "!! No camera HAL available !!\n");

        // Dump event log for error information
        dumpEventLog(fd);

        if (locked) mServiceLock.unlock();
        return NO_ERROR;
    }
    dprintf(fd, "\n== Service global info: ==\n\n");
    dprintf(fd, "Number of camera devices: %d\n", mNumberOfCameras);
    dprintf(fd, "Number of normal camera devices: %zu\n", mNormalDeviceIds.size());
    dprintf(fd, "Number of public camera devices visible to API1: %zu\n",
            mNormalDeviceIdsWithoutSystemCamera.size());
    for (size_t i = 0; i < mNormalDeviceIds.size(); i++) {
        dprintf(fd, "    Device %zu maps to \"%s\"\n", i, mNormalDeviceIds[i].c_str());
    }
    String8 activeClientString = mActiveClientManager.toString();
    dprintf(fd, "Active Camera Clients:\n%s", activeClientString.string());
    dprintf(fd, "Allowed user IDs: %s\n", toString(mAllowedUsers).string());

    dumpEventLog(fd);

    bool stateLocked = tryLock(mCameraStatesLock);
    if (!stateLocked) {
        dprintf(fd, "CameraStates in use, may be deadlocked\n");
    }

    int argSize = args.size();
    for (int i = 0; i < argSize; i++) {
        if (args[i] == TagMonitor::kMonitorOption) {
            if (i + 1 < argSize) {
                mMonitorTags = String8(args[i + 1]);
            }
            break;
        }
    }

    for (auto& state : mCameraStates) {
        String8 cameraId = state.first;

        dprintf(fd, "== Camera device %s dynamic info: ==\n", cameraId.string());

        CameraParameters p = state.second->getShimParams();
        if (!p.isEmpty()) {
            dprintf(fd, "  Camera1 API shim is using parameters:\n        ");
            p.dump(fd, args);
        }

        auto clientDescriptor = mActiveClientManager.get(cameraId);
        if (clientDescriptor != nullptr) {
            dprintf(fd, "  Device %s is open. Client instance dump:\n",
                    cameraId.string());
            dprintf(fd, "    Client priority score: %d state: %d\n",
                    clientDescriptor->getPriority().getScore(),
                    clientDescriptor->getPriority().getState());
            dprintf(fd, "    Client PID: %d\n", clientDescriptor->getOwnerId());

            auto client = clientDescriptor->getValue();
            dprintf(fd, "    Client package: %s\n",
                    String8(client->getPackageName()).string());

            client->dumpClient(fd, args);
        } else {
            dprintf(fd, "  Device %s is closed, no client instance\n",
                    cameraId.string());
        }

    }

    if (stateLocked) mCameraStatesLock.unlock();

    if (locked) mServiceLock.unlock();

    mCameraProviderManager->dump(fd, args);

    dprintf(fd, "\n== Vendor tags: ==\n\n");

    sp<VendorTagDescriptor> desc = VendorTagDescriptor::getGlobalVendorTagDescriptor();
    if (desc == NULL) {
        sp<VendorTagDescriptorCache> cache =
                VendorTagDescriptorCache::getGlobalVendorTagCache();
        if (cache == NULL) {
            dprintf(fd, "No vendor tags.\n");
        } else {
            cache->dump(fd, /*verbosity*/2, /*indentation*/2);
        }
    } else {
        desc->dump(fd, /*verbosity*/2, /*indentation*/2);
    }

    // Dump camera traces if there were any
    dprintf(fd, "\n");
    camera3::CameraTraces::dump(fd, args);

    // Process dump arguments, if any
    int n = args.size();
    String16 verboseOption("-v");
    String16 unreachableOption("--unreachable");
    for (int i = 0; i < n; i++) {
        if (args[i] == verboseOption) {
            // change logging level
            if (i + 1 >= n) continue;
            String8 levelStr(args[i+1]);
            int level = atoi(levelStr.string());
            dprintf(fd, "\nSetting log level to %d.\n", level);
            setLogLevel(level);
        } else if (args[i] == unreachableOption) {
            // Dump memory analysis
            // TODO - should limit be an argument parameter?
            UnreachableMemoryInfo info;
            bool success = GetUnreachableMemory(info, /*limit*/ 10000);
            if (!success) {
                dprintf(fd, "\n== Unable to dump unreachable memory. "
                        "Try disabling SELinux enforcement. ==\n");
            } else {
                dprintf(fd, "\n== Dumping unreachable memory: ==\n");
                std::string s = info.ToString(/*log_contents*/ true);
                write(fd, s.c_str(), s.size());
            }
        }
    }
    return NO_ERROR;
}
```



> 关于什么是dprintf，实际上跟printf是非常类似的。相对于printf加了一个句柄的指向。可以通过在linux中man dprintf查看。
>
> NAME
>        printf,   fprintf,   dprintf,  sprintf,  snprintf,  vprintf,  vfprintf,
>        vdprintf, vsprintf, vsnprintf - formatted output conversion
>
> SYNOPSIS
>        #include <stdio.h>
>
>        int printf(const char *format, ...);
>        int fprintf(FILE *stream, const char *format, ...);
>        int dprintf(int fd, const char *format, ...);
>        int sprintf(char *str, const char *format, ...);
>        int snprintf(char *str, size_t size, const char *format, ...);



# 总结

总的来说，就是通过简单的dumpsys进程的api调用，深入查看dumpsys media.camera背后做的事，里面其实涉及到Binder通信，管道，poll机制，主要的流程比较能够理解。本文是笔者用api和源码的角度去熟悉media.camera的过程，因为dumpsys里面的服务比较多，这里只是以一个Camera为例子，来说明整个dumpsys过程。



# 下载

本文dumpsys的源码[点击这里](https://github.com/YangYang48/project/tree/master/dumpsys)

本文CameraService启动的源码[点击这里](https://github.com/YangYang48/project/tree/master/CameraService)

本文以手机红米K30为例的dumpsys的数据[点击这里](https://github.com/YangYang48/project/tree/master/cameradump_redmiK30)

本文以手机空米K30为例的dumpsys的proto数据[点击这里](https://github.com/YangYang48/project/tree/master/activity_dump_proto)



# 参考

[[1] 小驰笔记, Dump media.camera分析, 2021.](https://blog.51cto.com/u_15152644/2690464)

[[2] __2017__, Android框架之Camera(1)Camera服务的前世今生, 2017.](https://blog.csdn.net/u013686019/article/details/72819908?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&utm_relevant_index=2)

[[3] weixin_39970855, adb命令打开摄像头_Camera(一):查看Camera设备详细信息, 2020.](https://blog.csdn.net/weixin_39970855/article/details/111946828?spm=1001.2101.3001.6650.15&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-15.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-15.pc_relevant_default&utm_relevant_index=25)

[[4] 袁辉辉, dumpsys源码, 2015.](http://gityuan.com/2015/08/22/tool-dumpsys/)

[[5] 拥抱@, 详解C语言中的 %*s 和 %.*s, 2018.](https://blog.csdn.net/tonglin12138/article/details/85467289)

[[6] 谷歌官方, dumpsys, 2021.](https://developer.android.google.cn/studio/command-line/dumpsys)

[[7] Little丶Jerry, Android 解析 Protocol Buffers 格式数据, 2018.](https://www.jianshu.com/p/1cc7615f030f?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[[8] 猿氏悟语, Android Dumpsys命令使用以及实现原理详解, 2020.](https://blog.csdn.net/ChaoY1116/article/details/109669865)

[[9] andy cong, 浅谈linux的命令行解析参数之getopt_long函数, 2018.](https://blog.csdn.net/qq_33850438/article/details/80172275)

[[10] 伟子时代, Android系统启动流程, 2021.](https://blog.csdn.net/qq_26498311/article/details/117128520)

[[11] 菜鸡UP, Android Camera之CameraServer的启动过程, 2022.](https://blog.csdn.net/weixin_42463482/article/details/118880978)

[[12] 海月汐辰, Android mk文件 LOCAL_INIT_RC 将RC文件编译到/system/etc/init或vendor/etc/init，system\core\init\init.cpp解析这些rc, 2022.](https://blog.csdn.net/qq_37858386/article/details/121416871)

[[13] astro-yang, init.rc语法详解, 2015.](https://blog.csdn.net/feigebangni/article/details/50300063)

[[14] BBOrange丶, pipe()函数详解, 2021.](https://blog.csdn.net/weixin_44471948/article/details/120846877)

[[15] Eliot_shao, Init.rc妙用及语法说明, 2016.](https://blog.csdn.net/eliot_shao/article/details/53163522)
