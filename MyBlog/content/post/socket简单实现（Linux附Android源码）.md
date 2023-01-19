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

为创建一个套接字，调用socket函数。

```c++
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```



{{< image classes="fancybox center fig-100" src="/socket/socket_7.png" thumbnail="/socket/socket_7.png" title="">}}

关于socket中的send和write的区别

> 对于linux来说,每个socket会有自己的send/receive buffer。调用write,只是说将用户进程的数据,拷贝到了内核的socket buffer里面,拷贝完之后,就没有write什么事了。内核自己会用自己的进程,调用TCP/IP协议栈,把用户进程的数据发出去。可以把内核的TCP/IP协议想象成另外一个进程,调用write的时候,只是把数据发给这个进程,这个进程会处理剩下的事情。

socket_write写到缓冲(本地[套接字](https://so.csdn.net/so/search?q=套接字&spm=1001.2101.3001.7020)缓冲) 实际有可能未发送出去

socket_send 已经从套接字缓冲发出





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