jni4 线程



```java
//java调用native
public native void nativeThread();

public void updateActivityUI(){
    if (Looper.getMainLooper() == Looper.myLooper()){
        new AlertDialog.Builder(MainActivity.this)
            .setTitle("UI")
            .setMessage("updateActivityUI activity ui ...")
            .setPositiveButton("xxxx", null)
            .show();
    }else {
        Log.d(TAG, "updateActivityUI 属于子线程");
        
        runOnUiThread(new Runnable(){
            @override
            public void run(){
                new AlertDialog.Builder(MainActivity.this)
                        .setTitle("UI")
                        .setMessage("updateActivityUI 属于子线程")
                        .setPositiveButton("xxxx", null)
                        .show(); 
            }
        });
    }
}
```

JNIEnv* env不能跨越线程，否则崩溃，可以跨越函数

jobject thiz不能跨越线程，否则崩溃，不能跨越函数，否则崩溃

JavaVM 能够跨越线程和函数

```c++
#include <pthread.h> //as不需要额外配置
class MyContext{
public:
    JNIEnv* jniEnv = nullptr;
    jobject instance;
};

void* myThreadTaskAction(void* pVoid){
    LOGD("myThreadTaskAction");
    
    MyContext* myContext = static_cast<MyContext*>(pVoid);
    //todo 解决方式（android进程只有一个JavaVM进程，是全局的，可以跨线程）
    JNIEnv* jniEnv = nullptr;//用于异步操作的jniEnv
    //附加当前异步线程后，会得到一个全新的env，此env相当于是子线程专用的env，这个是和线程绑定在一起的
    jint ret = ::javaVm->AttachCurrentThread(&jniEnv, nullptr);
    if(ret != JNI_OK){
        return -1;
    }
    
    //1.拿到class
    jclass mainActivityClass = jniEnv->GetObjectClass(myContext->instance);
    //2.拿到方法
    jmethodID updateActivityUI = jniEnv->GetMethodID(mainActivityClass, 
                                                     "updateActivityUI",
                                                     "()V");
    //3.调用方法
    jniEnv->CallVoidMethod(myContext->instance, updateActivityUI);
    
    ::javaVm->DetachCurrentThread();//必须解除附加，否则报错
    
    return nullptr;
}

//JNIEnv* env不能跨越线程，否则崩溃，可以跨越函数【解决方法：使用全局JavaVM即附加当前异步线程，获取env操作】
//jobject thiz不能跨越线程，否则崩溃，不能跨越函数，否则崩溃【解决方法：默认局部引用，提升至全局引用】
//JavaVM 能够跨越线程和函数
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeThread(JNIEnv* env, jobject thiz){
    //这里面是主线程
    MyContext* myContext = new MyContext;
    //myContext->jniEnv = env;
    //提升至全局引用，局部引用会崩溃，使用全局引用需要手动释放全局引用,通过Java的ondestory调用或者JNI_OnUnload
    myContext->instance = env->NewGlobalRef(thiz);
    
    pthread_t pid;
    pthread_create(&pid, nullptr, myThreadTaskAction, myContext);
    pthread_join(pid, nullptr);
}
```



细节点

关于env，jvm，jobject，clazz

这里nativeFun1和nativeFun2为主线程MainActivity1的调用的函数

nativeFun3为主线程MainActivity1的调用的静态函数

nativeFun4为主线程MainActivity1的调用的子线程函数

nativeFun5为主线程MainActivity2的调用的函数

![image-20220208221604086](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220208221604086.png)

![image-20220208220303722](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220208220303722.png)

```java
//MainActivity1.java
public native void nativeFun1();
public native void nativeFun2();
public static native void nativeFun3();
public static native void nativeFun4();
public void clickMethod(View view){
    nativeFun1();
    nativeFun2();
    nativeFun3();
    new Thread(){
        @override
        public void run(){
            super.run();
            nativeFun4();//子线程
        }
    }.start();
}

//MainActivity2.java
public native void nativeFun5();
```



```c++
//native的子线程调用
void* run(void*){
    JNIEnv* newEnv = nullptr;
    ::javaVm->AttachCurrentThread(&newEnv, nullptr);
    LOGD("run jvm地址%p， 当前run函数newEnv地址%p\n", ::javaVm, newEnv);
    ::javaVm->DetachCurrentThread();
    return nullptr;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeFun1(JNIEnv* env, jobject thiz){
    JavaVM* localJavaVm = nullptr;
    env->GetJavaVM(&localJavaVm);
    
    LOGD("nativeFun1当前函数env地址%p， 当前函数jvm地址%p, 当前函数job地址%p, JNI_OnLoad的jvm地址%p\n",
        env, localJavaVm, thiz, ::javaVm);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeFun2(JNIEnv* env, jobject thiz){
    JavaVM* localJavaVm = nullptr;
    env->GetJavaVM(&localJavaVm);
    
    LOGD("nativeFun2当前函数env地址%p， 当前函数jvm地址%p, 当前函数job地址%p, JNI_OnLoad的jvm地址%p\n",
        env, localJavaVm, thiz, ::javaVm);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeFun3(JNIEnv* env, jclass clazz){
    JavaVM* localJavaVm = nullptr;
    env->GetJavaVM(&localJavaVm);
    
    LOGD("nativeFun3当前函数env地址%p， 当前函数jvm地址%p, 当前函数jclazz地址%p, JNI_OnLoad的jvm地址%p\n",
        env, localJavaVm, clazz, ::javaVm);
    
    //run native子线程
    pthread_t pid;
    pthread_create(&pid, nullptr, run, nullptr);
}

//java子线程调用
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeFun4(JNIEnv* env, jclass clazz){
    JavaVM* localJavaVm = nullptr;
    env->GetJavaVM(&localJavaVm);
    
    LOGD("nativeFun4当前函数env地址%p， 当前函数jvm地址%p, 当前函数jclazz地址%p, JNI_OnLoad的jvm地址%p\n",
        env, localJavaVm, clazz, ::javaVm);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_nativeFun5(JNIEnv* env, jobject thiz){
    JavaVM* localJavaVm = nullptr;
    env->GetJavaVM(&localJavaVm);
    
    LOGD("nativeFun5当前函数env地址%p， 当前函数jvm地址%p, 当前函数job地址%p, JNI_OnLoad的jvm地址%p\n",
        env, localJavaVm, thiz, ::javaVm);
}
```

重点：

1.JavaVM全局，绑定当前线程，有且仅有一个地址

2.JNIEnv是与线程绑定，绑定主线程，绑定子线程

3.jobject谁调用JNI函数，谁的实例会给jobject

nativeFun1，nativeFun2，nativeFun3和nativeFun5都为主线程，但是nativeFun5是由MainActivity2的调用的函数，jobject会实例化

jobject和jclazz本质类似，前者每次实例化地址都不同，后者实例化为同一地址
