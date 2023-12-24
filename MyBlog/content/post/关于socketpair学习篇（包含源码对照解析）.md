---
title: "关于socketpair学习篇（包含源码对照解析）"
date: 2023-07-10
thumbnailImagePosition: left
thumbnailImage: socketpair/sp_thumb.jpg
coverImage: socketpair/sp_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- socketpair
- 2023
- July
tags:
- socket
- pipe
- BitTube
- epoll
- poll
- proto
- property_service
- CameraMetadata
- AudioGroup
showSocial: false
---

之前介绍了关于socket的用法，这里在此基础上再来看看跟socket类似的socketpair。


<!--more-->
# 0一个demo

这里以一个例子来说明socketpair的广泛性，这里先介绍一下BitTube类。这里只是简单介绍，具体关于BitTube的使用的原理可以点击[这里](https://www.cnblogs.com/roger-yu/p/16158539.html)。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_2.png" thumbnail="/socketpair/sp_2.png" title="">}}

## 0.1本进程多线程通信

{{< image classes="fancybox center fig-100" src="/socketpair/sp_1.png" thumbnail="/socketpair/sp_1.png" title="">}}



## 0.2跨进程通信

### 0.2.1发送句柄到其他进程

发送句柄的过程中，使用了fcntl的函数，是用来修改已经打开文件的属性的函数，这里起到了一个dup复制句柄的作用。

> fcntl介绍，具体可以点击[这里](https://blog.csdn.net/rikeyone/article/details/88828154)
>
> ```c
> #include <fcntl.h>
> int fcntl(int fd, int cmd, ...);
> ```
>
> fcntl是用来修改**已经打开文件的属性**的函数，包含5个功能：
>
> 1. **复制一个已有文件描述符**
>
>    功能和dup和dup2相同，对应的cmd：`F_DUPFD`、`F_DUPFD_CLOEXEC`。当使用这两个cmd时，需要传入第三个参数，fcntl返回复制后的文件描述符，此返回值是之前未被占用的描述符，并且必须一个大于等于第三个参数值。F_DUPFD命令要求返回的文件描述符会清除对应的FD_CLOEXEC标志；F_DUPFD_CLOEXEC要求设置新描述符的FD_CLOEXEC标志。
>
> 2. **获取、设置文件描述符标志**
>
>    对应的cmd：`F_GETFD`、`F_SETFD`。用于设置FD_CLOEXEC标志，此标志的含义是：当进程执行exec系统调用后此文件描述符会被自动关闭。
>
> 3. **获取、设置文件访问状态标志**
>
>    对应的cmd：`F_GETFL`、`F_SETFL`。获取当前打开文件的访问标志，设置对应的访问标志，一般常用来设置做非阻塞读写操作。
>
> 4. **获取、设置记录锁功能**
>
>    对应的cmd：`F_GETLK`、`F_SETLK`、`F_SETLKW`。
>
> 5. **获取、设置异步I/O所有权**
>
>    对应的cmd：`F_GETOWN`、`F_SETOWN`。
>    获取和设置用来接收SIGIO/SIGURG信号的进程id或者进程组id。返回对应的进程id或者进程组id取负值。





{{< image classes="fancybox center fig-100" src="/socketpair/sp_3.png" thumbnail="/socketpair/sp_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/socketpair/sp_4.png" thumbnail="/socketpair/sp_4.png" title="">}}

### 0.2.2从其他进程接收句柄

{{< image classes="fancybox center fig-100" src="/socketpair/sp_5.png" thumbnail="/socketpair/sp_5.png" title="">}}

{{< image classes="fancybox center fig-100" src="/socketpair/sp_6.png" thumbnail="/socketpair/sp_6.png" title="">}}

## 0.3总结

socketpair利用socket为双方建立了全双工的通信管道(communication pipe)。通过文件描述符的复用(dup/dup2)，可以传递socket handle到另一个进程，复用它并开启通信。

BitTube使用了Linux/Unix socket中的顺序数据包(sequenced packets，SOCK_SEQPACKET)，像SOCK_DGRAM，它只传送整包数据；又像SOCK_STREAM，面向连接且提供有序的数据报传送。

# 1socketpair

socketpair，跟socket类似，都为**套接字**。可以用于网络通信，也可以用于本机内的进程通信。

区别于管道pipe是半双工的，pipe两次才能实现全双工，使得代码复杂。socketpair直接就可以实现全双工，socketpair对两个文件描述符中的任何一个都可读和可写，而pipe是一个读，一个写。

# 2Linux socketpair

创建一对连接的套接字

## 2.1函数原型

```c
#include <sys/types.h>          
#include <sys/socket.h>
int socketpair(int domain, int type, int protocol, int sv[2]);
```

- **domain**：协议族，可以是AF_UNIX或AF_LOCAL。
- **type**：套接字类型，可以是SOCK_STREAM或SOCK_DGRAM。
- **protocol**：协议类型，可以是0或IPPROTO_TCP。
- **sv**：用于存储返回的套接字描述符的数组，其中sv[0]表示第一个套接字描述符，sv[1]表示第二个套接字描述符。

## 2.2描述

socketpair调用使用可选指定的套接字类型，在指定的域中以指定的类型创建一对未命名的已连接套接字。有关这些参数的更多详细信息，请参见socket，点击[这里](https://yangyang48.github.io/2023/01/socket%E7%AE%80%E5%8D%95%E5%AE%9E%E7%8E%B0linux%E9%99%84android%E6%BA%90%E7%A0%81/)。



## 2.3返回值

返回值如果成功，则返回0。如果出现错误，则返回-1，并适当地设置errno。

|      ERRORNO      |                  解释                  |
| :---------------: | :------------------------------------: |
|  `EAFNOSUPPORT`   |    在这台计算机上不支持指定的地址族    |
|     `EFAULT`      |  地址sv没有指定进程地址空间的有效部分  |
|     `EMFILE`      | 已达到每个进程打开的文件描述符数量限制 |
|     `ENFILE`      |     已达到打开文件总数的全系统限制     |
|   `EOPNOTSUPP`    |      指定的协议不支持创建套接字对      |
| `EPROTONOSUPPORT` |        这台机器不支持指定的协议        |



# 3源码中的socketpair

本文主要介绍三种套接字类型，**SOCK_STREAM, SOCK_SEQPACKET,SOCK_DGRAM**。但不代表socketpair只支持这三种，其余的套接字类型可以根据兴趣爱好自我探索。

## 3.1属性系统源码

属性系统中用到了epoll加上**SOCK_SEQPACKET**套接字类型的socketpair。这个源码更详细的可以点击[这里](https://yangyang48.github.io/2022/08/android-property%E8%AF%A6%E8%A7%A3/)。

### 3.1.1创建socketpair

创建socketpair，其中套接字类型是`SOCK_SEQPACKET`，另外sockets[0]为接收端，sockets[1]为发送端

```c++
//system/core/init/property_service.cpp
void StartPropertyService(int* epoll_socket) {
    InitPropertySet("ro.property_service.version", "2");

    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, sockets) != 0) {
        PLOG(FATAL) << "Failed to socketpair() between property_service and init";
    }
    //epoll_socket和from_init_socket都为接收端,会返回到init.cpp中的StartPropertyService
    *epoll_socket = from_init_socket = sockets[0];
    //init_socket为发送端
    init_socket = sockets[1];
    StartSendingMessages();
    //这里创建了套接字类型为SOCK_STREAM的TCP流的socket，句柄为property_set_fd
    if (auto result = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                   /*passcred=*/false, /*should_listen=*/false, 0666, /*uid=*/0,
                                   /*gid=*/0, /*socketcon=*/{});
        result.ok()) {
        property_set_fd = *result;
    } else {
        LOG(FATAL) << "start_property_service socket creation failed: " << result.error();
    }

    listen(property_set_fd, 8);

    auto new_thread = std::thread{PropertyServiceThread};
    property_service_thread.swap(new_thread);
}
```

{{< image classes="fancybox center fig-100" src="/socketpair/sp_7.png" thumbnail="/socketpair/sp_7.png" title="">}}

### 3.1.1属性系统客户端

属性的服务端是在启动流程中开启的，调用StartPropertyService，并传入一个句柄property_fd，很显然这个句柄获取得到是socketpait对应的socket[0]

```c++
//system/core/init/init.cpp
int SecondStageMain(int argc, char** argv) {
    ...
    StartPropertyService(&property_fd);   
}
```

但是这里获取到了property_fd，承载着客户端的操作。

这里调用SendLoadPersistentPropertiesMessage，可以发现关于property_fd从客户端发送到socketpair另外一端

```c++
//system/core/init/init.cpp
void SendLoadPersistentPropertiesMessage() {
    //这里使用的跨平台的protobuf，会被解析成c++类型
    auto init_message = InitMessage{};
    init_message.set_load_persistent_properties(true);
    if (auto result = SendMessage(property_fd, init_message); !result.ok()) {
        LOG(ERROR) << "Failed to send load persistent properties message: " << result.error();
    }
}
```

> 对应的protobuf文件
>
> ```protobuf
> //system/core/init/property_service.proto
> message InitMessage {
>     oneof msg {
>         bool load_persistent_properties = 1;
>         bool stop_sending_messages = 2;
>         bool start_sending_messages = 3;
>     };
> }
> ```



其中SendLoadPersistentPropertiesMessage并不是直接启动的，是通过rc文件解析之后Action类型执行的

```c++
//system/core/init/builtins.cpp
const BuiltinFunctionMap& GetBuiltinFunctionMap() {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    // clang-format off
    static const BuiltinFunctionMap builtin_functions = {
		...
        {"load_persist_props",      {0,     0,    {false,  do_load_persist_props}}},
		...
    };
    // clang-format on
    return builtin_functions;
}
```

Action类型执行对应的函数do_load_persist_props

```c++
//system/core/init/builtins.cpp
static Result<void> do_load_persist_props(const BuiltinArguments& args) {
    SendLoadPersistentPropertiesMessage();
	//从这里可以看出实际上常量属性加载完成之后才算完成这一步
    start_waiting_for_property("ro.persistent_properties.ready", "true");
    return {};
}
```



> `proto`说明
>
> 关于`proto`之前的Android属性系统中文章中提到过，但没有具体深入，当然本文也不会特别深入。如果需要进一步了解语法规则，可以点击[这里](http://t.zoukankan.com/GarfieldEr007-p-10113328.html)。
>
> `Protobuf` 是一种与平台无关、语言无关、可扩展且轻便高效的**序列化数据结构**的协议，可以用于网络通信和**数据存储**。而且由于和语言无关，`proto`文件既可以转化成`c++`文件，也可以转化成`java`，`python`这类的文件。当然在源码中，可以直接理解为定义了一种数据类，专门用于序列化存储。
>
> 目前我们定义的`proto`文件，比如上面的`property_service.proto`，可以在下面的源码路径中找到
>
> ```text
> 1//其中xxx为具体的架构，有arm架构，arm架构等
> 2/out/soong/.intermediates/system/core/init/libinit/android_xxx_static/gen/proto/system/core/init/property_service.pb.h
> 3/out/soong/.intermediates/system/core/init/libinit/android_xxx_static/gen/proto/system/core/init/property_service.pb.cc
> ```
>
> 具体的规则如下，所指定的消息字段修饰符必须是如下之一：
>
> - required：一个格式良好的消息一定要含有1个这种字段。表示该值是必须要设置的；
> - optional：消息格式中该字段可以有0个或1个值（不超过1个）。
> - repeated：在一个格式良好的消息中，这种字段可以重复任意多次（包括0次）。重复的值的顺序会被保留。表示该值可以重复，相当于java中的List。
>
> 关于`Oneof`
>
> 如果你的消息中有很多可选字段， 并且同时至多一个字段会被设置， 你可以加强这个行为，使用`oneof`特性节省内存.
>
> `Oneof`字段就像可选字段， 除了它们会共享内存， 至多一个字段会被设置。 设置其中一个字段会清除其它`oneof`字段。 你可以使用`case()`或者`WhichOneof()` 方法检查哪个`oneof`字段被设置， 看你使用什么语言了.
>
> 在产生的代码中, `oneof`字段拥有同样的 `getters` 和`setters`， 就像正常的可选字段一样. 也有一个特殊的方法来检查到底那个字段被设置. 你可以在相应的语言API中找到`oneof API`介绍.
>
> ```protobuf
> message SampleMessage {
> 	oneof test_oneof {
> 		string name = 4;
> 		SubMessage sub_message = 9;
> 	}
> }
> ```
>
> 设置`oneof`会自动清楚其它`oneof`字段的值. 所以设置多次后，只有最后一次设置的字段有值.
>
> ```protobuf
> SampleMessage message;
> message.set_name("name");
> CHECK(message.has_name());
> message.mutable_sub_message(); // Will clear name field.
> CHECK(!message.has_name());
> ```

根据上面可以看到protobuf文件，实际上我们获取了序列化的内容，即load_persistent_properties为true。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_8.png" thumbnail="/socketpair/sp_8.png" title="">}}

### 3.1.2属性系统服务端

服务端的话就是上述的线程，property_service_thread，对应的线程函数为PropertyServiceThread

```c++
//system/core/init/property_service.cpp
static void PropertyServiceThread() {
    //源码封装的epoll
    Epoll epoll;
    //就是创造epoll_fd句柄
    if (auto result = epoll.Open(); !result.ok()) {
        LOG(FATAL) << result.error();
    }
	//将property_set_fd注册到epoll中，回调函数为handle_property_set_fd
    if (auto result = epoll.RegisterHandler(property_set_fd, handle_property_set_fd);
        !result.ok()) {
        LOG(FATAL) << result.error();
    }
	//将init_socket注册到epoll中，回调函数为HandleInitSocket
    if (auto result = epoll.RegisterHandler(init_socket, HandleInitSocket); !result.ok()) {
        LOG(FATAL) << result.error();
    }
	//循环等待结果
    while (true) {
        //这里默认是无限等待，一直阻塞，直到有事件来的时候返回，并执行对应的回调函数
        auto pending_functions = epoll.Wait(std::nullopt);
        if (!pending_functions.ok()) {
            LOG(ERROR) << pending_functions.error();
        } else {
            for (const auto& function : *pending_functions) {
                (*function)();
            }
        }
    }
}
```

可以看到这里面又涉及到了epoll，不过这个epoll在源码中重新封装成了Epoll

```c++
//system/core/init/epoll.h
class Epoll {
  public:
    Epoll();
	//回调函数Handler
    typedef std::function<void()> Handler;

    Result<void> Open();
    //注册的默认事件为EPOLLIN
    Result<void> RegisterHandler(int fd, Handler handler, uint32_t events = EPOLLIN);
    Result<void> UnregisterHandler(int fd);
    Result<std::vector<std::shared_ptr<Handler>>> Wait(
            std::optional<std::chrono::milliseconds> timeout);

  private:
    //将回调函数的事件封装到一起
    struct Info {
        std::shared_ptr<Handler> handler;
        uint32_t events;
    };

    android::base::unique_fd epoll_fd_;
    std::map<int, Info> epoll_handlers_;
};
```

```c++
//system/core/init/epoll.cpp
Result<void> Epoll::Open() {
    epoll_fd_.reset(epoll_create1(EPOLL_CLOEXEC));
    ...
}

Result<void> Epoll::RegisterHandler(int fd, Handler handler, uint32_t events) {
    Info info;
    info.events = events;
    info.handler = std::make_shared<decltype(handler)>(std::move(handler));
    
    auto [it, inserted] = epoll_handlers_.emplace(fd, std::move(info));
    epoll_event ev;
    ev.events = events;
    ev.data.ptr = reinterpret_cast<void*>(&it->second);
	epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev);
}

Result<std::vector<std::shared_ptr<Epoll::Handler>>> Epoll::Wait(
        std::optional<std::chrono::milliseconds> timeout) {
    int timeout_ms = -1;
    
    const auto max_events = epoll_handlers_.size();
    epoll_event ev[max_events];
    //默认无限阻塞，直到有EPOLLIN事件过来
    auto num_events = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd_, ev, max_events, timeout_ms));
    
    std::vector<std::shared_ptr<Handler>> pending_functions;
    for (int i = 0; i < num_events; ++i) {
        //取出事件中的ptr，即对应的info指针
        auto& info = *reinterpret_cast<Info*>(ev[i].data.ptr);
        //将对应事件放入到数组中，通过数组来依次排队输出
        pending_functions.emplace_back(info.handler);
    }

    return pending_functions;
}
```

{{< image classes="fancybox center fig-100" src="/socketpair/sp_9.png" thumbnail="/socketpair/sp_9.png" title="">}}

最终会回调到两个函数，这两个函数的先后顺序是由于哪个事件先触发先调用的原则，会被放在**pending_functions**队列中。

#### 3.1.2.1handle_property_set_fd

这里回调实际上是对应socket，而不是socketpair，这里更多的是IO多路复用的经典场景，监听socket句柄，有新的连接进来了就会回调到这个函数。

```c++
//system/core/init/property_service.cpp
static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout = 2000; /* ms */
	//接收客户端的句柄，服务端这里与之连接的句柄为s，
    //其实这里可以把s也添加到epoll中，但是源码中并没有这么操作，原因是更高的处理效率,而是用了poll机制
    int s = accept4(property_set_fd, nullptr, nullptr, SOCK_CLOEXEC);
    if (s == -1) {
        return;
    }
	//这里包含权限问题，pid，uid，gid，
    ucred cr;
    socklen_t cr_size = sizeof(cr);
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
        ...
    }

    SocketConnection socket(s, cr);
    uint32_t timeout_ms = kDefaultSocketTimeout;

    uint32_t cmd = 0;
    if (!socket.RecvUint32(&cmd, &timeout_ms)) {
        ...
        return;
    }

    switch (cmd) {
    ...
    //这里默认使用PROP_MSG_SETPROP2 0x00020001
    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        if (!socket.RecvString(&name, &timeout_ms) ||
            !socket.RecvString(&value, &timeout_ms)) {
          ...
        }

        std::string source_context;
        if (!socket.GetSourceContext(&source_context)) {
            ...
        }

        const auto& cr = socket.cred();
        std::string error;
        uint32_t result = HandlePropertySet(name, value, source_context, cr, &socket, &error);
        ...
        socket.SendUint32(result);
        break;
      }

    default:
        ...
    }
}
```

这里还涉及到了系统管理的id问题

> 这里只是简单的介绍一下UID，PID和GID的关系，具体内容可以点击[这里](https://blog.csdn.net/xk7298/article/details/102291785)。
>
> 1. **UID**
>
>    在Linux中用户的概念分为：**普通用户、根用户和系统用户**。
>
>    | Linux用户 |                           解释说明                           |
>    | :-------: | :----------------------------------------------------------: |
>    | 普通用户  | 表示平时使用的用户概念，在使用Linux时，需要通过用户名和密码登录，获取该用户相应的权限，**其权限具体表现在对系统中文件的增删改查和命令执行的限制**，不同用户具有不同的权限设置，其**UID通常大于500** |
>    |  根用户   | 该用户就是ROOT用户，**其UID为0**，可以对系统中任何文件进行增删改查处理，执行任何命令，因此ROOT用户极其危险，如操作不当，会导致系统彻底崩掉 |
>    | 系统用户  | 该用户是系统虚拟出的用户概念，不对使用者开发的用户，**其UID范围为1-499**，例如运行MySQL数据库服务时，需要使用系统用户mysql来运行mysqld进程 |
>
>    
>
> 2. **PID**
>
>    系统在程序运行时，会为**每个可执行程序分配一个唯一的进程ID（PID）**，PID的直接作用是为了表明该程序所拥有的文件操作权限，不同的可执行程序运行时互不影响，相互之间的数据访问具有权限限制。
>
> 3. **GID**
>
>    GID顾名思义就是对于UID的封装处理，就是**包含多个UID的意思**，实际上在**Linux下每个UID都对应着一个GID**。设计GID是为了便于对系统的统一管理，例如增加某个文件的用户权限时，只对admin组的用户开放，那么在分配权限时，只需对该组分配，其组下的所有用户均获取权限。同样在删除时，也便于统一操作。
>
>    除了UID和GID外，其还包括其扩展的有效的用户、组（euid、egid）、文件系统的用户、组（fsuid、fsgid）和保存的设置用户、组（suid、sgid）等。
>
> - 不同的应用具有**唯一的UID**，**同一个UID可具有不同的PID**；
> - 针对不同的PID之间数据的暴露可采用私有暴露和权限暴露，针对不同的UID之间可通过完全暴露的方式；
> - 如果一个应用是系统应用，则不需要其他应用暴露，便可直接访问该应用的数据。

另外上面的socket还存在封装，封装成了对应的SocketConnection

```c++
//system/core/init/property_service.cpp
class SocketConnection {
  public:
    SocketConnection(int socket, const ucred& cred) : socket_(socket), cred_(cred) {}
	...
    //以接收string为例，另外的int和char都是类似    
    bool RecvString(std::string* value, uint32_t* timeout_ms) {
        uint32_t len = 0;
        /
        if (!RecvUint32(&len, timeout_ms)) {
            return false;
        }

        std::vector<char> chars(len);
        if (!RecvChars(&chars[0], len, timeout_ms)) {
            return false;
        }

        *value = std::string(&chars[0], len);
        return true;
    }

    bool SendUint32(uint32_t value) {
        if (!socket_.ok()) {
            return true;
        }
        //发送的话，直接发送
        int result = TEMP_FAILURE_RETRY(send(socket_, &value, sizeof(value), 0));
        return result == sizeof(value);
    }

    const ucred& cred() { return cred_; }

  private:
    //接收的过程用到了poll机制,这里传递进来的timeout_ms为2s
    bool PollIn(uint32_t* timeout_ms) {
        struct pollfd ufds[1];
        ufds[0].fd = socket_;
        ufds[0].events = POLLIN;
        ufds[0].revents = 0;
        while (*timeout_ms > 0) {
            auto start_time = std::chrono::steady_clock::now();
            //对socket_，其实就是s进行监听，是否会超时。只有fds中准备好读写，返回值nr大于0为s
            int nr = poll(ufds, 1, *timeout_ms);
            auto now = std::chrono::steady_clock::now();
            auto time_elapsed =
                std::chrono::duration_cast<std::chrono::milliseconds>(now - start_time);
            uint64_t millis = time_elapsed.count();
            *timeout_ms = (millis > *timeout_ms) ? 0 : *timeout_ms - millis;

            if (nr > 0) {
                return true;
            }

            if (nr == 0) {
                // Timeout
                break;
            }

            if (nr < 0 && errno != EINTR) {
                PLOG(ERROR) << "sys_prop: error waiting for uid " << cred_.uid
                            << " to send property message";
                return false;
            } else {  // errno == EINTR
                // Timer rounds milliseconds down in case of EINTR we want it to be rounded up
                // to avoid slowing init down by causing EINTR with under millisecond timeout.
                if (*timeout_ms > 0) {
                    --(*timeout_ms);
                }
            }
        }

        LOG(ERROR) << "sys_prop: timeout waiting for uid " << cred_.uid
                   << " to send property message.";
        return false;
    }
	
    bool RecvFully(void* data_ptr, size_t size, uint32_t* timeout_ms) {
        size_t bytes_left = size;
        char* data = static_cast<char*>(data_ptr);
        while (*timeout_ms > 0 && bytes_left > 0) {
            //用于判断接收是否超时2s，这里是不允许超时的
            if (!PollIn(timeout_ms)) {
                return false;
            }
			//这里实际上是流的形式，所以或循环去读取的操作
            int result = TEMP_FAILURE_RETRY(recv(socket_, data, bytes_left, MSG_DONTWAIT));
            if (result <= 0) {
                PLOG(ERROR) << "sys_prop: recv error";
                return false;
            }

            bytes_left -= result;
            data += result;
        }

        if (bytes_left != 0) {
            LOG(ERROR) << "sys_prop: recv data is not properly obtained.";
        }

        return bytes_left == 0;
    }

    unique_fd socket_;
    ucred cred_;

    DISALLOW_IMPLICIT_CONSTRUCTORS(SocketConnection);
};
```

内容比较多

接收。以接收string为例，另外的int和char都是类似，循环recv接收数据。先判断传输的string长度，长度限制为0-2^16-1。然后接收的过程以char类型接收，接收完成之后再转成string类。

发送。已发送int为例，这里发送直接通过send发送数据。

> **poll机制**
>
> poll是Linux的`事件轮询机制`函数，每个进程都可以管理一个`pollfd`队列，由`poll函数`进行事件注册和查询。内核将用户的`fds结构体数组`拷贝到内核中。当有事件发生时，再将所有事件都返回到`fds结构体数组`中，poll只返回已就绪事件的个数，所以用户要操作就绪事件就要用`轮询`的方法。
>
> 1）函数原型
>
> ```c++
> #include <poll.h>
> int poll(struct pollfd* fds, nfds_t nfds, int timeout);
> ```
>
> - **fds：**是一个struct pollfd类型的指针，用于存放需要检测其状态的socket描述符
> - **nfds：**是nfd_t类型的参数，用于标记fds数组中结构体元素的数量
> - **timeout：**没有接受事件时等待的事件，单位毫秒，若值为-1，则永远不会超时
>
> 
>
> 2）pollfd结构体
>
> ```c++
> struct pollfd
> {
> 	int fd; 
> 	short events;
> 	short revents; 
> }
> ```
>
> - **fd：**文件描述符
> - **events**：等待发生的事件类型
> - **revents：**检测之后返回的事件，当某个文件描述符有变化时，值就不为空
>
> |   事件标识   |           说明           |
> | :----------: | :----------------------: |
> |   `POLLIN`   |  普通或优先级带数据可读  |
> | `POLLRDNORM` |       普通数据可读       |
> | `POLLRDBAND` |     优先级带数据可读     |
> |  `POLLPRI`   |     高优先级数据可读     |
> |  `POLLOUT`   |       普通数据可写       |
> | `POLLWRNORM` | 普通数据可写不会导致阻塞 |
> |  `POLLERR`   |         发生错误         |
> |  `POLLHUP`   |         发生挂起         |
> |  `POLLNVAL`  | 描述字不是一个打开的文件 |
>
> 3）poll返回值
>
> poll机制会判断fds中的文件是否满足条件，如果休眠时间内条件满足则会唤醒进程；超过休眠时间，条件一直不满足则自动唤醒。
>
> - **返回值>0**：fds中准备好读写，或出错状态的那些socket描述符
> - **返回值=0：**fds中没有socket描述符需要读写或出错；此时poll超时，时长为timeout
> - **返回值=-1：**调用失败

poll机制也是IO多路复用的一种方式，这里主要是承接连接的客户端的socket，对应服务端会有一个连接socket句柄s，对其poll轮询机制，可以更好的去获取对应的TCP流。

这里的s接收到的数据，实际上是客户端发送的数据。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_11.png" thumbnail="/socketpair/sp_11.png" title="">}}

#### 3.1.2.2HandleInitSocket

```c++
//system/core/init/property_service.cpp
static void HandleInitSocket() {
    //处理socketpair对端传过来的数据
    auto message = ReadMessage(init_socket);
    if (!message.ok()) {
        LOG(ERROR) << "Could not read message from init_dedicated_recv_socket: " << message.error();
        return;
    }
	//初始化proto数据
    auto init_message = InitMessage{};
    //将序列化数据转化成string类
    if (!init_message.ParseFromString(*message)) {
        LOG(ERROR) << "Could not parse message from init";
        return;
    }
	//判断属性系统传过来的数据类型，是否为kLoadPersistentProperties
    switch (init_message.msg_case()) {
        case InitMessage::kLoadPersistentProperties: {
            load_override_properties();
            // 导入persist属性，并且初始化
            auto persistent_properties = LoadPersistentProperties();
            for (const auto& persistent_property_record : persistent_properties.properties()) {
                InitPropertySet(persistent_property_record.name(),
                                persistent_property_record.value());
            }
            InitPropertySet("ro.persistent_properties.ready", "true");
            persistent_properties_loaded = true;
            break;
        }
        default:
            LOG(ERROR) << "Unknown message type from init: " << init_message.msg_case();
    }
}
```

上面主要做了几件事

1. 处理socketpair对端传过来的数据
2. 将proto序列化数据转化成string类
3. 判断属性系统传过来的数据类型，如果是就加载并初始化persist属性

ReadMessage

```c++
//system/core/init/proto_utils.h
constexpr size_t kBufferSize = 4096;

inline Result<std::string> ReadMessage(int socket) {
    char buffer[kBufferSize] = {};
    auto result = TEMP_FAILURE_RETRY(recv(socket, buffer, sizeof(buffer), 0));
    if (result == 0) {
        return Error();
    } else if (result < 0) {
        return ErrnoError();
    }
    return std::string(buffer, result);
}
```

这个具体数据，实际上就是对应的msg_case为InitMessage::kLoadPersistentProperties的数据。



### 3.1.3总结

实际上，这里的socket的作用意图很明显，将**Init进程**和**非Init进程**的隔离开来，非Init进程操作属性的写入需要转到Init进程去操作，而非Init进程和Init进程通信这里用到了本地的socket通信。这样，属性的管控者就是Init进程，可以做到**过滤隔离**作用。

除此之外，这里还涉及到了epoll、poll、protobuf、socket和socketpair等概念，充分理解和掌握这些基本概念之后，对于源码的阅读可以理解的更加深入。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_10.png" thumbnail="/socketpair/sp_10.png" title="">}}

## 3.2Camera dump数据源码

Camera dump操作中用到了**SOCK_STREAM**套接字类型的socketpair。

### 3.2.1创建socketpair

```c++
//frameworks/base/core/jni/android_hardware_camera2_CameraMetadata.cpp
static void CameraMetadata_dump(JNIEnv *env, jclass thiz, jlong ptr) {
    ALOGV("%s", __FUNCTION__);
    CameraMetadata* metadata = CameraMetadata_getPointerThrow(env, ptr);
    if (metadata == NULL) {
        return;
    }
	int writeFd, readFd;
    {
        int sv[2];
        //这里创建socketpair，使用TCP的形式，一端发送，一端接受
        if (socketpair(AF_LOCAL, SOCK_STREAM, /*protocol*/0, &sv[0]) < 0) {
            jniThrowExceptionFmt(env, "java/io/IOException",
                    "Failed to create socketpair (errno = %#x, message = '%s')",
                    errno, strerror(errno));
            return;
        }
        writeFd = sv[0];
        readFd = sv[1];
    }

    pthread_t writeThread;
    DumpMetadataParams params = {
        writeFd,
        metadata
    };
	//开启write线程，作为写入段
    {
        int threadRet = pthread_create(&writeThread, /*attr*/NULL,
                CameraMetadata_writeMetadataThread, (void*)&params);

        if (threadRet != 0) {
            close(writeFd);
            close(readFd);

            jniThrowExceptionFmt(env, "java/io/IOException",
                    "Failed to create thread for writing (errno = %#x, message = '%s')",
                    threadRet, strerror(threadRet));
            return;
        }
    }
	...

    int res;

    // 等待数据完成后，退出线程
    if ((res = pthread_join(writeThread, /*retval*/NULL)) != 0) {
        ALOGE("%s: Failed to join thread (errno = %#x, message = '%s')",
                __FUNCTION__, res, strerror(res));
    }
}
```



### 3.2.2发送端

子线程用来发送数据

```c++
//frameworks/base/core/jni/android_hardware_camera2_CameraMetadata.cpp
struct DumpMetadataParams {
    int writeFd;
    const CameraMetadata* metadata;
};

static void* CameraMetadata_writeMetadataThread(void* arg) {
    DumpMetadataParams* p = static_cast<DumpMetadataParams*>(arg);

    p->metadata->dump(p->writeFd, /*verbosity*/2);

    if (close(p->writeFd) < 0) {
        ALOGE("%s: Failed to close writeFd (errno = %#x, message = '%s')",
                __FUNCTION__, errno, strerror(errno));
    }

    return NULL;
}
```

这里的p为DumpMetadataParams是结构体类型，实际上是CameraMetadata数据dump

```c++
//frameworks/av/camera/CameraMetadata.cpp
camera_metadata_t *mBuffer;
void dump(int fd, int verbosity = 1, int indentation = 0) const;
void CameraMetadata::dump(int fd, int verbosity, int indentation) const {
    dump_indented_camera_metadata(mBuffer, fd, verbosity, indentation);
}
```

调用到dump_indented_camera_metadata

```c++
//system/media/camera/src/camera_metadata.c
void dump_indented_camera_metadata(const camera_metadata_t *metadata,
        int fd,
        int verbosity,
        int indentation) {
    unsigned int i;
    dprintf(fd,
            "%*sDumping camera metadata array: %" PRIu32 " / %" PRIu32 " entries, "
            "%" PRIu32 " / %" PRIu32 " bytes of extra data.\n", indentation, "",
            metadata->entry_count, metadata->entry_capacity,
            metadata->data_count, metadata->data_capacity);
    dprintf(fd, "%*sVersion: %d, Flags: %08x\n",
            indentation + 2, "",
            metadata->version, metadata->flags);
    camera_metadata_buffer_entry_t *entry = get_entries(metadata);
    for (i=0; i < metadata->entry_count; i++, entry++) {

        const char *tag_name, *tag_section;
        tag_section = get_local_camera_metadata_section_name(entry->tag, metadata);
        if (tag_section == NULL) {
            tag_section = "unknownSection";
        }
        tag_name = get_local_camera_metadata_tag_name(entry->tag, metadata);
        if (tag_name == NULL) {
            tag_name = "unknownTag";
        }
        const char *type_name;
        if (entry->type >= NUM_TYPES) {
            type_name = "unknown";
        } else {
            type_name = camera_metadata_type_names[entry->type];
        }
        dprintf(fd, "%*s%s.%s (%05x): %s[%" PRIu32 "]\n",
             indentation + 2, "",
             tag_section,
             tag_name,
             entry->tag,
             type_name,
             entry->count);

        if (verbosity < 1) continue;

        if (entry->type >= NUM_TYPES) continue;

        size_t type_size = camera_metadata_type_size[entry->type];
        uint8_t *data_ptr;
        if ( type_size * entry->count > 4 ) {
            if (entry->data.offset >= metadata->data_count) {
                ALOGE("%s: Malformed entry data offset: %" PRIu32 " (max %" PRIu32 ")",
                        __FUNCTION__,
                        entry->data.offset,
                        metadata->data_count);
                continue;
            }
            data_ptr = get_data(metadata) + entry->data.offset;
        } else {
            data_ptr = entry->data.value;
        }
        int count = entry->count;
        if (verbosity < 2 && count > 16) count = 16;

        print_data(fd, data_ptr, entry->tag, entry->type, count, indentation);
    }
}
```

可以看到这里没有使用相应的write去发送，而是选择使用dprintf的方式来发送。

> 关于IO函数比较，write和dprintf区别
>
> **write**
>
> 向已打开文件描述符. 将缓存buf内容写count个字节到fd指向的文件. **buf不必以null终结符结尾**.
>
> ```c
> #include <unistd.h>
> ssize_t write(int fd, const void *buf, size_t count);
> ```
>
> 示例: 向stdout写入所有缓存字符
>
> ```c
> char buf[2] = {'a', 'b'};
> int n = write(STDOUT_FILENO, buf, sizeof(buf));
> if (n < 0) {
>  perror("write error");
>  exit(1);
> }
> ```
>
> **dprintf**
>
> 将格式化串输出到已打开文件描述符, 其他同printf.
>
> ```c
> #include <stdio.h>
> int sprintf(char *str, const char *format, ...);
> ```
>
> 示例: 向标准输出(默认已打开, 文件描述符 = STDOUT_FILENO)输出
>
> ```c
> int age = 20;
> dprintf(STDOUT_FILENO, "my age is %d\n", age);
> ```
>
> **区别**
>
> write函数和dprintf函数，都可以实现将字符串写入文件，但两者的区别在于，write会写入字符串中的NULL字符，因而以vim打开文件后会出现少量乱码，dprintf则不会。

### 3.2.3接收端

```c++
//frameworks/base/core/jni/android_hardware_camera2_CameraMetadata.cpp
static void CameraMetadata_dump(JNIEnv *env, jclass thiz, jlong ptr) {
	//当前线程读取数据
    {
        char out[] = {'\0', '\0'}; 
        String8 logLine;

        //每次读取一个数据，当读取到换行符的时候，就把当前缓冲区数据，打印出来，直到全部读完为止
        ssize_t res;
        while ((res = TEMP_FAILURE_RETRY(read(readFd, &out[0], /*count*/1))) > 0) {
            if (out[0] == '\n') {
                ALOGD("%s", logLine.string());
                logLine.clear();
            } else {
                logLine.append(out);
            }
        }

        ALOGD("%s", logLine.string());
        close(readFd);
    }
    ...
}
```

### 3.2.4总结

其实这一块内容比较简单，主要这里可能涉及到了Camera相关的CameraMetadata数据结构，这个比较复杂，具体关于CameraMetadata可以在参考中查找，这里不展开说明。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_12.png" thumbnail="/socketpair/sp_12.png" title="">}}

## 3.3AudioGroup相关源码

Audio相关操作中用到了**SOCK_DGRAM**套接字类型的socketpair。

### 3.3.1创建socketpair

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
bool AudioGroup::set(int sampleRate, int sampleCount)
{
    //这里也涉及到了epoll机制
    mEventQueue = epoll_create1(EPOLL_CLOEXEC);
    if (mEventQueue == -1) {
        ALOGE("epoll_create1: %s", strerror(errno));
        return false;
    }

    mSampleRate = sampleRate;
    mSampleCount = sampleCount;

    // 创建socketpair，是UDP的形式
    int pair[2];
    if (socketpair(AF_UNIX, SOCK_DGRAM, 0, pair)) {
        ALOGE("socketpair: %s", strerror(errno));
        return false;
    }
    //会在线程中使用
    mDeviceSocket = pair[0];

    // Create device stream.
    mChain = new AudioStream;
    //把另一端发给mChain保存
    if (!mChain->set(AudioStream::NORMAL, pair[1], NULL, NULL,
        sampleRate, sampleCount, -1, -1)) {
        close(pair[1]);
        ALOGE("cannot initialize device stream");
        return false;
    }

    // Give device socket a reasonable timeout.
    timeval tv;
    tv.tv_sec = 0;
    tv.tv_usec = 1000 * sampleCount / sampleRate * 500;
    //设置接收时间为微妙
    if (setsockopt(pair[0], SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv))) {
        ALOGE("setsockopt: %s", strerror(errno));
        return false;
    }

    // Add device stream into event queue.
    epoll_event event;
    event.events = EPOLLIN;
    event.data.ptr = mChain;
    //将pair[1]加入到epoll中，当发生变化，回调给mChain
    if (epoll_ctl(mEventQueue, EPOLL_CTL_ADD, pair[1], &event)) {
        ALOGE("epoll_ctl: %s", strerror(errno));
        return false;
    }

    // Anything else?
    ALOGD("stream[%d] joins group[%d]", pair[1], pair[0]);
    return true;
}
```

AudioStream::set

这里是一些初始化操作，包括模式，传输头的魔数

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
bool AudioStream::set(int mode, int socket, sockaddr_storage *remote,
    AudioCodec *codec, int sampleRate, int sampleCount,
    int codecType, int dtmfType)
{
    if (mode < 0 || mode > LAST_MODE) {
        return false;
    }
    mMode = mode;

    mCodecMagic = (0x8000 | codecType) << 16;
    mDtmfMagic = (dtmfType == -1) ? 0 : (0x8000 | dtmfType) << 16;
	...
    // 传入的pair[1]赋值给mSocket
    mSocket = socket;
 	...
    return true;
}
```

pair[0]赋值给了mDeviceSocket，pair[1]最终加入到了epoll中，等到写入操作触发。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_13.png" thumbnail="/socketpair/sp_13.png" title="">}}

### 3.3.2发送端

发送端主要是NetworkThread线程中的encode和decode

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
bool AudioGroup::NetworkThread::threadLoop()
{
    AudioStream *chain = mGroup->mChain;
    int tick = elapsedRealtime();
    int deadline = tick + 10;
    int count = 0;
	//根据mChain数量来确定epoll等待fd数量
    for (AudioStream *stream = chain; stream; stream = stream->mNext) {
        if (tick - stream->mTick >= 0) {
            //[1]首先编码传输
            stream->encode(tick, chain);
        }
        if (deadline - stream->mTick > 0) {
            deadline = stream->mTick;
        }
        ++count;
    }

    int event = mGroup->mDtmfEvent;
    if (event != -1) {
        for (AudioStream *stream = chain; stream; stream = stream->mNext) {
            stream->sendDtmf(event);
        }
        mGroup->mDtmfEvent = -1;
    }

    deadline -= tick;
    if (deadline < 1) {
        deadline = 1;
    }

    epoll_event events[count];
    //等待时间至少是1s
    count = epoll_wait(mGroup->mEventQueue, events, count, deadline);
    if (count == -1) {
        ALOGE("epoll_wait: %s", strerror(errno));
        return false;
    }
    //有写入事件发生，调用对应的ptr->decode,实际是mChain->decode回调函数
    //[2]然后解码传输
    for (int i = 0; i < count; ++i) {
        ((AudioStream *)events[i].data.ptr)->decode(tick);
    }

    return true;
}
```

上面主要是两个环节，先编码然后解码

#### 3.3.2.1循环调用encode

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
void AudioStream::encode(int tick, AudioStream *chain)
{
    ...  
    if (mMode != RECEIVE_ONLY && mDtmfEvent != -1) {
        int duration = mTimestamp - mDtmfStart;
        // Make sure duration is reasonable.
        if (duration >= 0 && duration < mSampleRate * DTMF_PERIOD) {
            duration += mSampleCount;
            int32_t buffer[4] = {
                static_cast<int32_t>(htonl(mDtmfMagic | mSequence)),
                static_cast<int32_t>(htonl(mDtmfStart)),
                static_cast<int32_t>(mSsrc),
                static_cast<int32_t>(htonl(mDtmfEvent | duration)),
            };
            if (duration >= mSampleRate * DTMF_PERIOD) {
                buffer[3] |= htonl(1 << 23);
                mDtmfEvent = -1;
            }
            //[1]第一次发送一些关于头部的信息，总共16个字节
            sendto(mSocket, buffer, sizeof(buffer), MSG_DONTWAIT,
                (sockaddr *)&mRemote, sizeof(mRemote));
            return;
        }
        mDtmfEvent = -1;
    }
    int32_t buffer[mSampleCount + 3];
    int16_t samples[mSampleCount];
    // Cook the packet and send it out.
    buffer[0] = htonl(mCodecMagic | mSequence);
    buffer[1] = htonl(mTimestamp);
    buffer[2] = mSsrc;
    //[2]在发送前先编码，调用AudioCodec编码
    int length = mCodec->encode(&buffer[3], samples);
    if (length <= 0) {
        ALOGV("stream[%d] encoder error", mSocket);
        return;
    }
    //[3]编码之后发送,MSG_DONTWAIT代表无阻塞
    sendto(mSocket, buffer, length + 12, MSG_DONTWAIT, (sockaddr *)&mRemote,
        sizeof(mRemote));
}
```



#### 3.3.2.2回调到decode函数

当完成发送的时候，epoll会回调到decode函数

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
void AudioStream::decode(int tick)
{
    ...
    int count = (BUFFER_SIZE - (mBufferTail - mBufferHead)) * mSampleRate;
    if (count < mSampleCount) {
        // 如果发送的数据越界，那么退出
        ALOGV("stream[%d] buffer overflow", mSocket);
        recv(mSocket, &c, 1, MSG_DONTWAIT);
        return;
    }

    // Receive the packet and decode it.
    int16_t samples[count];
    if (!mCodec) {
        ... 
    } else {
        //四字节对其
        __attribute__((aligned(4))) uint8_t buffer[2048];
        sockaddr_storage remote;
        socklen_t addrlen = sizeof(remote);

        int bufferSize = sizeof(buffer);
        //接收数据
        int length = recvfrom(mSocket, buffer, bufferSize,
            MSG_TRUNC | MSG_DONTWAIT, (sockaddr *)&remote, &addrlen);

        //进行数据头部校验
        if (length < 12 || length > bufferSize ||
            (ntohl(*(uint32_t *)buffer) & 0xC07F0000) != mCodecMagic) {
            ALOGV("stream[%d] malformed packet", mSocket);
            return;
        }
        int offset = 12 + ((buffer[0] & 0x0F) << 2);
        if (offset + 2 + (int)sizeof(uint16_t) > length) {
            ALOGV("invalid buffer offset: %d", offset+2);
            return;
        }
        if ((buffer[0] & 0x10) != 0) {
            offset += 4 + (ntohs(*(uint16_t *)&buffer[offset + 2]) << 2);
        }
        if ((buffer[0] & 0x20) != 0) {
            length -= buffer[length - 1];
        }
        length -= offset;
        if (length >= 0) {
            //接收完成后解码，调用AudioCodec
            length = mCodec->decode(samples, count, &buffer[offset], length);
        }
        if (length > 0 && mFixRemote) {
            mRemote = remote;
            mFixRemote = false;
        }
        count = length;
    }
    ...
}
```



### 3.3.3接收端

接收端也是另外一个DeviceThread线程，主要是用于本地的一些实时处理

```c++
//frameworks/opt/net/voip/src/jni/rtp/AudioGroup.cpp
bool AudioGroup::DeviceThread::threadLoop()
{
    //在线程初始化的时候就赋值了
    int deviceSocket = mGroup->mDeviceSocket;
    ...
    //[0]设置接收和发送的数据为最小帧数
    setsockopt(deviceSocket, SOL_SOCKET, SO_RCVBUF, &output, sizeof(output));
    setsockopt(deviceSocket, SOL_SOCKET, SO_SNDBUF, &output, sizeof(output));

    // [0]这里主要去除一些错误信息
    char c;
    while (recv(deviceSocket, &c, 1, MSG_DONTWAIT) == 1);
	...
    //本地回声处理的一些初始化操作 
    while (!exitPending()) {
        int16_t output[sampleCount];
        //[1]接收数据，缓冲区大小为2*sampleCount
        if (recv(deviceSocket, output, sizeof(output), 0) <= 0) {
            memset(output, 0, sizeof(output));
        }

        int16_t input[sampleCount];
        int toWrite = sampleCount;
        int toRead = (mode == MUTED) ? 0 : sampleCount;
        int chances = 100;
		...
		//处理数据
        if (mode != MUTED) {
            if (echo != NULL) {
                ALOGV("echo->run()");
                //[2]回声抵消操作
                echo->run(output, input);
            }
            //[3]处理完成之后，再把处理后的数据发送给对端,MSG_DONTWAIT非阻塞
            send(deviceSocket, input, sizeof(input), MSG_DONTWAIT);
        }
    }

exit:
    delete echo;
    return true;
}
```

### 3.3.4总结

这里是rtp协议，通过网络形式发送实时流，然后通过编码传输到本地，进行回声抵消等操作，然后处理完成后再从本地发送给网络端，并对其解码，整个流程是UDP的套接字，说明对数据的实时性要求非常高，且允许发送和接收过程丢包。

{{< image classes="fancybox center fig-100" src="/socketpair/sp_14.png" thumbnail="/socketpair/sp_14.png" title="">}}

# 4总结

总的来说，socketpair就是socket和pipe的结合，具有全双工，并且两端都可读写，不管是在源码中还是平时使用中，特别是多进程/多线程之间的通信，都会涉及到socketpair这一块。本文主要讲了三种类型的套接字类型`SOCK_STREAM`（TCP流），`SOCK_SEQPACKET`（整体发送），`SOCK_DGRAM`（UDP性质的），其实还可以有其他类型，这一块需要读者自己去研究了。耐心阅读，找到最根本的原理，即可一通百通。



# 参考

[[1] 明明1109. Linux C常见数I/O函数比较: printf, sprintf, fprintf, write..., 2021.](https://www.cnblogs.com/fortunely/p/14872465.html)

[[1] Donald_Shallwing. 经验分享：编写简易的温度监控程序（3）, 2019.](https://blog.csdn.net/Donald_Shallwing/article/details/86767572)

[[1] alibli. Camera MetaData介绍_独家原创, 2023.](https://blog.csdn.net/weixin_36389889/article/details/125820456)

[[1] IT技术分享网. Android Camera之CameraMetadata分析, 2023.](http://www.5ityx.com/cate100/261276.html)

[[1] cfc1243570631. CameraMetadata 知识学习整理, 2022.](https://blog.csdn.net/cfc1243570631/article/details/128033812)

[[1] armwind. Android Camera之CameraMetadata分析, 2016.](https://blog.csdn.net/armwind/article/details/52027010)

[[1] "小夜猫&小懒虫&小财迷"的男人. 【高通SDM660平台】(8) --- Camera MetaData介绍, 2020.](https://blog.csdn.net/Ciellee/article/details/105807436)

[[1]程序猿Ricky的日常干货 . Linux fcntl 函数详解, 2019.](https://blog.csdn.net/rikeyone/article/details/88828154)

[[1] 二的次方. Android 图像显示系统 - 基础知识之 BitTube, 2022.](https://www.cnblogs.com/roger-yu/p/16158539.html)

[[1] 程序员Android. Android 系统init进程启动流程, 2023.](https://mp.weixin.qq.com/s/gEiTIgStFkIrPPSXLlJ2WA)

[[1] 晓涵涵. Android中UID、GID和PID的讲解, 2019.](https://blog.csdn.net/xk7298/article/details/102291785)

[[1] 猿來孺兹. protobuf c/c++详解, 2022.](https://blog.csdn.net/lang151719/article/details/115214859)

[[1] 书山青鸟叫. 浅谈Linux poll机制, 2022.](https://blog.csdn.net/weixin_43389824/article/details/124731063)
