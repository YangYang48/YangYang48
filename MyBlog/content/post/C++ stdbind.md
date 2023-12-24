---
title: "C++ std::bind"
date: 2023-07-18
thumbnailImagePosition: left
thumbnailImage: c++/base/bind/bind_thumb.jpg
coverImage: c++/base/bind/bind_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- bind
- 2023
- July
tags:
- 基础
- c++
- 占位符
- placeholders
- is_placeholder
- is_bind_expression
- 引用传递
showSocial: false
---

学习一下关于C++基础知识std::bind


<!--more-->
# 1简介

## 1.1头文件

```c++
#include <functional>
```

## 1.2函数原型

```c++
/**
   *  std::bind的函数模板
   */
template<typename _Func, typename... _BoundArgs>
inline _GLIBCXX20_CONSTEXPR typename
_Bind_helper<__is_socketlike<_Func>::value, _Func, _BoundArgs...>::type
    bind(_Func&& __f, _BoundArgs&&... __args)
{
    typedef _Bind_helper<false, _Func, _BoundArgs...> __helper_type;
    return typename __helper_type::type(std::forward<_Func>(__f),
                                        std::forward<_BoundArgs>(__args)...);
}

/**
   *  std::bind<R>的函数模板
   */
template<typename _Result, typename _Func, typename... _BoundArgs>
inline _GLIBCXX20_CONSTEXPR
typename _Bindres_helper<_Result, _Func, _BoundArgs...>::type
    bind(_Func&& __f, _BoundArgs&&... __args)
{
    typedef _Bindres_helper<_Result, _Func, _BoundArgs...> __helper_type;
    return typename __helper_type::type(std::forward<_Func>(__f),
                                        std::forward<_BoundArgs>(__args)...);
}
```

## 1.3参数

std::bind返回一个基于f的函数对象，其参数被绑定到args上。

f的参数要么被绑定到值，要么被绑定到placeholders（占位符，如_1, _2, ..., _29）。

> 什么是占位符
>
> 在C++中，占位符通常是指模板中的占位类型或占位值。在模板中，我们可以使用占位类型来表示将在实例化时替换为实际类型的类型参数。例如，我们可以定义一个模板类，其中的T就是一个占位类型：
>
> ```c++
> template <typename T>
> class MyClass
> {
>   //类的定义  
> };
> ```
>
> 在实例化时，我们可以指定T的具体类型，例如int、float等，来创建一个具体的类：
>
> ```c++
> MyClass<int> obj;//使用int作为占用符
> ```
>
> 此外，在C++中，我们还可以使用占位值来表示将在函数调用或模板实例化时替换为实际值的参数。例如，我们可以定义一个函数，其中的参数使用占位值：
>
> ```c++
> void printNumber(int number){
>     cout << "This number is " << number << endl;
> }
> ```
>
> 在调用函数时，我们可以传递具体的值作为参数：
>
> ```c++
> int number = 10;
> printNumber(number);
> ```
>
> 这些是C++中常见的占位符的一些示例。请注意，占位符的具体使用方式可能会根据上下文和具体的编程需求而有所不同。

std::bind将可调用对象与其参数一起进行绑定，绑定后的结果可以使用std::function保存。

 std::bind是一个标准函数对象，作为一个函数对象适配器(function adaptor)，可接受一个函数作为输入，并可绑定一个或多个被引用函数中的形参（同时允许将他们重新排列），最终返回一个新的函数对象作为输出

通俗一点，也就是可以将设计好的**函数和它所需要用到的形参**（传入的形参位置可以调控）绑定起来作为一个单独对象（这种处理方法多用在多线程上）

其中

> 关于默认的占位符有29个，即超过29个的占位符是不合法的
>
> ```c++
> namespace placeholders
> {
> /* Define a large number of placeholders. There is no way to
>  * simplify this with variadic templates, because we're introducing
>  * unique names for each.
>  */
>   extern const _Placeholder<1> _1;
>   extern const _Placeholder<2> _2;
>   extern const _Placeholder<3> _3;
>   extern const _Placeholder<4> _4;
>   extern const _Placeholder<5> _5;
>   extern const _Placeholder<6> _6;
>   extern const _Placeholder<7> _7;
>   extern const _Placeholder<8> _8;
>   extern const _Placeholder<9> _9;
>   extern const _Placeholder<10> _10;
>   extern const _Placeholder<11> _11;
>   extern const _Placeholder<12> _12;
>   extern const _Placeholder<13> _13;
>   extern const _Placeholder<14> _14;
>   extern const _Placeholder<15> _15;
>   extern const _Placeholder<16> _16;
>   extern const _Placeholder<17> _17;
>   extern const _Placeholder<18> _18;
>   extern const _Placeholder<19> _19;
>   extern const _Placeholder<20> _20;
>   extern const _Placeholder<21> _21;
>   extern const _Placeholder<22> _22;
>   extern const _Placeholder<23> _23;
>   extern const _Placeholder<24> _24;
>   extern const _Placeholder<25> _25;
>   extern const _Placeholder<26> _26;
>   extern const _Placeholder<27> _27;
>   extern const _Placeholder<28> _28;
>   extern const _Placeholder<29> _29;
> }
> ```

另外，还有占位符的判断

> 1）**is_placeholder**判断T是否为占位符
>
> ```c++
> #include <functional>
> template<int _Num>
> struct is_placeholder<const _Placeholder<_Num> >
>     : public integral_constant<int, _Num>
>     { };
> ```
>
> 具体例子
>
> ```c++
> #include <iostream>     // std::cout, std::boolalpha
> #include <functional>   // std::is_placeholder, std::placeholders
> using namespace std;
> int main () {
>     std::cout << std::is_placeholder<decltype(placeholders::_1)>::value << '\n';
>     std::cout << std::is_placeholder<decltype(placeholders::_2)>::value << '\n';
>     std::cout << std::is_placeholder<decltype(placeholders::_29)>::value << '\n';
>     std::cout << std::is_placeholder<int>::value << '\n';
> 
>     return 0;
> }
> ```
>
> 显然占位符最大的值判断为29，非占位符的话默认为0
>
> 输出
>
> ```c++
> 1
> 2
> 29
> 0
> ```
>
> 2）**is_bind_expression**判断是否为bind表达式
>
> ```c++
> #include <functional>
> template<typename _Tp>
> struct is_bind_expression
>     : public false_type { };
> ```
>
> 具体例子
>
> ```c++
> #include <iostream>     // std::cout, std::boolalpha
> #include <functional>   // std::is_placeholder, std::placeholders
> using namespace std;
> double Func (double x, double y)
> {
>     return x / y;
> }
> 
> int main()
> {
>     cout << boolalpha;
>     auto NewCallable = bind(Func, placeholders::_1, 2);
>     cout << NewCallable (10) << endl;
>     auto NewCallable2 = []{
>         cout << "I am lamda expression" << endl;
>     };
>     NewCallable2();
> 
>     cout << is_bind_expression<decltype(NewCallable)>::value << endl;
>     cout << is_bind_expression<decltype(NewCallable2)>::value << endl;
> }
> ```
>
> 输出
>
> ```c++
> 5
> I am lamda expression
> true
> false
> ```

## 1.4作用

std::bind主要有以下两个作用：

1. 将可调用对象和其参数绑定成一个防函数
2. 只绑定部分参数，减少可调用对象传入的参数
   

# 2bind使用

## 2.1绑定普通函数

```c++
#include<iostream>
#include <functional>
using namespace std;
double Func (double x, double y)
{
    return x / y;
}

int main()
{
    auto NewCallable = bind(Func, placeholders::_1, 2);
    cout << NewCallable (10) << endl;

}
```

输出

```c++
5
```

bind的第一个参数是函数名，普通函数做实参时，会隐式转换成函数指针。

第一个参数被占位符占用，表示这个参数以调用时传入的参数为准，在这里调用Func时，给它传入了10，其实就想到于调用Func(10,2);

> 尽量不要把占位符省略写成`_1`,`_2`类似这种，需要写全`placeholders::_1`和`placeholders::_2`，如果一定要使用，需要加上命名空间`using namespace std::placeholders;`
>
> 不然可能会出现下面的错误
>
> ```c++
> error: '_1' was not declared in this scope; did you mean 'std::placeholders::_1'?
> ```

另外改进一下，bind还可以嵌套变化

```c++
#include<iostream>
#include <functional>
using namespace std;
double Func (double x, double y)
{
    return x / y;
}

double Gunc (double x)
{
    return x * 2 + 30.0;
}

int main()
{
    auto NewCallable = bind(Func, placeholders::_3, bind(Gunc, placeholders::_3));
    cout << NewCallable (10, 20, 30) << endl;
}
```

输出

```
0.333333
```

这里虽然是嵌套型的bind，是一步一步拆解之后也就那么回事。

```c++
auto NewCallable = bind(Func, placeholders::_3, bind(Gunc, placeholders::_3));
//拆解第一步bind(Gunc, placeholders::_3)，其中placeholders::_3对应30，
auto NewCallable = bind(Func, placeholders::_3, bind(Gunc, 30));
//拆解第二步，是外部的bind
auto NewCallable = bind(Func, 30, bind(Gunc, 30));
//拆解第三步，NewCallable(10,20,30)调用了仿函数，会让第一步的值为60.0，第二步的值为0.33333
```



> 关于bind写法，main函数中的内容也可以写成下面的方式，有点类似**调用仿函数**表达式，结果都是一致的
>
> ```c++
> auto NewCallable = bind(Func, placeholders::_3, bind(Gunc, placeholders::_3))(10, 20, 30);
> cout << NewCallable << endl;
> ```



## 2.2绑定一个成员函数

bind绑定类成员函数时，第一个参数表示对象的**成员函数的指针**，第二个参数表示**对象的地址**。
必须显式地指定&A::display_del，因为编译器不会将对象的成员函数隐式转换成函数指针，所以必须在A::display_del前添加&；
使用对象成员函数的指针时，必须要知道该指针属于哪个对象，因此第二个参数为对象的地址 &a；

```c++
#include<iostream>
#include <functional>
using namespace std;
class A
{
public:
    int display_del(int a1, int a2)
    {
        return a1 - a2;
    }
};

int main()
{
    A a;
    //placeholders::_2对应地2个占位符，placeholders::_1对应第1个占位符
    auto NewCallable = bind(&A::display_del, &a, placeholders::_2, placeholders::_1);
    cout << NewCallable(20, 10) << endl;
}
```

输出

```c++
-10
```





## 2.3绑定一个引用参数

默认情况下，bind的那些不是占位符的参数被拷贝到bind返回的可调用对象中。但是，与lambda类似，有时对有些绑定的参数希望以引用的方式传递，或是要绑定参数的类型无法拷贝。

### 2.3.1引用传递到bind中的参数

```c++
#include<iostream>
#include <functional>
using namespace std;
class A
{
public:
    //值传递
    int display_del(int a1, int a2, int n1, int n2)
    {
        cout << "display_del n1 = " << n1 << " n2 = " << n2 << endl;
        n1 = a1 + a2;
        n2 = a1 + a2;
        return a1 - a2;
    }
};

int main()
{
    A a;
    int n = 10;
    //默认bind也是值传递，拷贝一份参数到bind中，需要引用传递需要加上ref
    auto NewCallable = bind(&A::display_del, &a, placeholders::_2, placeholders::_1, ref(n), n);
    cout << "result = " << NewCallable(20, 10) << " n = " <<  n << endl;
    n = 25;
    cout << "result = " << NewCallable(20, 10) << " n = " <<  n << endl;
}
```

输出

```c++
result = display_del n1 = 10 n2 = 10
-10 n = 10
result = display_del n1 = 25 n2 = 10
-10 n = 25
```

可以发现只有加引用的当外面的n值变化时，会让bind中的对应引用参数也随之变化。

### 2.3.2函数体参数引用传递到bind中的参数

如果说可以让引用，把main方法中的变量引用到bind中的参数，而bind中的参数是实参，调用到形参实际上是对实参的拷贝，是一种值传递。我们改成引用传递又会有不一样的收获。

```c++
#include<iostream>
#include <functional>
using namespace std;
class A
{
public:
    int display_del(int a1, int a2, int& n, int& m)
    {
        cout << "display_del n1 = " << n << " n2 = " << m << endl;
        n = a1 + a2;
        m = a1 + a2;
        return a1 - a2;
    }
};

int main()
{
    A a;
    int n = 10;
    int m = 10;
    auto NewCallable = bind(&A::display_del, &a, placeholders::_2, placeholders::_1, ref(n), m);
    cout << "result = " << NewCallable(20, 10) << " n = " <<  n << " m = " << m << endl;
    n = 25;
    m = 25;
    cout << "result = " << NewCallable(20, 10) << " n = " <<  n << " m = " << m << endl;
}
```

输出

```c++
result = display_del n1 = 10 n2 = 10
-10 n = 30 m = 10
result = display_del n1 = 25 n2 = 30
-10 n = 30 m = 25
```

这个稍微看起来复杂一点，理清楚两边的引用之后就会很简单。

{{< image classes="fancybox center fig-100" src="/c++/base/bind/bind_1.png" thumbnail="/c++/base/bind/bind_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/c++/base/bind/bind_2.png" thumbnail="/c++/base/bind/bind_2.png" title="">}}

# 3总结

总的来说，bind的思想实际上是一种**延迟计算**的思想，将可调用对象保存起来，然后在需要的时候再调用。而且这种绑定是非常灵活的，不论是普通函数、函数对象、还是成员函数都可以绑定，而且其参数可以支持占位符。

# 参考

[[1] 云飞扬_Dylan. C++11中的std::bind 简单易懂, 2022.](https://blog.csdn.net/Jxianxu/article/details/107382049?spm=1001.2101.3001.6650.7&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-107382049-blog-123106149.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-107382049-blog-123106149.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=10)

[[1] 一苇渡江694. C++11新特性应用--占位符(std::placeholders std::is_placeholder std::is_bind_expression), 2016.](https://dabaojian.blog.csdn.net/article/details/50471709?spm=1001.2101.3001.6650.4&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-4-50471709-blog-123106149.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-4-50471709-blog-123106149.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=7&ydreferer=aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzE2MzcwNTU5L2FydGljbGUvZGV0YWlscy8xMjMxMDYxNDk%3D)

[[1] Jinxk8. 【C++深陷】之“decltype”, 2020.](https://blog.csdn.net/u014609638/article/details/106987131)