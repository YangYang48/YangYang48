---
title: "C语言中的`%*s` 和 `%.*s`"
date: 2022-08-28
thumbnailImagePosition: left
thumbnailImage: c++/base/base1_thumb.jpg
coverImage: c++/base/base1_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- scanf
- printf
- 2022
- August 
tags:
- c++
- c++基础
- Android
- 源码
showSocial: false
---

Android源码中有一些经常会遇到的c++基础的内容，笔者对C语言中的`%*s` 和 `%.*s`进行简单熟悉。

<!--more-->
# C语言中的`%*s` 和 `%.*s`

需要区分在`scanf`函数还是在`printf`函数中使用

# scanf

在`scanf`中使用,则添加了`*`的部分会被忽略，不会被参数获取

```c++
#include <iostream>

int main() {
    int a = 0;
    char b[10] = {0};
    scanf("%d%*s",&a,b);
    std::cout << a << "\t" << b << std::endl;
    scanf("%d%s",&a,b);
    std::cout << a << "\t" << b << std::endl;
}
```

结果输出

{{< image classes="fancybox center fig-100" src="/c++/base/base1_1.png" thumbnail="/c++/base/base1_1.png" title="">}}



# printf

`printf`函数就会有两种写法，一种是`%*s`，另外一种是`%.*s`

`%*s`

表示用后面的形参替代`*`的位置，实现**动态格式输出**。如果输出的字符串长度小于形参的大小，那么全面用空格补全，即输出至少为形参的大小

`%.*s`

`*`用来指定宽度，对应一个整数。`.`（点）是指定必须输出这个宽度。如果所输出的字符串长度大于形参大小，则字符串输出会被截断到形参大小长度，如果小于，则输出实际长度，即输出至多为形参的大小



举例说明

```c++
#include <iostream>

int main() {
    char s[20] = {0};
    scanf("%s",s);
    printf("%*s\n", 10, s);
    printf("%10s\n", s);
    printf("%.*s\n", 10, s);
}
```

1）输入的字符串小于形参大小

{{< image classes="fancybox center fig-100" src="/c++/base/base1_2.png" thumbnail="/c++/base/base1_2.png" title="">}}

2）输入的字符串大于形参大小

{{< image classes="fancybox center fig-100" src="/c++/base/base1_3.png" thumbnail="/c++/base/base1_3.png" title="">}}



> 笔者用的IDEA是`Clion`，关于`Clion`的获取方式，可以点击[这里](https://exception.site/essay/idea-reset-eval)