jni实战



调音网站fmod

https://www.fmod.com/



```
DSP digital signal process ==数字信号处理

fmod 音效引擎库 c/c++写出来的
CoCos2d， unity3d 游戏引擎，默认集成了fmod

CPU架构
arm64-v8a	//目前主流
armeabi-v7a	//2015的版本
armeabi		//2010的版本
x86			//手机模拟器
查看手机cpu平台
adb shell getprop ro.product.cpu.abi
```



> Pitch 
>
> 正常1.0
>
> 音调搞--萝莉2x
>
> 音调低--大叔0.8x
>
> 音调更低--老头0.5x
>
> 
>
> Tremole
>
> 非常颤抖20
>
> 正常 5



cmake解释

```cmake
cmake_minimum_required(VERSION 3.4.1) ##最低支持cmake版本

# 1.导入inc头文件
include_directories("inc")

# 批量导入所有源文件
file(GLOB allCPP *.c *.h *.cpp)

add_library(
        native-lib #最终生成libnative-lib.so
        SHARED #动态库
        # native-lib.cpp 如果只有一个cpp文件，可以直接加载进来，如果数量很多不用这种方式，用批量方式
        ${allCPP}
)

# 2.导入对应inc的库文件
# -L是链接方式
#其中CMAKE_CXX_FLAGS是一种规则，将新规则连接到先前的规则，原来的规则${CMAKE_CXX_FLAGS}
#这种规则类似win里面环境变量path，添加新的环境变量%JAVA_HOME%
# CMAKE_SOURCE_DIR == CMakeLists.txt所在的路径
# CMAKE_ANDROID_ARCH_ABI 为当前的CPU架构==arm64-v8a
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_SOURCE_DIR}/../jniLibs/${CMAKE_ANDROID_ARCH_ABI}")

#3.链接具体的库，到我们的libnative-lib.so总库
#寻找android win ndk的路径中的liblog.so
# 笔者的路径D:\android\sdk\ndk\21.4.7075529\platforms\android-30\arch-arm64\usr\lib\liblog.so
find_library(
        log-lib
        log #自动寻找
)

#链接到具体的库需要去掉lib和.so的前后缀
target_link_libraries(
        native-lib
        ${log-lib}
        fmod
        fmodL
)

#或者不写find_library,直接写target_link_libraries
# target_link_libraries(
#        native-lib
#        log #自动寻找路径的库
#)
```

关于set(CMAKE_CXX_FLAGS)可以看这篇文章[CMAKE_CXX_FLAGS中的标志"-l"不起作用](https://www.it1352.com/1560318.html)

默认创建jniLibs，跟java、cpp同级目录，也可以创建自定义目录，sourceSet

![image-20220120230252227](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220120230252227.png)

```cmake
externalNativeBuild {
    ndkBuild {
        path file("src/main/jni/Android.mk")
    }
}
sourceSets {
    main {
        jniLibs.srcDirs = ['libs']
    }
}
```



在app的gradle里面

```gradle
//修改前
defaultConfig {
    applicationId "com.derry.as_jni_projectkt"
    minSdkVersion 16
    targetSdkVersion 30
    versionCode 1
    versionName "1.0"

    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

    // TODO 第四步：指定CPU架构 Cmake中的本地库，例如：libnative-lib.so libderryku.so
    externalNativeBuild {
        cmake {
            cppFlags "" //默认四大平台
        }
    }
}

//修改后
defaultConfig {
    applicationId "com.derry.as_jni_projectkt"
    minSdkVersion 16
    targetSdkVersion 30
    versionCode 1
    versionName "1.0"

    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

    // TODO 第四步：指定CPU架构 Cmake中的本地库，例如：libnative-lib.so libderryku.so
    // 这个abiFilters说明，只有这个cpu架构下的cmake会编译本地libnative-lib.so，其他的架构不编译
    externalNativeBuild {
        cmake {
            abiFilters "armeabi-v7a"
        }
    }

    // TODO 第五步：指定CPU的架构  apk/lib/平台,不写这个，apk依然会有四个平台的so
    ndk {
        abiFilters("armeabi-v7a")
    }
}
```



![image-20220122122108173](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220122122108173.png)

注意：

> **兼容**
> 只适配armeabi的APP可以跑在armeabi,x86,x86_64,armeabi-v7a,arm64-v8上
>
> 只适配armeabi-v7a可以运行在armeabi-v7a和arm64-v8a
>
> 只适配arm64-v8a 可以运行在arm64-v8a上

app的build.gradle导入三方fmodjar包，在Mainactivity初始化fmod

```gradle
dependencies{
    //第六步导入jar包
    implementation files("libs\\fmod.jar")
}
```

在Mainactivity初始化fmod

```java
private static final int MODE_NORMAL = 0; // 正常
private static final int MODE_LUOLI = 1; //
private static final int MODE_DASHU = 2; //
private static final int MODE_JINGSONG = 3; //
private static final int MODE_GAOGUAI = 4; //
private static final int MODE_KONGLING = 5; //

static {
    System.loadLibrary("native-lib");
}

private String path;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    path =  "file:///android_asset/derry.mp3";

    FMOD.init(this);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    FMOD.close();
}

//设置声音模式和声音保存路径
private native void voiceChangeNative(int modeNormal, String path);
```

如果需要通过java自动生成native的宏(javah -包名.类名)

需要javah com.derry.derry_voicechange.MainActivity

![image-20220122123805963](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220122123805963.png)



开始准备变换声音

```c++
//change voice
/*
> Pitch 
正常1.0
音调搞--萝莉2x
音调低--大叔0.8x
音调更低--老头0.5x
> Tremole
非常颤抖20
正常 5
*/
using namespace FMOD;//使用fmod的命名空间

extern "C" JNIEXPORT void JNICALL Java_com_derry_derry_1voicechange_MainActivity_voiceChangeNative
        (JNIEnv * env, jobject thiz, jint mode, jstring path) {
    char* content = "默认播放";
    //路径
    const char * cpath = env->GetStringUTFChars(path, NULL);
    
    //fmod初始化指针,音效引擎，声音，通道，dsp调节指针
    System* system = nullptr;
    Sound* sound = nullptr;
    Channel* channel = nullptr;
    DSP* dsp = nullptr;
    
    //1.创建系统
    System_Create(&system);
    //2.系统初始化 参数1最大音轨数 参数2初始化标记 参数3 额外数据暂时不用
    system->init(32, FMOD_INIT_NORMAL, 0);
    //3.创建声音 参数1路径 参数2声音初始化标记 参数3额外数据 参数4声音指针
    system->createSound(cpath, FMOD_DEFAULT, 0, &sound);
    //4.播放声音(需要音轨和声音) 参数1 声音 参数2分组音轨 参数3控制pause 参数4通道
    system->playSound(sound, 0, false, &channel);
    //5.增加特效
    switch(mode){
        case com_derry_derry_voicechange_MainActivity_MODE_NORMAL:
            content = "原生播放"
            break;
        case com_derry_derry_voicechange_MainActivity_MODE_LUOLI:
            content = "萝莉播放"
            //1.创建dsp的pitch音调条件 2.通过dsp的pitch设置2.0 3.添加音效到音轨
            system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT, &dsp);
            dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH, 2.0f);
            channel->addDSP(0, dsp);//通道0
            break;
        case com_derry_derry_voicechange_MainActivity_MODE_DASHU:
            content = "大叔播放"
            //1.创建dsp的pitch音调条件 2.通过dsp的pitch设置2.0 3.添加音效到音轨
            system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT, &dsp);
            dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH, 0.7f);
            channel->addDSP(0, dsp);//通道0
            break;
        case com_derry_derry_voicechange_MainActivity_MODE_GAOGUAI:
            content = "搞怪播放"
            //改变的是声音频率，需要从音轨获取当前频率,最后修改频率
            float mFrequency;
            channel->getFrequency(&mFrequency);
            channel->setFrequency(mFrequency * 1.5f);
            break;
        case com_derry_derry_voicechange_MainActivity_MODE_KONGLING:    
            content = "广播播放"
            //存在回音
            system->createDSPByType(FMOD_DSP_TYPE_ECHO, &dsp);
            dsp->s
            break;
        case com_derry_derry_voicechange_MainActivity_MODE_JINGSONG:
            content = "惊悚播放"
            break; 
    }
    
    //等待延时播放直到结束，播放完成内部channel会修改isPlayer的状态值
    bool isPlayer = true;
    while(isPlayer){
        channel->isPlaying(&isPlayer);
        usleep(1000 * 1000);
    }
    //释放
    sound->release();
    system->close();
    system->release();
    env->ReleaseStringUTFChar(path, cpath);
    
}
```

