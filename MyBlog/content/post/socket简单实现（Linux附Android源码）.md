---
title: "socket简单实现（Linux附Android源码）"
date: 2023-01-02
thumbnailImagePosition: left
thumbnailImage: socket/socket_linux_thumb.png
coverImage: socket/socket_linux_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- socket
- 2023
- January
tags:
- Linux
- Unix
- TCP
- UDP
- 源码
- SOCK_RAW
- PF_NETLINK
- SOCK_SEQPACKET
- SOCK_STREAM
- SOCK_DGRAM
showSocial: false
---
在Linux中的socket用法，它是一种RPC的机制，通过通过客户端和服务端相连，产生socket句柄，通过句柄完成数据的传输通信。

<!--more-->
# 1Linux socket

socket是一类接口，在Linux中，一切皆为文件，socket亦然。因此，在Linux中的socket用法，它是一种RPC的机制，通过通过客户端和服务端相连，产生socket句柄，通过句柄完成数据的传输通信。

# 2 socket用法

为创建一个套接字，调用socket函数

## 2.1函数原型

```c++
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

## 2.2 描述

socket()为通信创建一个端点并返回一个描述符。

### 2.2.1第一个参数

domain参数指定通信域;这将选择用于通信的协议族。这些族定义在<sys/socket.h>中。目前理解的格式包括

|       Name        |       Purpose        |  Man page  |
| :---------------: | :------------------: | :--------: |
| AF_UNIX, AF_LOCAL |       本地通信       |  unix(7)   |
|      AF_INET      |   IPv4网络通信协议   |   ip(7)    |
|     AF_INET6      |   IPv6网络通信协议   |  ipv6(7)   |
|      AF_IPX       | IPX - Novell专用协议 |            |
|    AF_NETLINK     | 内核用户界面设备通信 | netlink(7) |
|     AF_PACKET     |   低层分组接口通信   | packet(7)  |

### 2.2.2第二个参数

套接字具有指定的类型，该类型指定通信语义。当前定义的类型有

|      类型      |                             含义                             |
| :------------: | :----------------------------------------------------------: |
|  SOCK_STREAM   | （**TCP**）提供有序的、可靠的、双向的、基于连接的字节流。可以支持带外数据传输机制。 |
|   SOCK_DGRAM   |  （**UDP**）支持数据报(无连接，固定最大长度的不可靠消息)。   |
| SOCK_SEQPACKET | 为固定最大长度的数据报提供一个有序的、可靠的、基于双向连接的数据传输路径<br />消费者在每次输入系统调用时都需要读取整个数据包。 |
|    SOCK_RAW    |                    提供原始网络协议访问。                    |
|    SOCK_RDM    |              提供一个不保证排序的可靠数据报层。              |

从Linux 2.6.27开始，type参数还有第二个用途:除了指定套接字类型外，它还可以包括按位的修改socket()的行为

> - **SOCK_NONBLOCK**
>
>   在新打开的文件描述中设置`O_NONBLOCK`文件状态标志。使用这个标志可以节省对`fcntl(2)`的额外调用，以达到相同的结果。
>
> - **SOCK_CLOEXEC**
>
>   在进程执行exec系统调用时关闭此打开的文件描述符。在子进程没有相应权限的情况下，防止父进程将打开的文件描述符泄露给子进程，防止子进程间接获得权限。

### 2.2.3第三个参数

协议指定了套接字使用的特定协议。在给定的协议族中，通常只有一个协议支持特定的套接字类型，在这种情况下，protocol可以指定为0。然而，可能存在许多协议，在这种情况下，必须以这种方式指定特定的协议。要使用的协议号是特定于将要发生通信的“通信域”的;看到协议(5)。关于如何将协议名字符串映射到协议号，请参见getprotoent(3)。

**1）SOCK_STREAM**

这个类型的套接字是**全双工字节流**。它们不保留记录边界。流套接字在发送或接收任何数据之前必须处于连接状态。到另一个套接字的连接是通过`connect(2)`调用创建的。一旦连接，数据可以使用`read(2)`和`write(2)`调用或`send(2)`和`recv(2)`调用的一些变体来传输。当一个会话完成时，可以执行`close(2)`。带外数据也可以像send(2)中描述的那样传输，并像recv(2)中描述的那样接收。

实现SOCK_STREAM的通信协议**确保数据不会丢失或重复**。如果对端协议有缓冲空间的一段数据不能在合理的时间内成功传输，则认为该连接已经死亡。当在套接字上启用SO_KEEPALIVE时，协议以特定于协议的方式检查另一端是否仍然活跃。如果进程在中断的流上发送或接收，则引发SIGPIPE信号;这将导致不处理信号的naive进程退出。

**2）SOCK_SEQPACKET**

这个套接字使用与SOCK_STREAM套接字**相同的系统调用**。唯一的区别是read(2)调用将只返回所请求的数据量，而到达数据包中剩余的任何数据将被丢弃。此外，传入数据报中的所有消息边界都被保留。

**3）SOCK_DGRAM和SOCK_RAW**

这两个套接字允许向**sendto(2)**调用中命名的通讯员发送数据报。数据报通常使用**recvfrom(2)**接收，它返回下一个数据报及其发送者的地址。

fcntl(2) F_SETOWN操作可用于指定进程或进程组在带外数据到达时接收SIGURG信号或在SOCK_STREAM连接意外中断时接收SIGPIPE信号。该操作还可用于设置通过SIGIO接收I/O和I/O事件异步通知的进程或进程组。使用F_SETOWN相当于使用FIOSETOWN或SIOCSPGRP参数进行ioctl(2)调用。

**4）其他**

套接字的操作由套接字级别的选项控制中定义了这些选项。函数**setsockopt(2)**和**getsockopt(2)**分别用于设置和获取选项。

关于**setsockopt**，具体可以点击[这里](https://baike.baidu.com/item/setsockopt/10069288?fr=aladdin)。

> setsockopt
>
> ```c++
> #include <sys/types.h>
> #include <sys/socket.h>
> int setsockopt(int sockfd, int level, int optname,const void *optval, socklen_t optlen);
> ```
>
> 1. **sockfd**：标识一个套接口的描述字。
> 2. **level**：选项定义的层次；支持**SOL_SOCKET**、IPPROTO_TCP、IPPROTO_IP和IPPROTO_IPV6。
> 3. **optname**：需设置的选项。
> 4. **optval**：指针，指向存放选项待设置的新值的缓冲区。
> 5. **optlen：optval**缓冲区长度。
>
> ```c++
> optname定义如下：
>  
> #define SO_DEBUG 1            -- 打开或关闭调试信息（BOOL）
> #define SO_REUSEADDR 2        -- 打开或关闭地址复用功能（BOOL）
> #define SO_TYPE 3             -- 
> #define SO_ERROR 4
> #define SO_DONTROUTE 5		  -- 禁止选径；直接传送。（BOOL）
> #define SO_BROADCAST 6		  -- 允许套接口传送广播信息（BOOL）
> #define SO_SNDBUF 7           -- 设置发送缓冲区的大小（int）
> #define SO_RCVBUF 8           -- 设置接收缓冲区的大小（int）
> #define SO_KEEPALIVE 9        -- 套接字保活（BOOL）
> #define SO_OOBINLINE 10		  -- 在常规数据流中接收带外数据（BOOL）
> #define SO_NO_CHECK 11
> #define SO_PRIORITY 12        -- 设置在套接字发送的所有包的协议定义优先权
> #define SO_LINGER 13		  -- 如关闭时有未发送数据，则逗留（struct linger FAR*）
> #define SO_BSDCOMPAT 14
> #define SO_REUSEPORT 15
> #define SO_PASSCRED 16
> #define SO_PEERCRED 17
> #define SO_RCVLOWAT 18
> #define SO_SNDLOWAT 19
> #define SO_RCVTIMEO 20       -- 设置接收超时时间
> #define SO_SNDTIMEO 21       -- 设置发送超时时间
> #define SO_ACCEPTCONN 30
> #define SO_SNDBUFFORCE 32
> #define SO_RCVBUFFORCE 33
> #define SO_PROTOCOL 38
> #define SO_DOMAIN 39
> ```
>
> 举例说明
>
> ```c++
> BOOL bReuseaddr = TRUE;
> setsockopt( s, SOL_SOCKET, SO_REUSEADDR, ( const char* )&bReuseaddr, sizeof( BOOL ) );
> ```

### 2.2.4sock协议族

关于**socketaddr**，**sockaddr_un**和**sockaddr_in**的关系和区别

**1）socketaddr**

基本地址结构，本身没有意义，仅用于泛型化参数，结构体总共16个字节

```c++
typedef unsigned short sa_family_t;

struct sockaddr
{
    sa_family_t  sa_family;//地址族
    char   sa_data[14];//地址值
};
```

**2）sockaddr_un**

本地地址结构，用于**AF_LOCAL/AF_UNIX**域的本地通信，，结构体总共110个字节

```c++
struct sockaddr_un
{
    sa_family_t  sun_family;//地址族(AF_LOCAL/AF_UNIX)
    char   sun_path[108];//本地套接字文件的路径，通常路径字符串不能超过108
};
```

**3）sockaddr_in**

网络地址结构，用于**AF_INET**域的IPv4网络通信，结构体总共16个字节

```c++
typedef uint16_t in_port_t;//无符号16位整数
typedef uint32_t in_addr_t;//无符号32位整数

struct  in_addr
{
    in_addr_t   s_addr;
};

struct sockaddr_in
{
    sa_family_t sin_family;//地址族(AF_INET)
    int_port_t  sin_port;//端口号（0~65535）
    struct   in_addr    sin_addr;//IP地址
    unsigned char sin_zero[8];//在linux中会有填充字段，sin_zero[8],全部置为0
};
```



### 2.2.5demo

#### 2.2.5.1基于TCP协议的demo



{{< image classes="fancybox center fig-100" src="/socket/socket_7.png" thumbnail="/socket/socket_7.png" title="">}}

##### 2.2.5.1.1服务端

```c++
//基于tcp协议的服务器
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<sys/socket.h>
#include<sys/types.h>
#include<arpa/inet.h>
#include<ctype.h>
#include<errno.h>
#include<signal.h>
#include<sys/wait.h>
//信号处理函数
void sigchild(int signum)
{
    for(;;)
    {
        pid_t pid = waitpid(-1,NULL,WNOHANG);//非阻塞方式回收子进程
        if(pid == -1)
        {
            if(errno == ECHILD)
            {
                printf("没有子进程可回收\n");
                break;
            }
            else
            {
                perror("waitpid");
                exit(-1);
            }
        }
        else if(pid == 0)
        {
            printf("子进程在运行，没法收\n");
            break;
        }
        else
        {
            printf("回收%d进程僵尸\n",getpid());
        }
    }
}
 
int main(void)
{
    //捕获17号信号
    if(signal(SIGCHLD,sigchild) == SIG_ERR)
    {
        perror("signal");
        return -1;
    }
    //创建侦听套接字
    printf("服务器：创建套接字\n");
    int sockfd = socket(AF_INET,SOCK_STREAM,0);//返回侦听套接字
    if(sockfd == -1)
    {
        perror("socket");
        return -1;
    }
    printf("sockfd:%d\n",sockfd);
    //组织地址结构，代表服务器
    printf("服务器：组织地址结构\n");
    struct sockaddr_in ser;
    ser.sin_family = AF_INET;
    ser.sin_port = htons(8888);//注意大小端问题，存是小端，服务器大端方式拿。
    //ser.sin_addr.s_addr = inet_addr("127.0.0.1")
    ser.sin_addr.s_addr = INADDR_ANY;//表示接受任意IP下的地址
    //绑定套接字和地址结构
    printf("服务器：绑定套接字和地址结构\n");
    int sockbind = bind(sockfd,(struct sockaddr*)&ser,sizeof(ser));
    if(sockbind == -1)
    {
        perror("bind");
        return -1;
    }
    //开启侦听功能
    printf("服务器：开启侦听功能\n");
    int socklisten = listen(sockfd,1024);
    if(socklisten == -1)
    {
        perror("listen");
        return -1;
    }
    //等待并接收客户端连接
    for(;;)
    {   //父进程负责和客户端建立通信
        printf("服务器：等待并接收客户端连接\n");
        struct sockaddr_in cli;//用来输出客户端的地址结构
        socklen_t len = sizeof(cli);
        int conn = accept(sockfd,(struct sockaddr*)&cli,&len);//返回后续用来通信的通信套接字
        if(conn == -1)
        {
            perror("accept");
            return -1;
        }
        printf("服务器：接收到%s:%hu的客户端\n",inet_ntoa(cli.sin_addr),ntohs(cli.sin_port));
        //子进程负责和客户端通信
        pid_t pid = fork();
        if(pid == -1)
        {
            perror("fork");
            return -1;
        }
        if(pid == 0)
        {
            close(sockfd);//子进程不用侦听套接字，关闭
            //业务处里（接发数据）
            printf("服务器：接发数据\n");
            //接收客户端发来的小写的串
            while(1)
            {
                char buf[128] = {};
                ssize_t size = read(conn,buf,sizeof(buf) - sizeof(buf[0]));
                if(size == -1)
                {
                    perror("read");
                    return -1;
                }
                if(size == 0)
                {
                    printf("服务端：客户端断开连接\n");
                    break;
                }
                for(int i = 0; i < size; i++)
                {
                    //转换大写
                    buf[i] = toupper(buf[i]);
                }
                //发送给客户端
                if(write(conn,buf,size) ==-1)
                {
                    perror("write");
                    return -1;
                }
            }
            //关闭套接字
            printf("服务器：关闭套接字\n");
            close(conn);
            return 0;
        }
        close(conn);//父进程关闭通信套接字
    }
    close(sockfd);
    return 0;
}
```

##### 2.2.5.1.2客户端

```c++
//基于TCP协议的客户端
#include<stdio.h>
#include<unistd.h>
#include<string.h>
#include<sys/socket.h>
#include<sys/types.h>
#include<arpa/inet.h>
 
int main()
{
    //创建服务器套接字
    printf("客户端：创建套接字\n");
    int sockfd = socket(AF_INET,SOCK_STREAM,0);
    if(sockfd ==-1)
    {
        perror("socket");
        return -1;
    }
    //组织服务器的地址结构
    printf("客户端：组织服务器的地址结构\n");
    struct sockaddr_in ser;
    ser.sin_family = AF_INET;
    ser.sin_port = htons(8888);
    ser.sin_addr.s_addr = inet_addr("127.0.0.1");//不能像服务器宏一样
    //发起连接
    printf("客户端：发起连接\n");
    if(connect(sockfd,(struct sockaddr*)&ser,sizeof(ser)) == -1 )
    {
        perror("connect");
        return -1;
    } 
    //业务处理
    printf("客户端：业务处理\n");
    for(;;)
    {
        char buf[128] = {};
        fgets(buf,sizeof(buf),stdin);
        //循环退出条件
        if(strcmp(buf,"!\n") == 0)
        {
            break;
        }
        //将小写传发送给服务器
        if(send(sockfd,buf,strlen(buf),0) == -1 )
        {
            perror("send");
            return -1;
        }
 
        //接收服务器回传的大写的串
        if(recv(sockfd,buf,sizeof(buf) - sizeof(buf[0]),0) == -1)
        {
            perror("recv");
            return -1;
        }
        printf(">>%s",buf);
    }
    //关闭套接字
    printf("客户端：关闭套接字\n");
    close(sockfd);
}
```

#### 2.2.5.2基于UDP协议的demo

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_9.png" thumbnail="/socket/socket_linux_9.png" title="">}}

##### 2.2.5.2.1服务端

```c++
//基于UDP的服务器
#include<stdio.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<sys/types.h>
 
int main()
{
    //创建套接字
    printf("服务器：创建套接字\n");
    int sockfd = socket(PF_INET,SOCK_DGRAM,0);
    if(sockfd == -1)
    {
        perror("socket");
        return -1;
    }
    //组织地址结构
    printf("服务器：组织地址结构\n");
    struct sockaddr_in ser;
    ser.sin_family = AF_INET;
    ser.sin_port = htons(9999);
    ser.sin_addr.s_addr = INADDR_ANY;//接受任意IP地址数据
    //绑定套接字和地址结构
    printf("服务器：绑定套接字和地址结构\n");
    if(bind(sockfd,(struct sockaddr *)&ser,sizeof(ser)) == -1)
    {
        perror("bind");
        return -1;
    }
    //处理数据
    printf("服务器：处理数据\n");
    while(1)
    {
        //接收客户端的串
        char buf[128] = {};
        struct sockaddr_in cli;//用来输出客户端的地址结构
        socklen_t len = sizeof(cli);
        ssize_t size = recvfrom(sockfd,buf,sizeof(buf)-sizeof(buf[0]),0,(struct sockaddr*)&cli,&len);
        if(size == -1)
        {
            perror("recvfrom");
            return -1;
        }
        //转成大写
        for(int i = 0;i < size;i++)
        {
            buf[i] = toupper(buf[i]);
        }
        //发送客户端
        if(sendto(sockfd,buf,size,0,(struct sockaddr *)&cli,len) == -1)
        {
            perror("sendto");
            return -1;
        }
    }
    //关闭套接字
    printf("服务器：关闭套接字\n");
    close(sockfd);
    return 0;
}
```

##### 2.2.5.2.2客户端

```c++
//客户端
#include<stdio.h>
#include<unistd.h>
#include<sys/socket.h>
#include<string.h>
#include<sys/types.h>
#include<arpa/inet.h>
 
int main()
{
    //创建套接字
    printf("客户端：创建套接字\n");
    int sockfd = socket(PF_INET,SOCK_DGRAM,0);
    if(sockfd == -1)
    {
        perror("socket");
        return -1;
    }
    //组织服务器地址结构
    printf("客户端：组织地址结构\n");
    struct sockaddr_in ser;
    ser.sin_family = PF_INET;
    ser.sin_port = htons(9999);
    ser.sin_addr.s_addr = inet_addr("127.0.0.1");
    //数据处理
    printf("客户端：数据处理\n");
    for(;;)
    {
        //通过键盘获取小写的串
        char buf[128] = {};
        fgets(buf,sizeof(buf),stdin);
        //循环退出条件
        if(strcmp(buf,"!\n") == 0)
        {
            break;
        }
 
        //发送给服务器
        if(sendto(sockfd,buf,strlen(buf),0,(struct sockaddr*)&ser,sizeof(ser)) == -1)
        {
            perror("sendto");
            return -1;
        }
        //接收服务器发送回来的大写的串
        if(recv(sockfd,buf,sizeof(buf)-sizeof(buf[0]),0) == -1)
        {
            perror("recv");
            return -1;
        }
        //打印输出
        printf("%s\n",buf);
    }
    //关闭套接字
    close(sockfd);
    return 0;
}
```





### 2.2.6socket一些补充

本文篇幅有限，主要做一个大致的介绍，具体内容还需要查看**《UNIX环境高级编程手册》**

#### 2.2.6.1关于socket中的write和send的区别

> ```c
> ssize_t write(int fd, const void*buf,size_t nbytes);
> 
> #include <sys/types.h>
> #include <sys/socket.h>
> ssize_t send(int sockfd, const void *buf, size_t len, int flags);
> ```
>
> 尽管可以通过read和write交换数据，但这就是这两个函数所能做的一切。如果想指定选项，从多个客户端接收数据包，或者发送带外数据，就需要使用6个为数据传递而设计的套接字函数中的一个。3个函数用来发送数据，3个用于接收数据。最简单的是**send**，它和write很像，但是可以指定标志来改变处理传输数据的方式。**即send()和write之间的唯一区别是是否存在标志。使用零标志参数，send()等效于write**。
>
> 类似write，使用send时套接字必须已经连接。参数buf和nbytes的含义与write中的一致。
> 然而，与write不同的是，send支持第4个参数args。
>
> |      flags      | 含义                                                         |
> | :-------------: | ------------------------------------------------------------ |
> | `MSG_CONFIRM `  | 告诉链接层:你从另一端得到了一个成功的回复。如果链路层没有得到这一点，它将定期重新探测邻居(例如，通过单播ARP)。仅对`SOCK_DGRAM`和`SOCK_RAW`套接字有效，目前仅对`IPv4`和`IPv6`实现。 |
> | `MSG_DONTROUTE` | 不要使用网关发送数据包，只发送到直连网络上的主机。这通常**只被诊断程序或路由程序使用**。 |
> | `MSG_DONTWAIT`  | **启用非阻塞操作**;如果操作将阻塞，则返回`EAGAIN`或`EWOULDBLOCK`。这提供了**类似于设置`O_NONBLOCK`**标志的行为(通过`fcntl(2) F_SETFL`操作)，但不同之处在于`MSG_DONTWAIT`是每个调用的选项，而`O_NONBLOCK`是对打开的文件描述的设置(参见`open(2)`)，这将影响调用进程中的所有线程，以及其他持有引用相同打开文件描述的文件描述符的进程。 |
> |    `MSG_EOR`    | **终止一条记录**(当支持此概念时，例如`SOCK_SEQPACKET`类型的套接字)。 |
> |   `MSG_MORE `   | **调用方有更多数据要发送**。此标志用于`TCP`套接字以获得与`TCP_CORK`套接字选项相同的效果(参见`TCP(7)`)，不同的是此标志可以在每次调用的基础上设置。<br />自Linux 2.6以来，这个标志也支持`UDP`套接字，并通知内核将所有在调用中发送的数据打包成一个单一的数据报，该数据报仅在执行不指定此标志的调用时传输。(请参见`udp(7)`中描述的`UDP_CORK`套接字选项。) |
> | `MSG_NOSIGNAL`  | 如果对等端在面向流的套接字上关闭了连接，则**不要生成`SIGPIPE`信号**。仍然返回`EPIPE`错误。这提供了**类似于使用`sigaction(2)`忽略`SIGPIPE`的行为**，但是，`MSG_NOSIGNAL`是每个调用的特性，忽略`SIGPIPE`会设置一个影响进程中所有线程的进程属性。 |
> |    `MSG_OOB`    | 在支持此概念的套接字(例如`SOCK_STREAM`类型)上**发送带外数据**;底层协议还必须支持带外数据。**紧急消息，会优先处理**。 |



#### 2.2.6.2关于socket中的write和writev的区别

> ```c
> #include <sys/uio.h>
> struct iovec {
>     void  *iov_base;    /* Starting address */
>     size_t iov_len;     /* Number of bytes to transfer */
> };
> ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
> ```
>
> writev 函数的功能可概括为：对数据进行整合接收及发送的函数。也就是说，通过 writev 函数可以将分散保存在多个缓冲中的数据一并发送，通过 readv 函数可以由多个缓冲分别接收。writev()在继续到iov[1]之前写出iov[0]的全部内容，writev()执行的数据传输是**原子**的。因此，**适当使用这两个函数可以减少 I/O 函数的调用次数**。
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_linux_10.png" thumbnail="/socket/socket_linux_10.png" title="writev函数结构图">}}
>
> **write和writev区别**
>
> 对于socket IO而言，write经常不能够一次写完，好在它会返回已经写了多少字节，如果继续写，此时就会阻塞；对于非阻塞socket而言，write会在buf不可写时返回的EAGAIN，那么在下一次write时，便可通过之前返回的值重新确定基址和长度。
>
> manual中对于writev的相关描述为：和write类似。也就是说，它也会返回已经写入的长度或者EAGAIN(errno)。千万不可天真地认为，每次传同样的iovec就能解决问题，writev并不会为你做任何事情，重新处理iovec是调用者的任务。
>
> 问题是，这个返回值“实用性”并不高，因为参数传入的是iovec数组，计量单位是iovcnt，而不是字节数，用户依旧需要通过遍历iovec来计算新的基址，另外写入数据的“结束点”可能位于一个iovec的中间某个位置，因此需要调整临界iovec的io_base和io_len。
>
> 对于socket，尤其是**非阻塞socket**，还是**尽可能避免writev**的好，实现连续的内存块反而可以简化实现。

#### 2.2.6.3关于socket中recvmsg 和 sendmsg 函数

具体内容可以点击[这里](https://www.cnblogs.com/jimodetiantang/p/9190958.html)

> ```c++
> #include <sys/types.h>
> #include <sys/socket.h>
> 
> ssize_t send(int sockfd, const void *buf, size_t len, int flags);
> 
> ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
>               const struct sockaddr *dest_addr, socklen_t addrlen);
> 
> ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
> ```
>
> 主要是有两个结构体**msghdr**和**cmsghdr**。
>
> 这两个函数的参数中都有一个指向msghdr结构的指针，该结构包含了所有关于要发送或要接收的消息的信息。该结构的定义大致如下：
>
> ```c++
> struct msghdr {
>     void          *msg_name;        // protocol address
>     socklen_t      msg_namelen;     // size of protocol address
>     struct iovec  *msg_iov;         // scatter/gather array
>     int            msg_iovlen;      // elements in msg_iov
>     void          *msg_control;     // ancillary data (cmsghdr struct)
>     socklen_t      msg_controllen;  // length of ancillary data
>     int            msg_flags;       // flags returned by recvmsg()
> };
> 
> struct cmsghdr {
>     size_t cmsg_len;
>     int cmsg_level;
>     int cmsg_type;
> };
> ```
>
> 另外还有几个宏定义用于上面结构体的计算
>
> ```c++
> #define CMSG_NXTHDR(mhdr, cmsg) __cmsg_nxthdr((mhdr), (cmsg))
> #define CMSG_ALIGN(len) ( ((len)+sizeof(long)-1) & ~(sizeof(long)-1) )
> #define CMSG_DATA(cmsg) (((unsigned char*)(cmsg) + CMSG_ALIGN(sizeof(struct cmsghdr))))
> #define CMSG_SPACE(len) (CMSG_ALIGN(sizeof(struct cmsghdr)) + CMSG_ALIGN(len))
> #define CMSG_LEN(len) (CMSG_ALIGN(sizeof(struct cmsghdr)) + (len))
> #define CMSG_FIRSTHDR(msg) \
>        ((msg)->msg_controllen >= sizeof(struct cmsghdr) \
>        ? (struct cmsghdr*) (msg)->msg_control : (struct cmsghdr*) NULL)
> #define CMSG_OK(mhdr, cmsg) ((cmsg)->cmsg_len >= sizeof(struct cmsghdr) &&   (cmsg)->cmsg_len <=   (unsigned long)   ((mhdr)->msg_controllen -   ((char*)(cmsg) - (char*)(mhdr)->msg_control)))
> ```
>
> 关于结构体**msghdr**
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_linux_7.png" thumbnail="/socket/socket_linux_7.png" title="">}}
>
> 关于结构体**msghdr**和**cmsghdr**联系
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_linux_8.png" thumbnail="/socket/socket_linux_8.png" title="">}}
>
> **send，sendto和sendmsg区别**
>
> **即使send成功返回，也并不表示连接的另一端的进程就一定接收了数据**。我们所能保证的只是当send成功返回时，数据已经被无错误地发送到网络驱动程序上。当消息没有放入套接字的发送缓冲区时，**send()通常会阻塞，除非套接字已被置于非阻塞I/O模式**。在非阻塞模式下，它会失败，在这种情况下会出现EAGAIN或EWOULDBLOCK错误。
>
> 对于支持报文边界的协议，如果尝试发送的单个报文的长度超过协议所支持的最大长度，那么send会失败，并将errno设为EMSGSIZE。对于字节流协议，send会阻塞直到整个数据传输完成。函数sendto和send很类似。区别在于sendto可以在无连接的套接字上指定一个目标地址。
>
> 对于面向连接的套接字，目标地址是被忽略的，因为连接中隐含了目标地址。对于无连接的套接字，除非先调用connect设置了目标地址，否则不能使用send。sendto提供了发送报文的另一种方式。
>
> 通过套接字发送数据时，还有一个选择。可以调用带有msghdr结构的sendmsg来指定多重缓冲区传输数据，这和writev函数很相似。sendmsg()调用还允许发送辅助数据(也称为控制信息)。
>
> ```c
> //下面两种方式一致
> send(sockfd, buf, len, flags);
> sendto(sockfd, buf, len, flags, NULL, 0);
> ```
>
> 如果sendto()用于**连接模式套接字(SOCK_STREAM, SOCK_SEQPACKET)**，则参数dest_addr和addrlen将被**忽略**(当它们不为NULL和0时可能返回EISCONN错误)，并且当套接字未实际连接时返回错误ENOTCONN。否则，目标的地址由dest_addr给出，addrlen指定其大小。对于sendmsg()，目标地址由msg给出。Msg_name，加上msg。msg_namelen指定它的大小。
>

#### 2.2.6.4socket发送函数总结

应用层的五个发送函数，实际上对应系统调用最终都回到sock_sendmsg，即把应用层的数据封装成msghdr，然后继续发送到协议栈。write可以看作是单个iovec的数据，且flags为0；writev可以看作是不定项数个iovec的数据，且flags为0；send可以看作是单个iovec的数据，且flags可以是send falgs表中任意值；sendto就是封装了send，多了flags值和sockaddr；sendmsg比较特殊，默认就构造了msghdr，直接发送到sock_sendmsg即可。

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_11.png" thumbnail="/socket/socket_linux_11.png" title="">}}

# 3 源码中的socket

先介绍一下源码中封装的socket，由于Android系统中很多都是本地通信，回连127.0.0.1，所以单独封装了相对应的源码。

## 3.0.1客户端连接

根据上面篇章可以知道客户端连接只需要创建socket，等待与服务端连接connect

```c++
//system/core/libcutils/socket_local_client_unix.cpp
#define ANDROID_SOCKET_DIR		"/dev/socket"
// Linux "abstract" (non-filesystem) namespace
#define ANDROID_SOCKET_NAMESPACE_ABSTRACT 0
// Android "reserved" (/dev/socket) namespace
#define ANDROID_SOCKET_NAMESPACE_RESERVED 1
// Normal filesystem namespace
#define ANDROID_SOCKET_NAMESPACE_FILESYSTEM 2

int socket_local_client(const char *name, int namespaceId, int type)
{
    int s;
	//1.创建socket，AF_LOCAL指定本地连接
    s = socket(AF_LOCAL, type, 0);
    if(s < 0) return -1;

    if ( 0 > socket_local_client_connect(s, name, namespaceId, type)) {
        close(s);
        return -1;
    }

    return s;
}

int socket_local_client_connect(int fd, const char* name, int namespaceId, int /*type*/) {
    struct sockaddr_un addr;
    socklen_t alen;
    int err;
	//这里的addr是协议族，主要包含本地连接的路径名
    err = socket_make_sockaddr_un(name, namespaceId, &addr, &alen);

    if (err < 0) {
        goto error;
    }
	//2.等待与服务端连接
    if(connect(fd, (struct sockaddr *) &addr, alen) < 0) {
        goto error;
    }

    return fd;

error:
    return -1;
}

int socket_make_sockaddr_un(const char *name, int namespaceId, 
        struct sockaddr_un *p_addr, socklen_t *alen)
{
    memset (p_addr, 0, sizeof (*p_addr));
    size_t namelen;
	//根据传入的id来区分sockaddr_un族中的socket文件路径
    switch (namespaceId) {
        //这个协议通常是bt中使用比较多，类似在/data/misc/bluedroid/.sco_ctrl
        case ANDROID_SOCKET_NAMESPACE_ABSTRACT:
            namelen  = strlen(name);

            if ((namelen + 1) > sizeof(p_addr->sun_path)) {
                goto error;
            }
            p_addr->sun_path[0] = 0;
            memcpy(p_addr->sun_path + 1, name, namelen);
        break;
	    //这个协议/dev/socket文件夹下面的socket文件，有日志相关的logd等
        case ANDROID_SOCKET_NAMESPACE_RESERVED:
            namelen = strlen(name) + strlen(ANDROID_RESERVED_SOCKET_PREFIX);
            if (namelen > sizeof(*p_addr) 
                    - offsetof(struct sockaddr_un, sun_path) - 1) {
                goto error;
            }

            strcpy(p_addr->sun_path, ANDROID_RESERVED_SOCKET_PREFIX);
            strcat(p_addr->sun_path, name);
        break;
        //这个协议是正常文件系统的协议，比如/data/system/ndebugsocket
        case ANDROID_SOCKET_NAMESPACE_FILESYSTEM:
            namelen = strlen(name);
            if (namelen > sizeof(*p_addr) 
                    - offsetof(struct sockaddr_un, sun_path) - 1) {
                goto error;
            }

            strcpy(p_addr->sun_path, name);
        break;
        default:
            return -1;
    }

    p_addr->sun_family = AF_LOCAL;
    *alen = namelen + offsetof(struct sockaddr_un, sun_path) + 1;
    return 0;
error:
    return -1;
}
```

### 3.0.2服务端连接

根据上面篇章可以知道服务端连接需要创建socket，关联地址和套接字与创建监听队列。

```c++
//system/core/libcutils/socket_local_server_unix.cpp
#define LISTEN_BACKLOG 4
#define SOCK_TYPE_MASK 0xf
int socket_local_server(const char *name, int namespaceId, int type)
{
    int err;
    int s;
    //1.创建socket，AF_LOCAL指定本地连接
    s = socket(AF_LOCAL, type, 0);
    if (s < 0) return -1;

    err = socket_local_server_bind(s, name, namespaceId);

    if (err < 0) {
        close(s);
        return -1;
    }
	//SOCK_STREAM表示这个服务端是以TCP协议的传输层的socket通信，否则就是UDP协议
    if ((type & SOCK_TYPE_MASK) == SOCK_STREAM) {
        int ret;
		//3.如果是TCP协议，那么创建监听队列，默认LISTEN_BACKLOG为4
        ret = listen(s, LISTEN_BACKLOG);

        if (ret < 0) {
            close(s);
            return -1;
        }
    }

    return s;
}

int socket_local_server_bind(int s, const char *name, int namespaceId)
{
    struct sockaddr_un addr;
    socklen_t alen;
    int n;
    int err;
	//同客户端类似，这里的服务端主要是生成addr
    err = socket_make_sockaddr_un(name, namespaceId, &addr, &alen);

    if (err < 0) {
        return -1;
    }
	//因为是本地通信，所以当id是RESERVED和FILESYSTEM的时候，删除addr协议族的路径
    if (namespaceId == ANDROID_SOCKET_NAMESPACE_RESERVED
        || namespaceId == ANDROID_SOCKET_NAMESPACE_FILESYSTEM) {
        unlink(addr.sun_path);
    }

    n = 1;
    //设置调用close(socket)后,仍可继续重用该socket
    //调用close(socket)一般不会立即关闭socket，而经历TIME_WAIT的过程
    setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &n, sizeof(n));
	//2.关联地址和套接字
    if(bind(s, (struct sockaddr *) &addr, alen) < 0) {
        return -1;
    }

    return s;
}
```

## 3.1demo1

这个是Java端服务端和C/C++层为客户端进行TCP协议的连接

> 在AF_INET通信域中，套接字类型**SOCK_STREAM**的默认协议是传输控制协议(Transmission Control Protocol,TCP)。字节流(SOCK_STREAM)要求在交换数据之前，在本地套接字和通信的对等进程的套接字之间建立一个逻辑连接。使用面向连接的协议通信就像与对方打电话。首先，需要通过电话建立一个连接，连接建立好之后，彼此能双向地通信。每个连接是端到端的通信链路。对话中不包含地址信息，就像呼叫两端存在一个点对点虚拟连接，并且连接本身暗示特定的源和目的地。
>
> SOCK_STREAM套接字提供字节流服务，所以应用程序分辨不出报文的界限。这意味着从SOCK_STREAM套接字读数据时，它也许不会返回所有由发送进程所写的字节数。最终可以获得发送过来的所有数据，但也许要通过若干次函数调用才能得到。

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_1.png" thumbnail="/socket/socket_linux_1.png" title="">}}

### 3.1.1服务端

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void startObservingNativeCrashes() {
    final NativeCrashListener ncl = new NativeCrashListener(this);
    ncl.start();
}
```

NativeCrashListener的Thread启动

```java
//frameworks/base/services/core/java/com/android/server/am/NativeCrashListener.java
final class NativeCrashListener extends Thread {
    static final String DEBUGGERD_SOCKET_PATH = "/data/system/ndebugsocket";
	@Override
    public void run() {
        //这里是响应客户端的ack，为一个0值
        final byte[] ackSignal = new byte[1];

        if (DEBUG) Slog.i(TAG, "Starting up");
        {
            //创建一个socket文件，DEBUGGERD_SOCKET_PATH为/data/system/ndebugsocket
            File socketFile = new File(DEBUGGERD_SOCKET_PATH);
            if (socketFile.exists()) {
                socketFile.delete();
            }
        }

        try {
            //1.创建服务端socket句柄，AF_UNIX为本地通信，SOCK_STREAM为TCP协议
            FileDescriptor serverFd = Os.socket(AF_UNIX, SOCK_STREAM, 0);
            //获取DEBUGGERD_SOCKET_PATH，转化为UnixSocketAddress
            final UnixSocketAddress sockAddr = UnixSocketAddress.createFileSystem(
                    DEBUGGERD_SOCKET_PATH);
            //2.关联地址和套接字
            Os.bind(serverFd, sockAddr);
            //3.如果是TCP协议，那么创建监听队列，这里队列只有一个，同时只能处理一个IO
            Os.listen(serverFd, 1);
            //设置socket文件权限为可读可写可执行
            Os.chmod(DEBUGGERD_SOCKET_PATH, 0777);
			//这里就是为了保证可以循环处理多个IO，服务端不能直接关闭
            while (true) {
                FileDescriptor peerFd = null;
                try {
                    if (MORE_DEBUG) Slog.v(TAG, "Waiting for debuggerd connection");
                    //4.服务端获取连接请求并建立连接
                    peerFd = Os.accept(serverFd, null /* peerAddress */);
                    if (peerFd != null) {
						//服务端处理客户端的请求
                        consumeNativeCrashData(peerFd);
                    }
                } catch (Exception e) {
                    Slog.w(TAG, "Error handling connection", e);
                } finally {
                    if (peerFd != null) {
                        try {
                            //5.存在客户端，那么发送一个响应
                            Os.write(peerFd, ackSignal, 0, 1);
                        } catch (Exception e) {
                            ...
                        }
                        try {
                            //6.关闭客户端句柄
                            Os.close(peerFd);
                        } catch (ErrnoException e) {
                            ...
                        }
                    }
                }
            }
        } catch (Exception e) {
            Slog.e(TAG, "Unable to init native debug socket!", e);
        }
    }
```

### 3.1.2客户端

```c++
//system/core/debuggerd/crash_dump.cpp
static bool activity_manager_notify(pid_t pid, int signal, const std::string& amfd_data) {
  //获取句柄amfd， 
  // "/data/system/ndebugsocket"名称
  //ANDROID_SOCKET_NAMESPACE_FILESYSTEM类型为普通文件系统
  //SOCK_STREAM为TCP连接
  android::base::unique_fd amfd(socket_local_client(
      "/data/system/ndebugsocket", ANDROID_SOCKET_NAMESPACE_FILESYSTEM, SOCK_STREAM));
  if (amfd.get() == -1) {
    PLOG(ERROR) << "unable to connect to activity manager";
    return false;
  }
  //设置一个收发时间1s	
  struct timeval tv = {
    .tv_sec = 1,
    .tv_usec = 0,
  };
  //发送时限为1s
  if (setsockopt(amfd.get(), SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv)) == -1) {
    PLOG(ERROR) << "failed to set send timeout on activity manager socket";
    return false;
  }
  tv.tv_sec = 3;
  //接收时限为3s  
  if (setsockopt(amfd.get(), SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv)) == -1) {
    PLOG(ERROR) << "failed to set receive timeout on activity manager socket";
    return false;
  }

  // 活动管理器协议:二进制32位网络字节顺序整数为pid和信号号，然后是转储的原始文本，最后是一个零字节，标志着数据结束。
  //TCP/IP协议栈使用大端字节序,地址用网络字节序来表示,所以应用程序有时需要在处理器的字节序与网络字节序之间转换
  //这里分别表示两个量，一个是pid，另外一个是信号量，分别以四个字节保存到网络字节序中
  //比如pid是2023，信号是11，那么表示为0x 00 00 07 E7 00 00 00 0B
  uint32_t datum = htonl(pid);
  if (!android::base::WriteFully(amfd, &datum, 4)) {
    PLOG(ERROR) << "AM pid write failed";
    return false;
  }
  datum = htonl(signal);
  if (!android::base::WriteFully(amfd, &datum, 4)) {
    PLOG(ERROR) << "AM signal write failed";
    return false;
  }
  //amfd_data为打印的错误信息log
  if (!android::base::WriteFully(amfd, amfd_data.c_str(), amfd_data.size() + 1)) {
    PLOG(ERROR) << "AM data write failed";
    return false;
  }

  // 3s之后接收来自服务端的内容，这里是一个值为0的ack
  char ack;
  android::base::ReadFully(amfd, &ack, 1);
  return true;
}
```

## 3.2demo2

一个具有报文服务的客户端和服务端传输

> **SOCK_SEQPACKET**套接字和SOCK_STREAM套接字很类似，只是从该套接字得到的是基于报文的服务而不是字节流服务。这意味着从SOCK_SEQPACKET套接字接收的数据量与对方所发送的一致。流控制传输协议(Stream Control Transmission Protocol,SCTP)提供了因特网域上的顺序数据包服务。

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_2.png" thumbnail="/socket/socket_linux_2.png" title="">}}

### 3.2.1服务端

服务端的底层原理比较复杂，这里涉及到event原理，笔者不做展开。主要对socket是做一个逻辑上的梳理。

```c++
//system/core/debuggerd/tombstoned/tombstoned.cpp
constexpr char kTombstonedCrashSocketName[] = "tombstoned_crash";
int main(int, char* []) {
  //1.生成一个socket句柄	
  int crash_socket = android_get_control_socket(kTombstonedCrashSocketName);

  if (intercept_socket == -1 || crash_socket == -1) {
    PLOG(FATAL) << "failed to get socket from init";
  }
  //设置服务端的socket句柄不阻塞	
  evutil_make_socket_nonblocking(crash_socket);

  event_base* base = event_base_new();
  if (!base) {
    LOG(FATAL) << "failed to create event_base";
  }
  intercept_manager = new InterceptManager(base, intercept_socket);
  //2.建立客户端与服务端的连接，将句柄和函数crash_accept_cb绑定，一旦有客户端连接调用该函数
  evconnlistener* tombstone_listener =
      evconnlistener_new(base, crash_accept_cb, CrashQueue::for_tombstones(), LEV_OPT_CLOSE_ON_FREE,
                         -1 /* backlog */, crash_socket);
  if (!tombstone_listener) {
    LOG(FATAL) << "failed to create evconnlistener for tombstones.";
  }
  ...
  LOG(INFO) << "tombstoned successfully initialized";
  //循环等待客户端连接
  event_base_dispatch(base);
}
//客户端连接成功，调用到这里
static void crash_accept_cb(evconnlistener* listener, evutil_socket_t sockfd, sockaddr*, int,
                            void*) {
  //3.设置一个监听
  event_base* base = evconnlistener_get_base(listener);
  Crash* crash = new Crash();
  //设置接收的超时时间1s
  struct timeval timeout = { 1, 0 };
  //创建接收函数，重新创建一个函数用于接收
  event* crash_event = event_new(base, sockfd, EV_TIMEOUT | EV_READ, crash_request_cb, crash);
  crash->crash_socket_fd.reset(sockfd);
  crash->crash_event = crash_event;
  event_add(crash_event, &timeout);
}
//接收packet报文函数
static void crash_request_cb(evutil_socket_t sockfd, short ev, void* arg) {
  ssize_t rc;
  Crash* crash = static_cast<Crash*>(arg);

  TombstonedCrashPacket request = {};

  if ((ev & EV_TIMEOUT) != 0) {
    LOG(WARNING) << "crash request timed out";
    goto fail;
  } else if ((ev & EV_READ) == 0) {
    LOG(WARNING) << "tombstoned received unexpected event from crash socket";
    goto fail;
  }
  //3.接收packet的报文
  rc = TEMP_FAILURE_RETRY(read(sockfd, &request, sizeof(request)));
  if (rc == -1) {
    PLOG(WARNING) << "failed to read from crash socket";
    goto fail;
  } 

  if (request.packet_type != CrashPacketType::kDumpRequest) {
    LOG(WARNING) << "unexpected crash packet type, expected kDumpRequest, received  "
                 << StringPrintf("%#2hhX", request.packet_type);
  }

  crash->crash_type = request.packet.dump_request.dump_type;
  if (crash->crash_type < 0 || crash->crash_type > kDebuggerdAnyIntercept) {
    LOG(WARNING) << "unexpected crash dump type: " << crash->crash_type;
  }

  if (crash->crash_type != kDebuggerdJavaBacktrace) {
    crash->crash_pid = request.packet.dump_request.pid;
  } 

  return;
}
```



### 3.2.2客户端

```c++
//system/core/debuggerd/tombstoned/tombstoned_client.cpp
constexpr char kTombstonedCrashSocketName[] = "tombstoned_crash";
enum class CrashPacketType : uint8_t {
  kDumpRequest = 0,
  kCompletedDump,
  kPerformDump = 128,
  kAbortDump,
};

struct DumpRequest {
  DebuggerdDumpType dump_type;
  int32_t pid;
};
//报文的结构体
struct TombstonedCrashPacket {
  CrashPacketType packet_type;
  union {
    DumpRequest dump_request;
  } packet;
};

bool tombstoned_connect(pid_t pid, unique_fd* tombstoned_socket, unique_fd* output_fd,
                        DebuggerdDumpType dump_type) {
  //1.socket连接
  //name为tombstoned_crash
  //id为ANDROID_SOCKET_NAMESPACE_RESERVED,socket路径为/dev/socket/tombstoned_crash
  //SOCK_SEQPACKET为基于报文的服务  
  unique_fd sockfd(
      socket_local_client((dump_type != kDebuggerdJavaBacktrace ? kTombstonedCrashSocketName
                                                                : kTombstonedJavaTraceSocketName),
                          ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_SEQPACKET));
  if (sockfd == -1) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "failed to connect to tombstoned: %s",
                          strerror(errno));
    return false;
  }

  TombstonedCrashPacket packet = {};
  packet.packet_type = CrashPacketType::kDumpRequest;
  packet.packet.dump_request.pid = pid;
  packet.packet.dump_request.dump_type = dump_type;
  //2.发送packet的报文  
  if (TEMP_FAILURE_RETRY(write(sockfd, &packet, sizeof(packet))) != sizeof(packet)) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "failed to write DumpRequest packet: %s",
                          strerror(errno));
    return false;
  }
  ...
  return true;
}
```



## 3.3demo3

这个demo是Android中的日志系统，主要是有两个线程，一个线程用于把日志写入到缓冲区，另一个线程用于把缓冲区的日志读出来。

写的时候使用的是使用的是SOCK_DGRAM，读的时候用到的是SOCK_SEQPACKET，这个demo2已经使用过，就不再介绍。

> 在AF_INET通信域中，套接字类型**SOCK_DGRAM**的默认协议是UDP。对于数据报（SOCK_DGRAM)接口，两个对等进程之间通信时不需要逻辑连接。只需要向对等进程所使用的套接字送出一个报文。因此数据报提供了一个无连接的服务。数据报是自包含报文。发送数据报近似于给某人邮寄信件。你能邮寄很多信，但你不能保证传递的次序，并且可能有些信件会丢失在路上。每封信件包含接收者地址，使这封信件独立于所有其他信件。每封信件可能送达不同的接收者。

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_4.png" thumbnail="/socket/socket_linux_4.png" title="">}}

### 3.3.1服务端

这里的服务端主要是以logd进程开启的两个线程作为服务端，分别是使用UDP协议的logd.writer线程和使用SEQPACKET基于报文的传输的logd.reader线程。

```c++
//system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    ...
    // LogReader监听/dev/socket/logdr .当客户端连接时，LogBuffer中的日志条目被写入客户端。
    LogReader* reader = new LogReader(logBuf);
    if (reader->startListener()) {
        return EXIT_FAILURE;
    }
    // LogListener监听/dev/socket/logdw上客户端发起的日志消息。
    //新的日志条目被添加到LogBuffer中，LogReader被通知向连接的客户端发送更新。
    LogListener* swl = new LogListener(logBuf, reader);
    // Backlog and /proc/sys/net/unix/max_dgram_qlen set to large value
    if (swl->startListener(600)) {
        return EXIT_FAILURE;
    }
    ...
}
```

#### 3.3.1.1LogReader服务端

1）构造LogReader

```c++
//system/core/logd/LogReader.cpp
LogReader::LogReader(LogBuffer* logbuf)
    : SocketListener(getLogSocket(), true), mLogbuf(*logbuf) {
}
//根据上面可以得知实际上是开启一个logdr的socket服务端
int LogReader::getLogSocket() {
    static const char socketName[] = "logdr";
    sock = socket_local_server(
        socketName, ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_SEQPACKET);
    return sock;
}
```

2）基类最先完成构造

```c++
//system/core/libsysutils/src/SocketListener.cpp
SocketListener::SocketListener(int socketFd, bool listen) {
    init(nullptr, socketFd, listen, false);
}

void SocketListener::init(const char *socketName, int socketFd, bool listen, bool useCmdNum) {
    mListen = listen;
    mSocketName = socketName;
    mSock = socketFd;
    mUseCmdNum = useCmdNum;
    pthread_mutex_init(&mClientsLock, nullptr);
}
```

3）LogReader开启logd.reader线程

```c++
//system/core/libsysutils/src/SocketListener.cpp
int SocketListener::startListener() {
    return startListener(4);
}

//这里的mListen构造时候传入就是true
int SocketListener::startListener(int backlog) {
    ...
    //在 Linux 内核 2.2 之后，backlog 是已完成连接建立的队列长度，所以现在通常认为 backlog 是 accept 队列。
    //这里使用的是默认值4
    if (mListen && listen(mSock, backlog) < 0) {
        SLOGE("Unable to listen on socket (%s)", strerror(errno));
        return -1;
    }
    //管道创建，最终用于关闭子线程使用
    if (pipe2(mCtrlPipe, O_CLOEXEC)) {
        SLOGE("pipe failed (%s)", strerror(errno));
        return -1;
    }
    //开启一个子线程
    if (pthread_create(&mThread, nullptr, SocketListener::threadStart, this)) {
        SLOGE("pthread_create (%s)", strerror(errno));
        return -1;
    }

    return 0;
}
//开启一个线程
void *SocketListener::threadStart(void *obj) {
    SocketListener *me = reinterpret_cast<SocketListener *>(obj);

    me->runListener();
    pthread_exit(nullptr);
    return nullptr;
}
//这里使用了poll机制，使得客户端连接能够IO多路复用
void SocketListener::runListener() {
    while (true) {
        std::vector<pollfd> fds;

        pthread_mutex_lock(&mClientsLock);
        fds.reserve(2 + mClients.size());
        fds.push_back({.fd = mCtrlPipe[0], .events = POLLIN});
        if (mListen) fds.push_back({.fd = mSock, .events = POLLIN});
        for (auto pair : mClients) {  
            const int fd = pair.second->getSocket();
            fds.push_back({.fd = fd, .events = POLLIN});
        }
        pthread_mutex_unlock(&mClientsLock);

        int rc = TEMP_FAILURE_RETRY(poll(fds.data(), fds.size(), -1));
        //轮询是否收到管道的关闭或者是唤醒
        if (fds[0].revents & (POLLIN | POLLERR)) {
            char c = CtrlPipe_Shutdown;
            TEMP_FAILURE_RETRY(read(mCtrlPipe[0], &c, 1));
            if (c == CtrlPipe_Shutdown) {
                break;
            }
            continue;
        }
        //这里的fds[1]实际上是logdr的服务端的socket句柄，用来接收客户端的队列，并将队列存在客户端关联容器mClients中
        if (mListen && (fds[1].revents & (POLLIN | POLLERR))) {
            int c = TEMP_FAILURE_RETRY(accept4(mSock, nullptr, nullptr, SOCK_CLOEXEC));
            if (c < 0) {
                SLOGE("accept failed (%s)", strerror(errno));
                sleep(1);
                continue;
            }
            pthread_mutex_lock(&mClientsLock);
            mClients[c] = new SocketClient(c, true, mUseCmdNum);
            pthread_mutex_unlock(&mClientsLock);
        }
        //最终从fds的句柄中找出对应的accept句柄，并且在关联容器mClients中将句柄对应的SocketClient数据结构保存起来
        std::vector<SocketClient*> pending;
        pthread_mutex_lock(&mClientsLock);
        const int size = fds.size();
        for (int i = mListen ? 2 : 1; i < size; ++i) {
            const struct pollfd& p = fds[i];
            if (p.revents & (POLLIN | POLLERR)) {
                auto it = mClients.find(p.fd);
                if (it == mClients.end()) {
                    SLOGE("fd vanished: %d", p.fd);
                    continue;
                }
                SocketClient* c = it->second;
                pending.push_back(c);
                c->incRef();
            }
        }
        pthread_mutex_unlock(&mClientsLock);

        for (SocketClient* c : pending) {
            //最终每一个客户端连接对应获取的accept句柄，都会有独立访问onDataAvailable函数
            SLOGV("processing fd %d", c->getSocket());
            if (!onDataAvailable(c)) {
                release(c, false);
            }
            c->decRef();
        }
    }
}
```

> 1）关于**backlog相关**，可以点击[这里](https://xiaolincoding.com/network/3_tcp/tcp_interview.html)
>
> 补充关于tcp相关的backlog
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_linux_5.png" thumbnail="/socket/socket_linux_5.png" title="">}}
>
> 2）另外关于poll的IO多路复用机制
>
> 上面的poll机制如下图所示
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_linux_6.png" thumbnail="/socket/socket_linux_6.png" title="">}}
>
> poll() 的机制与 select() 类似，与 select() 在本质上没有多大差别，管理多个描述符也是进行**轮询**，根据描述符的状态进行处理。
>
> ```c++
> #include <poll.h>
> int poll(struct pollfd *fds, nfds_t nfds, int timeout);
> ```
>
> 定义的结构体
>
> 1. fd：每一个 pollfd 结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示 poll() 监视多个文件描述符。
> 2. events：表示要告诉操作系统需要监测fd的事件（输入、输出、错误），每一个事件有多个取值
> 3. revents：revents 域是文件描述符的操作结果事件，内核在调用返回时设置这个域。events 域中请求的任何事件都可能在 revents 域中返回。
>
> ```c++
> struct pollfd {
>     int   fd;         /* file descriptor */
>     short events;     /* requested events */
>     short revents;    /* returned events */
> };
> ```
>
> 成员变量说明：
>
> 1. fd：每一个 pollfd 结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示 poll() 监视多个文件描述符。
> 2. events：表示要告诉操作系统需要监测fd的事件（输入、输出、错误），每一个事件有多个取值
> 3. revents：revents 域是文件描述符的操作结果事件，内核在调用返回时设置这个域。events 域中请求的任何事件都可能在 revents 域中返回。
>
> **events&revents的取值如下：**
>
> |    事件    |                      描述                      | 是否可作为输入（events） | 是否可作为输出（revents） |
> | :--------: | :--------------------------------------------: | :----------------------: | :-----------------------: |
> |   POLLIN   |       数据可读（包括普通数据&优先数据）        |            是            |            是             |
> |  POLLOUT   |         数据可写（普通数据&优先数据）          |            是            |            是             |
> | POLLRDNORM |                  普通数据可读                  |            是            |            是             |
> | POLLRDBAND |        优先级带数据可读（linux不支持）         |            是            |            是             |
> |  POLLPRI   |       高优先级数据可读，比如TCP带外数据        |            是            |            是             |
> | POLLWRNORM |                  普通数据可写                  |            是            |            是             |
> | POLLWRBAND |                优先级带数据可写                |            是            |            是             |
> | POLLRDHUP  | TCP连接被对端关闭，或者关闭了写操作，由GNU引入 |            是            |            是             |
> |  POPPHUP   |                      挂起                      |            否            |            是             |
> |  POLLERR   |                      错误                      |            否            |            是             |
>
> > **注意：**
> >
> > 每个结构体的 **events 域**是由**用户**来设置，告诉内核我们关注的是什么，而 revents 域是**返回时内核设置**的，以说明对该描述符发生了什么事件

4）onDataAvailable

```c++
//system/core/logd/LogReader.cpp
bool LogReader::onDataAvailable(SocketClient* cli) {
    static bool name_set;
    if (!name_set) {
        prctl(PR_SET_NAME, "logd.reader");
        name_set = true;
    }
    //读取对应的客户端发送的类似stream lids=0,1,2,3,4,6 timeout=xx start=xx.xx pid=xx
    char buffer[255];
    int len = read(cli->getSocket(), buffer, sizeof(buffer) - 1);
    buffer[len] = '\0';
    //这里开始解析收到的socket数据
    static const char _tail[] = " tail=";
    char* cp = strstr(buffer, _tail);
    tail = atol(cp + sizeof(_tail) - 1);
    static const char _start[] = " start=";
    cp = strstr(buffer, _start);
    start.strptime(cp + sizeof(_start) - 1, "%s.%q");
    static const char _timeout[] = " timeout=";
    cp = strstr(buffer, _timeout);
    timeout = atol(cp + sizeof(_timeout) - 1) * NS_PER_SEC +
                  log_time(CLOCK_REALTIME).nsec();
    static const char _logIds[] = " lids=";
    cp = strstr(buffer, _logIds);
    logMask = 0;
    cp += sizeof(_logIds) - 1;
    while (*cp && *cp != '\0') {
        int val = 0;
        while (isdigit(*cp)) {
            val = val * 10 + *cp - '0';
            ++cp;
        }
        logMask |= 1 << val;
        if (*cp != ',') {
            break;
        }
        ++cp;
    }
    static const char _pid[] = " pid=";
    cp = strstr(buffer, _pid);
    pid = atol(cp + sizeof(_pid) - 1);
    ...
    //开始发送数据给客户端，客户端就可以收到对应的日志了
    logbuf().flushTo(cli, sequence, nullptr, FlushCommand::hasReadLogs(cli),
                     FlushCommand::hasSecurityLogs(cli),
                     logFindStart.callback, &logFindStart);
    ...
    return true;
}
```

5）flushTo

```c++
//system/core/logd/LogBufferElement.cpp
log_time LogBufferElement::flushTo(SocketClient* reader, LogBuffer* parent, bool lastSame) {
    struct logger_entry entry = {};

    entry.hdr_size = sizeof(struct logger_entry);
    entry.lid = mLogId;
    entry.pid = mPid;
    entry.tid = mTid;
    entry.uid = mUid;
    entry.sec = mRealTime.tv_sec;
    entry.nsec = mRealTime.tv_nsec;

    struct iovec iovec[2];
    iovec[0].iov_base = &entry;
    iovec[0].iov_len = entry.hdr_size;

    char* buffer = nullptr;

    if (mDropped) {
        entry.len = populateDroppedMessage(buffer, parent, lastSame);
        if (!entry.len) return mRealTime;
        iovec[1].iov_base = buffer;
    } else {
        entry.len = mMsgLen;
        iovec[1].iov_base = mMsg;
    }
    iovec[1].iov_len = entry.len;
    //最终在这里发送给客户端
    log_time retval = reader->sendDatav(iovec, 1 + (entry.len != 0))
                          ? FLUSH_ERROR
                          : mRealTime;

    if (buffer) free(buffer);

    return retval;
}
```

#### 3.3.1.2LogListener服务端

1）构造LogListener

```c++
//system/core/logd/LogListener.cpp
//与上面的区别是这里在基类构造SocketListener传入false，等同mListener为false
LogListener::LogListener(LogBuffer* buf, LogReader* reader)
    : SocketListener(getLogSocket(), false), logbuf(buf), reader(reader) {}
//根据上面可以得知实际上是开启一个logdw的socket服务端
int LogListener::getLogSocket() {
    static const char socketName[] = "logdw";
    sock = socket_local_server(
        socketName, ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_DGRAM);
    int on = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on))) {
        return -1;
    }
    return sock;
}
```

> UDP的连接，设置SO_PASSCRED
>
> 在辅助消息中启用接收发送过程的凭据。当设置了此选项并且套接字尚未连接时，将**自动生成抽象名称空间中的唯一名称**。期望一个整数布尔标志。



2）LogListener开启logd.writer线程

```c++
//system/core/libsysutils/src/SocketListener.cpp
//与logd.reader线程区别是backlog这里传入了600，且传入的mListen=false
int SocketListener::startListener(int backlog) {
    if (!mListen)
        //这里默认就会创建一个SocketClient对象作为客户端接收的句柄
        mClients[mSock] = new SocketClient(mSock, false, mUseCmdNum);
    //管道创建，最终用于关闭子线程使用
    if (pipe2(mCtrlPipe, O_CLOEXEC)) {
        SLOGE("pipe failed (%s)", strerror(errno));
        return -1;
    }
    //创建子线程
    if (pthread_create(&mThread, nullptr, SocketListener::threadStart, this)) {
        SLOGE("pthread_create (%s)", strerror(errno));
        return -1;
    }

    return 0;
}

...
//这个线程中没有accept，因为是UDP协议，另外多路复用主要是监听管道句柄和线程创建前的一个SocketClient对象作为客户端接收的句柄 
void SocketListener::runListener() {
    while (true) {
        std::vector<pollfd> fds;

        pthread_mutex_lock(&mClientsLock);
        fds.reserve(2 + mClients.size());
        fds.push_back({.fd = mCtrlPipe[0], .events = POLLIN});
        for (auto pair : mClients) {
            const int fd = pair.second->getSocket();
            fds.push_back({.fd = fd, .events = POLLIN});
        }
        pthread_mutex_unlock(&mClientsLock);

        int rc = TEMP_FAILURE_RETRY(poll(fds.data(), fds.size(), -1));
        if (fds[0].revents & (POLLIN | POLLERR)) {
            char c = CtrlPipe_Shutdown;
            TEMP_FAILURE_RETRY(read(mCtrlPipe[0], &c, 1));
            if (c == CtrlPipe_Shutdown) {
                break;
            }
            continue;
        }

        std::vector<SocketClient*> pending;
        pthread_mutex_lock(&mClientsLock);
        //实际上size只有2，位置0位管道句柄，位置1为线程创建前的一个SocketClient对象作为客户端接收的句柄
        const int size = fds.size();
        for (int i = mListen ? 2 : 1; i < size; ++i) {
            const struct pollfd& p = fds[i];
            if (p.revents & (POLLIN | POLLERR)) {
                auto it = mClients.find(p.fd);
                if (it == mClients.end()) {
                    SLOGE("fd vanished: %d", p.fd);
                    continue;
                }
                SocketClient* c = it->second;
                pending.push_back(c);
                c->incRef();
            }
        }
        pthread_mutex_unlock(&mClientsLock);

        for (SocketClient* c : pending) {
            // Process it, if false is returned, remove from the map
            SLOGV("processing fd %d", c->getSocket());
            if (!onDataAvailable(c)) {
                release(c, false);
            }
            c->decRef();
        }
    }
}
```

3）onDataAvailable

```c++
//system/core/logd/LogListener.cpp
bool LogListener::onDataAvailable(SocketClient* cli) {
    static bool name_set;
    if (!name_set) {
        prctl(PR_SET_NAME, "logd.writer");
        name_set = true;
    }
    
    char buffer[sizeof(android_log_header_t) + LOGGER_ENTRY_MAX_PAYLOAD + 1];
    struct iovec iov = { buffer, sizeof(buffer) - 1 };
    alignas(4) char control[CMSG_SPACE(sizeof(struct ucred))];
    struct msghdr hdr = {
        nullptr, 0, &iov, 1, control, sizeof(control), 0,
    };

    int socket = cli->getSocket();
    //接收来自客户端的数据
    ssize_t n = recvmsg(socket, &hdr, 0);
    //处理保存数据到缓存中
    int res = logbuf->log(logId, header->realtime, cred->uid, cred->pid, header->tid, msg,
                          ((size_t)n <= UINT16_MAX) ? (uint16_t)n : UINT16_MAX);
    if (res > 0) {
        //通知另外一个logd.reader线程开始读取缓存数据
        reader->notifyNewLog(static_cast<log_mask_t>(1 << logId));
    }

    return true;
}
```



### 3.3.2客户端

客户端其实就是进程使用liblog动态库

#### 3.3.2.1日志从缓存中读取

读取通常是用户通过logcat来主动获取缓存中的logcat数据

```c++
//system/core/logcat/logcat.cpp
int Logcat::Run(int argc, char** argv)
{
    ...
    while (!max_count_ || print_count_ < max_count_) {
        ...
        int ret = android_logger_list_read(logger_list, &log_msg);  
        ...
    }  
    ...
}
```

1）android_logger_list_read

```c++
//system/core/liblog/logger_read.cpp
int android_logger_list_read(struct logger_list* logger_list, struct log_msg* log_msg) {
  ...
  if (logger_list->mode & ANDROID_LOG_PSTORE) {
    ret = PmsgRead(logger_list, log_msg);
  } else {
    //读取logcat LOG_ID_MAIN主日志
    ret = LogdRead(logger_list, log_msg);
  }
  ...
  return ret;
}
```

2）LogdRead

这里主要有两个操作

1. logdOpen获取路径为/dev/socket/logdr的socket句柄，并发送一次数据给服务端
2. recv接收数据结构为log_msg的logcat数据

```c++
//system/core/liblog/logd_reader.cpp
int LogdRead(struct logger_list* logger_list, struct log_msg* log_msg) {
  int ret = logdOpen(logger_list);
  /* NOTE: SOCK_SEQPACKET guarantees we read exactly one full entry */
  ret = TEMP_FAILURE_RETRY(recv(ret, log_msg, LOGGER_ENTRY_MAX_LEN, 0));
  ...
  return ret;
}
```

logdOpen

```c++
static int logdOpen(struct logger_list* logger_list) {
  char buffer[256], *cp, c;
  int ret, remaining, sock;

  sock = atomic_load(&logger_list->fd);
  if (sock > 0) {
    return sock;
  }
  //创建以报文传输的socket句柄的客户端，路径为/dev/socket/logdr
  sock = socket_local_client("logdr", SOCK_SEQPACKET);
  //开始拼接字符串，这里以stream开头，类似stream lids=0,1,2,3,4,6 timeout=xx start=xx.xx pid=xx
  strcpy(buffer, (logger_list->mode & ANDROID_LOG_NONBLOCK) ? "dumpAndClose" : "stream");
  cp = buffer + strlen(buffer);
  strcpy(cp, " lids");
  cp += 5;
  c = '=';
  remaining = sizeof(buffer) - (cp - buffer);
  for (size_t log_id = 0; log_id < LOG_ID_MAX; ++log_id) {
    if ((1 << log_id) & logger_list->log_mask) {
      ret = snprintf(cp, remaining, "%c%zu", c, log_id);
      ret = MIN(ret, remaining);
      remaining -= ret;
      cp += ret;
      c = ',';
    }
  }

  if (logger_list->tail) {
    ret = snprintf(cp, remaining, " tail=%u", logger_list->tail);
    ret = MIN(ret, remaining);
    remaining -= ret;
    cp += ret;
  }

  if (logger_list->start.tv_sec || logger_list->start.tv_nsec) {
    if (logger_list->mode & ANDROID_LOG_WRAP) {
      // ToDo: alternate API to allow timeout to be adjusted.
      ret = snprintf(cp, remaining, " timeout=%u", ANDROID_LOG_WRAP_DEFAULT_TIMEOUT);
      ret = MIN(ret, remaining);
      remaining -= ret;
      cp += ret;
    }
    ret = snprintf(cp, remaining, " start=%" PRIu32 ".%09" PRIu32, logger_list->start.tv_sec,
                   logger_list->start.tv_nsec);
    ret = MIN(ret, remaining);
    remaining -= ret;
    cp += ret;
  }

  if (logger_list->pid) {
    ret = snprintf(cp, remaining, " pid=%u", logger_list->pid);
    ret = MIN(ret, remaining);
    cp += ret;
  }
  //发送上述字符串给服务端
  ret = TEMP_FAILURE_RETRY(write(sock, buffer, cp - buffer));
  int write_errno = errno;
  ...
  return sock;
}
```



#### 3.3.2.2日志写入到缓存中

```c++
//system/core/liblog/logger_write.cpp
//LOG_ID_MAIN为主日志，prio为日志的等级，Verbose、Debug、Info、Warning、Error、Fatal、Silent(不输出)
int __android_log_write(int prio, const char* tag, const char* msg) {
    return __android_log_buf_write(LOG_ID_MAIN, prio, tag, msg);
}

int __android_log_buf_write(int bufID, int prio, const char* tag, const char* msg) {
  ...
  __android_log_message log_message = {
      sizeof(__android_log_message), bufID, prio, tag, nullptr, 0, msg};
  __android_log_write_log_message(&log_message);
  return 1;
}

void __android_log_write_log_message(__android_log_message* log_message) {
  ...
  logger_function(log_message);
}
```

又因为logger_function在初始化的时候就已经赋值，这里直接找到赋值的地方

```c++
//system/core/liblog/logger_write.cpp
#ifdef __ANDROID__
static __android_logger_function logger_function = __android_log_logd_logger;
#else
static __android_logger_function logger_function = __android_log_stderr_logger;
#endif
```

最终调用__android_log_logd_logger

```c++
//system/core/liblog/logger_write.cpp
void __android_log_logd_logger(const struct __android_log_message* log_message) {
  int buffer_id = log_message->buffer_id == LOG_ID_DEFAULT ? LOG_ID_MAIN : log_message->buffer_id;

  struct iovec vec[3];
  vec[0].iov_base =
      const_cast<unsigned char*>(reinterpret_cast<const unsigned char*>(&log_message->priority));
  vec[0].iov_len = 1;
  vec[1].iov_base = const_cast<void*>(static_cast<const void*>(log_message->tag));
  vec[1].iov_len = strlen(log_message->tag) + 1;
  vec[2].iov_base = const_cast<void*>(static_cast<const void*>(log_message->message));
  vec[2].iov_len = strlen(log_message->message) + 1;
  //buffer_id传入的就是LOG_ID_MAIN
  write_to_log(static_cast<log_id_t>(buffer_id), vec, 3);
}

static int write_to_log(log_id_t log_id, struct iovec* vec, size_t nr) {
  ...
  ret = LogdWrite(log_id, &ts, vec, nr);
  ...
  return ret;
}
```

LogdWrite

```c++
//system/core/liblog/logd_writer.cpp
int LogdWrite(log_id_t logId, struct timespec* ts, struct iovec* vec, size_t nr) {
  ssize_t ret;
  static const unsigned headerLength = 1;
  struct iovec newVec[nr + headerLength];
  android_log_header_t header;
  size_t i, payloadSize;
  static atomic_int dropped;
  static atomic_int droppedSecurity;
  //获取客户端的socket句柄
  GetSocket();

  header.tid = gettid();
  header.realtime.tv_sec = ts->tv_sec;
  header.realtime.tv_nsec = ts->tv_nsec;

  newVec[0].iov_base = (unsigned char*)&header;
  newVec[0].iov_len = sizeof(header);

  int32_t snapshot = atomic_exchange_explicit(&droppedSecurity, 0, memory_order_relaxed);
  if (snapshot) {
    android_log_event_int_t buffer;

    header.id = LOG_ID_SECURITY;
    buffer.header.tag = LIBLOG_LOG_TAG;
    buffer.payload.type = EVENT_TYPE_INT;
    buffer.payload.data = snapshot;

    newVec[headerLength].iov_base = &buffer;
    newVec[headerLength].iov_len = sizeof(buffer);
    //客户端发送给服务端
    ret = TEMP_FAILURE_RETRY(writev(logd_socket, newVec, 2));
    if (ret != (ssize_t)(sizeof(header) + sizeof(buffer))) {
      atomic_fetch_add_explicit(&droppedSecurity, snapshot, memory_order_relaxed);
    }
  }
  ...
  return ret;
}
//获取客户端的socket句柄
static void GetSocket() {
  int new_socket =
      TEMP_FAILURE_RETRY(socket(PF_UNIX, SOCK_DGRAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0));
  LogdConnect();
}

static void LogdConnect() {
  //使用UDP协议的本地socket，路径为/dev/socket/logdw
  sockaddr_un un = {};
  un.sun_family = AF_UNIX;
  strcpy(un.sun_path, "/dev/socket/logdw");
  TEMP_FAILURE_RETRY(connect(logd_socket, reinterpret_cast<sockaddr*>(&un), sizeof(sockaddr_un)));
}
```



## 3.4demo4

这个demo是SElinux中的日志打印相关的socket，使用用户空间和内核空间的通信，且使用数据报接口用于直接访问网络层。

> **SOCK RAW**套接字提供一个**数据报接口**，用于直接访问下面的网络层（即因特网域中的P层)。使用这个接口时，应用程序负责构造自己的协议头部，这是因为传输协议（如TCP和UDP)被绕过了。当创建一个原始套接字时，需要有超级用户特权，这样可以防止恶意应用程序绕过内建安全机制来创建报文。

{{< image classes="fancybox center fig-100" src="/socket/socket_linux_3.png" thumbnail="/socket/socket_linux_3.png" title="">}}

### 3.4.1服务端

```c++
//kernel/audit.c
static int __net_init audit_net_init(struct net *net)
{
	struct netlink_kernel_cfg cfg = {
        //2.这里的input就是用来接收用户空间的数据
		.input	= audit_receive,
		.bind	= audit_multicast_bind,
		.unbind	= audit_multicast_unbind,
		.flags	= NL_CFG_F_NONROOT_RECV,
		.groups	= AUDIT_NLGRP_MAX,
	};

	struct audit_net *aunet = net_generic(net, audit_net_id);
    //1.建立连接NETLINK_AUDIT，采用了异步方式
	aunet->sk = netlink_kernel_create(net, NETLINK_AUDIT, &cfg);
	if (aunet->sk == NULL) {
		audit_panic("cannot initialize netlink socket in namespace");
		return -ENOMEM;
	}
	/* limit the timeout in case auditd is blocked/stopped */
	aunet->sk->sk_sndtimeo = HZ / 10;

	return 0;
}
```

其中NETLINK_AUDIT的定义

```c++
//include/uapi/linux/netlink.h
#define NETLINK_AUDIT		9	/* auditing */
```

netlink_unicast采用的是单播消息

```c++
//kernel/audit.c
static int audit_multicast_bind(struct net *net, int group)
{
	int err = 0;

	if (!capable(CAP_AUDIT_READ))
		err = -EPERM;
	audit_log_multicast(group, "connect", err);
	return err;
}


static void audit_receive(struct sk_buff  *skb)
{
	struct nlmsghdr *nlh;
	int len;
	int err;

	nlh = nlmsg_hdr(skb);
	len = skb->len;

	audit_ctl_lock();
	while (nlmsg_ok(nlh, len)) {
		err = audit_receive_msg(skb, nlh);
		/* if err or if this message says it wants a response */
		if (err || (nlh->nlmsg_flags & NLM_F_ACK))
			netlink_ack(skb, nlh, err, NULL);

		nlh = nlmsg_next(nlh, &len);
	}
	audit_ctl_unlock();
    ...
}
//这里通常只有三种常见的AVC，初始化的时候用得AUDIT_SET(1001),与属性相关的AUDIT_USER_AVC(1107),其他AUDIT_AVC(1400)
static int audit_receive_msg(struct sk_buff *skb, struct nlmsghdr *nlh)
{
	u32			seq;
	void			*data;
	int			data_len;
	int			err;
	struct audit_buffer	*ab;
	u16			msg_type = nlh->nlmsg_type;
	struct audit_sig_info   *sig_data;
	char			*ctx = NULL;
	u32			len;

	err = audit_netlink_ok(skb, msg_type);
	if (err)
		return err;

	seq  = nlh->nlmsg_seq;
	data = nlmsg_data(nlh);
	data_len = nlmsg_len(nlh);

	switch (msg_type) {
	case AUDIT_GET: {
		struct audit_status	s;
		memset(&s, 0, sizeof(s));
		s.enabled		   = audit_enabled;
		s.failure		   = audit_failure;
		/* NOTE: use pid_vnr() so the PID is relative to the current
		 *       namespace */
		s.pid			   = auditd_pid_vnr();
		s.rate_limit		   = audit_rate_limit;
		s.backlog_limit		   = audit_backlog_limit;
		s.lost			   = atomic_read(&audit_lost);
		s.backlog		   = skb_queue_len(&audit_queue);
		s.feature_bitmap	   = AUDIT_FEATURE_BITMAP_ALL;
		s.backlog_wait_time	   = audit_backlog_wait_time;
		s.backlog_wait_time_actual = atomic_read(&audit_backlog_wait_time_actual);
		audit_send_reply(skb, seq, AUDIT_GET, 0, 0, &s, sizeof(s));
		break;
	}
	case AUDIT_SET: {
		...
		break;
	}
	case AUDIT_USER:
	case AUDIT_FIRST_USER_MSG ... AUDIT_LAST_USER_MSG:
	case AUDIT_FIRST_USER_MSG2 ... AUDIT_LAST_USER_MSG2:
		...
		break;
	default:
		err = -EINVAL;
		break;
	}
	return err < 0 ? err : 0;
}
```



### 3.4.2客户端

随便的一句log输出

```c++
//external/selinux/libselinux/src/selinux_restorecon.c
selinux_log(SELINUX_INFO,
 		     "filespec hash table stats: %d elements, %d/%d buckets used, longest chain length %d\n",
 		     nel, used, HASH_BUCKETS, longest);
```

或者是AVC log

```c++
//external/selinux/libselinux/src/avc.c
avc_log(SELINUX_AVC, "%s", avc_audit_buf);
```

avc_log调用

```c++
//external/selinux/libselinux/src/avc_internal.h
//因为设置了avc_func_log实际上就是selinux_log
#define avc_log(type, format...) \
 if (avc_func_log) \
   avc_func_log(format); \
 else \
   selinux_log(type, format);
```

反推到是否设置callback，没有设置按照默认stderr输出，设置则按照设置的输出

```c++
//external/selinux/libselinux/src/callbacks.c
//如果callback没有设置任何值，默认按照stderr标准错误来输出
static int __attribute__ ((format(printf, 2, 3)))
default_selinux_log(int type __attribute__((unused)), const char *fmt, ...)
{
	int rc;
	va_list ap;
	va_start(ap, fmt);
	rc = vfprintf(stderr, fmt, ap);
	va_end(ap);
	return rc;
}

static int
default_selinux_audit(void *ptr __attribute__((unused)),
		      security_class_t cls __attribute__((unused)),
		      char *buf __attribute__((unused)),
		      size_t len __attribute__((unused)))
{
	return 0;
}

int (*selinux_audit) (void *, security_class_t, char *, size_t) =
	default_selinux_audit;
int __attribute__ ((format(printf, 2, 3)))
(*selinux_log)(int, const char *, ...) =
	default_selinux_log;

/* callback setting function */
void selinux_set_callback(int type, union selinux_callback cb)
{
	switch (type) {
	case SELINUX_CB_LOG:
		selinux_log = cb.func_log;
		break;
	case SELINUX_CB_AUDIT:
		selinux_audit = cb.func_audit;
		break;
	case SELINUX_CB_VALIDATE:
		selinux_validate = cb.func_validate;
		break;
	case SELINUX_CB_SETENFORCE:
		selinux_netlink_setenforce = cb.func_setenforce;
		break;
	case SELINUX_CB_POLICYLOAD:
		selinux_netlink_policyload = cb.func_policyload;
		break;
	}
}
```

继续反推，Android会在开机的时候设置callback，所以日志打印会在SelinuxKlogCallback体现

```c++
//system/core/init/selinux.cpp
如果callback设置为func_log
void SelinuxSetupKernelLogging() {
    selinux_callback cb;
    cb.func_log = SelinuxKlogCallback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
}
//日志打印会在这里体现
int SelinuxKlogCallback(int type, const char* fmt, ...) {
    android::base::LogSeverity severity = android::base::ERROR;
    if (type == SELINUX_WARNING) {
        severity = android::base::WARNING;
    } else if (type == SELINUX_INFO) {
        severity = android::base::INFO;
    }
    char buf[kKlogMessageSize];
    va_list ap;
    va_start(ap, fmt);
    int length_written = vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);
    if (length_written <= 0) {
        return 0;
    }
    //如果是AVC log
    if (type == SELINUX_AVC) {
        SelinuxAvcLog(buf, sizeof(buf));
    } else {
        //如果是其他的log
        android::base::KernelLogger(android::base::MAIN, severity, "selinux", nullptr, 0, buf);
    }
    return 0;
}
```

1）如果是AVC log

```c++
//system/core/init/selinux.cpp
constexpr size_t kKlogMessageSize = 1024;
void SelinuxAvcLog(char* buf, size_t buf_len) {
    CHECK_GT(buf_len, 0u);

    size_t str_len = strnlen(buf, buf_len);
    // trim newline at end of string
    if (buf[str_len - 1] == '\n') {
        buf[str_len - 1] = '\0';
    }

    struct NetlinkMessage {
        nlmsghdr hdr;
        char buf[kKlogMessageSize];
    } request = {};

    request.hdr.nlmsg_flags = NLM_F_REQUEST;
    request.hdr.nlmsg_type = AUDIT_USER_AVC;
    request.hdr.nlmsg_len = sizeof(request);
    strlcpy(request.buf, buf, sizeof(request.buf));
	//1.创建socket句柄
    //PF_NETLINK是是一种在内核和用户应用间进行双向数据传输的非常好的方式
    //SOCK_RAW用于直接访问网络层
    //NETLINK_AUDIT定义的一个方式，NETLINK_AUDIT=9
    auto fd = unique_fd{socket(PF_NETLINK, SOCK_RAW | SOCK_CLOEXEC, NETLINK_AUDIT)};
    if (!fd.ok()) {
        return;
    }
    //2.数据报接口发送
    TEMP_FAILURE_RETRY(send(fd, &request, sizeof(request), 0));
}
```

> 其中nlmsghdr的定义
>
> ```c++
> //bionic/libc/kernel/uapi/linux/netlink.h
> struct nlmsghdr {
>   __u32 nlmsg_len;
>   __u16 nlmsg_type;
>   __u16 nlmsg_flags;
>   __u32 nlmsg_seq;
>   __u32 nlmsg_pid;
> };
> ```

2）如果是其他的log

```c++
//system/core/base/logging.cpp
void KernelLogger(android::base::LogId, android::base::LogSeverity severity, const char* tag,
                  const char*, unsigned int, const char* full_message) {
  SplitByLines(full_message, KernelLogLine, severity, tag);
}

static void KernelLogLine(const char* msg, int length, android::base::LogSeverity severity,
                          const char* tag) {
  // 设置日志等级
  static constexpr int kLogSeverityToKernelLogLevel[] = {
      [android::base::VERBOSE] = 7,              // KERN_DEBUG (there is no verbose kernel log
                                                 //             level)
      [android::base::DEBUG] = 7,                // KERN_DEBUG
      [android::base::INFO] = 6,                 // KERN_INFO
      [android::base::WARNING] = 4,              // KERN_WARNING
      [android::base::ERROR] = 3,                // KERN_ERROR
      [android::base::FATAL_WITHOUT_ABORT] = 2,  // KERN_CRIT
      [android::base::FATAL] = 2,                // KERN_CRIT
  };
  // clang-format on
  static_assert(arraysize(kLogSeverityToKernelLogLevel) == android::base::FATAL + 1,
                "Mismatch in size of kLogSeverityToKernelLogLevel and values in LogSeverity");
  //1.打开Kmsg句柄,这个并不是socket句柄，是linux中的文件句柄
  static int klog_fd = OpenKmsg();
  if (klog_fd == -1) return;

  int level = kLogSeverityToKernelLogLevel[severity];
  char buf[1024] __attribute__((__uninitialized__));
  size_t size = snprintf(buf, sizeof(buf), "<%d>%s: %.*s\n", level, tag, length, msg);
  if (size > sizeof(buf)) {
    size = snprintf(buf, sizeof(buf), "<%d>%s: %zu-byte message too long for printk\n",
                    level, tag, size);
  }

  iovec iov[1];
  iov[0].iov_base = buf;
  iov[0].iov_len = size;
  //2.文件中写入内容
  TEMP_FAILURE_RETRY(writev(klog_fd, iov, 1));
}

static int OpenKmsg() {
  // pick up 'file w /dev/kmsg' environment from daemon's init rc file
  
  const auto val = getenv("ANDROID_FILE__dev_kmsg");
  if (val != nullptr) {
    int fd;
    if (android::base::ParseInt(val, &fd, 0)) {
      auto flags = fcntl(fd, F_GETFL);
      if ((flags != -1) && ((flags & O_ACCMODE) == O_WRONLY)) return fd;
    }
  }
  //1.连接句柄，找到/dev/kmsg
  return TEMP_FAILURE_RETRY(open("/dev/kmsg", O_WRONLY | O_CLOEXEC));
}
```

调用SplitByLines函数

```c++
//system/core/base/logging_splitters.h
template <typename F, typename... Args>
static void SplitByLines(const char* msg, const F& log_function, Args&&... args) {
  const char* newline = strchr(msg, '\n');
  while (newline != nullptr) {
    log_function(msg, newline - msg, args...);
    msg = newline + 1;
    newline = strchr(msg, '\n');
  }
  //log_function最终会调用到KernelLogLine中
  log_function(msg, -1, args...);
}
```



# 4总结

本文介绍了Linux socket，包含以TCP和UDP协议传输层的socket用法，还介绍了几种Android中的socket用法，包含Java层，native层，还衍生出了一套封装的函数socket_local_client和socket_local_server，可以更简单的使用socket。



# 参考

[[1] yjy239, Android socket源码解析(一)socket的初始化原理, 2021.](https://www.jianshu.com/p/46efb4b102d9)

[[2] yjy239, Android socket源码解析(二)socket的绑定与监听, 2021.](https://www.jianshu.com/p/62dd608667e2)

[[3] yjy239, Android socket源码解析(三)socket的connect源码解析, 2021.](https://www.jianshu.com/p/da6089fdcfe1)

[[4] 段细狗, Linux中的socket编程, 2022.](https://blog.csdn.net/weixin_36828200/article/details/123754901?ops_request_misc=&request_id=&biz_id=102&utm_term=linux%20socket&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-123754901.nonecase&spm=1018.2226.3001.4187)

[[5] 李白不喝酒777777, 进程间通信——SOCKET（TCP）, 2022.](https://blog.csdn.net/m0_53311279/article/details/125317083)

[[6] 李白不喝酒777777, 进程间通信——SOCKET（UDP）, 2022.](https://blog.csdn.net/m0_53311279/article/details/125441819)

[[7] jiatingqiang, sock结构和socket结构详解, 2011.](https://blog.csdn.net/jiatingqiang/article/details/6438096)

[[8] linuxheik, writev, 2017.](https://blog.csdn.net/linuxheik/article/details/76125411)

[[9] winsonCCCC, Linux网络编程之sockaddr与sockaddr_in,sockaddr_un分析, 2021.](https://blog.csdn.net/weixin_43829445/article/details/116497954)

[[10] iteye_19603, 整理：Linux网络编程之sockaddr与sockaddr_in,sockaddr_un结构体详细讲解, 2013.](https://blog.csdn.net/iteye_19603/article/details/82486053)

[[11] 低调小一, UNIX Domain Socket IPC, 2015.](https://wangzhengyi.blog.csdn.net/article/details/44928691)

[[12] 低调小一 Linux C编程一站式学习读书笔记——socket编程, 2013.](https://wangzhengyi.blog.csdn.net/article/details/9226413)

[[13] linglongbayinhe, AF_INET与套接字, 2018.](https://blog.csdn.net/linglongbayinhe/article/details/83214171)

[[14] 一口Linux, Linux SOCKET介绍, 2021.](https://blog.csdn.net/daocaokafei/article/details/115091271)

[[15] 转角心静了, Linux下的Socket通信, 2021.](https://blog.csdn.net/qq_46323094/article/details/117704573)

[[16] penguin1990, LocalSocket实现进程间通信, 2016.](https://blog.csdn.net/daozi22/article/details/53344134)

[[17] IT先森, Android Java层和Native层通信实战大荟萃之原生socket实现通信, 2020.](https://blog.csdn.net/tkwxty/article/details/103064321)

[[18] a370352701, 人工智能 图解linux netlink, 2019.](https://www.dazhuanlan.com/a370352701/topics/1014749)

[[19] Shining-LY, poll方法的基本概念, 2018.](https://blog.csdn.net/qq_37964547/article/details/80697530)

[[20] linuxheik, writev, 2017.](https://blog.csdn.net/linuxheik/article/details/76125411)

[[21] yunfan188, Linux网络编程 - 多种 I/O 函数(send、recv、readv、writev), 2022.](https://blog.csdn.net/u010429831/article/details/122452030)