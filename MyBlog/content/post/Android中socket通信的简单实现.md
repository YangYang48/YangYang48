---
title: "socket简单实现（Java版）"
date: 2022-06-04
thumbnailImagePosition: left
thumbnailImage: socket/socket_thumb.jpg
coverImage: socket/socket_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- socket
- 2022
- June
tags:
- Android
- TCP
- UDP
showSocial: false
---
Android中除了Binder之外，还有其他的跨进程通信。socket就是其中之一，多用于进程的启动还有日志等模块中的通信，是Android系统不可缺少的一部分。

<!--more-->
socket简单实现（Java版）

# 1socket

专业术语来讲是套接字。`socket`的英文翻译为插座。把插座的含义引申开来，可以认为是一端接电源插头，另外插座上面的槽，可以提供多个其他设备使用。这里的电源插头端可以理解为是服务端，能够提供客户端交互的一端，插座上的槽可以理解为是客户端，能够尝试与服务端通信的一端。一旦通过`connect`连接(通电)起来就把的通信两端通过`socket`连接起来了。

> `TCP`和`UDP`
>
> 解释`TCP`和`UDP`之前，先阐述一下`OSI`模型
>
> 1. **物理层(`Physical`)**：单位是`bit比特`
>
>    设备之间的数据通信提供传输媒体及互连设备，为数据传输提供可靠的 环境。可以理解为网络传输的物理媒体部分，比如**网卡，网线，集线器，中继器，调制解调器**等。 在这一层，数据还没有被组织，仅作为原始的位流或电气电压处理
>
> 2. **数据链路层(`Datalink`)**：单位是`帧`
>
>    可以理解为数据通道，主要功能是如何在不可靠的物理线路上进行 数据的可靠传递，改层作用包括：物理地址寻址，数据的成帧，流量控制，数据检错以及重发等。 另外这个**数据链路指的是**：物理层要为终端设备间的数据通信提供传输媒体及其连接。媒体是 长期的，连接是有生存期的。在连接生存期内，收发两端可以进行不等的一次或多次数据通信。 每次通信都要经过建立通信联络和拆除通信联络两过程。这种建立起来的**数据收发关系**~ 该层的设备有：**网卡，网桥，网路交换机**
>
> 3. **网络层(`Network`)**：单位是`数据包`
>
>    主要功能是将网络地址翻译成对应的物理地址，并决定如何将数据从发 送方路由到接收方，所谓的路由与寻径：一台终端可能需要与多台终端通信，这样就产生的了 把任意两台终端设备数据链接起来的问题。简单点说就是：建立网络连接和为上层提供服务！ 该层的设备有：**路由**。另外`IP`协议就在这一层。
>
> 4. **传输层(`Transport`)**：单位是`数据段`
>
>    向上面的应用层提供通信服务，面向通信部分的最高层，同时也是 用户功能中的最低层。接收会话层数据，在必要时将数据进行分割，并将这些数据交给网络 层，并且保证这些数据段有效的到达对端。而这层有两个很重要 的协议就是：**`TCP`传输控制协议**与**`UDP`用户数据报协议**
>
> 5. **会话层(`Session`)**
>
>    负责在网络中的两节点之间建立、维持和终止通信。建立通信链接， 保持会话过程通信链接的畅通，同步两个节点之间的对话，决定通信是否被中断以及通信中断时 决定从何处重新发送，即不同机器上的用户之间会话的建立及管理！
>
> 6. **表示层(`Presentation`)**
>
>    对来自应用层的命令和数据进行解释，对各种语法赋予相应 的含义，并按照一定的格式传送给会话层。其主要功能是"处理用户信息的表示问题，如编码、 数据格式转换和加密解密，压缩解压缩"等
>
> 7. **应用层(`Application`)**
>
>    `OSI`参考模型的最高层，为用户的应用程序提供网络服务。 它在其他6层工作的基础上，负责完成网络中应用程序与网络操作系统之间的联系，建立与结束使用者之间的联系，并完成网络用户提出的各种网络服务及应用所需的监督、管理和服务等各种协议。此外，该层还负责协调各个应用程序间的工作。应用层为用户提供的服务和协议有：文件服务、目录服务、文件传输服务（`FTP`）、远程登录服务（`Telnet`）、电子邮件服务（`E-mail`）、打印服务、安全服务、网络管理服务、数据库服务等。

# 2Android中socket通信的简单实现(TCP)

考虑到`Android`中的`socket`实现，会被网络请求的框架封装起来，比如`Okhttp`，`Retrofit`等。所以下面所示都是直接使用`socket API`。

这里先展示`TCP`传输协议的`socket`。

## 2.1java中的socket接口

这要是需要认识两个类，一个是`ServerSocket.java`，另一个是`Socket.java`，这两个是对外提供的`API`接口。

## 2.2 ServerSocket构造

{{< image classes="fancybox center fig-100" src="/socket/socket_3.png" thumbnail="/socket/socket_3.png" title="">}}

1. 有参构造方法都会使服务器与特定端口绑定，该端口由参数`port`指定。端口被占用或是没有以超级用户身份运行服务器程序(比如绑定到`1-1023`的端口)，会抛出`BindException`。
2. 有参构造里面还有一个`backlog`，用于**设定客户连接请求队列的长度**。服务端进程运行时，可同时监听到多个客户的连接请求。这里用于客户端请求连接服务端的队列，默认为`50`。

```java
//服务端
ServerSocket serverSocket = null;
public final int port = 2022;
try {
    InetAddress addr = InetAddress.getLocalHost();
    System.out.println("local host:"+addr);
    serverSocket = new ServerSocket(port);
    Log.d(TAG,"->>>serverSocket success");
} catch (IOException e) {
    e.printStackTrace();
}
```



## 2.3 accept

一种阻塞的方法，如果没有客户端对其服务端有连接，那么就一直等待。

`ServerSocket`的`accept`方法从连接请求队列中取出一个客户的连接请求，然后创建与客户连接的`Socket`对象，并将它返回。如果队列中没有连接请求，`accept`方法就会一直等待，直到接收到了连接请求才返回。

```java
//服务端
//考虑到不只有一个客户端连接，所以需要一个循环
try {
    Socket socket = null;
    //等待连接，每建立一个连接，就新建一个线程
    while(true){
        //等待一个客户端的连接，在连接之前，此方法是阻塞的
        socket = serverSocket.accept();
        Log.d(TAG, "->>>connect to"+socket.getInetAddress()+":"+socket.getLocalPort());
    }
} catch (IOException e) {
    e.printStackTrace();
}
```



## 2.4 Socket初始化

{{< image classes="fancybox center fig-100" src="/socket/socket_5.png" thumbnail="/socket/socket_5.png" title="">}}



无参构造可设置超时时间。当客户端Socket构造方法与服务器建立连接时，需要等待一段时间，默认会一直等待下去，直到连接成功，或出现异常。

有参省略了无参的步骤，不需要设置远端地址和连接操作。

> `IP`地址获取举例
>
> 返回本地主机的`IP`地址
>
> ```java
> InetAddress addr1 = InetAddress.getLocalHost();
> ```
>
> 返回代表 "36.152.44.95"的` IP`地址，这个实际上是百度的`IP`
>
> ```java
> InetAddress addr2 = InetAddress.getByName("36.152.44.95");
> ```
>
> 返回域名为"yangyang48.github.io"的域名
>
> ```java
> InetAddress addr3 = InetAddress.getByName("yangyang48.github.io");
> String name = address.getHostName();
> String ip = address.getHostAddress();
> //日志打印yangyang48.github.io---185.199.109.153
> Log.d(TAG, name + "---" + ip);
> ```
>
> **注意点**：`addr3`这个例子不能放在`UI`子线程中，可以放在子线程中。因为输入网址是一个耗时操作，主线程会因此异常。
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_6.png" thumbnail="/socket/socket_6.png" title="">}}

```java
//客户端
public static String IP_ADDRESS = "";
public static int PORT = 2022;

IP_ADDRESS = tv_adress.getText().toString();
//这里的message为自己手动输入的字符串
new ConnectionThread(message).start();

//新建一个子线程，实现socket通信
class ConnectionThread extends Thread {
    String message = null;

    public ConnectionThread(String msg) {
        Log.d(TAG, "->>>ConnectionThread|msg = " + msg);
        message = msg;
    }

    @Override
    public void run() {
        if (soc == null) {
            try {
                Log.d(TAG,"->>>new socket");
                if ("".equals(IP_ADDRESS)) {
                    return;
                }
                //这个是阻塞的方法，连接成功后才会执行后面的语句
                soc = new Socket(IP_ADDRESS, PORT);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

除上述之外，还有一些获取`Socket`方法的信息

| 方法                    | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `getInetAddress`        | 获得连接的远程服务器的`IP`地址                               |
| `etRemoteSocketAddress` | 获取连接的远程服务器地址                                     |
| `getLocalSocketAddress` | 获取本地绑定的地址                                           |
| `getPort`               | 获得远程服务器的端口                                         |
| `getLocalAddress`       | 获得客户本地的`IP`地址                                       |
| `getLocalPort`          | 获得客户本地的端口                                           |
| `getInputStream`        | 获得输入流。如果Socket还没有连接，或者已经关闭，或者已经通过`shutdownInput`方法关闭输入流，那么此方法会抛出`IOException` |
| `getOutputStream`       | 获得输出流。如果Socket还没有连接，或者已经关闭，或者已经通过`shutdownOutput`方法关闭输出流，那么此方法会抛出`IOException` |



## 2.5获取输入输出流

因为存在服务端具有同时为多个客户提供服务的能力，所以每一次服务端都新连接都需要起**一个线程**来满足应对多个客户端。

```java
//服务端
try {
    Socket socket = null;
    while(true){
        socket = serverSocket.accept();
        //每次连接都开启一个线程，最好使用JDK的线程池
        new ConnectThread(socket).start();
    }
} catch (IOException e) {
    e.printStackTrace();
}

//向客户端发送信息
class ConnectThread extends Thread{
    Socket socket = null;
    //socket变量注入
    public ConnectThread(Socket socket){
        super();
        this.socket = socket;
    }

    @Override
    public void run(){
        try {
            //这里通过socket来获取二进制的输入输出流用于传递数据
            DataInputStream dis = new DataInputStream(socket.getInputStream());
            DataOutputStream dos = new DataOutputStream(socket.getOutputStream());
            while(true){
                i++;
                String msgRecv = dis.readUTF();
                Log.d(TAG, "->>>msg from client: " + msgRecv);
                //服务端的消息，只是在客户端的基本上加了一个符号
                dos.writeUTF(msgRecv + i);
                dos.flush();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

跟上面的服务端的流程类似，客户端也有相似的流程

```java
//客户端
public static String IP_ADDRESS = "";
public static int PORT = 2022;
String messageRecv = null;

IP_ADDRESS = tv_adress.getText().toString();
//这里的message为自己手动输入的字符串
new ConnectionThread(message).start();

//新建一个子线程，实现socket通信
class ConnectionThread extends Thread {
    @Override
    public void run() {
        if (soc == null) {
            try {
                soc = new Socket(IP_ADDRESS, PORT);
                //这里通过socket来获取二进制的输入输出流用于传递数据
                dis = new DataInputStream(soc.getInputStream());
                dos = new DataOutputStream(soc.getOutputStream());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        try {
            //客户端发送消息
            dos.writeUTF(message);
            dos.flush();
            //客户端接收消息
            messageRecv = dis.readUTF();//如果没有收到数据，会阻塞
            Message msg = new Message();
            Bundle b = new Bundle();
            b.putString("data", messageRecv);
            msg.setData(b);
            handler.sendMessage(msg);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

上述`socket`通信传递的信息，主要是通过`DataOutputStream`和`DataInputStream`来传递，本质上还是用于数据的收发。

`DataOutputStream`数据输出流允许应用程序将基本`Java`数据类型写到基础输出流中，而`DataInputStream`数据输入流允许应用程序以机器无关的方式从底层输入流中读取基本的`Java`类型。

{{< image classes="fancybox center fig-100" src="/socket/socket_2.png" thumbnail="/socket/socket_2.png" title="">}}



> **demo中的注意点**
>
> 1.`AndroidManifest.xml`中需要添加`<uses-permission android:name="android.permission.INTERNET" />`
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_1.png" thumbnail="/socket/socket_1.png" title="">}}
>
> 不添加上述权限，会影响下面检查权限时错误，在启动socket服务的时候会失败。
>
> ```text
> java.net.SocketException: socket failed: EACCES (Permission denied)
> ```
>
> 
>
> 2.服务端和客户端的端口号需要相同，且需要大于`1023`
>
> 即使`IP`相同。端口号不同也是会正常连接。包括正常的`Socket`初始化，但是去读写获取二进制流的时候，会读写错误。
>
> 端口号小于`1024`，可能需要设备的超级权限，默认设备`user`版本是无权限的。
>
> {{< image classes="fancybox center fig-100" src="/socket/socket_4.png" thumbnail="/socket/socket_4.png" title="">}}

# 3Android中socket通信的简单实现(UDP)

上面的`TCP`协议的`Socket`学习之后，继续来学习`UDP`协议的`Socket`。

进行数据传输时，首先需要将要传输的数据定义成数据报(`Datagram`)，在数据报中指明数据所要达到的`Socket`(主机地址和端口号)，然后再将数据报发送出去。

除上述之外，还有一些获取`DatagramPacket`方法的信息

| 方法                                          | 含义                                                 |
| --------------------------------------------- | ---------------------------------------------------- |
| `DatagramPacket()`                            | 构造数据报套接字并将其绑定到本地主机上任何可用的端口 |
| `DatagramPacket(int port)`                    | 创建数据报套接字并将其绑定到本地主机上的指定端口。   |
| `DatagramPacket(int port, InetAddress laddr)` | 创建数据报套接字，将其绑定到指定的本地地址。         |
| `getInetAddress`                              | 返回此套接字连接的地址                               |
| `getPort`                                     | 获得此套接字的端口                                   |
| `receive`                                     | 从此套接字接收数据报包                               |
| `send`                                        | 从此套接字发送数据报包                               |
| `close`                                       | 关闭此数据报套接字                                   |

## 3.1创建服务端

1. 创建服务器端`DatagramSocket`，指定端口号
2. 创建数据报，用于接收客户端发送的数据
3. 接收客户端发送的数据
4. 读取数据

```java
public void StartService() throws IOException {
    //1.创建服务器端DatagramSocket，指定端口号
    InetAddress addr = InetAddress.getLocalHost();
    datagramSocket = new DatagramSocket(PORT, addr);
    //2.创建数据报，用于接收客户端发送的数据
    byte[] data = new byte[1024];
    datagramPacket = new DatagramPacket(data, data.length);
    Log.d(TAG, "->>>UDP server is start ,wait client connected... ");
    //3.接收客户端发送的数据
    //receive方法是阻塞方法
    datagramSocket.receive(datagramPacket);
    //4.读取数据
    String info = new String(data, 0, datagramPacket.getLength());
    Log.d(TAG, "->>>UDP server : info = " + info);

    //响应客户端
    //1.定于客户端地址、端口号、数据
    InetAddress address = datagramPacket.getAddress();
    int port = datagramPacket.getPort();
    byte[] data2 = "hello".getBytes();
    //2.创建数据报，包含响应的数据信息
    DatagramPacket packet2 = new DatagramPacket(data2, data2.length, address, port);
    datagramSocket.send(packet2);
    //4.关闭资源
    //datagramSocket.close();
}
```



## 3.2创建客户端

1. 创建服务器端`DatagramSocket`，指定端口号
2. 创建数据报，包含发送的数据信息
3. 创建`DatagramSocket`对象
4. 向服务器端发送数据报

```java
public void StartClient() throws IOException {
    //1.创建服务器端DatagramSocket，指定端口号
    addr = InetAddress.getLocalHost();
    String IP_ADDRESS = addr.toString().split("/")[1];
    Log.d(TAG, "->>>addr = " + addr + " , IP_ADDRESS = " + IP_ADDRESS);
    //2.创建数据报，包含发送的数据信息
    byte[] data = "username:yangyang48;passwd:2023-01-01".getBytes();
    datagramPacket = new DatagramPacket(data, data.length, addr, PORT);
    //3.创建DatagramSocket对象
    datagramSocket = new DatagramSocket();
    //4.向服务器端发送数据报
    datagramSocket.send(datagramPacket);

    //接收服务端信息
    //1.创建数据报，用于接收服务器端响应的数据
    byte[] data2 = new byte[1024];
    DatagramPacket packet2 = new DatagramPacket(data2, data2.length);
    //2.接收服务器响应的数据
    datagramSocket.receive(packet2);
    //3.读取数据
    String reply = new String(data2, 0, packet2.getLength());
    Log.d(TAG, "->>>UDP client reply = " + reply);
    //4.关闭资源
    datagramSocket.close();
}
```

> 特别注意
>
> 不管是客户端还是服务端，如果里面有一个方法是阻塞的，是不能够放入`UI`主线程的，需要放到子线程中。类似出现下面错误
>
> ```text
> Caused by: android.os.NetworkOnMainThreadException
>             at android.os.StrictMode$AndroidBlockGuardPolicy.onNetwork(StrictMode.java:1145)
>             at java.net.InetAddress.lookupHostByName(InetAddress.java:385)
>             at java.net.InetAddress.getAllByNameImpl(InetAddress.java:236)
>             at java.net.InetAddress.getAllByName(InetAddress.java:214)
> ```
>
> 其中，`getLocalHost`和`receive`方法都是阻塞的

# 4总结

不管是`TCP`还是`UDP`，最根本的差异是传输层协议本身的差别，`API`的差异并不是很大，只要记住常见的`API`调用就能应付大多数的场景。在上层使用`Socket`相对比较少，更多的是`native`层使用进程`bin`，动态库`so`的方式调用底层`socket`方法建立网络连接。

|             异同             |                            `TCP`                             |                            `UDP`                             |
| :--------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|       `IP`地址（相同）       | 为了实现网络种不同的终端之间的通信，每个中断都必须有一个唯一的`IP`地址标识 | 为了实现网络种不同的终端之间的通信，每个中断都必须有一个唯一的`IP`地址标识 |
|        端口号（相同）        |                  可使用的端口号是1024-65535                  |                  可使用的端口号是1024-65535                  |
| `TCP`协议与`UDP`协议（差异） |             `TCP`是可靠的链接，三次握手四次挥手              | 非连接的协议，`UDP`把每个消息段放在队列中，应用程序每次从队列中读一个消息段 |

# 5源码下载

源码下载，点击[这里](https://github.com/YangYang48/project/tree/master/MySocket)

# 参考

[[1] 盖了帽, 简单说一下DataInputStream和DataOutputStream这两个流, 2020.](https://blog.csdn.net/ApolloNing/article/details/110522089)

[[2] 沉沦者,android socket 学习及示例, 2021.](https://blog.csdn.net/qq_33782617/article/details/112719793)

[[3] c小旭, Android Socket通信简单实现, 2021.](https://blog.csdn.net/c19344881x/article/details/120455491?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-1-120455491-blog-112719793.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-1-120455491-blog-112719793.pc_relevant_default&utm_relevant_index=2)

[[4] feng海涛, Android中socket通信的简单实现, 2020.](https://www.jb51.net/article/184151.htm)

[[5] Sally-he, Java中Socket的用法--Spring MVC, 2017.](https://blog.csdn.net/qq_31059475/article/details/77089650)

[[6] yjy239, Android socket源码解析(一)socket的初始化原理, 2021.](https://www.jianshu.com/p/46efb4b102d9)

[[7] 桥头放牛娃, ServerSocket详解, 2018.](https://www.jianshu.com/p/665994c2e784)

[[8] 菜鸟教程, 7.6.1 Socket学习网络基础准备, 2015.](https://www.runoob.com/w3cnote/android-tutorial-socket-intro.html)



