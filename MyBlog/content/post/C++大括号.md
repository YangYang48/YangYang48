---
title: "C++大括号"
date: 2023-07-19
thumbnailImagePosition: left
thumbnailImage: c++/base/brace/brace_thumb.jpg
coverImage: c++/base/brace/brace_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- brace
- 2023
- July
tags:
- 大括号
- 圆括号
- 等号
- initializer_list
- 类型转化
- 仿函数
showSocial: false
---

学习一下关于C++基础知识大括号。

<!--more-->
# 0一个例子

```c++
#include <iostream>
using namespace std;
struct A
{
    A(int x);
};

A::A(int x)
{
    cout << "A::A x = " << x << endl;
}

int main() {
    A a(1);
    A b{2};
    return 0;
}
```

输出结果

```c++
A::A x = 1
A::A x = 2
```

很显然，在本文的演示demo中，**构造函数使用大括号和小括号两种方式是相同的作用**。那么，是不是所有的情况大括号都可以使用呢？

再进一步研究

```c++
#include <iostream>
using namespace std;
struct A
{
    ~A(){ cout << "~A()" << endl; }
    A(int x)
    {
        cout << "A::A x = " << x << endl;
    }
};


struct B
{
    A a;
    //B(){ cout << "B::B()" << endl;}
    ~B(){ cout << "~B()" << endl; }
};

int main() {
    B b1(1);//报错
    B b2{1};//没问题
    return 0;
}
```

可以发现括号和大括号是有区别的，如果去掉报错输出

```c++
A::A x = 1
~B()
~A()
```

这里发现大括号有初始化类内的元素的作用，属于聚合初始化，相当于B b2={1}等价成b2.a = 1，然后隐式的对1进行构造为b2.a = A(1)。这里的A(1)是一种c++的匿名构造。关于匿名构造可以点击[这里]()。 因此可以调用单参数构造函数A::A(int)对b2.a进行初始化。

```c++
//1.会进行一次A的匿名构造，a=1会隐式转化成A(1)
A(int x)
{
	cout << "A::A x = " << x << endl;
}
//2.然后在进行右值运算,调用默认拷贝构造函数
//B(const B &)(B)或者是B(B &&)(B)
B(const B& b1)
{
	this->a = b1->a;
}
//相当于是b2.a=A(1)只会调用一次A的构造，然后会对B进行默认拷贝构造
```

本文所有demo，笔者在Clion中演示结果。

# 1大括号

自**C++11标准**开始就引入了列表初始化的概念，即**支持使用{}对变量或对象进行初始化**，且与传统的变量初始化的规则一样，也分为拷贝初始化和直接初始化两种方式。大括号的本质实际上是`std::initializer_list`，直接讲解感觉会感觉云里雾里。

Q:首先看一下下面的初始化，哪一种是正确的?

```
int x(0);    // 初始值在圆括号内

int y = 0;   // 初始值在等号后面

int z{0};    // 初始值在大括号内
```

A:实际上，三种初始化都是正确的。只是三种初始化的含义不同的，需要区别对待。

1. 第一种，**使用括号的方式**，对变量或对象使用括号初始化的方式被称为**直接初始化**，其**本质是调用了相应的构造函数**
2. 第二种，**使用等号的方式**，用等号初始化的方式则被称为**拷贝初始化**。
3. 第三种，**使用大括号的方式**，除非存在接受`std::initializer_list`的构造函数，否则使用大括号构造对象等效于使用括号。

初始化和赋值操作的**差别是模糊的。**但是对于用户定义的类，区分初始化和赋值操作是很重要的，因为这会导致**不同**的函数调用。

```c++
//以下面的demo为例
#include <iostream>
using namespace std;
struct A
{
    ~A(){ cout << "~A()" << endl; }
    A()
    {
        cout << "A::A " << endl;
    }

    A(A& a)
    {
        cout << "A::A copy constructor" << endl;
    }

    A& operator=(A &a)
    {
        cout << "A::A operator=" << endl;
        return *this;
    }
};

int main() {
    A a1;			//初始化，对应无参构造
    A a2 = a1;		//初始化，对应拷贝构造
    a1 = a2;		//赋值重载运算，对应operator=操作
    return 0;
}
```

## 1.1大括号类内初始化

```c++
class A {
  ...
private:
  int x{ 0 };   // x的默认初始值为0
  int y = 0;    // 同上
  int z( 0 );   // 报错
}
```

大括号也可以用于类内成员的默认初始值，在C++11中，等号”=”也可以实现，但是圆括号 '( )' 则不可以

## 1.2大括号初始化不可拷贝对象

```c++
std::atomic<int> ai1{ 0 };  // 可以

std::atomic<int> ai2( 0 );  // 可以

std::atomic<int> ai3 = 0;   // 报错
```

**不可拷贝对象**(例如，`std::atomic`)可以用**大括号**和**圆括号**初始化，但不能用等号

## 1.3免疫C++中的最让人头痛的歧义

```c++
#include <iostream>
using namespace std;
struct A
{
    ~A(){ cout << "~A()" << endl; }
    A()
    {
        cout << "A::A " << endl;
    }
    
    A(int x)
    {
        cout << "A::A x = " << x << endl;
    }

    A(A& a)
    {
        cout << "A::A copy constructor" << endl;
    }

    A& operator=(A &a)
    {
        cout << "A::A operator=" << endl;
        return *this;
    }
};

int main() {
    A a1(10);  // 调用的A带参构造函数，这个没有歧义
    A a2();   // 最让人头痛的歧义，声明了一个名为a2，不接受任何参数，返回A类型的函数！
	A a3;     // 正确：a3是个默认初始化的对象
    return 0;
}
```

这里产生歧义的点，不知道a2为无参数返回为A类型的函数名，还是为A的默认初始化对象实例。

使用下面大括号的方式，可以避免歧义产生，**使用大括号包含参数是无法声明为函数的**。

```c++
A a4{};   // 无歧义
```



## 1.4迭代器的使用

```c++
#include<iostream>
#include<vector>
#include<map>
 
class Test
{
public:
    Test(std::string s, int val) {}
};
 
 
int main()
{
    int arr[] = {10, 20, 30, 40};  //拷贝初始化
    int brr[]{10, 20, 30, 40};    //直接初始化
 
    std::vector<int> vc1 = {10, 20, 30 ,40};
    std::vector<int> vc2{10, 20, 30, 40};
    std::map<std::string, int> m1 = { {"a", 1}, {"b", 2}, {"c", 3} };
 
    Test *pt = new Test{"test", 100};
    delete pt;
 
    return 0;
}
```

上面所举的例子中用到了{}对标准库容器进行初始化，而**标准库容器之所以能够支持初始化列表**，除了有编译器的支持外，**更直接的是这些容器存在以std::initializer_list为形参的构造函数**。

vector容器初始化

![il_1](D:\hugo\MyBlog\static\c++\base\initializer_list\il_1.png)

map容器初始化

![il_2](D:\hugo\MyBlog\static\c++\base\initializer_list\il_2.png)

## 1.5与std::initializer_list混淆

### 1.5.1构造中没有添加initializer_list

```c++
#include<iostream>
using namespace std;

class A {
public:
  A(int i, bool b){
      cout << "constructor int/bool" << endl;
  }
  A(int i, double d){
      cout << "constructor int/double" << endl;
  }

};

int main()
{
    // 调用第一个构造函数
    A a1(10, true);
    // 调用第一个构造函数，使用大括号构造对象等效于使用括号
    A a2{10, true};
    // 调用第二个构造函数
    A a3(10, 5.0);
    // 调用第二个构造函数，使用大括号构造对象等效于使用括号
    A a4{10, 5.0};
    return 0;
}
```

输出

```c++
constructor int/bool
constructor int/bool
constructor int/double
constructor int/double
```

### 1.5.2构造中添加initializer_list

如果**构造函数的形参带有**`std::initializer_list`，调用构造函数时**大括号**初始化语法会**强制使用**带`std::initializer_list`参数的重载构造函

```c++
#include<iostream>
using namespace std;

class A {
public:
  A(int i, bool b){
      cout << "constructor int/bool" << endl;
  }
  A(int i, double d){
      cout << "constructor int/double" << endl;
  }

  A(std::initializer_list<long double> il){
      cout << "constructor initializer_list size = " << il.size() << endl;
      //使用迭代器输出
      for (const long double* i = il.begin();  i != il.end(); ++i)
      {
            cout << *i << endl;
      }
  }
};

int main()
{
    // 调用第一个构造函数
    A a1(10, true);
    // 调用第三个构造函数，里面参数强制转化为long double类型
    A a2{10, true};
    // 调用第二个构造函数
    A a3(10, 5.0);
    // 调用第三个构造函数，里面参数强制转化为long double类型
    A a4{10, 5.0};
    return 0;
}
```

输出

```c++
constructor int/bool
constructor initializer_list size = 2
10
1
constructor int/double
constructor initializer_list size = 2
10
5
```

### 1.5.3initializer_list类的类型转化

更进一步的，编译器用带有`std::initializer_list`构造函数匹配大括号初始值，即使这个构造函数是无法调用的且另一个构造函数还是**参数精确匹配**的，编译器也会忽略精准匹配的构造函数。

```c++
#include<iostream>
using namespace std;

class A {
public:
    A(int i, bool b){
      cout << "constructor int/bool" << endl;
    }
    A(int i, double d){
      cout << "constructor int/double" << endl;
    }

    A(std::initializer_list<long double> il){
      cout << "constructor initializer_list size = " << il.size() << endl;
      for (const long double* i = il.begin();  i != il.end(); ++i)
      {
            cout << *i << endl;
      }
    }

    operator float() const   // 支持隐式转换为float类型
    {
      float f = 1.0f;
      return f;
    }

    A(const A& a)
    {
      cout << "A::A copy constructor" << endl;

    }
};

int main()
{
    // 调用第一个构造函数
    A a1(10, true);
    // 不使用拷贝构造，将a1隐式转化为float类型，在使用第三个构造
    A a2{a1};
    // 调用拷贝构造
    A a3(a1);
    return 0;
}
```

输出

```c++
constructor int/bool
constructor initializer_list size = 1
1
A::A copy constructor
```

### 1.5.4initializer_list窄化转换

根据1.5.2的内容，稍作修改把构造参数initializer_list中的类型long double变成bool类型

```c++
#include<iostream>
using namespace std;

class A {
public:
    A(int i, bool b){
        cout << "constructor int/bool" << endl;
    }
    A(int i, double d){
        cout << "constructor int/double" << endl;
    }

    A(std::initializer_list<bool> il){
        cout << "constructor initializer_list size = " << il.size() << endl;
        //使用迭代器输出
        for (auto* i = il.begin();  i != il.end(); ++i)
        {
            cout << *i << endl;
        }
    }
};

int main()
{
    // 调用第一个构造函数
    A a1(10, true);
    // 调用第三个构造函数，里面参数强制转化为bool类型，但是int类型转化bool会错误
    A a2{10, true};//无法编译通过
    // 调用第二个构造函数
    A a3(10, 5.0);
    // 调用第三个构造函数，里面参数强制转化为bool类型，但是int、double类型转化bool会错误
    A a4{10, 5.0};//无法编译通过
    return 0;
}
```

输出

```c++
D:/project/CPPProject0710/main.cpp:28:18: error: narrowing conversion of '10' from 'int' to 'bool' [-Wnarrowing]
   28 |     A a2{10, true};
      |                  ^
D:/project/CPPProject0710/main.cpp:32:17: error: narrowing conversion of '10' from 'int' to 'bool' [-Wnarrowing]
   32 |     A a4{10, 5.0};
      |                 ^
ninja: build stopped: subcommand failed.
```

### 1.5.5 无法转化为initializer_list

只有当大括号内的值**无法转换**为`std::initializer_list`元素的类型时，编译器才会使用正常的重载选择方法

根据1.5.2的内容，稍作修改把构造参数initializer_list中的类型long double变成string类型

```c++
#include<iostream>
using namespace std;

class A {
public:
    A(int i, bool b){
        cout << "constructor int/bool" << endl;
    }
    A(int i, double d){
        cout << "constructor int/double" << endl;
    }

    A(std::initializer_list<string> il){
        cout << "constructor initializer_list size = " << il.size() << endl;
        //使用迭代器输出
        for (auto* i = il.begin();  i != il.end(); ++i)
        {
            cout << *i << endl;
        }
    }
};

int main()
{
    // 调用第一个构造函数
    A a1(10, true);
    // 调用第三个构造函数，里面参数强制转化为string类型，类型无法转化匹配，调用第一个构造函数
    A a2{10, true};
    // 调用第二个构造函数
    A a3(10, 5.0);
    // 调用第三个构造函数，里面参数强制转化为string类型，类型无法转化匹配，调用第一个构造函数
    A a4{10, 5.0};
    return 0;
}
```

输出

```c++
constructor int/bool
constructor int/bool
constructor int/double
constructor int/double
```

### 1.5.6关于混淆点补充

在STL中，许多容器都会使用到`std::initializer_list`构造函数来初始化，比如vector和map等。`std::vector`就是一个**被它们直接影响的类**。`std::vector`中有一个可以**指定**容器的**大小**和容器内**元素的初始值**的不带`std::initializer_list`构造函数，但它也有一个可以指定容器中元素值的**带`std::initializer_list`函数**。

```c++
std::vector<int> v1(10, 20);   // 使用不带std::initializer_list的构造函数
                               // 创建10个元素的vector，每个元素的初始值为20

std::vector<int> v2{10, 20};   // 使用带std::initializer_list的构造函数
                               // 创建2个元素的vector，元素值为10和20
```

上述调用走的不是同一个构造函数，即含义完全不同

```c++
//第一个小括号构造吗，创建10个元素的vector，每个元素的初始值为20
template<typename _InputIterator,
typename = std::_RequireInputIter<_InputIterator>>
    vector(_InputIterator __first, _InputIterator __last,
           const allocator_type& __a = allocator_type())
    : _Base(__a)
    {
        _M_range_initialize(__first, __last,
                            std::__iterator_category(__first));
    }
//第二个大括号构造，创建2个元素的vector，元素值为10和20
vector(initializer_list<value_type> __l,
       const allocator_type& __a = allocator_type())
    : _Base(__a)
    {
        _M_range_initialize(__l.begin(), __l.end(),
                            random_access_iterator_tag());
    }
```

另外使用一个大括号调用initializer_list无参构造的方式

> 默认使用大括号的无参构造
>
> 如果你想要用一个空的`std::initializer_list`参数来调用带`std::initializer_list`构造函数，那么你需要**把大括号作为参数**，即把空的大括号放在圆括号内或者大括号内，即initializer_list参数列表可以为空。
>
> ```c++
> #include<iostream>
> using namespace std;
> 
> class A {
> public:
>     A(){
>         cout << "constructor" << endl;
>     }
> 
> 
>     A(std::initializer_list<int> il){
>         cout << "constructor initializer_list size = " << il.size() << endl;
>         //使用迭代器输出
>         for (auto* i = il.begin();  i != il.end(); ++i)
>         {
>             cout << *i << endl;
>         }
>     }
> };
> 
> int main()
> {
>     //调用默认构造
>     A a1;
>     //调用默认构造
>     A a2{};
>     //调用initializer_list构造
>     A a3({});
>     //调用initializer_list构造,里面又是一个参数，为0
>     A a4{{}};
>     return 0;
> }
> ```
>
> 输出结果
>
> ```c++
> constructor
> constructor
> constructor initializer_list size = 0
> constructor initializer_list size = 1
> 0
> ```
>
> 可以发现有时候{}和()可以是同样的效果（上述a1和a2），又可以是不一样的效果（上述a3和a4）。

# 2最后来一个demo

```c++
#include<iostream>
#include<thread>
using namespace std;

struct A
{
    ~A(){ cout << "~A()" << endl; }
    A()
    {
        cout << "A::A " << endl;
    }
	//这里必须写成右值引用的形式，std::thread构造传入的值为右值，会进行一次拷贝构造
    A(const A& a)
    {
        cout << "A::A copy constructor" << endl;

    }

    A& operator=(A &a)
    {
        cout << "A::A operator=" << endl;
        return *this;
    }
    //仿函数
    void operator()()
    {
        cout << "A::operator() " << endl;
        //return true;
    }
};

void f1(int n)
{
    cout << "Thread f1 " << n << " executing\n";
}

int main()
{
    thread t1{};		//空的默认构造
    thread t2;			//空的默认构造，不能写成thread t2();会被认为是函数定义
    thread t3{f1, 1};	//使用函数指针，最终调用函数指针f1()回调到对应函数
    t3.join();
    thread t4(f1, 2);	//使用函数指针，最终调用函数指针f1()回调到对应函数
    t4.join();
    
   	thread t5{A()};		//使用仿函数，A()是匿名对象，传入thread中会调用拷贝构造
    t5.join();			//这里会释放上面的匿名对象，然后在传入的拷贝对象调用 "对象()"，这就是仿函数
    thread t6{A{}};		//同t5一致
    t6.join();
    A a;				//与上述不一致的是，这里调用的a是直接的对象
    thread t7(a);		//将直接对象传入t7中，传入thread中会调用拷贝构造
    t7.join();			//然后在传入的拷贝对象调用 "对象()"，这就是仿函数，因为a是局部变量，所以不会马上退出
    thread t8((A()));	//这里跟t5一致
    t8.join();
    thread t9(A{});		//这里跟t5一致
    t9.join();
    //parentheses were disambiguated as a function declaration [-Wvexing-parse]
    /*thread t10(A());	//t9会出错，需要变成t8那样才行，这里最外层使用括号，会有歧义
    t10.join();*/		//thread t9(A());会被认为是函数定义
    return 0;
}
```

> 关于std::thread
>
> ```c++
> thread(_Callable&& __f, _Args&&... __args);
> ```
>
> 当我们做并发工作时，需要使用std::thread()来用于创建、管理和同步线程。首先，我们要知道std::thread()的参数列表构成
>
> 第一个参数：可作为线程函数的三种形式
>
> 1. 函数指针
>
> 2. 函数对象（operator()()重载函数，也叫仿函数）
>
> 3. Lambda匿名表达式
>
> 其他参数：相当于可变参数列表，根据调用函数的参数列表来传入实参

输出结果，这里一个一个输出

```c++
//t3输出
Thread f1 1 executing
//t4输出
Thread f1 2 executing
//t5输出
A::A
A::A copy constructor
~A()
A::operator()
~A()
//t6输出
A::A
A::A copy constructor
~A()
A::operator()
~A()
//t7输出
A::A
A::A copy constructor
A::operator()
~A()
~A()
//t8输出
A::A
A::A copy constructor
~A()
A::operator()
~A()
//t9输出
A::A
A::A copy constructor
~A()
A::operator()
~A()
```



# 参考文献

[[1]  emin. C++创建对象时区分圆括号( )和大括号{ }, 2020.](https://zhuanlan.zhihu.com/p/268894227)

[[2] 留恋单行路,现代C++之std::initializer_list的特性分析,2022](https://blog.csdn.net/a574780196/article/details/122493579).

[[3] zhihu,c++11小括号和大括号初始化有什么区别？,2021](https://www.zhihu.com/question/497428726/answer/2213203090).