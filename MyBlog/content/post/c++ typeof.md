---
title: "c++ typeof"
date: 2023-02-11
thumbnailImagePosition: left
thumbnailImage: c++/base/typeof/typeof_thumb.jpg
coverImage: c++/base/typeof/typeof_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- typeof
- 2023
- February
tags:
- c++
- c++基础
showSocial: false
---
本文以最通俗易懂的方式熟悉c++ typeof。

<!--more-->
# 0简介

Typeof用于指定变量的数据类型，该关键字使用时的语法类似sizeof，但是其结构在语义上类似于用typedef定义的类型名称。
Typeof有两种形式的参数：**表达式** 或者 **类型**。

> GCC的一个扩展，具体可以点击[这里](https://gcc.gnu.org/onlinedocs/gcc/Typeof.html)
>
> 下面是使用表达式作为参数的例子：
>
> ```c
> typeof(x[0](1))
> ```
>
> 在此假设x是一个函数指针数组；该类型描述的是函数返回值。
>
> 下面是使用类型作为参数的例子：
>
> ```c
> typeof(int *)
> ```
>
> 在此该类型描述的是一个指向int的指针。
>
> 如果在一个被包含在ISO C程序中的头文件中使用该关键字，请使用__typeof__代替typeof。
> typeof可用在任何可使用typedef的地方，例如，可将它用在声明，sizeof或typeof内。
>
> typeof在声明中与表达式结合非常有用，下面是二者结合用来定义一个获取最大值的“宏“：
>
> ```c
> #define max(a,b) \
>        ({ typeof (a) _a = (a); \
>            typeof (b) _b = (b); \
>          _a > _b ? _a : _b; })
> ```
>
> 本地变量使用下划线命名是为了避免与表达式中的变量名a和b冲突。
>
> 下面是一些typeof使用示例：
>
> - 声明y为x指向的数据类型
>
>   ```c
>   typeof(*x) y;
>   ```
>
> - 声明y为x指向的数据类型的数组
>
>   ```c
>   typeof(*x) y[4];
>   ```
>
> - 声明y为字符指针数组
>
>   ```c
>   typeof(typeof(char *)[4]) y;
>   ```
>
> 上述声明等价于 `char *y[4];`
>
> Typeof在linux内中使用广泛，比如，宏container_of用于获取包含member的结构体type的地址：
>
> ```c
> /**
>  * container_of - cast a member of a structure out to the containing structure
>  *
>  * @ptr:    the pointer to the member.
>  * @type:   the type of the container struct this is embedded in.
>  * @member: the name of the member within the struct.
>  *
>  */
> #define container_of(ptr, type, member) ({          \
>     const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
>     (type *)( (char *)__mptr - offsetof(type,member) );})
> ```

# 1demo

## 1.1demo1

```c++
// demo1
int var = 2023;
//实际上为int* pvar = &var;
typeof(int *) pvar = &var;
printf("pvar:\t%p\n", pvar);
printf("&var:\t%p\n", &var);
printf("var:\t%d\n", var);
printf("*pvar:\t%d\n", *pvar);
```

我们先定义了一个 int 型变量 var，然后再定义一个指针型变量指向 var，一般我们就直接 int *xxx = &xx，但是我们为了演示 typeof 的用法，就不要这么直接了，typeof 是自动推导后面 ( ) 里的数据类型，所以 typeof(int *) 直接推导出了 int * 型，用这个类型声明了 pvar 并将其初始化为 var 的地址

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_1.png" thumbnail="/c++/base/typeof/typeof_1.png" title="">}}

## 1.2demo2

```c
// demo2
int *pvar = NULL;
//这里的typeof(pvar)相当于int*
typeof(*pvar) var = 2023;
printf("var:\t%d\n", var);
```

这个例子是先定义了一个整型指针变量 pvar，typeof 后面括号里的表达式为*pvar，pvar 的类型为 int * 型，那 *pvar 当然就被解析为 int 型，所以用这个类型声明的变量 var 也是 int 型

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_2.png" thumbnail="/c++/base/typeof/typeof_2.png" title="">}}

## 1.3demo3

```c
// demo3
int *pvar = NULL;
//根据demo2直到这里typeof(*pvar)其实就是int
typeof(*pvar) var[4] = {2023, 02, 11, 1508};
for (int i = 0; i < 4; i++)
	printf("var[%d]:\t%d\n", i, var[i]);
```

这次 typeof 解析出来的类型跟上一个一样，区别是 var 是一个包含四个元素的数组，相当于 int var[4] = {...}

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_3.png" thumbnail="/c++/base/typeof/typeof_3.png" title="">}}

## 1.4demo4

```c
// demo4
//内层的tyeopf是typeof(const char *) a;
//叠加外层的typeof是typeof(a[4])类型，实际上是指针数组
typeof(typeof(const char *)[4]) pchar = {"hello", "world", "i am", "yangyang48"};
for (int i = 0; i < 4; i++)
	printf("pchar[%d]:\t%s\n", i, pchar[i]);
```

这次嵌套了两层 typeof，我们一层一层的往外剥，先看最里层，typeof 先解析出一个 const char * 类型，有经验的 C 程序员应该马上就能联想到字符串了吧，而后面又跟着一个 [4]，说明这是一个包含四个字符串的数组类型，那么这个类型也就被最外层的 typeof 给解析到了，那么最终的 pchar 也就是这个类型了，相当于 const char *pchar[4] = {...}; 

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_4.png" thumbnail="/c++/base/typeof/typeof_4.png" title="">}}

## 1.5demo5

```c
// demo5
int add(int param1, int param2) {
	return param1 + param2;
}
 
int sub(int param1, int param2) {
	return param1 - param2;
}
 
int mul(int param1, int param2) {
	return param1 * param2;
}
 
int main() {
 //(int (*fun)(int, int)为函数指针)，这里就是函数指针数组
    int (*func[3]) (int, int) = {add, sub, mul};
    //被typeof包起来的函数指针，函数并不会被调用
    typeof(func[0](1, 1)) sum = 100;
    typeof(func[1](1, 1)) dif = 101;
    typeof(func[2](1, 1)) pro = 102;
    printf("sum:\t%d\n", sum);
    printf("dif:\t%d\n", dif);
    printf("pro:\t%d\n", pro);

    printf("start calc\n", pro);
    //函数真正被调用
    int sum1 = func[0](2023, 2);
    int dif1 = func[1](2023, 2);
    int pro1 = func[2](2023, 2);
    printf("sum1:\t%d\n", sum1);
    printf("dif1:\t%d\n", dif1);
    printf("pro1:\t%d\n", pro1);
    return 0;
}
```

这个 demo 中先定义了三个函数，这三个函数都是返回值为 int 类型，并且接受两个 int 型的参数，然后在 main 函数中定义了一个函数指针数组 func，并用上面三个函数名将其初始化，然后我们来看底下的第一个 typeof  会推导出什么类型，func[0] 就是指 add 这个函数，后面的括号里跟了两个参数，说白了就是简单的 add(1, 1) 的调用，而 add 会返回一个 int 型值，所以最终推导出的类型就是 int 型，其它两个都是同理，所以 sum、dif、pro 其实就是三个整型数，相当于 int sum = 100; int dif = 101; int pro = 102;

**说明了函数指针被typeof关键字包起来，并不会运行函数本身**。

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_5.png" thumbnail="/c++/base/typeof/typeof_5.png" title="">}}

## 1.6demo6

```c
// demo 06
#define pointer(T)  typeof(T *)
#define array(T, N) typeof(T[N])
 
int main() {
    //这里有点c++类模板的感觉pointer(char)实际上是typeof(char*)
    //外面的array(typeof(char*),4)实际上是typeof(typeof(char*)[4])，跟上述demo4一致
	array(pointer(char), 4) pchar = {"hello", "world", "i am", "yangyang48"};
	for (int i = 0; i < 4; i++)
		printf("pchar[%d]:\t%s\n", i, pchar[i]);
	return 0;
}
```

这里用到了宏函数，pointer(T) 会被替换为 typeof(T *)，也就是说 pointer 后面跟某个类型的名字，经过预处理之后就会变成用 typeof 解析相应类型的指针类型，而 array 后面跟一个类型名和一个整数，然后 typeof 就会解析为该类型的一个数组，这样 main 函数中的 array(pointer(char), 4)，pointer(char) 首先会被解析为 char * 型，然后外层的 array 会再被解析为包含 4 个 char * 元素的数组类型，所以就相当于 char *pchar[4] = {...}; 

{{< image classes="fancybox center fig-100" src="/c++/base/typeof/typeof_6.png" thumbnail="/c++/base/typeof/typeof_6.png" title="">}}

# 2总结

 typeof关键字是C语言中的一个新扩展，这个特性在linux内核中应用非常广泛。

由于`typeof()`是在编译时处理的，故**其中的表达式在运行时是不会被执行的**，比如`typeof(fun())`，`fun()`函数是不会被执行的，`typeof`只是在编译时分析得到了`fun()`的返回值而已。

另外`typeof`还有一些局限性，其中的变量是不能包含存储类说明符的，如`static`、`extern`这类都是不行的。



# 

