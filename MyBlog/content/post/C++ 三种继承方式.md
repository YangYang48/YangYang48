---
title: "C++ 三种继承方式"
date: 2022-09-12
thumbnailImagePosition: left
thumbnailImage: c++/base/base5_thumb.jpg
coverImage: c++/base/base5_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 三种继承方式
- 2022
- September
tags:
- c++
- c++基础
- Android
- 源码
showSocial: false
---

Android源码中有一些经常会遇到的c++基础的内容，笔者对C++ 三种继承方式进行简单熟悉。

<!--more-->
# C++ 三种继承方式

| 继承方式                           | public                | protected             | private               |
| ---------------------------------- | --------------------- | --------------------- | --------------------- |
| 派生类实例化对象访问基类成员       | public成员            | 不能访问              | 不能访问              |
| 派生类的成员函数访问基类成员       | public和protected成员 | public和protected成员 | public和protected成员 |
| 子类的引用（或指针）直接转换为父类 | 可以                  | 可以                  | 不可以                |



# 1公有继承

公有继承时，对基类的公有成员和保护成员的访问属性不变，派生类的新增成员可以访问基类的公有成员和保护成员，但是访问不了基类的私有成员。派生类的对象只能访问派生类的公有成员（包括继承的公有成员），访问不了保护成员和私有成员。

```c++
#include <iostream>
using namespace std;

class Base         
{
public: 
    Base(int nId) {mId = nId;}
    int GetId() {mId++;cout<< mId<<endl;return mId;}
protected:
    int GetNum() {cout<< 0 <<endl;return 0;}
private: 
    int mId; 
};

class Child : public Base
{
public:
    Child() : Base(7) {;}
    int GetCId() {return GetId();}    //新增成员可以访问公有成员
    int GetCNum() {return GetNum();}  //新增成员可以访问保护成员
                                      //无法访问基类的私有成员
protected:
    int y;
private:
    int x;
};

int main() 
{ 
    Base* base = new Child();//ok
    Child child;
    child.GetId();        //派生类的对象可以访问派生类继承下来的公有成员
    //child.GetNum();     //无法访问继承下来的保护成员GetNum()
    child.GetCId();   
    child.GetCNum();      //派生类对象可以访问派生类的公有成员
    //child.x;
    //child.y;            //无法访问派生类的保护成员y和私有成员x
    return 0;
}
```

# 2保护继承

保护继承中，基类的公有成员和保护成员被派生类继承后变成保护成员，派生类的新增成员可以访问基类的公有成员和保护成员，但是访问不了基类的私有成员。派生类的对象不能访问派生类继承基类的公有成员，保护成员和私有成员。

```c++
class Child : protected Base
{
public:
    Child() : Base(7) {;}
    int GetCId() {return GetId();}   //可以访问基类的公有成员和保护成员
    int GetCNum() {return GetNum();}
protected:
    int y;
private:
    int x;
};

int main() 
{ 
    Base* base = new Child();// not ok
    Child child;
    //child.GetId();//派生类对象访问不了继承的公有成员，因为此时保护继承时GetId()已经为          protected类型
    //child.GetNum(); //这个也访问不了
    child.GetCId();
    child.GetCNum();
    return 0;
}
```



# 3私有继承

私有继承时，基类的公有成员和保护成员都被派生类继承下来之后变成私有成员，派生类的新增成员可以访问基类的公有成员和保护成员，但是访问不了基类的私有成员。派生类的对象不能访问派生类继承基类的公有成员，保护成员和私有成员。

```c++
class Child : private Base
{
public:
    Child() : Base(7) {;}
    int GetCId() {return GetId();}   //可以访问基类的公有成员和保护成员
    int GetCNum() {return GetNum();}
protected:
    int y;
private:
    int x;
};

int main() 
{ 
    Base* base = new Child();// not ok
    Child child;
    //child.GetId();//派生类对象访问不了继承的公有成员，因为此时私有继承时GetId()已经为          private类型
    //child.GetNum(); //派生类对象访问不了继承的保护成员，而且此时私有继承时GetNum()已经为          private类型
    child.GetCId();
    child.GetCNum();
    return 0;
}
```

