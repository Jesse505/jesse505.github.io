---
title: Android与JNI（四）C操作Java
date: 2017-02-28 21:20:43
tags: JNI
categories: [Android,NDK]
---

既然Java层通过Native方法可以调用C代码，那么其实C代码也是可以调用Java代码的，JNI封装了一系列方法来达到调用java的目的，主要有以下几种常见场景：  

1. 读取设置成员属性；   
2. 读取设置静态属性；  
3. 调用Java成员方法；  
4. 调用Java静态方法； 

<!--more--> 

#### 1.读取设置成员属性

这部分还是直接上代码了，注释都很清楚，后续文章会把JNI的常用函数列举出来，在这里就不一一解释了。

Java层代码

```java
private String str = "成员属性";
public native void accessField();
```

Native层代码实现

```C
//设置成员属性
JNIEXPORT void JNICALL accessField(JNIEnv *env, jobject obj) {
    //获取类引用
    jclass cls = env->GetObjectClass(obj);
    //获取属性ID
    jfieldID fid = env->GetFieldID(cls, "str", "Ljava/lang/String;");
    if (NULL == fid) {
        return;
    }
    //获取属性值
    jstring jstr_value = (jstring)env->GetObjectField(obj, fid);
    const char *value = env->GetStringUTFChars(jstr_value, NULL);
    LOGI("成员属性：%s", value);
    env->ReleaseStringUTFChars(jstr_value, value);
    jstring cValue = env->NewStringUTF("修改后的成员属性");
    //设置属性值
    env->SetObjectField(obj, fid, cValue);
}
```


#### 2.读取设置静态属性

Java层代码

```java
private static String static_str= "静态属性";
public native void accessStaticField();
```

Native层代码实现

```C
//设置静态属性
JNIEXPORT void JNICALL accessStaticField(JNIEnv *env, jobject obj) {
    //获取类引用
    jclass cls = env->GetObjectClass(obj);
    //获取静态属性ID
    jfieldID fid = env->GetStaticFieldID(cls, "static_str", "Ljava/lang/String;");
    if (NULL == fid) {
        return;
    }
    jstring jstr_value = (jstring)env->GetStaticObjectField(cls, fid);
    const char *value = env->GetStringUTFChars(jstr_value, NULL);
    LOGI("静态成员属性：%s", value);
    env->ReleaseStringUTFChars(jstr_value, value);
    jstring cValue = env->NewStringUTF("修改后的静态属性");
    //设置静态属性值
    env->SetStaticObjectField(cls, fid, cValue);
}
```

#### 3.调用Java成员方法

Java层代码

```java
private int sum(int a, int b) {
    return a + b;
}
public native void accessMethod();
```

Native层代码实现

```C
//调用成员方法
JNIEXPORT void JNICALL accessMethod(JNIEnv *env, jobject obj) {
    //获取类引用
    jclass cls = env->GetObjectClass(obj);
    //获取成员方法ID
    jmethodID mid = env->GetMethodID(cls, "sum", "(II)I");
    if (NULL == mid) {
        LOGI("accessMethod failed");
        return;
    }
    int sum = env->CallIntMethod(obj, mid, 1, 2);
    LOGI("sum: %d", sum);
}
```

#### 4.调用Java静态方法

Java层代码

```java
private static int diff(int a, int b) {
    return a - b;
}
public native void accessStaticMethod();
```

Native层代码实现

```C
//调用静态方法
JNIEXPORT void JNICALL accessStaticMethod(JNIEnv *env, jobject obj) {
    //获取类引用
    jclass cls = env->GetObjectClass(obj);
    //获取静态方法ID
    jmethodID mid = env->GetStaticMethodID(cls, "diff", "(II)I");
    if (NULL == mid) {
        return;
    }
    int sum = env->CallStaticIntMethod(cls, mid, 1, 2);
    LOGI("diff: %d", sum);
}
```


[完整代码地址](https://github.com/Jesse505/BlogDemo/tree/master/JniDemo)，希望这篇文章能帮助到你，喜欢记得star下，谢谢！
