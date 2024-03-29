---
title: "操作系统学习笔记 8CPU调度"
date: 2022-07-17
thumbnailImagePosition: left
thumbnailImage: os/os8_thumb.jpg
coverImage: os/os8_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- OS
- 2022
- July
tags:
- 学习笔记
- 清华大学陈渝
showSocial: false
---
熟悉完操作系统的第七篇章，开始学习第八篇章，关于操作系统的CPU调度。

<!--more-->
# 0猜你喜欢

操作系统系列文章

[操作系统学习笔记 1概述](https://yangyang48.github.io/2022/06/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-1%E6%A6%82%E8%BF%B0/)

[操作系统学习笔记 2操作系统介绍](https://yangyang48.github.io/2022/06/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E4%BB%8B%E7%BB%8D/)

[操作系统学习笔记 3内存管理](https://yangyang48.github.io/2022/06/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-3%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)

[操作系统学习笔记 4非连续内存分配](https://yangyang48.github.io/2022/06/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-4%E9%9D%9E%E8%BF%9E%E7%BB%AD%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D/)

[操作系统学习笔记 5虚拟内存](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-5%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98/)

[操作系统学习笔记 6页面置换算法](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-6%E9%A1%B5%E9%9D%A2%E7%BD%AE%E6%8D%A2%E7%AE%97%E6%B3%95/)

[操作系统学习笔记 7进程管理](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-7%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/)

[操作系统学习笔记 8CPU调度](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-8cpu%E8%B0%83%E5%BA%A6/)

[操作系统学习笔记 9同步&互斥](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-9%E5%90%8C%E6%AD%A5%E4%BA%92%E6%96%A5/)

[操作系统学习笔记 10信号量&管程](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-10%E4%BF%A1%E5%8F%B7%E9%87%8F%E7%AE%A1%E7%A8%8B/)

[操作系统学习笔记 11死锁](https://yangyang48.github.io/2022/07/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-11%E6%AD%BB%E9%94%81/)

[操作系统学习笔记 12进程间通信](https://yangyang48.github.io/2022/08/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-12%E8%BF%9B%E7%A8%8B%E9%97%B4%E9%80%9A%E4%BF%A1/)

[操作系统学习笔记 13文件系统](https://yangyang48.github.io/2022/08/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-13%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/)



# 8CPU调度

1. 背景

  CPU调度

  CPU调度时间

2. 调度准则

3. 调度算法

4. 实时调度

5. 多处理器调度

6. 优先级反转



## 8.1背景

### 8.1.1CPU调度含义

**上下文切换**

- 切换CPU的当前任务，从一个进程/线程到另一个
- 保存当前进程/线程在PCB/TCB中的执行上下文（CPU状态）
- 读取下一个进程/线程的上下文

**CPU调度**

- 从就绪队列中挑选一个进程/线程作为CPU将要运行的下一个进程/线程
- 调度程序：挑选进程/线程的内核函数（通过一些调度策略）
- 什么时候进行调度



Q：在进程/线程的生命周期中的什么时候进行调度

A：从一个状态到另外一个状态切换的时候，会进行CPU的调度，主要是一些运行相关的状态变化。（就绪态--->运行态、运行态--->等待态、运行态--->退出态）

{{< image classes="fancybox center fig-100" src="/os/os8_1.png" thumbnail="/os/os8_1.png" title="">}}

内核运行调度程序的条件（满足一条即可）

- 一个进程从运行态切换到等待态
- 一个进程被终结了



### 8.1.2进程抢占

#### 8.1.2.1用户态进程抢占

**不可抢占**

- 调度程序必须等待事件结束

**可以抢占**

- 调度程序在中断被响应后执行
- 当前的进程从运行切换到就绪，或者一个进程从等待切换到就绪
- 当前运行的进程可以被换出



#### 8.1.2.2内核态进程抢占

上述抢占和不可抢占指的是用户态进程，当一个用户进程执行系统调用之后，如果系统调用在内核中不会让进程处于等待状态，系统调用返回之后一定会当前发起这个系统调用的进程，即**内核中不会出现抢占现象**（不可抢占式内核）。

如果在内核中允许抢占，有可能在内核中系统调用中，特殊事件的产生需要在内核中去切换，让优先级更高的进程去执行。这个时候，**从内核态返回到用户态的时候会返回到另一个进程去**（抢占式内核）。



## 8.2调度原则

### 8.2.1调度策略

- 调度策略

  确定如何从就绪队列中选择下一个执行进程

- 调度策略要解决的问题

  挑选就绪队列中的哪一个进程？

  通过什么样的准则来选择？

- 调度算法

  在调度程序中实现的调度策略

- 比较调度算法的准则

  哪一个策略/算法较好？



### 8.2.2程序执行模型

执行模型

程序在CPU突发和I/O中交替

- 每个调度决定都是关于在下一个CPU突发时将哪个工作交给CPU
- 在时间分片机制下，线程可能在结束当前CPU突发前被迫放弃CPU

{{< image classes="fancybox center fig-100" src="/os/os8_2.png" thumbnail="/os/os8_2.png" title="">}}



### 8.2.3评价指标

- **CPU使用率**

  CPU处于忙状态所占时间的百分比

- **吞吐量**

  在单位时间内完成的进程数量

- **周转时间**

  一个进程从初始化到结束，包括所有等待时间所花费的时间

- **等待时间**

  进程在就绪队列中的总时间

- **响应时间**

  从一个请求被提交到产生第一次响应所花费的总时间



对于算法来讲，什么指标来衡量比较关键

- **减少响应时间**

  及时处理用户的输出并且尽快将输出提供给用户

- **减少平均响应时间的波动**

  在交互系统中，可预测性比高差异低平均更重要

- **增加吞吐量--两个方面**

  减少开销（操作系统开销，上下文切换）

  系统资源的高效利用（CPU，I/O设备）

- **减少等待时间**

  减少每个进程的等待时间

> 和水管类比
>
> - **低延时**
>
>   喝水的时候想要，一打开水龙头就流出来
>
> - **高带宽**
>
>   给游泳池充水时希望从水龙头里同时流出大量的水，并且不介意是否存在延迟



**低延迟调度算法（增加了交互式表现）**

- 如果移动了鼠标，但是屏幕中的光标却没动，我可能会重启电脑

**但是操作系统需要保证吞吐量不受影响**

- 我想要结束长时间的编程，所以操作系统必须不时进行调度，及时存在很多交互任务。

**吞吐量是操作系统的计算带宽**（Linux服务器）

**响应时间是操作系统的计算延迟**（Linux桌面）





### 8.2.4公平

**公平**也会作为一个很重要的指标

举例

- 保证每个进程占用相同的CPU时间
- 这公平吗？如果一个用户比其他用户运行更多的进程

举例

- 保证每个进程都等待相同的时间

公平通常会增加平均响应时间



## 8.3调度算法

6种常用算法

1. **FCFS（先来先服务）**

   First Come， Frist Served

2. **SPN（SJF）SRT（短进程优先（短作业优先）短剩余时间优先）**

   Shortest Process Next（Shortest Job Frist）Shortest Remaining Time

3. **HRRN（最高响应比优先）**

   Highest Response Ratio Next

4. **Round Robin（轮询）**

   使用时间切片和抢占来轮流执行任务

5. **Multilevel Feedback Queues（多级反馈队列）**

   优先级队列中的轮询

6. **Fair Share Scheduling（公平共享调度）**



### 8.3.1FCFS

FCFS队列的规定

- 如果进程在执行中阻塞，队列中的下一个会得到CPU



举例说明

> 如下图8-1上边所示，三个进程，P1进程执行时间12，P2执行时间3，P3执行时间3。
>
> 周转时间：一个进程从初始化到结束，包括所有等待时间所花费的时间
>
> 这里除了本身三个进程的总时间，还需要加上P2和P3的等待总时间Average response time = (12 + 15 +18) / 3 = 15
>
> 如下图8-1下边所示，如果把P2和P3这两个花费时间少的进程放到前面执行，这个时候的周转时间会发生变化。
>
> Average response time = (3 + 6 +18) / 3 = 9
>
> {{< image classes="fancybox center fig-100" src="/os/os8_3.png" thumbnail="/os/os8_3.png" title="图8-1 FCFS调度算法示例图">}}



优点

- 简单

缺点

- 平均等待时间波动较大
- 花费时间少的任务可能排在花费时间长的任务后面
- 可能导致I/O和CPU之间的重叠处理。CPU密集型进程会导致IO设备闲置时，IO密集型进程也在等待



### 8.3.2SPN（SJF）SRT

选择下一个最短的进程（短任务优先）

- 按照预测的完成时间来将任务入队



这里有四个进程，会通过执行时间排序。按照执行时间的长短排序，执行时间短的排在前面。

在执行P~w~进程的时候，又来了一个进程，且进程时间比P~w~短。

有两种方案：

1. **SPN和SJF**

   一种是不打断当前进程P~w~，且将新进程放到就绪队列的队头。

2. **SRT**

   另一种是**抢占式**的，当前正在运行的进程P~w~从运行态到就绪态，新进程占用CPU执行

{{< image classes="fancybox center fig-100" src="/os/os8_4.png" thumbnail="/os/os8_4.png" title="">}}



举例说明

> 如下图8-2上边所示，一共有6个进程，分别为P1，P2，P3，P4，P5，P6，按照段任务优先方式，直接计算可得到周转时间
>
> Average response time = (r1 + r2 +r3 + r4 + r5 + r6) / 6
>
> 如果这个时候P3进程排列到P5进程后面，且每个进程执行时间都用c~i~表示，直接计算周转时间
>
> Average response time = (r1 + r2 +r4 - c3 + r5 -c3 + r4 + c4 + c5 + r6) / 6 = (r1 + r2 +r3 + r4 + r5 + r6 + (c4 + c5 -2c3)) / 6
>
> 很明显，c4，c5要大于c3所以下面的值会比上面的值大

{{< image classes="fancybox center fig-100" src="/os/os8_5.png" thumbnail="/os/os8_5.png" title="图8-2 SJF调度算法示例图">}}

缺点

**可能导致饥饿**

- 连续的短任务流会使长任务饥饿
- 短任务可用时间的任何长任务的CPU时间都会增加平均等待时间

**需要预知未来**

- 怎么预估下一个CPU突发的持续时间
- 简单的解决方法：询问用户
- 如果用户欺骗就杀死进程
- 如果用户不知道怎么办



计算历史预估执行时间

{{< image classes="fancybox center fig-100" src="/os/os8_6.png" thumbnail="/os/os8_6.png" title="">}}

τ~n+1~ = αt~n~ + (1 - α)τ~n~

其中t~n~指的是t这个时间段中大约占多少执行时间，τ~n~是通过算法估计的t这个时间段中大约占多少执行时间，τ~n+1~是下一时间段中大约占多少执行时间的预估。

得到下一时刻预估的值，可以对进程按执行时间大小进行排序。

通过过去，预测未来。下图所示是大致估计的情况。可以看到蓝色的曲线在逼近黑色的数据。

{{< image classes="fancybox center fig-100" src="/os/os8_7.png" thumbnail="/os/os8_7.png" title="">}}



### 8.3.3HRRN（最高响应比优先）

相比之前SPN的算法，多考虑了等待时间，有效地缓解了饥饿现象

- 在SPN调度基础上改进
- 不可抢占
- 关注进程等待了多长时间
- 防止无限期推迟

R = (w + s) / s

其中w是waiting time，等待时间

其中s是service time，执行时间

选择R值最高的进程，**R越高等待时间越长**



### 8.3.4Round Robin（轮询）

**轮询**

- 在叫作量子（或时间切片）的离散单元中分配处理器
- 时间片结束时，切换到下一个准备好的进程

各个进程轮流占用CPU去执行。

{{< image classes="fancybox center fig-100" src="/os/os8_8.png" thumbnail="/os/os8_8.png" title="">}}



举例说明

> 如下图8-3所示。这里有四个进程P1，P2，P3，P4，设定每个时间切片为20，那么四个进程会轮流分享这20个时间片，这种方式每个进程都有机会去执行。
>
> 等待时间计算
>
> P1 = (68 - 20) + (112 - 88) = 72
>
> P2 = (20 - 0) = 20
>
> P3 = (28- 0) + (88 - 48) + (125 - 108) = 85
>
> P4 = (48 - 0) + (106 - 68) = 88
>
> 平均等待时间ave = (72 + 20 + 85 + 88) / 4 = 66.25
>
> {{< image classes="fancybox center fig-100" src="/os/os8_9.png" thumbnail="/os/os8_9.png" title="图8-3 Round Robin调度算法示例图">}}



**RR**

- **花销存在额外的上下文切换**

- **时间量子太大**

  等待时间过长

  极限情况退化成FCFS

- **时间量子太小**

  反应迅速，但是吞吐量由于大量的上下文切换开销受到影响

- **目标**

  选择一个是和的时间量子

  经验规则：维持上下文切换开销处于1%以内



如下图8-3所示。可以看到设置不同的时间量子会有不同的时间开销。

{{< image classes="fancybox center fig-100" src="/os/os8_10.png" thumbnail="/os/os8_10.png" title="图8-3 不同的时间量子平均等待时间图">}}

### 8.3.5Multilevel Feedback Queues（多级反馈队列）

#### 8.3.5.1多级队列调度算法（MQ）

- 就绪队列被划分成独立的队列

  E.g. 前台交互，后台处理

- 每个队列拥有自己的调度策略

  E.g. 前台RR，后台FCFS

- 调度必须在队列间进行

- **固定优先级**

  先处理前台，后处理后台

  可能导致饥饿

- **时间切片**

  每个队列都得到一个确定的能够调度其进程的CPU总时间

  E.g. 80%给使用RR的前台，20%给使用FCFS的后台



#### 8.3.5.2多级反馈队列(MFQ)

 **可以根据情况（反馈）调整进程的优先级、队列。**

一个进程可以在不同的队列中移动

- RR在每个级别中，时间量子大小随**优先级级别增加而增加**
- RR在每个级别中，如果任务在当前的时间量子中没有完成，则降到下一个优先级。

{{< image classes="fancybox center fig-100" src="/os/os8_11.png" thumbnail="/os/os8_11.png" title="">}}



### 8.3.6Fair Share Scheduling（公平共享调度）

从用户这个级别中，实现资源公平的共享。

**Linux采用FSS这种调度策略**。



FSS控制用户对系统资源的访问

- 一些用户组比其他客户组更重要
- 保证不重要的组无法垄断资源
- 未使用的资源按照每个组所分配的资源的比例来分配
- 没有达到资源使用率目前的组获得更高的优先级

{{< image classes="fancybox center fig-100" src="/os/os8_13.png" thumbnail="/os/os8_13.png" title="">}}

### 8.3.7评价算法

- **确定性建模**

  确定一个工作量，然后计算每个算法的表现

- **队列模型**

  用来处理随机工作负载的数学方法

- **实现/模拟**

  建立一个允许算法运行实际数据的系统

  最灵活/具有一般性

{{< image classes="fancybox center fig-100" src="/os/os8_12.png" thumbnail="/os/os8_12.png" title="">}}

### 8.3.8总结

上述六种算法，跟实际上的调度算法是会有差异的。但是上述算法中的**公平性**、**最小等待时间**、**最小响应时间**等，也是实际操作系统中必须考虑的事情。

| 简单调度算法                      | 特点                                               |
| --------------------------------- | -------------------------------------------------- |
| FCFS 先来先服务                   | 不公平<br />平均等待时间较差                       |
| SPN/SRT 短进程优先                | 不公平<br />需要精确预测计算时间<br />可能导致饥饿 |
| HRRN 最高响应比优先               | 基于SPN调度改进<br />不可抢占                      |
| Round Robin 轮询                  | 公平，但是平均等待时间较差                         |
| MLFQ 多级反馈队列                 | 和SPN类似                                          |
| Fair-share scheduling公平共享调度 | 公平第一要素                                       |



## 8.4实时调度

上述更多的调度的算法是通用性的，而实时调度更多用在工业控制上。比如火车、机床、或者是嵌入式设备等，规定需要在某一段时间内必须完成。

- **定义**

  正确性依赖于其时间和功能两方面的一种操作系统

- **性能指标**

  时间约束的及时性

  速度和平均性能相对不重要

- **主要特性**

  时间约束的可预测性



### 8.4.1实时系统

- **强实时系统**

  需要在保证的时间内完成重要的任务，**必须完成**

- **弱实时系统**

  要求重要的进程的优先级更高，尽量完成，**并非必须**



### 8.4.2实时任务

ReleasedTime：让进程处于就绪态的时间

Relative deadline：相对最后期限，任务是一个一个间隔完成的，每个时间完成特定的任务

Execution time：任务是蓝色区域表示

Absolute deadline：绝对最后期限，任务的时间不能超过这个时间点

{{< image classes="fancybox center fig-100" src="/os/os8_14.png" thumbnail="/os/os8_14.png" title="">}}



举例说明

> 如下图8-4所示，有一系列相似的任务
>
> 其中公式使用率U = e / p
>
> {{< image classes="fancybox center fig-100" src="/os/os8_15.png" thumbnail="/os/os8_15.png" title="图8-4 实时任务使用率图">}}



### 8.4.3时限

- **硬时限**

  如果错过了最后期限，可能会发生灾难性或非常严重的后果

  必须验证：在最坏的情况下也能够满足时限

  保证**确定性**

- **软时限**

  理想情况下，时限应该被最大满足。如果有时限没有被满足，那么相应地降低要求

  尽量大努力去保证



表示一个实时系统是否能够满足deadline 要求

- 决定实时任务执行的顺序
- 静态优先级调度，比如先进先服务算法
- 动态优先级调度，比如循环调度算法



{{< image classes="fancybox center fig-100" src="/os/os8_16.png" thumbnail="/os/os8_16.png" title="">}}



### 8.4.4实际中的两类算法

**RM（Rate Monotonic）速率单调算法**

- 最佳**静态**优先级调度
- 通过**周期**安排优先级
- 周期越短优先级越高
- 执行周期最短的任务

**EDF（Earliest Deadline First）最早期限调度**

- 最佳的**动态**优先级调度
- Deadline最早优先级越高
- 执行Deadline最早的任务





## 8.5多处理器调度

多处理器重点考虑的两个问题：**系统的确定性调度**和**负载均衡**

- **多处理器的CPU调度更加复杂**

  多个相同的但处理器组成一个多处理器

  优点：负载共享

- **对称对处理器（SMP）**

  每个处理器运行自己的调度程序

  需要在调度程序中同步

{{< image classes="fancybox center fig-100" src="/os/os8_17.png" thumbnail="/os/os8_17.png" title="">}}



## 8.6优先级反转

火星探路者的操作系统不断重启。发生优先级反转现象

{{< image classes="fancybox center fig-100" src="/os/os8_18.png" thumbnail="/os/os8_18.png" title="">}}

正常情况下，有三个进程正在执行，分别是T1，T2和T3，且优先级T1>T2>T3。



### 8.6.1优先级反转现象

T1在执行的过程中，它跟T3一样也需要访问共享资源，需要等T3执行完毕。那么T3的优先级又比T2低，所以T3的执行需要等待T2执行完成，即T1执行也需要等待T2执行完毕，但实际上T1的优先级会比T2大。这个时候系统检测T1进程得不到执行，就会进行重启。

{{< image classes="fancybox center fig-100" src="/os/os8_19.png" thumbnail="/os/os8_19.png" title="">}}

### 8.6.2解决方式

1）**优先级继承**

T1在执行的过程中，它跟T3一样也需要访问共享资源，需要等T3执行完毕。那么T3的优先级会提升到跟T1一样，这个时候就不会被T2进程打断，直到T3共享资源部分执行完毕。

{{< image classes="fancybox center fig-100" src="/os/os8_20.png" thumbnail="/os/os8_20.png" title="">}}

2）**优先级天花板协议**

- 资源的优先级和所有可以锁定该资源的任务中优先级最高的那个任务的优先级相同（T1和T3的共享资源优先级最高）
- 除非优先级高于系统中所有被锁定的资源的优先级上限，否则任务尝试执行临界区的时候会被阻塞（T2会被阻塞）
- 持有最高优先级上限信号量锁的任务，会继承被该锁所阻塞的任务的优先级（T3会直接继承T1的优先级）