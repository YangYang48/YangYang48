
---
title: "Typora使用过期解决方案"
date: 2022-07-16
thumbnailImagePosition: left
thumbnailImage: software/typora_thumb.jpg
coverImage: software/typora_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- OS
- 2022
- July
tags:
- 学习笔记
- 软件安装打开
showSocial: false
---
Typora使用过期解决方案。基于This beta version of Typora is expired, please download and install a newer version.的解决方法。

<!--more-->
> **个人申明**
>
> Typora是一款收费的软件，我们用的是试用版。如果是企业用户，请退出本文，防止引来不必要的侵权风险。
>
> 这里还是呼吁一下大家，有钱就买正版，支持开发者。
>
> 这里是笔者自己记录的一些笔记，不对任何人的后果进行负责，如果后续有不必要的麻烦，请须知。



# 0软件无法打开

今天打开Typora软件，发现出现弹框提示，Typora This beta version of Typora is expired, please download and install a newer version。（too bad，说明到期了）

可以看到下面的日志，也可以印证当前本机时间超过了试用的时间。

{{< image classes="fancybox center fig-100" src="/software/typora1.png" thumbnail="/software/typora1.png" title="">}}



目前常用的几种方案都没有效果，比如这个博主的方法，在本机上使用依然存在过期操作[点击](https://blog.csdn.net/yyywxk/article/details/125133205)。

但是最终在博客中发现了一位博主的方式有用的，在本机上使用已经能够正常打开，不影响使用[点击](https://blog.csdn.net/no_say_you_know/article/details/125806545?spm=1001.2014.3001.5501)。



>  **[原理]**
>
> 其实跟上述第一位博主的原理相同，都是通过改动时间，让本机的时间回退到能够使用的时间。
>
> 与第一位博主不同的是，这里改动的不是本机时间，而是去修改校验的时间戳。



# 1准备工作

由于笔者之前没有这些环境，所以这些操作需要重新部署。

1.Python环境

可以直接在[这里](https://www.python.org/downloads/windows/)下载最新的，笔者下载的是Download [Windows installer (64-bit)](https://www.python.org/ftp/python/3.10.5/python-3.10.5-amd64.exe)。下载完成之后安装，需要勾选pip这个选项，默认是勾选的。

2.GitHub下载

直接点击下面的红色区域下载即可

{{< image classes="fancybox center fig-100" src="/software/typora2.png" thumbnail="/software/typora2.png" title="">}}

# 2进行python环境变量配置

这个环境配置，就是把下载python的安装目录放到系统路径中。

那么下次直接在cmd的可执行窗口就可以直接使用这个命令。

{{< image classes="fancybox center fig-100" src="/software/typora3.png" thumbnail="/software/typora3.png" title="">}}

然后在cmd中执行查看是否已经配置成功

{{< image classes="fancybox center fig-100" src="/software/typora4.png" thumbnail="/software/typora4.png" title="">}}

# 3执行命令

1.第一步，安装需要的包，pip默认安装在Scripts中

```shell
Admin@DESKTOP-B648RKM D:\python\Scripts
$ pip install -r D:\download\typoraCracker-master\typoraCracker-master\requirements.txt
```

2.cmd跳转到github下载的文件目录中

把原本的typora中的时间戳校验文件解析出来

```shell
Admin@DESKTOP-B648RKM D:\download\typoraCracker-master\typoraCracker-master
$ python typora.py "D:\Typora\Typora\resources\app.asar" .
```

3.修改时间戳

根据上一步，我们可以得到两个文件夹`dec_app`和`enc_app`。dec是解析文件，enc是加密文件。

查看 `dec_app` 目录的 `License.js` 文件，通过notepad++打开都行。搜索`This beta version of Typora is expired, please download and install a newer version.`

然后把内容修改成下面的，其实就是修改时间戳而已，将时间戳改为4102329600000 ，即2099-12-31。

{{< image classes="fancybox center fig-100" src="/software/typora5.png" thumbnail="/software/typora5.png" title="">}}

4.编译处对应的校验时间戳的文件`app.asar`

```shell
Admin@DESKTOP-B648RKM D:\download\typoraCracker-master\typoraCracker-master
$ python typora.py -u dec_app/ .
```



# 3软件正常打开

替换上面的文件`app.asar`

可以发现按照上面的方法，软件可以正常打开，记录一下，打卡202207161221。

{{< image classes="fancybox center fig-100" src="/software/typora6.png" thumbnail="/software/typora6.png" title="">}}
