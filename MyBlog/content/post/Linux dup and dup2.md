---
title: "Linux dup and dup2"
date: 2022-12-20
thumbnailImagePosition: left
thumbnailImage: linux/func/dup_dup2/dup_dup2_thumb.jpg
coverImage: linux/func/dup_dup2/dup_dup2_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- dup
- dup2
- 2022
- December
tags:
- Android
- Linux
- pipe
- fork
- NativeCrash
- 基础
- 源码
showSocial: false
---

学习Linux中的dup/dup2函数。

<!--more-->
# 0简介

## 1NAME
`dup`, `dup2`, `dup3` - duplicate a file descriptor

这类函数是用于复制文件描述符的。

## 2SYNOPSIS

```c
#include <unistd.h>

int dup(int oldfd);
int dup2(int oldfd, int newfd);
```

## 3DESCRIPTION

```text
dup()
系统调用创建文件描述符oldfd的副本，为新的描述符使用编号最低的未使用的描述符。
成功返回后，新旧文件描述符可以互换使用。它们引用相同的打开文件描述
(参见open(2))，从而共享文件偏移量和文件状态标志;例如，如果在一个文件上使用lseek(2)修改文件偏移量
在描述符中，另一个的偏移量也改变了。
这两个描述符不共享文件描述符标志(close-on-exec标志)。关闭执行标志(FD_CLOEXEC;它看到fcntl (2))
为重复描述符关闭。

dup2()
dup2()系统调用执行与dup()相同的任务，但它不是使用编号最低的未使用文件描述符，而是使用
在newfd中指定的描述符编号。如果描述符newfd以前是打开的，那么在重用它之前，它将被无声地关闭。
关闭和重用文件描述符newfd的步骤是原子地执行的。这很重要，因为试图实现
使用close(2)和dup()的等效功能将受到竞争条件的影响，因此newfd可能在两者之间被重用
步骤。发生这种重用的原因可能是主程序被分配文件描述符的信号处理程序中断
因为并行线程分配一个文件描述符。

注意以下几点:
*如果oldfd不是有效的文件描述符，则调用失败，newfd不会关闭。
*如果oldfd是一个有效的文件描述符，并且newfd与oldfd具有相同的值，那么dup2()不做任何事情，并返回newfd。
```

具体再解释下，由`dup`返回的新文件描述符一定是当前可用文件描述符中的最小数值。对于`dup2`，可以用`fd2`参数指定新描述符的值。如果`fd2`已经打开，则先将其关闭。如若`fd`等于`fd2`，则`dup2`返回`fd2`，而不关闭它。否则，`fd2`的`FD_CLOEXEC`文件描述符标志就被清除，这样`fd2`在进程调用`exec`时是打开状态。

## 4RETURN VALUE

```text
如果成功，这些系统调用将返回新的描述符。如果出现错误，则返回-1，并适当地设置errno。

错误
EBADF oldfd不是一个打开的文件描述符。
EBADF newfd超出了文件描述符的允许范围(请参阅getrlimit(2)中关于RLIMIT_NOFILE的讨论)。
EBUSY(仅限Linux)这可能由dup2()或dup3()在open(2)和dup()的竞态条件下返回。
EINTR dup2()或dup3()调用被信号中断;看到信号(7)。
EINVAL (dup3())标志包含无效值。
EINVAL (dup3()) oldfd等于newfd。
EMFILE每个进程打开的文件描述符数量的限制已经达到(参见getr‐中RLIMIT_NOFILE的讨论)
限制(2))。
```



# 1举例

## 1.1复制文件描述符（重定向）

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char* argv[])
{
    int i_fd = open("hello.txt", O_CREAT|O_APPEND|O_RDWR, 0666);

    if(i_fd < 0)
    {
        printf("open error!\n");
        return 0;
    }

    if(write(i_fd, "hello fd\n", 9) != 9)
     {
        printf("write fd error\n");

    }
	//这里操作之后，原本句柄3就被句柄4复制，也就是句柄3和句柄4都指向真实的hello.txt，关掉任意一个对另外一个都没影响
    int i_dup_fd = dup(i_fd);
    if(i_dup_fd < 0)
    {
        printf("dup error!\n");
        return 0;
    }

    printf("i_dup_fd = %d \t i_fd = %d\n", i_dup_fd, i_fd);
    close(i_fd);

    char c_buffer[100];
    int n = 0;
    while((n = read(STDIN_FILENO, c_buffer, 1000)) != 0)
    {
        if(write(i_dup_fd, c_buffer, n) != n)
        {
            printf("write dup fd error!\n");
            return 0;
        }
    }
    return 0;
}
```

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_1.png" thumbnail="/linux/func/dup_dup2/dup_dup2_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_2.png" thumbnail="/linux/func/dup_dup2/dup_dup2_2.png" title="">}}

## 1.2当前可用文件描述符中的最小数值

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char* argv[])
{
    int i_fd = open("hello.txt", O_CREAT|O_APPEND|O_RDWR, 0666);

    if(i_fd < 0)
    {
        printf("open error!\n");
        return 0;
    }

    if(write(i_fd, "hello fd\n", 9) != 9)
     {
        printf("write fd error\n");

    }
    //在重定向新句柄前先把默认的句柄0关闭了，那么复制的句柄将是当前可用最小的句柄，即为0
    close(STDIN_FILENO);
    int i_dup_fd = dup(i_fd);
    if(i_dup_fd < 0)
    {
        printf("dup error!\n");
        return 0;
    }

    printf("i_dup_fd = %d \t i_fd = %d\n", i_dup_fd, i_fd);
    close(i_fd);
    close(i_dup_fd);

    return 0;
}
```

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_3.png" thumbnail="/linux/func/dup_dup2/dup_dup2_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_4.png" thumbnail="/linux/func/dup_dup2/dup_dup2_4.png" title="">}}

## 1.3 dup2

这里的`dup2`，会通过后者设置的标准输出，重定向为文件的句柄`file_fd`，然后在重定向回来为标准输出。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{
    //先调用dup将标准输出拷贝一份，指向真正的标准输出
    int stdout_copy_fd = dup(STDOUT_FILENO);
    printf("stdout_copy_fd = %d \t STDOUT_FILENO = %d\n", stdout_copy_fd, STDOUT_FILENO);
    int file_fd = open("hello_dup2.txt", O_CREAT|O_APPEND|O_RDWR, 0666);
    //让标准输出指向文件
    int dup2_fd = dup2(file_fd, STDOUT_FILENO);
    //printf("before dup2_fd = %d \t file_fd = %d\n", dup2_fd, file_fd);
    //实际上printf("hello\n")也可以写成fprintf(STDOUT_FILENO, "hello\n");
    printf("hello\n");
    //刷新缓冲区
	//fflush(stdout);
    //恢复标准输出
    int dup2_recovery_fd =dup2(stdout_copy_fd, STDOUT_FILENO);
    printf("world\n");
    //printf("after dup2_fd = %d \t file_fd = %d \t dup2_recovery_fd = %d\n", dup2_fd, file_fd, dup2_recovery_fd);
    close(file_fd);
    close(stdout_copy_fd);
    return 0;
}
```

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_5.png" thumbnail="/linux/func/dup_dup2/dup_dup2_5.png" title="">}}

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_6.png" thumbnail="/linux/func/dup_dup2/dup_dup2_6.png" title="">}}

> 注：如果出现调用的两次`printf`仍然出现在屏幕上。。`hello`并没有写入到文件中。
>
> 这个时候需要刷新缓冲区，再重新输出。
>
> ```c
> //让标准输出指向文件
> dup2(file_fd, STDOUT_FILENO);
> printf("hello\n");
> //刷新缓冲区
> fflush(stdout);
> //恢复标准输出
> dup2(stdout_copy_fd, STDOUT_FILENO);
> ```



# 2源码（与pipe一起使用）

## 2.1libc中的debuggerd

以Native crash中的源码为例

```c++
//system/core/debuggerd/handler/debuggerd_handler.cpp
static int debuggerd_dispatch_pseudothread(void* arg) {
  //首先把标准输出和标准输入都置为null,这里的/dev/null用于丢弃整个输出中的标准输出或标准错误
  int devnull = TEMP_FAILURE_RETRY(open("/dev/null", O_RDWR));
  TEMP_FAILURE_RETRY(dup2(devnull, 1));
  TEMP_FAILURE_RETRY(dup2(devnull, 2));
  //这里设置管道，管道的用途，一端的管道写，一段的管道读	
  unique_fd input_read, input_write;
  unique_fd output_read, output_write;
  if (!Pipe(&input_read, &input_write) != 0 || !Pipe(&output_read, &output_write)) {
    fatal_errno("failed to create pipe");
  }
  //这种数据结构调用中读、写多个非连续缓冲区  
  struct iovec iovs[] = {
      {.iov_base = &version, .iov_len = sizeof(version)},
      {.iov_base = thread_info->siginfo, .iov_len = sizeof(siginfo_t)},
      {.iov_base = thread_info->ucontext, .iov_len = sizeof(ucontext_t)},
      {.iov_base = &thread_info->abort_msg, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->fdsan_table, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->gwp_asan_state, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->gwp_asan_metadata, .iov_len = sizeof(uintptr_t)},
  };
  //将数据以iov的形式传递
  ssize_t rc = TEMP_FAILURE_RETRY(writev(output_write.get(), iovs, arraysize(iovs)));
  //将debuggerd_handler所在进程fork复制一份
  pid_t crash_dump_pid = __fork();
  if (crash_dump_pid == -1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc",
                          "failed to fork in debuggerd signal handler: %s", strerror(errno));
  } else if (crash_dump_pid == 0) {
    //这个是子进程  
    TEMP_FAILURE_RETRY(dup2(input_write.get(), STDOUT_FILENO));
    TEMP_FAILURE_RETRY(dup2(output_read.get(), STDIN_FILENO));
    input_read.reset();
    input_write.reset();
    output_read.reset();
    output_write.reset();
    //这个是crash_dump进程
    execle(CRASH_DUMP_PATH, CRASH_DUMP_NAME, main_tid, pseudothread_tid, debuggerd_dump_type,
           nullptr, nullptr);

    return 1;
  }
  //外面是父进程
  input_write.reset();
  output_read.reset();
  char buf[4];
  rc = TEMP_FAILURE_RETRY(read(input_read.get(), &buf, sizeof(buf)));
  if (rc == -1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc", "read of IPC pipe failed: %s", strerror(errno));
    return 1;
  } else if (rc == 0) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc", "crash_dump helper failed to exec");
    return 1;
  } else if (rc != 1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc",
                          "read of IPC pipe returned unexpected value: %zd", rc);
    return 1;
  } else if (buf[0] != '\1') {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc", "crash_dump helper reported failure");
    return 1;
  }
}
```

把父子进程分成进程A和进程B

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_7.png" thumbnail="/linux/func/dup_dup2/dup_dup2_7.png" title="">}}

考虑到其中四个句柄是比较特殊的句柄，它们是管道，管道的一边是写，管道的一边是读

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_8.png" thumbnail="/linux/func/dup_dup2/dup_dup2_8.png" title="">}}

最终合并起来的父子进程表项

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_9.png" thumbnail="/linux/func/dup_dup2/dup_dup2_9.png" title="">}}

## 2.2`crash_dump`进程

```c++
int main(int argc, char** argv) {
  //将当前进程的标准输出复制为output_pipe，将便准输入复制为input_pipe
  unique_fd output_pipe(dup(STDOUT_FILENO));
  unique_fd input_pipe(dup(STDIN_FILENO));
  //出现的crash_dump进程之后又开始fork出一个新进程，将原先的进程退出  
  pid_t forkpid = fork();
  if (forkpid == -1) {
    PLOG(FATAL) << "fork failed";
  } else if (forkpid == 0) {
    //子进程不退出
    fork_exit_read.reset();
  } else {
    // We need the pseudothread to live until we get around to verifying the vm pid against it.
    // The last thing it does is block on a waitpid on us, so wait until our child tells us to die.
    //父进程退出
    fork_exit_write.reset();
    char buf;
    TEMP_FAILURE_RETRY(read(fork_exit_read.get(), &buf, sizeof(buf)));
    _exit(0);
  }
    
  {
    ATRACE_NAME("ptrace");
    ...
      if (thread == g_target_thread) {
        //这里重要是的ReadCrashInfo，将之前保存的CrashInfo读取出来，通过input_pipe可以获取iov传输的数据
        ReadCrashInfo(input_pipe, &siginfo, &info.registers, &abort_msg_address,
                      &fdsan_table_address, &gwp_asan_state, &gwp_asan_metadata);
        info.siginfo = &siginfo;
        info.signo = info.siginfo->si_signo;
      } else {
        ...
        }
      }

      thread_info[thread] = std::move(info);
    }
  }  
  //将数据写入到output_pipe句柄中
  if (TEMP_FAILURE_RETRY(write(output_pipe.get(), "\1", 1)) != 1) {
    PLOG(FATAL) << "failed to write to pseudothread";
  }
}
```

`crash_dump`进程`fork`出一个进程C之后

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_10.png" thumbnail="/linux/func/dup_dup2/dup_dup2_10.png" title="">}}

最终进程B死掉，进程A和进程C通过管道通信

{{< image classes="fancybox center fig-100" src="/linux/func/dup_dup2/dup_dup2_11.png" thumbnail="/linux/func/dup_dup2/dup_dup2_11.png" title="">}}


## 2.3总结

之所以上述的`Native crash`中的源码大费周章的将`input_write`和`output_read`重定向为标准输出和标准输入，然后又将标准输入和表述输出句柄`copy`到进程的新句柄`output_pipe`和`input_pipe`中。主要原因是这里用到了父子进程，其中子进程执行`exec`之后，变成了新的进程，原先的句柄无法直接通过变量带过来，可以通过标准输入和标准输出的形式将句柄复制过来。




# 参考

[[1] 非正经程序员, 浅谈dup和dup2的用法, 2017.](https://blog.csdn.net/u012058778/article/details/78705536)

[[2] Hackergin, dup和dup2用法小结, 2016.](https://www.cnblogs.com/0x12345678/p/5847734.html)

[[3] mayue_csdn, readv和writev函数, 2022.](https://blog.csdn.net/mayue_web/article/details/88672296)