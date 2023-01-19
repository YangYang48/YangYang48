---
title: "addr2line工具使用"
date: 2022-12-03
thumbnailImagePosition: left
thumbnailImage: AndroidTools/addr2line/addr2line_thumb.jpg
coverImage: AndroidTools/addr2line/addr2line_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- addr2line
- 2022
- December
tags:
- Android
- 调试工具
showSocial: false
---

Android中有一些常见的调试技巧，比如addr2line，用于将函数地址解析为函数名。

<!--more-->
# 0简介

`addr2line`这个工具是`Linux`源码自带的工具，位置`/user/bin/addr2line`

用`addr2line`可以将函数地址解析为函数名，在抓取调堆栈时`Java`层的堆栈本身就是显示函数名与行数，这个不需要转换，但对于`native`和`kernel`层的则是函数地址，需要借助`addr2line`来进行转换。

除了本来**动态库**之外，应用到**进程**也是可行的

# 1使用

|          参数           |                             解释                             |
| :---------------------: | :----------------------------------------------------------: |
|    `-a --addresses`     |    在函数名、文件和行号信息之前，显示地址，以十六进制形式    |
| `-b --target=<bfdname>` |                 指定目标文件的格式为bfdname                  |
| `-e --exe=<executable>` |                指定需要转换地址的可执行文件名                |
|     `-i --inlines`      | 如果需要转换的地址是一个内联函数，则输出的信息包括其最近范围内的一个非内联函数的信息 |
|  `-j --section=<name>`  |        给出的地址代表指定section的偏移，而非绝对地址         |
|   `-p --pretty-print`   |    使得该函数的输出信息更加人性化：每一个地址的信息占一行    |
|    `-s --basenames`     | 仅仅显示每个文件名的基址（即不显示文件的具体路径，只显示文件名） |
|    `-f --functions`     |        在显示文件名、行号输出信息的同时显示函数名信息        |
| `-C --demangle[=style]` |             将低级别的符号名解码为用户级别的名字             |
|       `-h --help`       |                         输出帮助信息                         |
|     `-v --version`      |                          输出版本号                          |

> 使用过程中如果出现的是问号
>
> 说明没有使用带符号的目录下对应的进程或者动态库，
>
> - 进程目录为`/out/target/product/xxx/xxx/symbols/system/bin/xxx`
> - 动态库目录是`/out/target/product/xxx/xxx/symbols/system/lib/xxx`



**如何使用**

## 1.1首先需要错误堆栈信息

找到一份有堆栈信息的错误打印日志

```text
//省略如下
--------- beginning of crash 
Fatal signal 11 (SIGSEGV), code 1, fault addr 0x4 in tid 1234 (demo.crash), pid 1234(demo) 
1234 1234 F DEBUG : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** 
1234 1234 F DEBUG : Build fingerprint: '' 
1234 1234 F DEBUG : Revision: '0' 
1234 1234 F DEBUG : ABI: 'arm' 
1234 1234 F DEBUG : pid: 1234, tid: 1234, name: demo >>> /system/bin/demo <<< 
1234 1234 F DEBUG : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x4 
1234 1234 F DEBUG : Cause: null pointer dereference 
1234 1234 F DEBUG : 
1234 1234 F DEBUG : backtrace: 
1234 1234 F DEBUG : #00 pc 000a56c6 /system/bin/demo (divide+761) 
1234 1234 F DEBUG : #01 pc 000f2d9d /system/bin/demo (main+110) 
1234 1234 F DEBUG : #02 pc 000f0480 /system/lib/libc.so (__pthread_start(void*)+22) 
1234 1234 F DEBUG : #03 pc 0001af8d /system/lib/libc.so (__start_thread+32) 
--------- beginning of main
```

可以看到这里有`backtrace`的堆栈信息，`#00`就是最终的错误发生点

## 1.2验证函数名和行数

这里先验证`##01`，如果函数名和行数正确，再来看`crash`对应的日志

```shell
$ addr2line -C -f -e /out/target/product/xxx/xxx/symbols/system/bin/demo 000f2d9d
main
/external/demo/main.c:13
```

发现是正确的

再次验证`##00`

```shell
$ addr2line -C -f -e /out/target/product/xxx/xxx/symbols/system/bin/demo 000a56c6
divide
/external/demo/main.c:5
```

到这里真正找到原因，原来是`demo`中出现了除以0的操作，导致异常`crash`。

## 1.3demo

```c++
//这是一个demo，用于验证addr2line的
#include<stdio.h>
int divide(int x, int y)
{
    return x/y;
}

int main()
{
    printf("hello world\n");
    int x = 3;
    int y = 0;
    int div = divide(x, y); 
    printf("--->>>div(%d)", div);
    return 0;
}
```

