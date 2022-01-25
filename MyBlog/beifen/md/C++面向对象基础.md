C++面向对象

## 1.C语言和C++语言的区别

C++语言面向对象，+标准特性

C语言面向过程，函数+结构体

以后85%以上都是用C++去写功能

## 2.常量之C常量和C++常量

C语言的常量，其实是一个假常量，可以修改const修饰的变量，而C++不能

![image-20211222215359421](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222215359421.png)

## 3.引用的原理的常量引用

常量引用不允许修改引用对象，**变成只读的**了

![image-20211222220659383](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222220659383.png)

## 4.函数重载与默认形参，无形参名的特殊写法

C语言不支持函数重载

![image-20211222220842699](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222220842699.png)

## 5.C++函数重载

这里的mode可先不写，可用于未来在去写入，相当于**前期抽象，后续扩展**

![image-20211222224137249](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222224137249.png)

c++面向对象

![image-20211222224012893](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222224012893.png)

## 6.c++面向对象

堆空间和栈空间的对象

![image-20211222225014028](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222225014028.png)

## 7.C++对象释放

![image-20211222225242350](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222225242350.png)

编译注意点，头文件需要对应的实现文件，即h和cpp要一一对应

![image-20211222225721983](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211222225721983.png)

## 7-1命名空间

```c++
using namespace std;
```

这里 命名空间特性类似Java语言的内部类

```c++
cout << "" << endl;
```

这里的<<为操作符重载

 

命名空间可以使用为全局或者局部

![image-20211223215215985](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223215215985.png)

使用方式 1用命名空间名::的形式（推荐，防止出现多个命名空间同名的方法或者变量），或者是2加上using namespace 命名空间名后，直接使用;

http://c.biancheng.net/view/2192.html



## 8.C++构造函数

C++的一些源码会经常出现:后面增加初始化内容，实际上跟this->name = name同样含义

![image-20211223220250017](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223220250017.png)



和java一样，让其在构造函数中互相调用

![image-20211223220940146](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223220940146.png)

## 9.C++析构函数

C++中使用C的内存分配实际上并不会调用C++的构造函数和析构函数。

![image-20211223221413469](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223221413469.png)

通常来说，析构函数做一个堆区释放工作

![image-20211223221929770](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223221929770.png)

在java语言中，设置stu为null才能够去回收stu

![image-20211223222534118](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223222534118.png)

![image-20211223222616660](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223222616660.png)

kotlin回收也需要如此

![image-20211223223036252](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223223036252.png)

![image-20211223223143088](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223223143088.png)

kotlin中没有finalize，默认继承any，这个时候就需要新建一个JavaHelp类，用于继承，最终会调用到close。

## 10.纠结new/delete



## 11.拷贝构造函数

![image-20211223224115096](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223224115096.png)

然而指针指向，不会调用拷贝构造函数

![image-20211223225027304](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223225027304.png)

![image-20211223224943542](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223224943542.png)

## 常量指针和指针常量

通常判断用const靠近谁，谁就是常量的形式区分，往const左边去查看

![image-20211223225524059](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211223225524059.png)

## 12浅拷贝和深拷贝

1.拷贝构造函数与析构函数执行流程分析
2.拷贝构造函数与析构函数原理细节图研究
3.拷贝构造函数配合析构函数制作奔溃
4.深拷贝解决奔溃，并分析原理  

C++默认拷贝构造函数是浅拷贝，需要自定义拷贝构造函数

浅拷贝

![image-20211227222659331](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211227222659331.png)



这里拷贝构造函数使得新地址和旧地址的name成员变量拥有一个堆的地址。如果其中一个释放，另一个调用，则会崩溃

![image-20211227225738104](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211227225738104.png)

深拷贝

两个name的堆区地址不一致，又重新创建了一个

![image-20211227230722959](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211227230722959.png)

![image-20211227230654867](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211227230654867.png)

## 13C++语言之 this 原理与友元函数友元类
1.C++可变参数。

```c++
//可变参数使用需要头文件
#include <stdarg.h>
//定义可变参数
void sum(int count, ...)
{
    va_list vp;//可变参数的动作
    //参数1：可变参数的动作
    //参数2：作用1：内部需要存储地址用的参考值，如果没有第二个参数，内部无法处理处方参数信息 
    //参数2：作用2：用于循环遍历
    va_start(vp, count);  
    //取出可变参数的一个值
    int number = va_arg(vp, int);
    cout << number << endl;
    number = va_arg(vp, int);
    cout << number << endl;
    number = va_arg(vp, int);
    cout << number << endl;
    //关闭阶段,必须关闭 
    va_end(vp);
}

//或者循环的形式
void sum(int count, ...)
{
    va_list vp;
    va_start(vp, count);
    
    for(int i = 0; i < count; i++)
    {
        int r = va_arg(vp, int);
        count << r << endl;
    }
    va_end(vp);
}
```



```c++
//调用可变参数
int main()
{
    sum(3, 6, 7, 8);
    
    return 0;
}
```

2.C++static 关键字。

在类里面定义的static必须先声明，然后再类外部初始化赋值

```c++
/*
**
*/
class Dog{
public:
    char* info;
    int age;
    //1.先声明
    static int id;
    
    //其中静态函数不能调用非静态函数
    static void update(){
        id += 100;
    }
    
    void update2(){
        id = 13;
    }
};
//2.在初始化赋值
int Dog::id = 9;
```



```c++
int main(){
    //如果没有在初始化赋值，下面的两个函数都是error
    Dog dog;
    dog.update2();//普通函数
    Dog::update();//静态函数
}
```

3.C++对象中，为什么需要 this。

this指针的作用

![image-20211228223920730](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211228223920730.png)



在类使用的使用调用了当前的构造函数，而构造函数生成的对象的地址即为this指针

默认的构造函数，栈区开辟空间暴露地址==this 指针

静态区域没有地址是**共享区域**，所有的值都是一样。

![image-20211228224047030](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211228224047030.png)

4.const 修饰函数的 this 意义何在。

const修饰函数的后面

![image-20211228224223234](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211228224223234.png)

```c++
class Worker{
public:
    char* name;
    int age;
    //默认持有隐式的this（Worker* const this）
    //指针常量：代表指针地址不能被修改，但是指向指针地址的值可以修改
    void change1(){
        this->age = 78;
    }
    
    //原理：修改隐式代码：函数后面加const，
    //等价const Worker* const常量指针常量（地址不能修改，地址对应的值不能修改）
    //声明一个函数用 const 关键字来说明这个函数是一个只读函数,this指针里面的变量不能修改
    void changeAction() const{
        //当前函数里面worker对象只读的，一般用于打印
    }
    
    //函数参数为const，限制传入的参数只读，不可以修改它的值
    void changeAction(const A& a, const B& b) const{
        
    }
};
```



5.友元函数与友元类实战运用。

  不能修改私有的age，除非定义一个友元函数

```c++
class Person{
private:    
    int age = 0;
public:
    Person(int age):age(age){}
    
    int getAge(){
        return this->age;
    }
    
    //内部声明一个友元函数
    friend void updateAge(Person* person, int age);
};

//如果定义了友元声明函数，那么友元函数的实现，可以访问所有私有成员
void updateAge(Person* person, int age){
    //person->age = age; //不是友元函数，不能修改私有的age
    //如果定义了友元，可以拿到所有的私有成员
    person->age = age;
}

int main(){
    Person person = Person(9);
    updateAge(&person, 88);
    
    return 0;
}
```



静态函数，友元函数，普通函数，构造函数，析构函数，拷贝构造函数，区别

![image-20211228231324320](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211228231324320.png)

友元类的故事

> ImageView私有成员，可以通过Class来访问，Class操作的是native代码
>
> 可以下载JDK native代码研究

定义一个友元类，可以通过**友元类**访问私有成员，友元类的类内部访问私有成员。

![image-20211228231929541](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211228231929541.png)



## 14C++重载和继承

1.类外运算符重载。

类外部运算重载，对运算符+重载

![image-20211230225557270](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211230225557270.png)

![image-20211230225510130](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211230225510130.png)

2.类里运算符重载。

1.类里面重载，默认持有this指针对象

![image-20211230225818474](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211230225818474.png)



2.系统代码是这样写的

![image-20211230230155374](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211230230155374.png)

总结：一般都定义在类的内部，如果定义到外部，使用const和&常量引用，就不能够拿到类的私有成员。

&不加会进行拷贝构造，加上&提升性能。

**注：这里不能使用getX()和getY()的主要原因，系统认为你会在函数内部修改**



3.++运算重载（分成前置和后置）

默认为前置

```c++
//++对象
void operator ++ (){
    this->x = this->x + 1;
    this->y = this->y + 1;
}


//对象++
void operator ++ (int){
    this->x = this->x + 1;
    this->y = this->y + 1;
}
```

3.括号运算符。

直接输出对象，需要对符号进行重载，涉及到规则。

**输出重载**

```c++
//直接输出一个对象会error，必须对其重载
cout << derry1;
```

这里的友元函数写在了类的内部，也可以写在外部，跟上面的友元函数类似。

[C++ 友元函数的使用&重载“输入输出”运算符](https://blog.csdn.net/qq_33195791/article/details/82053149)

```c++
//对象对内进行友元类重载
//这里的void相当于只能够一次输出重载
//这里面的_START就是系统的cout
friend void operator << (ostream& _START, const Derry& derry){
    _START << "单个输出" << derry.x << " ! " << derry.y << "end" << endl;
}

//可以多次重载
//这个重载符号不能相当，用另一个符号表示
friend ostream& operator >> (ostream& _START, const Derry& derry){
    _START << "多个输出" << derry.x << " ! " << derry.y << "end" << endl;
}
```

**外部实现友元类重载**

```c++
//定义友元函数
friend ostream& operator << (ostream& _START, const Derry& derry);
//实现友元函数
ostream& operator<<(ostream& _START, const Derry& derry){
     _START << "多个输出" << derry.x << " ! " << derry.y << "end" << endl;
}
```



```c++
//单次重载
cout << derry;
//多次重载
cout >> derry >> derry >> derry >> derry;
```



**输入重载**

```c++
//这里不能增加const关键字，因为需要修改derry对象的成员变量
friend istream& operator >> (istream& _START, Derry& derry){
    _START >> derry.x >> derry.y;
}
```



```c++
Derry derry;
//开始输入重载
cin >> derry;
cout << "Derry.x = " << derry.x << endl;
cout << "Derry.y = " << derry.y << endl;
```



这里加friend关键字主要是

cout 是 ostream 类的对象，cin 是 istream 类的对象，要想达到这个目标，就必须以全局函数（友元函数）的形式重载`<<`和`>>`。

运算符重载函数中用到了 Derry类的 private 成员变量，必须在 Derry类中将该函数声明为友元函数。



括号运算符

a[i] 相当于系统重载了 *(a+i)

```c++
class ArrayClass{
private:
    //c++默认size为 -6464654
    int size = 0;//大小必须赋值
    //数组存放int的很多值等价于int arrayValue[];
    int* arrayValue; 
    
public:
    void set(int index, int value){
        arrayValue[index] = value;
        size += 1;
    }
    
    int getSize(){
        return this->size;
    }
    
    int operator[](int index){
        return this->arrayValue[index];//系统
    }
}; 

void printArryClass(ArrayClass arrayClass){
    for(int i = 0; i < arrayClass.getSize(); i++){
        cout << arrayClass[i] <<endl;//这个是上面类里面重载的[]符号
    }
}
```

真实调用

```c++
ArrayClass arrayclass;
arrayclass.set(0, 100);
arrayclass.set(0, 200);
arrayclass.set(0, 300);
arrayclass.set(0, 400);

printArryClass(arrayclass);
```

4.C++对象继承。

**一般继承**

C++默认，private继承

另外，C++构造函数可以定义到private，但定义成private会导致此类不能直接被**外部实例化**

```c++
class Person{
private:
    char* name;
    int age;
public:
    Person(char* name, int age) : name(name){
        this->age = age;
        cout << "Person 构造函数" << endl;
    }
    
    void print(){
        cout << "name is = " << this->name << "age is = " << this->age << endl;
    }
};
```



```c++
//没有指定哪种继承，那么就为私有继承，不可直接实例化访问基类变量
//私有继承：在子类里面可以访问父类的成员，再类外不行
class Student : public Person{
public:
    Student(char* name, int age) : Person(name, age){
        cout<< "Student 构造函数" <<endl;
    }
    
    void test(){
        print();
    }
};
```



```
Student stu("xxx", 123);
stu.name = "李四";//默认私有继承是无法通过子类实例化对象访问基类变量的

```



多继承

出现二义性，子类实例化对象使用多个父类的同名函数，会产生歧义

```c++
class BaseActivcity1{
public:
    show(){
        cout << "BaseActivcity1::show()" << endl;
    }
};

class BaseActivcity2{
public:
    show(){
        cout << "BaseActivcity2::show()" << endl;
    }
};

class BaseActivcity3{
public:
    show(){
        cout << "BaseActivcity3::show()" << endl;
    }
};

class MainActivity : public BaseActivcity1, public BaseActivcity2, public BaseActivcity3{
public:
    myshow(){
        cout << "MainActivity::myshow()" << endl;
    }
};
```

二义性解决方法

```c++
MainActivity mainAtivity1;
//解决方法之一，明确指定父类,限定show是哪个父类的show
mainAtivity1.BaseActivity1::show();
mainAtivity1.BaseActivity2::show();
mainAtivity1.BaseActivity3::show();
//解决方法2：子类重写父类的show,让其不调用子的show
mainAtivity1.show();
```



二义性，继承如下图所示，成菱形状，子类访问祖父类会出现二义性

![image-20220103115755865](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220103115755865.png)

```c++
class Object{
public:
    int num;
};

class Father1 : public Object{
    
};

class Father2 : public Object{
    
};

class Son : public Father1, public Father2 {
    
};
```

解决方法：

```c++
Son son;
//解决方法1：明确指定哪个父类
son.Father1::number = 1000;
son.Father2::number = 1000;
//解决方法2：子类中定义同名函数覆盖父类相关成员
son.number = 1000;
//解决方法3：虚基类，修改两个父类的继承

```

//解决方法3：虚基类，修改两个父类的继承

修改如下

```c++
class Object{
public:
    int num;
};

class Father1 : virtual public Object{
    
};

class Father2 : virtual public Object{
    
};

class Son : public Father1, public Father2 {
    
};
```

用虚基类可以随便调用

```c++
Object obj;
Father1 fa1;
Father2 fa2;
Son son;

obj.number = 100;
fa1.number = 200;
fa2.number = 300;
son.number = 400;
```

## 15C++二义性,多态,纯虚函数,模版函数  

1.源码中属性初始化的方式。

如果类里面有对象成员，**需要按照源码方式初始化**，不能通过this->初始化

```c++
class Person{
private:
    string name;
    int age;
public:
    Person(string name, int age) : name(name), age(age){}
};

class Course{
private:
    string courseName;
public:
    Course(String name) : courseName(name){}
};

//如果定义的对象成员，必须这样初始化（构造函数的后面：对象成员（内容）），我们用第三种方式
class Student : public Person{
private:
    Course course;//对象成员
public:
    Student(string name, int age, Course course1, string nameInfo)
    :Person(name, age),
    //第二种方式，编译阶段认可，拷贝构造
    //course(course1)
    //第三种方式，构造初始化    
    course(nameInfo)    
    {
        this->course = course1;//这种方式是对象=对象，编译阶段不认可，无法检测到真的初始化
    }
}; 
```



2.虚继承， 二义性。

详见上述

3.多态（虚函数） 

**动态多态**

C++默认关闭多态，用虚函数，需要在基类上给函数增加virtual关键字

```c++
class BaseActivity{
public:
    virtual void onStart(){
        cout << "BaseActivity" << endl;
    }
};

class HomeActivity : public BaseActivity{
public:
    //重写父类的函数
    void onStart(){
        cout << "HomeActivity" << endl;
    }
};

class LoginActivity : public BaseActivity{
public:
    void onStart(){
        cout << "LoginActivity" << endl;
    }
};

//实现多态的注入
void startActivity(BaseActivity* baseactivity){
    baseactivity->onStart();
}
```



```c++

HomeActivity* homeactivity = new HomeActivity();
LoginActivity* loginactivity = new loginactivity();

//默认注入会只打印基类的打印，不会打印父类
startActivity(homeactivity);
startActivity(loginactivity);
```



> 多态：父类的引用指向子类的对象，同一方法可以有不同实现。重写（动态多态）和重载（静态多态）
>
> 程序在**运行期间**才能确定调用哪个类的函数（动态多态）。
>
> **编译期**已经决定调用哪个函数，这个属于静态多态。



**C++虚函数表**

关于私有虚函数或者保护虚函数，依然能够通过访问虚函数表实现多态

```c++
//在基类虚函数修改成非共有类，可以是private也可以是protected
//最终访问通过虚函数表访问
class BaseActivity{
private:
    virtual void onStart(){
        cout << "BaseActivity" << endl;
    }
};

class HomeActivity : public BaseActivity{
public:
    //重写父类的函数
    void onStart(){
        cout << "HomeActivity" << endl;
    }
};

class LoginActivity : public BaseActivity{
public:
    void onStart(){
        cout << "LoginActivity" << endl;
    }
};

typedef void(*Fun)(void);

BaseActivity* d = new LoginActivity();
Fun pFun = (Fun)*((int*)*(int*)(d)+0);
pFun();
```



[C++虚函数表解析](https://blog.csdn.net/wang37921/article/details/5599655)

4.静态多态。

重载就是静态多态

```c++
void add(int number1, int number2)
{
    cout << number1 + number2 << endl;
}

void add(float number1, float number2)
{
    cout << number1 + number2 << endl;
}

void add(double number1, double number2)
{
    cout << number1 + number2 << endl;
}
```



```c++
add(100, 100);
add(1.9f, 2.5f);
add(545.2, 125.9);
```

5.纯虚函数（Java 版抽象类）



```c++
//抽象类/纯虚函数 1.普通函数和2.抽象函数/纯虚函数
class BaseActivity{
private:
    void setContentView(string layoutResId){
        cout << "xmlResourseParse" << endl;
    }
public:
    //1.普通函数
    void onCreate(){
        setContentView(getLayoutId());
        
        initView();
        initData();
        initListener();
    }    
    //2.抽象函数/纯虚函数  
    
    virtual string getLayoutId() = 0;
    virtual void initView() = 0;
    virtual void initData() = 0;
    virtual void initListener() = 0;
};

//子类没有实现基类的纯虚函数，相当于子类为抽象类，抽象类不能实例化
class MainActivity : public BaseActivity{
    //重写所有的基类纯虚函数
    string getLayoutId(){}
    void initView(){}
    initData(){}
    initListener(){}
};
```

6.全纯虚函数（Java 版接口） 。

一个类中所有的函数都是纯虚函数，就相当于Java的接口

```c++
//实际上带有纯虚函数的类不能够在栈区和堆区实例化，所以这里类中默认为private不影响，访问不到
class IStudentDB{
    virtual void insertStudent() = 0;
    virtual void deleteStudent() = 0;
    virtual void updateStudent() = 0;
    virtual void queryStudent() = 0;
};

//c++实现类
class StudentDBImpl : public IStudentDB{
public:    
    void insertStudent(){}
    void deleteStudent(){}
    updateStudent(){}
    queryStudent(){}
};
```

7.回调。

这里需要注意，纯虚函数的类，绝对不能实例化

```c++
class SuccessBean{
public:
    string username;
    string userpwd;
    
    SuccessBean(string username, string userpwd) : 
    username(username), userpwd(userpwd){}
};

//定义c++接口
class ILoginResponse{
public:
    virtual void loginSuccess(int code, int message, SuccessBean successBean);
    
    virtual void loginError(int code, int message);

};

//登录API动作
void loginAction(string name, string pwd, ILoginResponse& loginResponse){
    if (name.empty() || pwd.empty()){
        cout << "用户名或密码为null" << endl;
        return;
    }
    
    if ("Derry" == name && "123" == pwd)
    {
        loginResponse.loginSuccess(200, "登陆成功", successBean(name, "恭喜你进入"));
    } else
    {
        loginResponse.loginError(404, "登陆失败");
    }
}
```

写一个实现类，继承接口

```c++
class LoginResponseImpl : public ILoginResponse{
public:
    void loginSuccess(int code, string message, SuccessBean successBean){
        cout << "恭喜登陆成功" << endl;
    }
    
    void loginError(int code, string message){
        cout << "登陆失败" << endl;
    }
};
```

最终调用

``` c++
string username;
cout << "输入用户。。。" << endl;
cin >> username;

string userpwd;
cout << "请输入密码。。。" << endl;

LoginResponseImpl loginResponseImpl;
loginResponseImpl(username ,userpwd, loginResponseImpl);
```

8.模版函数（Java 版泛型）  



```c++
template <typename T>
void addAction(T n1, T n2){
    cout << "模板函数：" << n1 + n2 << endl;
}

//调用
addAction(1, 2);
addAction(10.2f, 20.3f);
addAction(545.3, 324.3);
addAction<string>("AAA", "BBB");
```



9.继承关系的时候，构造函数和析构函数的顺序

一定是基类先构造，然后再派生类在构造。

析构一定是派生类先析构，然后再基类。



10.关于访问权限

[类中公有继承的访问权限](https://blog.csdn.net/daa20/article/details/48845253)





