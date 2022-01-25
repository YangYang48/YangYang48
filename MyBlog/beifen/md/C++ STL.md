C++ STL

## venctor

C++的向量容器

```c++
#include <iostream>
#include <vector> 

using namespace std;

int main(){
    vector<int> vector1;
    
    vector<int> vector2(10); //指定10个空间大小
    vector<int> vector3(10, 0);//有10个值，每个值都为0
    //插入数据vector1.begin()这个是迭代器
    //vector1.begin()插入到前面
    //vector1.end()插入到后面
    vector1.insert(vector1.begin(), 40);
    vector1.insert(vector1.begin(), 50);
    vector1.insert(vector1.begin(), 60);
    vector1.insert(vector1.begin(), 80);
    //输出第一个vector1.begin()
    //输出最后一个vector1.back()
    cout << vector1.front() << endl;
    
    //打印输出
    //方法1：直接打印,运算符重载[]
    for (int i = 0; i < vector1.size(); ++i){
        cout << vector1[i] << endl;
    }
    //方法2：迭代器输出
    for(vector<int>::iterator it; it != vector.end(); it++){
        cout << *it << endl;
    }
    
    return 0;
}
```



## stack

先进后出的数据结构

```c++
#include <iostream>
#include <stack> 

using namespace std;

int main(){
    stack<int> stackVar;
    
    stackVar.push(20);
    stackVar.push(40);
    stackVar.push(80);
    
    //打印所有元素的值
    while(!stackVar.empty())
    {
        cout << stackVar.top() << endl;
        stackVar.pop();//需要把最前面的弹出,相当于弹出栈区元素
    }
    
    return 0;
}
```



## queue

队列，一般最常用

```c++
#include <iostream>
#include <queue> //队列支持，内部链表或者数组

using namespace std;

int main(){
    queue<int> queueVar;
    
    queueVar.push(20);
    queueVar.push(40);
    queueVar.push(80);
    
    //打印所有元素的值
    while(!queueVar.empty())
    {
        cout << queue.front() << endl;
        queueVar.pop();//需要把最前面的弹出
    }
    
    return 0;
}
```

## 优先级队列

优先级队列是队列的子集

```c++
#include <iostream>
#include <queue> //队列支持，内部链表或者数组
//priority_queue内部对前面的vector有一定封装
//Deque为双端队列

using namespace std;

int main(){
    //优先级队列默认排序：从大到小
    //默认隐式代码priority_queue<int> == priority_queue<int, vector<int>, less<int>>
    //less在源码中是一个结构体，返回x<y,上一个元素x和当前元素y比较，返回bool类型，true交换
    priority_queue<int> priorityQueue;
    
    //从小到大排列，返回x>y
    //priority_queue<int, vector<int>, greater<int>> priorityQueue;
    
    priorityQueue.push(20);
    priorityQueue.push(40);
    priorityQueue.push(80);
    
    //打印所有元素的值
    while(!priorityQueue.empty())
    {
        cout << priorityQueue.front() << endl;
        priorityQueue.pop();//需要把最前面的弹出
    }
    
    return 0;
}
```



## list

其中java中的ArrayList采用Object[]，C++list内部实现采用链表

将元素按顺序储存在链表中. 与 向量(vectors)相比, 它允许快速的插入和删除，但是随机访问却比较慢.

```c++
#include <iostream>
#include <list> //list容器

using namespace std;

int main(){
    list<int> listVar;
    
    listVar.push_front(50);
    listVar.push_back(60);
    listVar.insert(listVar.begin(), 70);
    listVar.insert(listVar.end(), 80);
    //修改操作
    listVar.back() = 88;
    listVar.front() = 55;
    
    //删除
    list Var.erase(listVar.begin());
    list Var.erase(listVar.end());
    //打印所有元素的值,不能角标访问
    for (list<int>::iterator it = listVar.begin(); it != listVar.end(); it++){
        cout << *it << endl;
    }
    
    return 0;
}
```

## set

```c++
#include <iostream>
#include <set> //内部逻辑红黑树，对存入的数据进行维护，但绝不允许元素相同

using namespace std;

int main(){
    //从小到大排列，__x < __y上一个元素小于当前元素
    set<int, less<int>> setVar;
    setVar.insert(1);
    setVar.insert(2);
    setVar.insert(3);
    setVar.insert(4);
    //set重复插入不会报错，除非通过返回纸坊判断std::pair<iterator, bool>insert(value_type&& __x)
    //通过返回值pair查看是否插入成功
    pair<set<int, less<int>>::iterator, bool> res = setVar.insert(4);//相同的插入失败
    //拿到返回值的第二个泛型元素
    bool insert_success = res.second;
    if (insert_success){
        cout << "插入成功" << endl;
        for(;res.first != setVar.end(); res.first++)
        {
            cout << *res.first << endl;
        }
    }else{
         cout << "插入失败" << endl;
    }
    
    //遍历
    for (auto it = setVar.begin(); it != setVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```



函数谓词

谓词[如何最简单、通俗地理解C++的谓词？](https://www.zhihu.com/question/441638427)

```c++
class Person{
public:
    string name;
    int id;
    Person(string name,int id) : name(name), id(id){}
};
```



```c++
int main(){
    set<Person> setVar;
    
    //构建对象
    Person p1("x1", 1);
    Person p2("x2", 2);
    Person p3("x3", 3);
    Person p4("x4", 4);
    
    //构建的对象插入set容器
    setVar.insert(p1);
    setVar.insert(p2);
    setVar.insert(p3);
    
    for (set<Person>::iterator it = setVar.begin(); it != setVar.end(); it++){
        //错误：无法让set排序
        //cout << it->name << endl;
    }
    
    return 0;
}
```

注：string可以直接输出，cout对其重载了，必须是string类库中的string类型，而不是CString或者是string.h中的。

[C++中关于string类型究竟能不能用cout输出的问题](https://www.cnblogs.com/mzct123/p/4876185.html)



默认不作修改会崩溃，需要自定义谓词

![image-20220105215636992](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220105215636992.png)

默认源码是这么写的，仿照写一个自定义的即可。

![image-20220105215826411](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220105215826411.png)

```c++
//通过自定义谓词，比较id的大小来排序
struct doCompareAction2{
    bool operator() (const Person& __x, const Person& __y)const  
    {
        return __x.id < __y.id;
    }
};
```



## map

map会有key和value，其中map会对key进行排序

前三种方式key不能重复，后一种方式可以重复

map中insert重载

![image-20220106221737063](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220106221737063.png)

```c++
#include <iostream>
#include <map> //map容器

using namespace std;

int main(){
    //map会对key排序
    map<int, string> mapVar;
    
    //四种方式添加数据
    //前三种方式不可重复key插入
    mapVar.insert(pair<int, string>(1, "一"));
    mapVar.insert(make_pair(2, "二"));
    mapVar.insert(map<int, string>::value_type(3, "三"));
    mapVar.insert(pair<int, string>(3, "xx"));//会插入失败，相当于这次操作不存在
    //第四种方式====================================
    //第四种方式重复key插入会覆盖
    mapVar[4] = "四";//mapVar[key] = value
    mapVar[4] = "si";// "四"被覆盖成"si"
    
    //循环打印输出
    for (map<int, string>::iterator it = mapVar.begin(); it != mapVar.end(); it++)
    {
        cout << it->first << ", " << it->second << endl;
    }
    
    //insert函数原型std::pair<iterator, bool> insert(const value_type& __x)
    pair<map<int, string>::iterator, bool> result = mapVar.insert(pair<int, string>(3, "xx"));
    if (result.second){
        cout << "插入成功" << endl;
    }else
    {
         cout << "插入失败" << endl;
    }
    
    //查找map,返回的是一个查找的迭代器
    map<int, string>::iterator findResult = mapVar.find(3);
    if (findResult != mapVar.end())
    {
        cout << "恭喜，找到了" << findResult->second << endl;
    }else
    {
         cout << "没找到" << endl;
    }
    return 0;
}
```

map的迭代器

![image-20220106222600578](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220106222600578.png)



## multimap

这个key可以重复，也是map的子集

```c++
#include <iostream>
#include <map> //map容器

using namespace std;

int main(){
    //1.key可以重复 2.key重复的数据可以分组
    multimap<int, string> multimapVar;
    
    multimapVar,insert(make_pair(10, "1"));
    multimapVar,insert(make_pair(10, "2"));
    multimapVar,insert(make_pair(10, "3"));
    
    multimapVar,insert(make_pair(30, "1"));
    multimapVar,insert(make_pair(30, "2"));
    multimapVar,insert(make_pair(30, "3"));
    
    multimapVar,insert(make_pair(20, "1"));
    multimapVar,insert(make_pair(20, "2"));
    multimapVar,insert(make_pair(20, "3"));
    
    for(multimapVar<int, string>::iterator it = multimapVar.begin(); it != multimapVar.end(); it++){
        cout << it->first << ", " << it->second << endl;
    } 
    
    cin >> result;
    //分组查询
    multimap<int, string>::iterator it = multimapVar.find(result);
    while(iterator != multimapVar.end()){
        cout << it->first << ", " << endl;
        it++;
        //如果下一个不是当前的key，则退出循环
        if(it->first != result){
            break;
        }
    }
    return 0;
}
```

默认会排列

![image-20220108130755248](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108130755248.png)

分组查询

![image-20220108131532981](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108131532981.png)



## C++内部的仿函数

谓词包含仿函数，谓词还可以是回调函数

```c++
#include <iostream>
#include <set> //stl包
#include <algorithm> //算法包

using namespace std;

class CompareObj
{
public:
    //这里没有参数为空谓词，有一个参数为一元谓词
    void operator()(){
        cout << "仿函数" << endl;
    }
};
int main(){
    CompareObj obj;
    obj();
    return 0;
}
```

使用算法包需要自定义仿函数（一元谓词）

![image-20220108132830691](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108132830691.png)



```c++
#include <iostream>
#include <set> //stl包
#include <algorithm> //算法包

using namespace std;
//仿函数，c++内部源码仿函数用的最多
class showObj
{
public:
    //这里没有参数为空谓词，有一个参数为一元谓词
    void operator()(int content){
        cout << "仿函数" << content << endl;
    }
};

//回调函数
void showAction(int content){
    cout << "自定义一元谓词" << endl;
}

int main(){
    set<int> setVar;
    setVar.insert(10);
    setVar.insert(20);
    setVar.insert(30);
    setVar.insert(40);
    setVar.insert(50);
    setVar.insert(60);
    
    //属于运算符重载
    //for_each(setVar.begin(), setVar.end(), showObj());
    //属于回调函数，不需要加括号
    for_each(setVar.begin(), setVar.end(), showAction); 
    
    return 0;
}
```

可以对仿函数进行改造

```c++
#include <iostream>
#include <set> //stl包
#include <algorithm> //算法包

using namespace std;
//仿函数，c++内部源码仿函数用的最多
class showObj
{
public:
    int count = 0;
    void _count(){cout << this->count << endl;}
    //这里没有参数为空谓词，有一个参数为一元谓词
    void operator()(int content){
        cout << "仿函数" << content << endl;
        count++;
    }

};

//回调函数
void showAction(int content){
    cout << "自定义一元谓词" << endl;
}

int main(){
    set<int> setVar;
    setVar.insert(10);
    setVar.insert(20);
    setVar.insert(30);
    setVar.insert(40);
    setVar.insert(50);
    setVar.insert(60);
    
    //可以用对象来用
    showObj s;
    //前面的s上面showobj的s，后面的s也为上面showobj的s，但函数内部作用的s为后面s的拷贝构造，即值传递
    s = for_each(setVar.begin(), setVar.end(), s); 
    s._count();
    
    return 0;
}
```



## C++容器中对象的生命周期

```c++
#include <iostream>
#include <vector> //stl包
#include <algorithm> //算法包

using namespace std;
//仿函数，c++内部源码仿函数用的最多
class Person
{
private:
    string name;
public:
    Person(string name) : name(name){}
    
    void setName(string name){this->name = name;}
    
    string getName(){return this->name;}
    
    Person(const Person& person){
        this->name = person->name;//浅拷贝
    }
    
    ~Person(){
        
    }
};

int main(){
    //java：把对象存入添加到集合
    //C++ 调用拷贝构造函数，存进去的是一个新的对象
    vector<Person> vectorVar;
    Person person("Derry");
    vectorVar.insert(veCtotVar.begin(), person);
    
    person.setName("kevirn");
    
    Person newPerson = vectorVar.front();
    //这里的newPerson.getName()还是Derry，因为实际是拷贝构造进去的
    cout << newPerson.getName() << endl;
    return 0;
}
```



## 预定义函数

预定义函数为C++内置函数，需要引入算法包

```c++
#include <iostream>
#include <set>
#include <algorithm>
using namespace std;

int main()
{
    //c++预定义了函数plus，minus,multiplies
    plus<int> add_func;
    int r = add_func(1, 1);
    cout << r << endl;
    
    
    return 0;
}

```

stl_function.h

![image-20220108145237966](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108145237966.png)



## 函数适配器

C++算法包

![image-20220108221357888](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108221357888.png)

```c++
#include <iostream>
#include <set>
#include <algorithm>
using namespace std;

int main(){
    set<string, less<string>> setVar;
    setVar.insert("AAAA");
    setVar.insert("BBBB");
    setVar.insert("CCCC");
    setVar.insert("DDDD");
    //迭代器遍历
    for(auto it = setVar.begin(); it != setVar.end(); it++){
        cout << *it << endl;
    }
    
    //使用函数适配器bind2nd，就能够CCCC传递到const __Tp& __y;
    //setVar.begin(), setVar.end()会把这些元素取出来const __Tp& __y;
    //bind2nd() 表示第二个参数适配过去
    //bind1st() 表示第一个参数适配过去
    set<string, less<string>>::iterator result = 
        find_if(setVar.begin(), setVar.end(), bind2nd(equal_to<string>(), "CCCC"));
    if (result != setVar.end()){
        cout << "查找到了" << endl;
    }else{
        cout << "没有查找到" << endl;
    }
    
    return 0;
}
```

equal_to就是函数比较

![image-20220108223732007](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108223732007.png)



<img src="C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108223801136.png" alt="image-20220108223801136" style="zoom:80%;" />

[c++中的函数适配器](https://blog.csdn.net/liuyuchen282828/article/details/98482134)详情可以看这篇文章



## C++算法包内置源码全面研究  

1.for_each，自定义谓词

```c++
//stl_algo.h#for_each
template<typename _InputIterator, typename _Function>
_Function
for_each(_InputIterator __first, _InputIterator __last, _Function __f)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_requires_valid_range(__first, __last);
    for (; __first != __last; ++__first)
        __f(*__first);
    return __f; // N.B. [alg.foreach] says std::move(f) but it's redundant.
}
```

![image-20220108224747729](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108224747729.png)



2.transform

遍历打印和变换修改值

```c++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

class unary_op{
public:
    int operator()(const int first){
        return first + 9;//将每个元素都增加9
    }
};

int main(){
    vector<int> vectorVar;
    vectorVar.insert(vectorVar.begin(), 10000);
    vectorVar.insert(vectorVar.begin(), 20000);
    vectorVar.insert(vectorVar.begin(), 30000);
    vectorVar.insert(vectorVar.begin(), 40000);
    //第三个参数可以是本身的容器，覆盖之前的值
    transform(vectorVar.begin(), vectorVar.end(), vectorVar.begin(), unary_op());//自定义仿函数unary_op
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++){
        cout << *it << endl;
    }
    //第三个参数可以是其他容器，新的一个容器
    vector<int> result;
    result.resize(vectorVar.size());//必须设置容器大小
    transform(vectorVar.begin(), vectorVar.end(), result.begin(), unary_op());
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++){
        cout << *it << endl;
    }
    
    return 0;
}
```



```c++
//stl_algo.h#transform
template<typename _InputIterator, typename _OutputIterator,
typename _UnaryOperation>
_OutputIterator
transform(_InputIterator __first, _InputIterator __last,
          _OutputIterator __result, _UnaryOperation __unary_op)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    // "the type returned by a _UnaryOperation"
                                    __typeof__(__unary_op(*__first))>)
        __glibcxx_requires_valid_range(__first, __last);

    for (; __first != __last; ++__first, (void)++__result)
        *__result = __unary_op(*__first);
    return __result;
}
```

![image-20220108231101311](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108231101311.png)



3.find和find_if

find没有自定义仿函数，find_if有

find和find_if其实都是对__find_if的封装

```c++
//stl_algo.h#find
//开始的迭代器_InputIterator __first
//结尾的迭代器_InputIterator __last
//需要查找的元素const _Tp& __val
template<typename _InputIterator, typename _Tp>
inline _InputIterator
find(_InputIterator __first, _InputIterator __last,
     const _Tp& __val)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_InputIterator>::value_type, _Tp>)
        __glibcxx_requires_valid_range(__first, __last);
    return std::__find_if(__first, __last,
                          __gnu_cxx::__ops::__iter_equals_val(__val));
}
```



```c++
//stl_algo.h#find_if
//开始的迭代器_InputIterator __first
//结尾的迭代器_InputIterator __last
//自定义仿函数_Predicate __pred
template<typename _InputIterator, typename _Predicate>
inline _InputIterator
find_if(_InputIterator __first, _InputIterator __last,
        _Predicate __pred)
{
    // concept requirements
    //用于检测工作
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_UnaryPredicateConcept<_Predicate,
                                    typename iterator_traits<_InputIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);
	//这里传的第三个入参是predefined_ops.h中的__pred_iter，也是一个仿函数，仿函数参数为外部自定义的仿函数__pred
    return std::__find_if(__first, __last,
                          __gnu_cxx::__ops::__pred_iter(__pred));
}
```



```c++
//stl_algo.h#__find_if
template<typename _InputIterator, typename _Predicate>
inline _InputIterator
__find_if(_InputIterator __first, _InputIterator __last,
          _Predicate __pred, input_iterator_tag)
{
    //这里的__pred(__first)会被迷惑到，但实际上是内部仿函数的封装，实际传的值是*__first
    while (__first != __last && !__pred(__first))
        ++__first;
    return __first;
}
```

这里笔者用得是MinGW，实际上内部封装了仿函数传入迭代器的操作，所以我们外部定义仿函数操作的时候直接传入`*__first`值即可，这里传入的就是int型，而不是`int*`型。

```c++
//predefined_ops.h#_Iter_pred
//这个就是内部封装的仿函数，__find_if(__first)会调用到这里
template<typename _Predicate>
inline _Iter_pred<_Predicate>
__pred_iter(_Predicate __pred)
{ 
    return _Iter_pred<_Predicate>(_GLIBCXX_MOVE(__pred)); 
}

//predefined_ops.h#_Iter_pred
//从find_if函数中可以发现，最终的内部仿函数为_Iter_pred，再去调用外部定义的仿函数_M_pred
template<typename _Predicate>
struct _Iter_pred
{
    _Predicate _M_pred;

    explicit
        _Iter_pred(_Predicate __pred)
        : _M_pred(_GLIBCXX_MOVE(__pred))
        { }

    template<typename _Iterator>
    bool
    operator()(_Iterator __it)
    { return bool(_M_pred(*__it)); }
};
```

外部自定义仿函数

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

class __pred {
public:
    int number;
    __pred(int number) : number(number) {}
    bool operator() (const int value) {
        cout << "value = " << value << endl;
        return number == value;
    }
};

int main() {
    vector<int> vectorVar;
    vectorVar.insert(vectorVar.begin(), 10000);
    vectorVar.insert(vectorVar.begin(), 20000);
    vectorVar.insert(vectorVar.begin(), 30000);
    vectorVar.insert(vectorVar.begin(), 40000);

    auto it = find_if(vectorVar.begin(), vectorVar.end(), __pred(30000));

    if (it != vectorVar.end()) {
        cout << "查找到了" << endl;
    } else {
        cout << "没有找到" << endl;
    }
    return 0;
}
```



4.count和count_if

用于统计元素的个数

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    //统计2的个数
    int number = count(vectorVar.begin(), vectorVar.end(), 2);
    //count_if比较灵活可以统计比某个数大的，比某个数小的，和某个数相等的
    //count_if同时也需要用到函数适配器bind2nd
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(less<int>(), 2));
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(greater<int>(), 2));
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(equal_to<int>(), 2));
    return 0;
}
```

和上述的find和find_if类似，这里就不重复贴内部封装的仿函数

```c++
//stl_algo.h#count
template<typename _InputIterator, typename _Tp>
inline typename iterator_traits<_InputIterator>::difference_type
count(_InputIterator __first, _InputIterator __last, const _Tp& __value)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_InputIterator>::value_type, _Tp>)
        __glibcxx_requires_valid_range(__first, __last);

    return std::__count_if(__first, __last,
                           __gnu_cxx::__ops::__iter_equals_val(__value));
}

//stl_algo.h#count_if
template<typename _InputIterator, typename _Predicate>
inline typename iterator_traits<_InputIterator>::difference_type
count_if(_InputIterator __first, _InputIterator __last, _Predicate __pred)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_UnaryPredicateConcept<_Predicate,
                                    typename iterator_traits<_InputIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    return std::__count_if(__first, __last,
                           __gnu_cxx::__ops::__pred_iter(__pred));
}

//stl_algo.h#__count_if
template<typename _InputIterator, typename _Predicate>
typename iterator_traits<_InputIterator>::difference_type
    __count_if(_InputIterator __first, _InputIterator __last, _Predicate __pred)
{
    typename iterator_traits<_InputIterator>::difference_type __n = 0;
    for (; __first != __last; ++__first)
        if (__pred(__first))
            ++__n;
    return __n;
}
```



5.merge

用于容器的合并

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    vector<int> vectorVar1;
    vectorVar1.push_back(1);
    vectorVar1.push_back(2);
    vectorVar1.push_back(3);
    
    vector<int> result;
    result.resize(vectorVar.size() + vectorVar1.size());
    merge(vectorVar.begin(), vectorVar.end(), vectorVar1.begin(), vectorVar.end(), result.begin());
    
    for(auto it = result.begin(); it != result.end(); it++){
        cout << *it << endl;
    }
    return 0;
}
```

源码如下

```c++
//stl_algo.h#merge
template<typename _InputIterator1, typename _InputIterator2,
typename _OutputIterator>
inline _OutputIterator
merge(_InputIterator1 __first1, _InputIterator1 __last1,
          _InputIterator2 __first2, _InputIterator2 __last2,
          _OutputIterator __result)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator1>)
        __glibcxx_function_requires(_InputIteratorConcept<_InputIterator2>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    typename iterator_traits<_InputIterator1>::value_type>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    typename iterator_traits<_InputIterator2>::value_type>)
        __glibcxx_function_requires(_LessThanOpConcept<
                                    typename iterator_traits<_InputIterator2>::value_type,
                                    typename iterator_traits<_InputIterator1>::value_type>)	
        __glibcxx_requires_sorted_set(__first1, __last1, __first2);
    __glibcxx_requires_sorted_set(__first2, __last2, __first1);
    __glibcxx_requires_irreflexive2(__first1, __last1);
    __glibcxx_requires_irreflexive2(__first2, __last2);

    return _GLIBCXX_STD_A::__merge(__first1, __last1,
                                   __first2, __last2, __result,
                                   __gnu_cxx::__ops::__iter_less_iter());
}

//stl_algo.h#__merge
//前面先做一个排序工作，后面再进行一个copy工作
template<typename _InputIterator1, typename _InputIterator2,
typename _OutputIterator, typename _Compare>
_OutputIterator
__merge(_InputIterator1 __first1, _InputIterator1 __last1,
            _InputIterator2 __first2, _InputIterator2 __last2,
            _OutputIterator __result, _Compare __comp)
{
    while (__first1 != __last1 && __first2 != __last2)
    {
        if (__comp(__first2, __first1))
        {
            *__result = *__first2;
            ++__first2;
        }
        else
        {
            *__result = *__first1;
            ++__first1;
        }
        ++__result;
    }
    return std::copy(__first2, __last2,
                     std::copy(__first1, __last1, __result));
}
```



6.sort

排序

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //从小到大
    sort(vectorVar.begin(), vectorVar.end(), less<int>());
    //从大到小
    //sort(vectorVar.begin(), vectorVar.end(), greater<int>());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

sort源码，通过一层层剥离可以看到调用到`__comp(__i, __first)`可以看出仿函数是bool类型的

```c++
//stl_algo.h#sort
template<typename _RandomAccessIterator, typename _Compare>
inline void
sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
     _Compare __comp)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
                                _RandomAccessIterator>)
        __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
                                    typename iterator_traits<_RandomAccessIterator>::value_type,
                                    typename iterator_traits<_RandomAccessIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);
    __glibcxx_requires_irreflexive_pred(__first, __last, __comp);

    std::__sort(__first, __last, __gnu_cxx::__ops::__iter_comp_iter(__comp));
}

//stl_algo.h#__sort
template<typename _RandomAccessIterator, typename _Compare>
inline void
__sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
       _Compare __comp)
{
    if (__first != __last)
    {
        std::__introsort_loop(__first, __last,
                              std::__lg(__last - __first) * 2,
                              __comp);
        std::__final_insertion_sort(__first, __last, __comp);
    }
}

//stl_algo.h#__final_insertion_sort
template<typename _RandomAccessIterator, typename _Compare>
void
__final_insertion_sort(_RandomAccessIterator __first,
                       _RandomAccessIterator __last, _Compare __comp)
{
    if (__last - __first > int(_S_threshold))
    {
        std::__insertion_sort(__first, __first + int(_S_threshold), __comp);
        std::__unguarded_insertion_sort(__first + int(_S_threshold), __last,
                                        __comp);
    }
    else
        std::__insertion_sort(__first, __last, __comp);
}

//stl_algo.h#__insertion_sort
template<typename _RandomAccessIterator, typename _Compare>
void
__insertion_sort(_RandomAccessIterator __first,
                 _RandomAccessIterator __last, _Compare __comp)
{
    if (__first == __last) return;

    for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
    {
        if (__comp(__i, __first))
        {
            typename iterator_traits<_RandomAccessIterator>::value_type
                __val = _GLIBCXX_MOVE(*__i);
            _GLIBCXX_MOVE_BACKWARD3(__first, __i, __i + 1);
            *__first = _GLIBCXX_MOVE(__val);
        }
        else
            std::__unguarded_linear_insert(__i,
                                           __gnu_cxx::__ops::__val_comp_iter(__comp));
    }
}
```



7.random_shuffle

随记打乱

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //从小到大
    sort(vectorVar.begin(), vectorVar.end(), less<int>());
    //从大到小
    //sort(vectorVar.begin(), vectorVar.end(), greater<int>());
    //随机打乱
    random_shuffle(vectorVar.begin(), vectorVar.end());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，可以看到通过除以一个rand()数来计算

```c++
//stl_algo.h#random_shuffle
template<typename _RandomAccessIterator>
inline void
random_shuffle(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
                                _RandomAccessIterator>)
        __glibcxx_requires_valid_range(__first, __last);

    if (__first != __last)
        for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
        {
            // XXX rand() % N is not uniformly distributed
            _RandomAccessIterator __j = __first
                + std::rand() % ((__i - __first) + 1);
            if (__i != __j)
                std::iter_swap(__i, __j);
        }
}
```



8.copy

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    vector<int> vectorVar1;
    vector1.resize(vectorVar.size());
    copy(vectorVar.begin(), vectorVar.end(), vectorVar1.end());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，实际上是遍历原来的数组放进去

```c++
//stl_algo.h#copy
template<typename _II, typename _OI>
inline _OI
copy(_II __first, _II __last, _OI __result)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_II>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OI,
                                    typename iterator_traits<_II>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    return (std::__copy_move_a2<__is_move_iterator<_II>::__value>
            (std::__miter_base(__first), std::__miter_base(__last),
             __result));
}

//stl_algo.h#__copy_move_a2
template<bool _IsMove, typename _CharT>
typename __gnu_cxx::__enable_if<__is_char<_CharT>::__value,
ostreambuf_iterator<_CharT> >::__type
    __copy_move_a2(_CharT* __first, _CharT* __last,
                   ostreambuf_iterator<_CharT> __result)
{
    const streamsize __num = __last - __first;
    if (__num > 0)
        __result._M_put(__first, __num);
    return __result;
}
```



9.replace

替换容器元素

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //第三个参数为原来的值，最后一个参数为新值
    replace(vectorVar.begin(), vectorVar.end(), 5, 6);
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，可以看到这里用到了引用值，所以没有副本存在，这里面直接修改了迭代器的值，本身迭代器就跟智能指针类似，传入的迭代器修改会同步迭代器的本身。

```c++
template<typename _ForwardIterator, typename _Tp>
void
replace(_ForwardIterator __first, _ForwardIterator __last,
        const _Tp& __old_value, const _Tp& __new_value)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
                                _ForwardIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_ForwardIterator>::value_type, _Tp>)
        __glibcxx_function_requires(_ConvertibleConcept<_Tp,
                                    typename iterator_traits<_ForwardIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    for (; __first != __last; ++__first)
        if (*__first == __old_value)
            *__first = __new_value;
}
```

