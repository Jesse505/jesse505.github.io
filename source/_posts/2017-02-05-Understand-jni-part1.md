---
title: Android与JNI（一）实现动态注册
date: 2017-02-05 22:21:15
tags: JNI
categories: [Android,NDK]
---


### 问题背景

Android Studio对JNI的开发是越来越友好了，给Google老大哥大大的赞，AS升级到2.2以后就能通过CMake编译，十分方便，具体可参考官方网站[https://developer.android.com/studio/projects/add-native-code](https://developer.android.com/studio/projects/add-native-code)，至于JNI编译的其他方式，比如ndk-build，可参考这个[哥们的文章](https://blog.csdn.net/mabeijianxi/article/details/68525164)。好了说了这么多，并没有讲到今天文章的重点，哈哈，我们现在开始啦。  
静态注册, 每个 class 都需要使用 javah 生成一个头文件，并且生成的名字很长书写不便；初次调用时需要依据名字搜索对应的JNI层函数来建立关联关系，会影响运行效率。所以这时候动态注册就登场了。

### 动态注册

#### 一.入口函数

System.loadLibrary加载完 JNI 动态库之后，调用JNI_OnLoad函数，开始动态注册。这里就是登记注册的地方。注意：一个.so只能存在一个onload方法。

#### 二.登记本地方法

登记本地方法，作用是将 c/c++ 的函数映射到 JAVA 中，而在这里面起到关键作用的是结构体 JNINativeMethod 。他再jni.h文件中

```C
typedef struct { 
    const char* name; //Java中函数的名字
    const char* signature; //描述了函数的参数和返回值
    void* fnPtr; //函数指针，指向我们调用别人家c++的封装 JNI 函数方法
} JNINativeMethod;
```

<!--more-->

根据这个结构体，我们可以声明一个实例

```C
static JNINativeMethod gMethods[] = {
        { "stringFromJNI", "()Ljava/lang/String;", (void *) stringFromJNI },
};
```

参数分析：  
"stringFromJNI"：Java 中声明的本地方法名；  
(void *)stringFromJNI：映射对象，本地 c/c++ 函数，名字可以与 Java 中声明的本地方法名不一致。  
"()Ljava/lang/String;"：这个应该是最难理解的，也就是结构体中的 signature 。他描述了本地方法的参数和返回值。例如  
　　"()V"  
　　"(II)V"  
　　"(Ljava/lang/String;Ljava/lang/String;)V"  
　　实际上这些字符是与函数的参数类型一一对应的。  
　　"()" 中的字符表示参数，后面的则代表返回值。例如"()V" 就表示void Func();  
　　"(II)V" 表示 void Func(int, int);  
　　具体的每一个字符的对应关系如下  
　　字符    Jni类型        Java类型  
　　V      void         void  
　　Z      jboolean     boolean  
　　I       jint         int  
　　J       jlong        long  
　　D      jdouble       double  
　　F      jfloat            float  
　　B      jbyte            byte  
　　C      jchar           char  
　　S      jshort          short  
　　数组则以"["开始，用两个字符表示  
　　[I     jintArray       int[]  
　　[F     jfloatArray     float[]  
　　[B     jbyteArray     byte[]  
　　[C    jcharArray      char[]  
　　[S    jshortArray      short[]  
　　[D    jdoubleArray    double[]  
　　[J     jlongArray      long[]  
　　[Z    jbooleanArray    boolean[]  
　　上面的都是基本类型，如果参数是 Java 类，则以"L"开头，以";"结尾，中间是用"/"隔开包及类名，而其对应的jni类型则为 jobject，一个例外是 String 类，它对应的 jni 类型为 jstring。  
　　Ljava/lang/String;     jstring    String  
　　Ljava/net/Socket;      jobject    Socket  
　　如果 JAVA 函数位于一个嵌入类（也被称为内部类），则用$作为类名间的分隔符，例如："Landroid/os /FileUtils$FileStatus;"。

　　最终是通过登记所有记录在 JNINativeMethod 结构体中的 JNI 本地方法，登记方法如下：  

```C
/**
 * 向JNI环境注册一个本地方法
 * @param clazz  包含本地方法的Java类
 * @param methods 本地方法描述数组
 * @param nMethods 本地方法个数
 * @return 成功返回0，否则注册失败
 */
jint RegisterNatives(jclass clazz, const JNINativeMethod *methods, jint nMethods);
```


&emsp;&emsp;使用 registerNativeMethods 方法不仅仅是为了改变那丑陋的长方法名，最重要的是可以提高效率，因为当 Java 类透过 VM 呼叫到本地函数时，通常是依靠 VM 去动态寻找动态库中的本地函数(因此它们才需要特定规则的命名格式)，如果某方法需要连续呼叫很多次，则每次都要寻找一遍，所以使用 RegisterNatives 将本地函数向 VM 进行登记，可以让其更有效率的找到函数。  
&emsp;&emsp;registerNativeMethods 方法的另一个重要用途是，运行时动态调整本地函数与 Java 函数值之间的映射关系，只需要多次调用 registerNativeMethods 方法，并传入不同的映射表参数即可。      

#### 三.调用JNI_OnUnload结束

在这里可以进行一些收尾工作，比如对象的回收啊。

#### 四.代码实现

```C
#include <jni.h>
#include <string>

#include <android/log.h>

#define TAG "zhaoyifei"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define NELEM(x) ((int) (sizeof(x) / sizeof((x)[0])))


JNIEXPORT jstring JNICALL helloFromJNI(JNIEnv* env, jclass clazz)
{
    LOGI("动态加载完成啦");
    return env->NewStringUTF("hello world returned.");
}

static JNINativeMethod gMethods[] = {
        { "helloFromJNI", "()Ljava/lang/String;", (void *)helloFromJNI },
};

static int registerNatives(JNIEnv* env) {
    LOGI("registerNatives begin");
    jclass  clazz;
    clazz = env -> FindClass("com/example/github/jnidemo/MainActivity");

    if (clazz == NULL) {
        LOGI("clazz is null");
        return JNI_FALSE;
    }

    if (env ->RegisterNatives(clazz, gMethods, NELEM(gMethods)) != JNI_OK) {
        LOGI("RegisterNatives error");
        return JNI_FALSE;
    }

    return JNI_TRUE;
}

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {

    LOGI("jni_OnLoad begin");

    JNIEnv *env = NULL;
    jint result = -1;

    if (vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
        LOGI("ERROR: GetEnv failed\n");
        return result;
    }
    if (registerNatives(env) < 0) {
        LOGI("ERROR: registerNatives failed\n");
        return result;
    }

    return JNI_VERSION_1_4;
}

void JNI_OnUnload(JavaVM* vm, void *reserved)
{
    return;
}
```
 



