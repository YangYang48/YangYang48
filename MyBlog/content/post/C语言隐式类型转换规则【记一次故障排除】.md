---
title: "C语言隐式类型转换规则【记一次故障排除】"
date: 2023-07-01
thumbnailImagePosition: left
thumbnailImage: c++/base/Implicit_conversion/ic_thumb.jpg
coverImage: c++/base/Implicit_conversion/ic_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- c
- 2023
- July
tags:
- 基础
- 隐式转换
- 基本类型
showSocial: false
---

隐式类型转换造成的危害程度与数组和指针有的一拼。如果你混合使用不同类型，C使用一个规则集合来自动完成类型转换。这可能很方便，但也很危险。

<!--more-->
# 0 提问

```c
#include <stdio.h>
int main()
{
    char a= -1;
    signed char b=-1;
    unsigned char c=-1;
    printf("a=%d,b=%d,c=%d",a,b,c);
    return 0;
}
```

这个输出是什么，先说答案

```c
a=-1,b=-1,c=255
```

至于为什么先按下不表，先往下看下去。

# 1基本规则

> 基本的规则
>
> 1. 当**出现在表达式中时**（包括大小比较，位运算，输出等等），有符号和无符号的char类型和short类型都将自动被转换为int（**在需要的情况下**将自动被转换为unsigned int）；在K&R C下，float将被自动转换为double；因为这种自动转换都是将原来的类型转换为较大的类型，所以被称为提升
>
> 2. 在包含两种数据类型的任何运算中，两个值都被转换为两种数据类型中较高的级别
>
> 3. 类型级别从高到低：long double > double > float > unsigned long long > long long > unsigned long > long > unsigned int > int；当long和int具有相同大小时，unsigned int > long；short和char会被自动提升为int或unsigned int
>
>    {{< image classes="fancybox center fig-100" src="/c++/base/Implicit_conversion/ic_1.png" thumbnail="/c++/base/Implicit_conversion/ic_1.png" title="">}}
>
> 4. 在赋值语句中，计算的最后结果被转换为将要被赋值的那个变量的类型；该过程可能导致提升，也可能导致降级（demotion，将一个值转换成一个更低级的类型）
>
> 5. 当作为函数的参数进行传递时，char和short会被转换为int，float会被转换为double

另外，还有一张表格

|  符号属性  |  长度属性   |  基本型  | 所占位数<br />(32位) | 取值范围<br />（32位） | 所占位数<br />(64位) | 取值范围<br />（64位） |        输入字符        |
| :--------: | :---------: | :------: | :------------------: | :--------------------: | :------------------: | :--------------------: | :--------------------: |
|            |             |  `指针`  |         `32`         |                        |         `64`         |                        |                        |
|  `signed`  |             |  `char`  |         `8`          |      `-2^7~2^7-1`      |         `8`          |      `-2^7~2^7-1`      |          `%c`          |
| `unsigned` |             |  `char`  |         `8`          |       `0~2^8-1`        |         `8`          |       `0~2^8-1`        |          `%c`          |
| `[signed]` |   `short`   | `[int]`  |         `16`         |     `-2^15~2^15-1`     |         `16`         |     `-2^15~2^15-1`     |         `%hd`          |
| `unsigned` |   `short`   | `[int]`  |         `16`         |       `0~2^16-1`       |         `16`         |       `0~2^16-1`       |     `%hu,%ho,%hx`      |
| `[signed]` |             | `[int]`  |         `32`         |     `-2^31~2^31-1`     |         `32`         |     `-2^31~2^31-1`     |          `%d`          |
| `unsigned` |             | `[int]`  |         `32`         |       `0~2^32-1`       |         `32`         |       `0~2^32-1`       |       `%u,%o,%x`       |
| `[signed]` |   `long`    | `[int]`  |         `32`         |     `-2^31~2^31-1`     |         `64`         |     `-2^63~2^63-1`     |         `%ld`          |
| `unsigned` |   `long`    | `[int]`  |         `32`         |       `0~2^32-1`       |         `64`         |       `0~2^64-1`       |     `%lu,%lo,%lx`      |
| `[signed]` | `long long` | `[int]`  |         `64`         |     `-2^63~2^63-1`     |         `64`         |     `-2^63~2^63-1`     |        `%l64d`         |
| `unsigned` | `long long` | `[int]`  |         `64`         |       `0~2^64-1`       |         `64`         |       `0~2^64-1`       |  `%l64u,%l64o,%l64x`   |
|            |             | `float`  |         `32`         |   `+/-3.40282e+038`    |         `32`         |   `+/-3.40282e+038`    |       `%f,%e,%g`       |
|            |             | `double` |         `64`         |    `+/-1.79769+308`    |         `64`         |    `+/-1.79769+308`    | `%lf,%le,%lg,%f,%e,%g` |
|            |   `long`    | `double` |         `96`         |    `+/-1.79769+308`    |         `96`         |    `+/-1.79769+308`    |     `%Lf,%Le,%Lg`      |

> 这里可能会有人好奇，为什么float占用4字节，确比long long取值更大。
>
> C语言中的float类型一般是占用32位，其中8位用于指数（阶符和阶码）表示，剩余32位用于尾数（包括尾符和尾码）
>
> {{< image classes="fancybox center fig-100" src="/c++/base/Implicit_conversion/ic_3.png" thumbnail="/c++/base/Implicit_conversion/ic_3.png" title="">}}
>
> 因为阶码占用7位，所以范围是-128～127
>
> 尾码占用23位，所以有2^23=8388608种组合，**8388608有7位，但是不到9999999种组合，所以至少能保证6位有效数字，最多可表示7位。**
>
> 假设尾码全是1，那么这时为码的数值最大，接下来我们算一下最大值是多少：
>
> ```c
> #include <stdio.h>
> #include <math.h>
> int main()
> {
>     int count = 1;
>     double base = 2.0;
>     double Pow = -1.0;
>     double sum = 0.0;
>     while (count < 24)
>     {
>         sum += pow(base, Pow);
>         printf("%2d times sum is (%2e)\n", count, sum);
>         Pow--;
>         count++;
>     }
> }
> ```
>
>  {{< image classes="fancybox center fig-100" src="/c++/base/Implicit_conversion/ic_4.png" thumbnail="/c++/base/Implicit_conversion/ic_4.png" title="">}}
>
> 根据计算可知，23位全是1的时候，数值为0.999999999，我们学过极限的同学都知道0.999999就是等于1。
>
> 前面说过阶码的补码范围是-128～127。最大指数就是2^128=3.4E+38
>
> **所以float最大能表示数字为1*3.4E+38**
>
> 那么符号变一下，最小数字为1*-3.4E-38
>
> 综合一下float范围不就是`-3.4E-38～3.4E+38`

# 2解答

理清楚基本概念之后，回答开头的问题

```c
char a= -1;  //第一次隐式类型转换，-1是int型，要被隐式转换为有符号char型
```

```c
signed char b=-1;  // 跟上面一样，-1是int型，要被隐式转换为有符号char型
```

```c
unsigned char c=-1;   //-1是int型，要被转换成无符号的char型
```

因为在计算机中，数据都是**以补码的形式存储的**，所以-1在表示的时候需要转换成它的补码。

{{< image classes="fancybox center fig-100" src="/c++/base/Implicit_conversion/ic_5.png" thumbnail="/c++/base/Implicit_conversion/ic_5.png" title="">}}

所以0xff对于无符号char类型就为255。

## 2.1关于char和int隐式转换

```c
#include <stdio.h>
int main()
{
    char a = 128; 
    printf("0x%x,(%u)\n",a, a);
    return 0;
}
```

输出是

```c
0xffffff80,(4294967168)
```

解释

```c
char a = 128; //有符号的char类型数据范围为-128~+127，而128超出了它的表示范围，所以会发生溢出。0x80，结果为-128
```

```c
printf("0x%x,(%u)\n",a, a);//根据表达式，会转换为int型，所以原本0x80会高位补符号位1，为0xffffff80
```

128可以转换成127+1，结果会发现是-128。然后在高位补符号位1，就变成了0xffffff80，这是一个有符号的int类型。

由于输出需要转换为unsigned int型（这个就是上面基本规则中的**在需要的情况下**），然后在转换成unsigned int类型。对应0xffffff80转换为10进制为4294967168。

## 2.2int和unsigned int转换

```c
#include <stdio.h>
#include <string.h> 
int main()
{
    const char *str = "abcdef";
    int i = -1;

    if(strlen(str) > i){
        printf("Yes\n");
    }
    else{
        printf("No\n");
    }
    printf("-1的无符号类型：%u\n", -1); 

    return 0;
}
```

输出是

```c
No
-1的无符号类型：4294967295
```

解释

```c
if(strlen(str) > i)//这个strlen返回的size_t,实际上是unsigned int类型，那么比较的后面也转换为unsigned int
```

str的长度是6，必然大于-1。

这里的i是-1，对应的是0xffffffff，转化成unsigned int，**即有符号数变成了无符号数**。那么它就是最大值，即4,294,967,295。

从表达式看前者永远不可能比后者大，所以返回的是no。

> **关于隐式转换**
>
> char提升为int，高位应该填0还是填1？
>
> - 如果是unsigned char的类型转换成int类型，高位补0.
> - 如果是signed char的类型转换成int类型，**如果原来最高位是1则补1，如果是0则补0**。
>
> 如果**int和unsigned int一起运算，会将int转为unsigned int**，这种操作如果放在条件判断中，会有想不到的结果，所以要小心小心。

## 2.3经典隐式转换问题

```c
#include <stdio.h>
#define CHECK_DEAD_LOOP(c, r)               \
do                                          \
{                                           \
    int i = 0;                              \
    for (c = 0; c <= r; c++)                \
    {                                       \
        i++;                                \
        if(i > 5000)                        \
        {                                   \
            printf("dead Loop"#r"\n\n");    \
            break;                          \
        }                                   \
    }                                       \
}while(0)
int main()
{
    signed char c = 0;

    unsigned char uc = 255;
    signed char sc = 255;
    unsigned short us = 255;
    signed short ss = 255;
    unsigned int ui = 255;
    signed int si = 255;

    CHECK_DEAD_LOOP(c, uc);
    CHECK_DEAD_LOOP(c, sc);
    CHECK_DEAD_LOOP(c, us);
    CHECK_DEAD_LOOP(c, ss);
    CHECK_DEAD_LOOP(c, ui);
    CHECK_DEAD_LOOP(c, si);
    return 0;
}
```

输出是

```c
dead Loopuc

dead Loopus

dead Loopss

dead Loopsi
```

解释

```c
signed char c = 0;//这个c的默认取值范围为-128~127
```

```c
unsigned char uc = 255;//255为int型为0xff，这里uc为unsigned char型，将255转化为unsigned char
```

```c
signed char sc = 255;//255为int型为0xff，这里sc为unsigned char型，将255转化为char，对应为-1
```

```c
unsigned short us = 255;//255为int型为0xff，这里us为unsigned short型，将255转化为unsigned short
```

```c
signed short ss = 255;//255为int型为0xff，这里ss为short型，将255转化为short
```

```c
unsigned int ui = 255;//255为int型为0xff，这里ui为unsigned int型，将255转化为unsigned int
```

```c
signed int si = 255;//这里无需转换，为0xff
```

搞清楚这些变量之后，开始用替换这些变量

### 2.3.1CHECK_DEAD_LOOP(c, uc)

```c
CHECK_DEAD_LOOP(c, uc);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= uc; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop uc\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

对于

```c
c <= uc;//c和uc都被转为int,这里会将0x80的c转化成int对其补齐为0xffffff80
c++;//其实把c又带回char型，当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)
```

```c
c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)c(0x5)c(0x6)c(0x7)c(0x8)c(0x9)c(0xa)c(0xb)c(0xc)c(0xd)c(0xe)c(0xf)c(0x10)c(0x11)c(0x12)c(0x13)c(0x14)c(0x15)c(0x16)c(0x17)c(0x18)c(0x19)c(0x1a)c(0x1b)c(0x1c)c(0x1d)c(0x1e)c(0x1f)c(0x20)c(0x21)c(0x22)c(0x23)c(0x24)c(0x25)c(0x26)c(0x27)c(0x28)c(0x29)c(0x2a)c(0x2b)c(0x2c)c(0x2d)c(0x2e)c(0x2f)c(0x30)c(0x31)c(0x32)c(0x33)c(0x34)c(0x35)c(0x36)c(0x37)c(0x38)c(0x39)c(0x3a)c(0x3b)c(0x3c)c(0x3d)c(0x3e)c(0x3f)c(0x40)c(0x41)c(0x42)c(0x43)c(0x44)c(0x45)c(0x46)c(0x47)c(0x48)c(0x49)c(0x4a)c(0x4b)c(0x4c)c(0x4d)c(0x4e)c(0x4f)c(0x50)c(0x51)c(0x52)c(0x53)c(0x54)c(0x55)c(0x56)c(0x57)c(0x58)c(0x59)c(0x5a)c(0x5b)c(0x5c)c(0x5d)c(0x5e)c(0x5f)c(0x60)c(0x61)c(0x62)c(0x63)c(0x64)c(0x65)c(0x66)c(0x67)c(0x68)c(0x69)c(0x6a)c(0x6b)c(0x6c)c(0x6d)c(0x6e)c(0x6f)c(0x70)c(0x71)c(0x72)c(0x73)c(0x74)c(0x75)c(0x76)c(0x77)c(0x78)c(0x79)c(0x7a)c(0x7b)c(0x7c)c(0x7d)c(0x7e)c(0x7f)c(0xffffff80)c(0xffffff81)c(0xffffff82)c(0xffffff83)c(0xffffff84)c(0xffffff85)c(0xffffff86)c(0xffffff87)c(0xffffff88)c(0xffffff89)c(0xffffff8a)c(0xffffff8b)c(0xffffff8c)c(0xffffff8d)c(0xffffff8e)c(0xffffff8f)c(0xffffff90)c(0xffffff91)c(0xffffff92)c(0xffffff93)c(0xffffff94)c(0xffffff95)c(0xffffff96)c(0xffffff97)c(0xffffff98)c(0xffffff99)c(0xffffff9a)c(0xffffff9b)c(0xffffff9c)c(0xffffff9d)c(0xffffff9e)c(0xffffff9f)c(0xffffffa0)c(0xffffffa1)c(0xffffffa2)c(0xffffffa3)c(0xffffffa4)c(0xffffffa5)c(0xffffffa6)c(0xffffffa7)c(0xffffffa8)c(0xffffffa9)c(0xffffffaa)c(0xffffffab)c(0xffffffac)c(0xffffffad)c(0xffffffae)c(0xffffffaf)c(0xffffffb0)c(0xffffffb1)c(0xffffffb2)c(0xffffffb3)c(0xffffffb4)c(0xffffffb5)c(0xffffffb6)c(0xffffffb7)c(0xffffffb8)c(0xffffffb9)c(0xffffffba)c(0xffffffbb)c(0xffffffbc)c(0xffffffbd)c(0xffffffbe)c(0xffffffbf)c(0xffffffc0)c(0xffffffc1)c(0xffffffc2)c(0xffffffc3)c(0xffffffc4)c(0xffffffc5)c(0xffffffc6)c(0xffffffc7)c(0xffffffc8)c(0xffffffc9)c(0xffffffca)c(0xffffffcb)c(0xffffffcc)c(0xffffffcd)c(0xffffffce)c(0xffffffcf)c(0xffffffd0)c(0xffffffd1)c(0xffffffd2)c(0xffffffd3)c(0xffffffd4)c(0xffffffd5)c(0xffffffd6)c(0xffffffd7)c(0xffffffd8)c(0xffffffd9)c(0xffffffda)c(0xffffffdb)c(0xffffffdc)c(0xffffffdd)c(0xffffffde)c(0xffffffdf)c(0xffffffe0)c(0xffffffe1)c(0xffffffe2)c(0xffffffe3)c(0xffffffe4)c(0xffffffe5)c(0xffffffe6)c(0xffffffe7)c(0xffffffe8)c(0xffffffe9)c(0xffffffea)c(0xffffffeb)c(0xffffffec)c(0xffffffed)c(0xffffffee)c(0xffffffef)c(0xfffffff0)c(0xfffffff1)c(0xfffffff2)c(0xfffffff3)c(0xfffffff4)c(0xfffffff5)c(0xfffffff6)c(0xfffffff7)c(0xfffffff8)c(0xfffffff9)c(0xfffffffa)c(0xfffffffb)c(0xfffffffc)c(0xfffffffd)c(0xfffffffe)c(0xffffffff)c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)...
```

尝试打印发现是一个死循环。

自增+1。当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)

比较。c和uc都被转为int,这里会将0x80转化成int对其补齐为0xffffff80

自增+1。这里0xffffff80加1之后为0xffffff81，在转化成char，为0x81，即为-127

重复上述过程，一直死循环。

### 2.3.2CHECK_DEAD_LOOP(c, sc)

```c
CHECK_DEAD_LOOP(c, sc);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= sc; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop sc\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

对于本身sc就是-1，c是0，即便是将char类型转换成int型，这个大小依然无法改变，所以，直接判断不成立就退出了



### 2.3.3CHECK_DEAD_LOOP(c, us)

```c
CHECK_DEAD_LOOP(c, sc);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= us; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop us\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

对于

```c
c <= us;//c和us都被转为int,这里会将0x80的c转化成int对其补齐为0xffffff80
c++;//其实把c又带回char型，当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)
```

```c
c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)c(0x5)c(0x6)c(0x7)c(0x8)c(0x9)c(0xa)c(0xb)c(0xc)c(0xd)c(0xe)c(0xf)c(0x10)c(0x11)c(0x12)c(0x13)c(0x14)c(0x15)c(0x16)c(0x17)c(0x18)c(0x19)c(0x1a)c(0x1b)c(0x1c)c(0x1d)c(0x1e)c(0x1f)c(0x20)c(0x21)c(0x22)c(0x23)c(0x24)c(0x25)c(0x26)c(0x27)c(0x28)c(0x29)c(0x2a)c(0x2b)c(0x2c)c(0x2d)c(0x2e)c(0x2f)c(0x30)c(0x31)c(0x32)c(0x33)c(0x34)c(0x35)c(0x36)c(0x37)c(0x38)c(0x39)c(0x3a)c(0x3b)c(0x3c)c(0x3d)c(0x3e)c(0x3f)c(0x40)c(0x41)c(0x42)c(0x43)c(0x44)c(0x45)c(0x46)c(0x47)c(0x48)c(0x49)c(0x4a)c(0x4b)c(0x4c)c(0x4d)c(0x4e)c(0x4f)c(0x50)c(0x51)c(0x52)c(0x53)c(0x54)c(0x55)c(0x56)c(0x57)c(0x58)c(0x59)c(0x5a)c(0x5b)c(0x5c)c(0x5d)c(0x5e)c(0x5f)c(0x60)c(0x61)c(0x62)c(0x63)c(0x64)c(0x65)c(0x66)c(0x67)c(0x68)c(0x69)c(0x6a)c(0x6b)c(0x6c)c(0x6d)c(0x6e)c(0x6f)c(0x70)c(0x71)c(0x72)c(0x73)c(0x74)c(0x75)c(0x76)c(0x77)c(0x78)c(0x79)c(0x7a)c(0x7b)c(0x7c)c(0x7d)c(0x7e)c(0x7f)c(0xffffff80)c(0xffffff81)c(0xffffff82)c(0xffffff83)c(0xffffff84)c(0xffffff85)c(0xffffff86)c(0xffffff87)c(0xffffff88)c(0xffffff89)c(0xffffff8a)c(0xffffff8b)c(0xffffff8c)c(0xffffff8d)c(0xffffff8e)c(0xffffff8f)c(0xffffff90)c(0xffffff91)c(0xffffff92)c(0xffffff93)c(0xffffff94)c(0xffffff95)c(0xffffff96)c(0xffffff97)c(0xffffff98)c(0xffffff99)c(0xffffff9a)c(0xffffff9b)c(0xffffff9c)c(0xffffff9d)c(0xffffff9e)c(0xffffff9f)c(0xffffffa0)c(0xffffffa1)c(0xffffffa2)c(0xffffffa3)c(0xffffffa4)c(0xffffffa5)c(0xffffffa6)c(0xffffffa7)c(0xffffffa8)c(0xffffffa9)c(0xffffffaa)c(0xffffffab)c(0xffffffac)c(0xffffffad)c(0xffffffae)c(0xffffffaf)c(0xffffffb0)c(0xffffffb1)c(0xffffffb2)c(0xffffffb3)c(0xffffffb4)c(0xffffffb5)c(0xffffffb6)c(0xffffffb7)c(0xffffffb8)c(0xffffffb9)c(0xffffffba)c(0xffffffbb)c(0xffffffbc)c(0xffffffbd)c(0xffffffbe)c(0xffffffbf)c(0xffffffc0)c(0xffffffc1)c(0xffffffc2)c(0xffffffc3)c(0xffffffc4)c(0xffffffc5)c(0xffffffc6)c(0xffffffc7)c(0xffffffc8)c(0xffffffc9)c(0xffffffca)c(0xffffffcb)c(0xffffffcc)c(0xffffffcd)c(0xffffffce)c(0xffffffcf)c(0xffffffd0)c(0xffffffd1)c(0xffffffd2)c(0xffffffd3)c(0xffffffd4)c(0xffffffd5)c(0xffffffd6)c(0xffffffd7)c(0xffffffd8)c(0xffffffd9)c(0xffffffda)c(0xffffffdb)c(0xffffffdc)c(0xffffffdd)c(0xffffffde)c(0xffffffdf)c(0xffffffe0)c(0xffffffe1)c(0xffffffe2)c(0xffffffe3)c(0xffffffe4)c(0xffffffe5)c(0xffffffe6)c(0xffffffe7)c(0xffffffe8)c(0xffffffe9)c(0xffffffea)c(0xffffffeb)c(0xffffffec)c(0xffffffed)c(0xffffffee)c(0xffffffef)c(0xfffffff0)c(0xfffffff1)c(0xfffffff2)c(0xfffffff3)c(0xfffffff4)c(0xfffffff5)c(0xfffffff6)c(0xfffffff7)c(0xfffffff8)c(0xfffffff9)c(0xfffffffa)c(0xfffffffb)c(0xfffffffc)c(0xfffffffd)c(0xfffffffe)c(0xffffffff)c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)...
```

尝试打印发现是一个死循环，原理同2.3.1类似，这里不再展开。

### 2.3.4CHECK_DEAD_LOOP(c, ss)

```c
CHECK_DEAD_LOOP(c, ss);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= ss; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop ss\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

对于

```c
c <= ss;//c和ss都被转为int,这里会将0x80的c转化成int对其补齐为0xffffff80
c++;//其实把c又带回char型，当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)
```

```c
c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)c(0x5)c(0x6)c(0x7)c(0x8)c(0x9)c(0xa)c(0xb)c(0xc)c(0xd)c(0xe)c(0xf)c(0x10)c(0x11)c(0x12)c(0x13)c(0x14)c(0x15)c(0x16)c(0x17)c(0x18)c(0x19)c(0x1a)c(0x1b)c(0x1c)c(0x1d)c(0x1e)c(0x1f)c(0x20)c(0x21)c(0x22)c(0x23)c(0x24)c(0x25)c(0x26)c(0x27)c(0x28)c(0x29)c(0x2a)c(0x2b)c(0x2c)c(0x2d)c(0x2e)c(0x2f)c(0x30)c(0x31)c(0x32)c(0x33)c(0x34)c(0x35)c(0x36)c(0x37)c(0x38)c(0x39)c(0x3a)c(0x3b)c(0x3c)c(0x3d)c(0x3e)c(0x3f)c(0x40)c(0x41)c(0x42)c(0x43)c(0x44)c(0x45)c(0x46)c(0x47)c(0x48)c(0x49)c(0x4a)c(0x4b)c(0x4c)c(0x4d)c(0x4e)c(0x4f)c(0x50)c(0x51)c(0x52)c(0x53)c(0x54)c(0x55)c(0x56)c(0x57)c(0x58)c(0x59)c(0x5a)c(0x5b)c(0x5c)c(0x5d)c(0x5e)c(0x5f)c(0x60)c(0x61)c(0x62)c(0x63)c(0x64)c(0x65)c(0x66)c(0x67)c(0x68)c(0x69)c(0x6a)c(0x6b)c(0x6c)c(0x6d)c(0x6e)c(0x6f)c(0x70)c(0x71)c(0x72)c(0x73)c(0x74)c(0x75)c(0x76)c(0x77)c(0x78)c(0x79)c(0x7a)c(0x7b)c(0x7c)c(0x7d)c(0x7e)c(0x7f)c(0xffffff80)c(0xffffff81)c(0xffffff82)c(0xffffff83)c(0xffffff84)c(0xffffff85)c(0xffffff86)c(0xffffff87)c(0xffffff88)c(0xffffff89)c(0xffffff8a)c(0xffffff8b)c(0xffffff8c)c(0xffffff8d)c(0xffffff8e)c(0xffffff8f)c(0xffffff90)c(0xffffff91)c(0xffffff92)c(0xffffff93)c(0xffffff94)c(0xffffff95)c(0xffffff96)c(0xffffff97)c(0xffffff98)c(0xffffff99)c(0xffffff9a)c(0xffffff9b)c(0xffffff9c)c(0xffffff9d)c(0xffffff9e)c(0xffffff9f)c(0xffffffa0)c(0xffffffa1)c(0xffffffa2)c(0xffffffa3)c(0xffffffa4)c(0xffffffa5)c(0xffffffa6)c(0xffffffa7)c(0xffffffa8)c(0xffffffa9)c(0xffffffaa)c(0xffffffab)c(0xffffffac)c(0xffffffad)c(0xffffffae)c(0xffffffaf)c(0xffffffb0)c(0xffffffb1)c(0xffffffb2)c(0xffffffb3)c(0xffffffb4)c(0xffffffb5)c(0xffffffb6)c(0xffffffb7)c(0xffffffb8)c(0xffffffb9)c(0xffffffba)c(0xffffffbb)c(0xffffffbc)c(0xffffffbd)c(0xffffffbe)c(0xffffffbf)c(0xffffffc0)c(0xffffffc1)c(0xffffffc2)c(0xffffffc3)c(0xffffffc4)c(0xffffffc5)c(0xffffffc6)c(0xffffffc7)c(0xffffffc8)c(0xffffffc9)c(0xffffffca)c(0xffffffcb)c(0xffffffcc)c(0xffffffcd)c(0xffffffce)c(0xffffffcf)c(0xffffffd0)c(0xffffffd1)c(0xffffffd2)c(0xffffffd3)c(0xffffffd4)c(0xffffffd5)c(0xffffffd6)c(0xffffffd7)c(0xffffffd8)c(0xffffffd9)c(0xffffffda)c(0xffffffdb)c(0xffffffdc)c(0xffffffdd)c(0xffffffde)c(0xffffffdf)c(0xffffffe0)c(0xffffffe1)c(0xffffffe2)c(0xffffffe3)c(0xffffffe4)c(0xffffffe5)c(0xffffffe6)c(0xffffffe7)c(0xffffffe8)c(0xffffffe9)c(0xffffffea)c(0xffffffeb)c(0xffffffec)c(0xffffffed)c(0xffffffee)c(0xffffffef)c(0xfffffff0)c(0xfffffff1)c(0xfffffff2)c(0xfffffff3)c(0xfffffff4)c(0xfffffff5)c(0xfffffff6)c(0xfffffff7)c(0xfffffff8)c(0xfffffff9)c(0xfffffffa)c(0xfffffffb)c(0xfffffffc)c(0xfffffffd)c(0xfffffffe)c(0xffffffff)c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)...
```

尝试打印发现是一个死循环，原理同2.3.1类似，这里不再展开。

### 2.3.5CHECK_DEAD_LOOP(c, ui)

```c
CHECK_DEAD_LOOP(c, ui);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= ui; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop ui\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

这里处理会转换为int型，但是ui是unsigned int型，所以，c直接转化成unsigned int型。

```c
c <= ui;//c被转为unsigned int,这里会将0x80的c转化成unsigned int对其补齐为0x00000080
c++;//其实把c又带回char型，当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)
```

尝试打印发现是**不是一个死循环**。

自增+1。当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)

比较。c被转化成unsigned int,这里会将0x80转化成unsigned int对其补齐为0x00000080，**虽然在char中已经是-1了，但是比较过程中依然可以判断，且0x00000080 > 0x0000007f(128>127等式不成立退出循环)**

```c
start c(0x0)r(0xff)
c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)c(0x5)c(0x6)c(0x7)c(0x8)c(0x9)c(0xa)c(0xb)c(0xc)c(0xd)c(0xe)c(0xf)c(0x10)c(0x11)c(0x12)c(0x13)c(0x14)c(0x15)c(0x16)c(0x17)c(0x18)c(0x19)c(0x1a)c(0x1b)c(0x1c)c(0x1d)c(0x1e)c(0x1f)c(0x20)c(0x21)c(0x22)c(0x23)c(0x24)c(0x25)c(0x26)c(0x27)c(0x28)c(0x29)c(0x2a)c(0x2b)c(0x2c)c(0x2d)c(0x2e)c(0x2f)c(0x30)c(0x31)c(0x32)c(0x33)c(0x34)c(0x35)c(0x36)c(0x37)c(0x38)c(0x39)c(0x3a)c(0x3b)c(0x3c)c(0x3d)c(0x3e)c(0x3f)c(0x40)c(0x41)c(0x42)c(0x43)c(0x44)c(0x45)c(0x46)c(0x47)c(0x48)c(0x49)c(0x4a)c(0x4b)c(0x4c)c(0x4d)c(0x4e)c(0x4f)c(0x50)c(0x51)c(0x52)c(0x53)c(0x54)c(0x55)c(0x56)c(0x57)c(0x58)c(0x59)c(0x5a)c(0x5b)c(0x5c)c(0x5d)c(0x5e)c(0x5f)c(0x60)c(0x61)c(0x62)c(0x63)c(0x64)c(0x65)c(0x66)c(0x67)c(0x68)c(0x69)c(0x6a)c(0x6b)c(0x6c)c(0x6d)c(0x6e)c(0x6f)c(0x70)c(0x71)c(0x72)c(0x73)c(0x74)c(0x75)c(0x76)c(0x77)c(0x78)c(0x79)c(0x7a)c(0x7b)c(0x7c)c(0x7d)c(0x7e)c(0x7f)
```



### 2.3.6CHECK_DEAD_LOOP(c, si)

```c
CHECK_DEAD_LOOP(c, si);
//对应的宏定义如下
do                                          
{                                           
    int i = 0;                              
    for (c = 0; c <= si; c++)//这个表达式根据基本规则，有符号和无符号的char类型和short类型都将自动被转换为int           
    {                                       
        i++;                                
        if(i > 5000)                        
        {                                   
            printf("dead Loop si\n\n");     
            break;                          
        }                                   
    }                                       
}while(0)
```

对于

```c
c <= si;//c被转为int,这里会将0x80的c转化成int对其补齐为0xffffff80
c++;//其实把c又带回char型，当c为0x7f之后，达到char类型最大值为127，c++之后就为-128(0x80)
```

```c
c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)c(0x5)c(0x6)c(0x7)c(0x8)c(0x9)c(0xa)c(0xb)c(0xc)c(0xd)c(0xe)c(0xf)c(0x10)c(0x11)c(0x12)c(0x13)c(0x14)c(0x15)c(0x16)c(0x17)c(0x18)c(0x19)c(0x1a)c(0x1b)c(0x1c)c(0x1d)c(0x1e)c(0x1f)c(0x20)c(0x21)c(0x22)c(0x23)c(0x24)c(0x25)c(0x26)c(0x27)c(0x28)c(0x29)c(0x2a)c(0x2b)c(0x2c)c(0x2d)c(0x2e)c(0x2f)c(0x30)c(0x31)c(0x32)c(0x33)c(0x34)c(0x35)c(0x36)c(0x37)c(0x38)c(0x39)c(0x3a)c(0x3b)c(0x3c)c(0x3d)c(0x3e)c(0x3f)c(0x40)c(0x41)c(0x42)c(0x43)c(0x44)c(0x45)c(0x46)c(0x47)c(0x48)c(0x49)c(0x4a)c(0x4b)c(0x4c)c(0x4d)c(0x4e)c(0x4f)c(0x50)c(0x51)c(0x52)c(0x53)c(0x54)c(0x55)c(0x56)c(0x57)c(0x58)c(0x59)c(0x5a)c(0x5b)c(0x5c)c(0x5d)c(0x5e)c(0x5f)c(0x60)c(0x61)c(0x62)c(0x63)c(0x64)c(0x65)c(0x66)c(0x67)c(0x68)c(0x69)c(0x6a)c(0x6b)c(0x6c)c(0x6d)c(0x6e)c(0x6f)c(0x70)c(0x71)c(0x72)c(0x73)c(0x74)c(0x75)c(0x76)c(0x77)c(0x78)c(0x79)c(0x7a)c(0x7b)c(0x7c)c(0x7d)c(0x7e)c(0x7f)c(0xffffff80)c(0xffffff81)c(0xffffff82)c(0xffffff83)c(0xffffff84)c(0xffffff85)c(0xffffff86)c(0xffffff87)c(0xffffff88)c(0xffffff89)c(0xffffff8a)c(0xffffff8b)c(0xffffff8c)c(0xffffff8d)c(0xffffff8e)c(0xffffff8f)c(0xffffff90)c(0xffffff91)c(0xffffff92)c(0xffffff93)c(0xffffff94)c(0xffffff95)c(0xffffff96)c(0xffffff97)c(0xffffff98)c(0xffffff99)c(0xffffff9a)c(0xffffff9b)c(0xffffff9c)c(0xffffff9d)c(0xffffff9e)c(0xffffff9f)c(0xffffffa0)c(0xffffffa1)c(0xffffffa2)c(0xffffffa3)c(0xffffffa4)c(0xffffffa5)c(0xffffffa6)c(0xffffffa7)c(0xffffffa8)c(0xffffffa9)c(0xffffffaa)c(0xffffffab)c(0xffffffac)c(0xffffffad)c(0xffffffae)c(0xffffffaf)c(0xffffffb0)c(0xffffffb1)c(0xffffffb2)c(0xffffffb3)c(0xffffffb4)c(0xffffffb5)c(0xffffffb6)c(0xffffffb7)c(0xffffffb8)c(0xffffffb9)c(0xffffffba)c(0xffffffbb)c(0xffffffbc)c(0xffffffbd)c(0xffffffbe)c(0xffffffbf)c(0xffffffc0)c(0xffffffc1)c(0xffffffc2)c(0xffffffc3)c(0xffffffc4)c(0xffffffc5)c(0xffffffc6)c(0xffffffc7)c(0xffffffc8)c(0xffffffc9)c(0xffffffca)c(0xffffffcb)c(0xffffffcc)c(0xffffffcd)c(0xffffffce)c(0xffffffcf)c(0xffffffd0)c(0xffffffd1)c(0xffffffd2)c(0xffffffd3)c(0xffffffd4)c(0xffffffd5)c(0xffffffd6)c(0xffffffd7)c(0xffffffd8)c(0xffffffd9)c(0xffffffda)c(0xffffffdb)c(0xffffffdc)c(0xffffffdd)c(0xffffffde)c(0xffffffdf)c(0xffffffe0)c(0xffffffe1)c(0xffffffe2)c(0xffffffe3)c(0xffffffe4)c(0xffffffe5)c(0xffffffe6)c(0xffffffe7)c(0xffffffe8)c(0xffffffe9)c(0xffffffea)c(0xffffffeb)c(0xffffffec)c(0xffffffed)c(0xffffffee)c(0xffffffef)c(0xfffffff0)c(0xfffffff1)c(0xfffffff2)c(0xfffffff3)c(0xfffffff4)c(0xfffffff5)c(0xfffffff6)c(0xfffffff7)c(0xfffffff8)c(0xfffffff9)c(0xfffffffa)c(0xfffffffb)c(0xfffffffc)c(0xfffffffd)c(0xfffffffe)c(0xffffffff)c(0x0)c(0x1)c(0x2)c(0x3)c(0x4)...
```

尝试打印发现是一个死循环，原理同2.3.1类似，这里不再展开。

# 总结

以后写代码的时候请重视，无形之中的类型转换。所以，养成一个好的习惯比较重要，不然到时候根本连错误都很难排查出来。

# 验证

如何快速验证结果，笔者除了使用CLion工具编译，Linux gcc编译，还有在线编译网站的编译。

其中一个在线编译网站，操作更加简单，有网就能使用，点击[这里](https://www.tutorialspoint.com/compile_c_online.php)。

{{< image classes="fancybox center fig-100" src="/c++/base/Implicit_conversion/ic_2.png" thumbnail="/c++/base/Implicit_conversion/ic_2.png" title="">}}

# 参考

[[1] 范桂飓, C 语言编程 — 基本数据类型, 2023.](https://blog.csdn.net/Jmilk/article/details/105263837)

[[2] 今年没有雪, 关于C语言float类型范围的理解, 2022.](https://blog.csdn.net/qq_51627033/article/details/124371622)

[[3] Forster-C, 再学C语言17：类型转换, 2022.](https://blog.csdn.net/changxiaoyong8/article/details/128324952)

[[4] 二进制人生, C语言进阶修炼之--隐式类型转换的陷阱, 2020.](https://www.modb.pro/db/227657)

[[5] Solieaor, 隐式类型转换的陷阱, 2019.](https://blog.csdn.net/qq_43514458/article/details/89946117/)