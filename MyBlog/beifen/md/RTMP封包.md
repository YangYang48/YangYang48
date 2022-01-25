---
title: "RTMP封包"
date: 2021-07-25
thumbnailImagePosition: left
thumbnailImage: unicode_thumb.jpg
coverImage: unicode_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- RTMP
- 2021
- July
tags:
- Android
- 音视频
---
RTMP简介

RTMP是Real Time Messaging Protocol（实时消息传输协议）的首字母缩写。该协议基于TCP，是一个协议族，包括RTMP基本协议及RTMPT、RTMPS、RTMPE等多个变种协议。
<!--more-->
RTMP是一种被设计用来进行实时数据通信的网络协议，主要用在Flash平台和支持RTMP协议的流媒体/交互服务器之间进行音视频和数据通信。支持该协议的软件包括Adobe Media Server、Ultrant Media Server、Red5等。 **RTMP**是目前主流的流媒体传输协议，广泛应用于直播领域，可以说市面上绝大多数的直播产品都采用了这个协议。不过这个基于TCP协议，所以基于在弱网环境丢包率高的情况下问题明显。  



针对编码过的音视频进行封包，音频为aac格式，视频为H264格式



本文使用的是[RTMP dump](https://blog.csdn.net/leixiaohua1020/article/details/14229047)包使用NDK在JNI层调用RTMPDump来完成RTMP通信 。 

官网下载地址http://rtmpdump.mplayerhq.hu/download/

![image-20210724114159108](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210724114159108.png)

源码在librtmp文件夹，可以看到librtmp的目录结构形式

![image-20210724113945338](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210724113945338.png)

CMakeLists.txt里面的内容不多，这样可以不进行预编译直接放在as里调试  

```cmake
#关闭ssl 不支持rtmps
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DNO_CRYPTO"  )
#所有源文件放入 rtmp_source 变量
file(GLOB rtmp_source *.c)

#编译静态库
add_library(
             rtmp
             STATIC
            ${rtmp_source} )
```

在AS中复制librtmp置于: src/main/cpp/librtmp ，并为其编写CMakeLists.txt  

```cmake
cmake_minimum_required(VERSION 3.4.1)

add_subdirectory(librtmp)
add_library(
             Auvirtmp
             SHARED
        	 Auvirtmp_hard.cpp)

target_link_libraries(
               		   Auvirtmp
                       rtmp
                       log)
```

编写完CMakeLists.txt ，在同级目录新建Auvirtmp_hard.cpp，目录结构大致是这样

![image-20210724125321751](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210724125321751.png)



RTMP视频body结构

|    字段    | 字节 | 描述                                          |
| :--------: | :--: | --------------------------------------------- |
|    类型    |  1   | 0x08音频/ 0x09视频/ 0x12脚本（描述信息）      |
|  数据大小  |  3   | 数据区大小，不包括包头                        |
|   时间戳   |  3   | 当前相对时间戳，单位毫秒。相对于第一个Tag数据 |
| 时间戳拓展 |  1   | 如果大于0xffffff，则会存在该字节              |
|    流ID    |  3   | 总为0                                         |
|   数据区   |  n   | 数据包                                        |

视频数据

|  帧类型  | 占位 | 描述                          |
| :------: | :--: | ----------------------------- |
|  帧类型  |  4   | 1 关键帧 2普通帧              |
|  编码ID  |  4   | 7 高级视频编码AVC             |
| 视频数据 |  n   | AVC则需要下面的AVCVIDEOPACKET |

AVCVIDEOPACKET

|   字段   | 字节 | 类型                                 |
| :------: | :--: | ------------------------------------ |
|   类型   |  1   | 0：AVC序列头/1：其他单元（其他NALU） |
| 合成时间 |  3   | 对于AVC序列头，全为0                 |
|   数据   |  n   | 类型不同，数据不同                   |



H264的起始码为0x00000001或者0x000001

下表罗列后五位值得具体含义

| nal_unit_type | NAL类型               |   c   |
| :-----------: | --------------------- | :---: |
|       0       | 未使用                |       |
|       1       | 不分区、非IDR图像的片 | 2,3,4 |
|      211      | 片分区A               |   2   |
|      123      | 片分区B               |   3   |
|       4       | 片分区C               |   4   |
|       5       | IDR图像的片           |  2,3  |
|       6       | 补充增强信息单元SEI   |   5   |
|       7       | 序列参数集            |   0   |
|       8       | 图像参数集            |   1   |
|       9       | 分界符                |   6   |
|      10       | 序列结束              |   7   |
|      11       | 码流结束              |   8   |
|      12       | 填充                  |   9   |
|     13-23     | 保留                  |       |
|     24-31     | 未使用                |       |

AVC序列头

在AVCVIDEOPACKACKET中如果类型为0，则后续数据为：

|         类型         | 字节 | 说明                                       |
| :------------------: | :--: | ------------------------------------------ |
|         版本         |  1   | 0x01                                       |
|       编码规格       |  3   | sps[1]+sps[2]+sps[3]                       |
| 几个字节表示NALU长度 |  1   | 0xff，包长为(0xff & 3) + 1，即用四字节表示 |
|       sps个数        |  1   | 0xe1，个数为0xe1 & 0x 1f ，即为1           |
|       sps长度        |  2   | 整个sps长度                                |
|       sps内容        |  n   | 整个sps                                    |
|       pps个数        |  1   | 0x01                                       |
|       pps长度        |  2   | 整个pps长度                                |
|       pps内容        |  n   | 整个pps                                    |

其他在AVCVIDEOPACKET类型为1，即非AVC序列头，则数据

| 类型 |      字节       | 说明            |
| :--: | :-------------: | --------------- |
| 包长 | 由AVC序列头定义 | 后续长度，4字节 |
| 数据 |        n        | H264视频数据    |

音频的TagHeader

|   类型   | 字段 | 说明                                             |
| :------: | :--: | ------------------------------------------------ |
| 音频格式 |  4   | 02 mp3/ 03 Linear PCM/ ..10 AAC                  |
|  采样率  |  2   | 00 5.5KHz/ 01 11KHz/ 02 22KHz/ 03 44KHz,AAC总为3 |
| 采样长度 |  1   | 00 send8bit/ 01 send16bit 压缩过后的都是16bit    |
| 音频类型 |  1   | 00 sendMono/ 01 sendStereo,AAC总为1              |





Audio specific config

|         类型         | 字段 | 说明                               |
| :------------------: | :--: | ---------------------------------- |
|   audioObjectType    |  5   | 编码结构类型， 02 AAC-LC           |
| sampleFrequencyIndex |  4   | 音频采样索引值，04 44100/ 08 16000 |
| channelConfiguration |  4   | 音频输出声道 01 单声道/ 02 双声道  |
|   GASpecificConfig   |      | 总共3字节，包括以下三项            |
|   frameLengthFlag    |  1   | 标志位，用于表明IMDCT窗口长度，0   |
|  dependsOnCoreCoder  |  1   | 标志位，表明是否依赖corecoder，0   |
|    extensionFlag     |  1   | 选择AAC-LC，必须为0                |



# 参考文献

[[1] 雷霄骅, RTMPdump 使用说明，2013.](https://blog.csdn.net/leixiaohua1020/article/details/14229047)

[[2] 壹号T馆, Android平台下RTMPDump的使用,2017.](https://www.jianshu.com/p/3ee9e5e4d630)

[[3] lory17, RTMP推流及协议学习, 2017.](https://blog.csdn.net/lory17/article/details/61916351)

[[4] 视界音你而不同, 手撕RTSP协议系列（1）——Rtsp基本流程, 2020](https://blog.csdn.net/mlfcjob/article/details/109119694?spm=1001.2014.3001.5501)

[[5] DoubleLi, AAC_LC用LATM封装header信息解析 Audio Specific Config格式分析, 2017.](https://www.cnblogs.com/lidabo/p/7235039.html)
