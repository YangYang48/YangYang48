---
title: "关于可变函数va_list原理及用法"
date: 2023-07-02
thumbnailImagePosition: left
thumbnailImage: c++/base/va_list/vl_thumb.jpg
coverImage: c++/base/va_list/vl_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- c
- 2023
- July
tags:
- 基础
- va_list
- printf
- vprintf
- vfprintf
- vsprintf
showSocial: false
---

C语言允许定义参数数量可变的函数，这种函数需要固定数量的强制参数，后面是数量可变的可选参数。


<!--more-->
变参问题是指参数的个数不定，可以是传入一个参数也可以是多个;可变参数中的每个参数的类型可以不同,也可以相同;可变参数的每个参数并没有实际的名称与之相对应，用起来是很灵活。

> VA函数（variable argument function），参数个数可变函数，又称可变参数函数。
>
> C/C++编程中，**系统提供给编程人员的va函数很少**。*printf()/*scanf()系列函数，用于输入输出时格式化字符串；exec*()系列函数，用于在程序中执行外部文件(main(int argc,char*argv[]算不算呢，与其说main()也是一个可变参数函数，倒不如说它是exec*()经过封装后的具备特殊功能和意义的函数，至少在原理这一级上有很多相似之处)。由于参数个数的不确定，使va函数具有很大的灵活性，易用性。

# 0可变函数简介

说起可变函数，最先想到的肯定是printf和scanf函数。

对于printf函数

```c
extern int printf(const char *format,...);
```

> printf FORMAT [ARGUMENT]...
>
> printf OPTION
>
> FORMAT控制输出，就像C printf一样。解释序列是:
>
> |     名称     |                             说明                             |
> | :----------: | :----------------------------------------------------------: |
> |     `\"`     |                            双引号                            |
> |     `\\`     |                            反斜杠                            |
> |     `\a`     |                          警报(BEL)                           |
> |     `\b`     |                             退格                             |
> |     `\c`     |                     不会产生进一步的输出                     |
> |     `\e`     |                             逃脱                             |
> |     `\f`     |                           格式馈送                           |
> |     `\n`     |                           new line                           |
> |     `\r`     |                            回车符                            |
> |     `\t`     |                        horizontal TAB                        |
> |     `\v`     |                          垂直制表符                          |
> |    `\NNN`    |         字节与八进制值`NNN`(1到3位数字`\077`代表`>`)         |
> |    `\xHH`    |        字节，十六进制值`HH`(1到2位数字`\x3e`代表`>`)         |
> |   `\uHHHH`   |     Unicode (ISO/IEC 10646)字符，十六进制值HHHH(4位数字)     |
> | `\UHHHHHHHH` |            十六进制值为HHHHHHHH的Unicode字符(8位)            |
> |     `%%`     |                            单个%                             |
> |     `%b`     | ARGUMENT作为字符串，解释'\'转义，但八进制转义的形式为\0或\0NNN |
> |     `%q`     | ARGUMENT以一种可以重用为shell输入的格式打印，用POSIX $ "语法转义不可打印的字符。 |
>
> 以及所有以`diouxXfeEgGcs`之一结尾的C格式规范，首先将ARGUMENTs转换为适当的类型。处理可变宽度。



- 参数format表示如何来格式字符串的指令，…
- 表示可选参数，调用时传递给"..."的参数可有可无，根据实际情况而定。

系统提供了**vprintf**系列格式化字符串的函数，用于编程人员封装自己的I/O函数。

# 1可变函数使用

## 1.1使用对应存在的可变函数

```c
 // 从标准输入/输出格式化字符串
int vprintf / vscanf(const char * format, va_list ap);
```

```c
// 从文件流
int vfprintf / vfsacanf(FILE * stream, const char * format, va_list ap);
```

```c
// 从字符串
int vsprintf / vsscanf(char * s, const char * format, va_list ap);
```

比如上述几个可变函数

```c
#include <stdio.h>
#include <stdarg.h>
char buffer[80] = {0};
int WriteLog(const char * format, ...)
{
    va_list arg_ptr;
    va_start(arg_ptr, format);
    int nWrittenBytes = vsprintf(buffer, format, arg_ptr);
    va_end(arg_ptr);
    return nWrittenBytes;
}


int main()
{
    int nYear = 2023;
    int nMonth = 7;
    int nDay = 1;
    int nHour = 21;
    int nMinute = 31;
    int nSec = 0;
    char szUserName[] = "CoolFly";
    int nUserID = 9527;
    // 调用时，与使用printf()没有区别。
    WriteLog("%04d-%02d-%02d %02d:%02d:%02d %s/%04d",
             nYear, nMonth, nDay, nHour, nMinute, nSec, szUserName, nUserID);
    printf("%s\n", buffer);
    return 0;
}
```

对应输出

```c
2023-07-01 21:31:00 CoolFly/9527
```

## 1.2构造可变函数

通过va_arg方式

```c
#include <stdio.h>
#include <stdarg.h>
int SqSum(int n1, ...)
{
    va_list arg_ptr;
    int nSqSum = 0, n = n1;
    va_start(arg_ptr, n1);
    while (n > 0) {
        nSqSum += (n * n);
        n = va_arg(arg_ptr, int);
    }
    va_end(arg_ptr);
    return nSqSum;
}

int main()
{
    int nSqSum = SqSum(9, 9, 5, 2, 7, -1);
    printf("%d", nSqSum);
    return 0;
}
```

输出

```c
240
```



# 2可变函数原理

## 2.1va_list arg_ptr

定义一个指向个数可变的参数列表指针

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_1.png" thumbnail="/c++/base/va_list/vl_1.png" title="">}}

## 2.2va_start(arg_ptr, argN)

1. 使参数列表指针**arg_ptr**指向函数参数列表中的第一个**可选参数**
2. argN是位于**第一个可选参数之前的固定参数**（或者说，最后一个固定参数；…之前的一个参数）

> 如果有一va函数的声明是
>
> ```c
> void va_test(char a, char b, char c, ...)
> ```
>
> 它的固定参数依次是a,b,c，最后一个固定参数argN为**c**，因此就是va_start(arg_ptr, c)，函数参数列表中参数在内存中的顺序与函数声明时的顺序是一致的。

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_2.png" thumbnail="/c++/base/va_list/vl_2.png" title="">}}

## 2.3va_arg(arg_ptr, type)

返回参数列表中指针arg_ptr所指的参数，返回类型为type，并使指针arg_ptr**指向参数列表中下一个参数**。(优点类似，出栈出队列)

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_3.png" thumbnail="/c++/base/va_list/vl_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_4.png" thumbnail="/c++/base/va_list/vl_4.png" title="">}}

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_5.png" thumbnail="/c++/base/va_list/vl_5.png" title="">}}

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_6.png" thumbnail="/c++/base/va_list/vl_6.png" title="">}}

{{< image classes="fancybox center fig-100" src="/c++/base/va_list/vl_7.png" thumbnail="/c++/base/va_list/vl_7.png" title="">}}

## 2.4va_copy(dest, src)

dest，src的类型都是va_list，va_copy()用于复制参数列表指针，将dest初始化为src。

va_copy()后，必须得有相应的va_end()与之匹配。参数指针可以在参数列表中随意地来回移动，但必须在va_start()和va_end()之内。

## 2.5va_end(arg_ptr)

清空参数列表，并置参数指针arg_ptr无效。说明：指针arg_ptr被置无效后，可以通过调用va_start()、va_copy()恢复arg_ptr。

每次调用va_start()后，必须得有相应的va_end()与之匹配。参数指针可以在参数列表中随意地来回移动，但必须在va_start()和va_end()之内。

## 2.6x86平台定义

```c
typedef char* va_list;
#define _ADDRESSOF(v) (&(v))
#define _INTSIZEOF(n)          ((sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1))
 
#define __crt_va_start_a(ap, v) ((void)(ap = (va_list)_ADDRESSOF(v) + _INTSIZEOF(v)))
#define __crt_va_arg(ap, t)     (*(t*)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)))
#define __crt_va_end(ap)        ((void)(ap = (va_list)0))
 
#define va_start __crt_va_start
#define va_arg   __crt_va_arg
#define va_end   __crt_va_end
#define va_copy(destination, source) ((destination) = (source))
```



# 3总结

从可变参数的实现可以看出，指针的合理运用，把C语言简洁、灵活的特性表现得淋漓尽致，叫人不得不佩服C的强大和高效。不可否认的是，给编程人员太多自由空间必然使程序的安全性降低。可变参数中，为了得到所有传递给函数的参数，需要用va_arg依次遍历。其中存在两个隐患：

1. **如何确定参数的类型**

   va_arg在类型检查方面与其说非常灵活，不如说是很不负责，因为是强制类型转换，va_arg都把当前指针所指向的内容强制转换到指定类型

2. **结束标志**

   如果没有结束标志的判断，va将按默认类型依次返回内存中的内容，直到访问到非法内存而出错退出。

# 参考文献

[[1]  张珂荣. va_list原理及用法, 2019.](https://blog.csdn.net/ZKR_HN/article/details/99558135)

[[2] 迷夏牛,关于C中函数的可变参数va_list...,2009](https://blog.csdn.net/homer1984/article/details/3859036).

[[3] smilexiaowei,深入浅出VA函数的使用技巧,2005](https://blog.csdn.net/smilexiaowei/article/details/528735?spm=1001.2101.3001.6650.5&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-5-528735-blog-3859036.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-5-528735-blog-3859036.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=6).