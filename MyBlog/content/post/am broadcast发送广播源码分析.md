---
title: "am broadcast发送广播源码分析"
date: 2022-12-05
thumbnailImagePosition: left
thumbnailImage: AndroidTools/am_broadcast/am_broadcast_thumb.jpg
coverImage: AndroidTools/am_broadcast/am_broadcast_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- broadcast
- 2022
- December
tags:
- Android
- 源码
- 调试工具
- Binder
- AMS
showSocial: false
---

Android中广播可以adb调试发送，am broadcast，本文主要分析这个广播源码。

<!--more-->
# 用命令发送广播

这里不展开说明参数，具体可以参考[这里](https://yangyang48.github.io/2021/08/%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E6%90%9E%E5%AE%9A%E5%B9%BF%E6%92%AD/)解析。

```shell
$ adb shell am broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
$ adb shell cmd activity broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
```

> 例如`-a`指定`action`，`-d`指定`uri`，`-p`指定包名，`-n`指定组件名，`--es`指定`{key,value}`
>
> `--es` 表示使用字符串类型，参数 `--ei`表示int类型，参数` --ez `表示boolean类型

# 1am

`am`命令是个`shell`脚本，非`instrument`时调用`cmd`命令。其中调用的`ja`r路径是设备上的`/system/framework/am.jar`

```shell
#/frameworks/base/cmds/am/am
#!/system/bin/sh

if [ "$1" != "instrument" ] ; then
    cmd activity "$@"
else
    base=/system
    export CLASSPATH=$base/framework/am.jar
    #这里的app_process实际上对应的就是zygote，fork and exec一个进程，执行/system/bin com.android.commands.am.Am "$@"
    exec app_process $base/bin com.android.commands.am.Am "$@"
fi
```

可以简单的看到，当`am`脚本后面的参数不为`instrument`时候，实际上执行的命令是`cmd activity "$@"`，其中`"$@"`就是后面的参数集。

> 后续在`bp`文件中可以看到，`/system/bin com.android.commands.am.Am`实际是`src/**/*.java`文件中的调用
>
> ```groovy
> //frameworks/base/cmds/am/Android.bp
> java_binary {
>     name: "am",
>     wrapper: "am",
>     srcs: [
>         "src/**/*.java",
>         "proto/**/*.proto",
>     ],
>     proto: {
>         plugin: "javastream",
>     },
>     static_libs: ["libprotobuf-java-lite"],
> }
> ```
>
> 

```txt
vince:/ $ cmd activity
Activity manager (activity) commands:
  help
      Print this help text.
  start-activity [-D] [-N] [-W] [-P <FILE>] [--start-profiler <FILE>]
          [--sampling INTERVAL] [--streaming] [-R COUNT] [-S]
          [--track-allocation] [--user <USER_ID> | current] <INTENT>
      Start an Activity.  Options are:
      -D: enable debugging
      -N: enable native debugging
      -W: wait for launch to complete
---------------------------------------
  broadcast [--user <USER_ID> | all | current] <INTENT>
      Send a broadcast Intent.  Options are:
      --user <USER_ID> | all | current: Specify which user to send to; if not
          specified then send to all users.
      --receiver-permission <PERMISSION>: Require receiver to hold permission.
--------------------------------------- 
    [-f <FLAG>]
    [--grant-read-uri-permission] [--grant-write-uri-permission]
    [--grant-persistable-uri-permission] [--grant-prefix-uri-permission]
    [--debug-log-resolution] [--exclude-stopped-packages]
    [--include-stopped-packages]
    [--activity-brought-to-front] [--activity-clear-top]
    [--activity-clear-when-task-reset] [--activity-exclude-from-recents]
    [--activity-launched-from-history] [--activity-multiple-task]
    [--activity-no-animation] [--activity-no-history]
    [--activity-no-user-action] [--activity-previous-is-top]
    [--activity-reorder-to-front] [--activity-reset-task-if-needed]
    [--activity-single-top] [--activity-clear-task]
    [--activity-task-on-home]
    [--receiver-registered-only] [--receiver-replace-pending]
    [--receiver-foreground] [--receiver-no-abort]
    [--receiver-include-background]
    [--selector]
    [<URI> | <PACKAGE> | <COMPONENT>]
```

因为这里实际上是广播，对其他的命令做了不必要的省略。这里的`activity`就是服务，可以理解成类似`dumpsys`出来的众多服务之一，可以通过`cmd`命令来调用。

# 2cmd

不过这个`cmd`命令比较特殊，通常而言一般服务直接调用名字即可，不过这里需要通过`cmd`命令再来调用到服务。本质是前者直接调用是`C/C++`偏底层的服务，可以直接通过进程的方式直接使用，后者的话借助了`Android`的封装间接调用，这个属于`Java`偏上层的服务。

```c++
//frameworks/native/cmds/cmd/main.cpp
int main(int argc, char* const argv[]) {
    //忽略管道相关信号
    signal(SIGPIPE, SIG_IGN);

    std::vector<std::string_view> arguments;
    //这里的reserve是预留大小，并没有分配大小
    arguments.reserve(argc - 1);
    //这里的arguments就是除了cmd之外所有的args
    for (int i = 1; i < argc; ++i) {
        arguments.emplace_back(argv[i]);
    }
	//最终调用到cmdMain
    return cmdMain(arguments, android::aout, android::aerr, STDIN_FILENO, STDOUT_FILENO,
                   STDERR_FILENO, RunMode::kStandalone);
}   
```

> 1. **std::string_view属于C++ 17的特性**
>
>    std::string_view来获取一个字符串的视图，字符串视图并不真正的创建或者拷贝字符串，而只是拥有一个字符串的查看功能。std::string_view比std::string的性能要高很多，因为每个std::string都独自拥有一份字符串的拷贝，而std::string_view只是记录了自己对应的字符串的指针和偏移位置。当我们在只是查看字符串的函数中可以直接使用std::string_view来代替std::string。
>
>    可以把原始的**字符串**当作一条**马路**，而我们是在马路边的一个**房子**里，我们只能通过**房间**的窗户来观察外面的**马路**。这个房子就是`std::string_view`，你只能看到马路上的车和行人，但是你无法去修改他们，可以理解你对这个马路是**只读**的。
>
> 2. **关于resize()和reserve()区别**
>
>    打个比方：正在建造的一辆公交车，车里面可以设置40个座椅（reserve(40);），这是它的容量，但并不是说它里面就有了40个座椅，只能说明这部车内部空间大小可以放得下40张座椅而已。而车里面安装了40个座椅(resize(40);)，这个时候车里面才真正有了40个座椅，这些座椅就可以使用了

```c++
//frameworks/native/cmds/cmd/cmd.cpp
//这里需要记住C++风格的特点，返回不为0，说明是有错误的，只有返回0才是正确的
int cmdMain(const std::vector<std::string_view>& argv, TextOutput& outputLog, TextOutput& errorLog,
            int in, int out, int err, RunMode runMode) {
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool();
	//获取默认的servicemanager大管家
    sp<IServiceManager> sm = defaultServiceManager();
    if (runMode == RunMode::kStandalone) {
        fflush(stdout);
    }
    if (sm == nullptr) {
        ALOGW("Unable to get default service manager!");
        return 20;
    }

    int argc = argv.size();

    if (argc == 0) {
        return 20;
    }
	//罗列所有的服务，这里用到了checkService，这个非阻塞的服务查询
    if ((argc == 1) && (argv[0] == "-l")) {
        Vector<String16> services = sm->listServices();
        services.sort(sort_func);
        outputLog << "Currently running services:" << endl;

        for (size_t i=0; i<services.size(); i++) {
            sp<IBinder> service = sm->checkService(services[i]);
            if (service != nullptr) {
                outputLog << "  " << services[i] << endl;
            }
        }
        return 0;
    }

    bool waitForService = ((argc > 1) && (argv[0] == "-w"));
    int serviceIdx = (waitForService) ? 1 : 0;
    //这个默认serviceIdx为0，那么就是 const auto cmd = argv[0];即为activity
    const auto cmd = argv[serviceIdx];
	
    Vector<String16> args;
    String16 serviceName = String16(cmd.data(), cmd.size());
    //把参数添加到args中，args为broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
    for (int i = serviceIdx + 1; i < argc; i++) {
        args.add(String16(argv[i].data(), argv[i].size()));
    }
    sp<IBinder> service;
    if(waitForService) {
        service = sm->waitForService(serviceName);
    } else {
        //通过binder直接用sm大管家查询叫activity的服务
        service = sm->checkService(serviceName);
    }

    if (service == nullptr) {
        errorLog << "cmd: Can't find service: " << cmd << endl;
        return 20;
    }
	//通常cb都是回调，这里设置错误信息的回调
    sp<MyShellCallback> cb = new MyShellCallback(errorLog);
    sp<MyResultReceiver> result = new MyResultReceiver();

    // 这个同步命令，进行阻塞，直到有MyResultReceiver类的结果产生
    //通过代理发起ipc调用
    //其中service指的是上述BpBinder传递的activity的service，in是STDIN_FILENO，out是STDOUT_FILENO
    //err是STDERR_FILENO，cb是上述的回调
    status_t error = IBinder::shellCommand(service, in, out, err, args, cb, result);
    if (error < 0) {
        const char* errstr;
        switch (error) {
            case BAD_TYPE: errstr = "Bad type"; break;
            case FAILED_TRANSACTION: errstr = "Failed transaction"; break;
            case FDS_NOT_ALLOWED: errstr = "File descriptors not allowed"; break;
            case UNEXPECTED_NULL: errstr = "Unexpected null"; break;
            default: errstr = strerror(-error); break;
        }
        //如果shellCommand执行失败，会打印到这里
        if (runMode == RunMode::kStandalone) {
            ALOGW("Failure calling service %.*s: %s (%d)", static_cast<int>(cmd.size()), cmd.data(),
                  errstr, -error);
        }
        outputLog << "cmd: Failure calling service " << cmd << ": " << errstr << " (" << (-error)
                  << ")" << endl;
        return error;
    }

    cb->mActive = false;
    status_t res = result->waitForResult();
    return res;
}
```



上述的`shellCommand`执行时阻塞的，需要等待这个函数执行完成

```c++
//frameworks/native/libs/binder/Binder.cpp
status_t IBinder::shellCommand(const sp<IBinder>& target, int in, int out, int err,
    Vector<String16>& args, const sp<IShellCallback>& callback,
    const sp<IResultReceiver>& resultReceiver)
{
    Parcel send;
    Parcel reply;
    //in,out,error,分别为标准输入设备，标准输除设备，标准错误设备
    send.writeFileDescriptor(in);
    send.writeFileDescriptor(out);
    send.writeFileDescriptor(err);
    //使用Parcel方式序列化参数broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
    const size_t numArgs = args.size();
    send.writeInt32(numArgs);
    for (size_t i = 0; i < numArgs; i++) {
        send.writeString16(args[i]);
    }
    send.writeStrongBinder(callback != nullptr ? IInterface::asBinder(callback) : nullptr);
    send.writeStrongBinder(resultReceiver != nullptr ? IInterface::asBinder(resultReceiver) : nullptr);
    //与服务端建立通信，IPC调用，通过command值为SHELL_COMMAND_TRANSACTION查找
    return target->transact(SHELL_COMMAND_TRANSACTION, send, &reply);
}
```

`onTransact`

由于服务端`ActivityManagerService`没有`SHELL_COMMAND_TRANSACTION`命令，只能从继承的`AIDL`中的`Binder.java`去找

```java
//frameworks/base/core/java/android/os/Binder.java
protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply,
                             int flags) throws RemoteException {
    ...
    else if (code == SHELL_COMMAND_TRANSACTION) {
        ParcelFileDescriptor in = data.readFileDescriptor();
        ParcelFileDescriptor out = data.readFileDescriptor();
        ParcelFileDescriptor err = data.readFileDescriptor();
        String[] args = data.readStringArray();
        ShellCallback shellCallback = ShellCallback.CREATOR.createFromParcel(data);
        ResultReceiver resultReceiver = ResultReceiver.CREATOR.createFromParcel(data);
        try {
            if (out != null) {
                //正常执行回到这里，调用shellCommand
                shellCommand(in != null ? in.getFileDescriptor() : null,
                             out.getFileDescriptor(),
                             err != null ? err.getFileDescriptor() : out.getFileDescriptor(),
                             args, shellCallback, resultReceiver);
            }
        } finally {
            ...
        }
        return true;
    }
    return false;
}
```

`shellCommand`会继续调用`onShellCommand`

```java
//frameworks/base/core/java/android/os/Binder.java
public void shellCommand(@Nullable FileDescriptor in, @Nullable FileDescriptor out,
                         @Nullable FileDescriptor err,
                         @NonNull String[] args, @Nullable ShellCallback callback,
                         @NonNull ResultReceiver resultReceiver) throws RemoteException {
    onShellCommand(in, out, err, args, callback, resultReceiver);
}
```

调用到服务端的`ActivityManagerService`中的`onShellCommand`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@Override
public void onShellCommand(FileDescriptor in, FileDescriptor out,
                           FileDescriptor err, String[] args, ShellCallback callback,
                           ResultReceiver resultReceiver) {
    //这里的false指的是dumpsys中的函数，这里不dump关于AMS的信息
    //这里的new ActivityManagerShellCommand会初始化mInterface，mInterface就是this
    (new ActivityManagerShellCommand(this, false)).exec(
        this, in, out, err, args, callback, resultReceiver);
}
```

{{< image classes="fancybox center fig-100" src="/AndroidTools/am_broadcast/am_broadcast_1.png" thumbnail="/AndroidTools/am_broadcast/am_broadcast_1.png" title="">}}

由于类图的关系如上所示，所以调用`exec`实际上是抽象类`BasicShellCommandHandler`中的方法

```java
//frameworks/base/core/java/android/os/BasicShellCommandHandler.java
public int exec(Binder target, FileDescriptor in, FileDescriptor out, FileDescriptor err,
                String[] args) {
    String cmd;
    int start;
    if (args != null && args.length > 0) {
        //cmd实际上是broadcast
        cmd = args[0];
        start = 1;
    } else {
        cmd = null;
        start = 0;
    }
    //这个init用于初始化一些command信息，target是用于之后执行错误之后的一些错误打印
    init(target, in, out, err, args, start);
    mCmd = cmd;

    int res = -1;
    try {
        //这里传入的this是ActivityManagerService，所以onCommand是AMS中的
        res = onCommand(mCmd);
        if (DEBUG) Log.d(TAG, "Executed command " + mCmd + " on " + mTarget);
    } catch (Throwable e) {
        ...
    } finally {
        ...
    }
    return res;
}
```

`onCommand`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@Override
public int onCommand(String cmd) {
    if (cmd == null) {
        return handleDefaultCommands(cmd);
    }
    final PrintWriter pw = getOutPrintWriter();
    try {
        switch (cmd) {
            case "start":
            case "start-activity":
                return runStartActivity(pw);
			...
            //显然，会调用到这里
            case "broadcast":
                return runSendBroadcast(pw);
            ...
            default:
                return handleDefaultCommands(cmd);
}
    } catch (RemoteException e) {
        pw.println("Remote exception: " + e);
    }
    return -1;
}
```



# 3am broadcast

根据上面得知`am broadcast`通过`cmd`脚本，从`ServiceManager`大管家中找到名叫`"activity"`的服务，通过客服端`IPC`调用服务端`AMS`方法，参与执行命令的`cmd`为`broadcast`，最终会调用到`runSendBroadcast`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerShellCommand.java
//传入的pw，其实就是JDK中的工具，向文本输出流打印对象的格式化表示形式
int runSendBroadcast(PrintWriter pw) throws RemoteException {
    Intent intent;
    try {
        //[3.1]这里的makeIntent开始真正解析传入的args
        //UserHandle.USER_CURRENT用来指示当前活动用户的用户id
        intent = makeIntent(UserHandle.USER_CURRENT);
    } catch (URISyntaxException e) {
        throw new RuntimeException(e.getMessage(), e);
    }
    //Flags是intent的跳转类型，这里说明是通过shell来发送广播的
    intent.addFlags(Intent.FLAG_RECEIVER_FROM_SHELL);
    //这里的IntentReceiver是一个ActivityManagerShellCommand.java中的内部类
    //主要用于设置回调完成广播会有打印
    IntentReceiver receiver = new IntentReceiver(pw);
    //这里的mReceiverPermission为null
    String[] requiredPermissions = mReceiverPermission == null ? null
        : new String[] {mReceiverPermission};
    pw.println("Broadcasting: " + intent);
    pw.flush();
    //这里的Bundle依然是null
    Bundle bundle = mBroadcastOptions == null ? null : mBroadcastOptions.toBundle();
    //[3.2]mInterface就是初始化的ActivityManagerShellCommand注入的，为AMS
    mInterface.broadcastIntentWithFeature(null, null, intent, null, receiver, 0, null, null,
                                          requiredPermissions, android.app.AppOpsManager.OP_NONE, bundle, true, false, mUserId);
    //这里让当前线程等待，直到上述的操作完成，直到receiver回调performReceive
    receiver.waitForFinish();
    return 0;
}
```

## 3.1makeIntent

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerShellCommand.java
private Intent makeIntent(int defUser) throws URISyntaxException {

    mUserId = defUser;
	...
	//这里开始解析args，这里的this指的是ActivityManagerShellCommand
    //对于发送广播而言,后者的匿名内部类new Intent.CommandOptionHandler()的回调不会发生
    return Intent.parseCommandArgs(this, new Intent.CommandOptionHandler() {
        @Override
        public boolean handleOption(String opt, ShellCommand cmd) {
            if (opt.equals("-D")) {
                mStartFlags |= ActivityManager.START_FLAG_DEBUG;
            ...
            } else {
                return false;
            }
            return true;
        }
    });
}
```

`parseCommandArgs`

```java
//frameworks/base/core/java/android/content/Intent.java
@UnsupportedAppUsage
public static Intent parseCommandArgs(ShellCommand cmd, CommandOptionHandler optionHandler)
    throws URISyntaxException {
    Intent intent = new Intent();
    Intent baseIntent = intent;
    boolean hasIntentInfo = false;

    Uri data = null;
    String type = null;

    String opt;
    //对cmd进行遍历，然后对每一个符号进行解析
    //getNextArgRequired就是参数后面紧跟的值
    //最终不管是es，ei，ez其实都是存放到Bundle中的k-v键值对
    while ((opt=cmd.getNextOption()) != null) {
        switch (opt) {
            case "-a":
                //intent.setAction(android.intent.action.SENDLOVE);
                intent.setAction(cmd.getNextArgRequired());
                if (intent == baseIntent) {
                    hasIntentInfo = true;
                }
                break;
            case "-e":
            case "--es": {
                //key="love",value="爱你"
                String key = cmd.getNextArgRequired();
                String value = cmd.getNextArgRequired();
                intent.putExtra(key, value);
            }
                break;
            case "--ei": {
                //key="days",value="10000"
                String key = cmd.getNextArgRequired();
                String value = cmd.getNextArgRequired();
                intent.putExtra(key, Integer.decode(value));
            }
                break;
            case "--ez": {
                //key="reality",value="true"
                String key = cmd.getNextArgRequired();
                String value = cmd.getNextArgRequired().toLowerCase();
                boolean arg;
                if ("true".equals(value) || "t".equals(value)) {
                    arg = true;
                } else if ("false".equals(value) || "f".equals(value)) {
                    arg = false;
                } else {
                    try {
                        arg = Integer.decode(value) != 0;
                    } catch (NumberFormatException ex) {
                        throw new IllegalArgumentException("Invalid boolean value: " + value);
                    }
                }

                intent.putExtra(key, arg);
            }
                break;
            case "-n": {
                //str=com.example.broadcast/.MyTanabataReceiver
                String str = cmd.getNextArgRequired();
                ComponentName cn = ComponentName.unflattenFromString(str);
                if (cn == null)
                    throw new IllegalArgumentException("Bad component name: " + str);
                intent.setComponent(cn);
                //这里的intent和baseIntent是一致的
                if (intent == baseIntent) {
                    hasIntentInfo = true;
                }
            }
                break;
            ...
            default:
                if (optionHandler != null && optionHandler.handleOption(opt, cmd)) {
                    // Okay, caller handled this option.
                } else {
                    throw new IllegalArgumentException("Unknown option: " + opt);
                }
                break;
        }
    }
    //这里的data和type依然是null
    intent.setDataAndType(data, type);
	//这里有两个操作intent.replaceExtras((Bundle)null);和intent.replaceExtras(extras);
    //把它置空，又把它放进去，这里的目的跟baseIntent相关，用于Uri和extras的交换，这里没有uri所以流程比较简单
    if (baseIntent != null) {
        Bundle extras = intent.getExtras();
        intent.replaceExtras((Bundle)null);
        Bundle uriExtras = baseIntent.getExtras();
        baseIntent.replaceExtras((Bundle)null);
        if (intent.getAction() != null && baseIntent.getCategories() != null) {
            HashSet<String> cats = new HashSet<String>(baseIntent.getCategories());
            for (String c : cats) {
                baseIntent.removeCategory(c);
            }
        }
        intent.fillIn(baseIntent, Intent.FILL_IN_COMPONENT | Intent.FILL_IN_SELECTOR);
        if (extras == null) {
            extras = uriExtras;
        } else if (uriExtras != null) {
            uriExtras.putAll(extras);
            extras = uriExtras;
        }
        intent.replaceExtras(extras);
        hasIntentInfo = true;
    }

    if (!hasIntentInfo) throw new IllegalArgumentException("No intent supplied");
    return intent;
}
```

最终`makeIntent`的结果

{{< image classes="fancybox center fig-100" src="/AndroidTools/am_broadcast/am_broadcast_2.png" thumbnail="/AndroidTools/am_broadcast/am_broadcast_2.png" title="">}}

## 3.2broadcastIntentWithFeature

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public final int broadcastIntentWithFeature(IApplicationThread caller, String callingFeatureId,
                                            Intent intent, String resolvedType, IIntentReceiver resultTo,
                                            int resultCode, String resultData, Bundle resultExtras,
                                            String[] requiredPermissions, int appOp, Bundle bOptions,
                                            boolean serialized, boolean sticky, int userId) {
    //强制非隔离调用者，判断调用方是否有selinux权限
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
        //校验intent的正确性
        intent = verifyBroadcastLocked(intent);
		//获取client端caller进程record,如果对端进程已经被kill则直接抛出异常。这里传入的是null不处理
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        //作用是清空远程调用端的uid和pid，用当前本地进程的uid和pid替代
        final long origId = Binder.clearCallingIdentity();
        try {
            //这里就是真正发送广播，由于这里是标准流程，这里不展开
            return broadcastIntentLocked(callerApp,
                                         callerApp != null ? callerApp.info.packageName : null, 
                                         callingFeatureId, intent, resolvedType, resultTo, resultCode, 
                                         resultData, resultExtras, requiredPermissions, appOp, bOptions, 
                                         serialized, sticky, callingPid, callingUid, callingUid, callingPid, 
                                         userId);
        } finally {
            //作用是恢复远程调用端的uid和pid信息，正好是`clearCallingIdentity`的反过程
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```

`verifyBroadcastLocked`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final Intent verifyBroadcastLocked(Intent intent) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    int flags = intent.getFlags();
	//这里面指的是system系统还未完整引导完成，引导完成mProcessesReady为true
    if (!mProcessesReady) {
        if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT) != 0) {
            // This will be turned into a FLAG_RECEIVER_REGISTERED_ONLY later on if needed.
        } else if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
            Slog.e(TAG, "Attempt to launch receivers of broadcast intent " + intent
                   + " before boot completion");
            throw new IllegalStateException("Cannot broadcast before boot completed");
        }
    }
	//不能直接发送广播升级
    if ((flags&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0) {
        throw new IllegalArgumentException(
            "Can't use FLAG_RECEIVER_BOOT_UPGRADE here");
    }
	//我们的flag是这个
    if ((flags & Intent.FLAG_RECEIVER_FROM_SHELL) != 0) {
        //显而易见我们的广播发送必须是root权限或者是shell权限，不然没有权限发送
        switch (Binder.getCallingUid()) {
            case ROOT_UID:
            case SHELL_UID:
                break;
            default:
                Slog.w(TAG, "Removing FLAG_RECEIVER_FROM_SHELL because caller is UID "
                       + Binder.getCallingUid());
                intent.removeFlags(Intent.FLAG_RECEIVER_FROM_SHELL);
                break;
        }
    }

    return intent;
}
```

`clearCallingIdentity`和`restoreCallingIdentity`，关于这两个作用具体可以点击[这里](https://blog.csdn.net/fchyang/article/details/82345927)

```java
//frameworks/base/core/java/android/os/Binder.java
@CriticalNative
public static final native long clearCallingIdentity();

/* @see #clearCallingIdentity
*/
public static final native void restoreCallingIdentity(long token);
```

调用到native方法里面

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
int64_t IPCThreadState::clearCallingIdentity()
{
    // ignore mCallingSid for legacy reasons
    int64_t token = ((int64_t)mCallingUid<<32) | mCallingPid;
    clearCaller();
    return token;
}

void IPCThreadState::clearCaller()
{
    mCallingPid = getpid();
    mCallingSid = nullptr;  // expensive to lookup
    mCallingUid = getuid();
}

void IPCThreadState::restoreCallingIdentity(int64_t token)
{
    mCallingUid = (int)(token>>32);
    mCallingSid = nullptr;  // not enough data to restore
    mCallingPid = (int)token;
}
```

> 1. 线程A通过Binder远程调用线程B：则线程B的`IPCThreadState`中的`mCallingUid`和`mCallingPid`保存的就是线程A的`UID`和`PID`。这时在线程B中调用`Binder.getCallingPid()`和`Binder.getCallingUid()`方法便可获取线程A的`UID`和`PID`，然后利用`UID`和`PID`进行权限比对，判断线程A是否有权限调用线程B的某个方法。
> 2. 线程B通过`Binder`调用当前线程的某个组件：此时线程B是线程B某个组件的调用端，则`mCallingUid`和`mCallingPid`应该保存当前线程B的`PID`和`UID`，故需要调用`clearCallingIdentity()`方法完成这个功能。当线程B调用完某个组件，由于线程B仍然处于线程A的被调用端，因此`mCallingUid`和`mCallingPid`需要恢复成线程A的`UID`和`PID`，这是调用`restoreCallingIdentity()`即可完成。
>
> {{< image classes="fancybox center fig-100" src="/AndroidTools/am_broadcast/am_broadcast_3.png" thumbnail="/AndroidTools/am_broadcast/am_broadcast_3.png" title="">}}

# 4总结

总的来说，`am broadcast`还是比较常用的`shell`命令，通过一系列的操作，最终可以达到`Context.sendBroadcast`同等效果的广播。

```shell
$ adb shell am broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
```

以上面的`shell`命令为例

1. `am`命令会调用到`/frameworks/base/cmds/am`目录中的`am`脚本，并且在脚本里会调用`cmd`命令

   ```shell
   $ adb shell cmd activity broadcast -a android.intent.action.SENDLOVE -n com.example.broadcast/.MyTanabataReceiver –es “love” “爱你” –ei “days” 10000 –ez “reality” true
   ```

2. `cmd`命令会建立`Binder`通信，通过传入的参数为`activity`，从`ServiceManager`大管家中找到名叫`"activity"`的服务，接着调用`shellCommand`，通过`IPC`中的`command`序号`SHELL_COMMAND_TRANSACTION`

3. 在`AMS`中找到`Binder`通信的真实调用`onShellCommand`，然后通过命令的形式在基类执行`BasicShellCommandHandler`的`exec`方法，最终会在`AMS`中找到对应的实现方法`onCommand`，这个就是处理参数`broadcast`方式

4. 在`AMS`开始处理`broadcast`方式`runSendBroadcast`。主要有创建广播的`Intent`内容，并发送广播。发送广播完成之后，回调`receiver`的`performReceive`方法

   创建广播其实也是解析命令行的过程，这里解析之后，会被送到`AMS`中进行校验，包括`selinux`权限、广播时机和广播发送的`uid`等，最终通过标准流程`broadcastIntentLocked`实现发送广播。

{{< image classes="fancybox center fig-100" src="/AndroidTools/am_broadcast/am_broadcast_4.png" thumbnail="/AndroidTools/am_broadcast/am_broadcast_4.png" title="">}}

# 源码下载

[am broadcast源码](https://github.com/YangYang48/project/tree/master/am_broadcast)

