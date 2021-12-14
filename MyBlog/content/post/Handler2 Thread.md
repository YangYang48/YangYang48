---
title: "Handler2 Thread"
date: 2021-08-15
thumbnailImagePosition: left
thumbnailImage: handler/handler2_thumb.jpg
coverImage: handler/handler2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- Handler
- 2021
- August
tags:
- Android
- 源码
- framework
showSocial: false
---
Android是基于事件驱动的，即所有的不管是Activity、Service生命周期都是通过handler事件驱动的。
那么Handler内部的线程是如何管理而且还能够保证安全。

<!--more-->
# Handler2 Thread

## 开篇问题

1.子线程维护的Looper，消息队列无消息的时候，处理方法是什么

2.如何保证Handler的内部线程安全

3.Handler中message的线程

4.Handler中Looper死循环会导致应用卡死吗





# 问题回答

1.子线程维护的Looper，消息队列无消息的时候，处理方法是什么

2.如何保证Handler的内部线程安全

3.Handler中message的线程

4.Handler中Looper死循环会导致应用卡死吗



# 参考





# 猜你想看

[Handler1 Looper](https://yangyang48.github.io/2021/08/handler1-looper/)

[Handler3 同步屏障](https://yangyang48.github.io/2021/08/handler3-同步屏障/)

[Handler4 HandlerThread](https://yangyang48.github.io/2021/08/handler4-handlerthread/)

[Handler5 IntentService](https://yangyang48.github.io/2021/08/handler5-intentservice/)