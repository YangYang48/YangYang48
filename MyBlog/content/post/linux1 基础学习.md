---
title: "linux1 基础学习"
date: 2022-03-12
thumbnailImagePosition: left
thumbnailImage: linux/linux1_thumb.jpeg
coverImage: linux/linux1_cover.jpeg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 2022
- March 
tags:
- linux
- 手册
- 系统
- Linux基础
showSocial: false
---

Linux是一个操作系统。它有正则表达式、网络协议、系统管理、版本控制、运维、交叉编译，烧录等等功能。所以一名Android开发人员熟练掌握Linux是基本功。

<!--more-->
# Linux常用指令

考虑到笔者当下写的时候可能会有所遗漏，所以会不定期更新~

## 常用指令

|      |      |
| ---- | ---- |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |



## 基本指令

### 两次 Tab 键

补全命令，还可以补全文件名、路径名

- 空格键：用于跳到下一页
- 回车键：用于跳到下一行
- q ：用于退出列表

{{< image classes="fancybox center fig-100" src="/linux/linux1_1.png" thumbnail="/linux/linux1_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_2.png" thumbnail="/linux/linux1_2.png" title="">}}

### history

history 命令可以方便我们了解自己之前输入过那些命令。history 列出的使用过的命令，是有编号的。

{{< image classes="fancybox center fig-100" src="/linux/linux1_3.png" thumbnail="/linux/linux1_3.png" title="">}}

!编号：重新运行对应编号的命令

{{< image classes="fancybox center fig-100" src="/linux/linux1_4.png" thumbnail="/linux/linux1_4.png" title="">}}

### Ctrl + R 

用于查找使用过的命令

之前使用过 ndk这个命令，它就为我自动补全了 ndk 命令

{{< image classes="fancybox center fig-100" src="/linux/linux1_5.png" thumbnail="/linux/linux1_5.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_6.png" thumbnail="/linux/linux1_6.png" title="">}}

### Ctrl + A 

光标跳到一行命令的开头。一般来说，Home 键有相同的效果；

{{< image classes="fancybox center fig-100" src="/linux/linux1_7.png" thumbnail="/linux/linux1_7.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_8.png" thumbnail="/linux/linux1_8.png" title="">}}

### Ctrl + E 

光标跳到一行命令的结尾。一般来说，End 键有相同的效果

{{< image classes="fancybox center fig-100" src="/linux/linux1_8.png" thumbnail="/linux/linux1_8.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_9.png" thumbnail="/linux/linux1_9.png" title="">}}

### Ctrl + U 

删除所有在光标左侧的命令字符

{{< image classes="fancybox center fig-100" src="/linux/linux1_10.png" thumbnail="/linux/linux1_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_11.png" thumbnail="/linux/linux1_11.png" title="">}}

### Ctrl + K 

删除所有在光标右侧的命令字符

{{< image classes="fancybox center fig-100" src="/linux/linux1_10.png" thumbnail="/linux/linux1_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_12.png" thumbnail="/linux/linux1_12.png" title="">}}

### Ctrl + W 

删除光标左侧的一个“单词”，这里的“单词”指的是用空格隔开的一个字符串。例如 -a 就是一个“单词”

{{< image classes="fancybox center fig-100" src="/linux/linux1_10.png" thumbnail="/linux/linux1_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_13.png" thumbnail="/linux/linux1_13.png" title="">}}

### Ctrl + Y 

粘贴用 Ctrl + U、 Ctrl + K 或 Ctrl + W “删除”的字符串，有点像“剪切-粘贴”

{{< image classes="fancybox center fig-100" src="/linux/linux1_10.png" thumbnail="/linux/linux1_10.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_14.png" thumbnail="/linux/linux1_14.png" title="">}}

## 文件组织

区分windows，对于Linux 来说，**一切都是文件**！

{{< image classes="fancybox center fig-100" src="/linux/linux1_15.png" thumbnail="/linux/linux1_15.png" title="">}}

> - bin：英语 binary 的缩写，表示“二进制文件”（我们知道可执行文件是二进制的）。包含了会被所有用户使用的可执行程序；
>
> - boot：英语 boot 表示“启动”，包含与 Linux 启动密切相关的文件；
>
> - dev：英语 device 的缩写，表示“设备”，包含外设。它里面的子目录，每一个对应一个外设。比如代表我们的光盘驱动器的文件就会出现在这个目录下面；
>
> - etc：etc 有点不能顾名思义了。因为 etc 是法语 et cetera 的缩写，翻成英语就是“and so on”，表示“…等等”，包含系统的配置文件。至于为什么在 /etc 下面存放配置文件， 按照原始的 Unix 说法（Linux 文件结构参考 Unix 的教学实现 MINIX），这下面放的都是一堆零零碎碎的东西，类似杂七杂八的目录。
>
> - home：英语 home 表示“家”，用户的私人目录。有点类似 Windows 中的 Documents 这个文件夹，也叫“我的文档”。Linux 中的每个普通用户（除了大管家用户，也就是超级用户 root 外。root 因为太厉害，拥有所有权限，所以比较“任性”，跟普通用户不住在一起）都在 home 目录下有自己的一个私人目录。比如我的用户名是yangyang，那么我的私人目录就是 /home/yangyang；如果另一个用户叫 spiderman，那么他的私人目录就是 /home/spiderman；
>
>   {{< image classes="fancybox center fig-100" src="/linux/linux1_18.png" thumbnail="/linux/linux1_18.png" title="">}}
>
> - lib：英语 library 的缩写，表示“库”，包含被程序所调用的库文件。例如 .so 结尾的文件，在 Windows 下这样的库文件是以 .dll 结尾的；
>
> - media：英语 media 表示“媒体”。当一个可移动的外设（比如 USB 盘、SD 卡、DVD、光盘等等）插入电脑时，Linux 就可以让我们通过 media 的子目录来访问这些外设中的内容。
>
> - mnt：英语 mount 的缩写，表示“挂载”。有点类似 media，但一般用于临时挂载一些装置；
>
> - opt：英语 optional application software package 的缩写，表示“可选的应用软件包”，用于安装多数第三方软件和插件；
>
> - root：英语“根”的意思。超级用户 root 的家目录/主目录。一般用户的家目录是位于 /home 下，不过 root 用户是个例外。之前的课程我们也提到过，root 是整个系统的超级用户，拥有一切权限，在服务器中推荐使用普通用户来执行操作。虽然有root账号，笔者一般还是用普通帐号来登录。
>
>   {{< image classes="fancybox center fig-100" src="/linux/linux1_16.png" thumbnail="/linux/linux1_16.png" title="">}}
>
> - sbin：英语 system binary 的缩写，表示“系统二进制文件”。比起 bin 目录多了一个前缀 system，所以包含的是系统级的重要可执行程序；
>
> - srv：英语 service的缩写，表示“服务”。包含一些网络服务启动之后所需要取用的数据；
>
> - tmp：英语 temporary 的缩写，表示“临时的”。普通用户和程序存放临时文件的地方；
>
> - usr：英语 Unix Software Resource 的缩写，表示“Unix 操作系统软件资源”（也是个历史遗留的命名）。这个目录是最庞大的目录之一。有点类似 Windows 中的 C:\Windows 和 C:\Program Files 这两个文件夹的集合。在这里面安装了大部分用户要调用的程序；
>
> - var：英语 variable 的缩写，表示“动态的，可变的”。通常包含程序的数据，比如一些 log（日志）文件，记录电脑中发生了什么事。
>
> 具体可以点击[这里](https://linuxtoy.org/archives/linux-file-structure.html)，需要科学上网

### pwd

打印当前工作目录

{{< image classes="fancybox center fig-100" src="/linux/linux1_17.png" thumbnail="/linux/linux1_17.png" title="">}}

### which

获取命令的可执行文件的位置

{{< image classes="fancybox center fig-100" src="/linux/linux1_19.png" thumbnail="/linux/linux1_19.png" title="">}}

## 目录指令

### ls

用于列出文件和目录

{{< image classes="fancybox center fig-100" src="/linux/linux1_20.png" thumbnail="/linux/linux1_20.png" title="">}}

终端默认是有颜色标注的，一般来说：

- 蓝色 --> 目录，aosp
- 绿色 --> 可执行文件，ndk_EXE
- 红色 --> 压缩文件，jdk-8u311-linux-aarch64.tar.gz
- 浅蓝色 --> 链接文件
- 灰色 --> 其他文件，core

**ls -l** 

详细列表

{{< image classes="fancybox center fig-100" src="/linux/linux1_21.png" thumbnail="/linux/linux1_21.png" title="">}}

> - total：有一个 total xxx 的，是表示当前目录所有文件的总大小是 total 后面的那个数字所表示的千字节
>
> - 每一列是一个单独的信息
>
>   文件权限：第一列的drwxrwxr-x之类的。具体在下面权限中会提到。
>
>   链接的数目：第二列的 1，2 ，4之类的。
>
>   文件所有者的名称：第三列的 yangyang，前者表示这个文件或目录的所有者是哪个用户。
>
>   文件所在的群组：第四列的 yangyang，表示这个文件或目录的是属于哪个群组。
>
>   文件大小：第五列的 4096 或 640 等等。单位是byte，是英语“字节”的意思。比如 aosp那个目录的大小是 4096个字节。这里列出的文件都是目录，因此这些大小并**没有显示目录中所有文件的总大小**。
>
>   最近一次修改的时间：第六列。比如 aosp那个目录的最近一次修改时间是 Nov  7 16:36，也就是 2021 年 11月 07 日。
>
>   文件或目录的名称：第七列。

**ls -h**

以 Ko，Mo，Go 的形式显示文件大小，更符合人类阅读

{{< image classes="fancybox center fig-100" src="/linux/linux1_22.png" thumbnail="/linux/linux1_22.png" title="">}}

**ls -a**

显示所有文件和目录，包括隐藏的

{{< image classes="fancybox center fig-100" src="/linux/linux1_23.png" thumbnail="/linux/linux1_23.png" title="">}}

**ls -R**

列出现在处于的文件夹中的a文件夹的所有子目录层

{{< image classes="fancybox center fig-100" src="/linux/linux1_24.png" thumbnail="/linux/linux1_24.png" title="">}}

**ls -i**

列出文件名和inode标识，这个inode标识会在链接中详细解析

{{< image classes="fancybox center fig-100" src="/linux/linux1_41.png" thumbnail="/linux/linux1_41.png" title="">}}

### cd

切换目录

```shell
#比如我想去根目录 / 转转
cd /
#返回上一级
cd ..
#默认会回到当前用户的主目录，这里就回到了/home/yangyang目录
#下面三条指令为同一个意思
cd 
cd ~cd
cd /home/yangyang
```



### du

显示目录包含的文件大小（真正意义上的的文件大小）

**du -h：以 Ko，Mo，Go 的形式显示文件大小**

如果目录层级比较多会非常慢，可以得知/home/yangyang目录下的总的文件大小为5.5G

{{< image classes="fancybox center fig-100" src="/linux/linux1_25.png" thumbnail="/linux/linux1_25.png" title="">}}

**du -a：显示文件和目录的大小，同样目录层级比较多会非常慢**

{{< image classes="fancybox center fig-100" src="/linux/linux1_26.png" thumbnail="/linux/linux1_26.png" title="">}}

**du -s：只显示总计大小**

{{< image classes="fancybox center fig-100" src="/linux/linux1_27.png" thumbnail="/linux/linux1_27.png" title="">}}

**du -h --max-depth=1 | sort** 

查看当前目录下所有一级子目录文件夹大小并排序，这里的排序只按数字，不按照算上运算单位

{{< image classes="fancybox center fig-100" src="/linux/linux1_28.png" thumbnail="/linux/linux1_28.png" title="">}}

## 文件指令

### 显示指令

#### cat

一次性显示文件的所有内容

cat命令是**concatenate** 的缩写，表示链接

**cat file1 file2：连接两个文件的内容，将其一并输出**

{{< image classes="fancybox center fig-100" src="/linux/linux1_29.png" thumbnail="/linux/linux1_29.png" title="">}}

**cat -n：显示行号**

{{< image classes="fancybox center fig-100" src="/linux/linux1_30.png" thumbnail="/linux/linux1_30.png" title="">}}

#### more

分页显示文件内容，**并且能显示百分比**

more命令中最基本最常用的快捷键

- 回车键： 向下n行，需要定义。默认为1行
- 空格键/Ctrl + F： 向下滚动一屏
- Ctrl +b键： 返回上一屏
- =号： 输出当前行的行号
- :f 键：输出文件名和当前行的行号
- v键：调用vi编辑器
- !命令：调用Shell，并执行命令
- q 键：退出more

#### less

分页显示文件内容（**推荐使用**）

more有的功能less都有，less功能更强大

less 命令中最基本最常用的快捷键

- 空格键：文件内容读取下一个终端屏幕的行数，相当于前进一个屏幕（页）。很常用的快捷键。与键盘上的 PageDown（下一页）效果一样
- 回车键：文件内容读取下一行，也就是前进一行，与键盘上的向下键效果是一样的
- d 键：前进半页（半个屏幕）
- b 键：后退一页，与键盘上的 PageUp（上一页）效果一样
- y 键：后退一行，与键盘上的向上键效果是一样的
- u 键：后退半页（半个屏幕）
- q 键：停止读取文件，中止 less 命令
- v键：调用默认编辑器nano

{{< image classes="fancybox center fig-100" src="/linux/linux1_34.png" thumbnail="/linux/linux1_34.png" title="">}}

- = 号：显示你在文件中的什么位置（包括第几行到第几行，整个文件所含行数，所含字符数，整个文件所含字符）

{{< image classes="fancybox center fig-100" src="/linux/linux1_31.png" thumbnail="/linux/linux1_31.png" title="">}}

- h 键：显示帮助文档，按 q 键退出帮助文档

{{< image classes="fancybox center fig-100" src="/linux/linux1_32.png" thumbnail="/linux/linux1_32.png" title="">}}

- /（斜杠）：进入搜索模式，只要在斜杠后面输入你要搜索的文字，按下回车键，就会把所有符合的结果都标识出来
- n 键：跳到下一个符合的搜索结果
- N 键(shift + n)：跳到上一个符合的搜索结果

{{< image classes="fancybox center fig-100" src="/linux/linux1_33.png" thumbnail="/linux/linux1_33.png" title="">}}

#### head

显示文件开头，默认情况下，head 会显示文件的**头 10 行**

**head -n 5 file1**

显示指定行数，这里可写成head -5 file1

{{< image classes="fancybox center fig-100" src="/linux/linux1_35.png" thumbnail="/linux/linux1_35.png" title="">}}

#### tail

显示文件结尾

**tail -n 5 file1**

显示指定行数，这里可写成tail -5 file1

{{< image classes="fancybox center fig-100" src="/linux/linux1_36.png" thumbnail="/linux/linux1_36.png" title="">}}

**tail -f file1**

追加内容，如果有，就显示新增内容，常用于日志更新

{{< image classes="fancybox center fig-100" src="/linux/linux1_37.png" thumbnail="/linux/linux1_37.png" title="">}}

指定间隔检查的秒数，用 -s 参数

```shell
#每隔 4 秒检查一次文件是否有更新
tail -f -s 4 file1      
```



### 创建指令

#### touch

创建空白文件，文件名尽可能不带空格，否则会认为两个文件

{{< image classes="fancybox center fig-100" src="/linux/linux1_38.png" thumbnail="/linux/linux1_38.png" title="">}}

#### mkdir

创建目录

**mkdir -p**

参数来递归地创建目录结构

{{< image classes="fancybox center fig-100" src="/linux/linux1_39.png" thumbnail="/linux/linux1_39.png" title="">}}

### 操作指令

#### cp

拷贝文件或目录，可使用正则表达式

**cp src_file one/dst_file_copy**

拷贝文件

**cp -r src_file dst_file_copy**

拷贝文件夹

#### mv

移动/重命名文件

> cp 命令就好比 Windows 中的**复制+粘贴**，而 mv 命令就好比 Windows 中的**剪切+粘贴**。

**mv src_file dst_file_move**

这个src_file文件移动到dst_file_move这个目录

**mv old_filename renamed_file**

如果renamed_file不存在，那么即为重命名

#### rm

删除文件

**rm -i file**

向用户确认是否删除

{{< image classes="fancybox center fig-100" src="/linux/linux1_40.png" thumbnail="/linux/linux1_40.png" title="">}}

**rm -rf** 

强制递归删除

> 禁止输入命令rm -rf /* 或者 rm -rf /，会使得整个系统删除。特别是在root状态下执行。



#### ln

创建链接，类似win的快捷方式

**ln org_file1 hard_file2**

创建硬链接,org_file1 hard_file2拥有相同的inode标识。

```shell
ln ndk_EXE NDK_EXE
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_52.png" thumbnail="/linux/linux1_52.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_42.png" thumbnail="/linux/linux1_42.png" title="">}}

**ln -s org_file1 soft_file2**

创建软连接，会有->标识指向

```shell
ln -s ndk_EXE s_NDK_EXE
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_51.png" thumbnail="/linux/linux1_51.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/linux1_43.png" thumbnail="/linux/linux1_43.png" title="">}}

> Physical link：物理链接或硬链接，类似指针或引用，拥有相同的指向文件内容，删除其中之一不会使得文件内容消失。只能是文件。
>
> Symbolic link：符号链接或软链接，类似快捷方式，快捷方式删除不影响，但是快捷方式指向的文件删除，会使得快捷方式失效。可以是文件或者文件夹。

### 查找指令

#### locate 

#### find 



### 数据指令

#### grep

筛选数据

#### sort

为文件排序

#### wc

文件的统计

#### uniq

删除文件中的重复内容

#### cut

剪切文件的一部分内容

## 时间指令

1. date 命令：调节时间
2. at 命令：延时执行一个程序
3. sleep 命令：休息一会
4. crontab 命令：定时执行程序

## 压缩指令

1. tar 命令：将多个文件归档
2. gzip 和 bzip2 命令：压缩归档
3. zip / unzip 和 rar / unrar 命令：压缩 / 解压 zip 和 rar 文件

## 网络指令

1. 网络链接

   用 SSH 创建一个安全的通信管道

   用 SSH 进行连接

2. 网络传输

   wget ：下载文件

   scp ：网间拷贝

   ftp & sftp ：传输文件

   rsync ：同步备份

3. 网络分析

   host 和 whois 命令：告诉我你是谁

   ifconfig 和 netstat 命令：控制和分析网络流量

   netstat : 网络统计

   iptables / nftables ：防火墙

4. 网络安装软件

   大多数 Linux 发行版的软件都可以用包管理工具 apt / apt-get 来安装（对于 Debian 一族）；

   有些软件不能通过 apt / apt-get 来安装，因为没有被收录到 Ubuntu 的软件仓库中。在这种情况下，我们可以试着在网上找软件的 deb 安装包；

   假如前两种方法都不行，我们只能选择从源代码编译安装的方法。一般通用的步骤如下：

A. 从网上下载程序的源代码（通常被打包压缩为 .tar.gz 的格式）；

B. 解压压缩包（`tar zxvf xxx.tar.gz`）；

C. 运行解压之后的文件夹里的 configure 文件： `./configure`；

D. 运行 `make` 来编译；

E. 运行 `sudo make install` 完成安装。

# 进程

1. w 命令：都有谁，在做什么？

2. ps 命令和 top 命令：列出运行的进程

3. top：进程的动态列表

4. Ctrl + C 和 kill 命令：停止进程

5. halt 命令和 reboot 命令：停止和重启系统

6. 前后台

   & 符号和 nohup 命令：后台运行进程

   Ctrl + Z，jobs，bg 和 fg 命令：控制进程的前后台切换

# 数据流

## 重定向

1. “> 和 >>”：重定向到文件
2. “2>，2>>，2>&1”：重定向错误输出

## 管道



# man

1. 可执行程序或 Shell 命令；
2. 系统调用（Linux 内核提供的函数）；
3. 库调用（程序库中的函数）；
4. 特殊文件（通常在 /dev 下）；
5. 文件（例如 /etc/passwd）；
6. 游戏；
7. 杂项（比如 man (7)，groff (7)）；
8. 系统管理命令（通常只能被 root 用户使用）；
9. 内核子程序。




# 猜你喜欢

[linux2 权限和管理](https://yangyang48.github.io/2022/04/linux2-%E6%9D%83%E9%99%90%E5%92%8C%E7%AE%A1%E7%90%86/)

[linux3 Vim和shell脚本](https://yangyang48.github.io/2022/04/linux3-vim%E5%92%8Cshell%E8%84%9A%E6%9C%AC/)

[linux4 NDK交叉编译](https://yangyang48.github.io/2022/03/linux4-ndk%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91/)



# 参考

[[1] Oscar, Linux命令行与Shell脚本编程大全,2019.](http://www.imooc.com/read/39)

[[2] onebigday, linux 中把命令放到后台执行的方法,2020.](https://blog.csdn.net/onebigday/article/details/107896918)

[[3] 独L无二, Vim查找全匹配字符串,2020.](https://blog.csdn.net/sinat_31275315/article/details/104535192)

[[4] 一生只画眉, Shell if 条件判断,2018.](https://blog.csdn.net/zhan570556752/article/details/80399154)

[[5] Y_H_, linux 实时线程优先级问题——数值越大优先级越高吗？,2016.](https://blog.csdn.net/u010479322/article/details/51347245)

[[6] 伏特加的滋味, linux下创建用户和添加用户权限,2017.](https://blog.csdn.net/yuanyuan214365/article/details/75153928)

[[7]panqidong95, Linux添加用户及用户权限管理,2019.](https://blog.csdn.net/panqidong95/article/details/95517278)

[[8]panqidong95, linux adduser,2016.]([拼尽全力前进](https://blog.csdn.net/gaoqiao1988))