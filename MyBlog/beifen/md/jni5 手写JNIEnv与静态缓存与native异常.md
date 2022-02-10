jni5 手写JNIEnv与静态缓存与native异常

```kotlin
//定义native函数
//public native String stringFromJNI();
external fun stringFromJNI(): String
//public native void sort(int[] arr);
external fun sort(arr: IntArray)
//static {System.loadLibrary("native-lib");}
companion object{
    init{
        System.loadLibrary("native-lib")
    }
}

fun sortAction(view: View){
    val arr = intArrayof(11, 2, -3, 2, 4, 6, -15);
    sort(arr)
    
    for(element in arr){
        Log.d(TAG, "->>>sortAction " + element.toString() + "\t")
    }
}
```



```c++
//比较函数
int compare(const jint* number1, const jint* number2){
    return *number1 - *number2;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_sort(JNIEnv* env, jobject thiz, jnitArray arr){
    jnit* intArray = env->GetIntArrayElements(arr, nullptr);
    int len = env->GetArrayLength(arr);
    //这个qsort函数是ndk对stdlib封装的函数
    //参数1：void* 数组的首地址
    //参数2：数组的大小长度
    //参数3：元素的大小
    //参数4：对比的指针
    qsort(intArray, len, sizeof(int),
         reinterpret_cast<int (*)(const void*, const void*)>(compare));
    env->ReleaseIntArrayElements(arr, intArray, 0);//0为操纵杆，更新KT数组
}
```



静态缓存

opencv和webrtc会大量使用静态缓存,主要是为了提高性能

```java
//MainActivity2.java
static String name1 = "T1";
static String name2 = "T2";
static String name3 = "T3";

public static native void localCache();//通过局部缓存，演示弊端
//静态缓存
public static native void initCache();//这个在构造函数中去调用
public static native void staticCache(String name);//如果外部多次调用这个，可以比localCache提高性能
public static native void clearStaticCache();//这个在ondestory调用
```



```c++
static jfieldID f_name1_id = nullptr;
static jfieldID f_name2_id = nullptr;
static jfieldID f_name3_id = nullptr;

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity2_initCache(JNIEnv* env, jclass clazz){
    f_name1_id = env->GetStaticFieldID(clazz, "name1", "Ljava/lang/String;");
    f_name2_id = env->GetStaticFieldID(clazz, "name2", "Ljava/lang/String;");
    f_name3_id = env->GetStaticFieldID(clazz, "name3", "Ljava/lang/String;");
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity2_staticCache(JNIEnv* env, jclass clazz, jstring name){
    //不会反复GetStaticFieldID，提高性能
    env->SetStaticObjectField(clazz, f_name1_id, name);
    env->SetStaticObjectField(clazz, f_name2_id, name);
    env->SetStaticObjectField(clazz, f_name3_id, name);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity2_clearCache(JNIEnv* env, jclass clazz, jstring name){
    jfieldID f_name1_id = nullptr;
    jfieldID f_name2_id = nullptr;
    jfieldID f_name3_id = nullptr;
}
```



native异常

```java
//MainActivity3.java
static String name1 = "T1";
//前两个是在native的主动异常，后面一个native的被动异常
public static native void exception();
//这里的NoSuchFieldException是让c++层可以抛异常上来
public static native void exception2() throws NoSuchFieldException;
public static native void exception3();

public void exceptionAction(View view){
    exception();
    
    //捕获C++层抛上来的异常
    try{
        exception2();
    }catch(NoSuchFieldException e){
        e.printStackTrace();
        Log.d(TAG, "exception2 异常捕获");
    }
    exception3();
}

//专门给c++、native调用的函数
public static void show() throws Exception{
    Log.d(TAG, "->>>show 1111");
    throw new NullPointerException("我是java中抛出的异常");
}
```



```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity3_exception(JNIEnv* env, jclass clazz){
    //name999是无法被找到的
    jfieldID f_id = env->GetStaticFieldID(clazz, "name999", "Ljava/lang/String;");
    
    //奔溃后的解决方法：
    //方式1:补救措施
    
    //监测本次运行有没有异常
    jthrowable thr = env->ExceptionOccurred();
    if(thr){
        LOGD("c++有异常 检测到了");
        env->ExceptionClear();//检测到的异常被清除
        
        //开始补救措施
        jfieldID f_id = env->GetStaticFieldID(clazz, "name1", "Ljava/lang/String;");
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity3_exception2(JNIEnv* env, jclass clazz){
    //name999是无法被找到的
    jfieldID f_id = env->GetStaticFieldID(clazz, "name888", "Ljava/lang/String;");
    
    //奔溃后的解决方法：
    //方式2:把异常抛给java层
    
    //监测本次运行有没有异常
    jthrowable thr = env->ExceptionOccurred();
    if(thr){
        LOGD("c++有异常 检测到了");
        env->ExceptionClear();//检测到的异常被清除
        
        jclass clz = env->FindClass("java/lang/NoSuchFieldException");
        env->ThrowNew(clz, "NoSuchFieldException 找不到name888，抛给你了");
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity3_exception2(JNIEnv* env, jclass clazz){
    //这个函数本身没有异常，是java回调过去的异常
    jMethodID showID = env->GetStaticMethodID(clazz, "show", "()V");
    env->CallStaticVoidMethod(clazz, showID);//并不是这句话奔溃的，这里是慢慢的奔溃
    //ExceptionCheck == 慢慢的奔溃，相当于给你了空余时间,日志可以打印,但不可以加其他的，不然异常清除动作失效
    LOGD("c++层  1111");
    if (env->ExceptionCheck()){
        env->ExceptionDescribe();//输出描述信息
        env->ExceptionClear();//此异常清除
    }
    
}
```



C++异常

其中*和&可以同时存在

```c++
#include <iostream>
#include <string>
using namespace std;

void exceptionMethod(){
    throw "我报废了";//const char*
}

class Student{
public:
    char* getInfo(){
        return "自定义";
    }
};

void exceptionMethod2(){
    Student student;
    throw student;
}

int main(){
    try{
        exceptionMethod();
    }catch(const char* & msg){
        cout << "捕获到异常： " << msg << endl;
    }
    
    try{
        exceptionMethod2();
    }catch(Student& msg){
        cout << "捕获到异常2： " << msg.getInfo() << endl;
    }
}
```



手写JNIEnv

JNIEnv的本质

env是JNINativeInterface的二级指针

```c++
//jni.h
#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;
#endif
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
    ...
};
```

![image-20220210223252012](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220210223252012.png)

关于原理介绍

[JNIEnv介绍](https://blog.csdn.net/caiqiiqi/article/details/77845447)

[Android系统的JNI原理分析（三）- 关于JNIEnv](https://blog.csdn.net/Xiaoma_Pedro/article/details/112259792)

```c++
//env c++是1级指针(内部_JNIEnv对JNINativeInterface做了封装)，c为二级指针
#include <iostream>
#include <string>
using namespace std;
//模拟jstring,jobject
typedef char* jstring;
typedef char* jobject;
typedef const struct JNINativeInterface* JNIEnv;

struct JNINativeInterface{
    //300多个函数，只列出一个
    jstring (*NewStringUTF)(JNIEnv*, char *);
    //...
};

//函数指针真正实现
jstring NewStringUTF(JNIEnv* env, char* str){
    //真正str--jstring转化比较复杂，这里简单指代
    return str;
}

jstring Java_com_derry_as_1jni_1project_MainActivity_derryAction(JNIEnv* env, jobject job){
    //env为JNINativeInterface的二级指针
    return (*env)->NewStringUTF(env, "123456");
}

int main(){
    struct JNINativeInterface nativeInterface;
    nativeInterface.NewStringUTF = NewStringUTF;//赋值给函数指针
    
    JNIEnv env = &nativeInterface;
    JNIEnv* jniEnv = &env;
    
    jstring ret = Java_com_derry_as_1jni_1project_MainActivity_derryAction(jniEnv, "com/derry/jni/MainActivity");
    
    printf("Java层拿到C++的String result：%s", ret);
}
```

