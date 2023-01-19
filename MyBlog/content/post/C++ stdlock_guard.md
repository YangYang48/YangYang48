---
title: "C++ std::lock_guard"
date: 2022-09-04
thumbnailImagePosition: left
thumbnailImage: c++/base/base4_thumb.jpg
coverImage: c++/base/base4_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 锁
- 2022
- September
tags:
- c++
- c++基础
- Android
- 源码
showSocial: false
---

Android源码中有一些经常会遇到的c++基础的内容，笔者对C++ std::lock_guard进行简单熟悉。

<!--more-->
# C++ std::lock_guard

在Android源码中，经常会遇到std::lock_guard这个锁的用法。

```c++
//system/core/init/init.cpp
static class PropWaiterState {
  public:
    bool StartWaiting(const char* name, const char* value) {
        auto lock = std::lock_guard{lock_};
        if (waiting_for_prop_) {
            return false;
        }
        if (GetProperty(name, "") != value) {
            // Current property value is not equal to expected value
            wait_prop_name_ = name;
            wait_prop_value_ = value;
            waiting_for_prop_.reset(new Timer());
        } else {
            LOG(INFO) << "start_waiting_for_property(\"" << name << "\", \"" << value
                      << "\"): already set";
        }
        return true;
    }

    void ResetWaitForProp() {
        auto lock = std::lock_guard{lock_};
        ResetWaitForPropLocked();
    }

    void CheckAndResetWait(const std::string& name, const std::string& value) {
        auto lock = std::lock_guard{lock_};
        if (name == kColdBootDoneProp) {
            auto time_waited = waiting_for_prop_ ? waiting_for_prop_->duration().count() : 0;
            std::thread([time_waited] {
                SetProperty("ro.boottime.init.cold_boot_wait", std::to_string(time_waited));
            }).detach();
        }

        if (waiting_for_prop_) {
            if (wait_prop_name_ == name && wait_prop_value_ == value) {
                LOG(INFO) << "Wait for property '" << wait_prop_name_ << "=" << wait_prop_value_
                          << "' took " << *waiting_for_prop_;
                ResetWaitForPropLocked();
                WakeMainInitThread();
            }
        }
    }

    bool MightBeWaiting() {
        auto lock = std::lock_guard{lock_};
        return static_cast<bool>(waiting_for_prop_);
    }

  private:
    void ResetWaitForPropLocked() {
        wait_prop_name_.clear();
        wait_prop_value_.clear();
        waiting_for_prop_.reset();
    }

    std::mutex lock_;
    std::unique_ptr<Timer> waiting_for_prop_{nullptr};
    std::string wait_prop_name_;
    std::string wait_prop_value_;

} prop_waiter_state;
```

可以看到一个简单的类中，公有函数内部都有这种std::lock_guard的写法。



# 1lock_guard
lock_guard是一种在作用域内控制**可锁对象所有权**的类型。

> An object of type lock_guard controls the ownership of a lockable object within a scope. 

直接查看源码，可以得知实际上lock_guard作用是在lock_guard实例化对象构造的时候会**持有**对应的锁，而且这个锁是传入锁的别名，即是同一把锁。然后我这个lock_guard实例化对象析构的时候，对先前持有的锁进行**释放**。

```c++
//std_mutex.h
constexpr adopt_lock_t	adopt_lock { };

template<typename _Mutex>
class lock_guard
{
    public:
    typedef _Mutex mutex_type;
    //持有锁
    explicit lock_guard(mutex_type& __m) : _M_device(__m)
    { _M_device.lock(); }
    //这个构造有两个参数，有adopt_lock参数，构造时不加锁
    lock_guard(mutex_type& __m, adopt_lock_t) noexcept : _M_device(__m)
    { } // 调用线程拥有互斥锁
    //释放锁
    ~lock_guard()
    { _M_device.unlock(); }
    //屏蔽拷贝构造
    lock_guard(const lock_guard&) = delete;
    lock_guard& operator=(const lock_guard&) = delete;

    private:
    mutex_type&  _M_device;
};
```



# 2使用方式

lock_guard具有两种构造方法

1. `lock_guard(mutex& m)`
2. `lock_guard(mutex& m, adopt_lock)`

其中`mutex& m`是互斥量，参数`adopt_lock`表示假定调用线程已经获得互斥体所有权并对其进行管理了。

lock_guard的析构方式，只要退出所定义的作用域，这个lock_guard的实例化就会调用内部析构，释放当前的锁。

```c++
#include <iostream>
#include <vector>
#include <mutex>
#include <string>
#include <thread>
#include <cstring>

using namespace std;
#define N 10
// 时间模拟消息
bool g_flag = true;
string mock_msg()
{
    char tmp[128];
    memset(tmp, 0x0, sizeof(tmp));
    static int i = N;
    if(i == 0)
    {
        g_flag = false;
    }
    cout << "buff " << i-- << endl;
    snprintf(tmp, sizeof(tmp), "buff(%d)", i + 1);
    return tmp;
}

class lockGuardTest
{
public:
    void recv_msg(); 			//接收消息
    void read_msg(); 			//处理消息
private:
    vector<string> msg_queue; 	//消息队列
    mutex m_mutex;				//互斥量
};

// 模拟接收消息
void lockGuardTest::recv_msg()
{
    while (g_flag)
    {
        string msg = mock_msg();
        cout << "recv the msg " << msg << endl;

        // 这个括号就是lock_guard实例化mylockguard的作用域，出了作用域就会调用析构
        {
            lock_guard<mutex> mylockguard(m_mutex);
            // 消息添加到队列
            msg_queue.emplace_back(msg);
        }
        this_thread::sleep_for(chrono::milliseconds(10));//方便观察数据
    }
}

// 模拟读取处理
void lockGuardTest::read_msg()
{
    while (g_flag)
    {
        // 已经加锁，这个是采用直接加锁的方式
        m_mutex.lock();
        // 使用双参数构造lock_guard实例化mylockguard，其中传入adopt_lock就为第二种构造方式
        //且adopt_lock是std_mutex.h头文件默认定义的
        lock_guard<mutex> mylockguard(m_mutex, adopt_lock);
        if (!msg_queue.empty())
        {
            string msg = msg_queue.front();
            cout << "read the msg " << msg << endl;
            msg_queue.erase(msg_queue.begin());
        }
        this_thread::sleep_for(chrono::milliseconds(10));//方便观察数据
    }
}

int main()
{
    lockGuardTest test1;
    thread recv_thread(&lockGuardTest::recv_msg, &test1); //接收线程
    thread read_thread(&lockGuardTest::read_msg, &test1); //处理线程
    //等待线程，知道退出
    recv_thread.join();
    read_thread.join();
}
```

输出结果

```txt
buff 10
recv the msg buff(10)
read the msg buff(10)
buff 9
recv the msg buff(9)
read the msg buff(9)
buff 8
recv the msg buff(8)
read the msg buff(8)
buff 7
recv the msg buff(7)
read the msg buff(7)
buff 6
recv the msg buff(6)
read the msg buff(6)
buff 5
recv the msg buff(5)
read the msg buff(5)
buff 4
recv the msg buff(4)
read the msg buff(4)
buff 3
recv the msg buff(3)
read the msg buff(3)
buff 2
recv the msg buff(2)
read the msg buff(2)
buff 1
recv the msg buff(1)
read the msg buff(1)
buff 0
recv the msg buff(0)

Process finished with exit code 0
```

