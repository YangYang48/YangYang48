---
title: "Vsync信号机制浅析"
date: 2022-05-31
thumbnailImagePosition: left
thumbnailImage: vsync/vsync_thumb.jpg
coverImage: vsync/vsync_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- Vsync
- 2022
- May
tags:
- Surfaceflinger
- Android
- performance
- Choreographer
- 源码
showSocial: false
---
在前面的学习中，了解到了实际上是存在硬件的Vsync信号和软件的Vsync信号，那么这两个信号是怎么工作的呢

<!--more-->
Vsync信号机制浅析







# 参考

[[1] SwallowJoe, Android UI架构(六)--探秘刷新动力Vsync(2)之DispSync.md, 2020.](https://blog.csdn.net/u014535072/article/details/106482313/)

[[2] wbo4958, Android SurfaceFlinger SW Vsync模型, 2017.](https://www.jianshu.com/p/d3e4b1805c92)

[[3] 泡面先生_Jack, Vsync同步机制 一, 2020.](https://www.jianshu.com/p/a5632f962608)

[[4] 泡面先生_Jack, Vsync同步机制 二, 2020.](https://www.jianshu.com/p/526e28bfb2c5)

[[5] 小翼龙守护者, Android中VSync信号的生成, 2022.](https://zhuanlan.zhihu.com/p/470378955)

[[6] 释然小师弟, 深入研究源码：DispSync详解, 2019.](https://juejin.cn/post/6844903986194022414)

[[7] Gracker, Android Systrace 基础知识 - Vsync 解读, 2019.](https://www.androidperformance.com/2019/12/01/Android-Systrace-Vsync/#/%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E7%9B%AE%E5%BD%95)

[[8] yjy239, Android 重学系列 Vsync同步信号原理, 2020.](https://www.jianshu.com/p/82c0556e9c76)

[[9] sheldon_blogs, Android 显示系统：Vsync机制, 2019.](https://www.cnblogs.com/blogs-of-lxl/p/11443693.html)

[[10] 积木zz, 又卡了～从王者荣耀看Android屏幕刷新机制, 2021.](https://mp.weixin.qq.com/s/q4Xjy7snARM8R9LZ5nxu5g)

[[11] houliang120, Android垂直同步信号VSync的产生及传播结构详解, 2016.](https://blog.csdn.net/houliang120/article/details/50908098)

[[12] 王小二的Android站, [070]一文带你看懂Vsync Phase, 2021.](https://www.jianshu.com/p/637c3ac93df3)

[[13] windrunnerlihuan, Android SurfaceFlinger 学习之路(五)----VSync 工作原理, 2017.](https://windrunnerlihuan.com/2017/05/25/Android-SurfaceFlinger-%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-%E4%BA%94-VSync-%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)