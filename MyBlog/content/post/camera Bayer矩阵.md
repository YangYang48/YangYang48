---
title: "camera Bayer矩阵"
date: 2023-12-10
thumbnailImagePosition: left
thumbnailImage: camera/bayer/bayer_thumb.jpg
coverImage: camera/bayer/bayer_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- camera
- 2023
- December
tags:
- isp
- RAW
- RGB
- HSV
- HSL
- YUV
- JPG
- JPEG
showSocial: false
---

本文主要记录一些关于camera相关的概念和资料，主要是整体的一些概念，细节方面后续会单独有篇章去介绍。

<!--more-->
{{< wyymusic 1397345903 >}}

# 0一些基本概念

> 开篇先问一个问题
>
> RAW格式、Bayer阵列、RGB格式、HSV格式、YUV格式、JPG格式、JPEG格式等这些专业名词都是什么意思？
>

说实话，如果是新手或者刚开始接触的时候肯定只能一个个百度。

这里直接将概念阐述清楚

## 0.1RAW格式

原意就是“未经加工”。

可以理解为：RAW图像就是CMOS或者CCD图像感应器将捕捉到的光源信号转化为数字信号的原始数据。RAW文件是一种记录了数码相机传感器的原始信息，同时记录了由相机拍摄所产生的一些元数据（Metadata，如ISO的设置、快门速度、光圈值、白平衡等）的文件。RAW是未经处理、也未经压缩的格式，可以把RAW概念化为“原始图像编码数据”或更形象的称为“数字底片”。

通俗的说，就是照相机按下快门之后，光信号到达Sensor之后产生的数字信号，等同于以前的老式机械相机的底片，你还看不到里面的图案，只不过是将里面的元素给数字化了。

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_10.png" thumbnail="/camera/bayer/bayer_10.png" title="">}}



# 0.2Bayer阵列

这个格式，也是我们这篇的重点

对于彩色图像，需要采集多种最基本的颜色，如rgb三种颜色，最简单的方法就是用滤镜的方法，红色的滤镜透过红色的波长，绿色的滤镜透过绿色的波长，蓝色的滤镜透过蓝色的波长。

如果要采集rgb三个基本色，则需要三块滤镜，这样价格昂贵，且不好制造，因为三块滤镜都必须保证每一个像素点都对齐。当用bayer格式的时候，很好的解决了这个问题。bayer 格式图片在一块滤镜上设置的不同的颜色，通过分析人眼对颜色的感知发现，人眼对绿色比较敏感，所以一般bayer格式的图片绿色格式的像素是是r和g像素的和。

另外，Bayer格式是相机内部的原始图片, 一般后缀名为.raw。很多软件都可以查看, 比如PS。我们相机拍照下来存储在存储卡上的.jpeg或其它格式的图片, 都是从.raw格式转化过来的。如下图，为bayer色彩滤波阵列，由一半的G，1/4的R，1/4的B组成。



# 0.3RGB格式

人类眼睛中有三种对不同电磁波频率敏感的视锥细胞，分别是 **Small** **Mediam** **Long**，所有的颜色就是对这三种细胞不同强度刺激的组合结果，比如：

- 黄色就是同时刺激 M 细胞和 L 细胞的结果
- 同时刺激三种细胞时，我们就会得到白色
- 如果都没有刺激，那就是黑色

这三种基本的光就是我们通常意义上所说的三原色，通过这三种原色，基于同时用540nm 和 570 nm 波长的光刺激锥状细胞在人类大脑看来大约等价于用 560nm光刺激的原理我们可以模拟出任意颜色的效果。不过这种等价只基于人类成立，鸡只有两种视锥细胞，而皮皮虾则有足足十六对，人类眼中相同的颜色在其他生物眼中可能差异非常大，不过这并不重要，只要能骗过我们的大脑就行，这种原理就是我们现代彩色显示器的理论基础。

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_14.png" thumbnail="/camera/bayer/bayer_14.png" title="">}}

RGB 是我们接触最多的颜色空间，由三个通道表示一幅图像，分别为红色(R)，绿色(G)和蓝色(B)。这三种颜色的不同组合可以形成几乎所有的其他颜色。RGB 颜色空间是图像处理中最基本、最常用、面向硬件的颜色空间，比较容易理解。利用三个颜色分量的线性组合来表示颜色，任何颜色都与这三个分量有关，而且这三个分量是高度相关的。

RGB 颜色的十六进制由六位十六进制数字组成的，形式如 `#RRGGBB`，其中 RR、GG、BB 分别是两位十六进制数字，分别表示红、绿、蓝三原色通道的**色阶**。比如我们使用的截图工具snipaste，可以在截图的时候显示显示器上任意一点的RGB的颜色，默认博客的背景就是255，显示未（255,255,255）对应RGB三色。

色阶可以表示某个通道的强弱。

两位十六进制数字可以表示 256 个数字（0 ~ 255），也就是 256 个色阶。所以理论上一共能表示 2 的 24次方，也就是一共 16777216 种不同的颜色。

我们可以用一个三维立方体，把 RGB 能表示的所有颜色形象地描述出来。效果如下图：

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_17.png" thumbnail="/camera/bayer/bayer_17.png" title="">}}



不过 RGB 并不能表示人眼所能看见的所有颜色，如下图所示

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_18.png" thumbnail="/camera/bayer/bayer_18.png" title="">}}

# 0.4HSV/HSL格式

自然环境下获取的图像容易受自然光照、遮挡和阴影等情况的影响，即对亮度比较敏感。而 RGB 颜色空间的三个分量都与亮度密切相关，即只要亮度改变，三个分量都会随之相应地改变，而没有一种更直观的方式来表达。

## 0.4.1HSV颜色空间

在图像处理中使用较多的是 **HSV 颜色空间**，它比 RGB 更接近人们对彩色的感知经验。非常直观地表达颜色的色调、鲜艳程度和明暗程度，方便进行颜色的对比。

在 HSV 颜色空间下，比RGB 更容易跟踪某种颜色的物体，常用于分割指定颜色的物体。

HSV 表达彩色图像的方式由三个部分组成：

- Hue（色调、色相）:直接决定了什么颜色，RGB是通过三个值决定颜色，而HSV只通过H值就可以决定；

  用角度度量，取值范围为0°～360°，从红色开始按逆时针方向计算，红色为0°，绿色为120°,蓝色为240°，。它们的补色是：黄色为60°，青色为180°,品红为300°，0°-  359°时颜色会依次变换当角度到达360°时也就是红色，角度也就又回到0°了，所以总共为360°，每变换1°时，色相就会有轻微的变化！如果是顺时针的话这个变换过程会从红色逐渐变换到绿色，在由绿色逐渐变换到蓝色，在由蓝色逐渐变换到红色！逆时针的话就是相反的！
  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_20.png" thumbnail="/camera/bayer/bayer_20.png" title="">}}

- Saturation（饱和度、色彩纯净度）：决定了颜色的纯度，饱和度越高，颜色越鲜艳；

  饱和度S表示颜色接近光谱色的程度。一种颜色，可以看成是某种光谱色与白色混合的结果。其中光谱色所占的比例愈大，颜色接近光谱色的程度就愈高，颜色的饱和度也就愈高。饱和度高，颜色则深而艳。光谱色的白光成分为0，饱和度达到最高。通常取值范围为0%～100%，值越大，颜色越饱和。

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_21.png" thumbnail="/camera/bayer/bayer_21.png" title="">}}

- Value（明度）：决定了颜色的亮度，亮度越高，颜色越亮；

  明度表示颜色明亮的程度，对于光源色，明度值与发光体的光亮度有关；通常取值范围为0%（黑）到100%（白）。

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_22.png" thumbnail="/camera/bayer/bayer_22.png" title="">}}

用下面这个圆柱体来表示 HSV 颜色空间，圆柱体的横截面可以看做是一个极坐标系 ，H 用极坐标的极角表示，S 用极坐标的极轴长度表示，V 用圆柱中轴的高度表示。

HSV数学模型

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_15.png" thumbnail="/camera/bayer/bayer_15.png" title="">}}

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_38.png" thumbnail="/camera/bayer/bayer_38.png" title="">}}

H参数表示色彩信息，即所处的光谱颜色的位置。该参数用一角度量来表示，红、绿、蓝分别相隔120度。HSV中每一种颜色的互补色分别相差180度。意思就是说：两种颜色在互补时最大为180°

例如：

在HSV模型中红与绿的互补色为黄色，其角度为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_23.png" thumbnail="/camera/bayer/bayer_23.png" title="">}}

绿色与蓝色的互补光为青色其角度也为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_24.png" thumbnail="/camera/bayer/bayer_24.png" title="">}}

蓝色与红色的互补光为品红色其角度也为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_25.png" thumbnail="/camera/bayer/bayer_25.png" title="">}}

那么按逆反的方向来算，绿色到红色的互补光为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_26.png" thumbnail="/camera/bayer/bayer_26.png" title="">}}

蓝色到绿色的互补光也为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_27.png" thumbnail="/camera/bayer/bayer_27.png" title="">}}

红色到蓝色的互补光也为60°

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_28.png" thumbnail="/camera/bayer/bayer_28.png" title="">}}

所以通过以上知识可以知道，红色到到绿色之间的互补光为60°，而绿色到红色之间的互补光也为60°所以每一种颜色的色差是：60°+  60°=  120°

互补光的色差在HSV颜色模型中是这样来算的！

纯度S为一比例值，范围从0到1，它表示成所选颜色的纯度和该颜色最大的纯度之间的比率。S=0时，只有灰度。

V表示色彩的明亮程度，范围从0到1。有一点要注意：它和光强度之间并没有直接的联系。 

HSV对用户来说是一种直观的颜色模型。我们可以从一种纯色彩开始，即指定色彩角H，并让V=S=1，然后我们可以通过向其中加入黑色和白色来得到我们需要的颜色。增加黑色可以减小V而S不变，同样增加白色可以减小S而V不变。例如，要得到深蓝色，V=0.4 S=1 H=240度。要得到浅蓝色，V=1 S=0.4 H=240度。

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_29.jpg" thumbnail="/camera/bayer/bayer_29.jpg" title="">}}

一般说来，人眼最大能区分128种不同的色彩，130种色饱和度，23种明暗度。如果我们用16Bit表示HSV的话，可以用7位存放H，4位存放S，5位存放V，即745或者655就可以满足我们的需要了。

## 0.4.2HSL颜色空间

HSL〔Hue-SaturationIntensity(Lightness),HSI或HSL〕颜色模型用H、S、L三参数描述颜色特性，其中H定义颜色的频率，称为色调；S表示颜色的深浅程度，称为饱和度；L表示强度或亮度。

- 色调(色相)

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_30.jpg" thumbnail="/camera/bayer/bayer_30.jpg" title="">}}

  我们修改一下色调，当把色调调低时，颜色更加偏向于红色

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_31.jpg" thumbnail="/camera/bayer/bayer_31.jpg" title="">}}

  当我们把色调调高一点时，颜色更加偏向于绿色

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_32.jpg" thumbnail="/camera/bayer/bayer_32.jpg" title="">}}

  当颜色在调高一点时，颜色更加偏向于蓝色

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_33.jpg" thumbnail="/camera/bayer/bayer_33.jpg" title="">}}

  所以由此可以得出色调是决定一个像素点中的颜色更偏向于哪一方(RGB)

- 饱和度

  饱和度决定了颜色空间中颜色分量，饱和度越高，说明颜色越深，饱和度越低，说明颜色越浅！

  当饱和度为55时，可以发现该颜色空间能显示的颜色分量非常低

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_34.jpg" thumbnail="/camera/bayer/bayer_34.jpg" title="">}}

  当我把饱和度调高一点时，可以发现颜色分量显示的明显要深！

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_35.jpg" thumbnail="/camera/bayer/bayer_35.jpg" title="">}}

  可以与上图形成鲜明的对比。

  所以饱和度在颜色空间中是起到一个控制RGB组合色的颜色深度的作用。

- 亮度

  亮度决定颜色空间中颜色的明暗程度！

  如图，亮度设置比较高的时候会发现颜色显示的较为鲜艳

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_36.jpg" thumbnail="/camera/bayer/bayer_36.jpg" title="">}}

  当我们把亮度调低一点时

  {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_37.jpg" thumbnail="/camera/bayer/bayer_37.jpg" title="">}}

  会发现颜色会变得非常暗！

  所以亮度在颜色空间中起到一个控制RGB组合色的明暗程度的作用。

 彩色图片中，色调决定彩色图片更加偏于哪一方！

当人观察一个彩色物体时，用色调、饱和度、亮度来描述物体的颜色。色调是描述纯色的属性(纯黄色、橘黄或者红色)；饱和度给出一种纯色被白光稀释的程度的度量；亮度是一个主观的描述，实际上，它是不可以测量的，体现了无色的强度概念，并且是描述彩色感觉的关键参数。

而强度(灰度)是单色图像最有用的描述，这个量是可以测量且很容易解释。则将提出的这个模型称作为HSL(色调、饱和度、强度)彩色模型，该模型可在彩色图像中从携带的彩色信息(色调和饱和度)里消去强度分量的影响，使得HSL模型成为开发基于彩色描述的图像处理方法的良好工具，而这种彩色描述对人来说是自然而直观的。

数学模型

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_16.png" thumbnail="/camera/bayer/bayer_16.png" title="">}}

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_39.png" thumbnail="/camera/bayer/bayer_39.png" title="">}}

1. 明度为0的时候，只有一个点，只能是黑色。没有光，啥都看不见。后面我们要让明度是某个不为零的值，才好谈下去。基于这个条件，
2. 纯度为0的时候，只有一条线，只能是黑白的。没有对比度，就没有彩色。
3. 纯度也不为0了，才可能出现彩色，至于到底是哪一种颜色，就要看色度了。



不知道大家有没有发现，无论你怎么修改色调，饱和度，亮度，RGB三色值会跟随而变化，其实色调，饱和度，亮度都是通过特定的算法经过计算修改RGB三色而达到的控制颜色效果！




两者比较

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_19.png" thumbnail="/camera/bayer/bayer_19.png" title="">}}

## 0.4.3HSL 和 HSV 的局限性

1. 色相相差相同的情况下，各个颜色直接的颜色变化并不是均匀的；
2. 不同色相相同亮度的颜色看起来亮度却不一样，这是由于人眼对不同频率的光的敏感度不同造成的。

因此，HSL 依然不是最完美的颜色方法，我们还需要建立一套针对人类知觉的标准，这个标准在描述颜色的时候要尽可能地满足以下 2 个原则：

1. 人眼看到的色差 = 颜色向量间的欧氏距离
2. 相同的亮度，能让人感觉亮度相同

于是，一个针对人类感觉的颜色描述方式就产生了，它就是比较新的颜色表示技术 CIE Lab。

> CIE Lab
>
> CIE L*a*b*（CIELAB）是惯常用来描述人眼可见的所有颜色的最完备的色彩模型，简称 Lab。它是为这个特殊目的而由国际照明委员会（Commission Internationale d'Eclairage 的首字母是 CIE）提出的。L、a 和 b 后面的星号（*）是全名的一部分，因为它们表示L*, a* 和 b*，不同于 L, a 和 b。因为红／绿和黄／蓝对立通道被计算为（假定的）锥状细胞响应的类似孟塞尔值的变换的差异，CIELAB 是 Adams 色彩值（Chromatic Value）空间。
>
> L* 表示亮度, L* = 0 生成黑色而 L* = 100 指示白色，
>
> a* 表示在红色／品红色和绿色之间的位置（a* 负值指示绿色而正值指示品红）
>
> b* 表示在黄色和蓝色之间的位置（b* 负值指示蓝色而正值指示黄色）。



# 0.5 YUV格式

 YUV是一种彩色编码系统，主要用在视频、图形处理流水线中(pipeline)。相对于 RGB 颜色空间，设计 YUV的目的就是为了编码、传输的方便，减少带宽占用和信息出错。

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_11.png" thumbnail="/camera/bayer/bayer_11.png" title="">}}

上面的图像为原始图像，可以拆分为RGB的三色图

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_12.png" thumbnail="/camera/bayer/bayer_12.png" title="">}}

原始图像也可以用YUV表示

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_13.png" thumbnail="/camera/bayer/bayer_13.png" title="">}}

根据上面的图片，不难看出：

- Y分量对呈现出清晰的图像有着很大的贡献
- Cb、Cr分量的内容不太容易识别清楚

此外，你是否感觉：Y分量的内容看着有点眼熟？其实以前黑白电视的画面就是长这样子的。

YUV的发明处在彩色电视与黑白电视的过渡时期。

- YUV将亮度信息（Y）与色度信息（UV）分离，没有UV信息一样可以显示完整的图像，只不过是黑白的
- 这样的设计很好地解决了彩色电视与黑白电视的兼容性问题，使黑白电视也能够接收彩色电视信号，只不过它只显示了Y分量
- 彩色电视有Y、U、V分量，如果去掉UV分量，剩下的Y分量和黑白电视相同

# 0.6JPG、JPEG格式

二者没有本质的区别，**Joint Photographic Experts Group**即都是**联合图像专家组**，是用于连续色调静态图像压缩的一种标准，文件后缀名为.jpg或.jpeg，是最常用的图像文件格式。

通俗的说就是我们保存一张照片的格式，这种格式里面有一些特点：用有损压缩方式去除冗余的图像数据，用较少的磁盘空间得到较好的图像品质。而且JPEG是一种很灵活的格式，具有调节图像质量的功能，它允许用不同的压缩比例对文件进行压缩等等。

# 1bayer矩阵



{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_1.png" thumbnail="/camera/bayer/bayer_1.png" title="">}}

这就是图像传感器上面的感光单元，为一个个红绿蓝的滤光片阵列按照某种规则排布的，这种格式的一般规律是：

​    奇数扫描行输出 RGRG……

​    偶数扫描行输出 GBGB……

这种格式也就是所说的专业名词：**Bayer格式**

>  很多人会说，一个像素不是由三基色组成吗？为什么这一个像素只有一种颜色呢？那不是不合理吗？
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_2.png" thumbnail="/camera/bayer/bayer_2.png" title="">}}
>
> 1.人眼观察到的图像  2.RAW原始图像
> 3.Bayer格式输出图像 4.运用插值计算后输出图像
>
> 对于图1来说，对1人眼观察到的图像（也可以说是保存在存储设备当中的.JPEG格式的图片）进行放大，我们能观察到其中的每个像素点，也就是RGB颜色空间
>
> 对于图2来说，然后使用相机对着这个景色拍照，得到的RAW数据就是2所示，也就是非常原始的图像信息，其中只有亮度分量，而没有色度、色调、饱和度等其他分量（这里就是没有Y、U分量，只有V的意思吧？应该可以这么理解吧？引出YUV颜色空间知识点），这个图像相对应就是之前所说的老式机械照相机的黑白底片。
>
> 对于图3来说，经过Bayer阵列的采集之后就得到了图3所示相机传感器获取到的原始图像信息，粗略看好像是和人眼看到的大致相同，但是放大之后观察
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_3.png" thumbnail="/camera/bayer/bayer_3.png" title="">}}
>
> 放大后效果其实是和上述所说的Bayer格式一样的，奇数扫描行输出 RGRG……，偶数扫描行输出 GBGB……，每一个像素点只有RGB通道当中的一个分量，也就是说有2/3的颜色分量是没有的。
>
> 对于图4来说，计算机进行后期处理的第一步就是猜色，也叫去马赛克。
> 如果一个像素只可能有三种颜色，那么怎么能拍出彩色照片呢？前面说了，每个滤光点周围有“规律”地分布其他颜色的滤光点，那么就有可能结合它们的值，判断出光线本来的颜色。

# 2bayer插值

## 2.1bayer格式图像传感器硬件

图像传感器的结构如下所示，每一个感光像素之间都有金属隔离层，光纤通过显微镜头，在色彩滤波器过滤之后，投射到相应的漏洞式硅的感光元件上。

{{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_4.png" thumbnail="/camera/bayer/bayer_4.png" title="">}}

当Image Sensor往外逐行输出数据时，像素的序列为GRGRGR.../BGBGBG...（顺序RGB）。这样阵列的Sensor设计，使得RGB传感器减少到了全色传感器的1/3，如下所示。

```c
(1/2+1/4+1/4)/(1+1+1)=1/3
```

## 2.2bayer格式插值红蓝算法实现

这里只介绍算法，具体算法怎么实现需要自己查阅资料了

每一个像素仅仅包括了光谱的一部分，必须通过插值来实现每个像素的RGB值。为了从Bayer格式得到每个像素的RGB格式，我们需要通过插值填补缺失的2个色彩。插值的方法有很多（包括领域、线性、3*3等），速度与质量权衡，最好的线性插值补偿算法。其中算法如下： 

R和B通过线性领域插值，但这有四种不同的分布，如下图所示：

 {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_5.png" thumbnail="/camera/bayer/bayer_5.png" title="">}}

在（a）与（b）中，R和B分别取邻域的平均值。

在（c）与（d）中，取领域的4个B或R的均值作为中间像素的B值。 

> 关于为什么叫拜耳矩阵
>
> 2012年末，一位对世界，特别是对蜂鸟网所有网友的生活，产生巨大影响的老人，进入了天堂，他的名字叫布莱斯·拜尔（Bryce Bayer）。
>
> 拜尔在天堂遇见了上帝。
>
> 上帝：拜尔，你这个骗子！看看你在下面做的好事，现在还有脸来见我？
>
> 拜尔：我的主啊，我是您忠实的信徒，我怎么会是骗子呢？
> 上帝：你在下面发明了个什么“拜尔阵列”，这玩意儿几乎垄断了人类的数码相机行业，这是你干的吗？！
>
> 拜尔：是啊，我的主。我发明了这种传感器，让数码摄影技术在人类中得到了普及，这难道不是干了个大好事吗？
>
> 上帝：普及数码摄影技术当然是好事，但关键是，你的传感器偷工减料啊！我在创世纪的时候，说要有光，于是就有了光，我创造的光里面包含了红、绿、蓝三种基本色。
>
> 但是你的传感器里面根本就没有完整记录我这三种颜色，你每个像素只记下了一种颜色的亮度值，然后通过后期处理（=PS？）软件，胡乱猜出像素里另两个基本色，再弄出图像来糊弄人，一张照片里只有1/3的色彩是真实的，这还不算骗子啊你！
>
> 拜尔：我的主啊，您看我一脸老实相，会是骗子吗！我的传感器这么干，是有苦衷的啊！您得听我慢慢道来。
>
> 上帝：好，那你说吧。要是说得有道理，能说服我，特别是能说服蜂鸟网上的网友，那才能让你进入名人堂。否则的话，你还得回到人间去，自己去收拾你的烂摊子。
>
> 拜尔手臂一挥，上帝面前出现了一个大屏幕，结合着屏幕上的图文，拜尔开上给上帝解释起来。
>
> 拜尔：那是上个世纪的70年代，我在柯达公司从事科研工作，其中一个重要课题，就是怎么样才能将影像转换成数字信号储存下来。我们都是凡人啊，凡人没有您那样无边法力，所以我们的光电传感器只能够记录光的强度，而无法分辨光的颜色，即使是现在21世纪了，依然是这样。
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_6.png" thumbnail="/camera/bayer/bayer_6.png" title="">}}
>
> 但是凡人也是贪心的，我们不满足于只能拍黑白数码照片，只能在有限的条件下想尽办法，去尽可能的获得色彩。我绞尽了脑汁，要想办法解决颜色的记录问题，直到有一天我的小狗菲儿帮我解决了这个问题。
>
> 上帝：你的狗？
>
> 拜尔：是的，我的小狗狗菲儿。这天它嘴里咬着一件东西，跑到我面前，是一个黄色滤镜。就是我以前拍黑白照片时常用的那种滤镜，黄色的滤镜可以滤除或减小红、蓝光对照片的影响。
>
> 看到这个滤镜，我脑中顿时灵光一闪，如果我在每个像素上分别装上这三种滤镜，不就能得到三种光的亮度了吗，然后一合成，色彩就重现了！上帝啊，难道不是您指示我的小狗来帮助我的吗？
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_7.jpg" thumbnail="/camera/bayer/bayer_7.jpg" title="">}}
>
> 上帝：咳咳，嗯，这倒是我干的……
>
> 拜尔：但是，我们凡人做不到。我们的元器件制造水平达不到这个要求。我们无法轻易地在一个像素里造进去三个滤镜和感光元件。即使勉强能做到，这个成本也不是绝大多数人能够承受的。无法商业化的东西，对于我们商业化的公司而言，没有现实价值。
>
> 在随后的日子里，我又尝试了各种方法，最后，发明了一种基于单个颜色微小滤镜的影像传感器系统，就是现在被人们称为“拜尔阵列”的传感器系统。
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_8.jpg" thumbnail="/camera/bayer/bayer_8.jpg" title="">}}
>
> 这只用一块图像传感器，就解决了颜色的识别。做法是在图像传感器前面，设置一个滤光层（Color filter array），上面布满了一个个滤光点，与下层的像素一一对应。
>
> 每个滤光点只能通过红、绿、蓝之中的一种颜色，这意味着在它下层的像素点只可能有三种颜色：红、绿、蓝，或者什么也没有（黑）。
>
> 不同颜色的滤光点的排列是有规律的：每个绿点的四周，分布着2个红点、2个蓝点、4个绿点。这意味着，整体上，绿点的数量是其他两种颜色点的两倍。这是因为研究显示人眼对绿色最敏感，所以滤光层的绿点最多。
>
> 以黄光为例，它由红光和绿光混合而成，那么通过滤光层以后，红点和绿点下面的像素都会有值，但是蓝点下面的像素没有值，因此看一个像素周围的颜色分布----有红色和绿色，但是没有蓝色----就可以推测出来这个像素点的本来颜色应该是黄色。
>
> {{< image classes="fancybox center fig-100" src="/camera/bayer/bayer_9.png" thumbnail="/camera/bayer/bayer_9.png" title="">}}
>
> 在得到每个像素的RGB颜色后，后期软件还要加入白平衡矫正、gamma校正，并应用风格曲线、降噪锐化设置等参数，这些参数多数可以人为去设定，这个参数设定的过程，就是俗称的ps，准确地讲就是人工地后期处理。
>
> 处理完后最后生成一幅位图，可以用TIFF、JPG等位图格式保存在磁盘上。
>
> 上帝：不对，不对！你说是要完成后期处理后才能看到图像，但是多数相机都有实时取景模式，我都没拍呢，哪来的后期！还有我看到我们天国摄协里也在玩LR什么的，这不是在做后期调整前就看到图了？还是看着图一点一点地调呢！这你怎么解释？
>
> 拜尔：我的主啊，人类是很狡猾的！实时取景和LR等软件里给先你看到的图像，都是先用它内部默认的后期参数进行粗略后处理，先糊弄一下人眼，满足一下心理。当你全部后期参数设置完后，这不还有一个“导出”的过程吗？这才是真正在进行完整全面的图像后处理呢！
>
> 上帝：我的上帝啊！哦，也就是我的我啊！弄出个照片还有这么多花花肠子！
>
> 拜尔：在您还未赐予我们人类全色彩的感光器件前，我也是没办法才想出这么个主意，通过计算去猜测颜色的啊！
>
> 上帝：那后来不是有个适马公司搞出个叫FOVEON X3的全色彩图像传感器吗？有了这个怎么还在用你的老拜尔？
>
> 拜尔：我的主啊，FOVEON X3的全色彩传感器是好，但是一来有专利权的限制，难以推广，二来也是由于技术水平限制，这个FOVEON X3传感器在高感光度下的表现实在是不行，所以多数厂家还是在沿用我的老拜尔阵列。
>
> 哦，还有一家富士公司，改进了我拜尔阵列的颜色滤镜的排列方式，也取得了很好的效果。据说佳能公司也有了全色彩的传感器专利。不知道今后那天，我的老拜尔阵列也会和我一起进入天堂，存入天堂博物馆了。

# 参考

[[1] JJarven. 数字图像处理—Bayer初学（小记）, 2023.](https://blog.csdn.net/JJarven/article/details/130638186?app_version=6.2.2&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22130638186%22%2C%22source%22%3A%22unlogin%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app)