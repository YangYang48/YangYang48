---
title: "C++ 变长数组"
date: 2022-08-28
thumbnailImagePosition: left
thumbnailImage: c++/base/base2_thumb.jpg
coverImage: c++/base/base2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 变长数组
- 2022
- August 
tags:
- c++
- c++基础
- Android
- 源码
showSocial: false
---

Android源码中有一些经常会遇到的c++基础的内容，笔者对C++ 变长数组进行简单熟悉。

<!--more-->
# C++ 变长数组

在Android系统源码中，会经常出现变长数组，类似如下的结构

```c++
//bionic/libc/system_properties/include/prop_area.h
struct prop_bt {
  uint32_t namelen;
  atomic_uint_least32_t prop;
  atomic_uint_least32_t left;
  atomic_uint_least32_t right;
  atomic_uint_least32_t children;
  char name[0];

  prop_bt(const char* name, const uint32_t name_length) {
    this->namelen = name_length;
    memcpy(this->name, name, name_length);
    this->name[name_length] = '\0';
  }

 private:
  BIONIC_DISALLOW_COPY_AND_ASSIGN(prop_bt);
};
```

可以看到除去`namelen`和几个原子操作的变量，最后是一个`char name[0]`的写法，这种写法就是变长数组的用法。

> **变长数组作用**
>
> 变长数组又叫柔性数组。让数组长度是可变的，根据需要进行分配。方便操作，节省空间。不需要一开始定义，需要的时候再分配大小。



```c++
#include <iostream>
#include<string.h>

typedef struct Data
{
    int namelen;
    char name[0];
}Data_t;

int main()
{
    char str[] = "IronMan";
    int nLen = strlen(str);
    printf("no malloc struct size (%u), nLen(%u) \n", sizeof(Data_t), nLen);

    Data_t *myData = (Data_t*)malloc(sizeof(Data_t) + nLen);
    myData->namelen = nLen;
    memmove(myData->name, str, nLen + 1);
    printf(" Data struct len(%d), name(%s) \n", myData->namelen, myData->name );

    free(myData);
    return 0;
}
```

结果输出

{{< image classes="fancybox center fig-100" src="/c++/base/base2_1.png" thumbnail="/c++/base/base2_1.png" title="">}}

结论：可以看到结构体的大小仅只有`namelen`的大小。分配完结构体的堆区之后，堆区的前4个字节是`namelen`，后面都是变长数组的内容。



> **进一步探讨**
>
> 上述的`name[0]`也可以写成`name[]`，**同样不占用空间**，且地址紧跟在结构后面。但是如果写成`char *name`，那么这样就会作为指针，占用4个字节，地址不在结构之后，整个结构体就包含指针的大小。
> 另外采用`char *name`，**需要进行二次分配**，操作比较麻烦，很容易造成内存泄漏。而直接采用变长的数组，只需要分配一次，然后进行取值即可以。