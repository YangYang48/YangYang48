---
title: "service call源码分析(以蓝牙打开关闭为例)"
date: 2023-03-14
thumbnailImagePosition: left
thumbnailImage: AndroidTools/service_call/service_call_thumb.jpg
coverImage: AndroidTools/service_call/service_call_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- service call
- 2023
- March
tags:
- Android
- 源码
- 调试工具
- Binder
- Bluetooth
showSocial: false
---

Android中如果需要通过adb指令调用系统服务中的方法，可以通过service call的形式。

<!--more-->
# 0简介

举例说明，service call的常见用法

```shell
# 1.绑定port启动ViewServer
adb shell service call window 1 i32 $port

# 2.停止ViewServer
adb shell service call window 2

# 3.检查ViewServer是否正在运行
adb shell service call window 3

# 关闭HW overlays，code为1008，一个参数为int的1 
adb shell service call SurfaceFlinger 1008 i32 1

# 4.显示fps。来源见SurfaceFlinger.cpp中函数onTransact的switch片段，使用见surface_stats_collector.py
adb shell service call SurfaceFlinger 1013 
```

service call的形式本质也是使用service调试命令。

```shell
picasso:/ $ service -h
Usage: service [-h|-?]
       service list
       service check SERVICE
       service call SERVICE CODE [i32 N | i64 N | f N | d N | s16 STR ] ...
Options:
   i32: Write the 32-bit integer N into the send parcel.
   i64: Write the 64-bit integer N into the send parcel.
   f:   Write the 32-bit single-precision number N into the send parcel.
   d:   Write the 64-bit double-precision number N into the send parcel.
   s16: Write the UTF-16 string STR into the send parcel.
```

> service本身作为一个命令的bin文件
>
> 可使用的参数为
>
> 1. **service list**
>
>    显示系统当前所有在service manager注册的service，类似dumpsys -l
>
> 2. **service check SERVICE**
>
>    查询SERVICE是否存在
>
> 3. **service call SERVICE CODE [i32 N | i64 N | f N | d N | s16 STR ] ...**
>
>    可以通过binder给service发送code，还可以向service发送intent等

本文结合蓝牙服务来分析，蓝牙的开关的源码(**android13源码**)

```shell
# 打开蓝牙
adb shell service call blutooth_manager 5
# 关闭蓝牙
adb shell service call blutooth_manager 7
# 查看蓝牙状态,enable为true为打开，false为关闭
adb shell dumpsys blutooth_manager | grep enable
```

# 1 service call

这里的service位置手机目录的/system/bin下面，这里是一个二进制的进程

对应的bp文件

```go
//frameworks/native/cmds/service/Android.bp
cc_binary {
    //进程的名字是service
    name: "service",
	//生成进程的源文件service.cpp
    srcs: ["service.cpp"],
	//依赖的系统动态库
    shared_libs: [
        "libcutils",
        "libutils",
        "libbinder",
    ],
	//编译相关参数
    cflags: [
        "-DXP_UNIX",
        "-Wall",
        "-Werror",
    ],
}
```

可以知道最终service会调用的service.cpp的main函数中

# 2service call bluetooth_manager 5

本文这里以蓝牙的打开为例，即对应的蓝牙服务为**blutooth_manager** 

```c++
//frameworks/native/cmds/service/service.cpp
int main(int argc, char* const argv[])
{
    bool wantsUsage = false;
    int result = 0;
    //这里的prog_name就是通过adb指令调用的service进程
    char* prog_name = basename(argv[0]);
    //启动ServiceManager大管家，后续在大管家中找对应服务
    sp<IServiceManager> sm = defaultServiceManager();
    fflush(stdout);
    ...
    if (optind >= argc) {
        wantsUsage = true;
    //兵分三路，分别对应service的三个参数，service check、service list、service call
    } else if (!wantsUsage) {
        if (strcmp(argv[optind], "check") == 0) {
            //service check走这里
            ...
        }
        else if (strcmp(argv[optind], "list") == 0) {
            //service list走这里
            ...
        } else if (strcmp(argv[optind], "call") == 0) {
            //这里optind参数后移一位，对应的是具体的服务
            optind++;
            if (optind+1 < argc) {
                int serviceArg = optind;
                //【1】binder服务查询，通过sm大管家查询是否存在对应服务，这里为blutooth_manager 
                sp<IBinder> service = sm->checkService(String16(argv[optind++]));
                //这里的ifName为android.bluetooth.IBluetoothManager
                String16 ifName = (service ? service->getInterfaceDescriptor() : String16());
                //这里的code对应的是aidl方法中的对应的顺序，起始方法为1，按照次序往下
                int32_t code = atoi(argv[optind++]);
                if (service != nullptr && ifName.size() > 0) {
                    Parcel data, reply;
                    data.markForBinder(service);

                    // 【2】通过序列化的方式，将查询出来的服务对应的名称写到内存中去
                    data.writeInterfaceToken(ifName);

                    // 处理剩余的一些参数，主要是传递的数据，这里没有其他数据，暂时不处理
                    // 这里会涉及到数据传输
                    // 32位的int型，参数为i32
                    // 64位的int型，参数为i64
                    // 单精度的浮点数类型，参数为f
                    // 双精度的浮点数类型，参数为d
                    // string16字符串类型，参数为s16
                    while (optind < argc) {...}
		            //【3】真实的调用，这里用到了binder原理，通过客户端的代理调用真实的远端服务
                    // 这里会调用到BluetoothManagerService
                    service->transact(code, data, &reply);
                    aout << "Result: " << reply << endl;
                } else {
                    aerr << prog_name << ": Service " << argv[serviceArg]
                        << " does not exist" << endl;
                }
            } else {
                aerr << prog_name << ": No service specified for call" << endl;
            }
        } else {
            aerr << prog_name << ": Unknown command " << argv[optind] << endl;
        }
    }
    ...
    return result;
}
```

这个main函数摘取部分内容，主要对上面标注的三处进行分析

1. **查询blutooth_manager服务是否存在**

   由于已经有sm（servicemanager）这个大管家了，直接通过输入的参数blutooth_manager 找到对应的服务，这里并不是真正的服务，而是相对于服务端的一个代理，实际上是BluetoothManager.Proxy（代理可以通过binder方式跨进程调用最终服务）。

2. **蓝牙服务对应的ifName序列化**

   通过查询到的BluetoothManager.Proxy，获取对应的ifName，即为"android.bluetooth.IBluetoothManager"，通过Parcelable方式序列化这个字符串，类似下面的序列化

   ```shell
     0x000002c0: 00000000 006e0061 00720064 0069006f ' . .a.n.d.r.o.i.'
     0x000002d0: 002e0064 006c0062 00650075 006f0074 'd...b.l.u.e.t.o.'
     0x000002e0: 0074006f 002e0068 00420049 0075006c 'o.t.h...I.B.l.u.'
     0x000002f0: 00740065 006f006f 00680074 0061004d 'e.t.o.o.t.h.M.a.'
     0x00000300: 0061006e 00650067 00240000 00000000 'n.a.g.e.r. . . .'
   ```

3. **蓝牙的binder通信**

   通过code找到aidl中的序列号对应的方法，然后在对应BluetoothManagerService服务中也找到对应的方法（**不同android版本对应不同的序列号**，android11,6是打开，8是关闭）

```aidl
//packages/modules/Bluetooth/system/binder/android/bluetooth/IBluetoothManager.aidl
interface IBluetoothManager
{
    IBluetooth registerAdapter(in IBluetoothManagerCallback callback);

    void unregisterAdapter(in IBluetoothManagerCallback callback);
    @UnsupportedAppUsage
    void registerStateChangeCallback(in IBluetoothStateChangeCallback callback);
    @UnsupportedAppUsage
    void unregisterStateChangeCallback(in IBluetoothStateChangeCallback callback);
    //这个是打开
    boolean enable(in AttributionSource attributionSource);    
    boolean enableNoAutoConnect(in AttributionSource attributionSource);
    //这个是关闭
    boolean disable(in AttributionSource attributionSource, boolean persist);    
    int getState();
    @UnsupportedAppUsage   
    IBluetoothGatt getBluetoothGatt();
    ...
}
```

对应的BluetoothManagerService服务

```java
//packages/modules/Bluetooth/service/java/com/android/server/bluetooth/BluetoothManagerService.java
public class BluetoothManagerService extends IBluetoothManager.Stub {
    private static final String TAG = "BluetoothManagerService";
    //具体为打开蓝牙逻辑，不展开
    public boolean enable(AttributionSource attributionSource) throws RemoteException {
        ...
    }
    //具体为关闭蓝牙逻辑，不展开
    public boolean disable(AttributionSource attributionSource, boolean persist) {
        ...
    }
            
}    
```



# 3最终结果

如果显示如下没有权限，需要给这个命令root权限，蓝牙服务本身也会对执行命令进行权限校验

```shell
vince:/ $ service call bluetooth_manager 5
Result: Parcel(
  0x00000000: ffffffff 0000006e 0065004e 00640065 '....n...N.e.e.d.'
  0x00000010: 00420020 0055004c 00540045 004f004f ' .B.L.U.E.T.O.O.'
  0x00000020: 00480054 00410020 004d0044 004e0049 'T.H. .A.D.M.I.N.'
  0x00000030: 00700020 00720065 0069006d 00730073 ' .p.e.r.m.i.s.s.'
  0x00000040: 006f0069 003a006e 004e0020 00690065 'i.o.n.:. .N.e.i.'
  0x00000050: 00680074 00720065 00750020 00650073 't.h.e.r. .u.s.e.'
  0x00000060: 00200072 00300032 00300030 006e0020 'r. .2.0.0.0. .n.'
  0x00000070: 0072006f 00630020 00720075 00650072 'o.r. .c.u.r.r.e.'
  0x00000080: 0074006e 00700020 006f0072 00650063 'n.t. .p.r.o.c.e.'
  0x00000090: 00730073 00680020 00730061 00610020 's.s. .h.a.s. .a.'
  0x000000a0: 0064006e 006f0072 00640069 0070002e 'n.d.r.o.i.d...p.'
  0x000000b0: 00720065 0069006d 00730073 006f0069 'e.r.m.i.s.s.i.o.'
  0x000000c0: 002e006e 004c0042 00450055 004f0054 'n...B.L.U.E.T.O.'
  0x000000d0: 0054004f 005f0048 00440041 0049004d 'O.T.H._.A.D.M.I.'
  0x000000e0: 002e004e 00000000                   'N.......        ')
```

如果有root权限，通常会返回一串序列化的二进制数字，**这里的0x01，实际上返回的就是true**，说明蓝牙打开成功。

```shell
vince:/ # service call bluetooth_manager 5
service call bluetooth_manager 5
Result: Parcel(00000000 00000001   '........')
```

# 4总结

总的来说，这就是Android本身的一个cmds下面的命令，来用增加开发者对于Android系统开发调测功能，更加的便利。

{{< image classes="fancybox center fig-100" src="/AndroidTools/service_call/service_call_1.png" thumbnail="/AndroidTools/service_call/service_call_1.png" title="">}}

# 参考

[[1] SevenUUp, 【翻译】adb命令之service call, 2019.](https://blog.csdn.net/u014711665/article/details/91875959)

[[2] lyf5231, android—调试命令service, 2016.](https://blog.csdn.net/lewif/article/details/50723236?spm=1001.2101.3001.6650.11&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-50723236-blog-91875959.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-50723236-blog-91875959.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=16)

[[3] Mr_老冷, adb shell命令整理之service, 2016.](https://blog.csdn.net/mr_oldcold/article/details/53761759?spm=1001.2101.3001.6650.6&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-6-53761759-blog-91875959.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-6-53761759-blog-91875959.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=11)