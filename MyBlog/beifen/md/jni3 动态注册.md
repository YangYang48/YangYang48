jni3 动态注册



默认是静态注册，自动生成包名加类名的形式的函数名的jni

默认情况下，就是静态注册，静态注册是最简单的方式，NDK开发过程中，基本上使用静态注册

Android系统的C++源码，基本上使用动态注册 

静态注册：

缺点：jni函数比较长，捆绑上层包名+类名，运行期才会匹配jni函数，性能低于动态注册

![image-20220207211317614](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220207211317614.png)

动态注册相等于在System.loadLibrary()初始化的时候就已经准备了，而静态注册需要每次调用才能初始化

```java
//java调用
public native void dynamicJavaMethod1();//动态注册调用
public native void dynamicJavaMethod2(String valueStr);//动态注册调用2,有参
```



```c++
//Java:像Java的构造函数。如果不写构造函数，默认就有构造函数。如果写构造函数，会覆盖默认的
//JNI JNI_OnLoad函数，如果不写JNI_OnLoad函数，默认就有JNI_OnLoad，如果你写JNI_OnLoad则覆盖默认的
//System.loadLibrary()这个在Java层调用后会默认调用JNI_OnLoad函数
JavaVm* javaVm = nullptr;//c++11后，推荐使用nullptr

const char* mainActivityClassName = "com/derry/as_jni_project/MainActivity";
const char* javaMethod1 = "dynamicJavaMethod1";

//声明native真正的函数
//其中函数JNIEnv* env, jobject thiz可以省略，推荐写全
void dynamicMethod1(JNIEnv* env, jobject thiz){
    
}

int dynamicMethod2(JNIEnv* env, jobject thiz, jstring valueStr){
    const char* text = env->GetStringUTFChars(valueStr, nullptr);
    LOGD("我是动态注册text(%s)", text);
    env->ReleaseStringUTFChars(valueStr, text);
    return 0;
}

static const JNINativeMethod jniNativeMethod[] = {
    {javaMethod1, "()V", (void*)(dynamicMethod1)},
    {"dynamicJavaMethod2", "(Ljava/lang/String;)I", (int*)(dynamicMethod2)},
};

JNIEXPORT jint JNI_OnLoad(JavaVM* javaVm, void* reserved){
    
    //等同this.javaVm = javaVm
    ::javaVm = javaVm;
    
    JNIEnv* jniEnv = nullptr;
    int result = javaVm->GetEnv(reinterpret_cast<void**>(&jniEnv), JNI_VERSION_1_6);
    //result成功为0
    if (result != JNI_OK){
        return -1;//会直接奔溃
    }
    
    jclass mainActivityClass = jniEnv->FindClass(mainActivityClassName);
    //jint RegisterNatives(jclass clazz, const JNINativeMethod* methods, jnit nMethods)
    //需要jclass即mainActivity的this
    //其中JNINativeMethod为结构体
    //typedef struct{
    //    const char* name;//函数名
    //    const char* signature;//函数签名
    //    void* fnPtr;//函数指针
	//}JNINativeMethod;
    jniEnv->RegisterNatives(mainActivityClass, 
                            jniNativeMethod,
                            sizeof(jniNativeMethod)/sizeof(JNINativeMethod));
    
    return JNI_VERSION_1_6;//as jdk 默认jni最高1.6
}
```



[android jni (jni_onload方式)](https://blog.csdn.net/qq_33782617/article/details/111770261)
