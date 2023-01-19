---
title: "linux4 NDK交叉编译"
date: 2022-03-06
thumbnailImagePosition: left
thumbnailImage: linux/linux2_thumb.jpeg
coverImage: linux/linux2_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- NDK
- 2022
- March 
tags:
- linux
- Android
- 交叉编译
showSocial: false
---

在当前编译平台下，编译出来的程序能运行在另一种目标平台上，但是编译平台本身却不能运行该程序。而我们在linux服务器编译的库可以直接在Android工程中编译。

<!--more-->
交叉编译
# 1.编译简单的交叉编译
```shell
#ubuntu下载命令
sudo apt-get install xxx
#centOs下载命令
sudo yum install xxx
# 下载文件命令，两者都可用
sudo wget xxx(下载地址)
# NDK下载地址
sudo wget https://dl.google.com/android/repository/android-ndk-r17c-linux-x86_64.zip?
hl=zh_cn
```

查看手机 CPU架构（arm64-v8a架构）

笔者用得是root过的红米5手机，Android8.1，Api版本27。

```shell
picasso:/ $ cat /proc/cpuinfo
# arm64的
Processor       : AArch64 Processor rev 14 (aarch64)
```

error01:stdio.h: No such file or directory

```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ export NDK_GCC="/home/yangyang/NDK/android-ndk-r17c/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/bin/aarch64-linux-android-gcc"
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC ndk01.c -o ndk01_exe         ndk01.c:1:19: fatal error: stdio.h: No such file or directory
 #include <stdio.h>
                   ^
compilation terminated.
```

注：这里需要写全路径，不能写成~/NDK/android-ndk-r17c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-gcc会提示找不到路径



 error02: undefined reference to 'main'

```shell
/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-arm64

/home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include

/home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/aarch64-linux-android

export NDKPATH="--sysroot=/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-arm64 -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/aarch64-linux-android"

yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC $NDKPATH ndk01.c -o ndk_EXE
/home/yangyang/NDK/android-ndk-r17c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: /tmp/ccetB5z2.o: incompatible target
/home/yangyang/NDK/android-ndk-r17c/platforms/android-28/arch-arm64/usr/lib/crtbegin_dynamic.o:crtbegin.c:function _start_main: error: undefined reference to 'main'
/home/yangyang/NDK/android-ndk-r17c/platforms/android-28/arch-arm64/usr/lib/crtbegin_dynamic.o:crtbegin.c:function _start_main: error: undefined reference to 'main'
collect2: error: ld returned 1 exit status
```

上述问题，可能是两种情况引起的。

第一种情况：配置的NDK_GCC和NDKPATH中的工具链版本不一致（32不能匹配64）。

第二种情况：需要增加请在编译命令中加上**-shared** 或加在LD_FLAGS 后。

```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC $NDKPATH ndk01.c -o ndk_EXE
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ ls -al
-rw-rw-r--  1 yangyang yangyang   82 Mar  6 10:31 ndk01.c
-rwxrwxr-x  1 yangyang yangyang 7832 Mar  6 11:17 ndk_EXE
```

手机中验证如下

error03：position-independent executables (-fPIE)

```shell
vince:/data # ./ndk_EXE
"./ndk_EXE": error: Android 5.0 and later only support position-independent executables (-fPIE).
```

服务器编译的时候加上pie

```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC -pie $NDKPATH ndk01.c -o ndk_EXE
```

在手机中继续验证

```shell
$ adb push C:\Users\Admin\Desktop\ndk_EXE /data
C:\Users\Admin\Desktop\ndk_EXE: 1 file pushed. 1.3 MB/s (7832 bytes in 0.006s)
1|vince:/data # ls -al ndk_EXE
-rw-rw-rw- 1 root root 7832 2022-03-06 17:42 ndk_EXE
vince:/data # chmod +x ndk_EXE
vince:/data # ./ndk_EXE
->>>hello world!
```

每次配置环境变量比较麻烦，一般直接在linux服务器中添加到内部环境变量中。

```shell
#【i686-xx x86=32】
export NDK_GCC_x86="/home/yangyang/NDK/android-ndk-r17c/toolchains/x86-4.9/prebuilt/linux-x86_64/bin/i686-linux-android-gcc"
#【x86_64-xx x86_64=64】
export NDK_GCC_x64="/home/yangyang/NDK/android-ndk-r17c/toolchains/x86_64-4.9/prebuilt/linux-x86_64/bin/x86_64-linux-android-gcc"
#【arm32 == arm-linuxandroideabi-gcc】
export NDK_GCC_arm="/home/yangyang/NDK/android-ndk-r17c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-gcc"
#【arm64 == aarch64-linux-android-gcc】
export NDK_GCC_arm_64="/home/yangyang/NDK/android-ndk-r17c/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/bin/aarch64-linux-android-gcc"
```



```shell
export NDK_CFIG_x86="--sysroot=/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-x86/ -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/i686-linux-android"

export NDK_CFIG_x64="--sysroot=/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-x86_64/ -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/x86_64-linux-android"

export NDK_CFIG_arm="--sysroot=/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-arm/ -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/arm-linux-androideabi"

export NDK_CFIG_arm_64="--sysroot=/home/yangyang/NDK/android-ndk-r17c/platforms/android-27/arch-arm64/ -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include -isystem /home/yangyang/NDK/android-ndk-r17c/sysroot/usr/include/aarch64-linux-android"
```



```shell
#方法1：让/etc/profile文件修改后立即生效 ,可以使用如下命令:
.  /etc/profile
#注意: . 和 /etc/profile 有空格
#方法2：让/etc/profile文件修改后立即生效 ,可以使用如下命令:
source /etc/profile
```

# 2.交叉编译动态库

```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC_arm_64 $NDK_CFIG_arm_64 -fPIC -shared get.c -o libgetndkso.so

yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ file libgetndkso.so
libgetndkso.so: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped
```

查看是否为get函数

```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ nm -D libgetndkso.so |awk '{if($2 == "T"){print $3}}'
get
```



# 3.交叉编译静态库

```shell
export NDK_AR_x86="/home/yangyang/NDK/android-ndk-r17c/toolchains/x86-4.9/prebuilt/linux-x86_64/bin/i686-linux-android-ar"
export NDK_AR_x64="/home/yangyang/NDK/android-ndk-r17c/toolchains/x86_64-4.9/prebuilt/linux-x86_64/bin/x86_64-linux-android-ar"
export NDK_AR_arm="/home/yangyang/NDK/android-ndk-r17c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-ar"
export NDK_AR_arm_64="/home/yangyang/NDK/android-ndk-r17c/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/bin/aarch64-linux-android-ar"
```



```shell
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_GCC_arm_64 $NDK_CFIG_arm_64 -fPIC -c get.c -o getndka.o
yangyang@iZbp1788g5aoi5p9z8e0ugZ:~/other$ $NDK_AR_arm_64 rcs -o libgetndka.a getndka.o
```



# 4.Android Studio编译验证

配置文件

cmake

## 静态库

```cmake
cmake_minimum_required(VERSION 3.10.2)

project("mycrosscompile")

file(GLOB allCPP *.cpp)

add_library( # Sets the name of the library.
             native-lib
             SHARED
             ${allCPP} )

find_library( # Sets the name of the path variable.
              log-lib
              log )

## 导入静态库,这里的import是一个标识
add_library(getStaticNdk STATIC IMPORTED)

set_target_properties(getStaticNdk PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libgetndka.a)

target_link_libraries( # Specifies the target library.
        native-lib
        ${log-lib}
        getStaticNdk)
```

## 动态库

关于动态库，笔者还会增加额外的新篇幅来阐述，这里简单的当一个demo编译。

2 files found with path 'lib/arm64-v8a/libgetndkso.so'.
If you are using jniLibs and CMake IMPORTED targets, see
https://developer.android.com/r/tools/jniLibs-vs-imported-targets

```cmake
cmake_minimum_required(VERSION 3.10.2)

project("mycrosscompile")

file(GLOB allCPP *.cpp)

add_library( # Sets the name of the library.
             native-lib
             SHARED
             ${allCPP} )

find_library( # Sets the name of the path variable.
              log-lib
              log )

## 导入静态库,这里的import是一个标识
#add_library(getStaticNdk STATIC IMPORTED)

#set_target_properties(getStaticNdk PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libgetndka.a)

## 导入动态库,这里的import是一个标识
add_library(getSharedNdk SHARED IMPORTED)

set_target_properties(getSharedNdk PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/../jniLibs/arm64-v8a/libgetndkso.so)

target_link_libraries( # Specifies the target library.
        native-lib
        ${log-lib}
        getSharedNdk)
```

build.gradle

```groovy
android {
    compileSdkVersion 30
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "com.example.mycrosscompile"
        minSdkVersion 21
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            //===========新增
            cmake {
                 //cppFlags''
                abiFilters "arm64-v8a"
            }
            //===========
        }
        //===========新增
        ndk{
            abiFilters "arm64-v8a"
        }
        //===========
    }
```



# 4.关于手机没有root权限

网上有好多刷机工具可选择。笔者用得是红米5的手机，直接可以在官网下载开发版本[红米5 Plus](https://www.miui.com/download-340.html)。下载完成之后，根据官网提示手动选择自动安装包，选择下载的开发版本即可。

没有手动更新选项，需要连续点击MIUI的LOGO10次。

下载完成后，通过应用权限中开启root权限，稍等片刻即会有root权限。如果应用权限需要通过解锁工具，点击[这里](https://github.com/YangYang48/project/tree/master/miflash_v5.5.224.55)下载。



[本文所有实例代码下载](https://github.com/YangYang48/project/tree/master/MyCrossCompile)



# 猜你喜欢

[linux1 基础学习](https://yangyang48.github.io/2022/03/linux1-%E5%9F%BA%E7%A1%80%E5%AD%A6%E4%B9%A0/)

[linux2 权限和管理](https://yangyang48.github.io/2022/04/linux2-%E6%9D%83%E9%99%90%E5%92%8C%E7%AE%A1%E7%90%86/)

[linux3 Vim和shell脚本](https://yangyang48.github.io/2022/04/linux3-vim%E5%92%8Cshell%E8%84%9A%E6%9C%AC/)
