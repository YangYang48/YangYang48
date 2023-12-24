---
title: "camera 水波纹"
date: 2023-12-24
thumbnailImagePosition: left
thumbnailImage: camera/flicker/flicker_thumb.png
coverImage: camera/flicker/flicker_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- camera
- 2023
- December
tags:
- isp
- sensor
- flicker
- banding
- 卷帘快门
- 曝光行
showSocial: false
math: true
---

本文主要记录关于camera水波纹或者说是flicker相关的概念和产生的原因，并说明如何去除flicker。

<!--more-->
{{< wyymusic 1377642003 >}}

# 0现象

Sensor banding现象（这种现象有时候也被成为Flicker现象）如视频所示，画面会出现频闪，感觉有水波纹一样的纹路在跳变；具体来说可能会有如下表现（这些表现并不一定会同时出现）：

- 同一帧的不同行的亮度各不相同，存在亮暗变化的条纹
- 不同帧的相同行的亮度不相同，出现视频中水波纹一样的纹路跳变
- 前后帧的整体亮度存在差异，画面亮度出现明显的亮暗变化

该现象对目前的相机来说是常见问题，但是理解起来有些困难，因此在这里对这个问题进行一个详细的解释。

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_1.png" thumbnail="/camera/flicker/flicker_1.png" title="">}}


# 1flicker说明

## 1.1交流电变化规律

交流电其实是一种正弦波，其频率有两种：50HZ和60HZ，中国、泰国、印度、大部分欧洲国家等地采用50HZ，美国、加拿大、墨西哥等地采用60HZ。下面以50HZ为例进行解释，交流电以1/50s，即20ms的周期进行变化，其变化规律如图所示：

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_2.png" thumbnail="/camera/flicker/flicker_2.png" title="">}}

对于波有正负之分，能量没有正负之分，因此能量周期为1/100s，即10ms。我国的以交流电为电源的白炽灯的亮度实际上在一直在以10ms为周期随着交流电变化而发生变化，只不过人眼感知不到画面亮度的变化罢了。

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_3.png" thumbnail="/camera/flicker/flicker_3.png" title="">}}

## 1.2快门

常见图像传感器Sensor有**卷帘快门**和**全局快门**之分

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_4.png" thumbnail="/camera/flicker/flicker_4.png" title="">}}

卷帘快门的特点是Sensor是**一行一行进行感光**的，并不是同时感光的，这就导致了不同行之间感光的时刻存在差异，而感光的过程可以看成是对亮度进行积分，**积分的大小直接决定了画面亮度**。

如果在感光过程外界灯光亮度发生了变化就可能导致不同行感应到的亮度有差异，导致了banding现象，下面具体进行分析。

全局曝光也不能幸免于时间调制的照明效果，它表现为一种曝光的“呼吸”，其中光脉冲可能与传感器的整合期同步（较亮），或与传感器的读出期同步（较暗）。具体现象是在视频中的帧之间会出现明暗闪烁的现象，同时这种闪烁的灯光现象也会混淆HDR多重曝光的图像融合。

| **卷帘曝光（Rolling Shutter）**  | **全局曝光（Global Shutter）** |
| :------------------------------: | :----------------------------: |
|          1.像素逐行曝光          |         1.像素同时曝光         |
|       2.适用于较高的像素数       |       2.比较适合运动物体       |
|  3.比较适合静物或低速运动的物体  |     3.运动物体没有图像畸变     |
| 4.对运动物体可能会造成畸变与拖影 | 4.增加噪声读出（相同曝光时间） |
|   5.帧速调节与噪声控制相对灵活   |                                |



## 1.3曝光分析

### 1.3.1曝光时间的控制

因为Sensor本身并没有时间的概念，它是通过pixel clock数和pixel clock的频率来表示sensor曝光时间的。为了得到简便的表达方式，就用曝光行数表示曝光时间了。曝光行数=pixel clock数/每条line的pixel数。 只需要知道sensor的pixel clock频率和每行的pixel数（有效pixel+dummy pixel），便可以计算出任何曝光时间，sensor需要曝光多少行。

### 1.3.2sensor曝光过程：

 如下图过程可见，曝光是逐个像素从上往下进行的，曝光行为蓝色，未曝光的是白色，完成曝光后也是白色。可以看到整个画面以12*8的格子来说明，开始曝光行为5行，随着时间的推移，逐个往后往下曝光的过程，也就是曝光行的移动过程，也说明sensor的曝光时间是5条lines。

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_5.png" thumbnail="/camera/flicker/flicker_5.png" title="">}}

传统的相机模组使用行曝光的方式来进行成像，但由于交流电是呈周期性的，所以在拍摄灯光的时候可能会出现flicker现象

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_6.gif" thumbnail="/camera/flicker/flicker_6.gif" title="">}}

不会产生flicker现象，那么需要每次正选内的积分大小一致，即10ms的整数倍

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_7.gif" thumbnail="/camera/flicker/flicker_7.gif" title="">}}

## 1.4 同一帧内的不同行分析

下面所述的曝光时间即为上面1.3.2中的曝光行

### 1.4.1曝光时间为10ms时

如图所示，假如第M行和第N行分别在tm和tn时刻开始曝光，曝光时间为都10ms，图中阴影部分的面积就表示该行的亮度

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_8.png" thumbnail="/camera/flicker/flicker_8.png" title="">}}

我们知道第M行的积分面积与第N行的积分面积是相同的，因为积分时间刚好是周期的整数倍，此时不同行的亮度是相同的，不会产生banding现象

### 1.4.2曝光时间为8ms时

如图所示，假如第M行和第N行分别在tm和tn时刻开始曝光，曝光时间为都8ms，图中阴影部分的面积就表示该行的亮度

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_9.png" thumbnail="/camera/flicker/flicker_9.png" title="">}}

我们可以知道看出来第M行的积分面积（跨越波峰）大于第N行的积分面积（跨越波谷），因此这两行的亮度就会有差异，在当前帧中就会出现不同行亮度不同的水波纹现象。

### 1.4.3以30fs为例说明

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_10.gif" thumbnail="/camera/flicker/flicker_10.gif" title="">}}

- 10ms曝光状态下，面积P始终是不变的，这里的面积不是指的是曝光行面积，而是在工频电压下的正弦积分面积
- 8ms和12ms曝光状态下，面积P1和P2都会有规律的产生变化（ 因为曝光时间是8ms、12ms，积分面积不同，导致每行所获得的能量不同）。8ms 和12ms的曝光帧会出现banding现象

## 1.5对不同帧的同一行进行分析

假设，第M帧的第N行在t<sub>m</sub>时刻开始曝光，第M+1帧的第N行会在t<sub>m+1</sub>开始曝光，如果此时的帧率为`30FPS`，每帧时间为为1/30*s*﻿，即`33ms`。我们可以知道：
$$
t_{m+1} = t_{m} + 33ms
$$

### 1.5.1曝光时间为10ms时

如图所示，此时我们会发现，这两行的亮度是一样的，因此不同帧的亮度也是相同的，即画面亮度不会出现闪烁跳变

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_11.png" thumbnail="/camera/flicker/flicker_11.png" title="">}}

### 1.5.2曝光时间为8ms时

如图所示，第M帧的第N行的积分面积要大于第M+1帧的第N行的积分面积（经过谷底），此时不同帧的相同行亮度也会发生表现，加上上面的分析，我们可以得知，在30FPS情况下，以8ms进行曝光时**不仅会出现水波纹，而且水波纹还会滚动**，出现上面视频中的表现。

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_12.png" thumbnail="/camera/flicker/flicker_12.png" title="">}}

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_14.gif" thumbnail="/camera/flicker/flicker_14.gif" title="">}}

但是，如果此时以25FPS的帧率进行分析，每帧为1/25s，即40ms时，情况会变得不一样，此时：
$$
t_{m+1} = t_{m} + 40ms
$$
{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_13.png" thumbnail="/camera/flicker/flicker_13.png" title="">}}

{{< image classes="fancybox center fig-100" src="/camera/flicker/flicker_15.gif" thumbnail="/camera/flicker/flicker_15.gif" title="">}}

如图所示，由于40ms为周期10ms的整数倍，因此这两行的起始时刻相位是相同的，所以两行的积分面积是相同的，因此此时会出现水波纹现象但不会出现画面闪烁。



### 1.5.3总结

这里可以给出结论：

- 当曝光时间为光源能量周期的整数倍时，不会出现banding现象；
- 当曝光时间不为光源能量周期的整数倍时，一定会出现不同行之间的亮度差异，即水波纹一样现象；但是水波纹会不会上下滚动还要看帧率；
- 当每帧时间(1/fps)为光源能量周期的整数倍时，不同帧之间的相同行不会出现亮度变化，即哪怕有水波纹也不会滚动，；反之水波纹会上下滚动；

# 2如何避免

只要曝光时间是光能量周期（100Hz）的整数倍，即可规避工频干扰导致的闪烁问题。

但是当曝光时间低于光能量周期（10ms）时，有应该如何规避这个问题呢？

以50Hz为例说明，实现这个有两种办法：

1. 设置曝光控制，强制为10ms整数倍变化，但是这样会浪费一部分曝光时间，导致曝光无法用满，在室内自然就会损失性能。
2. 修改帧率，使每帧图像分到的时间是10ms的整数倍，则可以用满每帧曝光时间在，室内效果更好。修改帧率可以插入Dummy Line或者Dummy Pixel。这需要一点点计算，具体计算需要看sensor输出Timing。


例如把帧率设置为7.14fps，则每帧曝光时间是140ms。如果是15fps，则每帧曝光时间是66.66ms，如果强制曝光为10ms整数倍，最大即60ms，则有6.66ms无法参与曝光，损失性能。

> 具体调整帧率方法得和sensor或者isp的FAE沟通，每个sensor都可能不一样，不能一概而论。调整帧率还有个原则要注意，预览一般不能低于 10fps，再低就很卡，常用14.28fps和12.5fps；抓拍不能低于5fps，否则用手就很难拍出清晰的照片，常用7.14fps。





# 参考

[[1] 木 东. 关于 Sensor flicker/banding现象的解释, 2022.](https://blog.csdn.net/qq_35247586/article/details/125118763?app_version=6.2.2&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22125118763%22%2C%22source%22%3A%22unlogin%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app)