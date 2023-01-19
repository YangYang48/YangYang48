---
title: "Linux clone"
date: 2022-12-21
thumbnailImagePosition: left
thumbnailImage: linux/func/clone/clone_thumb.jpg
coverImage: linux/func/clone/clone_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- clone
- 2022
- December
tags:
- Android
- Linux
- 基础
- 源码
showSocial: false
---

学习Linux中的clone函数。

<!--more-->
# 0简介

## 1NAME
​       clone, __clone2 - create a child process

用于创建子进程和子线程

## 2SYNOPSIS

```c
/*glibc包装器函数的原型 */
#include <sched.h>
int clone(int (*fn)(void *), void *child_stack,
          int flags, void *arg, ...
          /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );

/* 原始系统调用的原型 */
long clone(unsigned long flags, void *child_stack,
           void *ptid, void *ctid,
           struct pt_regs *regs);
```

## 3DESCRIPTION

`Clone()`以类似于`fork(2)`的方式创建一个新进程。

​       本页描述了`glibc clone()`包装器函数和它所基于的底层系统调用。正文介绍了包装器的功能;原始系统调用的区别将在本页的末尾描述。与`fork(2)`不同，`clone()`允许子进程与调用进程共享部分执行上下文，例如内存空间、文件描述符表和信号处理程序表。(请注意，在本手册中，“调用进程”通常对应于“父进程”。但是请参阅下面对`CLONE_PARENT`的描述。)

​       `clone()`的一个用途是实现线程:在一个程序中并发运行在共享内存空间中的多个控制线程。

​       当使用`clone()`创建子进程时，它执行函数`fn(arg)`。(这与`fork(2)`不同，后者从`fork(2)`调用的点开始在子进程中继续执行。)`fn`参数是一个指向子进程在执行开始时调用的函数的指针。参数`arg`被传递给`fn`函数。

​       当`fn(arg)`函数应用程序返回时，子进程终止。`fn`返回的整数是子进程的退出码。子进程也可以通过调用`exit(2)`或在接收到致命信号后显式终止。

​       `child_stack`参数指定了子进程使用的堆栈的位置。由于子进程和调用进程可能共享内存，因此子进程不可能与调用进程在同一个堆栈中执行。因此，调用进程必须为子进程设置内存空间，并将指向该空间的指针传递给`clone()`。在所有运行Linux的处理器上(除了HP PA处理器)，堆栈向下增长，因此`child_stack`通常指向为子堆栈设置的内存空间的最顶层地址。标志的低字节包含子进程死亡时发送给父进程的终止信号的数目。**如果这个信号不是被指定为`SIGCHLD`，那么父进程在使用`wait(2)`等待子进程时必须指定`__WAL`L或`__WCLONE`选项。如果未指定信号，则在子进程终止时不向父进程发出信号。**

为了指定调用进程和子进程之间共享的内容，标志也可以用零个或多个以下常量进行位序运算:

```c
CLONE_CHILD_CLEARTID (since Linux 2.5.49)
              当子线程退出时，在子内存中的ctid位置擦除子线程ID，并在该地址的futex上进行唤醒。所涉及的地址可以通过set_tid_address(2)系统调用改变。线程库使用它。

CLONE_CHILD_SETTID (since Linux 2.5.49)
              将子线程ID存储在子内存中的ctid位置。

CLONE_FILES (since Linux 2.0)
              如果设置了CLONE_FILES，则调用进程和子进程共享相同的文件描述符表。由调用进程或子进程创建的任何文件描述符在其他进程中也有效。类似地，如果其中一个进程关闭了一个文件描述符，或者改变了它的相关标志(使用fcntl(2) F_SETFD操作)，另一个进程也会受到影响。如果共享文件描述符表的进程调用execve(2)，则其文件描述符表将被复制(非共享)。
              如果没有设置CLONE_FILES，子进程将继承在clone()时调用进程中打开的所有文件描述符的副本。(子进程中复制的文件描述符与调用进程中对应的文件描述符引用相同的打开文件描述符(参见open(2))。)由调用进程或子进程执行的打开或关闭文件描述符或更改文件描述符标志的后续操作不会影响其他进程。

CLONE_IO (since Linux 2.6.25)
              如果设置了CLONE_IO，则新进程与调用进程共享I/O上下文。如果没有设置这个标志，那么(就像fork(2)一样)新进程有自己的I/O上下文。I/O上下文是磁盘调度器的I/O范围(即I/O调度器用来模拟进程I/O的调度)。如果进程共享相同的I/O上下文，则I/O调度程序将它们视为一个进程。因此，它们可以共享磁盘时间。对于某些I/O调度器，如果两个进程共享一个I/O上下文，则允许它们交叉访问磁盘。如果几个线程代表同一个进程(例如aio_read(3))执行I/O，它们应该使用CLONE_IO来获得更好的I/O性能。如果内核没有配置CONFIG_BLOCK选项，这个标志是一个no-op。

CLONE_NEWIPC (since Linux 2.6.19)
              如果设置了CLONE_NEWIPC，则在新的IPC命名空间中创建进程。如果没有设置这个标志，那么(就像fork(2)一样)，进程被创建在与调用进程相同的IPC命名空间中。此标志用于容器的实现。IPC命名空间提供了System V IPC对象(参见svipc(7))和(自Linux 2.6.30起)POSIX消息队列(参见mq_overview(7))的隔离视图。这些IPC机制的共同特征是IPC对象由文件系统路径名以外的机制标识。在IPC名称空间中创建的对象对该名称空间的所有其他成员进程可见，但对其他IPC名称空间中的进程不可见。当一个IPC命名空间被销毁时(即，当作为该命名空间成员的最后一个进程终止时)，该命名空间中的所有IPC对象都将自动销毁。只有特权进程(CAP_SYS_ADMIN)可以使用CLONE_NEWIPC。此标志不能与CLONE_SYSVSEM一起指定。有关IPC名称空间的更多信息，请参见namespaces(7)。

CLONE_NEWNET (since Linux 2.6.24)
              (该标志的实现仅在内核版本2.6.29时完成。)
如果设置了CLONE_NEWNET，则在新的网络名称空间中创建进程。如果没有设置这个标志，那么(就像fork(2)一样)进程被创建在与调用进程相同的网络命名空间中。此标志用于容器的实现。
              网络命名空间提供了网络堆栈(网络设备接口、IPv4和IPv6协议栈、IP路由表、防火墙规则、/proc/net和/sys/class/net目录树、套接字等)的隔离视图。一个物理网络设备只能存在于一个网络名称空间中。虚拟网络设备(“veth”)对提供了一个类似管道的抽象，可用于在网络名称空间之间创建隧道，并可用于创建到另一个名称空间中的物理网络设备的桥接。当网络命名空间被释放(即命名空间中的最后一个进程终止)时，其物理网络设备将被移回初始网络命名空间(而不是该进程的父进程)。有关网络名称空间的更多信息，请参见namespaces(7)。
只有特权进程(CAP_SYS_ADMIN)可以使用CLONE_NEWNET。

CLONE_NEWNS (since Linux 2.4.19)
              如果设置了CLONE_NEWNS，克隆的子进程将在一个新的挂载名称空间中启动，并使用父进程名称空间的副本进行初始化。如果没有设置CLONE_NEWNS，则子进程与父进程位于相同的挂载名称空间中。只有特权进程(CAP_SYS_ADMIN)可以使用CLONE_NEWNS。不允许在同一个clone()调用中同时指定CLONE_NEWNS和CLONE_FS。

CLONE_NEWPID (since Linux 2.6.24)
              如果设置了CLONE_NEWPID，则在新的PID名称空间中创建进程。如果没有设置这个标志，那么(和fork(2)一样)进程被创建在与调用进程相同的PID命名空间中。此标志用于容器的实现。只有特权进程(CAP_SYS_ADMIN)可以使用CLONE_NEWPID。此标志不能与CLONE_THREAD或CLONE_PARENT一起指定。

CLONE_NEWUSER
             (这个标志第一次在Linux 2.6.23中对clone()有意义，当前的clone()语义在Linux 3.5中合并，最后使用户名称空间完全可用的部分在Linux 3.8中合并。)如果设置了CLONE_NEWUSER，则在新的用户名称空间中创建进程。如果没有设置这个标志，那么(与fork(2)一样)进程将被创建在与调用进程相同的用户命名空间中。在Linux 3.8之前，使用CLONE_NEWUSER要求调用者具有三个功能:CAP_SYS_ADMIN、CAP_SETUID和CAP_SETGID。从Linux 3.8开始，创建用户名称空间不需要任何特权。

此标志不能与CLONE_THREAD或CLONE_PARENT一起指定。出于安全原因，不能同时指定CLONE_NEWUSER和CLONE_FS。

CLONE_NEWUTS (since Linux 2.6.19)
             如果设置了CLONE_NEWUTS，则在新的UTS名称空间中创建进程，其标识符通过复制调用进程的UTS名称空间中的标识符来初始化。如果未设置此标志，则(与fork(2)一样)在与调用进程相同的UTS名称空间中创建进程。此标志用于容器的实现。UTS名称空间是uname(2)返回的标识符集;其中，域名可以通过setdomainname(2)修改，主机名可以通过sethostname(2)修改。对UTS名称空间中的标识符所做的更改对同一名称空间中的所有其他进程可见，但对其他UTS名称空间中的进程不可见。只有特权进程(CAP_SYS_ADMIN)可以使用CLONE_NEWUTS。

CLONE_PARENT (since Linux 2.3.12)
              如果设置了CLONE_PARENT，则新子进程的父进程(由getppid(2)返回)将与调用进程的父进程相同。如果没有设置CLONE_PARENT，那么(与fork(2)一样)子进程的父进程就是调用进程。请注意，它是由getppid(2)返回的父进程，当子进程终止时发出信号，因此如果设置了CLONE_PARENT，那么将发出信号的是调用进程的父进程，而不是调用进程本身。
                  
CLONE_PARENT_SETTID (since Linux 2.5.49)
              将子线程ID存储在父线程内存中的ptid位置。(在Linux 2.5.32-2.5.48中，有一个标记CLONE_SETTID可以做到这一点。)

CLONE_PID (obsolete)(废弃)
              如果设置了CLONE_PID，则创建子进程时使用与调用进程相同的进程ID。这对入侵系统很有帮助，但在其他方面用处不大。从2.3.21开始，这个标志只能由系统引导进程(PID 0)指定。它在Linux 2.5.16中消失了。从那时起，内核就会无声地忽略它而不会出错。

CLONE_PTRACE (since Linux 2.2)
              如果指定了CLONE_PTRACE，并且正在跟踪调用进程，那么也要跟踪子进程(参见ptrace(2))。

CLONE_SETTLS (since Linux 2.5.32)
              newtls参数是新的TLS(线程本地存储)描述符。(见set_thread_area(2))。

CLONE_SIGHAND (since Linux 2.0)
             如果设置了CLONE_SIGHAND，则调用进程和子进程共享同一个信号处理程序表。如果调用进程或子进程调用sigaction(2)来更改与某个信号相关的行为，则该行为在其他进程中也会被更改。但是，调用进程和子进程仍然具有不同的信号掩码和挂起信号集。因此，其中一个进程可以使用sigprocmask(2)阻塞或解除阻塞某些信号，而不影响另一个进程。如果没有设置CLONE_SIGHAND，子进程将在调用clone()时继承调用进程的信号处理程序的副本。其中一个进程稍后执行的对sigaction(2)的调用对另一个进程没有影响。从Linux 2.6.0-test6开始，如果指定了CLONE_SIGHAND，标志也必须包括CLONE_VM。

CLONE_STOPPED (since Linux 2.6.0-test2)
              如果设置了CLONE_STOPPED，那么子进程最初会被停止(就像它被发送了SIGSTOP信号一样)，并且必须通过向它发送SIGCONT信号来恢复。这个标志从Linux 2.6.25以后就被弃用了，在Linux 2.6.38中完全被移除。从那时起，内核就会无声地忽略它而不会出错。

CLONE_SYSVSEM (since Linux 2.5.10)
              如果设置了CLONE_SYSVSEM，那么子进程和调用进程共享一个System V信号量调整(semadj)值列表(参见semop(2))。在这种情况下，共享列表在所有共享列表的进程中累积semadj值，只有当最后一个共享列表的进程终止(或使用unshare(2)停止共享列表)时才执行信号量调整。如果没有设置这个标志，那么子进程有一个单独的semadj列表，初始值为空。

CLONE_THREAD (since Linux 2.4.0-test8)
              如果设置了CLONE_THREAD，则将子进程放在与调用进程相同的线程组中。为了使关于CLONE_THREAD的其余讨论更具可读性，使用术语“线程”来指代线程组中的进程。线程组是Linux 2.4中添加的一个特性，用于支持POSIX线程概念，即一组线程共享一个PID。在内部，这个共享PID是线程组的所谓线程组标识符(TGID)。从Linux 2.4开始，对getpid(2)的调用返回调用方的TGID。
              一个组中的线程可以通过它们(系统范围内的)惟一线程id (TID)来区分。一个新线程的TID作为返回给clone()调用者的函数结果是可用的，并且线程可以使用gettid(2)获得自己的TID。当调用clone()而不指定CLONE_THREAD时，生成的线程被放置在一个新的线程组中，该线程组的TGID与线程的TID相同。这个线程是新线程组的领导者。使用CLONE_THREAD创建的新线程具有与clone()调用方相同的父进程(例如，类似CLONE_PARENT)，因此调用getppid(2)将为线程组中的所有线程返回相同的值。当一个CLONE_THREAD线程终止时，使用clone()创建它的线程不会被发送一个SIGCHLD(或其他终止)信号;使用wait(2)也不能获得这样一个线程的状态。(线程被称为分离。)
              当线程组中的所有线程终止后，线程组的父进程将被发送一个SIGCHLD(或其他终止)信号。如果线程组中的任何线程执行了execve(2)，则除线程组领导之外的所有线程都将终止，新程序将在线程组领导中执行。如果线程组中的一个线程使用fork(2)创建了一个子线程，那么该组中的任何线程都可以等待(2)这个子线程。从Linux 2.5.35开始，如果指定了CLONE_THREAD，标志也必须包括CLONE_SIGHAND(注意，从Linux 2.6.0-test6开始，CLONE_SIGHAND还要求包括CLONE_VM)。信号可以使用kill(2)发送到整个线程组(即TGID)，也可以使用tgkill(2)发送到特定线程(即TID)。
			  信号配置和操作是进程范围的:如果一个未处理的信号被传递到一个线程，那么它将影响(终止、停止、继续、被忽略)线程组的所有成员。每个线程都有自己的信号掩码，由sigprocmask(2)设置，但信号也可以挂起:对于整个进程(即，可传递给线程组的任何成员)，当与kill(2)一起发送时;或者对于单独的线程，当与tgkill(2)一起发送时。调用sigpending(2)将返回一个信号集，该信号集是整个进程的挂起信号和调用线程的挂起信号的并集。
			  如果kill(2)被用于向线程组发送信号，并且线程组已经为该信号安装了处理程序，那么该处理程序将在未阻塞信号的线程组中任意选择的一个成员中调用。如果一个组中的多个线程正在使用sigwaitinfo(2)等待接受同一个信号，内核将任意选择其中一个线程来接收使用kill(2)发送的信号。

CLONE_UNTRACED (since Linux 2.5.46)
              如果指定了clone_untrace，则跟踪进程不能对该子进程强制执行CLONE_PTRACE。

CLONE_VFORK (since Linux 2.2)
              如果设置了CLONE_VFORK，则调用进程的执行将被挂起，直到子进程通过调用execve(2)或_exit(2)(与vfork(2)一样)释放其虚拟内存资源。如果没有设置CLONE_VFORK，那么调用进程和子进程在调用之后都是可调度的，应用程序不应该依赖于以任何特定顺序发生的执行。

CLONE_VM (since Linux 2.0)
              如果设置了CLONE_VM，则调用进程和子进程运行在相同的内存空间中。特别是，调用进程或子进程执行的内存写入在其他进程中也是可见的。此外，由子进程或调用进程使用mmap(2)或munmap(2)执行的任何内存映射或取消映射也会影响其他进程。
              如果没有设置CLONE_VM，子进程在clone()时运行在调用进程的内存空间的单独副本中。由一个进程执行的内存写入或文件映射/取消映射不会影响另一个进程，就像fork(2)一样。
```

## 4RETURN VALUE

```text
成功时，在调用者的执行线程中返回子进程的线程ID。失败时，在调用者的上下文中返回-1，不创建任何子进程，并适当地设置errno。
```

# 1example

更改子进程的主机名，但是不影响父进程的主机名

```c
#define _GNU_SOURCE
#include <sys/wait.h>
#include <sys/utsname.h>
#include <sched.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                               } while (0)

static int              /* Start function for cloned child */
childFunc(void *arg)
{
    struct utsname uts;
    /* 在子的UTS名称空间中更改主机名 */
    if (sethostname(arg, strlen(arg)) == -1)
        errExit("sethostname");

    /* 检索和显示主机名 */
    if (uname(&uts) == -1)
        errExit("uname");
    printf("uts.nodename in child:  %s\n", uts.nodename);
    //通过休眠使命名空间打开一段时间。这允许进行一些实验——例如，另一个进程可能加入名称空间。
    sleep(200);

    return 0;           /* Child terminates now */
}
//这个空间至少是4k的整数倍
#define STACK_SIZE (1024 * 1024)    /* Stack size for cloned child */

int main(int argc, char *argv[])
{
    char *stack;                    /* Start of stack buffer */
    char *stackTop;                 /* End of stack buffer */
    pid_t pid;
    struct utsname uts;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <child-hostname>\n", argv[0]);
        exit(EXIT_SUCCESS);
    }

    /* 为子进程分配堆栈 */
    stack = malloc(STACK_SIZE);
    if (stack == NULL)
        errExit("malloc");
    //假设堆栈向下增长
    stackTop = stack + STACK_SIZE;

    /* 创建具有自己的UTS名称空间的子程序;子程序在childFunc()中开始执行 */
    pid = clone(childFunc, stackTop, CLONE_NEWUTS | SIGCHLD, argv[1]);
    if (pid == -1)
        errExit("clone");
    printf("clone() returned %ld\n", (long) pid);

    /* 父进程走这里 */
    /* 给子节点修改主机名的时间 */
    sleep(1);           
    /* 在父节点的UTS命名空间中显示主机名。这将不同于子节点的UTS名称空间中的主机名。 */
    if (uname(&uts) == -1)
        errExit("uname");
    printf("uts.nodename in parent: %s\n", uts.nodename);

    if (waitpid(pid, NULL, 0) == -1)    /* Wait for child */
        errExit("waitpid");
    printf("child has terminated\n");

    exit(EXIT_SUCCESS);
}
```

可以看到子父进程中，通过参数`CLONE_NEWUTS`，可以设置不同的主机名

{{< image classes="fancybox center fig-100" src="/linux/func/clone/clone_1.png" thumbnail="/linux/func/clone/clone_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/func/clone/clone_2.png" thumbnail="/linux/func/clone/clone_2.png" title="">}}

# 2源码分析

以libc的Native crash为例

## 2.1clone debuggerd_dispatch_pseudothread

```c++
static void debuggerd_signal_handler(int signal_number, siginfo_t* info, void* context) {
  //CLONE_THREAD,说明clone不是进程是线程 
  //这里还是同一个进程，设置了不同的线程，其中CLONE_CHILD_SETTID设置之后，
  //进入debuggerd_dispatch_pseudothread线程之后，最后一个元素为子线程tid被保存起来  
  pid_t child_pid =
    clone(debuggerd_dispatch_pseudothread, pseudothread_stack,
          CLONE_THREAD | CLONE_SIGHAND | CLONE_VM | CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID,
          &thread_info, nullptr, nullptr, &thread_info.pseudothread_tid);
  if (child_pid == -1) {
    fatal_errno("failed to spawn debuggerd dispatch thread");
  }

  //等待子线程启动，这里的futex_wait功能实际上是后者的变量与前者指向元素相同，那就阻塞
  futex_wait(&thread_info.pseudothread_tid, -1);

  // 等待子线程结束，子线程启动后，thread_info.pseudothread_tid变量为子线程变量，直到退出恢复
  futex_wait(&thread_info.pseudothread_tid, child_pid);
}

void debuggerd_init(debuggerd_callbacks_t* callbacks) {
  if (callbacks) {
    g_callbacks = *callbacks;
  }

  size_t thread_stack_pages = 8;
  void* thread_stack_allocation = mmap(nullptr, PAGE_SIZE * (thread_stack_pages + 2), PROT_NONE,
                                       MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  if (thread_stack_allocation == MAP_FAILED) {
    fatal_errno("failed to allocate debuggerd thread stack");
  }

  char* stack = static_cast<char*>(thread_stack_allocation) + PAGE_SIZE;
  if (mprotect(stack, PAGE_SIZE * thread_stack_pages, PROT_READ | PROT_WRITE) != 0) {
    fatal_errno("failed to mprotect debuggerd thread stack");
  }

  // Stack grows negatively, set it to the last byte in the page...
  stack = (stack + thread_stack_pages * PAGE_SIZE - 1);
  // and align it.
  stack -= 15;
  //分配的栈顶在这里
  pseudothread_stack = stack;

  struct sigaction action;
  memset(&action, 0, sizeof(action));
  sigfillset(&action.sa_mask);
  action.sa_sigaction = debuggerd_signal_handler;
  action.sa_flags = SA_RESTART | SA_SIGINFO;

  // Use the alternate signal stack if available so we can catch stack overflows.
  action.sa_flags |= SA_ONSTACK;
  debuggerd_register_handlers(&action);
}
```

{{< image classes="fancybox center fig-100" src="/linux/func/clone/clone_3.png" thumbnail="/linux/func/clone/clone_3.png" title="">}}

## 2.2clone _fork

```c++
static pid_t __fork() {
  return clone(nullptr, nullptr, 0, nullptr);
}

static int debuggerd_dispatch_pseudothread(void* arg) {
  pid_t crash_dump_pid = __fork();
  if (crash_dump_pid == -1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc",
                          "failed to fork in debuggerd signal handler: %s", strerror(errno));
  } else if (crash_dump_pid == 0) {
	//子进程
  }
  //父进程
}
```



# 源码下载

点击[这里](https://github.com/YangYang48/project/tree/master/clone)

# 参考

[[1] charlieroro, Linux Clone函数, 2021.](https://cloud.tencent.com/developer/article/1776329?from=15425)

[[2] nedwons, prctl()函数详解, 2018.](https://blog.csdn.net/hunter___/article/details/83063131)

[[3] 沧海一粟之, Linux Futex浅析, 2016.](https://blog.csdn.net/mitushutong11/article/details/51336136/)