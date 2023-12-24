---
title: "Android rc文件和解析"
date: 2023-06-24
thumbnailImagePosition: left
thumbnailImage: rc/rc_thumb.jpg
coverImage: rc/rc_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- rc
- 2023
- June
tags:
- Android
- 源码
- AIL
- parser
- Actions
- Commands
- Services
- Options
- Imports
showSocial: false
---

Android进程对应的文件有后缀为rc文件的，那么rc文件是什么又是怎么加载启动的呢

<!--more-->
# 0rc文件简介

init.rc由AIL语言编写而成。可以参考system/core/init/README.md来学习AIL语法相关知识。具体可以点击下面源码文件路径访问

不同Android版本关于AIL的说明存在一些细微差异，但基本语法和总的思路是不变的。往往我们可以先查看对应的system/core/init/README.md来了解这些差异。

Android Init Language由**五大类语句**组成：

- **Actions**
- **Commands**
- **Services**
- **Options**
- **Imports**

换行为语句分隔符。空格为词分隔符。反斜杠转义符可用于插入不做分隔的空格符。双引号用于包括不需要空格分隔的词。行尾反斜杠用于多行衔接为一行。行首以 # 开始用于注释。使用${var}可进行变量扩展。
一段内容往往以Actions 或者Services开头，并以Commands或者Options结尾。
Commands 或者Options用于修饰Actions 或者Services。（阅读时注意）
Service有唯一名称，如果已经定义过一个相同的Service，后定义的被忽略，且log中记录了此异常。

# 1init.rc

平时最常见的就是开机需要用到的rc文件。

一般设备系统中多个路径下都存在rc文件。一般规范的路径为：**/{system,vendor,odm}/etc/init/*.rc**。

根目录/init.rc为主rc文件，它是rc文件被加载的入口。它通过Imports语句将其他各个路径下的rc文件都加载进来。

## 1.1rc文件目录规范

| 路径                | 功能                                                         |
| ------------------- | ------------------------------------------------------------ |
| `/system/etc/init/` | 用于核心系统功能，例如`SurfaceFlinger`、`MediaService`和`logd`等 |
| `/vendor/etc/init/` | 用于芯片供应商的一些核心操作或者守护进程功能                 |
| `/odm/etc/init/`    | 用于设备制造厂商的功能，比如传感器和其他外围设备所需的操作或者守护进程 |

服务对应的二进制程序如果放在system|vendor|odm分区下，相应的定义服务的rc文件也需要在相同分区的etc/init/目录下。一般我们开发的时候rc和二进制程序是在相同的工程目录下，在Android.bp 是通过配置init_rc。

> 更改分区的方法
>
> - 默认安装到/system目录下
>
> - 安装到vendor
>
>   vendor: true
>
> - 安装到product
>
>   product_specific: true
>
> - 安装到odm
>
>   device_specific: true

```go
//logcatd开发的工程目录在system/core/logcat
//logcatd  logcatd.rc都在此工程目录下
//通过Android.bp指定init_rc 

//@system/core/logcat/Android.bp
sh_binary {
    name: "logpersist.start",
    src: "logpersist",
    init_rc: ["logcatd.rc"],
    required: ["logcatd"],
    symlinks: [
        "logpersist.stop",
        "logpersist.cat",
    ],
//最终编译出来的logcatd.rc就位于system/etc/init/logcatd.rc
$ cd out/target/product/${TARGET_PRODUCT}
$ find -name logcatd.rc
./system/etc/init/logcatd.rc
```

# 2语法详解

## 2.1Actions

Actions 是一系列commands的集合，同时包含一个trigger来决定是否要执行这一系列的命令。当触发器被触发时，Action就被加入执行队列(已在队列中则忽略)。每个队列中的Action是按顺序执行和移除的，而Action中的命令也是按顺序执行的。而其他的一些操作包括设备创建销毁，property的设置以及进程的重启是穿插在各个命令执行之间的。格式如下：

```rc
on <trigger> [&& <trigger>]*
   <command1>
   <command2>
   <command3>
```

同时被触发的Action将按照他们在rc文件中定义的先后顺序加入到执行队列中。例如：

```rc
on boot
   setprop a 1
   setprop b 2

on boot && property:test=true
   setprop c 1
   setprop d 2

on boot
   setprop e 1
   setprop f 2
```

如果当前启机且test属性为true，则执行顺序如下：

```rc
setprop a 1
setprop b 2
setprop c 1
setprop d 2
setprop e 1
setprop f 2
```

> Triggers 
>
> | Trigger                   | 含义                                      |
> | ------------------------- | ----------------------------------------- |
> | `bott`                    | 这是`init`程序启动后出发的第一个事件      |
> | `< name > = < value >`    | 当属性`< name >`满足特定`< value >`时触发 |
> | `device-added-< path>`    | 当设备节点添加/删除时触发此事件           |
> | `service-exited-< name >` | 当指定的服务`< name >`存在时触发          |
>
> 一系列明确的事件，用于触发action操作。触发器分为**事件触发器**和**属性触发器**
>
> - 事件触发器
>
>   由“trigger”命令或init程序中的QueueEventTrigger()函数触发的。它们一般采用简单字符串的形式，例如“boot”或“late init”
>
> - 属性触发器
>
>   由指定的Property更改时触发的。Property变更为某个指定的value，格式为property:<name>=<value>. Property只要有变更就触发，格式为property:<name>=\*
>
> 一个Action可以有多个属性触发器，但只能有一个事件触发器。
>
> ```rc
> #表示在开机并且 property:a=b时触发action。
> on boot && property:a=b
> ```



## 2.2Commands

| 名称                                                         | 含义                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `exec < path > [< argument >]*`                              | `Fork`并执行一个程序，其路径为`< path >`,这条命令将阻塞直到该程序启动完成，因此他有可能造成`init`程序在某个点不停的等待。 |
| `export < name >< value >`                                   | 设置某个环境变量`< name >`的值为`< value >`,这是对全局有效的，即其后的所有进程都将继承这个变量。 |
| `ifup < interface >`                                         | 使网络接口`< interface >`成功连接                            |
| `hostname < name >`                                          | 设置主机名为`< name >`                                       |
| `chdir < directory >`                                        | 更改工作目录为`< directory >`                                |
| `chmod < octal-modc >`                                       | 更改文件访问权限                                             |
| `chown < owner >< group >< path >`                           | 更改文件所有者和组群                                         |
| `chroot < directory >`                                       | 更改根目录位置                                               |
| `class_start < serviceclass >`                               | 启动由`< servicesclass >`类名指定的所有相关服务，如果他们不存在运行状态的话 |
| `class_stop < serviceclass >`                                | 停止所有由`< serviceclass >`指定的服务，如果他们当前正在运行的话 |
| `domainname < name >`                                        | 设置域名                                                     |
| `insmod < path >`                                            | 在`< path >`路径上安装一个模块                               |
| `mkdir < path >[mode][owner][group]`                         | 在`< path >`上新建一个目录                                   |
| `mount < type >< device >< dir >[mountoption]`               | 尝试在指定路径上挂载一个设备                                 |
| `setprop< name >< value >`                                   | 设置系统属性`< name >`的值`< value >`                        |
| `start < service >`                                          | 这个命令将启动一个服务，如果他没有处于运行状态的话           |
| `stop < service >`                                           | 这个命令将停止一个服务，如果他没有处于运行状态的话           |
| `exec [ <seclabel> [ <user> [ <group>\* ] ] ] -- <command> [ <argument>\* ]` | 使用给定的参数`Fork`并执行命令。该命令在“——”之后开始，以便提供可选的安全上下文、用户和补充组。在此命令完成之前，不会运行其他命令。`Seclabel`可以是`a -`来表示默认。属性在参数内展开。Init停止执行命令，直到`fork`进程退出。<br />exec -- /system/bin/bootstrap/linkerconfig --target /linkerconfig/bootstrap<br />exec - system system -- /system/bin/vdc checkpoint markBootAttempt |
| `symlink < target >< path >`                                 | 创建一个`< path >`路径的连接，目标为`< target >`             |
| `sysclktz`                                                   | 设置基准时间，如果当前时间是`GMT`,这个值为0                  |
| `trigger < event >`                                          | 触发一个事件                                                 |
| `write < path >< string >[ < string >]*`                     | 打开一个文件，并写入一个或多个字符串                         |

`mkdir <path> [<mode>] [<owner>] [<group>] [encryption=<action>] [key=<key>]`

创建目录，可以指定mode,owner和group. 如果不指定则按权限为755，root用户和root组创建。如果指定了，但目录已存在，则目录的权限和用户，组信息将被更新。

> action 可以指定为:
>
> - None: 不需要进行加密操作，目录根据上层目录加密算法进行处理。
> - Require: 需要加密，如果加密失败则终止开机进程。
> - Attempt: 尝试加密，加密失败仍可继续。
> - DeleteIfNecessary: 递归处理：如果存在子目录，但当前策略的加密算法不匹配，则删除子目录。
>
> key 可以指定为:
>
> - ref: 使用systemwide DE key
> - per_boot_ref: 使用每次开机都更新的key

`mount_all [ <fstab> ] [--<option>] `

 对给出的 fstab 表调用 fs_mgr_mount_all 接口，option为“early” 或者 “late”. option如果是--early则init将会跳过“latemount”标识的的item,并触发fs encryption state 事件。option如果是--late则init就只mount “latemount”标识的的item. 如果不携带[–]则mount的是fstab 表中全部item。 

```rc
on fs
      mount_all /vendor/etc/fstab.${ro.hardware} --early
  
on late-fs
      # Start services for bootanim
      start servicemanager
      start hwcomposer-2-1
      start gralloc-2-0
      start surfaceflinger
      start bootanim
  
      exec_start wait_for_keymaster
      # Mount RW partitions which need run fsck
      mount_all /vendor/etc/fstab.${ro.hardware} --late
```

调用到对应的fstab.${ro.hardware}

```shell
$ cat /vendor/etc/fstab.rk30board                                                                           # Android fstab file.
#<src>  <mnt_point> <type> <mnt_flags and options> <fs_mgr_flags>
# The filesystem that contains the filesystem checker binary (typically /system) cannot
# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK

/dev/block/by-name/system                          /                          ext4      ro,barrier=1                                                      wait,avb,slotselect
/dev/block/by-name/cache                           /cache                    ext4       noatime,nodiratime,nosuid,nodev,noauto_da_alloc,discard                wait,check
/dev/block/by-name/metadata                        /mnt/vendor/metadata       ext4       noatime,nodiratime,nosuid,nodev,noauto_da_alloc,discard                wait
/dev/block/by-name/misc                            /misc                      emmc       defaults                                                           defaults
/dev/block/by-name/frp                             /frp                       emmc       defaults                                                           defaults
/dev/block/by-name/parameter                        /parameter                 emmc      defaults                                                            defaults
/dev/block/by-name/baseparamer                    /baseparamer                emmc        defaults                                                            defaults
/dev/block/by-name/resource	                     /resource	                  emmc	     defaults				                              defaults
/devices/platform/*usb*                               auto                     vfat       defaults                                                        voldmanaged=usb:auto
/dev/block/zram0                                     none                      swap       defaults                                                        zramsize=50%
# For sdmmc
/devices/platform/fe320000.dwmmc/mmc_host*           auto                      auto           defaults                                                  voldmanaged=sdcard1:auto
# Full disk encryption has less effect on rk3399, so default to enable this.
/dev/block/by-name/userdata                         /data                      f2fs      noatime,nodiratime,nosuid,nodev,discard,inline_xattr,fsync_mode=strict     wait,check,notrim,forceencrypt=/cache/key_file,quota,reservedsize=128M
```



## 2.3Services

Service指的是需要在初始化就启动的服务，通过定义还可以让他们在退出的时候自动重启。格式如下：

```rc
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
```

举例说明

### 2.3.1无参数

```rc
#init.rc
service ueventd /system/bin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
```

### 2.3.2有参数

```rc
# logcatd service
service logcatd /system/bin/logcatd -L -b ${logd.logpersistd.buffer:-all} -v threadtime -v usec -v printable -D -f /data/misc/logd/logcat -r ${logd.logpersistd.rotate_kbytes:-2048} -n ${logd.logpersistd.size:-256} --id=${ro.build.id}
    class late_start
    disabled
    # logd for write to /data/misc/logd, log group for read from log daemon
    user logd
    group log
    task_profiles ServiceCapacityLow
    oom_score_adjust -600
```



## 2.4Options

Option 是修饰service的属性，它用于定义service何时启动以及怎样启动。

| **option**                                                   | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `critical`                                                   | 表明这个对设备至关重要的服务，如果他在四分钟内退出超过4次，则设备将重启进入`bootloader`模式 |
| `capabilities [ <capability>\* ]`                            | 表示执行该`service`的能力，如果没有增加`capabilities`修饰，则表示该服务没有任何`capabilities` |
| `console [<console>]`                                        | 用于指定该服务输出 是否要打印到`dmesg`。 默认输出到`/dev/console`, 而`/dev/console`可以通过设置`androidboot.console`来指定 |
| `timeout_period <seconds>`                                   | 超过这个时间kill掉`service`。`oneshot service`不会被重启，其他服务会自动重启。与`restart_period`配合使用可以创建周期运行的`service`。 |
| `disable`                                                    | 此服务不会自动启动，而是需要通过显示调用服务名来启动         |
| `file <path> <type>`                                         | 打开文件，并将它的`fd`传递给进程，C 层可使用`android_get_control_file`获取到`fd`。 type类为：`“r”`,` “w”` ,`“rw”` |
| `interface <interface name> <instance name>`                 | 关联`HIDL`接口，名称必须是完整的，这样才能方便`hwservicemanager`快速启动这些`HIDL`服务。可关联多个`HIDL `接口 |
| `keycodes <keycode> [ <keycode>\* ]`                         | 设置`keycode`，当同时触发这些按键事件时，服务将启动。常用于`bugreport service` |
| `setenv < name >< value >`                                   | 设置环境变量`< name >`为某个值`< value >`                    |
| `socket < name >< type >< perm >[< user >[ < group >]]`      | 创建一个名为`/dev/socket/< name >`的`Unix domain socket`，然后将他的`fd`值传给启动它的进程，有效的`< type >`值包括`dgram`,`steam`和`seqpacket`.而`user`和`group`的默认值是0 |
| `ioprio <class> <priority>`                                  | 通过`SYS_ioprio_set syscall`设置此`service`的IO优先级和IO优先级类。类为`“rt”`、`“be”`或`“idle”`。优先级为0-7 |
| `user < username >`                                          | 在启动服务前将用户切换至`< username >`,默认情况下用户都是`root` |
| `group < groupname >[< groupname >]`                         | 在启动服务将用户组切换至`< groupname >`                      |
| `oneshot`                                                    | 当此服务退出时，不要主动去重启他                             |
| `class < name >`                                             | 为该服务指定一个`class`名，同一个`class`的所有服务必须同时自动或者停止，默认情况下服务的`class`名是`”default”` |
| `onrestart`                                                  | 当此服务重启时，执行某些命令                                 |
| `enter_namespace <type> <path>`                              | 输入路径的namespace。network type可以指定为"net". 只能设置一个type |
| `namespace <pid|mnt>`                                        | `namespace` 是 `Linux` 内核用来隔离内核资源的方式。通过 `namespace` 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在 |
| `oom_score_adjust <value>`                                   | 自定义`oom_score`，等同于优先级，优先级越高值越小（-1000 ~1000）。分值越小越不会被kill |
| `override`                                                   | 一般是`odm service`使用，用于覆盖之前`system`或者`vendor`已经定义的服务配置。如果`rc`中存在多个`override`标识的相同类名服务，则最后一个被解析的服务才生效。可以通过`imports`分析，避免出错。 |
| `priority <priority>`                                        | 设置`service`进程的优先级。范围为-20~19. 默认为0. 实际是通过`setpriority()`函数设置 |
| `reboot_on_failure <target>`                                 | 当该服务启动失败或者运行中异常退出时,执行重启。 参数 其实就是设置`property sys.powerctl` 的值。格式一般为`"reboot,reason"`。 |
| `restart_period <seconds>`                                   | 设置后`non-oneshot` 服务将以此周期重启服务。比如`restart_period 3600` 表示该服务将会每隔1小时启动 |
| `rlimit <resource> <cur> <max>`                              | 对当前服务以及子进程生效。等同于`setrlimit`命令。`rootfs`,`ueventd`, `adbd`等，默认为`init`. |
| `shutdown <shutdown_behavior>`                               | 指定当关机时`service`进程行为。如果没有具体说明，关机时，使用`SIGTERM`和`SIGKILL`关闭`service`。关机期间，`“critical”` 标识的`service`不会被关闭直到关机超时。`“critical”` 标识的`service`在关机开始时，如果没处于运行状态，则它还会被启动 |
| `socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ]` | 创建一个名为`/dev/socket/name`的`UNIX`域`socket`，并将其`fd`传递给`service`程序。type必须为`“dgram”`、`“stream”`或`“seqpacket”`。type可以加后缀`“+passcred”`，启用`SO_PASSCRE`。用户和组默认值为0。`”seclabel’`指的是`selinux context`. 进程中C层可以使用 `libcutils ``android_get_control_socket()`来获取这个`fd` |
| `user <username>`                                            | 指定`user`，默认为`root`。早期要获取`Linux capabilities`,进程需要以`root`运行并请求`capabilities`，之后再修改成它期望的运行`uid`。通过`fs_config`有一种新的机制来替换早期的那种实现，现在允许厂商将程序的`Linux capabilities`添加到文件系统中 |
| `task_profiles <profile> [ <profile>\* ]`                    | 设置任务`profiles`。这是为了取代`writepid`来设置当前`service`的`profiles` |
| `writepid <file> [ <file>\* ]`                               | 将子进程的`pid`写入文件中，以支持`cgroup/cpuset`的使用。如果指定的不是`/dev/cpuset/`，但是`property ‘ro.cpuset.default’` 中设置了非空的`cpuset`名称 (比如： `‘/foreground’`)，那么`pid会`被写入`/dev/cpuset/cpuset_name/tasks`。 此`option`仅适用于子进程，如果要设置当前`service`的`task profiles`, 就需要使用`task_profiles`. |

- `capabilities [ <capability>\* ]` 表示执行该service的能力，如果没有增加capabilities修饰，则表示该服务没有任何capabilities。

  ```rc
    ###example1, BLOCK_SUSPEND：
    service vendor.audio-hal /vendor/bin/hw/android.hardware.audio.service.ranchu
        class hal
        user audioserver
        # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
        group audio camera drmrpc inet media mediadrm net_bt net_bt_admin net_bw_acct wakelock context_hub
        #有能力block系统休眠
        capabilities BLOCK_SUSPEND
        ioprio rt 4
        task_profiles ProcessCapacityHigh HighPerformance
        onrestart restart audioserver
        
    ###example2,NET_ADMIN NET_RAW:
    service ril-daemon /vendor/bin/hw/libgoldfish-rild
        class main
        user radio
        group radio cache inet misc audio log readproc wakelock
        #NET_ADMIN：执行和网络相关的一系列操作，例如接口配置，防火墙配置等等
        #NET_RAW: 使用RAW and PACKET socket接字绑定到任何地址进行透明代理
        capabilities BLOCK_SUSPEND NET_ADMIN NET_RAW
    
    ###example3,SYS_BOOT:
    service charger /system/bin/charger
        class charger
        seclabel u:r:charger:s0
        user system
        group system wakelock input
        #有能力进行reboot或者调用kexec_load
        capabilities SYS_BOOT
        file /dev/kmsg w
        file /sys/fs/pstore/console-ramoops-0 r
        file /sys/fs/pstore/console-ramoops r
        file /proc/last_kmsg r
  ```

  

- `class <name> [ <name>\* ]` 用于指定service的类名，相同的类名可以在action中配置成一起启动或者一起停止。 一个service至少有一个class,如果没有指定 class name， 默认class 为"default"。当然也可以指定多个class name， 多个class name便于service分组管理。

  ```rc
    ###example1, animation 
    service vendor.gralloc-3-0 /vendor/bin/hw/android.hardware.graphics.allocator@3.0-service
        interface android.hardware.graphics.allocator@3.0::IAllocator default
        #`animation` calss是指可以进行开机动画或者关机动画的服务组。
        #这些服务往往很早就运行或者在关机的最后阶段运行的
        #因此需要注意他们不能去打开/data下的文件，应该要保证/data分区即便卸载了也能正常运行。
        class hal animation
        interface android.hardware.graphics.allocator@3.0::IAllocator default
        user system
        group graphics drmrpc
        capabilities SYS_NICE
        onrestart restart surfaceflinger
        
    ###example2, core
    service shutdownanim /system/bin/bootanimation shutdown
        #core 一般是早启动，最晚结束的一些系统核心程序
        class core
        user graphics
        group graphics audio
        disabled
        oneshot
    	
    
    ###example3, main
    service wpa_supplicant /vendor/bin/hw/wpa_supplicant \
        /vendor/etc/wifi/wpa_config.txt
    #   we will start as root and wpa_supplicant will switch to user wifi
    #   after setting up the capabilities required for WEXT
    #   user wifi
    #   group wifi inet keystore
        interface android.hardware.wifi.supplicant@1.0::ISupplicant default
        interface android.hardware.wifi.supplicant@1.1::ISupplicant default
        interface android.hardware.wifi.supplicant@1.2::ISupplicant default
        interface android.hardware.wifi.supplicant@1.3::ISupplicant default
        #class main 一般为framework层进程
        class main
        socket wpa_wlan0 dgram 660 wifi wifi
        disabled
        oneshot
    
    
    ###example4, late_start
    service bugreport /system/bin/dumpstate -d -p -B -z \
            -o /data/user_de/0/com.android.shell/files/bugreports/bugreport
        #late_start: 一般在初始化最后阶段才需要启动的程序
        class late_start
        disabled
        oneshot
        keycodes 114 115 116
  ```

  

- `console [<console>]` 用于指定该服务输出 是否要打印到**dmesg**。 默认输出到/dev/console, 而/dev/console可以通过设置androidboot.console来指定。

  ```rc
  #@*.rc
  service recovery /sbin/recovery
      console
      seclabel u:r:recovery:s0
  #@BoardConfig_*.mk
   BOARD_KERNEL_CMDLINE := androidboot.wificountrycode=CN androidboot.hardware=rk30board androidboot.console=ttyFIQ0 firmware_class.path=/vendor/etc/firmware init=/init rootwait ro init=/init
  
  #@device 可以查看console相关属性：
  $ getprop |grep conso                                                                                                                                                               
  [init.svc.console]: [running]
  [ro.boot.console]: [ttyFIQ0]
  [ro.boottime.console]: [3672952642]
  [ro.consoleable]: [1]
  ```

  

- `critical`用于说明关键服务，当该服务4分钟内或者在开机完成前退出超过4次，将会导致设备进入bootloader模式。

  ```rc
  service ueventd /sbin/ueventd
        #指明是关键服务
        critical
        seclabel u:r:ueventd:s0
  ```

  

- `file <path> <type>` 打开文件，并将它的fd传递给进程，C 层可使用android_get_control_file获取到fd。 type类为：“r”, “w” ,“rw”。

  ```rc
  service charger /system/bin/charger
        class charger
        seclabel u:r:charger:s0
        user system
        group system wakelock input
        capabilities SYS_BOOT
        file /dev/kmsg w
        file /sys/fs/pstore/console-ramoops-0 r
        file /sys/fs/pstore/console-ramoops r
        file /proc/last_kmsg r
  ```

  

- `interface <interface name> <instance name>`关联**HIDL**接口，名称必须是完整的，这样才能方便hwservicemanager快速启动这些HIDL服务。可关联多个HIDL 接口。

  ```rc
  service wpa_supplicant /vendor/bin/hw/wpa_supplicant -Dnl80211 -iwlan0 -c/vendor/etc/wifi/wpa_supplicant.conf -g@android:wpa_wlan0
        interface android.hardware.wifi.supplicant@1.0::ISupplicant default
        interface android.hardware.wifi.supplicant@1.1::ISupplicant default
        interface android.hardware.wifi.supplicant@1.2::ISupplicant default
        interface android.hardware.wifi.supplicant@1.3::ISupplicant default
        socket wpa_wlan0 dgram 660 wifi wifi
        group system wifi inet
        oneshot
        disabled
  ```

  

- `keycodes <keycode> [ <keycode>\* ]` 设置keycode，当同时触发这些按键事件时，服务将启动。常用于bugreport service。

  ```rc
  service vendor.audio-hal /vendor/bin/hw/android.hardware.audio.service.ranchu
        class hal
        user audioserver
        # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
        group audio camera drmrpc inet media mediadrm net_bt net_bt_admin net_bw_acct wakelock context_hub
        capabilities BLOCK_SUSPEND
        ioprio rt 4
        task_profiles ProcessCapacityHigh HighPerformance
        onrestart restart audioserver
  ```

  

- `onrestart` 当该service重启时，运行命令

  ```rc
  # bugreport is triggered by holding down volume down, volume up and power
  service bugreport /system/bin/dumpstate -d -p -B -z \
            -o /data/user_de/0/com.android.shell/files/bugreports/bugreport
        class late_start
        disabled
        oneshot
        keycodes 114 115 116
  ```

  



## 2.5Imports

`import <path>`导入一个新的rc文件内容，如果path是个目录，则会导入该目录下所有rc文件内容，仅导入当前目录，不支持递归，不能导入子目录下的rc文件。

import关键字并不是一个命令，它遵循以下规则

    1. 开机时，加载根目录下的/init.rc 或者加载 `ro.boot.init_rc` 中指定的脚本。
    2. 在加载/init.rc之后mount {system,vendor,odm}的时候就加载对应的 /{system,vendor,odm}/etc/init/ 下的rc文件。

关于加载顺序

```rc
fn Import(file)
  Parse(file)
  for (import : file.imports)
    Import(import)

Import(/init.rc)
Directories = [/system/etc/init, /vendor/etc/init, /odm/etc/init]
for (directory : Directories)
  files = <Alphabetical order of directory's contents>
  for (file : files)
    Import(file)
```

1. /init.rc 先被解析加载，同时在它内部import的rc文件会被递归加载。
2. /system/etc/init/下的rc文件按名字的字母排序排序后加载进来，加载一个的同时递归加载import进来的rc文件。
3. 再重复2继续处理r /vendor/etc/init 之后  /odm/etc/init 的rc文件。



## 2.6一点点补充关于属性

`ctl.[<target>_]<command>` 以ctl打头的这个格式的属性能够触发init的响应，且该属性是只用于set而不用于read的。`<target>`是可选的，一般为一个service的option，只能指定一个option。`<commands>`只能为start restart stop 中的一个。这个属性的value就是对应的service名称。 

```rc
##就会运行logd的start option。
SetProperty("ctl.start", "logd")
#就会启动一个提供了 `aidl aidl_lazy_test_1`接口 的service 的start option. 		
SetProperty("ctl.interface_start", "aidl/aidl_lazy_test_1")
#按oneshort启动，如果进程挂掉不再重启。
SetProperty("ctl.oneshot_one_start", "logd") 
# 进程挂掉后还要重新启动。
SetProperty("ctl.oneshot_off_start", "logd")
#发送SIGSTO给该服务。用于调试。
SetProperty("ctl.sigstop_on_start", "logd")
SetProperty("ctl.sigstop_off_start", "logd")
```



# 源码文件

https://github.com/YangYang48/project/tree/master/rc

# 参考

[[1] 一切皆是定数, Android启动流程（四）——rc文件解析, 2023.](https://blog.csdn.net/weixin_49274713/article/details/130533998?spm=1001.2014.3001.5502)

[[2] astro-yang, init.rc语法详解, 2015.](https://blog.csdn.net/feigebangni/article/details/50300063)

[[3] 面包派, Android init.rc整理, 2022.](https://blog.csdn.net/panbinxian/article/details/123660864)

[[4] 时光如刀, Android 8.0 系统启动流程之init.rc语法规则(六), 2018.](https://blog.csdn.net/marshal_zsx/article/details/80272760)

[[5] 安德路, init.rc深入学习, 2017.](https://blog.csdn.net/ch853199769/article/details/78427362)

[[6] Eliot_shao, Init.rc妙用及语法说明, 2016.](https://blog.csdn.net/eliot_shao/article/details/53163522)

[[7] 疯人院的院长大人, Android系统init进程启动及init.rc全解析, 2022.](https://zhonglunshun.blog.csdn.net/article/details/78615980)

[[8] 疯人院的院长大人, recovery下的init.rc语法解析, 2017.](https://zhonglunshun.blog.csdn.net/article/details/78625375)

