---
title: "Battery Historian简单搭建"
date: 2023-02-21
thumbnailImagePosition: left
thumbnailImage: AndroidTools/BatteryHistorian/batteryhistorian_thumb.jpg
coverImage: AndroidTools/BatteryHistorian/batteryhistorian_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- Battery Historian
- 2023
- February
tags:
- go
- Android
- 功耗优化
- bugreport
showSocial: false
---

Battery Historain是谷歌开发的Android耗电量分析工具，其开发语言为go语言。使用这个工具可以将bugreport内容通过可视化界面更加直观的展示出来。

<!--more-->
# 1介绍

battery historian是一款用于检测与电池有关的信息和事件的工具，运行在Android 5.0 Lollipop (API level 21)及其之后。它会生成一张具有时间坐标的图纸，用户可以查看各种事件耗电时间。本文使用的是win10操作，没有在Linux操作系统中依托docker搭建。

# 2自建Battery Historain环境

## 2.1配置环境

需要配置jdk（openjdk），go（golang），python，git（docker）环境

- openjdk下载地址，点击[这里](https://mirrors.huaweicloud.com/openjdk/)
- golang下载地址，点击[这里](https://golang.google.cn/dl/)
- python下载地址，点击[这里](https://www.python.org/ftp/python/)
- git下载地址，点击[这里](https://git-scm.com/)
- docker下载地址，点击[这里](https://docs.docker.com/desktop/windows/install/)

go的环境变量配置如下：


{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_2.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_2.png" title="">}}

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_3.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_3.png" title="">}}

环境配置完成之后，目前工程下是空的。

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_4.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_4.png" title="">}}

## 2.2安装Battery Historain

环境配置结束后，安装Android工具Battery Historain。

### 2.2.1开始安装

```shell
# 后面有三个点
go get -d -u github.com/google/battery-historian/...
```

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_5.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_5.png" title="">}}

实际上不能完全下载下来，这个是因为不能够下载对应go的protobuf工程。

### 2.1.2下载依赖包

必须要到battery-historian工程目录下，执行命令

```shell
go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go
```

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_6.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_6.png" title="">}}

### 2.1.3执行 go run setup.go命令

```shell
// 进入battery-historian目录
cd battery-historian
// 执行setup.go
go run setup.go
```

这个过程，主要用于执行脚本，下载三方插件，包括flot-axislabels、closure-compiler和closure-library

如果下载不下来，或者有类似问题，可以直接使用离线下载，然后再把三个包放入对应的目录下即可

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_7.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_7.png" title="">}}

如果其中有对应js下载不下来，可能需要科学上网。

直到出现下面这张图，才是完成

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_8.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_8.png" title="">}}

### 2.1.4启动battery-historian.go

```shell
// 启动battery-historian.go
go run cmd/battery-historian/battery-historian.go
```

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_10.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_9.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_9.png" title="">}}

# 3 使用Battery Historain

## 3.1上传bugreport

低android版本，生成bugreport格式

```
adb shell bugreport > bugreport.zip
```

将zip文件上传，等待工具分析，列出可视化界面，下图为红米3，系统为android5.0

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_12.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_12.png" title="">}}

高版本android版本，生成bugreport模式

```shell
adb shell bugreportz
```

执行这个命令之后，原本的目录会发生变化，目前笔者使用红米5+，系统为android7.0

原本的在手机目录中的 bugreports文件会变成一个专门存放bugreport的目录

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_14.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_14.png" title="执行bugreportz之前的手机目录">}}

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_15.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_15.png" title="执行bugreportz之后的手机目录">}}

具体导入该zip文件，即为bugreport文件

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_13.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_13.png" title="">}}

参数含义

| Battery Historain参数  | 含义                                     |
| :--------------------: | ---------------------------------------- |
|       CPU runing       | cpu运行的状态                            |
|   Kernel only uptime   | 只有kernell运行                          |
|   Userspace wakelock   | 用户空间申请的锁                         |
|         Screen         | 屏幕是否点亮                             |
|        Top app         | 当前在内存中的应用，按内存占用率排序     |
| Activity Manager Proc  | 活跃的用户进程                           |
|    Crashes(logcat)     | 某个时间点出现crash的应用                |
|          Doze          | 是否进入doze模式 Device                  |
|         active         | 和Doze相反                               |
|      JobScheduler      | 异步作业调度                             |
|      SyncManager       | 同步操作                                 |
|    Temp White List     | 电量优化白名单                           |
|       Phone call       | 是否打电话                               |
|          GPS           | 是否使用GPS                              |
|  Network connectivity  | 网络连接状态（wifi、mobile是否连接）     |
| Mobile signal strength | 移动信号强度（great\good\moderate\poor） |
|       Wifi scan        | 是否在扫描wifi信号                       |
|    Wifi supplicant     | 是否有wifi请求                           |
|       Wifi radio       | 是否正在通过wifi传输数据                 |
|  Wifi signal strength  | wifi信号强度                             |
|      Wifi running      | wifi组件是否在工作(未传输数据)           |
|        Wifi on         | 同上                                     |
|         Audio          | 音频子系统                               |
|         Camera         | 相机是否在工作                           |
|         Video          | 是否在播放视频                           |
|   Foreground process   | 前台进程                                 |
|    Package install     | 是否在进行包安装                         |
|     Package active     | 包管理在工作                             |
|     Battery level      | 电池当前电量                             |
|      Temperature       | 电池温度                                 |
|        Plugged         | 连接usb或者充电                          |
|      Charging on       | 在充电                                   |
|      Logcat misc       | 是否在导出日志                           |

本文篇幅内容不对具体分析做介绍

## 3.2Android性能优化之电量篇

特效篇：Android性能优化之电量篇 - http://hukai.me/android-performance-battery/

```text
// 得到整个设备的电量消耗信息
$ adb shell dumpsys batterystats > xxx.txt
// 得到指定app相关的电量消耗信息
$ adb shell dumpsys batterystats > com.package.name > xxx.txt

// 通过Google编写的python脚本把数据信息转换成可读性更好的html文件
// https://github.com/google/battery-historian/blob/master/scripts/historian.py
$ python historian.py xxx.txt > xxx.html  
```




# 4目前遇到的问题

1.执行Android Historian脚本的时候，出现下面类似提示

```text
go: go.mod file not found in current directory or any parent directory; see 'go help modules'
```

go 的环境配置相关，设置GO111MODULE=auto

```go
//依次执行
go env -w GO111MODULE=auto
go mod init
go mod verify
go mod vendor
```

1.导入bugreport之后，然后运行Android Historian脚本，启动的网页UI没有很好的适配

需要科学上网，有部分js没有正确的下载下来

2.导入bugreport之后，然后运行脚本，打开网页全是字符串，正常的UI界面都没有显示

可能是导入的bugreport方式有误，例如android11部分机型，需要点击开发者模式中的错误报告，然后把对应手机侧的/bugreport目录下对应的bugreport重新导入，可以重新显示出来

3.执行对应的Android Historian脚本有大量的语法错误，类似

```text
/work/src/github.com/google/battery-historian/third_party/closure-library/closure/goog/streams/full_test.js:492: ERROR - Parse error. '(' expected
```

这个需要回退三方插件的版本号，将**closure-library**这个仓库回退到20170409稳定版本

可以用下面两种方式回退

第一种直接在官方下载对应版本，然后将版本对应替换到目标的三方插件目录下，官网地址是，点击[这里](https://github.com/google/closure-library/releases)，并找到对应的v20170409的版本

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_1.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_1.png" title="">}}

第二种直接通过git版本控制回退当前插件代码

```shell
cd third_party/closure-library/
git reset --hard v20170409
cd -
go run setup.go
```



# 3总结

总的来说，这是一种新的可视化方式，来解读Android的功耗相关。除了上述方式之外，如果还是不能够自建Android Historian工具，可以尝试其他的`Battery Historain`线上环境，效果也是一样的，点击[这里](https://bathist.ef.lc/)。

{{< image classes="fancybox center fig-100" src="/AndroidTools/BatteryHistorian/batteryhistorian_11.png" thumbnail="/AndroidTools/BatteryHistorian/batteryhistorian_11.png" title="">}}

# 参考

[[1] weixin_30871905, Android Historian安装使用,2019.](https://blog.csdn.net/weixin_30871905/article/details/96898782)

[[2] 东名夜雨, closure-library 第三章 Closure基本库,2012.](https://blog.csdn.net/wswqiang/article/details/7197696)

[[3] ,AaronDDD go安装proto、grpc、protobuf等工具失败,2021.](https://blog.csdn.net/sinat_35162715/article/details/120828930)

[[4] 普通网友, 【Go报错】go go.mod file not found in current directory or any parent directory 错误解决,2022.](https://blog.csdn.net/m0_67401417/article/details/126080567)

[[5] 小木箱, 功耗优化 · 入门篇 · 浅析Android耗电量优化,2023.](https://mp.weixin.qq.com/s/4AcV0oi_xIad0SJTodTuLQ)

[[6] zeqiao, 电量分析工具 Battery Historian 的配置及使用 ,2017.](https://blog.csdn.net/zeqiao/article/details/77504477)

[[7] bjxiaxueliang, Mac 中 Battery Historain 安装与使用,2021.](https://blog.csdn.net/xiaxl/article/details/117758299)

[[8] 小米修修, Mac上安装Battery Historain遇到的问题,2020.](https://blog.csdn.net/wei_ada/article/details/106127654)