C++智能指针

C++11推出智能指针，是为了媲美java的jvm，完全不用管对象的回收问题。

需要补充

![image-20220113224553780](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220113224553780.png)

## 智能指针初探

```c++
#include <iostream>
#include <memory> //智能指针头文件
using namespace std;

class Person{
public:
    ~Person(){
        cout << "析构函数" << endl;
    }
};

int main(){
    Person* person1 = new Person();//堆区开辟空间 
    //delete person1;
    
    //如果堆区没有析构，可以使用智能指针用它协助析构
    //栈区开辟的sharedptr会在加入person1，内部+1，main函数结束就会弹栈，内部-1，==0，调用person1的析构函数
    shared_ptr<Person> sharedPtr1(person1); //栈区开辟的sharedptr1
    return 0;
}
```



## 智能指针的各种使用。

智能指针有循环依赖的问题，不要用的太复杂

下面例子，通过运算符重载，重载中会再次加一，使得main弹栈不能释放per1和per2指针

```c++
//这个demo是由循环依赖的问题，不要用
#include <iostream>
#include <memory> //智能指针头文件
using namespace std;

class Person2;//Person2先声明

class Person1{
public:
    shared_ptr<Person2> person2;
    ~Person1(){
        cout << "析构函数1" << endl;
    }
};

class Person2{
public:
    shared_ptr<Person1> person1;
    ~Person2(){
        cout << "析构函数2" << endl;
    }
};

int main(){
    Person1* per1 = new Person1();
    Person2* per2 = new Person2();
    
    shared_ptr<Person1> sharedptr1(per1);
    shared_ptr<Person2> sharedptr2(per2);
    
    cout << "before sharedptr1的引用计算是：" << sharedptr1.use_count() << endl;//1
    cout << "before sharedptr2的引用计算是：" << sharedptr2.use_count() << endl;//1
    //这里是运算符重载，重载中会再次加一
    per1->person2 = sharedptr2;
    per2->person1 = sharedptr1;
    //如果计算是2，弹栈减一不为0，就不会析构per1和per2的堆空间 
    cout << "before sharedptr1的引用计算是：" << sharedptr1.use_count() << endl;//2
    cout << "before sharedptr2的引用计算是：" << sharedptr2.use_count() << endl;//2
    return 0;
}
```

解决上述问题的话，可以使用`weak_ptr`

`unique_ptr`设计足够简单，没那么多功能，属于独占式智能指针。但他不是四大智能指针的范畴。

```c++
#include <iostream>
#include <memory>
using namespace std;
class Person{};

int main(){
    Person* person1 = new Person();
    Person* person2 = new Person();
    unique_ptr<Person> unique1(person1);
    //下面方式不允许
    //unique_ptr<Person> unique2 = unique1; //unique不允许，因为它是独占的
    return 0;
}
```



## 手写智能指针

```c++
//customptr.h
#ifndef CPP_CUSTOMPTR_H
#define CPP_CUSTOMPTR_H

#pragma once
#include <iostream>
using namespace std;

template<typename T>
class Ptr{
private:
    T* object;//用于智能指针指向管理的对象
    int* count;//引用计数
public:
    //构造函数
    Ptr(){
        count = new int(1);
        object = 0;//没有传参对象，默认为0
    }
   //有参构造函数
    Ptr(T* t) : object(t){
        count = new int(1);
    }
    //析构函数
    ~Ptr(){
        //引用计数减1，最终为0释放泛型object对象
        if(0 == --(*count)){
            if(object){
                delete object;
            }
            delete count;
            object = 0;
            count = 0;
        }
    }
    
    //拷贝构造函数
    Ptr(const Ptr<T>& p){
        cout << "拷贝构造函数" << endl;
        ++(*p.count);
        
        object = p.obejct;
        count = p.count;
    }
    
    //自定义等号符重载
    Ptr<T>& operator = (const Ptr<T>& p){
        cout << "等号符重载" << endl;
        
        ++(*p.count);
        //这个比较绕，主要是有两种情况
        //1.一开始直接空构造产生的count和object，回收这两个指针
        //2.有参构造制后有调用运算符重载，那么在重载copy前，需要把原先count和object回收
        if (--(*count) == 0){
            if(object){
                delete object;
            }
            delete count;
        }

        object = p.obejct;
        count = p.count;
        
        return *this;
    }
    //引用计数
    int use_count(){
        return *this->count;
    }
};
```

手写的智能指针和std的智能指针比较

```c++
#include "customptr.h"

class Student{
public:
    ~Student(){
        cout << "析构函数" << endl;
    }
};

//用std的智能指针
void action(){
    Student* student1 = new Student();
    Student* student2 = new Student();
    
    //todo 1.有参构造
    //shared_ptr<Student> sharedptr1(student1);
    //shared_ptr<Student> sharedptr2(student2);
    
    //todo 2.拷贝构造
    //shared_ptr<Student> sharedptr1(student1);
    //shared_ptr<Student> sharedptr2 = sharedptr1;
    
    //通用打印
    cout << "智能指针内置sharedptr1 = " << sharedptr1.use_count() << endl;
    cout << "智能指针内置sharedptr2 = " << sharedptr2.use_count() << endl;
}

//用自定义的智能指针
void action2(){
    Student* student1 = new Student();
    Student* student2 = new Student();
    
    //todo 1.有参构造
    //Ptr<Student> ptr1(student1);
    //Ptr<Student> ptr2(student2);
    
    //todo 2.拷贝构造函数
    //Ptr<Student> ptr1(student1);
    //Ptr<Student> ptr2 = ptr1;
    
    //todo 3.运算符重载1
    //Ptr<Student> ptr1(student1);
    //Ptr<Student> ptr2;//无参构造
    //ptr2 = ptr1;//运算符重载
    
    //todo 3.运算符重载2
    Ptr<Student> ptr1(student1);
    Ptr<Student> ptr2(student2);//有参构造
    //在运算符重载前，需要释放student2指针 
    ptr2 = ptr1;//运算符重载
    
    //通用打印
    cout << "智能指针内置ptr1 = " << ptr1.use_count() << endl;
    cout << "智能指针内置ptr2 = " << ptr2.use_count() << endl;
}

int main(){
    cout << "下面是C++内置智能指针" << endl;
    action();
    cout << "下面是自定义智能指针" << endl;
    action2();
    return 0;
}
```

打印会变成2，是由于两个智能指针指向同一个变量，最终两个指针在栈区回收调用两次析构才能回收person1指针。

![image-20220115134850501](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220115134850501.png)

## 四种类型转换

`const_cast`const修饰都可以去转换

```c++
//转成非常量指针
#include <iostream>

using namespace std;

class Person{
public:
    string name = "default";
};
int main(){
    const Person* p1 = new Person();
    //p1->name = "Derry";//常量指针不能修改指向的值
    Person* p2 = const_cast<Person*>(p1);
    p2->name = "Derry";
    cout << p1->name << endl;
    return 0;
}
```

![image-20220113231828437](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220113231828437.png)

`static_cast`指针相关的操作，可用这个类型转化，包括子父类

`static_cast`转换看左边，在编译期就已经决定

```c++
#include <iostream>
using namespace std;

class Fu{
public:
    void show(){
        cout << "父类show" << endl;
    }
};

class Zi : public Fu{
public:
    void show(){
        cout << "子类show" << endl;
    }
};

int main(){
    int n = 88;
    void* pVoid = &n;
    int* num = static_cast<int*>(pVoid);
    cout << *num << endl;
    //===========================
    Fu* fu = new fu();
    fu->show();
    
    //static_cast转换看左边
    Zi* zi = static_cast<Zi*>(fu);
    zi->show();
    
    delete fu;//回收原则，一定是谁new了，我就delete谁
    
    return 0;
}
```

3.`dynamic_cast`子类父类多态，在运行期转换

**动态转换，必须定义虚函数**

动态转化有返回值，返回null转化失败

```c++
#include <iostream>
using namespace std;

class Fu{
public:
    virtual void show(){
        cout << "父类show" << endl;
    }
};

class Zi : public Fu{
public:
    virtual void show(){
        cout << "子类show" << endl;
    }
};

int main(){
    //下面一定会失败，因为Fu类已经有了有个Fu类的对象
    /*Fu* fu = new Fu();
    Zi* zi = dynamic_cast<Zi*>(fu);
    
    if (zi){
        cout << "转换成功" << endl;
        zi->show();
    } else
    {
        cout << "转换失败" << endl;
    }*/
    
    //已成定局，是子类
    Fu* fu = new Zi();
    Zi* zi = dynamic_cast<Zi*>(fu);
    
    if (zi){
        cout << "转换成功" << endl;
        zi->show();
    } else
    {
        cout << "转换失败" << endl;
    }
    
    return 0;
}
```

4.`reinterpret_cast`强制类型转化，强制转化比`static_cast`要强大，除了`static_cast`能够做的事情，同时还增加新功能

```c++
#include <iostream>
using namespace std;

class DerryPlayer{
public:
    void show(){
        cout << "DerryPlayer::show()" << endl;
    }
};

int main(){
    DerryPlayer* derryPlayer = new DerryPlayer();
    //把对象指针变成数值,static_cast这个强制转化不行
    //long player = static_cast<long>(derryPlayer);
    long player = reinterpret_cast<long>(derryPlayer);
    
    //把地址在变成对象指针,系统源码大量使用这种写法
    DerryPlayer* derry2 = reinterpret_cast<DerryPlayer*>(player);
    derry2->show();
    return 0;
}
```

这里每次使用一个功能，需要返回一个long类型的地址，方便后续通过这个地址来执行功能。避免重复创建对象，通过long类型地址强制转化，保证对象统一化。

![image-20220115145427885](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220115145427885.png)

## nullptr

```c++
#include <iostream>
using namespace std;

void show(int* i){
    cout << " show(int* i) " << endl;
}

void show(int i){
    cout << " show(int i) " << endl;
}

int main(){
    show(9);
    //是表示空指针,nullptr是一个C++关键字，从C++11开始引入的
    show(nullptr);
    //不能写成下面的方式，会产生歧义，在c++中代表0，而非(void*)0
    //show(NULL);
    return 0;
}
```

![image-20220115150306644](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220115150306644.png)

![image-20220115151024742](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220115151024742.png)

可见在c++中，NULL为0，在C中NULL为(void*)0

具体为什么引入nullptr，详见[关于C/C++中的NULL](https://www.cnblogs.com/yutongqing/p/6508327.html)

