---
title: "C++ operator重载"
date: 2023-07-23
thumbnailImagePosition: left
thumbnailImage: c++/base/operator/op_thumb.png
coverImage: c++/base/operator/op_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- c++
- 2023
- July
tags:
- 基础
- 类型转换
- 仿函数
- 谓词
- 指针操作
- 自定义类型
showSocial: false
---

源码中出现很多关于operator重载的操作符，但是本文着重讲述关于小括号，指针操作，自定义类型和类型转换。


<!--more-->
# 1介绍

我们在设计一个类的时候，不可避免的需要在某些时候对这个类的实例进行操作符运算，例如比较、自增、自减等。此时就必须在类中重载相应的操作符，如果能够自定义一些规则，就可以让类之间像基本整数一样能够自由的运算。比如string类，可以使用加号`+`和`==`等。

```c++
#include <string>
#include <iostream>

using namespace std;

int main()
{
    //使用 + 操作符拼接两个字符串
    string str1("2023 ");
    string str2("yangyang48");
    string str3 = str1 + str2;//"2023 yangyang48"
    cout << str3 << endl;
    //使用==来比较两个字符串的内容
    if (str1 == str2)
        cout << "str1 = str2" << endl;
    else if (str1 < str2)
        cout << "str1 < str2" << endl;
    else
        cout << "str1 > str2" << endl;

    return 0;
}
```

输出结果

```c++
2023 yangyang48
str1 < str2
```

多数的C++操作符都可以被重载，重载的操作符（有些情况例外）不必是成员函数，但必须至少有一个操作数是用户定义的类型。下面详细介绍C++对用户定义的操作符重载的**限制**

1. 运算符的优先级（precedence）不可改变。例如，除法的运算优先级永远高于加法。

2. 不能引入新的操作符。例如，不能定义operator**()函数来表示求幂。

3. 不能重载以下操作符

   `sizeof` ——**sizeof操作符**

   `.`——**成员操作符**

   `.*`——**成员指针操作符**

   `::`——**作用域解析操作符**

   `?:`——**条件操作符**

   `typeid`——**一个RTTI操作符**

   `const_cast`——**强制类型转换操作符**

   `dynamic_cast`——**强制类型转换操作符**

   `reinterpret_cast`——**强制类型转换操作符**

   `static_cast`——**强制类型转换操作符**

   可以重载的运算符表

   |   `=`   |    `()`    | `[]`  |   `->`   |
   | :-----: | :--------: | :---: | :------: |
   |   `+`   |    `-`     |  `*`  |   `/`    |
   |   `%`   |    `^`     |  `&`  |   `|`    |
   |  `~=`   |    `!`     |  `<`  |   `>`    |
   |  `+=`   |    `-=`    | `*=`  |   `/=`   |
   |  `%=`   |    `^=`    | `&=`  |   `|=`   |
   |  `<<`   |    `>>`    | `>>=` |  `<<=`   |
   |  `==`   |    `!=`    | `<=`  |   `>=`   |
   |  `&&`   |    `||`    | `++`  |   `--`   |
   |   `,`   |   `->*`    | `new` | `delete` |
   | `new[]` | `delete[]` |       |          |

   

# 2具体demo

## 2.1仿函数

仿函数(functor)，就是使一个类的使用看上去像一个函数。其实现就是类中实现一个`operator()`，这个类就有了类似函数的行为，就是一个仿函数类了。仿函数是比较特殊的重载符号，为括号`()`

```c++
#include <string>
#include <vector>
#include <iostream>

using namespace std;
//筛选大于5的数
class GreaterFive
{
public:
    GreaterFive()
    {
        cout << "GreaterFive constructor" << endl;
    }

    bool operator()(int val)
    {
        cout << "GreaterFive functor" << endl;
        return val > 5;
    }
};

class MyCompare{
public:
    //排序从大到小
    bool operator()(int v1,int v2)
    {
        return v1 > v2;
    }
};
int main()
{
    vector v{2,0,2,3,0,7,0,8};
    //传入的GreaterFive()是一个匿名的对象，通过传入的匿名对象调用这个对象的仿函数
    vector<int>::iterator iter = find_if(v.begin(), v.end(), GreaterFive());
    for(int i = 0;i <v.size(); i++)
    {
        cout << v[i] << "\t";
    }
    cout << endl;
    sort(v.begin(), v.end(), MyCompare());
    for(int i = 0;i <v.size(); i++)
    {
        cout << v[i] << "\t";
    }
}
```

这里可以看到处理STL中的查找函数和排序函数，都使用到了仿函数。

这里的仿函数作用原理也是比较巧妙，**利用匿名的构造函数调用传入对象**，**然后STL内部会调用对应的仿函数达到目的**。

另外特别的是，如果仿函数的**返回类型为bool类型，那么可以称为谓词**，有几个参数可以称为几元谓词。

```c++
//一元谓词
bool GreaterFive::operator()(int val);
//二元谓词
bool MyCompare::operator()(int v1,int v2);
```



## 2.2指针/引用相关运算符重载

在强指针中，出现了使用指针运算重载的运算符操作，具体可以查看[Android智能指针解析](https://yangyang48.github.io/2022/11/android%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88%E8%A7%A3%E6%9E%90/)

```c++
//system_core/blob/HEAD/libutils/include/utils/StrongPointer.h
template<typename T>
class sp {
public:
    ...
    // Accessors
    inline T&       operator* () const     { return *m_ptr; }
    inline T*       operator-> () const    { return m_ptr;  }
    inline T*       get() const            { return m_ptr; }
    inline explicit operator bool () const { return m_ptr != nullptr; }
};
```

如果这个时候调用

```c++
using namespace android;
class A : public RefBase
{
public:
    virtual ~A(){};
    int value()
    {
        return 723;
    }
};

int main()
{
    sp<A> a = new A();
    cout << a->value() << endl;
    return 0;
}
```

输出结果

```
723
```

上面的a->value，这里的a并不是一个指针（a保存的不是一个地址，a是一个`sp<a>`的临时变量），但是有一个运算符重载，这个运算符重载相当于是指针的运算。

```c++
a->value();
//等价于 a.operator->()->value();
//等价于 m_ptr->value();
//等价于 (*m_ptr).value();这里才真正调用到了默认的指针运算方式，调用到函数value
```



## 2.3自定义字符串重载

这里举一个源码中的操作，用于FPS的计算，即1/60s的计算

```c++
//frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/Fps.h
class Fps {
public:
    constexpr Fps() = default;

    static constexpr Fps fromValue(float frequency) {
        return frequency > 0.f ? Fps(frequency, static_cast<nsecs_t>(1e9f / frequency)) : Fps();
    }

    static constexpr Fps fromPeriodNsecs(nsecs_t period) {
        return period > 0 ? Fps(1e9f / period, period) : Fps();
    }

    constexpr bool isValid() const { return mFrequency > 0.f; }

    constexpr float getValue() const { return mFrequency; }
    int getIntValue() const { return static_cast<int>(std::round(mFrequency)); }

    constexpr nsecs_t getPeriodNsecs() const { return mPeriod; }

private:
    constexpr Fps(float frequency, nsecs_t period) : mFrequency(frequency), mPeriod(period) {}

    float mFrequency = 0.f;
    nsecs_t mPeriod = 0;
};

struct FpsRange {
    Fps min = Fps::fromValue(0.f);
    Fps max = Fps::fromValue(std::numeric_limits<float>::max());

    bool includes(Fps) const;
};
//类外定义
constexpr Fps operator""_Hz(unsigned long long frequency) {
    return Fps::fromValue(static_cast<float>(frequency));
}
//类外定义
constexpr Fps operator""_Hz(long double frequency) {
    return Fps::fromValue(static_cast<float>(frequency));
}
```

这个类，只有两个内部私有变量，一个是外部传入定义的帧率，可以是60.0，也可以是其他浮点数，后一个参数即为转换为毫秒的数，对应具体的1/mFrequency的值。这个封装起来非常巧妙，使用默认的例子即可。

```c++
//使用方式
// Frames per second, stored as floating-point frequency. Provides conversion from/to period in
// nanoseconds, and relational operators with precision threshold.
//
//     const Fps fps = 60_Hz;
//
//     using namespace fps_approx_ops;
//     assert(fps == Fps::fromPeriodNsecs(16'666'667));
//
```

完整版

```c++
#include <iostream>
#include <stdio.h>
using namespace std;
class Fps {
public:
    constexpr Fps() = default;

    static constexpr Fps fromValue(float frequency) {
        return frequency > 0.f ? Fps(frequency, static_cast<long long>(1e9f / frequency)) : Fps();
    }

    static constexpr Fps fromPeriodNsecs(long long period) {
        return period > 0 ? Fps(1e9f / period, period) : Fps();
    }

    constexpr bool isValid() const { return mFrequency > 0.f; }

    constexpr float getValue() const { return mFrequency; }

    constexpr long long getPeriodNsecs() const { return mPeriod; }

private:
    constexpr Fps(float frequency, long long period) : mFrequency(frequency), mPeriod(period) {}

    float mFrequency = 0.f;
    long long mPeriod = 0;
};

struct FpsRange {
    Fps min = Fps::fromValue(0.f);

    bool includes(Fps) const;
};
//类外定义
constexpr Fps operator""_Hz(unsigned long long frequency) {
    printf("unsigned long long frequency %llu\n", frequency);
    return Fps::fromValue(static_cast<float>(frequency));
}
//类外定义
constexpr Fps operator""_Hz(long double frequency) {
    printf("unsigned long double frequency %llf\n", frequency);
    return Fps::fromValue(static_cast<float>(frequency));
}

int main()
{
    const Fps fps = 59.5_Hz;
    cout << fps.getValue() << endl;
    cout << fps.getPeriodNsecs() << endl;
}
```

输出结果

```c++
//60
//16666667
59.5
16806722
```

另外上面加printf或者cout日志，在C++20会出错，但是低版本使用printf不会报错。constexpr函数中原因是使用了non-'constexpr'的函数，具体可以点击[这里](https://stackoverflow.com/questions/60841136/c-error-call-to-non-constexpr-function)。

{{< image classes="fancybox center fig-100" src="/c++/base/operator/op_1.png" thumbnail="/c++/base/operator/op_1.png" title="">}}

## 2.4类型转换

这里类型转换，实际上就将类强制转化成某成类型（多见基本类型）

```c++
#include <iostream>
using namespace std;
class A{
public:
    A(int num) : mNum(num) {
        cout << "A constructor " << this << endl;
    }
    ~A(){
        cout << "A destructor " << this << endl;
    }
    //使用显示方式，使用explicit关键字后不支持隐式转换
    explicit operator int() const
    {
        cout << "A operator int " << this << endl;
        return mNum;
    }
private:
    int mNum = 0;
};
int main()
{
    A a(2023);
    //类本身转换成int类型和0723就行运算
    cout << (int)a + 0723 << endl;
    //cout << static_cast<int>(a) + 0723 << endl;
}
```

输出结果为

```c++
A constructor 0x72c59ffa2c
A operator int 0x72c59ffa2c
2490
A destructor 0x72c59ffa2c
```

进一步的，转换成其他类型

```c++
#include <iostream>
using namespace std;
class B{
public:
    B(){
        cout << "B constructor " << this << endl;
    }

    ~B(){
        cout << "B destructor " << this << endl;
    }
    void print()
    {
        cout << "B print " << this << endl;
    }
};

class A{
public:
    A(int num, B& b) : mNum(num), mB(b) {
        cout << "A constructor " << this << endl;
    }
    ~A(){
        cout << "A destructor " << this << endl;
    }
    //默认隐式转换
    explicit operator B() const
    {
        cout << "A operator int " << this << endl;
        return mB;
    }
private:
    int mNum = 0;
    B mB;
};

int main()
{
    B b;
    A a(2023, b);
    static_cast<B>(a).print();
}
```

输出结果

```c++
B constructor 0xb33cbffc5e
A constructor 0xb33cbffc54
A operator int 0xb33cbffc54
B print 0xb33cbffc5f
B destructor 0xb33cbffc5f
A destructor 0xb33cbffc54
B destructor 0xb33cbffc58
B destructor 0xb33cbffc5e
```

转换结果虽好，但不建议不同类型之间转换，一般这种类转换成其他类型并不是很多见，最多会用到在大括号的隐式类型转化中，具体可以点击[这里查看](https://yangyang48.github.io/2023/07/c-%E5%A4%A7%E6%8B%AC%E5%8F%B7/)。

# 3总结

总的来说，操作符的高阶玩法，远远不止上述，上述也只是笔者遇到的一个小总结。如果后续还出现一些比较高阶的用法会持续更新。另外操作符的重载虽然可以重载`new`和`delete`，但是不建议去修改，因为这样使用很容易造成逻辑混乱，导致出现意想不到的异常。

# 关于上述的在线编译

[点击这里](https://c.runoob.com/compile/12/)。使用的是菜鸟教程的c++编译器，运行速度还不较快，也不容易挂掉

{{< image classes="fancybox center fig-100" src="/c++/base/operator/op_2.png" thumbnail="/c++/base/operator/op_2.png" title="">}}

# 参考

[[1] 时空-大海水, std::chrono::duration详解, 2017.](https://blog.csdn.net/t114211200/article/details/78029553)

[[2] Iron Fist, C++ error: Call to non-constexpr function, 2020.](https://stackoverflow.com/questions/60841136/c-error-call-to-non-constexpr-function)

[[3] 时空-大海水, std::chrono::duration详解, 2017.](https://blog.csdn.net/t114211200/article/details/78029553)

[[4] shadow_xwl, C++操作符重载, 2022.](https://blog.csdn.net/shadow_xwl/article/details/125237563)

[[5] zzl_python, 关于'->'运算符的重载问题, 2018.](https://blog.csdn.net/zzl_python/article/details/83243353)

[[6] 软件技术分享, 记住这个，能少走弯路，C++两种隐式类型转换, 2020.](https://baijiahao.baidu.com/s?id=1679542777359982612&wfr=spider&for=pc)