C++线程

引言

默认用minGW是没有thread这个类的，需要用CygWin才有。



## threads

```c++
#include <iostream>
#include <thread> //c++ 11后出现自带Thread,不是重头戏，封装了pthread
#include <unistd.h> //sleep需要用到这个头文件
using namespace std;

void runAction(int num){
    for(int i = 0; i < 10; i++){
        cout << "runAction" << number << endl;
        sleep(1);
    }
}

int main(){
    //Todo 方式一，main函数只等3s
    thread thread1(runAction, 100);
    sleep(3);
    cout << "main函数弹栈" << endl;
    
    //Todo 方式二，等到线程执行完成
    thread thread2(runAction, 100);
    //join 等待runAction执行完，在执行下面的代码
    thread2.join();
    cout << "main函数弹栈" << endl;
    
    return 0;
}
```



## pthread

```c++
#include <iostream>
#include <pthread.h>
using namespace std;
/******函数原型
*int pthread_create(pthread_t *thread, 					//参数1：线程ID
                    const pthread_attr_t *attr,			//参数2：线程属性
                    void *(*start_routine) (void *), 	//参数3：函数指针规则
                    void *arg);							//参数4：线程需要的参数
*******/

//这里必须写返回值，void* 那么是0或者NULL
void* customPthreadTask(void* pvoid){
    int number = *static_cast<int*>(pvoid);
    cout << "异步线程" << number << endl;
    return 0;
}

int main(){
    pthread_t pthreadId;//线程id
    int num = 5;
    pthread_create(&pthreadId, 0, customPthreadTask, &num);
    return 0;
}
```

这里的函数线程返回类型`void*`必须有返回值0或者NULL。 



pthread的三种情况分析

```c++
//第一种情况，main函数结束，子线程也全部结束
#include <iostream>
#include <pthread.h>
using namespace std;

void* customPthreadTask(void* pvoid){
    int number = *static_cast<int*>(pvoid);
    cout << "异步线程" << number << endl;
    
    for(int i = 0; i < 10; i++){
        cout << "异步线程执行" << endl;
        sleep(1);
    }
    return 0;
}

int main(){
    pthread_t pthreadId;//线程id
    int num = 5;
    pthread_create(&pthreadId, 0, customPthreadTask, &num);
    
    return 0;
}
```



```c++
//第二种情况，在main函数中通过睡眠方式，去等待异步线程，强烈不推荐
#include <iostream>
#include <pthread.h>
using namespace std;

void* customPthreadTask(void* pvoid){
    int number = *static_cast<int*>(pvoid);
    cout << "异步线程" << number << endl;
    
    for(int i = 0; i < 10; i++){
        cout << "异步线程执行" << endl;
        sleep(1);
    }
    return 0;
}

int main(){
    pthread_t pthreadId;//线程id
    int num = 5;
    pthread_create(&pthreadId, 0, customPthreadTask, &num);
    sleep(500);//这种方式不推荐
    return 0;
}
```



```c++
//第三种情况，main函数一直等待异步线程，只有异步线程执行完成后，执行join后面的语句
#include <iostream>
#include <pthread.h>
using namespace std;

void* customPthreadTask(void* pvoid){
    int number = *static_cast<int*>(pvoid);
    cout << "异步线程" << number << endl;
    
    for(int i = 0; i < 10; i++){
        cout << "异步线程执行" << endl;
        sleep(1);
    }
    return 0;
}

int main(){
    pthread_t pthreadId;//线程id
    int num = 5;
    pthread_create(&pthreadId, 0, customPthreadTask, &num);
    pthread_join(pthreadId, 0);//阻塞，直到线程执行完成
    
    return 0;
}
```



## 分离线程和非分离线程

区别

分离线程：各个线程都是自己运行自己的，老死不相往来

形如上述mian函数执行完成之后，所有子线程也全部结束。【多线程情况下场景】

非分离线程：main函数线程会等待，直到异步线程完成后，在执行main函数的代码【协作，顺序执行】

例如pthread_join()



## 锁

C++互斥锁类似于java中的synchronized关键字（多线程安全的，内置锁）

```c++
#include <iostream>
#include <queue>
#include <unistd.h>
using namespace std;

queue<int> queueData;//定义一个全局的队列

pthread_mutex_t mutex;//定义一个互斥锁

void* task(void* pVoid){
    pthread_mutex_lock(&mutex);
    cout << "异步线程-当前线程的标记" << *static_cast<int*>(pVoid) << endl;
    
    if (!queueData.empty()){
        cout << "异步线程获取队列数据" << queueData.front() << endl;
        queueData.pop();
    }else{
        cout << "异步线程获取队列数据失败" << endl;
    }
    sleep(1);
    
    pthread_mutex_unlock(&mutex);
    return 0;
}

int main(){
    //初始化互斥锁
    pthread_mutex_init(&mutex, NULL);
    
    //给队列初始化数据，手动添加数据进去
    for(int i = 10001; i < 10006; ++i){
        queueData.push(i);
    }
    //一次性定义10个线程
    pthread_t pthreadIDArray[10];
    for(int i = 0; i < 10; i++){
        pthread_create(&pthreadIDArray[i], 0, task, &i);
    }
    //摧毁线程
    pthread_mutex_destroy(&mutex);
    return 0;
}
```



## 条件变量+互斥锁==Java（notify与wait）



```c++
//safe_queue_tool.h
#ifndef SAFE_QUEUE_TOOL_H
#define SAFE_QUEUE_TOOL_H
#endif

#pragma once//防止重复写include的控制，防止重复引入比如写了多个#include <iostream>

#include <iostream>
#include <string>
#include <pthread.h>
#include <queue>
using namespace std;

//定义模板函数
template<typename T>
class SafeQueue{
private:
    queue<T> queue;
    pthread_mutex_t mutex;	//定义互斥锁
    pthread_cond_t cond;	//定义条件变量，实现等待和读取功能（不能允许有野指针）
    
public:
    SafeQueue(){
        pthread_mutex_init(&mutex, 0);
        pthread_cond_init(&cond, 0);
    }
    
    ~SafeQueue(){
        pthread_mutex_destroy(&mutex);
        pthread_cond_destroy(&cond);
    }
    
    //加入到队列
    void add(T t){
        pthread_mutex_lock(&mutex);
        queue.push(t);
        //通知消费者，已经生产好了
        //pthread_cond_signal(&cond);//类似java notify
        pthread_cond_broadcast(&cond);//类似java notifyAll
        cout << "通知线程" << endl;
        
        pthread_mutex_unlock(&mutex);
    }
    
    void get(T& t){
        pthread_mutex_lock(&mutex);
        
        if (queue.empty()){
            cout << "等待被唤醒" << endl;
            pthread_cond_wait(&cond, &mutex);
        }
        t = queue.front();//得到队列中的元素数据
        queue.pop();//删除元素
        
        pthread_mutex_unlock(&mutex);
    }
};
```



```c++
//调用生产消费者
#pragma once
#include <iostream>
#include "safe_queue_tool.h"
using namespace std;

SafeQueue<int> sq;

void* getMethod(void* pVoid){
    while(true){
        cout << "getMethod" << endl;
        int value;
        sq.get(value);
        cout << "消费者数据：" << value << endl;
        
        if (-1 == value){
            break;
        }
    }
    return 0;
}

void* setMethod(void* pVoid){
    while(true){
        cout << "输入要生成的信息" << endl;
        cin >> value;
        if (-1 == value){
            sq.add(value);
            cout << "生产者结束" << endl;
            break;
    	}	
    	sq.add(value);
    }
    
    return 0;
}

int main(){
    pthread_t pthreadGet;
    pthread_create(&pthreadGet, 0, getMethod, 0);
    pthread_t pthreadSet;
    pthread_create(&pthreadSet, 0, setMethod, 0);
    
    pthread_join(pthreadGet, 0);
    
    pthread_join(pthreadSet, 0);
    return 0;
}
```

![image-20220111225452221](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220111225452221.png)



## 关于win中没有pthread包的解决

![image-20220111230437562](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220111230437562.png)

这里需要修改系统源码配置，pthread源码中增加一个宏，但是不推荐

采取下面这种办法，类似设置环境变量。-D增加宏

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS } -DHAVE_STRUCT_TIMESPEC")#定义预编译宏

dll是动态库，lib是静态库



[`#pragma`](https://blog.csdn.net/lmhuanying1012/article/details/78549763)



笔者实际上实战，x86_64-8.1.0-release-posix-sjlj-rt_v6-rev0\mingw64默认是支持pthread，唯一需要改的就是添加编译的。h和。cpp包括链接到目标库需要增加pthread

```cmake
cmake_minimum_required(VERSION 3.17)
project(CPPClionProject) # 目标库 CPPClionProject

set(CMAKE_CXX_STANDARD 14)
add_executable(CPPClionProject  T7.cpp safe_queue_too.h)

# TODO 链接到目标库  必须增加：pthread
target_link_libraries(CPPClionProject pthread)
```

