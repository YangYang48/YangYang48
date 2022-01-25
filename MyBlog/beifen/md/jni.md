jni



## 预处理器

 

```c++
//预处理器
#include 	//导入头文件
#if 		//if判断操作
#elif		//else if
#else		//else
#endif		//结束if
#define		//定义一个宏
#ifdef		//如果定义了这个宏,if的范畴，必须跟上#endif,是否定义了这个宏
#ifndef		//如果没有定义这个宏
#undef		//取消宏定义
#pragma		//设定编译器的状态
```



```c++
#include <iostream>
using namespace std;

int main(){
    cout << "宏" << endl;
#if 0
    cout << "不走这边" << endl;
#elif 1
    cout << "走这边" << endl;
#endif
    
    return 0;
}
```



```c++
//definecpp.h
#ifndef DEFINECPP_H
#define DEFINECPP_H

#ifndef isRelease
#define isRelease 1

#if isRelease == true
#define RELEASE
#elif isRelease == false
#define DEBUG

#endif
#endif
#endif
```



```c++
#include <iostream>
#include "definecpp.h"
using namespace std;

int main(){
#ifdef DEBUG
    cout << "测试环境" << endl;
#else RELEASE
    cout << "正式环境" << endl;
#endif    
    return 0;
}
```

宏的取消

```c++
//undef xx，xx宏的取消
#include <iostream>
using namespace std;

int main(){
#ifndef DERRY
#define DERRY
#ifdef DERRY
    for(int i = 0; i < 6; i++){
        cout << "Derry 1" << endl;
    }
#endif
#ifdef DERRY
    for(int i = 0; i < 6; i++){
        cout << "Derry 2" << endl;
    }
#endif

#undef DERRY//取消宏
    
#ifdef DERRY
    cout << "定义了这个宏" << endl;
#else
    cout << "没有定义宏" << endl;
#endif
    
#endif    
    
    return 0;
}
```

宏变量

```c++
#include <iostream>
using namespace std;

#define VALUE_I 9527
#define VALUE_S "AAA"
#define VALUE_F 123.4f

int main(){
    int i = VALUE_I;
    string s = VALUE_S;
    float f = VALUE_F;
    
    return 0;
}
```

宏函数

```c++
#include <iostream>
using namespace std;

//参数列表无需类型，返回值看不到,这个SHOW类型简化版的模板函数
#define SHOW(V) cout << V << endl;
#define ADD(n1, n2) n1 + n2
//宏函数的注意事项。需要加括号
#define CHE(n1. n2) ((n1) * (n2))
int main(){
    SHOW(8);
    SHOW(8.8f);
    SHOW(8.99);
    cout << ADD(1, 2) << endl;
    return 0;
}
```

宏函数的优缺点

优点

1.文本替换不会造成函数的调用开销（开辟栈空间，形参压栈，函数弹栈释放）

缺点

1.会导致代码体积增大



一、预处理

宏定义的展开，宏定义的替换等

二、预编译

主要用于代码检查

三、汇编阶段

生成。o文件

四、链接阶段

生成静态库和动态库



## JNI

推荐书籍JNI开发指南

jni（JAVA native interface）先出来，ndk后出来，ndk包含（java jni，c++，gcc，各种工具链）。它更多是桥梁，连接JAVA和C++。

JNI 概述:
定义： Java Native Interface， 即 Java 本地接口
作用： 使得 Java 与 本地其他类型语言（如 C、 C++） 交互
JNI 是 Java 调用 Native 语言的一种特性
JNI 是属于 Java 的， 与 Android 无直接关系
实际中的驱动都是 C/C++开发的,通过 JNI,Java 可以调用 C/c++实现的驱动， 从
而扩展 Java 虚拟机的能力。 另外， 在高效率的数学运算、 游戏的实时渲染、 音
视频的编码和解码等方面， 一般都是用 C 开发的
（Java 代码 里调用 C/C++等语言代码 或 C/C++代码调用 Java 代码）  

![image-20220116163631130](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116163631130.png)

 通过javah生成头文件的jni函数

![image-20220116163810560](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116163810560.png)

```c++
//头文件里面定义函数如果是c++，采用c的方式，不允许函数重载，函数名一样的问题
#ifdef __cplusplus
extern "C" {
#endif
    
#ifdef __cplusplus    
}
#endif
```



```c++
//native-lib.cpp
#include "com_derry_as_jni_project_MainActivity.h"

//JNIEXPORT标记该方法可以被外部调用(as不加没问题，vs必须加)linux平台可不加，win必须加
//jstring java---native转换用得
//这里的1也代表下划线Java_com_derry_as_1jni_1project
//Java_包名_类名_方法名
//非静态函数参数为jobject jobj，谁调用就是谁的实例，MainActivity.this
//静态函数参数为jclass clazz，谁调用就是谁的实例，class MainActivity.class
extern "C" JNIEXPORT jstring JNICALL Java_com_derry_as_1jni_1project_MainActivity_getStringPwd(JNInv* env, jobject jobj){
    //无论是c还是c++最终调用的是C的JNINativeInterface，需要加上采用C的方式extern "C"
    
} 
```

![image-20220116165128022](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116165128022.png)

[【Android NDK 开发】JNI 方法解析 ( JNIEXPORT 与 JNICALL 宏定义作用 )](https://blog.csdn.net/shulianghan/article/details/104072587)

> JNIEXPORT**该声明的作用是保证在本动态库中声明的方法 , 能够在其他项目中可以被调用 ;**
>
>  JNICALL**__stdcall 用于 定义 函数入栈规则 ( 从右到左 ) , 和 堆栈清理规则 ;**

NDK集成的库#include <android/log.h>

![image-20220116173950243](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116173950243.png)

签名规则

![image-20220116175041110](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116175041110.png)

javap -s -p xxx.class //-s输出xxx.class的所有属性和方法的签名， -p忽略私有公开的所有属性方法，全部输出

[通过javap命令分析java汇编指令](https://www.jianshu.com/p/6a8997560b05)

[官网-javap - The Java Class File Disassembler](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/javap.html)

```java
D:\android\open-jdk\openjdk-17.0.1_windows-x64_bin\jdk-17.0.1\bin\javap -s -p MainActivity.class
Compiled from "MainActivity.java"
public class com.derry.as_jni_project.MainActivity extends androidx.appcompat.app.AppCompatActivity {
  public static final int A;
    descriptor: I
  public java.lang.String name;
    descriptor: Ljava/lang/String;
  public static int age;
    descriptor: I
  public com.derry.as_jni_project.MainActivity();
    descriptor: ()V

  public native java.lang.String getStringPwd();
    descriptor: ()Ljava/lang/String;

  public static native java.lang.String getStringPwd2();
    descriptor: ()Ljava/lang/String;

  public native void changeName();
    descriptor: ()V

  public static native void changeAge();
    descriptor: ()V

  public native void callAddMethod();
    descriptor: ()V

  public int add(int, int);
    descriptor: (II)I

  protected void onCreate(android.os.Bundle);
    descriptor: (Landroid/os/Bundle;)V

  static {};
    descriptor: ()V
}
```

**开始native函数交互**

定义java类，调用native方法

```java
package com.derry.as_jni_project;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.widget.TextView;

// 生成头文件：javah com.derry.as_jni_project.MainActivity
public class MainActivity extends AppCompatActivity {

    static {
        System.loadLibrary("native-lib");
    }

    public String name = "Derry"; // 签名：Ljava/lang/String;

    public static int age = 29; // 签名：I
    // -------------  交互操作 JNI
    public native void changeName();
    public static native void changeAge();
    public native void callAddMethod();


    // 专门写一个函数，给native成调用
    public int add(int number1, int number2) {
        return number1 + number2 + 8;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

         // Example of a call to a native method
         TextView tv = findViewById(R.id.sample_text);
        changeName();
        tv.setText(name);

        changeAge();
        tv.setText("" + age);

        callAddMethod();
    }

}
```

定义native方法，修改java层中的属性，需要先获取属性名`GetFieldID`

```c++
//-------------------------------------
//非静态获取和修改，java端的属性
//非静态方法，通过obj获取clazz对象
jclass GetObjectClass(jobject obj)
// jfieldID GetFieldID(MainActivity.class, 属性名, 属性的签名)
jfieldID GetFieldID(jclass clazz, const char* name, const char* sig);
jobject GetObjectField(jobject obj, jfieldID fieldID);
//将jstring转化成char* 默认后者为NULL
 const char* GetStringUTFChars(jstring string, jboolean* isCopy);
//将char*字符串转化成jstring
jstring NewStringUTF(const char* bytes);
//设置属性的新值，第三个参数必须为jstring，不能为char*
void SetObjectField(jobject obj, jfieldID fieldID, jobject value);
//-------------------------------------
//静态获取和修改，java端的属性
//静态获取,不用再获取clazz了
//获取静态的属性id
jfieldID GetStaticFieldID(jclass clazz, const char* name, const char* sig);   
//获取int类的java属性，返回jint型
jint GetStaticIntField(jclass clazz, jfieldID fieldID); 
//设置静态的jint属性值给java的int类型
void SetStaticIntField(jclass clazz, jfieldID fieldID, jint value);
//-------------------------------------
//非静态获取，java端的方法
//GetMethodID(MainActivity.class, 方法名, 方法的签名)
jmethodID GetMethodID(jclass clazz, const char* name, const char* sig);
//真实调用java端的非静态方法，CallIntMethod实际上是返回值来jint类型的
CALL_TYPE(jint, Int)
//展开宏定义    
#define CALL_TYPE(_jtype, _jname)                                           \
    CALL_TYPE_METHOD(_jtype, _jname)                                        \
    CALL_TYPE_METHODV(_jtype, _jname)                                       \
    CALL_TYPE_METHODA(_jtype, _jname)

#define CALL_TYPE_METHOD(_jtype, _jname)                                \
_jtype Call##_jname##Method(jobject obj, jmethodID methodID, ...)       \
{                                                                       \
    _jtype result;                                                      \
    va_list args;                                                       \
    va_start(args, methodID);                                           \
    result = functions->Call##_jname##MethodV(this, obj, methodID,      \
                args);                                                  \
    va_end(args);                                                       \
    return result;                                                      \
}
//这里调用了CALL_TYPE_METHOD(jint, Int)实际为
CALLIntMETHOD(jobject obj, jmethodID methodID, ...);
```

获取name，从java的String先转换成jstring，然后将jstring转换成char*

设置name，从c端的char*转换到jstring，然后通过SetObjectField来设置到java端

唯一区别c中的字符串的，jint可以直接操作。

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_changeName(JNIEnv *env, jobject thiz) {
   // 获取class
   jclass j_cls = env->GetObjectClass(thiz);
   // 获取属性  L对象类型 都需要L
   jfieldID j_fid = env->GetFieldID(j_cls, "name", "Ljava/lang/String;");
   // 转换工作
   jstring j_str = static_cast<jstring>(env->GetObjectField(thiz ,j_fid));
   // 打印字符串  目标
   char * c_str = const_cast<char *>(env->GetStringUTFChars(j_str, NULL));
   LOGD("native : %s\n", c_str);

    // 修改成 Beyond
   jstring jName = env->NewStringUTF("Beyond");
   env->SetObjectField(thiz, j_fid, jName);
}

//java静态的属性修改
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_changeAge(JNIEnv *env, jclass jcls) {
   jfieldID j_fid = env->GetStaticFieldID(jcls, "age", "I");
   jint age = env->GetStaticIntField(jcls, j_fid);
   age += 10;
   env->SetStaticIntField(jcls, j_fid, age);
}

//java方法名的调用
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_callAddMethod(JNIEnv *env, jobject job) {
    // 自己得到 MainActivity.class
    jclass  mainActivityClass = env->GetObjectClass(job);
    // GetMethodID(MainActivity.class, 方法名, 方法的签名)
   jmethodID j_mid = env->GetMethodID(mainActivityClass, "add", "(II)I");
   // 调用 Java的方法
   jint sum = env->CallIntMethod(job, j_mid, 3, 3);
   LOGE("sum result:%d", sum);
}
```



## NDK开发

as没有提示c++语法，需要打开ndk插件开关

![image-20220116214102381](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116214102381.png)



```java
//java里的so加载方式
static {
    //System.load("D:/xxx/xxxx/xxxx/native-lib");//这种可以是绝对路径加载动态库
    System.loadLibrary("native-lib");//会去寻找apk里面的/lib/libnative-lib.so
}
```

基本元素

```c++
//这里接收的是一个数组，Elements为数组的所有元素集合
jint* GetIntArrayElements(jintArray array, jboolean* isCopy);
//这里是Element，不是上面的Elements，是获取Object对象单个元素
jobject GetObjectArrayElement(jobjectArray array, jsize index);
//第三个参数，实际有三种模式
//mode为0			刷新JAVA数组，并释放C++层数组
//mode为JNI_COMMIT=1	刷新JAVA数组，不释放C++层数组
//mode为JNI_ABORT=2	只释放C++层数组
void ReleaseIntArrayElements(jintArray array, jint* elems, jint mode);
```



```c++
// jni == java 数据类型对应
// jint == int
// jstring == String
// jintArray == int[]
// jobjectArray == 引用类型对象，例如 String[]   Test[]   Student[]  Person[]
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_testArrayAction(JNIEnv *env, jobject thiz,
                                                             jint count,
                                                             jstring text_info,
                                                             jintArray ints,
                                                             jobjectArray strs) {
    int countInt = count;//jint类型就是int
    //env就相当于操纵杆，去操纵jvm
    const char* textInfo = env->GetStringUTFChars(text_info, NULL); 
    
    int* jniArray = env->GetIntArrayElements(ints, NULL);
    
    jsize size = env->GetArrayLength(ints);
    //这里的修改不会同步到java层
    for(int i = 0; i < size; i++){
        *(jintArray) += 100;
    }
    //env就相当于操纵杆，去操纵jvm,将数据刷新到java层

    env-> ReleaseIntArrayElements(ints, jintArray, 0);
    
    //获取string引用对象的数组
    jsize strSize = env->GetArrayLength(strs);
    for(int i = 0; i < strSize; i++){
        //这里只能获取单个元素
        jstring jobj = static_cast<jstring>(env->GetObjectArrayElement(strs, i));
        const char* jobjchar = env->GetStringUTFChars(jobj, NULL);
        
        //释放jstring,可以不释放，native会占用大量内存，释放可以减少在native的回收率
        env->ReleaseStringUTFChars(jobj, jobjchar);
    }
}
```

![image-20220116222917554](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220116222917554.png)

```c++
/*
 * Manifest constants.
 */
#define JNI_FALSE   0
#define JNI_TRUE    1

#define JNI_VERSION_1_1 0x00010001
#define JNI_VERSION_1_2 0x00010002
#define JNI_VERSION_1_4 0x00010004
#define JNI_VERSION_1_6 0x00010006

#define JNI_OK          (0)         /* no error */
#define JNI_ERR         (-1)        /* generic error */
#define JNI_EDETACHED   (-2)        /* thread detached from the VM */
#define JNI_EVERSION    (-3)        /* JNI version error */
#define JNI_ENOMEM      (-4)        /* Out of memory */
#define JNI_EEXIST      (-5)        /* VM already created */
#define JNI_EINVAL      (-6)        /* Invalid argument */

#define JNI_COMMIT      1           /* copy content, do not free buffer */
#define JNI_ABORT       2           /* free buffer w/o copying back */
```



> "GetStringChars"和"GetStringUTFChar"的第三个参数的需要额外的解释:
> const jchar *
> GetStringChars(JNIEnv *env, jstring str, jboolean *isCopy) ;
>
> 
>
> 当来自"GetStringChars"返回时，通过"isCopy"被设置为"JNI_TRUE",返回的字符串是个在原始"java.lang.String"实例的字符串的一个复制，内存定位指向它。通过"isCopy"被设置为"JNI_FALSE",返回字符串时一个直接指向在原始的"java.lang.String"实例中的字符串，内存定位指向它。当通过"isCopy"被设置为"JNI_FALSE"，内存定位指向的字符串时，本地代码必须不能修改返回的字符串的内容。违反了这个规则将引起原始"java.lang.String"实例也被修改。这破坏了"java.lang.String"永恒不可变性。
>
>  
>
> 你最常传递"NULL"作为"isCopy"参数，因为你不关心Java虚拟机是否返回在"java.lang.String"实例中的一个复制字符串和直接指向原始字符串。
>
>  
>
> 一般不可能预测虚拟机将是不是复制字串对一个给定的"java.lang.String"实例。因此程序员必须假设函数如"GetStringChars"可以使用对应的空间和时间到在"java.lang.String"实例中的大量字符上。在一个典型Java虚拟器实现中，垃圾收集器搬迁堆上的对象。一旦直接指向一个"java.lang.String"实例的指针被传递给本地代码，垃圾收集器再不能搬移这"java.lang.String"实例了。换另一种说法，虚拟器必须固定住"java.lang.String"实例。因为过多地固定导致内存碎片，虚拟器实现可以，为灭个单独地"GetStringChars"调用，酌情地决定是复制字串还是固定实例。
>
>  
>
> 不要忘记调用"ReleaseStringChars",当你不再需要访问来自"GetStringChars"返回的"string"元素的时候。"ReleaseStringChars"调用是必须的，无论"GetStringChars"设置"* isCopy"为"JNI_TRUE"或"JNI_FALSE"。"ReleaseStringChars"释放副本，或解除实例的固定，这依赖"GetStringChars"是返回一个副本还是指向实例。

 

基本对象

```java
//java定义student类
class Student {
    private final static String TAG = Student.class.getSimpleName();

    public String name;
    public int age;
    public String getName() {
        return name;
    }

    public void setName(String name) {
        Log.d(TAG, "Java setName name:" + name);
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        Log.d(TAG, "Java setAge age:" + age);
        this.age = age;
    }

    public static void showInfo(String info) {
        Log.d(TAG, "showInfo info:" + info);
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

public native void testArrayAction(int count, String textInfo, int[] ints, String[] strs); // String引用类型，玩数组
public native void putObject(Student student, String str); // 传递引用类型，传递对象
```

jni来进行修改对象

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_putObject(JNIEnv *env,
                                                       jobject thiz,
                                                       jobject student,
                                                       jstring str) {
    const char* strChar = env->GetStringUTFChars(str, NULL);
    
    //上面定义的strChar指针需要释放
    env->ReleaseStringUTFChars(str, strChar);
    //=======================
    //类似反射查找到setName
    //1.寻找类Student 找到jclass
    //第一种方式，通过包名查找
    //jclass studentClass = env->FindClass("com/derry/as_jni_project/Student");
    //通过jobject对象查找
    jclass studentClass = env->GetObjectClass(student);
    
    //2.Student类里面的函数规则，涉及签名
    jmethodID setName = env->GetMethod(studentClass, "setName", "(Ljava/lang/String;)V");
    jmethodID getName = env->GetMethod(studentClass, "getName", "()Ljava/lang/String;");
     jmethodID showInfo = env->GetStaticMethodID(studentClass, "showInfo", "(Ljava/lang/String;)V");
    
    //3.调用setName
    jstring value = env->NewStringUTF("AAA");
    env->CallVoidMethod(student, setName, value);
    //4.调用getName
    jstring getNameResult = static_cast<jstring>(env->CallObjectMethod(student, getName));
    const char* getNameValue = env->GetStringUTFChars(getNameResult, NULL);
    //5.调用静态showInfo
    jstring jstringValue = env->NewStringUTF("静态方法你好，我是c++");
    env->CallStaticVoidMethod(studentClass, showInfo, jstringValue);
}
```

凭空创建java对象,**这种方式的性能比java的反射要高**

```java
public class Person {
    private static final String TAG = Person.class.getSimpleName();
    
    public Student student;
    
    public void setStudent(Student student) {
        Log.d(TAG, "call setStudent student:" + student.toString());
        this.student = student;
    }

    public static void putStudent(Student student) {
        Log.d(TAG, "call static putStudent student:" + student.toString());
    }
}

public native void insertObject(); // 凭空创建Java对象
```



```c++
/*
AllocObject和NewObject区别，两个都是实例化对象，后者会调用构造函数
jobject AllocObject(jclass clazz)
{ return functions->AllocObject(this, clazz); }

jobject NewObject(jclass clazz, jmethodID methodID, ...)
{
    va_list args;
    va_start(args, methodID);
    jobject result = functions->NewObjectV(this, clazz, methodID, args);
    va_end(args);
    return result;
}
*/
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_insertObject(JNIEnv *env, jobject thiz) {
    //1.通过包名+类名的方式拿到Student class,凭空拿class
    //因为这里没有对象不能用上述env->GetObjectClass(student)获取
    const char* studentstr = "com/derry/as_jni_project/Student";
    jclass studentClass = env->FindClass(studentstr);
    
    //2.通过student的class，实例化Student对象,类似c++new对象
    //这里的AllocObject只实例化对象，不会调用对象的构造函数
    jobject studentObj = env->AllocObject(studentClass);
    //这种方式会调用java对象的构造函数
    //jobject studentObj = env->NewObject(studentClass, ...);
    
    //3.方法签名规则
    jmethodID setName = env->GetMethodID(studentClass, "setName", "(Ljava/lang/String;)V");
    jmethodID setAge = env->GetMethodID(studentClass, "setAge", "(I)V");
    
    //4.调用方法
    jstring strValue = env->NewStringUTF("Derry");
    env->CallVoidMethod(studentObj, setName, strValue);
    env->CallVoidMethod(studentObj, setAge, 99);
    
    //==========================
    //重复上述1-4，通过包名+类名获取class
    const char* personstr = "com/derry/as_jni_project/Person";
    jclass personClass = env->FindClass(personstr);
    
    jobject personObj = env->AllocObject(personClass);
    
    jmethodID setStudent = env->GetMethodID(personClass, "setStudent", 
                                           "(Lcom/derry/as_jni_project/Student;)V");
    env->CallVoidMethod(personObj, setStudent, studentObj);
    
    //需要记得释放掉
    //比如const char* str = env->GetStringUTFChars();
    //对应需要释放env->ReleaseStringUTFChars()
    env->DeleteLocalRef(studentClass);
    env->DeleteLocalRef(studentObj);
    env->DeleteLocalRef(personClass);
    env->DeleteLocalRef(personObj);
    
    //上述jobject jclass jstring也可不释放
    //局部引用，函数结束后会自动释放。建议手动释放
    //这里区别于C++的堆和栈区，这里只有局部引用和全局引用
}
```

关于释放内存[Android NDK 开发中正确释放 JNI 对象](https://www.jianshu.com/p/5cde114159d4)

![image-20220118224209328](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220118224209328.png)

jni属于JVM的一部分



对象高阶--引用

定义一个dog类

```java
public class Dog { // NewObject 调用我们的构造函数

    public Dog() { // <init>
        Log.d("Dog", "Dog init...");
    }

    public Dog(int n1) { // <init>
        Log.d("Dog", "Dog init... n1:" + n1);
    }

    public Dog(int n1, int n2) { // <init>
        Log.d("Dog", "Dog init... n1:" + n1 + " n2:" + n2);
    }

    public Dog(int n1, int n2, int n3) { // <init>
        Log.d("Dog", "Dog init... n1:" + n1 + " n2:" + n2 + " n3:" + n3);
    }
}

public native void testQuote(); // 测试引用
public native void delQuote(); // 释放全局引用
```



```c++
//<init>是构造函数，靠签名决定的
/*
jobject NewObject(jclass clazz, jmethodID methodID, ...)
{
    va_list args;
    va_start(args, methodID);
    jobject result = functions->NewObjectV(this, clazz, methodID, args);
    va_end(args);
    return result;
}
*/
jclass dogClass; // 你以为这个是全局引用，实际上他还是局部引用

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_testQuote(JNIEnv *env, jobject thiz) {
    if (NULL == dogClass){
        const char* dogStr = "com/derry/as_jni_project/Dog";
        dogClass = env->FindClass(dogStr);
    }
    
    //构造函数1
    //<init> V是不会变的
    jmethod init = env->GetMethodID(dogClass, "<init>", "()V");
    //下面这种方式会调用构造函数
    jobject dog = env->NewObject(dogClass, init);
    
    init = env->GetMethodID(dogClass, "<init>", "(I)V");
    dog = env->NewObject(dogClass, init, 100);
    
    init = env->GetMethodID(dogClass, "<init>", "(II)V");
    dog = env->NewObject(dogClass, init, 100, 200);
    
    init = env->GetMethodID(dogClass, "<init>", "(III)V");
    dog = env->NewObject(dogClass, init, 100, 200, 300);
    
    //释放dog引用
    env->DeleteLocalRef(dog);
}
```

上面这段执行没有问题，但是第二次点击就会error，原因是函数外部声明的dogClass是局部变量，函数结束之后就会释放掉，dogClass就为野指针，那么第二次就不会初始化，调用GetMethodID时候就会error。



解决思路：

1.将dogClass在函数结束前及时释放

```c++
env->DeleteLocalRef(dogClass);
dogClass = NULL;
```

2.将dogClass升级成全局引用，不调用释放会一直存在



修改后，需要手动释放

```c++
jclass dogClass; // 你以为这个是全局引用，实际上他还是局部引用

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_testQuote(JNIEnv *env, jobject thiz) {
    if (NULL == dogClass){
        const char* dogStr = "com/derry/as_jni_project/Dog";
        jclass temp = env->FindClass(dogStr);
        dogClass = static_cast<jclass>(env->NewGlobalRef(temp));//升级为全局引用
        env->DeleteLocalRef(temp);
    }
    
    //构造函数1
    //<init> V是不会变的
    jmethod init = env->GetMethodID(dogClass, "<init>", "()V");
    //下面这种方式会调用构造函数
    jobject dog = env->NewObject(dogClass, init);
    
    init = env->GetMethodID(dogClass, "<init>", "(I)V");
    dog = env->NewObject(dogClass, init, 100);
    
    init = env->GetMethodID(dogClass, "<init>", "(II)V");
    dog = env->NewObject(dogClass, init, 100, 200);
    
    init = env->GetMethodID(dogClass, "<init>", "(III)V");
    dog = env->NewObject(dogClass, init, 100, 200, 300);
    
    //释放dog引用
    env->DeleteLocalRef(dog);
}

// 手动释放全局引用,一般在destory的时候使用
extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_delQuote(JNIEnv *env, jobject thiz) {
   if (dogClass != NULL) {
       LOGE("全局引用释放完毕，上面的按钮已经失去全局引用，再次点击会报错");
       env->DeleteGlobalRef(dogClass);
       // 指向NULL的地址，不要去成为悬空指针，为了好判断悬空指针的出现，不加在调用testQuote会闪退，必须加
       dogClass = NULL;
   }
}
```



如果放在Person类放在其他类的内部

![image-20220118230056949](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220118230056949.png)

kotlin版的native写法

![image-20220118230236178](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220118230236178.png)

## extern

```c++
//声明
extern int age;
extern void show();

extern "C"
JNIEXPORT void JNICALL
Java_com_derry_as_1jni_1project_MainActivity_delQuote(JNIEnv *env, jobject thiz) {
   if (dogClass != NULL) {
       LOGE("全局引用释放完毕，上面的按钮已经失去全局引用");
       env->DeleteGlobalRef(dogClass);
       // 指向NULL的地址，不要去成为悬空指针，为了好判断悬空指针的出现，不加在调用testQuote会闪退，必须加
       dogClass = NULL;
   }
    
   show(); 
}
```

真实的show()和age可以定义在另外的文件里

```c++
//Test.cpp
int age = 99;

void show(){
    //5000行代码
    LOGI("show run age = %d\n", age);
}
```

如果出错，需要在cmakeLists.txt中添加文件

![image-20220120221805134](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220120221805134.png)
