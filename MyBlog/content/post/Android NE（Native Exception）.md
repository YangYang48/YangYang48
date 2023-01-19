---
title: "Android NE（Native Exception）"
date: 2023-01-07T4:00:00
thumbnailImagePosition: left
thumbnailImage: NE/ne_thumb.jpg
coverImage: NE/ne_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- NE
- 2023
- January
tags:
- c++
- 源码
- libc
- tombstone
- crash_dump
- socket
- clone
- fork
- signal
- dup_dup2
- pipe
showSocial: false
---
Android是基于Linux存在的，常常有使用c/c++代码库编写native进程或者动态库的情况，这些库在运行时发生的异常统称Native Exception。

<!--more-->
# 0 NE简介

通常native代码，出现错误，比如空指针异常，出现signal 11的信号错误，会打印如下的堆栈信息

```text
--------- beginning of crash
08-04 17:02:06.305   361   361 F libc    : Fatal signal 11 (SIGSEGV), code 1, fault addr 0xa in tid 361 (demo), pid 361 (demo)
08-04 17:02:06.305   361   361 F libc    : ->>>after clone|pseudothread_tid(-1)
08-04 17:02:06.306   361   361 F libc    : ->>>debuggerd_dispatch_pseudothread_1|pseudothread_tid(1668)
08-04 17:02:06.312  1669  1669 F libc    : ->>>main_tid(361),pseudothread_tid(1668),debuggerd_dump_type(1)
08-04 17:02:06.360   361   361 F libc    : ->>>1-1buf[0][1][2][3](0x1 0x0 0x0 0x0),rc(1),pseudothread_tid(1668)
08-04 17:02:06.360   361   361 F libc    : ->>>debuggerd_dispatch_pseudothread_1-2|pseudothread_tid(1668)
08-04 17:02:06.360   361   361 F libc    : ->>>debuggerd_dispatch_pseudothread_2|pseudothread_tid(1668)
08-04 17:02:06.361   361   361 F libc    : ->>>debuggerd_dispatch_pseudothread_3|pseudothread_tid(1668)
08-04 17:02:06.361   361   361 F libc    : ->>>after futex_wait|pseudothread_tid(0)
08-04 17:02:06.366  1671  1671 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
08-04 17:02:06.366  1671  1671 F DEBUG   : Build fingerprint: 'xxx'
08-04 17:02:06.366  1671  1671 F DEBUG   : Revision: '0'
08-04 17:02:06.366  1671  1671 F DEBUG   : ABI: 'arm'
08-04 17:02:06.366  1671  1671 F DEBUG   : pid: 361, tid: 361, name: demo  >>> /system/bin/demo <<<
08-04 17:02:06.366  1671  1671 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xa
08-04 17:02:06.366  1671  1671 F DEBUG   : Cause: null pointer dereference
08-04 17:02:06.366  1671  1671 F DEBUG   :     r0 0000000a  r1 00000061  r2 0538ad06  r3 fff162b0
08-04 17:02:06.366  1671  1671 F DEBUG   :     r4 b91d04ed  r5 b91dde08  r6 00000004  r7 fff1639c
08-04 17:02:06.366  1671  1671 F DEBUG   :     r8 00000000  r9 00000000  sl 00000000  fp fff1638c
08-04 17:02:06.367  1671  1671 F DEBUG   :     ip fff1639c  sp fff16308  lr eb97f0eb  pc b91d0342  cpsr 20010030
08-04 17:02:06.373  1671  1671 F DEBUG   :
08-04 17:02:06.373  1671  1671 F DEBUG   : backtrace:
08-04 17:02:06.373  1671  1671 F DEBUG   :     #00 pc 00006342  /system/bin/demo (main+489)
08-04 17:02:06.373  1671  1671 F DEBUG   :     #01 pc 0007669d  /system/lib/libc.so (__libc_init+48)
08-04 17:02:06.373  1671  1671 F DEBUG   :     #02 pc 000024d8  /system/bin/demo (_start_main+88)
```



# 1 NE流程分析

## 1.1从架构begin.S开始

```assembly
//bionic/linker/arch/arm/begin.S
ENTRY(_start)
  // Force unwinds to end in this function.
  .cfi_undefined r14
  mov r0, sp
  //汇编语言，开始调用函数__linker_init
  bl __linker_init
  bx r0
END(_start)
```

> `sp`寄存器为堆栈指针寄存器，为**栈顶指针**，指向栈顶地址
>
> `r0`寄存器用作传入函数参数，传出函数返回值，被调用函数在返回之前不必恢复`r0`寄存器
>
> `bx`为16位**基址寄存器**，且`lr`起存器保存`r0`的地址，其中地址还可以分为高位低位（`BH`，`BL`），常存放访问内存的地址
>
> `bl`调用子程序，其中`bl`跳转范围为（`-32MB~32MB`）
>
> 1.将栈顶元素，赋值给`r0`寄存器（16位）
>
> ```assembly
> mov r0, sp
> ```
>
> 2.调用子程序`__linker_init`，把返回地址保存在`lr`寄存器里面
>
> ```assembly
> bl __linker_init
> ```
>
> 3.返回子程序，并出现返回值，指向`__linker_init`的下一个地址
>
> ```assembly
> bx r0
> ```

## 1.2__linker_init

在`__linker_init()`方法中调用了`__linker_init_post_relocation()`，`__linker_init_post_relocation()`初始化`linker`的全局变量，之后调用`linker_main()`函数获取可执行程序的开始地址，然后跳转到开始地址继续执行。

可以看到`linker_main()`函数中对系统环境进行了确认，对系统属性进行了初始化设置，之后调用

```c++
//bionic/linker/linker_main.cpp
static ElfW(Addr) linker_main(KernelArgumentBlock& args, const char* exe_to_load) {
  ...
  // Sanitize the environment.
  __libc_init_AT_SECURE(args.envp);
  // Initialize system properties
  __system_properties_init(); // may use 'environ'
  // Register the debuggerd signal handler.
  linker_debuggerd_init();
  ...
}

extern "C" ElfW(Addr) __linker_init(void* raw_args) {
  ...  
  return __linker_init_post_relocation(args, tmp_linker_so);
}

static ElfW(Addr) __attribute__((noinline))
__linker_init_post_relocation(KernelArgumentBlock& args, soinfo& tmp_linker_so) {
  ...
  ElfW(Addr) start_address = linker_main(args, exe_to_load);
  ...  
}
```

## 1.3linker_debuggerd_init

设置几个回调方法，定义了一个callbacks结构体，之后把这个callbacks作为参数传递给debuggerd_init()

```c++
//bionic/linker/linker_debuggerd_android.cpp
void linker_debuggerd_init() {
  debuggerd_callbacks_t callbacks = {
    .get_abort_message = []() {
      return __libc_shared_globals()->abort_msg;
    },
    .post_dump = &notify_gdb_of_libraries,
    .get_gwp_asan_state = []() {
      return __libc_shared_globals()->gwp_asan_state;
    },
    .get_gwp_asan_metadata = []() {
      return __libc_shared_globals()->gwp_asan_metadata;
    },
  };
  debuggerd_init(&callbacks);
}
```

## 1.4debuggerd_init

- 首先把`callbacks`赋给`g_callbacks`
- 之后调用`mmap`为线程的栈分配空间
- 然后调用`mprotect`函数，设置`stack`对应的内存区的保护属性为可读可写，之后把栈的起始点定义在页末尾并对齐
- 最后调用`debuggerd_register_handlers()`去注册一些异常信号，而异常信号的处理函数是`debuggerd_signal_handler()`

```c++
//system/core/debuggerd/handler/debuggerd_handler.cpp
void debuggerd_init(debuggerd_callbacks_t* callbacks) {
  if (callbacks) {
    g_callbacks = *callbacks;
  }

  size_t thread_stack_pages = 8;
  //设置一个mmap匿名的空间
  void* thread_stack_allocation = mmap(nullptr, PAGE_SIZE * (thread_stack_pages + 2), PROT_NONE,
                                       MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  if (thread_stack_allocation == MAP_FAILED) {
    fatal_errno("failed to allocate debuggerd thread stack");
  }

  char* stack = static_cast<char*>(thread_stack_allocation) + PAGE_SIZE;
  if (mprotect(stack, PAGE_SIZE * thread_stack_pages, PROT_READ | PROT_WRITE) != 0) {
    fatal_errno("failed to mprotect debuggerd thread stack");
  }

  //设置一个栈，指向栈顶，大小为4*1024的整数倍，为后续clone的逻辑空间地址
  stack = (stack + thread_stack_pages * PAGE_SIZE - 1);
  stack -= 15;
  pseudothread_stack = stack;
  //为当前进程设置signal处理机制
  struct sigaction action;
  memset(&action, 0, sizeof(action));
  sigfillset(&action.sa_mask);
  action.sa_sigaction = debuggerd_signal_handler;
  //SA_RESTART代表函数程序不会异常终止，比如read，write等
  action.sa_flags = SA_RESTART | SA_SIGINFO;
  //如果可用，使用备用信号堆栈，这样我们可以捕捉堆栈溢出。
  action.sa_flags |= SA_ONSTACK;
  //信号注册函数如下所示
  debuggerd_register_handlers(&action);
}
```

> `debuggerd_register_handlers`注册了8种信号，`SIGABRT(6)`、`SIGBUS(7)`、`SIGFPE(8)`、`SIGILL(4)`、`SIGSEGV(11)`、`SIGSTKFLT(16)`、`SIGSYS(31)`、`SIGTRAP(5)`
>
> ```c++
> //system/core/debuggerd/include/debuggerd/handler.h
> static void __attribute__((__unused__)) debuggerd_register_handlers(struct sigaction* action) {
>   char value[PROP_VALUE_MAX] = "";
>   bool enabled =
>       !(__system_property_get("ro.debuggable", value) > 0 && !strcmp(value, "1") &&
>         __system_property_get("debug.debuggerd.disable", value) > 0 && !strcmp(value, "1"));
>   if (enabled) {
>     sigaction(SIGABRT, action, nullptr);
>     sigaction(SIGBUS, action, nullptr);
>     sigaction(SIGFPE, action, nullptr);
>     sigaction(SIGILL, action, nullptr);
>     sigaction(SIGSEGV, action, nullptr);
>     sigaction(SIGSTKFLT, action, nullptr);
>     sigaction(SIGSYS, action, nullptr);
>     sigaction(SIGTRAP, action, nullptr);
>   }
> 
>   sigaction(BIONIC_SIGNAL_DEBUGGER, action, nullptr);
> }
> ```
>
> 对应的信号为
>
> ```txt
>  1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
>  6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
> 11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
> 16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
> 21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
> 26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
> 31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
> 38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
> 43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
> 48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
> 53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
> 58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
> 63) SIGRTMAX-1  64) SIGRTMAX
> ```

## 1.5debuggerd_signal_handler

上述信号一旦产生，直接调用到这里

```c++
//system/core/debuggerd/handler/debuggerd_handler.cpp
static void debuggerd_signal_handler(int signal_number, siginfo_t* info, void* context) {
  ...
  if (signal_number == BIONIC_SIGNAL_DEBUGGER) {
      ...
  } else {
    if (g_callbacks.get_abort_message) {
      abort_message = g_callbacks.get_abort_message();
    }
    if (g_callbacks.get_gwp_asan_state) {
      gwp_asan_state = g_callbacks.get_gwp_asan_state();
    }
    if (g_callbacks.get_gwp_asan_metadata) {
      gwp_asan_metadata = g_callbacks.get_gwp_asan_metadata();
    }
  }
  //打印第一句信息
  //Fatal signal 11 (SIGSEGV), code 1, fault addr 0xa in tid 361 (demo), pid 361 (demo)
  log_signal_summary(info);

  debugger_thread_info thread_info = {
      .crashing_tid = __gettid(),
      .pseudothread_tid = -1,
      .siginfo = info,
      .ucontext = context,
      .abort_msg = reinterpret_cast<uintptr_t>(abort_message),
      .fdsan_table = reinterpret_cast<uintptr_t>(android_fdsan_get_fd_table()),
      .gwp_asan_state = reinterpret_cast<uintptr_t>(gwp_asan_state),
      .gwp_asan_metadata = reinterpret_cast<uintptr_t>(gwp_asan_metadata),
  };
  //开启一个新线程，设置CLONE_CHILD_SETTID之后会把thread_info.pseudothread_tid进入函数设置为新线程pid，退出又恢复
  pid_t child_pid =
    clone(debuggerd_dispatch_pseudothread, pseudothread_stack,
          CLONE_THREAD | CLONE_SIGHAND | CLONE_VM | CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID,
          &thread_info, nullptr, nullptr, &thread_info.pseudothread_tid);
  if (child_pid == -1) {
    fatal_errno("failed to spawn debuggerd dispatch thread");
  }
  //主线程等待
  //正好对应上述线程的进入和退出，等待进入函数debuggerd_dispatch_pseudothread
  futex_wait(&thread_info.pseudothread_tid, -1);
  //等待退出函数debuggerd_dispatch_pseudothread  
  futex_wait(&thread_info.pseudothread_tid, child_pid);
  ...
}
```

## 1.6debuggerd_dispatch_pseudothread

开启子线程

```c++
//system/core/debuggerd/handler/debuggerd_handler.cpp
static int debuggerd_dispatch_pseudothread(void* arg) {
  debugger_thread_info* thread_info = static_cast<debugger_thread_info*>(arg);
  //使用系统调用关闭socket句柄，不使用close用来避免bionic的文件描述符所有权检查
  for (int i = 0; i < 1024; ++i) {
    syscall(__NR_close, i);
  }
  //创建四个管道，后续的操作主要在管道中进行	
  unique_fd input_read, input_write;
  unique_fd output_read, output_write;
  if (!Pipe(&input_read, &input_write) != 0 || !Pipe(&output_read, &output_write)) {
    fatal_errno("failed to create pipe");
  }
  ...
  struct iovec iovs[] = {
      {.iov_base = &version, .iov_len = sizeof(version)},
      {.iov_base = thread_info->siginfo, .iov_len = sizeof(siginfo_t)},
      {.iov_base = thread_info->ucontext, .iov_len = sizeof(ucontext_t)},
      {.iov_base = &thread_info->abort_msg, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->fdsan_table, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->gwp_asan_state, .iov_len = sizeof(uintptr_t)},
      {.iov_base = &thread_info->gwp_asan_metadata, .iov_len = sizeof(uintptr_t)},
  };
  //当前线程写入一个iovs给管道句柄output_write，后续在crash_dump进程中接收
  ssize_t rc = TEMP_FAILURE_RETRY(writev(output_write.get(), iovs, arraysize(iovs)));


  //fork一个子进程，这里运用了clone的方式来fork，用来避免调用pthread_atfork处理程序
  pid_t crash_dump_pid = __fork();
  if (crash_dump_pid == -1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc",
                          "failed to fork in debuggerd signal handler: %s", strerror(errno));
  } else if (crash_dump_pid == 0) {
    //子进程最终通过调用/system/bin/crash_dump调用到这个进程中
    TEMP_FAILURE_RETRY(dup2(input_write.get(), STDOUT_FILENO));
    TEMP_FAILURE_RETRY(dup2(output_read.get(), STDIN_FILENO));
    input_read.reset();
    input_write.reset();
    output_read.reset();
    output_write.reset();
    ...  
    async_safe_format_buffer(main_tid, sizeof(main_tid), "%d", thread_info->crashing_tid);
    async_safe_format_buffer(pseudothread_tid, sizeof(pseudothread_tid), "%d",
                             thread_info->pseudothread_tid);
    async_safe_format_buffer(debuggerd_dump_type, sizeof(debuggerd_dump_type), "%d",
                             get_dump_type(thread_info));
    //这里传入三个参数
    //main_tid为当前crash进程号，
    //pseudothread_tid为当前的子线程号，即之前主线程clone出来的线程号
    //debuggerd_dump_type为kDebuggerdTombstone  
    execle(CRASH_DUMP_PATH, CRASH_DUMP_NAME, main_tid, pseudothread_tid, debuggerd_dump_type,
           nullptr, nullptr);
    return 1;
  }

  input_write.reset();
  output_read.reset();
  //父进程等待，通过管道句柄input_read接收到刚刚已经fork的子进程的一个\1
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

  // Crash_dump正在跟踪我们，复制我们的地址空间的副本供它使用
  create_vm_process();
  //等待子进程结束，防止变成僵尸进程
  int status;
  if (TEMP_FAILURE_RETRY(waitpid(crash_dump_pid, &status, 0)) == -1) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc", "failed to wait for crash_dump helper: %s",
                          strerror(errno));
  } else if (WIFSTOPPED(status) || WIFSIGNALED(status)) {
    async_safe_format_log(ANDROID_LOG_FATAL, "libc", "crash_dump helper crashed or stopped");
  }
  return 0;
}
```

## 1.7crash_dump

进入/system/bin/crash_dump进程的入口函数

- 初始化，包括初始化相关信号处理，初始化管道，并且fork一个子进程
- 子父进程处理，父进程等待子进程退出，子进程解析传入的三个参数，传入子进程的iovs，并将相关crash的子线程保存
- 客户端连接tombstoned服务端，并且返回一个文件句柄g_output_fd
- 打印部分堆栈信息，将所有堆栈信息写入到句柄g_output_fd中去
- 通知AMS做进一步的堆栈处理
- 通知tombstoned服务端，将`tombstone`文件生成

```c++
//system/core/debuggerd/crash_dump.cpp
int main(int argc, char** argv) {
  //初始化相关信号处理  
  DefuseSignalHandlers();
  InstallSigPipeHandler();
  // 内核中似乎有一个错误，我们的死亡会导致SIGHUP发送到我们的进程组
  //如果我们退出，而它已经停止了工作(例如，因为wait_for_gdb)。
  //使用setsid创建一个新的进程组来避免碰到这个问题。
  setsid();
  //打开进程的/proc/pid作为target_proc_fd句柄
  std::string target_proc_path = "/proc/" + std::to_string(target_process);
  int target_proc_fd = open(target_proc_path.c_str(), O_DIRECTORY | O_RDONLY);
  //把之前的两个句柄还原回来
  unique_fd output_pipe(dup(STDOUT_FILENO));
  unique_fd input_pipe(dup(STDIN_FILENO));
  //这里的句柄是为了判断是否还有数据
  unique_fd fork_exit_read, fork_exit_write;
  if (!Pipe(&fork_exit_read, &fork_exit_write)) {
    PLOG(FATAL) << "failed to create pipe";
  }
  //这里创建一个新子进程和当前进程一致
  pid_t forkpid = fork();
  if (forkpid == -1) {
    PLOG(FATAL) << "fork failed";
  } else if (forkpid == 0) {
    //子进程保留写，其实是没有数据可写  
    fork_exit_read.reset();
  } else {
    //我们需要伪线程存在，直到我们开始验证vm pid。它做的最后一件事是阻塞我们的waitpid，所以等待直到我们的child告诉我们死亡
    //阻塞，直到子进程退出，当前的父进程也退出  
    fork_exit_write.reset();
    char buf;
    TEMP_FAILURE_RETRY(read(fork_exit_read.get(), &buf, sizeof(buf)));
    _exit(0);
  }
  //子进程
  //提取出传入的三个参数，pseudothread_tid为上述clone的子线程号，dump_type为kDebuggerdTombstone
  Initialize(argv);
  ParseArgs(argc, argv, &pseudothread_tid, &dump_type);
  //获取crash进程所有的线程号
  std::set<pid_t> threads;
  if (!android::procinfo::GetProcessTids(g_target_thread, &threads)) {
    PLOG(FATAL) << "failed to get process threads";
  }

  std::map<pid_t, ThreadInfo> thread_info;
  siginfo_t siginfo;
  std::string error;

  {
    ATRACE_NAME("ptrace");
    for (pid_t thread : threads) {
      if (thread == pseudothread_tid) {
        continue;
      }
      ThreadInfo info;
      info.pid = target_process;
      info.tid = thread;
      info.uid = getuid();
      info.process_name = process_name;
      info.thread_name = get_thread_name(thread);
      if (thread == g_target_thread) {
        //最重要的是这个，这里的input_pipe对应之前crash子线程中的iovs，这里是真正收集到堆栈信息的函数
        ReadCrashInfo(input_pipe, &siginfo, &info.registers, &abort_msg_address,
                      &fdsan_table_address, &gwp_asan_state, &gwp_asan_metadata);
        info.siginfo = &siginfo;
        info.signo = info.siginfo->si_signo;
      } else {
        info.registers.reset(unwindstack::Regs::RemoteGet(thread));
        if (!info.registers) {
          PLOG(WARNING) << "failed to fetch registers for thread " << thread;
          ptrace(PTRACE_DETACH, thread, 0, 0);
          continue;
        }
      }

      thread_info[thread] = std::move(info);
    }
  }

  // Trace the pseudothread with PTRACE_O_TRACECLONE and tell it to fork.
  if (!ptrace_seize_thread(target_proc_fd, pseudothread_tid, &error, PTRACE_O_TRACECLONE)) {
    LOG(FATAL) << "failed to seize pseudothread: " << error;
  }
  //通过管道句柄output_pipe，回写给之前crash子线程
  if (TEMP_FAILURE_RETRY(write(output_pipe.get(), "\1", 1)) != 1) {
    PLOG(FATAL) << "failed to write to pseudothread";
  }
  //这里会double fork，用于作为守护进程
  //第一次fork是为了脱离父进程，setsid让子进程变成session leader，脱离控制终端
  //第二次fork是因为session leader有可能会获取控制终端，这样终端断开会发送信号到该进程，导致退出
  //并且开始ptrace各个线程
  pid_t vm_pid = wait_for_vm_process(pseudothread_tid);
  if (ptrace(PTRACE_DETACH, pseudothread_tid, 0, 0) != 0) {
    PLOG(FATAL) << "failed to detach from pseudothread";
  }
  fork_exit_write.reset();
  for (const auto& [tid, thread] : thread_info) {
    int resume_signal = thread.signo == BIONIC_SIGNAL_DEBUGGER ? 0 : thread.signo;
    LOG(DEBUG) << "detaching from thread " << tid;
    if (ptrace(PTRACE_DETACH, tid, 0, resume_signal) != 0) {
      PLOG(ERROR) << "failed to detach from thread " << tid;
    }
  }

  //tombstoned_connect连接，作为客户端连接tombstoned服务端，并且返回一个文件句柄g_output_fd
  {
    ATRACE_NAME("tombstoned_connect");
    LOG(INFO) << "obtaining output fd from tombstoned, type: " << dump_type;
    g_tombstoned_connected =
        tombstoned_connect(g_target_thread, &g_tombstoned_socket, &g_output_fd, dump_type);
  }

  int signo = siginfo.si_signo;
  bool fatal_signal = signo != BIONIC_SIGNAL_DEBUGGER;
  bool backtrace = false;
  // si_value is special when used with BIONIC_SIGNAL_DEBUGGER.
  //   0: dump tombstone
  //   1: dump backtrace
  if (!fatal_signal) {
    int si_val = siginfo.si_value.sival_int;
    if (si_val == 0) {
      backtrace = false;
    } else if (si_val == 1) {
      backtrace = true;
    } else {
      LOG(WARNING) << "unknown si_value value " << si_val;
    }
  }

  //初始化寄存器信息
  unwindstack::UnwinderFromPid unwinder(256, vm_pid);
  if (!unwinder.Init(unwindstack::Regs::CurrentArch())) {
    LOG(FATAL) << "Failed to init unwinder object.";
  }

  std::string amfd_data;
  if (backtrace) {
    ...
  } else {
      //打印堆栈信息
      engrave_tombstone(std::move(g_output_fd), &unwinder, thread_info, g_target_thread,
                        abort_msg_address, &open_files, &amfd_data, gwp_asan_state,
                        gwp_asan_metadata);

  }
  //通知AMS
  if (fatal_signal) {
    if (thread_info[target_process].thread_name != "system_server") {
      activity_manager_notify(target_process, signo, amfd_data);
    }
  }
  close(STDOUT_FILENO);
  //通知写tombstoned文件  
  if (g_tombstoned_connected && !tombstoned_notify_completion(g_tombstoned_socket.get())) {
    LOG(ERROR) << "failed to notify tombstoned of completion";
  }

  return 0;
}
```

上述的内容比较的，主要是新建立的几个`socket`处理

### 1.7.1tombstoned_connect

客户端连接`tombstoned`服务端，并且返回一个文件句柄`g_output_fd`

#### 1.7.1.1客户端

```c++
//system/core/debuggerd/tombstoned/tombstoned_client.cpp
bool tombstoned_connect(pid_t pid, unique_fd* tombstoned_socket, unique_fd* output_fd,
                        DebuggerdDumpType dump_type) {
  //建立socket连接，这里是kTombstonedCrashSocketName,对应/dev/socket/crash_dump
  //使用SOCK_SEQPACKET传输  
  unique_fd sockfd(
      socket_local_client((dump_type != kDebuggerdJavaBacktrace ? kTombstonedCrashSocketName
                                                                : kTombstonedJavaTraceSocketName),
                          ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_SEQPACKET));
  }
  //使用kDumpRequest，pid，kDebuggerdTombstone数据传输
  TombstonedCrashPacket packet = {};
  packet.packet_type = CrashPacketType::kDumpRequest;
  packet.packet.dump_request.pid = pid;
  packet.packet.dump_request.dump_type = dump_type;
  if (TEMP_FAILURE_RETRY(write(sockfd, &packet, sizeof(packet))) != sizeof(packet)) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "failed to write DumpRequest packet: %s",
                          strerror(errno));
    return false;
  }
  //收到服务端的packet，并且多携带了一个文件句柄tmp_output_fd
  unique_fd tmp_output_fd;
  ssize_t rc = ReceiveFileDescriptors(sockfd, &packet, sizeof(packet), &tmp_output_fd);
  *tombstoned_socket = std::move(sockfd);
  *output_fd = std::move(tmp_output_fd);
  return true;
}
```

#### 1.7.1.2服务端

```c++
//system/core/debuggerd/tombstoned/tombstoned.cpp
//tombstoned是开机就启动的进程
int main(int, char* []) {
  umask(0137);

  //初始化信号量，并且注册进去
  struct sigaction action = {};
  action.sa_handler = [](int signal) {
    LOG(ERROR) << "received fatal signal " << signal;
    _exit(1);
  };
  debuggerd_register_handlers(&action);
  //初始化句柄，对应/dev/socket/tombstoned_java_trace
  int intercept_socket = android_get_control_socket(kTombstonedInterceptSocketName);
  //初始化句柄，对应/dev/socket/crash_dump
  int crash_socket = android_get_control_socket(kTombstonedCrashSocketName);

  evutil_make_socket_nonblocking(intercept_socket);
  evutil_make_socket_nonblocking(crash_socket);
  //这里用到了event数据结构，最终回调到crash_accept_cb函数
  evconnlistener* tombstone_listener =
      evconnlistener_new(base, crash_accept_cb, CrashQueue::for_tombstones(), LEV_OPT_CLOSE_ON_FREE,
                         -1 /* backlog */, crash_socket);

  //这里初始化成功，开始处理数据，因为这个函数是轮询，所以不会退出tombstoned进程
  event_base_dispatch(base);
}
```

回调到`crash_accept_cb`函数

服务端分别调用四个函数

1. `crash_accept_cb`，接收到客户端连接对应的句柄
2. `crash_request_cb`，接收客户端发的信息并校验
3. `perform_request`，发给客户端一个文件句柄
4. `crash_completed_cb`，接收客户端完成的信息并校验，通过`linkat`将之前的`fd`句柄对应到新文件名`/data/tombstones/tombstone_xxx`

```c++
//system/core/debuggerd/tombstoned/tombstoned.cpp
static void crash_accept_cb(evconnlistener* listener, evutil_socket_t sockfd, sockaddr*, int,
                            void*) {
  event_base* base = evconnlistener_get_base(listener);
  Crash* crash = new Crash();

  // TODO: Make sure that only java crashes come in on the java socket
  // and only native crashes on the native socket.
  struct timeval timeout = { 1, 0 };
  //注册crash_request_cb函数，并且函数参数为crash
  event* crash_event = event_new(base, sockfd, EV_TIMEOUT | EV_READ, crash_request_cb, crash);
  crash->crash_socket_fd.reset(sockfd);
  crash->crash_event = crash_event;
  event_add(crash_event, &timeout);
}
//传入的参数args为上述的crash
static void crash_request_cb(evutil_socket_t sockfd, short ev, void* arg) {
  ssize_t rc;
  Crash* crash = static_cast<Crash*>(arg);
  //服务端接收一个request，并且校验客户端传入的数据request
  TombstonedCrashPacket request = {};
  rc = TEMP_FAILURE_RETRY(read(sockfd, &request, sizeof(request)));
  if (request.packet_type != CrashPacketType::kDumpRequest) {
    LOG(WARNING) << "unexpected crash packet type, expected kDumpRequest, received  "
                 << StringPrintf("%#2hhX", request.packet_type);
    goto fail;
  }
  ...
  crash->crash_type = request.packet.dump_request.dump_type;
  if (CrashQueue::for_crash(crash)->maybe_enqueue_crash(crash)) {
    LOG(INFO) << "enqueueing crash request for pid " << crash->crash_pid;
  } else {
    //调用到这里
    perform_request(crash);
  }

  return;
}
//处理这个请求
static void perform_request(Crash* crash) {
  unique_fd output_fd;
  bool intercepted =
      intercept_manager->GetIntercept(crash->crash_pid, crash->crash_type, &output_fd);
  if (!intercepted) {
    if (crash->crash_type == kDebuggerdNativeBacktrace) {
      ...
    } else {
      //这里没有intercepted，会调用到这里，并且将/data/tombstone/tombstone_xxx文件的句柄传出，为output_fd
      std::tie(crash->crash_tombstone_path, output_fd) = CrashQueue::for_crash(crash)->get_output();
      crash->crash_tombstone_fd.reset(dup(output_fd.get()));
    }
  }
  //传入的参数为kPerformDump，pid，output_fd
  TombstonedCrashPacket response = {
    .packet_type = CrashPacketType::kPerformDump
  };
  ssize_t rc =
      SendFileDescriptors(crash->crash_socket_fd, &response, sizeof(response), output_fd.get());
  output_fd.reset();

    // TODO: Make this configurable by the interceptor?
    struct timeval timeout = { 10, 0 };

    event_base* base = event_get_base(crash->crash_event);
    //最终调用crash_completed_cb,参数为crash
    event_assign(crash->crash_event, base, crash->crash_socket_fd, EV_TIMEOUT | EV_READ,
                 crash_completed_cb, crash);
    event_add(crash->crash_event, &timeout);


  CrashQueue::for_crash(crash)->on_crash_started();
  return;
}

static void crash_completed_cb(evutil_socket_t sockfd, short ev, void* arg) {
  ssize_t rc;
  Crash* crash = static_cast<Crash*>(arg);
  TombstonedCrashPacket request = {};
  ...
  //开始阻塞读取，直到有数据过来    
  rc = TEMP_FAILURE_RETRY(read(sockfd, &request, sizeof(request)));
  //数据过来之后，校验数据的合法性  
  if (request.packet_type != CrashPacketType::kCompletedDump) {
    LOG(WARNING) << "unexpected crash packet type, expected kCompletedDump, received "
                 << uint32_t(request.packet_type);
    goto fail;
  }
  //校验通过之后，通过linkat将之前的fd句柄对应到新文件名/data/tombstones/tombstone_xxx
  if (crash->crash_tombstone_fd != -1) {
    //这个句柄就是之前的output_fd  
    std::string fd_path = StringPrintf("/proc/self/fd/%d", crash->crash_tombstone_fd.get());
    std::string tombstone_path = CrashQueue::for_crash(crash)->get_next_artifact_path();
    rc = linkat(AT_FDCWD, fd_path.c_str(), AT_FDCWD, tombstone_path.c_str(), AT_SYMLINK_FOLLOW);
    ...
  }
}
```



### 1.7.2engrave_tombstone

打印部分堆栈信息，将所有堆栈信息写入到句柄`g_output_fd`中去

```c++
//system/core/debuggerd/libdebuggerd/tombstone.cpp
void engrave_tombstone_ucontext(int tombstone_fd, uint64_t abort_msg_address, siginfo_t* siginfo,
                                ucontext_t* ucontext) {
  //初始化信息
  ...
  std::unique_ptr<unwindstack::Regs> regs(
      unwindstack::Regs::CreateFromUcontext(unwindstack::Regs::CurrentArch(), ucontext));

  std::map<pid_t, ThreadInfo> threads;
  threads[gettid()] = ThreadInfo{
      .registers = std::move(regs),
      .uid = uid,
      .tid = tid,
      .thread_name = thread_name,
      .pid = pid,
      .process_name = process_name,
      .siginfo = siginfo,
  };
  //初始化寄存器
  unwindstack::UnwinderFromPid unwinder(kMaxFrames, pid);
  if (!unwinder.Init(unwindstack::Regs::CurrentArch())) {
    LOG(FATAL) << "Failed to init unwinder object.";
  }

  engrave_tombstone(unique_fd(dup(tombstone_fd)), &unwinder, threads, tid, abort_msg_address,
                    nullptr, nullptr, 0u, 0u);
}

void engrave_tombstone(unique_fd output_fd, unwindstack::Unwinder* unwinder,
                       const std::map<pid_t, ThreadInfo>& threads, pid_t target_thread,
                       uint64_t abort_msg_address, OpenFilesList* open_files,
                       std::string* amfd_data, uintptr_t gwp_asan_state_ptr,
                       uintptr_t gwp_asan_metadata_ptr) {
  // don't copy log messages to tombstone unless this is a dev device
  bool want_logs = android::base::GetBoolProperty("ro.debuggable", false);

  log_t log;
  log.current_tid = target_thread;
  log.crashed_tid = target_thread;
  //设置log中的tfd的句柄为先前的文件句柄output_fd
  log.tfd = output_fd.get();
  log.amfd_data = amfd_data;
  //开始打印第一行
  _LOG(&log, logtype::HEADER, "*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***\n");
  //打印header，包括架构信息版本号等  
  dump_header_info(&log);
  //打印时间戳
  dump_timestamp(&log, time(nullptr));

  auto it = threads.find(target_thread);
  if (it == threads.end()) {
    LOG(FATAL) << "failed to find target thread";
  }

  GwpAsanCrashData gwp_asan_crash_data(unwinder->GetProcessMemory().get(),
                                       gwp_asan_state_ptr,
                                       gwp_asan_metadata_ptr, it->second);
  //打印线程中的进程号线程号、信号及其产生原因、堆栈等信息
  dump_thread(&log, unwinder, it->second, abort_msg_address, true,
              gwp_asan_crash_data);

  if (want_logs) {
    dump_logs(&log, it->second.pid, 50);
  }

  for (auto& [tid, thread_info] : threads) {
    if (tid == target_thread) {
      continue;
    }
    //打印相关线程的相关信息
    dump_thread(&log, unwinder, thread_info, 0, false, gwp_asan_crash_data);
  }

  if (open_files) {
    _LOG(&log, logtype::OPEN_FILES, "\nopen files:\n");
    dump_open_files_list(&log, *open_files, "    ");
  }

  if (want_logs) {
    dump_logs(&log, it->second.pid, 0);
  }
}
```

> 这里的部分堆栈打印原理，只选择性的打印三类：`HEADER`、`REGISTERS`、`BACKTRACE`
>
> ```c++
> __attribute__((__weak__, visibility("default")))
> void _LOG(log_t* log, enum logtype ltype, const char* fmt, ...) {
>   va_list ap;
>   va_start(ap, fmt);
>   _VLOG(log, ltype, fmt, ap);
>   va_end(ap);
> }
> 
> __attribute__((__weak__, visibility("default")))
> void _VLOG(log_t* log, enum logtype ltype, const char* fmt, va_list ap) {
>   bool write_to_tombstone = (log->tfd != -1);
>   bool write_to_logcat = is_allowed_in_logcat(ltype)
>                       && log->crashed_tid != -1
>                       && log->current_tid != -1
>                       && (log->crashed_tid == log->current_tid);
>   //是否加入kmsg日志  
>   static bool write_to_kmsg = should_write_to_kmsg();
> 
>   std::string msg;
>   android::base::StringAppendV(&msg, fmt, ap);
> 
>   if (msg.empty()) return;
>   //之前传入的tfd已经是指定的文件句柄，所有的数据都会被写到该句柄中
>   if (write_to_tombstone) {
>     TEMP_FAILURE_RETRY(write(log->tfd, msg.c_str(), msg.size()));
>   }
>   //根据是否是HEADER、REGISTERS、BACKTRACE判断是否需要部分写入logcat
>   if (write_to_logcat) {
>     __android_log_buf_write(LOG_ID_CRASH, ANDROID_LOG_FATAL, LOG_TAG, msg.c_str());
>     if (log->amfd_data != nullptr) {
>       *log->amfd_data += msg;
>     }
> 
>     if (write_to_kmsg) {
>       unique_fd kmsg_fd(open("/dev/kmsg_debug", O_WRONLY | O_APPEND | O_CLOEXEC));
>       if (kmsg_fd.get() >= 0) {
>         std::vector<std::string> fragments = android::base::Split(msg, "\n");
>         for (const std::string& fragment : fragments) {
>           static constexpr char prefix[] = "<3>DEBUG: ";
>           struct iovec iov[3];
>           iov[0].iov_base = const_cast<char*>(prefix);
>           iov[0].iov_len = strlen(prefix);
>           iov[1].iov_base = const_cast<char*>(fragment.c_str());
>           iov[1].iov_len = fragment.length();
>           iov[2].iov_base = const_cast<char*>("\n");
>           iov[2].iov_len = 1;
>           TEMP_FAILURE_RETRY(writev(kmsg_fd.get(), iov, 3));
>         }
>       }
>     }
>   }
> }
> 
> bool is_allowed_in_logcat(enum logtype ltype) {
>   if ((ltype == HEADER)
>    || (ltype == REGISTERS)
>    || (ltype == BACKTRACE)) {
>     return true;
>   }
>   return false;
> }
> ```

着重突出

1）

```c++
_LOG(&log, logtype::HEADER, "*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***\n");
```

对应

```txt
08-04 17:02:06.366  1671  1671 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
```

2）`dump_header_info`对应

```c++
08-04 17:02:06.366  1671  1671 F DEBUG   : Build fingerprint: 'xxx'
08-04 17:02:06.366  1671  1671 F DEBUG   : Revision: '0'
08-04 17:02:06.366  1671  1671 F DEBUG   : ABI: 'arm'
```

3）`dump_thread_info`对应

```c++
08-04 17:02:06.366  1671  1671 F DEBUG   : pid: 361, tid: 361, name: demo  >>> /system/bin/demo <<<
```

4）`dump_signal_info`对应

```c++
08-04 17:02:06.366  1671  1671 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xa
```

5）`dump_probable_cause`对应

```c++
08-04 17:02:06.366  1671  1671 F DEBUG   : Cause: null pointer dereference
```

6）`dump_registers`对应

```c++
08-04 17:02:06.366  1671  1671 F DEBUG   :     r0 0000000a  r1 00000061  r2 0538ad06  r3 fff162b0
08-04 17:02:06.366  1671  1671 F DEBUG   :     r4 b91d04ed  r5 b91dde08  r6 00000004  r7 fff1639c
08-04 17:02:06.366  1671  1671 F DEBUG   :     r8 00000000  r9 00000000  sl 00000000  fp fff1638c
08-04 17:02:06.367  1671  1671 F DEBUG   :     ip fff1639c  sp fff16308  lr eb97f0eb  pc b91d0342  cpsr 20010030
```

7）`log_backtrace`对应

```c++
08-04 17:02:06.373  1671  1671 F DEBUG   :
08-04 17:02:06.373  1671  1671 F DEBUG   : backtrace:
08-04 17:02:06.373  1671  1671 F DEBUG   :     #00 pc 00006342  /system/bin/demo (main+489)
08-04 17:02:06.373  1671  1671 F DEBUG   :     #01 pc 0007669d  /system/lib/libc.so (__libc_init+48)
08-04 17:02:06.373  1671  1671 F DEBUG   :     #02 pc 000024d8  /system/bin/demo (_start_main+88)
```



### 1.7.3activity_manager_notify

通知`AMS`做进一步的堆栈处理

#### 1.7.3.1客户端

```c++
//system/core/debuggerd/crash_dump.cpp
static bool activity_manager_notify(pid_t pid, int signal, const std::string& amfd_data) {
  //建立socket连接，连接本机socket路径为/data/system/ndebugsocket
  android::base::unique_fd amfd(socket_local_client(
      "/data/system/ndebugsocket", ANDROID_SOCKET_NAMESPACE_FILESYSTEM, SOCK_STREAM));

  struct timeval tv = {
    .tv_sec = 1,
    .tv_usec = 0,
  };
  //设置发送超时时间为1s
  if (setsockopt(amfd.get(), SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv)) == -1) {
    PLOG(ERROR) << "failed to set send timeout on activity manager socket";
    return false;
  }
  //设置接收超时时间为1s
  tv.tv_sec = 3;  // 3 seconds on handshake read
  if (setsockopt(amfd.get(), SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv)) == -1) {
    PLOG(ERROR) << "failed to set receive timeout on activity manager socket";
    return false;
  }

  //下面传输的是crash进程的pid和signal
  //这里将主机字节序转化成网络字节序
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
  //amfd_data就是先前crash进程的所有打印堆栈内容，就是0NE简介中的打印
  if (!android::base::WriteFully(amfd, amfd_data.c_str(), amfd_data.size() + 1)) {
    PLOG(ERROR) << "AM data write failed";
    return false;
  }

  //最终回收到一个ack，这里服务端没有设置任何值，所以客户端实际上收到的是0x0
  char ack;
  android::base::ReadFully(amfd, &ack, 1);
  return true;
}
```

> 关于这里为什么有`pid`和`signal`之后还需要重新拼接为网络字节序
>
> 网络协议指定了字节序，因此异构计算机系统能够交换协议信息而不会被字节序所混淆。
>
> TCP/IP协议栈使用大端字节序。**应用程序交换格式化数据时，字节序问题就会出现**。对于TCP/IP，地址用网络字节序来表示，所以应用程序有时需要在处理器的字节序与网络字节序之间转换它们。例如，以一种易读的形式打印一个地址时，这种转换很常见。

#### 1.7.3.2服务端

服务端即为AMS服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void startObservingNativeCrashes() {
    final NativeCrashListener ncl = new NativeCrashListener(this);
    ncl.start();
}
```

`NativeCrashListener`的`Thread`启动

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

`consumeNativeCrashData`处理来自客户端的请求

```java
//frameworks/base/services/core/java/com/android/server/am/NativeCrashListener.java
void consumeNativeCrashData(FileDescriptor fd) {
    final byte[] buf = new byte[4096];
    final ByteArrayOutputStream os = new ByteArrayOutputStream(4096);

    try {
        StructTimeval timeout = StructTimeval.fromMillis(SOCKET_TIMEOUT_MILLIS);
        Os.setsockoptTimeval(fd, SOL_SOCKET, SO_RCVTIMEO, timeout);
        Os.setsockoptTimeval(fd, SOL_SOCKET, SO_SNDTIMEO, timeout);
        //这里的buf实际上就是pid和signal的拼接，本文是0x169 0xB 对应正好pid为361和signal为11
        int headerBytes = readExactly(fd, buf, 0, 8);
        int pid = unpackInt(buf, 0);
        int signal = unpackInt(buf, 4);

        // now the text of the dump
        if (pid > 0) {
            final ProcessRecord pr;
            synchronized (mAm.mPidsSelfLocked) {
                pr = mAm.mPidsSelfLocked.get(pid);
            }
            //因为这个是自定义的native，没有涉及到system_server进程，所以不会找到对应的pr
            if (pr != null) {
                ...
            } else {
                Slog.w(TAG, "Couldn't find ProcessRecord for pid " + pid);
            }
        } else {
            Slog.e(TAG, "Bogus pid!");
        }
    } catch (Exception e) {
        Slog.e(TAG, "Exception dealing with report", e);
    }
}
```

这里最终会打印`Couldn't find ProcessRecord for pid 361`

### 1.7.4tombstoned_notify_completion

通知`tombstoned`服务端，将`tombstone_xxx`文件生成

#### 1.7.4.1客户端

这里发送了一个包，类型为`kCompletedDump`

```c++
//system/core/debuggerd/tombstoned/tombstoned_client.cpp
bool tombstoned_notify_completion(int tombstoned_socket) {
  TombstonedCrashPacket packet = {};
  packet.packet_type = CrashPacketType::kCompletedDump;
  if (TEMP_FAILURE_RETRY(write(tombstoned_socket, &packet, sizeof(packet))) != sizeof(packet)) {
    return false;
  }
  return true;
}
```



#### 1.7.4.2服务端

```c++
static void crash_completed_cb(evutil_socket_t sockfd, short ev, void* arg) {
  ...
  //之前在这里阻塞，现在可以接受到packet了    
  rc = TEMP_FAILURE_RETRY(read(sockfd, &request, sizeof(request)));
  //解析packet的合法性
  if (request.packet_type != CrashPacketType::kCompletedDump) {
    LOG(WARNING) << "unexpected crash packet type, expected kCompletedDump, received "
                 << uint32_t(request.packet_type);
    goto fail;
  }
  //这里这一步最关键，通过之前保存的output_fd文件句柄，通过linkat将tombstone文件生成
  //这里面的tombstone_path就是对应的/data/tombstones/tombstone_xxx
  if (crash->crash_tombstone_fd != -1) {
    std::string fd_path = StringPrintf("/proc/self/fd/%d", crash->crash_tombstone_fd.get());
    std::string tombstone_path = CrashQueue::for_crash(crash)->get_next_artifact_path();
  }

  rc = linkat(AT_FDCWD, fd_path.c_str(), AT_FDCWD, tombstone_path.c_str(), AT_SYMLINK_FOLLOW);
  //成功之后会打印这句话
  LOG(ERROR) << "Tombstone written to: " << tombstone_path;
  ...
}
```



# 2关于一些细节

1)堆栈信息是怎么打印出来的

堆栈信息实际上是最后`tombstoned_notify_completion`去通知服务端，让其将将`tombstone_xxx`文件生成。对应`tombstone`也会出现对应的`tombstone_xxx `，里面有更加完整的堆栈信息

2)这个堆栈信息是不是每次都会打印，什么情况不再打印

如果拦截堆栈信息，即自定义`signal`信号处理就不会打印`signal`信息。但是如果使用`sigcation`之后，不做拦截，继续调用就会继续打印

```c
#include <signal.h>
struct sigaction newact, oldact;
void sig_func(int signo, siginfo_t *info, void *context)
{
    //handler signal SIGINT
    ...
    //continue signal SIGINT,添加这句可以继续调用打印
    sigaction(SIGINT, &oldact, NULL);
}

void func(){
  
  newact.sa_handler = sig_func;
  sigemptyset(&newact.sa_mask);
  newact.sa_flags = 0;
  newact.sa_flags |= SA_SIGINFO;

  sigaction(SIGINT, &newact, &oldact);
}
```



# 3总结

1）**NE进程**句柄之间的关系

{{< image classes="fancybox center fig-100" src="/NE/NE_1.png" thumbnail="/NE/NE_1.png" title="">}}

2）**NE进程**的调用流程

{{< image classes="fancybox center fig-100" src="/NE/NE_4.png" thumbnail="/NE/NE_4.png" title="">}}

# 参考

[[1] 内核工匠, Tombstone原理分析, 2021.](https://blog.csdn.net/feelabclihu/article/details/113011145)

[[2] 内核工匠, Android内存异常机制(用户空间)_NE, 2020.](https://blog.csdn.net/feelabclihu/article/details/107724651)

[[3] 内核工匠, Android内存异常机制(用户空间)_JE, 2020.](https://blog.csdn.net/feelabclihu/article/details/107572844?spm=1001.2014.3001.5502)

[[4] 袁辉辉,  理解Native Crash处理流程, 2016.](http://gityuan.com/2016/06/25/android-native-crash/)

[[5] 袁辉辉, 理解Android Crash处理流程, 2016.](http://gityuan.com/2016/06/24/app-crash/)

[[6] 袁辉辉, 解读Java进程的Trace文件, 2016.](http://gityuan.com/2016/11/26/art-trace/)

[[7] 袁辉辉, Native进程之Trace原理, 2016.](http://gityuan.com/2016/11/27/native-traces/)

[[8] onITLoad, PRCTL - Linux手册页, 2020.](https://www.onitroad.com/jc/linux/man-pages/linux/man2/prctl.2.html)

[[9] liuanhf, linux daemon进程为何 fork 两次, 2020.](https://blog.51cto.com/liuanhf/2565065)

[[10] man7.org , ptrace(2) — Linux manual page, 2020.](https://man7.org/linux/man-pages/man2/ptrace.2.html)

[[11] gitbook.net, linkat()函数 Unix/Linux, 2020.](http://gitbook.net/unix_system_calls/linkat.html)

[[12] 小林code, TCP 三次握手与四次挥手面试题, 2022.](https://xiaolincoding.com/network/3_tcp/tcp_interview.html#%E4%B8%BA%E4%BB%80%E4%B9%88%E6%98%AF%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B-%E4%B8%8D%E6%98%AF%E4%B8%A4%E6%AC%A1%E3%80%81%E5%9B%9B%E6%AC%A1)

[[13] 皇甫懿, linux的SA_RESTART信号, 2019.](https://blog.csdn.net/weixin_43743847/article/details/90299204)

[[14] Spring__Rider, 函数link、linkat、unlink、unlinkat和remove, 2018.](https://blog.csdn.net/weixin_43743847/article/details/90299204)

[[15] xinjing_wangtao, 高性能网络编程(4)--TCP连接的关闭 (B), 2016.](https://blog.csdn.net/wangtaomtk/article/details/52489390)

