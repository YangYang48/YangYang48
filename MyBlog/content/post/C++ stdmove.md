---
title: "C++ std::move"
date: 2022-08-28
thumbnailImagePosition: left
thumbnailImage: c++/base/base3_thumb1.jpg
coverImage: c++/base/base3_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- std::move
- 2022
- August 
tags:
- c++
- c++基础
- Android
- 源码
showSocial: false
---

Android源码中有一些经常会遇到的c++基础的内容，笔者对std::move函数进行简单熟悉。

<!--more-->
# C++ std::move

> 开篇三连问
>
> 1. 为什么会引入`std::move`这个函数
> 2. 另外什么是左值，什么是右值
> 3. 有哪些是右值引用参数的函数



# 0前言

```c++
class DataClass {
public:
    DataClass(int size) : size(size) {
        data = new char[size];
        memset(data, 0, size);
    }

    void setData(const char* string)
    {
        std::cout<< "Data len " << strlen(data) << std::endl;
        if(!strlen(data))
        {
            memmove(data, string, size);
        }
        std::cout<< "Data string " << data << std::endl;
    }

    // 深拷贝构造
    DataClass(const DataClass& temp_array) {
        std::cout<< " DataClass deep copy " << temp_array.data << std::endl;
        size = temp_array.size;
        data = new char[size];
        for (int i = 0; i < size; i ++) {
            data[i] = temp_array.data[i];
        }
    }

    // 深拷贝赋值
    DataClass& operator=(const DataClass& temp_array) {
        std::cout<< " DataClass deep  operator " << temp_array.data << std::endl;
        delete[] data;

        size = temp_array.size;
        data = new char[size];
        for (int i = 0; i < size; i ++) {
            data[i] = temp_array.data[i];
        }
        return *this;
    }

    ~DataClass() {
        std::cout << "start ~DataClass" << std::endl;
        delete[] data;
    }

private:
    char *data;
    int size;
};
```

这是一个数据类，我需要一个数据类的对象直接使用之前的对象内容，这个时候往往需要一次深拷贝，比如拷贝构造函数，或者是运算符拷贝函数等。

```c++
int main()
{
    DataClass a(12);
    a.setData((char*)"Hello, Iron");
    DataClass b(a);//拷贝构造,深拷贝
    DataClass c(12);
    c = a;//运算符拷贝，深拷贝
    return 0;
}
```

如果说需要的一个数据类的对象直接使用之前的对象内容，并且之前的对象可以释放，那么可以采用移动函数。

移动函数可以将之前对象的内容移到新的对象中来。

```c++
//如果使用浅拷贝来作为移动函数，往往会有问题
DataClass(const DataClass& temp_array) {
    data = temp_array.data;
    size = temp_array.size;
    // 为防止temp_array析构时delete data，提前置空其data 
    temp_array.data = nullptr;
}
```

会提示`Variable 'temp_array' declared const here cannot assign to variable 'temp_array' with const-qualified type 'const DataClass &'`，即不能直接修改传参temp_array。

无法实现！`temp_array`是个const左值引用，无法被修改，所以`temp_array.data_ = nullptr;`这行会编译不过。当然函数参数可以改成非const：`DataClass(DataClass & temp_array){...}`，这样也有问题，由于左值引用不能接右值，`DataClass a = DataClass(DataClass());`这种调用方式就没法用了。

这个时候需要一个右值的引用来解决这个问题。

使用方式

```c++
//左值a，用std::move转化为右值
DataClass c(std::move(a));
```



# 1左值右值

左值是表达式结束后依然存在的持久对象(代表一个在内存中占有确定位置的对象)

右值是表达式结束时不再存在的临时对象(不在内存中占有确定位置的表达式）

便携方法：对表达式取地址，如果能，则为左值，否则为右值

```c++
int val;
val = 4; // 正确 ①
4 = val; // 错误 ②
```

运算符=要求等号左边是可修改的左值，4是临时参与运算的值，一般在寄存器上暂存，运算结束后在寄存器上移除该值，故①是对的，②是错的。一个对象被用作右值时，使用的是它的内容(值)，被当作左值时，使用的是它的地址**。**

**左值引用**

左值引用的基本语法

```c++
T& TypeName = 左值表达式；
```

**右值引用**

右值引用基本语法

```c++
T&& TypeName = 左值表达式；
```

而**std::move函数**

主要可以将一个左值转换成右值引用，从而可以调用C++11右值引用的拷贝构造函数



**剖析`std::move`源码**

转发左值，将左值引用转化成右值引用

```c++
template<typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type& __t) noexcept
{ return static_cast<_Tp&&>(__t); }
```

转发右值，将右值引用转化成右值引用

```c++
template<typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
{
  static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
        " substituting _Tp is an lvalue reference type");
  return static_cast<_Tp&&>(__t);
}
```

实际上，`std::move`被广泛用于在STL和自定义类中**实现移动语义，避免拷贝，从而提升程序性能**。



# 2右值引用参数的函数

在STL的很多容器中，都实现了以**右值引用为参数**的移动构造函数和移动赋值重载函数。

最常见的是vector数组中的推数据函数，`push_back`和`emplace_back`，目前更多使用的是`emplace_back`。

**参数为左值引用意味着拷贝，为右值引用意味着移动。**

```c++
#include <iostream>
#include <vector>

int main() {
    std::string str1 = "20220828";
    std::vector<std::string> v;

    v.push_back(str1); // 1.传统方法，copy
    std::cout << v.back() << std::endl;
    v.push_back(std::move(str1)); // 2.调用移动语义的push_back方法，避免拷贝，str1会失去原有值，变成空字符串
    std::cout << v.back() << std::endl;
    v.emplace_back(std::move(str1)); // 3.emplace_back效果相同，str1会失去原有值
    std::cout << v.back() << std::endl;
    v.emplace_back("20220829"); // 当然可以直接接右值
    std::cout << v.back() << std::endl;
}
```

结果

{{< image classes="fancybox center fig-100" src="/c++/base/base3_1.png" thumbnail="/c++/base/base3_1.png" title="">}}

查看源码可以得知

```c++
//stl_vector.h
//第1个函数的原型，是一个左值引用
void push_back(const value_type& __x)
{
    ...
}

//第2个函数的原型，是一个右值引用，实际调用的是emplace_back
#if __cplusplus >= 201103L
void push_back(value_type&& __x)
{
    emplace_back(std::move(__x));
}

//stl_bvector.h
void emplace_back(_Args&&... __args)
{
    push_back(bool(__args...));
    #if __cplusplus > 201402L
    return back();
    #endif
}
```

**因此，可移动对象在<需要拷贝且被拷贝者之后不再被需要>的场景，建议使用**`std::move`**触发移动语义，提升性能。**



# 3解答

> - 为什么会引入`std::move`这个函数
>
>   `STL`和自定义类中，左值转换成右值引用实现移动语义，避免深拷贝，从而提升程序性能
>
> - 另外什么是左值，什么是右值
>
>   通俗来讲左值是位于运算符左边，右值是运算符右边的值。
>
>   左值是表达式结束后依然存在的持久对象(代表一个在内存中占有确定位置的对象)
>
>   右值是表达式结束时不再存在的临时对象(不在内存中占有确定位置的表达式）
>
> - 有哪些是右值引用参数的函数
>
>   `STL`中常见的函数都有右值引用参数的函数，比如vector里面的`emplace_back`,`unique_ptr`
>
>   例如
>
>   ```c++
>   std::unique_ptr<A> ptr_a = std::make_unique<A>();
>   std::unique_ptr<A> ptr_b = std::move(ptr_a); // unique_ptr只有移动赋值重载函数，参数是&& ，只能接右值，因此必须用std::move转换类型
>   std::unique_ptr<A> ptr_b = ptr_a; // 编译不通过
>   ```

