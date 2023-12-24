---
title: "C++ using使用"
date: 2023-07-22
thumbnailImagePosition: left
thumbnailImage: c++/base/using/using_thumb.jpg
coverImage: c++/base/using/using_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- c++
- 2023
- July
tags:
- 基础
- using
- 私有继承
showSocial: false
---

源码中出现很多关于私有继承的操作（默认不加public），但是通过using外部可以直接调用，这个是什么原理？


<!--more-->
# 1介绍

(1)父类的public成员在private/protected继承后，在派生类中就成了private/protected权限而不是public权限，子类对象就不能再调用父类的public成员

(2)可能在父类中有100个public成员，在子类对象中只需要访问1个成员，其余99个成员都不需要访问，合适的做法是将100个父类的public成员以private/protected权限继承到子类，然后使用using关键字对需要访问的那一个父类public成员进行权限修改

# 2demo

# 2.1使用动态机制突破private权限

通过引用来突破private关键字的方式

```c++
#include <iostream>
using namespace std;

class A
{
public:
	virtual void func() { cout << "func in A" << endl; }
};

class B : public A
{
private:
	virtual void func() { cout << "func in B" << endl; }
};

int main()
{
	B b;

	//b.func(); /* 错误：private函数对外不可见 */

	/* 通过虚函数机制，可以突破private的限制，实现对B中的func进行调用 */
	A &ra = b;
	ra.func(); /* 输出：func in B */
}
```

{{< image classes="fancybox center fig-100" src="/c++/base/using/using_1.png" thumbnail="/c++/base/using/using_1.png" title="">}}

因为class B继承自class A（**严格来说是public继承**），所以依据动态绑定规则，class A的引用`ra`可以绑定到class B的实例`b`上。之后，通过`ra`调用`func`函数，实际调用到的就是class B中的`func`。

## 2.2当做private来使用

如果把虚继承直接当做一个类内的private字段来操作，那就简单很多了

```c++
#include <iostream>
#include <string>
using namespace std;

class A : private string
{
public:
    A() : string("yangyang48") {}

    size_t GetSize1() const
    {
        return ((const string *)this)->size();
    }

    size_t GetSize2() const
    {
        return string::size();
    }
};

int main()
{
    A a;
    cout << a.GetSize1() << endl; /* 输出： 10 */
    cout << a.GetSize2() << endl; /* 输出： 10 */
}
```

这里说明下，我们操作的string，实际上是`basic_string`;

```c++
//string
using string    = basic_string<char>;
//basic_string.h
template<typename _CharT, typename _Traits, typename _Alloc>
class basic_string
{
public:
    size_type
    size() const _GLIBCXX_NOEXCEPT
    { return _M_string_length; }
};
```

这里是string是一个共有方法的size()，只是用到了私有继承的方式。

实际上上面的两个get方法，本质还是公有函数中调用私有函数，变相增加了两个get方法，但不使用公有get直接调用也是不行的。

> 领导（string）给马屁精（class A）传达了一些信息（公开了.empty()、.size()、.substr()等public接口），并告诉马屁精：「这些资料，你要原封不动地公开给其它同事（外部调用）
>
> 马屁精（class A）转头回到同事中又开始鸡毛当令箭了（将继承方式改为 private继承）。抠抠搜搜地公开了一些信息，还美其名曰“帮大家整理总结”（将size()封装到GetSize1/GetSize2中供外部调用）。
>
> 同事（外部调用）一看所谓的“整理总结”，就是糊了个封皮封底（class A公布的接口中，除了无条件调用string的size()接口，别的什么也没做），这个时候同事们（外部调用）不干了，马屁精拍马屁（保持private继承）可以，但能不能别给其他人添乱（不要给接口套层壳，直接让外部调用string的原始接口），马屁精（class A）一看只能用using方式来替换自己的整理，毕竟如果不使用私有继承，自己就没啥用了也不需要using。

C++还确实提供了这种机制：在class A中使用`using string::size;`可以直接将string的`size()`接口提升为public权限。即外部可以直接调用该接口，形式上等同于针对该接口使用了public继承（string提供的其余接口仍旧是private继承）。

```c++
#include <iostream>
#include <string>
using namespace std;

class A : private string
{
public:
    A() : string("yangyang48") {}
    using string::size;
};

int main() {
    A a;
    cout << a.size() << endl; /* 输出： 10 */
}
```

输出结果跟上面一样

```
10
```



## 2.3源码分析

在源码中，也会存在使用using来提升继承成员的访问权限

```c++
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
namespace android {
namespace scheduler {
    class Scheduler : impl::MessageQueue {
        using Impl = impl::MessageQueue;

    public:
        Scheduler(ICompositor&, ISchedulerCallback&, FeatureFlags);
        virtual ~Scheduler();
        ...
        using Impl::initVsync;
        using Impl::setInjector;
        using Impl::getScheduledFrameTime;
        using Impl::setDuration;
        using Impl::scheduleFrame;
    }; 
}
}

namespace android {
namespace impl {

class MessageQueue : public android::MessageQueue {
...
public:
    explicit MessageQueue(ICompositor&);

    void initVsync(scheduler::VSyncDispatch&, frametimeline::TokenManager&,
                   std::chrono::nanoseconds workDuration) override;
    void setDuration(std::chrono::nanoseconds workDuration) override;
    void setInjector(sp<EventThreadConnection>) override;
    void waitMessage() override;
    void postMessage(sp<MessageHandler>&&) override;
    void scheduleFrame() override;
    std::optional<Clock::time_point> getScheduledFrameTime() const override;
};
}    
}
```

上述Scheduler使用默认继承impl::MessageQueue，默认继承为private继承，也就是**当类的继承方式是私有继承时，基类中的公有和保护成员在派生类中变成私有成员，而基类的私有成员在派生类中不能直接访问**。

```c++
//frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::initScheduler(const sp<DisplayDevice>& display) {
    mScheduler->initVsync(mScheduler->getVsyncDispatch(), *mFrameTimeline->getTokenManager(),
                          configs.late.sfWorkDuration);
}
```

但是在源码中，发现使用到了initVsync，这个可以直接使用调用到基类impl::MessageQueue中的方法。

往前面看，class Scheduler定义了using Impl::initVsync;这就说明权限不一致了，把using可以修改子类继承自父类的成员权限成public

> (1)using可以修改子类继承自父类的成员权限成public，但并不是所有继承自父类的成员都可以修改成public权限；
>
> (2) 如果成员在父类中本来就是private权限，那在子类中是无法使用using声明成public权限的；
>
> 总结：using只是用来找回在继承中损失的权限，给部分成员开特例；



# 3总结

总的来说，使用using可以让私有继承中原本父类的公有方法，在子类中恢复到公有的方法，可以在外部直接调用，这样可以隔离部分私有继承中需要不公开的方法的一种方式。

# 参考

[[1] 正在起飞的蜗牛. 【C++入门】使用using重新定义继承的成员访问权限, 2022.](https://blog.csdn.net/weixin_42031299/article/details/127579689)

[[2] Renekton_bhk. C++的private并没有听起来那么“保密”（virtual）（using）, 2020.](https://blog.csdn.net/Renekton_bhk/article/details/104130622)