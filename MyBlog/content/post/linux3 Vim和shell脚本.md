---
title: "linux3 Vim和shell脚本"
date: 2022-04-05
thumbnailImagePosition: left
thumbnailImage: linux/linux3_thumb.jpg
coverImage: linux/linux3_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 2022
- April 
tags:
- linux
- 手册
- 系统
- Vim
- Shell脚本
showSocial: false
---

在Linux文本编辑器的江湖中一直有一个神器Vim，这个神奇可以满足日常编辑的所有需求。通过Vim可以进一步的学习关于Shell的脚本，提升操作Linux效率。

<!--more-->
# 1文本编辑器

## 1.1Nano

一个自带的文本编辑器（大多数情况下不使用）

快捷键在编辑器的底部

{{< image classes="fancybox center fig-100" src="/linux/linux1_66.png" thumbnail="/linux/linux1_66.png" title="Nano界面图">}}

## 1.2配置文件/etc/nanorc

> Linux 或 Unix 的许多程序在启动时，都需要 rc 后缀的初始文件或配置文件。rc，它是` runcomm `的缩写，即` run command`（运行命令）的简写。熟悉Android架构的小伙伴会比较熟悉，系统各种服务都是靠着rc文件启动的。

```shell
#/etc/nanorc
set historylog#激活历史日志
set locking#激活锁定
set nowrap#激活不隐藏输入
set suspend#激活拼接模式
include "/usr/share/nano/*.nanorc"
```

一共五条语句，每一行一句配置语句，配置语句是以 set（用于激活。set 是英语“放置，设置”的意思）或 unset（用于关闭）开头，后接你要配置的项目。

```shell
#激活鼠标
set mouse
#激活首行缩进
set autoindent
```

> 对于每个用户来说，家目录下的 .bashrc 文件的优先级比系统的 /etc/bash.bashrc 文件高。例如同样的配置选项，如果 .bashrc 和 /etc/bash.bashrc 不同，那么以 .bashrc 的为准。同样的原则也适用于其它配置文件，例如 .nanorc 和 /etc/nanorc。
>
> 
>
> 如果我们修改了 .bashrc 和 profile 文件后，默认是在用户下次登录系统时才能生效。但是我们可以用 source 命令来使改动立即生效
>
> ```shell
> source .bashrc
> source .profile
> ```



## 1.2Vim

 文本编辑器的进阶版，比前者更加简单实用。（实际用得比较多）

### 1.2.1Vim的界面

打开vim之后的界面如下图所示，可以看到有光标，界面，还有尾行说明。

光标指向的地方，会在尾行说明中体现出来，前面1,1代表第一行第一列，后面会显示百分比或者是Top(文本的头部)或者是Bot(文本的尾部)。

{{< image classes="fancybox center fig-100" src="/linux/linux3_2.png" thumbnail="/linux/linux3_2.png" title="Vim界面图">}}

### 1.2.2Vim的相关配置

```shell
#定义自己的vim
cp /etc/vim/vimrc ~/.vimrc
#./vimrc
#配置语法高亮
syntax on
#背景着色为黑色
set background=dark
#显示行号
set number
#显示当前命令
set showcmd
#忽略大小写搜索
set ignorecase
#设置鼠标使用
set mouse=a
```

以下对Vim的特性和shell脚本展开说明。



# 2Vim 

## 2.1Vim的三种工作模式

1. 交互模式：Interactive Mode。这是 Vim 的默认模式，每次我们运行 Vim 程序的时候，就会进入这个模式。复制粘贴文本，跳转到指定行，撤销操作等等

2. 插入模式：Insert Mode。这就是我们熟悉的文本编辑器的“一贯作风”。我们输入文本，文本就被插入到光标所在之处。为了进入这个模式，有几种方法，最常用的方法是按字母键 i（i 是 insert 的首字母，是英语“插入”的意思，除了i之外，还有五种a，A，s，S，I）。为了退出这种模式，只需要按下 Esc 键（一般在键盘左上角）。Esc 是 escape 的缩写，是英语“脱离，逃脱”的意思。左下角有一个 `-- INSERT --`，说明我们就处在 Insert Mode（插入模式）中。

   > 关于插入的i，I，a，A，s和S的区别
   >
   > 1.**i是当前插入**
   >
   > 2.**I是当前行首插入**
   >
   > 3.**a会往后移动一个字符插入**
   >
   > 4.**A是当前行尾插入**
   >
   > 5.**s是删除当前字符插入**
   >
   > 6.**S是删除当前行插入**

3. 命令模式：Command Mode。也有称之为底线命令模式（Last line mode）的。这个模式下，我们可以运行一些命令，例如“退出”、“保存”等等。也可以用这个模式来激活一些 Vim 的配置（例如语法高亮、显示行号等等）。

{{< image classes="fancybox center fig-100" src="/linux/linux1_67.png" thumbnail="/linux/linux1_67.png" title="">}}

### 2.1.1命令模式

表2.1命令模式输入

| 命令模式输入                 | 含义                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| :0                           | 移动到行首                                                   |
| :$                           | 移动到行末                                                   |
| :w                           | 保存文件                                                     |
| :q                           | 直接退出                                                     |
| :q!                          | 强制退出，不保存修改                                         |
| :wq/:x                       | 保存然后退出                                                 |
| :行号                        | 跳转到指定行                                                 |
| /                            | 查找。<br />n 键，查找下一个匹配项<br />Shift + n，查找上一个匹配 |
| `/\<string_to_exact_match\>` | 查找全匹配字符串                                             |
| :s                           | 查找并替换                                                   |
| :s/旧字符串/新字符串         | 替换光标所在行的第一个匹配的字符串                           |
| :s/旧字符串/新字符串/g       | 替换光标所在行的所有匹配的旧字符串为新字符串                 |
| :%s/旧字符串/新字符串/g      | 替换文件中所有匹配的字符串                                   |
| :sp                          | split，横向分屏                                              |
| :vsp                         | 垂直分屏                                                     |
| :!命令                       | 运行外部命令                                                 |

除了上面常用之外，还有以命令模式激活选项参数，可以参考1.2.2的Vim的相关配置。

```shell
# “短暂性”配置选项参数
:set 选项名
# 而不激活（取消）一个选项参数
:set no 选项名
# 查询选项的参数
:set 选项名?
# 输出结果为background=light
:set background?

# 几个常见的设置命令
# 显示当前命令
:set showcmd
# 显示行号
:set number
# 在查找时忽略大小写
:set ignorecase
# 鼠标支持
:set mouse=a
```



### 2.1.2交互模式

表2.2交互模式控制方向的按键

| 按键   | 作用             |
| ------ | ---------------- |
| h/左键 | 向左移动一个字符 |
| j/下键 | 向下移动一个字符 |
| k/上键 | 向上移动一个字符 |
| l/右键 | 向右移动一个字符 |

表2.3交互模式输入

| 交互模式命令     | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| w                | 一个单词一个单词地移动                                       |
| x                | 删除字符                                                     |
| dd               | 删除行，可以结合p粘贴一起使用                                |
| dw               | 删除一个单词                                                 |
| d0               | 删除从光标处到行首的所有字符                                 |
| d$               | 删除从光标处到行末的所有字符                                 |
| yy               | yank，复制行到内存中。和 `dd` 类似，`dd` 用于“剪切”光标所在行到内存中，而 `yy` 是“复制”。可以结合p粘贴一起使用 |
| yw               | 复制一个单词                                                 |
| y$               | 复制从光标所在处到行末的所有字符                             |
| y0               | 复制从光标所在处到行首的所有字符                             |
| p                | 粘贴，可以配合dd或者yy                                       |
| r                | 替换一个字符                                                 |
| u                | 撤销操作                                                     |
| Ctrl + r         | 取消撤销，也就是重做之前的修改，redo                         |
| Shift + g        | 要跳转到最后一行                                             |
| gg               | 要跳转到第一行                                               |
| 行号 + Shift + g | 跳转到指定行(间隔时间短，不好控制，可以直接进入命令模式:行号跳转) |
| 行号 +gg         | 跳转到指定行(间隔时间短，不好控制，可以直接进入命令模式:行号跳转) |



### 2.1.3交互模式

考虑到插入模式里面可以多用来输入，所以本文不具体展开。

### 2.1.4 其他

除了上述模式用到的常用命令之外，还可以通过按F1，获取更多的信息。

{{< image classes="fancybox center fig-100" src="/linux/linux3_21.png" thumbnail="/linux/linux3_21.png" title="">}}

例如，跳转到指定行除了上述两种方式之外，还可以在vim的时候添加行号**vim file +行号**

{{< image classes="fancybox center fig-100" src="/linux/linux3_28.png" thumbnail="/linux/linux3_28.png" title="">}}



# 3shell

shell 是英语“壳，外壳”的意思。你可以把它想象成嵌入在 Linux 这样的操作系统中的一个“微型编程语言”。

Shell 不像 C 语言，C++，Java 等编程语言那么完整，不需要安装，不需要编译，但是 Shell 这门语言可以帮我们完成很多自动化任务，例如：保存数据，监测系统的负载等等。

## 3.1主流的shell

几种主流的 Shell：

- Sh : Bourne Shell 的缩写。可以说是目前所有 Shell 的祖先。
- Bash : Bourne Again Shell 的缩写， 可以看到比 Bourne Shell 多了一个 again。again 在英语中是“又，再，此外”的意思，说明 Bash 是 Sh 的一个进阶版本，比 Sh 更优秀。Bash 是目前大多数 Linux 发行版和苹果的 macOS 操作系统的默认 Shell。
- Ksh : Korn Shell 的缩写。一般在收费的 Unix 版本上比较多见，但也有免费版本的。
- Csh : C Shell 的缩写。 此 Shell 的语法有点类似 C 语言。
- Tcsh : Tenex C Shell 的缩写。Csh 的优化版本。
- Zsh : Z Shell 的缩写。比较新近的一个 Shell，集 Bash，Ksh 和 Tcsh 各家之大成。

{{< image classes="fancybox center fig-100" src="/linux/linux1_68.png" thumbnail="/linux/linux1_68.png" title="">}}

可以看到，箭头的源头就是来自Sh，所有的shell都是源于Sh，目前主流用的最多的就是Bash。

## 3.2shell的用途

事实上，Shell 提供了所有可以让你运行命令的基础功能。通过Shell 这个桥梁，去**操控作用系统内核**（内核的英语是 kernel）。

{{< image classes="fancybox center fig-100" src="/linux/linux1_69.png" thumbnail="/linux/linux1_69.png" title="">}}

为了切换Shell，需要用到以下命令：

```shell
chsh
```



# 4shell脚本

我们之前用得到所有命令，都是依赖于shell的。有时候我们需要去批量，自动化的处理一些交互事务，每次手动去敲命令显得有点突兀，这个时候需要用到的是shell脚本。

## 4.1创建脚本文件

我们用 Vim 这个文本编辑器来创建一个 Shell 脚本文件，这里可以命名sh的后缀，强调这是一个 Shell 脚本文件。

```shell
vim test.sh
```

脚本有特定的语法

```shell
# 指定运行脚本用得shell
#!/bin/bash
#输入最简单的指令
ls
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_70.png" thumbnail="/linux/linux1_70.png" title="">}}

## 4.2运行脚本

```shell
# 添加脚本可执行权限
chmod +x test.sh
#运行脚本
./test.sh
```

结果

{{< image classes="fancybox center fig-100" src="/linux/linux1_71.png" thumbnail="/linux/linux1_71.png" title="">}}

## 4.3调试模式运行

会对shell脚本里的每条指令**单独分析**。其中每一条命令前面都会有一个标识符`+`，然后对应会有命令的结果。

```shell
bash -x test.sh
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_72.png" thumbnail="/linux/linux1_72.png" title="">}}

## 4.4基本运算

### 4.4.1定义变量

```shell
#特别注意，在等号两边不要加空格。
#错误 a = "Hello world"
a="Hello world"
```

### 4.4.2显示内容

```shell
echo "Hello World"
#-e 参数，为了使“转义字符”发生作用
echo -e "First line\nSecond line"
```

### 4.4.3显示变量

```shell
echo $a
#下面显示变量的方式有点类似kotlin的用法
echo "The message is $a"
```

### 4.4.4引号

分成三种，单引号，双引号和反引号。

1. 单引号：忽略被它括起来的所有特殊字符

2. 双引号：忽略大多数特殊字符，除了`$`，`\` ，`(美元符号，反斜杠，反引号)

3. 反引号：要求 Shell 执行被它括起来的内容


```shell
# 单引号
a='Hello World'
echo 'The message is $a'
echo "The message is $a"
dir=`pwd`
echo "The dir is $dir"
```

### 4.4.5请求输入

类似c++中的cin

```shell
read msg
echo "Hello, $msg"
read Date Time
echo "today is $a, $b"
# 显示输入提示
read -p 'This is a shell learn' action
echo "my shell echo :$action"
# 限制字符数
read -p 'name limit is 8' -n 8 limit_length
echo "\nmy shell echo :$limit_length"
# 限制输入事件
read -p 'time is limit 10 second, you must hurry up' -t 10 limit_time
echo -e "\nBoom !"
echo "\nmy shell echo :$limit_time"
# 隐藏输入内容
read -p 'you input secret :' -s password
echo "\nOk, passwd is $password"
```

###  4.4.6数学运算

在 Bash 中，**所有的变量都是字符串**，需要特定字符让其运算。

```shell
# let 命令可以用于赋值
let "num1 = 1"
let "num2 = 1"
let "num3 = num1 + num2"
echo "num3 = $num3"
```

具体数学运算如下表所示

| 运算 | 符号 |
| ---- | ---- |
| 加   | +    |
| 减   | -    |
| 乘   | *    |
| 除   | /    |
| 幂   | **   |
| 余   | %    |

这些跟基本的编程符号相似，所以不展开描述。

```shell
# 下面两种方式是一样的
a=1
let "a = a * 3"
let "a *= 3"
```

### 4.4.7任意精度计算

bc 命令是任意精度计算器语言，通常在linux下当计算器用

```shell
# 方式1，直接输入bc，在bc模式下运算，一般根据你的输入精度来对应输出精度，通过输入quit回车后退出
bc
sqrt(3.0000000000*10^0)
1.7320508075
# 方式2，通过管道
# scale=2，代表保留2位
echo 'scale=2; (2.777 - 1.4744)/1' | bc
# 通过ibase和Obase来进制转化，原始进制位ibase，目标进制位Obase
echo "ibase=2;111" |bc
abc=192 
echo "obase=2;$abc" | bc
abc=11000000 
echo "obase=10;ibase=2;$abc" | bc
```

### 4.4.8环境变量

几个常见的环境变量，默认环境变量都是大写的

> SHELL：指明目前你使用的是哪种 Shell。我目前用的是 Bash（因为 `SHELL=/bin/bash`）。
>
> PATH：是一系列路径的集合。只要有可执行程序位于任意一个存在于 PATH 中的路径，那我们就可以直接输入可执行程序的名字来执行。
>
> ```shell
> echo $PATH
> ```
>
> {{< image classes="fancybox center fig-100" src="/linux/linux3_22.png" thumbnail="/linux/linux3_22.png" title="">}}
>
> HOME：你的家目录所在的路径。
>
> PWD：目前所在的目录。

#### 4.4.8.1环境变量的分类

环境变量可以简单的分成用户自定义的环境变量以及系统级别的环境变量。

- 用户级别环境变量定义文件：`~/.bashrc`、`~/.profile`（部分系统为：`~/.bash_profile`）
- 系统级别环境变量定义文件：`/etc/bashrc`、`/etc/profile`(部分系统为：`/etc/bash_profile`）、`/etc/environment`

另外在用户环境变量中，系统会首先读取`~/.bash_profile`（或者`~/.profile`）文件，如果没有该文件则读取`~/.bash_login`，根据这些文件中内容再去读取`~/.bashrc`。

#### 4.4.8.2环境变量加载顺序

具体顺序如下图所示，其中红色代表系统环境变量文件，绿色是当前用户文件

{{< image classes="fancybox center fig-100" src="/linux/linux3_23.png" thumbnail="/linux/linux3_23.png" title="">}}

目前开始的时候或从/etc/envirnment开始加载，然后加载/etc/profile和~/.profile。

其中/etc/profile获取加载/etc/bash.bashrc和/etc/profile.d/*.sh

```shell
# /etc/profile,系统的环境变量
# 这里先后加载了/etc/bash.bashrc和/etc/profile.d/*.sh
if [ "$PS1" ]; then
  if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
    ...
fi

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
```

然后~/.profile会加载~/.bashrc，可以知道每次运行shell的时候都会去加载一次~/.bashrc。从`~/.profile`文件中代码不难发现，`~/.profile`文件**只在用户登录的时候读取一次**，而`~/.bashrc`会在每次运行`Shell`脚本的时候读取一次

```shell
# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi

PATH="$HOME/bin:$HOME/.local/bin:$PATH"
```

#### 4.4.8.3环境变量配置

在配置之前，需要确定原先有哪些环境配置

```shell
export
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_24.png" thumbnail="/linux/linux3_24.png" title="">}}

这里罗列6种方式，有命令方式和修改配置文件方式，通常设置配置文件为永久性的修改

1. `export PATH`

   使用`export`命令直接修改`PATH`的值，例如配置jdk的路径。

   ```shell
   export PATH=/home/yangyang/jdk1.8.0_311/bin:$PATH
   # 或者另外一种写法
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：立即生效
   > - 生效期限：当前终端有效，窗口关闭后无效
   > - 生效范围：仅对当前用户有效
   > - 配置的环境变量中不要忘了加上原来的配置，即`$PATH`部分，避免覆盖原来配置

2. `vim ~/.bashrc`

   通过修改用户目录下的`~/.bashrc`文件进行配置

   ```shell
   # 在最后一行加上
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：使用相同的用户打开新的终端时生效，或者手动`source ~/.bashrc`生效
   > - 生效期限：永久有效
   > - 生效范围：**仅对当前用户有效**
   > - 如果有后续的环境变量加载文件覆盖了`PATH`定义，则可能不生效

3. `vim ~/.profile`

   和修改`~/.bashrc`文件类似，也是要在文件最后加上新的路径

   ```shell
   # 在最后一行加上
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：使用相同的用户打开新的终端时生效，或者手动`source ~/.profile`生效
   > - 生效期限：永久有效
   > - 生效范围：**仅对当前用户有效**
   > - 如果没有`~/.profile`文件，则可以编辑`~/.profile`文件或者新建一个

4. `vim /etc/bashrc`

   该方法是修改系统配置，需要管理员权限（如root）或者对该文件的写入权限，笔者没有这个文件，所以这一步修改不了

   ```shell
   # 在最后一行加上
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：新开终端生效，或者手动`source /etc/bashrc`生效
   > - 生效期限：永久有效
   > - 生效范围：对所有用户有效

5. `vim /etc/profile`

   该方法修改系统配置，需要管理员权限或者对该文件的写入权限，通常笔者配置环境变量都选择用该方式

   ```shell
   # 在最后一行加上
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：新开终端生效，或者手动`source /etc/profile`生效
   > - 生效期限：永久有效
   > - 生效范围：对所有用户有效

6. `vim /etc/environment`

   该方法是修改系统环境配置文件，需要管理员权限或者对该文件的写入权限

   ```shell
   # 在最后一行加上
   export PATH=$PATH:/home/yangyang/jdk1.8.0_311/bin
   ```

   > - 生效时间：新开终端生效，或者手动`source /etc/environment`生效
   > - 生效期限：永久有效
   > - 生效范围：对所有用户有效

### 4.4.9参数变量

```shell
./variable.sh 参数1 参数2 参数3 ...
```

> $# ：**包含参数的数目**。
> $0 ：包含被运行的脚本的名称 （我们的示例中就是 variable.sh ）。
> $1：**包含第一个参数**。
> $2：包含第二个参数。
> …
> $8 ：包含第八个参数。

shift参数移位

```shell
#!/bin/bash

echo "The first parameter is $1"
shift
echo "The first parameter is now $1"
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_25.png" thumbnail="/linux/linux3_25.png" title="">}}



### 4.4.10数值变量

shell中默认把变量值当作字符串，例如：

```shell
age=22
age=${age}+1
echo ${age}
```

输出结果为22+1，而不是23。

原因是shell将其解释为字符串，而不是数学运算。

有两种方式使其进行数学运算。

```shell
# 用let命令使其进行数学运算
let age=${age}+1
# 用declare把变量定义为整型
declare -i age=22
```

此后每次运算，都把age的右值识别为算术表达式或数字。



### 4.4.11数组

定义数组

```shell
array=('value0' 'value1' 'value2')
```

访问数组元素

```shell
${array[2]}
```

对数组元素赋值

```shell
array[3]='value3'
```



> Shell 中的数组的下标（index）也基本是从 0 开始的，而不是从 1 开始。因此，第一个元素的编号（下标）就是 0，第二个元素的下标就是 1，以此类推。
> 不过，也不是所有 Shell 语言的数组下标都是从 0 开始，不少 Shell 语言（例如 Csh，Tcsh，Zsh，等等）的数组下标是从 1 开始的。

可以一次性显示数组中所有的元素值，需要用到通配符 `*`（星号）

```shell
#!/bin/bash

array=('value0' 'value1' 'value2')
array[5]='value5'
echo ${array[*]}
echo ${array[@]}
# 获取数组元素个数为: ${#array[@]}或者${#array[*]}
echo ${#array[@]}
echo ${#array[*]}
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_26.png" thumbnail="/linux/linux3_26.png" title="">}}

关于shell数组的进阶，包括二维数组，详见[这里](https://blog.csdn.net/ee230/article/details/48316317?_t_t_t=0.856576693346367)，本文不展开描述。

> 特别注意：
>
> ${array[@]}和${array[*]} 一般情况下都能显示全部的数组元素，但是要实现二维数组的效果，不能使用${array[*]}。

### 4.4.12特殊变量

表4.4.1特殊变量

| 特殊变量 | 含义                                                         |
| :------: | ------------------------------------------------------------ |
|    $0    | 当前脚本的文件名                                             |
|   $num   | num为从1开始的数字<br />$1是第一个参数，$2是第二个参数，${10}是第十个参数 |
|    $#    | 传入脚本的参数的个数                                         |
|    $*    | 所有的位置参数(作为单个字符串)                               |
|    $@    | 所有的位置参数(每个都作为独立的字符串)                       |
|    $?    | 当前shell进程中，上一个命令的返回值<br />如果上一个命令成功执行则$?的值为0，否则为其他非零值，常用做if语句条件 |
|    $$    | 当前shell进程的pid                                           |
|    $!    | 后台运行的最后一个进程的pid                                  |
|    $-    | 显示shell使用的当前选项                                      |
|    $_    | 之前命令的最后一个参数                                       |



## 4.5条件运算

### 4.5.1if 条件语句

```shell
if [ condition ]
then 
    dosomething
fi
```

可以看到condition这个条件为真的时候，就可以dosomething了。

> 注意：方括号 `[]` 中的 `条件测试` 两边必须要空一格。不能写成 `[condition]`，而要写成 `[ condition ]`。

除了上述的写法之外，还有另外一种常见的写法

```shell
if [ condition ]; then
    dosomething
fi
```

如果存在多个if else判断的话，那就是下面的写法方式

```shell
if [ condition1 ];then
    dosomething1
elif [ condition2 ];then
    dosomething2
else
    dosomething3
fi
```

表4.5.1数值的判断

| 数值判断语句  | 判断说明                     |
| :-----------: | ---------------------------- |
| NUM1 -eq NUM2 | NUM1和NUM2两数相等为真 ,=    |
| NUM1 -ne NUM2 | NUM1和NUM2两数不相等为真 ,<> |
| NUM1 -gt NUM2 | NUM1大于NUM1为真 ,>          |
| NUM1 -ge NUM2 | NUM1大于等于NUM1为真,>=      |
| NUM1 -lt NUM2 | NUM1小于NUM1为真 ,<          |
| NUM1 -le NUM2 | NUM1小于等于NUM1为真,<=      |

表4.5.2逻辑判断

| 逻辑判断语句 | 判断说明 |
| :----------: | :------: |
|      -a      |    与    |
|      -o      |    或    |
|      !       |    非    |

表4.5.3字符串的判断

|      字符串判断语句      | 判断说明                         |
| :----------------------: | -------------------------------- |
|      [ -z STRING ]       | 如果STRING的长度为零则为真,len=0 |
| [ -n STRING ]/[ STRING ] | 如果STRING的长度非零则为真,len>0 |
|  [ STRING1 = STRING2 ]   | 如果两个字符串相同则为真         |
|  [ STRING1 != STRING2 ]  | 如果字符串不相同则为真           |

> **“等于”是用一个等号（ = ）来表示的**，C 语言中“等于”是用两个等号（ == ）来表示的。但 Shell 中用两个等号来表示“等于”的判断也是可以的。

表4.5.4文件/文件夹的判断

|   文件/文件夹判断   | 判断说明                                                     |
| :-----------------: | ------------------------------------------------------------ |
|     [ -e FILE ]     | 如果 FILE 存在则为真                                         |
|     [ -f FILE ]     | 如果 FILE 存在且是一个普通文件则为真                         |
|     [ -s FILE ]     | 如果 FILE 存在且大小不为0则为真                              |
|     [ -r FILE ]     | 如果 FILE 存在且是可读的则为真                               |
|     [ -w FILE ]     | 如果 FILE存在且是可写的则为真                                |
|     [ -x FILE ]     | 如果 FILE 存在且是可执行的则为真                             |
|     [ -b FILE ]     | 如果 FILE 存在且是一个块特殊文件则为真                       |
|     [ -c FILE ]     | 如果 FILE 存在且是一个字特殊文件则为真                       |
| [ FILE1 -nt FILE2 ] | 如果 FILE1的最近修改时间比FILE新<br />或者如果 FILE1存在，FILE2不存在则为真。 |
| [ FILE1 -ot FILE2 ] | 如果 FILE1的最近修改时间比 FILE2 老<br />或者 FILE2 存在且，FILE1 不存在则为真。 |
| [ FILE1 -ef FILE2 ] | 如果 FILE1 和 FILE2 指向相同的设备和节点号则为真             |
|     [ -d DIR ]      | 如果 FILE 存在且是一个目录则为真                             |

上面的表知识列举了比较常见的判断，如果遇到非常见的判断，还是需要手册查询。

> - 一.[] 和 test
>
>   两者是一样的。`test expr`和 `[ expr ]` 的效果相同。
>
>   一般常用于**判断文件**、**判断字符串**、**判断整数**。
>
>   字符串比较还是整数比较都千万不要使用 `<`, `>`
>
> ```shell
> #!/bin/bash
> var1="1"
> var2="2"
> 
> # 下面是并且的运算符-a，另外注意，用一个test命令就可以了，还有if条件后面的分号
> if test $var1 = "1" -a $var2 = "2" ; then
>   echo "equal"
> fi
> # 下面是或运算符 -o，有一个为真就可以
> if test $var1 != "1" -o $var2 != "3" ; then
>   echo "not equal"
> fi
> # 下面是非运算符 ！,if条件是为真的时候执行，如果使用！运算符，那么原表达式必须为false
> if ! test $var1 != "1"; then
>   echo "not 1"
> fi
> ```
>
> {{< image classes="fancybox center fig-100" src="/linux/linux3_3.png" thumbnail="/linux/linux3_3.png" title="">}}
>
> - 二.[[ ]]
>
>   支持字符串的模式匹配（使用 `=~` 操作符时甚至支持shell的正则表达式）。逻辑组合可以不使用`test` 的 `-a` , `-o` 而使用 `&&`, `||` 。
>
>   字符串比较时可以把右边的作为一个模式字符串，不仅仅是一个字符串，比如 `[[ hello == hell? ]]`，结果为真。如果右边的字符串加了双引号，则认为是一个文本字符串。
>
> - 三.(()) 和 let
>
>   两者是一样的(`双括号` 比 `let`稍弱一些)。都是一种结构拓展，用于计算算术表达式的值。
>
>   1.直接使用熟悉的 `<` , `>` 等比较运算符。
>
>   2.可以直接使用变量名如 `var` 而不需要 `$var` 这样的形式
>
>   3.支持分号隔开的多个表达式
>
>   4.如果表达式值为0，会返回1；如果是非零值的表达式，返回一个0
>
> - 四.if中[[ ]]和[ ]区别
>
>   1.`[[]]`是关键字，许多 shell (如 ash bsh)并不支持这种方式。
>
>   `[]` 是一条shell命令， 与 `test` 等价，大多数 shell 都支持。
>
>   2.`[[]]` 结构比 Bash 版本的 `[]` 更通用。`&&`, `||` , `<` 和 `>` 操作符能在一个 `[[]]` 测试里通过，但在 `[]` 结构会发生错误。
>
>   3.`[[ ... && ... && ... ]]` 和 `[ ... -a ... -a ...]` 不一样，`[[ ]]` 是逻辑短路操作，而 `[ ]` 不会进行逻辑短路
>   
>   4.一般比较两个字符串相等，推荐方式，避免出现`"[: =: unary operator expected"`，原因是可能出现未定义的字符串，字符串为空。
>   
>   ```shell
>   # 不推荐的方式[ $STATUS = "OK" ]
>   # 推荐方式
>   [[ "$STATUS"x == "OK"x ]]
>   ```
>
> 举例说明运算
>
> ```shell
> # example1 a>b且a<c，下面三种方式等价
> if (( a > b )) && (( a < c ))
> if [[ $a > $b ]] && [[ $a < $c ]]
> if [ $a -gt $b -a $a -lt $c ]
> # example2 a>b或a<c，下面三种方式等价
> if (( a > b )) || (( a < c ))
> if [[ $a > $b ]] || [[ $a < $c ]]
> if [ $a -gt $b -o $a -lt $c ]
> ```



### 4.5.2case条件语句

linux编程语言中的 switch 语句

```shell
#!/bin/bash

case $1 in
    "Cupcake")
        echo "Hello Cupcake !"
        ;;
    "Donut")
        echo "Hello Donut !"
        ;;
    "Éclair")
        echo "Hello Éclair !"
        ;;
    "Froyo")
        echo "Hello Froyo !"
        ;;
    *)
        echo "Sorry, I do not know which version.Maybe Oreo"
        ;;
esac
```

> - case $1 in ：$1 表示我们要测试的变量是输入的第一个参数。in 是英语“在…之中”的意思。
>
> - "Cupcake") ：测试其中一个 case，也就是 $1 是否等于 "Cupcake"。也可以用**星号来做通配符来匹配多个字符**，例如 "M*") 可以匹配所有以 M 开头的字符串。
>
> - ;; ：类似于主流编程语言中的 break;，表示结束 case 的读取，程序跳转到 esac 后面执行。
>
> - *) ：相当于 if 条件语句的 else，表示“否则”，就是“假如不等于上面任何一种情况”。
> - esac ：是 case 的反写，表示 case 语句的结束。

{{< image classes="fancybox center fig-100" src="/linux/linux3_4.png" thumbnail="/linux/linux3_4.png" title="">}}

```shell
#!/bin/bash
case $1 in
    "dog" | "cat" | "pig")
        echo "It is a mammal"
        ;;
    "pigeon" | "swallow")
        echo "It is a bird"
        ;;
    *)
        echo "I do not know what it is"
        ;;
esac
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_5.png" thumbnail="/linux/linux3_5.png" title="">}}

上面判断变量的分类是dog，cat，pig是一类，pigeon和swallow是另外一类，只要输入的参数是其中之一即可满足这一类别的判断。

出上面之外，还支持正则表达式，判断一个字符

```shell
#!/bin/bash
# 输入一个字符，用于后续判断是字母，数字还是其他类型
read -p "press some key ,then press return :" KEY
case $KEY in
    [a-z]|[A-Z])
        echo "It's a letter."
        ;;
    [0-9]) 
        echo "It's a digit."
        ;;
    *)
        echo "It's function keys、Spacebar or other ksys."
        ;;
esac
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_6.png" thumbnail="/linux/linux3_6.png" title="">}}

### 4.5.3条件变量替换

表4.5.5条件变量替换

|         引用格式          | 返回值及用法                                                 |
| :-----------------------: | ------------------------------------------------------------ |
|           $var            | 返回变量值，例如var="Barry"，则$var即为Barry，这里的""是界定符号，不是字符串中的字符 |
|          ${var}           | 返回变量值，推荐写法。                                       |
|          ${#var}          | 返回该变量字符串的长度，例如var="Barry"，则${#var}返回5      |
|    ${var:start_index}     | 默认返回从start_index开始到末尾的字符串。例如var="Barry"，则${var:0}返回Barry,${var:2-4}返回Bar，这里的2-4，需要从字符串的尾部开始计算 |
| ${var:start_index:length} | 默认返回从start_index开始的length个字符，可以为负数。例如var="0123456789",${var:2:5}返回23456，${var:5:-2}返回567，这里的-2代表剩余两个字符不要，${var:0-3:-1}返回567 |
|     ${var:-newstring}     | 返回值分两种情况。<br />1.如果var为空或者未定义，则返回newstring<br />2.如果var不为空，则返回变量值 |
|     ${var:=newstring}     | 返回值分两种情况。<br />1.如果var为空或者未定义，则返回newstring，并把newstring赋值给var<br />2.如果var不为空，则返回变量值 |
|     ${var:?newstring}     | 返回值分两种情况。<br />1.如果var为空或者未定义，则返回newstring写入标准错误流，本语句失败<br />2.如果var不为空，则返回变量值 |
|     ${var:+newstring}     | 返回值分两种情况。<br />1.如果var不为空，则返回newstring<br />2.如果var为空，则返回空值 |
|       ${var#string}       | 返回从左边删除string后的字符串，尽可能短的去匹配。例如var="http://127.0.0.1/index.php",则${var#*/},返回"/127.0.0.1/index.php" |
|      ${var##string}       | 返回从左边删除string后的字符串，尽可能长的去匹配。例如var="http://127.0.0.1/index.php",则${var##*/},返回"index.php" |
|       ${var%string}       | 返回从右边删除string后的字符串，尽可能短的去匹配。例如var="http://127.0.0.1/index.php",则${var%/*},返回"http://127.0.0.1" |
|      ${var%%string}       | 返回从右边删除string后的字符串，尽可能长的去匹配。例如var="http://127.0.0.1/index.php",则${var%%/*},返回"http:" |
|  ${var/substring/string}  | 返回var中第一个substring被替换成newstring后的字符串。例如var="08880",则${var/0/Barry},返回Barry88880 |
| ${var//substring/string}  | 返回var中中所有substring被替换成newstring后的字符串。例如var="08880",则${var//0/Barry},返回Barry8888Barry |
|        $(command)         | 返回command命令指令后所输出的结果，例如$(date)返回就是date命令执行后的输出，相当于反引号下的date |
|      $((算术表达式))      | 返回双括号内的运算结果，例如$((500+5*4))返回520，**可以不需要两边加空格** |

这里举一个简单的例子

```shell
#!/bin/bash
var="Barry"
echo $var
echo ${var}
echo ${#var}
var1="0123456789abcdef"
echo ${var1:0}
echo ${var1:0-5}
echo ${var1:2:5}
echo ${var1:5:-2}
echo ${var1:0-3:-1}
echo "================"
var6="http://127.0.0.1/index.php"
echo ${var6#*/}
echo ${var6##*/}
echo ${var6%/*}
echo ${var6%%/*}
echo "================"
echo $(date)
echo $((500+5*4))
echo "================"
var7="08880"
echo ${var7/0/Barry}
echo ${var7//0/Barry}
echo "================"
newstring="new"
echo ${var2:-newstring}
echo $var2
echo ${var:-newstring}
echo $var
echo "================"
echo ${var3:=newstring}
echo $var3
echo ${var:=newstring}
echo $var
echo "================"
echo ${var4:+newstring}
echo $var4
echo ${var:+newstring}
echo $var
echo "================"
echo ${var:?newstring}
echo ${var5:?newstring}
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_7.png" thumbnail="/linux/linux3_7.png" title="">}}

## 4.6循环运算

Shell 中，主要的循环语句有三种：while 循环，until 循环和for 循环。除了上述的循环关键字，还有三个关键字。

- break：跳出整个循环
- exit：跳出脚本
- continue：跳出本次循环，接着执行下一次循环

### 4.6.1 while循环

{{< image classes="fancybox center fig-100" src="/linux/linux3_9.png" thumbnail="/linux/linux3_9.png" title="">}}

while循环的逻辑

```shell
while [ condition ]
do
    dosomething
done
```

或者是下面的写法

```shell
while [ condition ]; do
    dosomething
done
```

举例说明

```shell
#!/bin/bash

while [ -z $response ] || [ $response != 'yes' ]
do
    read -p 'Say yes : ' response
    if [[ $response == 0 ]]; then
        echo "shell exit"
        # 退出此shell脚本
        exit 0
    fi
done

n=0
# 创造一个死循环
# 这样的写法也可以while :
while [ 1 ]
do
  sleep 1
  # 算法加法，不加双括号被认为是字符串
  ((n++))
  echo loop $n time
  if [[ $n == 5 ]]; then
    echo "this time need break"
    # 跳出循环
    break
  elif [[ $n == 3 ]]; then
    echo "need to continue"
    # 直接跳到变量指向的下一个循环列表
    continue
  else
    echo "just go"
  fi
done
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_11.png" thumbnail="/linux/linux3_11.png" title="">}}

### 4.6.2 until循环

{{< image classes="fancybox center fig-100" src="/linux/linux3_8.png" thumbnail="/linux/linux3_8.png" title="">}}

它也可以实现循环，只不过逻辑和 while 循环正好相反。

```shell
#!/bin/bash
# 推荐的一种比较相等的方式[[ "$response"x == "yes"x ]]
until [[ "$response"x == "yes"x ]]
do
    read -p 'Say yes : ' response
    if [[ $response = 0 ]]; then
        echo "shell exit"
        # 退出此shell脚本
        exit 0
    fi
done

n=0
# 创造一个死循环until [ ]
until [ ]
do
  sleep 1
  # 算法加法，不加双括号被认为是字符串
  ((n++))
  echo loop $n time
  if [[ $n == 5 ]]; then
    echo "this time need break"
    # 跳出循环
    break
  elif [[ $n == 3 ]]; then
    echo "need to continue"
    # 直接跳到变量指向的下一个循环列表
    continue
  else
    echo "just go"
  fi
done
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_12.png" thumbnail="/linux/linux3_12.png" title="">}}

### 4.6.3 for循环

{{< image classes="fancybox center fig-100" src="/linux/linux3_10.png" thumbnail="/linux/linux3_10.png" title="">}}

for 循环可以遍历一个“取值列表”，基本的逻辑如下

```shell
for var in 'var1' 'var2' 'var3' ... 'varn'
do
    dosomething
done
```

> Shell 中的 for 循环和习惯的 for 循环方式略有不同。shell中的for循环更倾向于，遍历列表的形式，当然也是兼容我们的习惯写法。
>
> ```shell
> #!/bin/bash
> # $1 为shell脚本传入的第一个参数
> j=$1
> for ((i=1; i<=j; i++))
> do
>     touch file$i && echo file $i is ok
> done
> ```

这里遍历的列表可以是自定义，也可以是变量

```shell
#!/bin/bash

# 列表可以是自定义
for animal in 'dog' 'cat' 'pig'
do
    echo "Animal being analyzed : $animal"
done
echo "==================="
# 列表可以是变量
listfile=`ls`
for file in $listfile
do
    echo "File found : $file"
done
echo "==================="

# 列表可以是变量，可以直接省略直接通过反引号的方式写入
for file1 in `ls`
do
    echo "File found : $file1"
done
echo "==================="

# 列表可以是文件的集合
for i in /boot/*
do
    echo "/boot/ dir list : $i"
done
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_13.png" thumbnail="/linux/linux3_13.png" title="">}}

通常我们会用到一些数字，通过seq来获取循环列表

```shell
#!/bin/bash
# 如果是从1开始，可以简写成for i in `seq n`
for i in `seq 1 5`
do
    echo $i
done
echo "==================="

for i in `seq 8`
do
    echo $i
done
echo "==================="

# 从1开始，间隔为2
for i in `seq 1 2 10`
do
    echo $i
done
echo "==================="

# 从5开始，间隔为-1，倒着数数
for i in $(seq 5 -1 1)
do
    echo  "$i";sleep 1
done
echo "==================="

# 从0开始到10的集合
for i in {0..10}; do
    echo $i
done
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_14.png" thumbnail="/linux/linux3_14.png" title="">}}

或者通过外部传入的参数当做循环的列表

```shell
#!/bin/bash
echo "there are $# arguments in this scripts"
# 通过变量N用来计数
N=1    
for i in $@
do
    echo "\$$N is $i"
    ((N++))
done
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_15.png" thumbnail="/linux/linux3_15.png" title="">}}

## 4.7函数运算

我们可以把函数比作一个香肠制造机，在输入那一头你把猪装进去，输出那一头就出来香肠了。

{{< image classes="fancybox center fig-100" src="/linux/linux3_16.png" thumbnail="/linux/linux3_16.png" title="">}}

函数的定义

```shell
funname () {
    action
    [return int]
}
```

或者是

```shell
function funname {
    action
    [return int]
}
```



> 注意事项：
>
> 1. 如果选择第一种，函数名后面跟着的圆括号里不加任何参数。
> 2. 可以带function fun() 定义，也可以直接fun() 定义,不带任何参数。
> 3. 参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n**(0-255)**
> 4. 函数的完整定义必须置于函数的调用之前。

### 4.7.1传递参数

```shell
#!/bin/bash
print_func_1 () {
    echo "Hello, I am a function"
}

print_func_2 () {
    echo Hello $1
}

print_func_1
# print_func_1是无参，所以加参数也是没有意义的
print_func_1 Matthew
print_func_2 Matthew
print_func_2 Mark
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_17.png" thumbnail="/linux/linux3_17.png" title="">}}

> 它传递参数的方式其实很像给 Shell 脚本传递命令行参数。我们把参数直接置于函数名字后面，然后就像我们之前 Shell 脚本的参数那样：`$1`，`$2`，`$3`等等。

### 4.7.2返回值

```shell
#!/bin/bash
# 显示返回
print_func_3 () {
    echo Hello $1
    return 1
}

# 显示返回
print_func_3_1 () {
    echo Hello $1
    return 255
}

# 显示返回，这个超过255之后，就会对返回的值取余(N-1)%255
print_func_3_2 () {
    echo Hello $1
    return 258
}


#隐式返回
print_func_4 () {
    cat $1 | wc -l
}

print_func_3 $1
echo res = $?
echo "================"

print_func_3_1 $1
echo res = $?
echo "================"

print_func_3_2 $1
echo res = $?
echo "================"

line_num=$(print_func_4 $1)
echo The file $1 has $line_num lines
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_18.png" thumbnail="/linux/linux3_18.png" title="">}}

### 4.7.3全局变量和局部变量

要定义一个局部变量，我们只要在第一次给这个变量赋值时在变量名前加上关键字 local 即可。定义在函数体里面，可不加local关键字。

在函数中，如果全局变量和局部变量相等，会优先使用局部变量的赋值，也就是说局部变量会被改变。

```shell
#!/bin/bash

local_global_func () {
    local var1='local'
    echo Inside function: var1 is $var1 : var2 is $var2
    # 这里的 var1 是函数中定义的局部变量
    # 这里的 var2 是函数外定义的全局变量
    var1='local_var1'   
    var2='local_var2'
    echo function call: var1 is $var1 : var2 is $var2
}

var1='global_var1'
var2='global_var2'

echo Before function call: var1 is $var1 : var2 is $var2

local_global_func

echo After function call: var1 is $var1 : var2 is $var2
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_19.png" thumbnail="/linux/linux3_19.png" title="">}}

### 4.7.4 函数重载

把函数的名字取成与我们通常在**命令行用的命令相同**的名字。函数重载的作用会起到一种包装的作用，比如命令前的前处理和命令后的后处理等等

```shell
#!/bin/bash
ls () {
    echo $1
    command ls -lh
}

ls $1
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_27.png" thumbnail="/linux/linux3_27.png" title="">}}

## 4.8Shell进阶

通过一个简单的统计来对shell进行一个回顾和总结

首先把需要的素材，放到和脚本同目录的地方，脚本地址[如下所示](https://github.com/frogoscar/english-dictionary)。

下载完成后，开始做一个英语字典做统计，对各个字母出现进行排序。

```shell
#!/bin/bash

# Verification of parameter
# 确认参数
if [ -z $1 ]
then
    echo "Please enter the file of dictionary !"
    exit 0
fi

# Verification of file existence
# 确认文件存在
if [ ! -e $1 ]
then
    echo "Please make sure that the file of dictionary exists !"
    exit 0
fi

# Definition of function
# 函数定义
count_charactor () {
    for char in {a..z}
    do
        # 这里的加o代表一个单词可能出现多个同一字母，一个字符匹配到一个算一个，匹配到多个算多个
        # tr含义是字符串替换，前面的a-z变为A-Z，代表的是小写转换为大写
        echo "$char - `grep -io "$char" $1 | wc -l`" | tr /a-z/ /A-Z/ >> tmp.txt
    done
    # sort是排序，r为从大到小，n为数字排序，k指定根据哪几列进行排序，这里为第2列，t为指定分割符，这里是-
    sort -rn -k 2 -t - tmp.txt
    rm tmp.txt
}

# Use of function
# 函数使用
count_charactor $1
```

{{< image classes="fancybox center fig-100" src="/linux/linux3_20.png" thumbnail="/linux/linux3_20.png" title="">}}

# 总结

总的来说，本文主要是介绍了关于文本编辑器Nano和Vim，并且Vim有三个模式，各个模式可以互相切换。通过Vim可以去自己写shell的脚本，通过基本的概念，最后通过一个例子对条件，循环，函数等进行一个回顾和总结。



# 猜你喜欢

[linux1 基础学习](https://yangyang48.github.io/2022/03/linux1-%E5%9F%BA%E7%A1%80%E5%AD%A6%E4%B9%A0/)

[linux2 权限和管理](https://yangyang48.github.io/2022/04/linux2-%E6%9D%83%E9%99%90%E5%92%8C%E7%AE%A1%E7%90%86/)

[linux4 NDK交叉编译](https://yangyang48.github.io/2022/03/linux4-ndk%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91/)



# 参考

[[1] 一生只画眉, Shell if 条件判断, 2018.](https://blog.csdn.net/zhan570556752/article/details/80399154)

[[2] aaron_agu, shell if \[[ \]]和[ ]区别 || &&, 2016.](https://www.cnblogs.com/aaron-agu/p/5700650.html)

[[3] dreamtdp, Linux Shell编程case语句, 2012.](https://blog.csdn.net/dreamtdp/article/details/8048720)

[[4] 也许明天, shell变量的替换, 2015.](https://www.cnblogs.com/alex-13/p/4295012.html)

[[5] 运维自动化&云计算, shell脚本报错：": =: unary operator expected", 2017.](https://blog.csdn.net/h106140873/article/details/78383467)

[[6] 斯言甚善, shell中的for循环用法详解, 2017.](https://blog.csdn.net/qq_18312025/article/details/78278989)

[[7] 小白的进阶, shell脚本去重的几种方法, 2019.](https://blog.csdn.net/laobai1015/article/details/91455406)

[[8] 嘉年华, Linux shell tr 命令详解, 2019.](https://www.cnblogs.com/faberbeta/p/linux-shell003.html)

[[9] 悠悠i, Linux环境变量配置全攻略, 2019.](https://www.cnblogs.com/youyoui/p/10680329.html)

[[10] ee230, shell 二维数组, 2015.](https://blog.csdn.net/ee230/article/details/48316317?_t_t_t=0.856576693346367)
