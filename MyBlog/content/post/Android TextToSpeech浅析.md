---
title: "Android TextToSpeech浅析"
date: 2023-02-14
thumbnailImagePosition: left
thumbnailImage: tts/tts_thumb.jpg
coverImage: tts/tts_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- tts
- 2023
- February
tags:
- 源码
- java
- audio
- audiotrack
showSocial: false
---

今天就顺便来拆解一下用到的核心，TextToSpeech功能(即tts)。

<!--more-->
前几天突然看到以前手机安装的App叫Instapaper，可以保存网页以便稍后阅读的服务，提供离线阅读功能。那年16年，我用这个App保存公众号文章，方便走路的时候边走边听。

以笔者的手机型号为例，红米K30，MUUI11.1.23版本。

tts功能可以在无障碍模式中找到，无障碍模式通常是专门给特别的人群使用的功能(但是现在好多人都用来淘宝刷任务金币或者其他操作等)。

可以看到笔者的无障碍模式中，除了自带的tts输出，还有一个产商定制的TalkBack的tts功能

{{< image classes="fancybox center fig-100" src="/tts/tts_5.jpg" thumbnail="/tts/tts_5.jpg" title="">}}

进到TalkBack设置里面，可以看到除了tts功能，还有其他辅助功能，类似一个工具箱。

{{< image classes="fancybox center fig-100" src="/tts/tts_6.jpg" thumbnail="/tts/tts_6.jpg" title="">}}

打开tts的ui内容，默认的引擎是系统引擎，**说明是小米产商自己开发的一套东西**。当然如果有需要的话，也可以类似讯飞，思必驰，云知声等一些语音解决方案的公司一样，可以自行开发一套新的，覆盖掉这个默认引擎。这里支持的语言主要是中文和英文。

{{< image classes="fancybox center fig-100" src="/tts/tts_7.jpg" thumbnail="/tts/tts_7.jpg" title="">}}

# 1简介

TextToSpeech 即文字转语音服务，是Android系统提供的原生接口服务，原生的tts引擎应用通过检测系统语言，用户可以下载对应语言的资源文件，达到播报指定语音的文字的能力。一般google原生时不支持中文的。

通过 TextToSpeech机制，任意App都可以方便地采用系统内置或第三方提供的TTS Engine进行播放铃声提示、语音提示的请求，Engine 可以由系统选择默认的provider来执行操作，也可由App具体指定偏好的目标Engine来完成。

# 2demo

```java
public class TtsTest implements TextToSpeech.OnInitListener{
    private TextToSpeech textToSpeech = null;
    private Context mContext = null;

    TtsTest(Context context)
    {
        mContext = context;
        textToSpeech = new TextToSpeech(mContext, this); // 参数Context,TextToSpeech.OnInitListener
        textToSpeech.setOnUtteranceProgressListener(new UtteranceProgressListener() {
            @Override
            public void onStart(String utteranceId) {
                //自定义回调函数的处理
            }

            @Override
            public void onDone(String utteranceId) {
                //自定义回调函数的处理
            }

            @Override
            public void onError(String utteranceId) {
                //自定义回调函数的处理
            }

            @Override
            public void onAudioAvailable(String utteranceId, byte[] audio) {
                super.onAudioAvailable(utteranceId, audio);
                //这里的audio是原始数据，可以进行音频数据处理，比如音频3A处理
            }
        });
    }
    /**
     * 初始化TextToSpeech引擎
     * status:SUCCESS或ERROR
     * setLanguage设置语言
     */
    @Override
    public void onInit(int status) {
        if (status == TextToSpeech.SUCCESS) {
            int result = textToSpeech.setLanguage(Locale.CHINA);
            if (result == TextToSpeech.LANG_MISSING_DATA
                    || result == TextToSpeech.LANG_NOT_SUPPORTED) {
                Toast.makeText(mContext, "数据丢失或不支持", Toast.LENGTH_SHORT).show();
            }
        }
    }

    public void speak() {
        if (textToSpeech != null && !textToSpeech.isSpeaking()) {
            //初始化配置
            Bundle bundle = new Bundle();
            bundle.putInt("streamType", 0);
            bundle.putInt("sessionId", 0);
            bundle.putString("utteranceId", null);
            bundle.putFloat("volume", 1.0f);
            bundle.putFloat("pan", 0.0f);
            
            textToSpeech.setPitch(0.0f);// 设置音调
            textToSpeech.speak("我是要播放的文字",
                    TextToSpeech.QUEUE_FLUSH, bundle, null);
        }
    }

     public void onStop() {
        textToSpeech.stop(); // 停止tts
        textToSpeech.shutdown(); // 关闭，释放资源
    }

}
```

这里App里面的步骤分成三步

1. 初始化tts服务和tts的引擎，初始化成功，会有onInit回调
2. 给tts设置一个监听，用于监听tts引擎处理的状态，可以是onStart，onDone，onError，还可以有原数数据的处理
3. 开始进行播报文字，这里需要tts最核心的引擎处理

# 3源码解析

为了看明白后续的源码，这里简单梳理了比较重要的几个类图，分别是`SpeechItem`、`AbstractSynthesisCallback`和`PlaybackQueueItem`。

{{< image classes="fancybox center fig-100" src="/tts/tts_1.png" thumbnail="/tts/tts_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/tts/tts_2.png" thumbnail="/tts/tts_2.png" title="">}}

{{< image classes="fancybox center fig-100" src="/tts/tts_3.png" thumbnail="/tts/tts_3.png" title="">}}

## 3.1init 绑定

首先从 `TextToSpeech()` 的实现入手，以了解在 TTS 播报之前，系统和 TTS Engine 之间做了什么准备工作。

### 3.1.1 initTTS

将按照如下顺序查找需要连接到哪个 Engine

1. 如果构造 TTS 接口的实例时指定了目标 Engine 的 package，那么首选连接到该 Engine
2. 反之，获取设备设置的 default Engine 并连接，设置来自于 TtsEngines 从系统设置数据 SettingsProvider 中 读取 TTS_DEFAULT_SYNTH 而来
3. 如果 default 不存在或者没有安装的话，从 TtsEngines 获取第一位的系统 Engine 并连接。第一位指的是从所有 TTS Service 实现 Engine 列表里获得第一个属于 system image 的 Engine

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeech.java
public TextToSpeech(Context context, OnInitListener listener) {
    this(context, listener, null);
}

private TextToSpeech( ... ) {
    ...
    initTts();
}
//这里直接是第二种方式，默认的系统引擎
private int initTts() {
    // Step 1: Try connecting to the engine that was requested.
    if (mRequestedEngine != null) {
        ...
    }

    // Step 2: Try connecting to the user's default engine.
    final String defaultEngine = getDefaultEngine();
    ...

    // Step 3: Try connecting to the highest ranked engine in the system.
    final String highestRanked = mEnginesHelper.getHighestRankedEngineName();
    ...

    dispatchOnInit(ERROR);
    return ERROR;
}
```



### 3.1.2连接

调用 connectToEngine()，其将依据调用来源来采用不同的 Connection内部实现去 connect()：

具体是直接获取名为 texttospeech 、管理 TTS Service 的系统服务 TextToSpeechManagerService 的接口代理并直接调用它的 createSession() 创建一个 session，同时暂存其指向的 ITextToSpeechSession 代理接口。

> 该 session 实际上还是 `AIDL` 机制，TTS 系统服务的内部会创建专用的 `TextToSpeechSessionConnection` 去 bind 和 cache Engine，这里不再赘述

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeech.java
private boolean connectToEngine(String engine) {
    Connection connection;
    if (mIsSystem) {
        connection = new SystemConnection();
    } else {
        connection = new DirectConnection();
    }
    //相当于直接连接绑定服务了，用于aidl通信
    boolean bound = connection.connect(engine);
    if (!bound) {
        return false;
    } else {
        mConnectingServiceConnection = connection;
        return true;
    }
}
```

### 3.1.3OnInit

connect 执行完毕并结果 OK 的话，还要暂存到 mConnectingServiceConnection，以在结束 TTS 需求的时候释放连接使用。并通过 dispatchOnInit() 传递 SUCCESS 给 Request App

实现很简单，将结果 Enum 回调给初始化传入的 OnInitListener 接口
如果连接失败的话，则调用 dispatchOnInit() 传递 ERROR

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeech.java
private class Connection implements ServiceConnection {
        private ITextToSpeechService mService;
        //异步方式
        private class SetupConnectionAsyncTask extends AsyncTask<Void, Void, Integer> {
            private final ComponentName mName;

            public SetupConnectionAsyncTask(ComponentName name) {
                mName = name;
            }

            @Override
            protected Integer doInBackground(Void... params) {
                 ...
            }

            @Override
            protected void onPostExecute(Integer result) {
                synchronized(mStartLock) {
                    if (mOnSetupConnectionAsyncTask == this) {
                        mOnSetupConnectionAsyncTask = null;
                    }
                    mEstablished = true;
                    //最终调用到dispatchOnInit
                    dispatchOnInit(result);
                }
            }
        }

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            synchronized(mStartLock) {
                ...
				//这个mService指代的是TextToSpeechService的代理binder
                mService = ITextToSpeechService.Stub.asInterface(service);
                mServiceConnection = Connection.this;
                mEstablished = false;
                //最终调用异步操作，进行回调
                mOnSetupConnectionAsyncTask = new SetupConnectionAsyncTask(name);
                mOnSetupConnectionAsyncTask.execute();
            }
        }
}
```

## 3.2 speak

### 3.2.1调用远端接口Action

首先将 speak() 对应的调用远程接口的操作封装为 Action 接口实例，并交给 init() 时暂存的已连接的 Connection 实例去调度。

```java

public int speak(final CharSequence text,
                 final int queueMode,
                 final Bundle params,
                 final String utteranceId) {
    //最终回调到这里，service就是Connection#runAction中的mService
    return runAction((ITextToSpeechService service) -> {
      ...
    }, ERROR, "speak");
}

private class Connection implements ServiceConnection {
    ...
    public <R> R runAction(Action<R> action, R errorResult, String method,
                           boolean reconnect, boolean onlyEstablishedConnection) {
        synchronized (mStartLock) {
            try {
				...
                 //直接执行run，传入的mService就是3.1中初始化的TextToSpeechService的代理binder
                return action.run(mService);
            } catch (RemoteException ex) {
                ...
                return errorResult;
            }
        }
    }
}
```

### 3.2.2 Uri区分

Action 的实际内容是先从 mUtterances Map 里查找目标文本是否有设置过本地的 audio 资源

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeech.java
public int speak(final CharSequence text, ... ) {
    //最终回调到这里，service就是Connection#runAction中的mService
    return runAction((ITextToSpeechService service) -> {
        Uri utteranceUri = mUtterances.get(text);
        if (utteranceUri != null) {
            //这里分成两种情况，直接播放音频，铃声闹钟等
            return service.playAudio(getCallerIdentity(), utteranceUri, queueMode,
                                     getParams(params), utteranceId);
        } else {
            //这里是tts转换
            return service.speak(getCallerIdentity(), text, queueMode, getParams(params),
                                 utteranceId);
        }
    }, ERROR, "speak");
}
```

## 3.4TextToSpeechService实现

### 3.4.1封装成SpeechItem

TextToSpeechService 内接收的实现是向内部的 SynthHandler 发送封装的 speak 或 playAudio 请求的 **SpeechItem**。

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeechService.java
private final ITextToSpeechService.Stub mBinder =
    new ITextToSpeechService.Stub() {
    @Override
    public int speak(
        IBinder caller,
        CharSequence text,
        int queueMode,
        Bundle params,
        String utteranceId) {
        SpeechItem item =
            new SynthesisSpeechItem(
            caller,
            Binder.getCallingUid(),
            Binder.getCallingPid(),
            params,
            utteranceId,
            text);
        return mSynthHandler.enqueueSpeechItem(queueMode, item);
    }

    @Override
    public int playAudio( ... ) {
        SpeechItem item =
            new AudioSpeechItem( ... );
        ...
    }
    ...
};
```

### 3.4.2异步handler

SynthHandler 绑定到 TextToSpeechService 初始化的时候启动的、名为 "SynthThread" 的 HandlerThread

1. speak 请求封装给 Handler 的是 SynthesisSpeechItem
2. playAudio 请求封装的是 AudioSpeechItem

SynthHandler 拿到 SpeechItem ，封装之后进一步 play 的操作 Message 给 Handler

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeechService.java
private class SynthHandler extends Handler {
    ...
    public int enqueueSpeechItem(int queueMode, final SpeechItem speechItem) {
        UtteranceProgressDispatcher utterenceProgress = null;
        if (speechItem instanceof UtteranceProgressDispatcher) {
            utterenceProgress = (UtteranceProgressDispatcher) speechItem;
        }
		//queueMode就是speak的时候参数
        if (queueMode == TextToSpeech.QUEUE_FLUSH) {
            stopForApp(speechItem.getCallerIdentity());
        }
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                if (setCurrentSpeechItem(speechItem)) {
                    //最终会调用到这里
                    speechItem.play();
                    removeCurrentSpeechItem();
                } else {
                    speechItem.stop();
                }
            }
        };
        //这里就是handler操作，相当于开启另一个线程来执行异步操作
        Message msg = Message.obtain(this, runnable);
        msg.obj = speechItem.getCallerIdentity();
        if (sendMessage(msg)) {
            return TextToSpeech.SUCCESS;
        }
    }
    ...
}
```

### 3.4.3playimpl

play() 具体是调用 playImpl() 继续。对于 SynthesisSpeechItem 来说，将初始化时创建的 SynthesisRequest 实例和 SynthesisCallback 实例（此处的实现是 PlaybackSynthesisCallback）收集和调用 onSynthesizeText() 进一步处理，用于请求和回调结果。

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeechService.java
private abstract class SpeechItem {
    public void play() {
        ...
        playImpl();
    }
}
//这里是tts，所以只看SynthesisSpeechItem这个item
class SynthesisSpeechItem extends UtteranceSpeechItemWithParams {
    public SynthesisSpeechItem(
                Object callerIdentity,
                int callerUid,
                int callerPid,
                Bundle params,
                String utteranceId,
                CharSequence text) {
        //初始化SynthesisRequest，将text文本封装成一个请求
        mSynthesisRequest = new SynthesisRequest(mText, mParams);
        ...
    }
    ...
    @Override
    protected void playImpl() {
        AbstractSynthesisCallback synthesisCallback;

        synchronized (this) {
            ...
            //这个createSynthesisCallback实际上是PlaybackSynthesisCallback实例
            mSynthesisCallback = createSynthesisCallback();
            synthesisCallback = mSynthesisCallback;
        }
		//最终调用到这里onSynthesizeText，这个是需要引擎去实现的
        TextToSpeechService.this.onSynthesizeText(mSynthesisRequest, synthesisCallback);
        if (synthesisCallback.hasStarted() && !synthesisCallback.hasFinished()) {
            synthesisCallback.done();
        }
    }
    ...
}
```

### 3.4.4synthesisCallback回调

关于onSynthesizeText这个是需要引擎做的操作，具体可以参照demo来做，一般SDK由产商实现（离线在线的引擎，讯飞，阿里，百度，腾讯，思必驰，云之声等等）。

> 当然系统也为我们提供了一个例子
> /development/samples/TtsEngine/src/com/example/android/ttsengine/RobotSpeakTtsService.java
> 当然，需要实现TextToSpeechService中的抽象方法即可。

需要 Engine 复写以将 text 合成 audio 数据，也是 TTS 功能里最核心的实现，功能实现完成之后，就会监听的回调

按照顺序，依次是start、audioAvailable和done。

```java
//frameworks/base/core/java/android/speech/tts/PlaybackSynthesisCallback.java
class PlaybackSynthesisCallback extends AbstractSynthesisCallback {
    ...
    @Override
    public int start(int sampleRateInHz, int audioFormat, int channelCount) {
        ...
        synchronized (mStateLock) {
            ...
            //封装成一个个Item，发送到阻塞队列中
            SynthesisPlaybackQueueItem item = new SynthesisPlaybackQueueItem(
                    mAudioParams, sampleRateInHz, audioFormat, channelCount,
                    mDispatcher, mCallerIdentity, mLogger);
            mAudioTrackHandler.enqueue(item);
            mItem = item;
        }
        return TextToSpeech.SUCCESS;
    }

    @Override
    public int audioAvailable(byte[] buffer, int offset, int length) {
        SynthesisPlaybackQueueItem item = null;
        synchronized (mStateLock) {
            ...
            item = mItem;
        }
        //对数据进行拷贝
        final byte[] bufferCopy = new byte[length];
        System.arraycopy(buffer, offset, bufferCopy, 0, length);
        //将拷贝的内容发送给App选择处理
        mDispatcher.dispatchOnAudioAvailable(bufferCopy);

        try {
            //处理完成之后加入队列中以供消费
            //上述的 QueueItem 触发 Lock 接口的 take Condition 恢复执行，最后调用 AudioTrack 去播放
            item.put(bufferCopy);
        }
        ...
        return TextToSpeech.SUCCESS;
    }

    @Override
    public int done() {
        ...
        return TextToSpeech.SUCCESS;
    }
    ...
}
```

### 3.4.5dispatchOnAudioAvailable数据分发

上述 PlaybackSynthesisCallback 在通知 QueueItem 的同时，会通过 UtteranceProgressDispatcher 接口将数据、结果一并发送给 Request App

UtteranceProgressDispatcher 接口的实现就是 TextToSpeechService 处理 speak 请求的 UtteranceSpeechItem 实例，其通过缓存着各 `ITextToSpeechCallback` 接口实例的 CallbackMap 发送回调给 TTS 请求的 App。（这些 Callback 来自于 TextToSpeech 初始化时候通过 ITextToSpeechService 将 Binder 接口传递来和缓存起来的。）

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeechService.java
private abstract class UtteranceSpeechItem extends SpeechItem
    implements UtteranceProgressDispatcher  {
    @Override
    public void dispatchOnAudioAvailable(byte[] audio) {
        final String utteranceId = getUtteranceId();
        if (utteranceId != null) {
            //直接调用到mCallbacks中方法，mCallbacks实际上是CallbackMap
            mCallbacks.dispatchOnAudioAvailable(getCallerIdentity(), utteranceId, audio);
        }
    }
    ...
}

private class CallbackMap extends RemoteCallbackList<ITextToSpeechCallback> {
    public void dispatchOnAudioAvailable(Object callerIdentity, String utteranceId, byte[] buffer) {
        ITextToSpeechCallback cb = getCallbackFor(callerIdentity);
        if (cb == null) return;
        try {
            //最终回调到cb
            cb.onAudioAvailable(utteranceId, buffer);
        } ...
    }
    ...
}
```

### 3.4.6 回调状态给App

ITextToSpeechCallback，这个非常眼熟。实际上在上述2demo中的执行将通过 TextToSpeech 的中转抵达请求 App 的 Callback

而上述cb的初始化正好和App对接上了

```java
//frameworks/base/core/java/android/speech/tts/TextToSpeech.java
public int setOnUtteranceProgressListener(UtteranceProgressListener listener) {
    mUtteranceProgressListener = listener;
    return TextToSpeech.SUCCESS;
}
```

App的回调

```java
//App
textToSpeech.setOnUtteranceProgressListener(new UtteranceProgressListener() {
    @Override
    public void onStart(String utteranceId) {
        //自定义回调函数的处理
    }

    @Override
    public void onDone(String utteranceId) {
        //自定义回调函数的处理
    }

    @Override
    public void onError(String utteranceId) {
        //自定义回调函数的处理
    }

    @Override
    public void onAudioAvailable(String utteranceId, byte[] audio) {
        super.onAudioAvailable(utteranceId, audio);
        //这里的audio是原始数据，可以进行音频数据处理，比如音频3A处理
    }
});
```



# 4总结

总的来说流程不是很复杂，其实就是App与框架的交互过程。

1. TTS Request App 调用 `TextToSpeech` 构造函数，由系统准备播报工作前的准备，比如通过 `Connection` 绑定和初始化目标的 TTS Engine
2. Request App 提供目标 *text* 并调用 `speak()` 请求
3. TextToSpeech 会检查目标 text 是否设置过本地的 audio 资源，没有的话回通过 Connection 调用 `ITextToSpeechService` AIDL 的 speak() 继续
4. `TextToSpeechService` 收到后封装请求 `SynthesisRequest` 和用于回调结果的 `SynthesisCallback` 实例
5. 之后将两者作为参数调用核心实现 `onSynthesizeText()`，其将解析 Request 并进行 Speech 音频数据合成
6. 此后通过 SynthesisCallback 将合成前后的关键回调告知系统，尤其是 `AudioTrack` 播放
7. 同时需要将 speak 请求的结果告知 Request App，即通过 `UtteranceProgressDispatcher`中转，实际上是调用 `ITextToSpeechCallback` AIDL
8. 最后通过 `UtteranceProgressListener` 告知 TextToSpeech 初始化时设置的各回调

{{< image classes="fancybox center fig-100" src="/tts/tts_4.png" thumbnail="/tts/tts_4.png" title="">}}

# 参考

[[1] Zephyr Cai, android系统tts TextToSpeech源码原理解析及定制tts引擎, 2020.](https://blog.csdn.net/caizehui/article/details/103898611)

[[2] 小虾米君, 直面原理：5 张图彻底了解 Android TextToSpeech 机制, 2023.](https://mp.weixin.qq.com/s/pWNhQ1PgmocGkZXsX-TObA)
