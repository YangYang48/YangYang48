---
title: "select、poll、epoll这三种多路复用的技术原理"
date: 2023-08-06
thumbnailImagePosition: left
thumbnailImage: socket/io/io_thumb.jpg
coverImage: socket/io/io_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- io多路复用
- 2023
- August
tags:
- socket
- select
- poll
- epoll
- property_service
showSocial: false
---

之前介绍了关于epoll机制，实际上除了epoll以外，还存在两个io多路复用的系统调用函数select和poll


<!--more-->
# 1简介

## 1.1select

select, pselect, FD_CLR, FD_ISSET, FD_SET, FD_ZERO -同步I/O多路复用

### 1.1.1函数原型

```c++
/* 根据POSIX.1-2001, POSIX.1-2008标准 */
#include <sys/select.h>

/* 根据以前的标准 */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```



### 1.1.2描述

select()和pselect()允许程序监视多个文件描述符，直到其中一个或多个文件描述符为某类I/O操作(例如，输入成为可能)“准备就绪”。如果有可能执行相应的I/O操作(例如，不阻塞地读(2)，或足够小的写(2))，则认为文件描述符就绪。

除了以下三点不同之外，select()和pselect()的操作是相同的。

1. select()使用的超时时间是结构体timeval(包含秒和微秒)，而pselect()使用结构体timespec(包含秒和纳秒)。
2.  select()可以更新timeout参数以指示剩余的时间。Pselect()不会改变这个参数

3.  select()没有sigmask参数，并且表现为pselect(调用时sigmask为NULL)

我们观察了三组独立的文件描述符。readfds中列出的字符将被监视，以查看是否有字符可供读取(更准确地说，查看读取是否不会阻塞;特别是，在文件末尾也准备了一个文件描述符)，writefds中的文件描述符将被监视，以查看是否有空间可供写入(尽管大的写入仍然可能阻塞)，而exceptfds中的文件描述符将被监视以查看异常。在退出时，这些集合将被修改，以表明哪些文件描述符实际上更改了状态。如果没有监视对应类的文件描述符，则这3个文件描述符集都可以指定为NULL。

提供了4个宏来操作这些集合。FD_ZERO()清除一个集合FD_SET()和FD_CLR()分别从集合中添加和删除给定的文件描述符。FD_ISSET()测试文件描述符是否属于集合;这在select()返回后很有用。NFDS是三个集合中编号最高的文件描述符，加1。

timeout参数指定了select()阻塞等待文件描述符准备就绪的时间间隔。call will阻塞直到其中之一:

- **文件描述符准备就绪;**
- **该调用被信号处理程序中断**
- **超时到期**。

请注意，超时时间间隔会向上取整到系统时钟粒度，而内核调度延迟意味着阻塞时间间隔可能会超出一小部分。如果timeval结构体的两个字段都为0，则select()立即返回。(这对轮询很有用。)如果timeout为NULL(没有超时)，select()会无限阻塞。

Sigmask是一个指向信号掩码的指针(参见sigprocmask(2))。如果它不是NULL，那么pselect()首先用sigmask指向的掩码替换当前的信号掩码，然后执行“选择”函数，然后还原原始信号掩码。除了timeout参数的精度不同，下面的pselect()调用:

```c++
ready = pselect(nfds, &readfds, &writefds, &exceptfds,
                timeout, &sigmask);
```

等价于原子执行以下调用:

```c++
sigset_t origmask;

pthread_sigmask(SIG_SETMASK, &sigmask, &origmask);
ready = select(nfds, &readfds, &writefds, &exceptfds, timeout);
pthread_sigmask(SIG_SETMASK, &origmask, NULL);
```

需要pselect()的原因是，如果想等待信号或文件描述符就绪，则需要一个原子测试来防止竞态条件。(假设信号处理程序设置了一个全局标志并返回。然后，如果信号正好在测试之后、调用之前到达，那么在测试这个全局标志之后再调用select()方法可能会无限期地挂起。

相比之下，pselect()允许用户首先阻塞信号，处理传入的信号，然后使用所需的sigmask调用pselect()，从而避免竞争。)

```c++
//超时,涉及的时间结构定义在<sys/time.h>中，类似于
struct timeval {
    long    tv_sec;         /* seconds */
    long    tv_usec;        /* microseconds */
};

struct timespec {
    long    tv_sec;         /* seconds */
    long    tv_nsec;        /* nanoseconds */
};
```

有些代码在调用select()时将所有三个集合都设置为空，nfds为零，超时设置为非null，这是一种相当可移植的以亚秒精度睡眠的方式。

在Linux上，select()修改timeout以反映未睡眠的时间;大多数其他实现不会这样做。(POSIX.1允许两种行为。)当将读取timeout的Linux代码移植到其他操作系统时，以及将代码移植到Linux时，在循环中为多个select()重用struct timeval而不重新初始化它时，都会导致问题。假设timeout在select()返回后是未定义的。

### 1.1.3返回值

如果成功，select()和pselect()将返回三个返回的描述符集中包含的文件描述符的数目(即在readfds、writefds和exceptfds中设置的总位数)，如果在发生任何有趣的事情之前超时过期，则该数目可能为零。如果出现错误，则返回-1，并设置errno来指示错误;文件描述符集未被修改，超时变为未定义。

| 错误码   | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| `EBADF`  | 在其中一个集合中给出了无效的文件描述符。(可能是一个已经关闭的文件描述符，或者已经发生错误的文件描述符。) |
| `EINTR`  | 一个信号被捕捉到了;看到信号(7)。                             |
| `EINVAL` | `nfds`为负值或超过`RLIMIT_NOFILE`资源限制，`timeout`中包含的值无效。 |
| `ENOMEM` | 无法为内部表分配内存。                                       |



### 1.1.4一些bug

Glibc 2.0提供了一个不带sigmask参数的pselect()版本。

从2.1版本开始，glibc提供了使用sigprocmask(2)和select()实现的pselect()的仿真。这个实现仍然容易受到竞态条件的影响，而pselect()的设计正是为了防止竞态条件的出现。现代版本的glibc在内核上使用(无竞争的)pselect()系统调用。在缺乏pselect()的系统上，可以使用自管道技巧实现可靠的(和更可移植的)信号捕获。在这种技术中，信号处理程序将一个字节写入管道，管道的另一端由主程序中的select()监视。(为了避免在写入可能已满的管道或从可能为空的管道中读取时可能出现阻塞，在读写管道时使用非阻塞I/O。)

在Linux下，select()可能会将套接字文件描述符报告为“准备好读取”，然而随后的读取阻塞。例如，当数据到达，**但检查时校验和错误并被丢弃时，可能会发生这种情况。可能在其他情况下，文件描述符被错误地报告为就绪。**因此，在不应该阻塞的套接字上使用`O_NONBLOCK`可能更安全。
在Linux上，如果调用被信号处理程序中断(即EINTR错误返回)，select()也会修改超时。这是POSIX.1不允许的。Linux的pselect()系统调用具有相同的行为，但是glibc包装器通过在内部将超时复制到一个局部变量并将该变量传递给系统调用来隐藏此行为。

### 1.1.4源码中的select

```c++
//system/core/libcutils/socket_network_client_unix.cpp
int socket_network_client_timeout(const char* host, int port, int type, int timeout,
                                  int* getaddrinfo_error) {
    ...
    int result = -1;
    for (struct addrinfo* addr = addrs; addr != NULL; addr = addr->ai_next) {
        // The Mac doesn't have SOCK_NONBLOCK.
        int s = socket(addr->ai_family, type, addr->ai_protocol);
        if (s == -1 || toggle_O_NONBLOCK(s) == -1) break;

        int rc = connect(s, addr->ai_addr, addr->ai_addrlen);
        if (rc == 0) {
            result = toggle_O_NONBLOCK(s);
            break;
        } else if (rc == -1 && errno != EINPROGRESS) {
            close(s);
            continue;
        }

        fd_set r_set;
        FD_ZERO(&r_set);
        FD_SET(s, &r_set);
        fd_set w_set = r_set;

        struct timeval ts;
        ts.tv_sec = timeout;
        ts.tv_usec = 0;
        if ((rc = select(s + 1, &r_set, &w_set, NULL, (timeout != 0) ? &ts : NULL)) == -1) {
            close(s);
            break;
        }
        if (rc == 0) {  // we had a timeout
            errno = ETIMEDOUT;
            close(s);
            break;
        }
		...
    }

    freeaddrinfo(addrs);
    return result;
}
```

其中，关于addrinfo定义

```c++
//bionic/libc/include/netdb.h
struct addrinfo {
	int	ai_flags;	/* AI_PASSIVE, AI_CANONNAME, AI_NUMERICHOST */
	int	ai_family;	/* PF_xxx */
	int	ai_socktype;	/* SOCK_xxx */
	int	ai_protocol;	/* 0 or IPPROTO_xxx for IPv4 and IPv6 */
	socklen_t ai_addrlen;	/* length of ai_addr */
	char	*ai_canonname;	/* canonical name for hostname */
	struct	sockaddr *ai_addr;	/* binary address */
	struct	addrinfo *ai_next;	/* next structure in linked list */
};
```

{{< image classes="fancybox center fig-100" src="/socket/io/io_7.png" thumbnail="/socket/io/io_7.png" title="">}}

## 1.2poll

Poll, ppoll:等待文件描述符上的某个事件

### 1.2.1函数原型

```c++
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <signal.h>
#include <poll.h>

int ppoll(struct pollfd *fds, nfds_t nfds,
          const struct timespec *tmo_p, const sigset_t *sigmask);
```



### 1.2.2描述

poll()执行的任务与select(2)类似:它等待一组文件描述符中的一个准备好执行I/O。

要监视的文件描述符的集合由fds参数指定，它是一个结构数组，格式如下:

```c++
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

调用者应该指定nfds中fds数组中的项数。

字段fd包含一个打开文件的文件描述符。如果该字段为负，则忽略相应的events字段，并将revents字段返回零。(这提供了一种在poll()调用中忽略文件描述符的简单方法:只需对fd字段取反即可。但要注意，该技术不能用来忽略文件描述符0。)

字段events是一个输入参数，它是一个位掩码，指定了应用程序对文件描述符fd感兴趣的事件。这个字段可以指定为0，在这种情况下，events中只能返回**POLLHUP**、**POLLERR**和**POLLNVAL**(见下文)。

revents字段是一个输出参数，由内核填充实际发生的事件。events返回的比特位可以是events中指定的任何一个，也可以是POLLERR、POLLHUP或POLLNVAL中的一个值。(这3位在events字段中没有意义，只要相应的条件为真，就会在revents字段中设置。)

如果对任何文件描述符都没有请求的事件(也没有错误)发生，则poll()将阻塞，直到发生其中一个事件。

timeout参数指定poll()应该阻塞等待文件描述符准备就绪的毫秒数。调用将阻塞，直到:

- 文件描述符准备就绪;
- 该调用被信号处理程序中断;或

- 超时到期。

请注意，超时时间间隔会向上取整到系统时钟粒度，而内核调度延迟意味着阻塞时间间隔可能会超出一小部分。在timeout中指定负数意味着无限超时。将超时时间指定为0会导致poll()立即返回，即使没有准备好文件描述符。

在events和revents中可以设置/返回的比特位定义在<poll.h>中:

events定义

| events值                        | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `POLLIN`                        | 有数据要读取。                                               |
| `POLLPRI`                       | 有紧急数据需要读取(例如，`TCP`套接字上的带外数据;在分组模式下的伪终端`master`在`slave`状态发生了变化)。 |
| `POLLOUT`                       | 现在可以写入了，但写入大于套接字或管道中的可用空间仍然会阻塞(除非设置了`O_NONBLOCK`)。 |
| `POLLRDHUP`(从Linux 2.6.17开始) | 流套接字的一端关闭了连接，或者关闭了写入连接的一半。为获得该定义，必须定义`_GNU_SOURCE`特性测试宏(在包含任何头文件之前)。 |
| `POLLERR`                       | 错误条件(仅在事件中返回;在事件中忽略)。                      |
| `POLLHUP`                       | 挂断电话(只有在紧急情况下才返回;在事件中忽略)。请注意，当从管道或流套接字等通道读取数据时，此事件仅表示一端关闭了其通道。只有在通道中所有未完成的数据都被消耗完之后，从通道中读取的后续数据才会返回0(文件结束)。 |
| `POLLNVAL`                      | 无效请求:`fd`未打开(仅在事件中返回;在事件中忽略)。           |

revents定义

在用`_XOPEN_SOURCE`定义的编译时，还可以得到以下信息，除了上述比特位之外，没有其他信息:

| events值     | 说明                                          |
| ------------ | --------------------------------------------- |
| `POLLRDNORM` | 相当于`POLLIN`。                              |
| `POLLRDBAND` | 可以读取优先频带数据(在`Linux`上通常未使用)。 |
| `POLLWRNORM` | 相当于`POLLOUT`。                             |
| `POLLWRBAND` | 优先级数据可以写入。                          |



### 1.2.3返回值

如果成功，则返回一个正数;这是具有非零revents字段的结构的数量(换句话说，那些报告了事件或错误的描述符)。值0表示调用超时，没有文件描述符准备好。发生错误时，返回-1，并适当地设置errno。

| 返回值   | 说明                                             |
| -------- | ------------------------------------------------ |
| `EFAULT` | 作为参数给出的数组不包含在调用程序的地址空间中。 |
| `EINTR`  | 在任何请求事件之前发生的信号;看到信号(7)。       |
| `EINVAL` | `nfds`值超过`RLIMIT_NOFILE`值。                  |
| `ENOMEM` | 没有空间分配文件描述符表。                       |



### 1.2.4源码中的例子

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

{{< image classes="fancybox center fig-100" src="/socketpair/sp_11.png" thumbnail="/socketpair/sp_11.png" title="">}}

## 1.3epoll

epoll - I/O事件通知设施

### 1.3.1函数原型

```c++
 #include <sys/epoll.h>
```



### 1.3.2描述

epoll API执行的任务与poll(2)类似:监视多个文件描述符，看看它们是否有I/O能力。epoll API既可以作为边缘触发接口，也可以作为电平触发接口，并且可以很好地扩展到大量监视的文件描述符。下列系统调用用于创建和管理epoll实例:

1. **epoll_create**(2)创建一个epoll实例，并返回指向该实例的文件描述符。(最近的epoll_create1(2)扩展了epoll_create(2)的功能。)
2. 然后通过**epoll_ctl**(2)注册对特定文件描述符感兴趣的文件。当前注册在epoll实例上的文件描述符集合有时称为epoll集。
3. **epoll_wait**(2)等待I/O事件，如果当前没有事件可用，则阻塞调用线程。

**水平触发**和**边缘触发**

epoll事件分布接口可以表现为边触发(ET)和水平触发(LT)。两种机制之间的区别可以描述如下。假设发生了以下情况:

1. 表示管道读端(rfd)的文件描述符注册在epoll实例上。
2. 管道写入器在管道的写入端写入2 kB的数据。
3. 完成了对epoll_wait(2)的调用，该调用将返回作为就绪文件描述符的rfd。

4. 管道读取器从rfd读取1 kB的数据。
5. 完成了对epoll_wait(2)的调用。

如果使用EPOLLET(边缘触发)标志将rfd文件描述符添加到epoll接口，那么在步骤5中完成的对epoll_wait(2)的调用可能会挂起，尽管在文件输入缓冲区中仍然存在可用数据;与此同时，远程端可能正在期待基于它已经发送的数据的响应。这样做的原因是，边缘触发模式只在被监控的文件描述符发生变化时才发送事件。因此，在步骤5中，调用者可能会等待一些已经存在于输入缓冲区中的数据。在上面的例子中，由于在2中完成了写入，而在3中使用了事件，因此将生成rfd上的事件。由于第4步中完成的读取操作不会消耗整个缓冲区数据，因此第5步中完成的对epoll_wait(2)的调用可能会无限阻塞。

使用EPOLLET标志的应用程序应该使用非阻塞的文件描述符，以避免阻塞读写阻塞正在处理多个文件描述符的任务。使用epoll作为边触发(EPOLLET)接口的建议方法如下:

- 具有非阻塞的文件描述符和

- 只有在**read(2)**或**write(2)**再次返回后才等待事件。

相比之下，当用作级别触发接口时(默认情况，未指定EPOLLET时)，epoll只是一个更快的轮询(2)，并且可以在使用后者的任何地方使用，因为它共享相同的语义。

因为即使使用边缘触发的epoll，也可以在接收到多个数据块时生成多个事件，因此调用者可以指定epollonshot标志，告诉epoll在接收到epoll_wait事件后禁用相关的文件描述符。在指定epolllonshot标志时，调用者负责使用epoll_ctl(2)和EPOLL_CTL_MOD重新武装文件描述符。

**与自动睡眠的交互**

如果系统通过/sys/power/autosleep处于自动休眠模式，并且发生了唤醒设备的事件，设备驱动程序将保持设备唤醒，直到该事件排队。为了让设备在事件处理完毕之前保持唤醒状态，需要使用epoll(7) EPOLLWAKEUP标志。

当在struct epoll_event的events字段中设置EPOLLWAKEUP标志时，系统将从事件进入队列的那一刻起保持唤醒状态，通过epoll_wait(2)调用来返回事件，直到后续的epoll_wait(2)调用。如果事件应该使系统在该时间之后保持唤醒，那么应该在第二次epoll_wait(2)调用之前执行一个独立的wake_lock。

**/ proc接口**
下列接口可用于限制epoll消耗的内核内存数量。

/proc/sys/fs/epoll/max_user_watches(从Linux 2.6.28开始)该参数指定了用户可以通过系统上的所有epoll实例注册的文件描述符的总数限制。限制是每个真实用户ID。每个注册的文件描述符在32位内核上大约消耗90字节，在64位内核上大约消耗160字节。当前，max_user_watches的默认值是可用低端内存的1/25(4%)，除以以字节为单位的注册开销。



### 1.3.3问答以及补充

> Q0:**用什么键来区分epoll集中注册的文件描述符?**
>
> A0:是文件描述符编号和打开文件描述符(也称为“打开文件句柄”，是内核对打开文件的内部表示)的组合。
>
> 
>
> Q1:**如果在一个epoll实例上注册两次相同的文件描述符会发生什么?**
>
> A1:你可能会得到EEXIST。但是，可以向同一个epoll实例添加一个重复的描述符(dup(2)， dup2(2)， fcntl(2) F_DUPFD)。如果重复的文件描述符使用不同的事件掩码注册，那么这是一种过滤事件的有用技术。
>
> 
>
> Q2:**两个epoll实例可以等待相同的文件描述符吗?如果是，事件是否报告给两个epoll文件描述符?**
>
> A2:事件会报告给双方。然而，要正确地做到这一点，可能需要仔细编程。
>
> 
>
> Q3:**epoll文件描述符本身是poll/epoll/可选择的吗?**
>
> A3:如果epoll文件描述符有事件等待，则将其标记为可读。
>
> 
>
> Q4:**如果试图将epoll文件描述符放入自己的文件描述符集中会发生什么?**
>
> A4:epoll_ctl(2)调用将失败(EINVAL)。但是，您可以在另一个epoll文件描述符集中添加epoll文件描述符。
>
> 
>
> Q5:**我可以通过UNIX域套接字向另一个进程发送epoll文件描述符吗?**
>
> A5:是的，但是这样做是没有意义的，因为接收进程在epoll集中没有文件描述符的副本。
>
> 
>
> Q6:**关闭一个文件描述符会导致它被自动从所有的epoll集合中移除吗?**
>
> A6:请注意以下几点。文件描述符是对打开文件描述的引用(参见open(2))。每当通过dup(2)、dup2(2)、fcntl(2) F_DUPFD或fork(2)复制一个描述符时，都会创建一个引用相同打开的文件描述符的新文件描述符。一个打开的文件描述符会一直存在，直到所有引用它的文件描述符都被关闭。只有在引用底层打开的文件描述符的所有文件描述符都关闭之后(如果描述符使用epoll_ctl(2) EPOLL_CTL_DEL显式删除，则在关闭之前)，文件描述符才会从epoll集中移除。这意味着，即使在epoll集中的文件描述符关闭后，如果引用同一基础文件描述符的其他文件描述符仍处于打开状态，则可能会报告该文件描述符的事件。
>
> 
>
> Q7:**如果在epoll_wait(2)调用之间发生了多个事件，它们是合并还是单独报告?**
>
> A7:它们会结合在一起。
>
> 
>
> Q8:**对文件描述符的操作是否会影响已经收集但尚未报告的事件?**
>
> A8:你可以对现有的文件描述符执行两种操作。在这种情况下，Remove是没有意义的。修改将重新读取可用的I/O。
>
> 
>
> Q9:**当使用EPOLLET标志(边缘触发行为)时，我是否需要持续读取/写入文件描述符直到EAGAIN ?**
>
> A9:从epoll_wait(2)接收到一个事件，应该表明这样的文件描述符已经为请求的I/O操作做好了准备。你必须认为它已经准备好了，直到下一次(非阻塞的)读/写操作再次发生。何时以及如何使用文件描述符完全取决于你。
>
> 对于数据包/令牌导向的文件(例如，数据报套接字，标准模式下的终端)，检测读/写I/O空间结束的唯一方法是继续读/写，直到EAGAIN。
>
> 对于面向流的文件(例如pipe、FIFO、流套接字)，也可以通过检查从目标文件描述符读取/写入的数据量来检测读/写I/O空间耗尽的情况。例如，如果调用read(2)时要求读取一定数量的数据，而read(2)返回的字节数更少，那么可以肯定已经耗尽了用于文件描述符的读I/O空间。使用write(2)写入时也是如此。(如果你不能保证被监控的文件描述符总是指向一个面向流的文件，请避免使用后一种技术。)

**可能的陷阱和避免它们的方法**
o饥饿(边缘触发)

如果有大量的I/O空间，则可能通过尝试清空它而导致其他文件无法处理
饥饿。(这个问题不是epoll特有的。)

解决方案是维护一个就绪列表，并在相关的数据结构中将文件描述符标记为就绪，从而允许应用程序记住哪些文件需要处理，但仍然在所有就绪文件之间轮询。这还支持忽略已经准备好的文件描述符的后续事件。

o如果使用事件缓存…

如果使用事件缓存或存储从epoll_wait(2)返回的所有文件描述符，则确保提供一种动态标记其闭包的方法(即，由前一个事件的处理引起的闭包)。假设你从epoll_wait(2)收到100个事件，在事件#47中，一个条件导致事件#13关闭。如果你移除该结构并关闭事件#13的文件描述符(2)，那么你的事件缓存可能仍然认为有事件在等待该文件描述符，从而导致混乱。

一种解决方案是在事件47处理期间调用epoll_ctl(EPOLL_CTL_DEL)删除文件描述符13并关闭(2)，然后将其关联的数据结构标记为已删除，并将其链接到一个清理列表。如果你在批处理中发现了另一个关于文件描述符13的事件，你会发现文件描述符之前已经被删除了，不会有任何混淆。

### 1.3.4源码中的例子

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

{{< image classes="fancybox center fig-100" src="/socketpair/sp_10.png" thumbnail="/socketpair/sp_10.png" title="">}}

# 2技术原理

## 2.1select

```c++
int listenfd = socket(PF_INET, SOCK_STREAM, 0);
bind(listenfd, (struct sockaddr*)&address, sizeof(address));
listen(listenfd, 5);
fd_set read_fds;

for(int i = 0; i < 5; i++)
{
    fd[i] = accept(listenfd, (struct sockaddr*)&client, &addr_len);
    if (fd[i] > max)
    {
        max = fd[i];
    }
}
while(1)
{
    FD_ZERO(&read_fds);
    for(int i = 0; i < 5; i++)
    {
        FD_SET(fd[i], &read_fds);
    }
    //第一个参数是最大文件描述符+1，在内核遍历中限制长度，减少无谓的遍历
    //第二个参数是读文件描述符集合
    //第三个参数是写文件描述符集合
    //第四个参数是异常事件文件描述符集合
    //第五个参数是超时时间，NULL是永不超时一直阻塞，直到有数据
    //服务端主要需要读文件描述符集合
    ret = select(max + 1, &read_fds, NULL, NULL, NULL);
    for (int i = 0; i < 5; i++)
    {
        if (FD_ISSET(fd[i], &read_fds))
        {
            ret = recv(fd[i], buff, sizeof(buff) - 1, 0);
        }
    }
}
```

{{< image classes="fancybox center fig-100" src="/socket/io/io_1.png" thumbnail="/socket/io/io_1.png" title="">}}

初始化上述过程之后，在内核会一直**阻塞**，等待客户端发送数据。

1）如果这个时候客户端把数据发送到服务端的网卡设备上

2）这个时候会发生软中断，然后会调用`DMA`拷贝技术，将数据拷贝到内核环形缓冲队列中

3）然后根据文件描述符信息，把数据拷贝到对应的socket的数据接收队列里面去。哪个socket文件描述符对应的数据接收队列有数据到达，那么标志这个文件描述符已就绪，它们就会标记成已就绪的状态。

4）select返回。返回不仅仅是拷贝已就绪，是把所有的bitmap的文件描述符拷贝到用户态，同时告诉用户态有几个文件描述符已经就绪，只通知个数不通知具体哪个文件描述符。用户态通过`ISSET`判断是否就绪，通过`recv`去解除阻塞。

{{< image classes="fancybox center fig-100" src="/socket/io/io_2.png" thumbnail="/socket/io/io_2.png" title="">}}

## 2.2poll

poll模型是select的改进版

```c++
struct pollfd {
    int fd; 			/*文件描述符*/
    short events; 		/*注册的事件*/
    short revents; 		/*实际发生的事件，由内核填充*/
};
```

举例说明

```c++
int sscfd = socket(PF_INET, SOCK_STREAM, 0);
bind(sscfd, (struct sockaddr*)&address, sizeof(address));
listen(sscfd, 4);
struct pollfd fds[4];
for(int i = 0; i < 4; i++)
{
    fds[i].fd = accept(sscfd, (struct sockaddr*)&cliAddr, &addrLen);
    fd[i].events = POLLIN;
}
sleep(1);
while(1)
{
    //第一个参数，fds就是传入的文件描述符的结构体数组
    //第二个参数，就是这个结构体数组最大长度
    //第三个参数，阻塞等待时间
    ret = poll(fds, 4, 4000);
    for(int i = 0; i < 4; i++)
    {
        if(fds[i].revents & POLLIN)
        {
            fds[i].revents = 0;
            ret = recv(fds[i].fd, buff, sizeof(buff) - 1, 0);
		}
    }
}
```

poll方法和select是一样的，一次性将一批文件描述符发送到内核态。需要监听和关注事件对应的文件描述符所执行的进程，并放到等待队列里面去等待。

{{< image classes="fancybox center fig-100" src="/socket/io/io_3.png" thumbnail="/socket/io/io_3.png" title="">}}

初始化上述过程之后，在内核会一直**阻塞**，等待客户端发送数据。

1）如果这个时候客户端把数据发送到服务端的网卡设备上

2）这个时候会发生软中断，然后会调用`DMA`拷贝技术，将数据拷贝到内核环形缓冲队列中

3）然后根据文件描述符信息，把数据拷贝到对应的socket的数据接收队列里面去。哪个socket文件描述符对应的数据接收队列有数据到达，就把对应的revents字段对应置为POLLIN，表示对应的文件描述符已经就绪。

4）poll返回。返回不仅仅是拷贝已就绪，是把所有的文件描述符的结构体数组拷贝到用户态，同时告诉用户态有几个文件描述符已经就绪，只通知个数不通知具体哪个文件描述符。用户态通过revents比较判断POLLIN是否就绪，然后重置revents数据为0，通过`recv`去解除阻塞。

{{< image classes="fancybox center fig-100" src="/socket/io/io_4.png" thumbnail="/socket/io/io_4.png" title="">}}

## 2.3epoll

epoll服务端代码

```c++
int sscfd = socket(PF_INET, SOCK_STREAM, 0);
bind(sscfd, (struct sockaddr*)&address, sizeof(address));
listen(sscfd, 4);
struct epoll_event events[5];
//内核会去创建eventpoll结构体
//eventpoll中有三个重要字段
//rdyList已就绪的文件描述双链表
//rbr就是一颗红黑树，用红黑树去管理用户进程放进来的所有socket连接
//wq为等待队列，当某个进程他需要关注的事件未就绪的时候，就会把当前进程的描述符和回调函数放到进程等待队列中去
int epoll_fd = epoll_create(5);
for (int i = 0; i < 5; i++)
{
    struct epoll_event event;
    event.data.fd = accept(sscfd, (struct sockaddr*)&cliAddr, &addrLen);
    //将对应事件注册到eventpoll中
    event.events = EPOLLIN;
    //每个socket连接在epoll_ctl后会分配一个epitem
    //
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, event.data.fd, &event);
}

while (1)
{
    //检查内核中的双链表是否有就绪的事件，存在事件立刻返回
    int ret = epoll_wait(epoll_fd, events, 5, 2000);
    for (int i = 0; i < ret; i++)
    {
        if (events[i].events & EPOLLIN)
        {
            recv(sockfd, buf, BUFFER_SIZE - 1, 0);
        }
    }
}
```

其中的结构体定义

```c++
struct union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t events;
    epoll_data_t data;
};
```

> 对应内核的结构体epitem
>
> ```c++
> struct epitem {
>     //每一个socket连接对应的红黑树节点
>     struct rb_node rbn;
>     //对应的文件描述符信息
>     struct epoll_filefd ffd;
>     //socket连接对应的eventpoll数据
>     struct eventpoll *ep;
>     //进程等待队列，对应ep中的wq等待队列
>     struct list_head pwqlist;
> };
> ```

每当创建一个客户端，和服务端socket连接之后，就会把这些连接添加到一个红黑树里面。

{{< image classes="fancybox center fig-100" src="/socket/io/io_5.png" thumbnail="/socket/io/io_5.png" title="">}}

1）当客户端把数据发送到了socket的数据接收队列里面去，这个时候就会将已就绪的读事件把它们插入到已就绪的链表里（这里通过回调函数而不是遍历）。

2）内核检查是否有阻塞的进程存在。如果这个阻塞的进程恰好是执行就绪链表里面对应的描述符的进程，那么就把它唤醒。

3）唤醒后，用户态进程A进入到CPU的运行，将对应的文件描述符返回给用户态，通过用户态遍历通过events比较判断EPOLLIN是否就绪，通过`recv`去解除阻塞。

{{< image classes="fancybox center fig-100" src="/socket/io/io_6.png" thumbnail="/socket/io/io_6.png" title="">}}



# 3总结

## 3.1关于三种IO类型的原理和不足总结

### 3.1.1Select IO多路复用执行原理

1）将当前进程的所有文件描述符，一次性的从用户态拷贝到内核态

2）在内核态中快速的无差别遍历每个fd，判断是否有数据达到

3）将所有fd状态，从内核态拷贝到用户态，并返回已就绪的fd的个数

4）在用户态遍历判断具体哪个fd已就绪，然后进行相应的事件处理

### 3.1.2Select IO多路复用的限制和不足

1）文件描述符表位bitmap，且有长度为1024的限制

2）fdset无法做到重用，每次循环必须重新创建

3）频繁的用户态和内核态拷贝，性能开销较大

4）需要对文件描述符表进行遍历，o(n)的轮询时间复杂度

### 3.1.3poll IO多路复用执行原理

1）将当前进程的所有文件描述符，一次性的从用户态拷贝到内核态

2）在内核中快速的无差别遍历每个fd，判断是否有数据达到

3）将所有fd状态，从内核态拷贝到用户态，并返回已就绪fd的个数

4）在用户态遍历判断具体哪个fd已就绪，然后进行相应的事件处理

### 3.1.4poll IO多路复用的限制和不足

1）poll模型采用的pollfd结构数组，解决了select的1024个文件描述符的限制

2）但仍存在频繁的用户态和内核态拷贝，性能开销较大

3）需要对文件描述符表进行遍历，o(n)的轮询时间复杂度

### 3.1.5epoll

1）在epoll_ctl函数中，为每个文件描述符都指定了回调函数，基于回调函数把就绪事件放到就绪队列中，因此把时间复杂度从o(n)降到了o(1)

2）只需要在epoll_ctl时传递一次文件描述符，epoll_wait不需要再次传递文件描述符

3）epoll基于红黑树+双链表存储事件，没有最大连接数的限制

4）注意epoll没有使用mmap零拷贝技术

## 3.2关于epoll机制额外补充

关于epoll_ctl 并不是所有fd都支持，目前支持管道，FIFO，套接字（本文主要内容），POSIX消息队列，终端，设备等，但是就是不支持普通文件或目录的fd。

```erlang
//有很多fd都支持，这里列举几个常用的：
./fs/notify/fanotify/fanotify_user.c-520- .poll = fanotify_poll,
./fs/notify/inotify/inotify_user.c-322- .poll = inotify_poll,
./fs/proc/inode.c-404- .poll = proc_reg_poll,
./fs/signalfd.c-258- .poll = signalfd_poll,
./fs/timerfd.c-370- .poll = timerfd_poll,
./fs/userfaultfd.c-1928- .poll = userfaultfd_poll,
./ipc/mqueue.c-1632- .poll = mqueue_poll_file,
./net/socket.c-154- .poll = sock_poll,
```

Epoll可以使用一次等待监听多个描述符的可读/可写状态。等待返回时携带了可读的描述符或自定义的数据，使用者可以据此读取所需的数据后可以再次进入等待。因此不需要为每个描述符创建独立的线程进行阻塞读取，避免了资源浪费的同时又可以获得较快的响应速度。