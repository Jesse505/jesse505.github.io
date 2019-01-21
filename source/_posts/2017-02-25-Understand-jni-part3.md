---
title: Android与JNI（三）复杂对象的传递
date: 2017-02-25 22:40:34
tags: JNI
categories: [Android,NDK]
---



关于JNI方面的代码实例讲解太少了，所以我决定自己整理一份，这篇文章我们主要讲讲Android Java层与Native层之间复杂对象传递的主要场景，主要分为：   

1. Java层向Native层传递
2. Native层向Java层传递

<!--more-->

### Java层向Native层传递

##### 1. Java层传递数组给Native

直接上代码，注释写的都很清楚，自己照着敲一遍基本就明白了：  

Java层原型方法：

```java
public native void inputSortArray(int[] array);
```

Native层方法实现：

```C
// 比较两个数的大小
int compare(const void *a, const void *b) {
    return (*(int *) a - *(int *) b);
}

//Java层传递数组给Native
JNIEXPORT void JNICALL inputSortArray(JNIEnv *env, jobject obj, jintArray jintArray) {
    LOGI("inputSortArray begin");
    //获取数组中的元素
    jint *int_arr = env->GetIntArrayElements(jintArray, NULL);
    //获取数组的长度
    int length = env->GetArrayLength(jintArray);
    //快速排序
    qsort(int_arr, length, sizeof(int), compare);
    // 同步操作后的数组内存到java中
    // 最后一个参数mode的解释
    // 0：Java的数组更新同步，然后释放C/C++的数组内存
    // JNI_ABORT：Java的数组不会更新同步，但是释放C/C++的数组内存
    // JNI_COMMIT：Java的数组更新同步，不释放C/C++的数组内存（但是函数执行完了局部的变量还是会释放掉）
    env->ReleaseIntArrayElements(jintArray, int_arr, 0);
}
```

##### 2. Java层传递复杂对象给Native

Java层原型方法：

```java
public native void printStuInfoAtNative(Student stu);
```

Student是一个静态内部类：

```java
    public static class Student {
        int age = 21;
        String name = "zhaoyifei";

        public Student() {
        }

        public Student(int age, String name) {
            this.age = age;
            this.name = name;
        }

        @Override
        public String toString() {
            return "name:" + name + " age:" + age;
        }
    }
```

Native层方法实现：

```C
//Java层传递复杂对象给Native
JNIEXPORT void JNICALL printStuInfoAtNative(JNIEnv *env, jobject obj, jobject stu_obj) {
    jclass stu_cls = env->GetObjectClass(stu_obj);//获得student类引用
    if (NULL == stu_cls) {
        LOGI("GetObjectClass failed");
    }
    jfieldID ageFieldId = env->GetFieldID(stu_cls, "age", "I"); //获得属性ID
    jfieldID nameFieldId = env->GetFieldID(stu_cls, "name", "Ljava/lang/String;"); //获得属性ID
    jint age = env->GetIntField(stu_obj, ageFieldId); //获得属性值
    jstring name = (jstring) env->GetObjectField(stu_obj, nameFieldId); //获得属性值
    const char *c_name = env->GetStringUTFChars(name, NULL); //转换为char*
    LOGI("name: %s, age: %d", c_name, (int) age);
    env->ReleaseStringUTFChars(name, c_name); //释放引用
    return;
}
```


### Native层向Java层传递

##### 1.Native层传递数组给Java层

Java层原型方法：

```java
public native int[] getArray(int len);
```

Native层方法实现：

```C
//Native层传递数组给Java层
JNIEXPORT jintArray JNICALL getArray(JNIEnv *env, jobject obj, jint len) {
    jintArray jintArray = env->NewIntArray(len);
    jint *elements = env->GetIntArrayElements(jintArray, NULL);
    for (int i = 0; i < len; ++i) {
        elements[i] = i;
    }
    env->ReleaseIntArrayElements(jintArray, elements, 0);
    return jintArray;
}
```

##### 2.Native层传递二维数组给Java层

Java层原型方法：

```java
public native int[][] getTwoArray(int len);
```

Native层方法实现：

```C
//Native层传递二维数组给Java层
JNIEXPORT jobjectArray JNICALL getTwoArray(JNIEnv *env, jobject obj, jint len) {
    //获得一维数组的类引用，即jintArray类型
    jclass intArrayClass = env->FindClass("[I");
    //构造指向jintArray一维数组的对象数组，数组长度为len
    jobjectArray objectIntArray = env->NewObjectArray(len, intArrayClass, NULL);
    for (int i = 0; i < len; ++i) {
        jintArray intArray = env->NewIntArray(len);
        //初始化一个容器，假设len<10
        jint temp[10];
        for (int j = 0; j < len; ++j) {
            temp[j] = j + i;
        }
        //设置一维数组的值
        env->SetIntArrayRegion(intArray, 0, len, temp);
        //设置对象数组的值
        env->SetObjectArrayElement(objectIntArray, i, intArray);
        //删除局部引用
        env->DeleteLocalRef(intArray);
    }
    return objectIntArray;
}
```

##### 3.Native层传递复杂对象给Java层

Java层原型方法：

```java
public native Student getStuInfo();
```

Native层方法实现：

```C
//Native层传递复杂对象给Java层
JNIEXPORT jobject JNICALL getStuInfo(JNIEnv *env, jobject obj) {
    LOGI("getStuInfo begin");
    jclass stu_cls = env->FindClass("com/example/github/jnidemo/MainActivity$Student");
    if (stu_cls == NULL) {
        return NULL;
    }
    //获得构造方法Id,函数名为<init>,返回类型为void,即V
    jmethodID constructMId = env->GetMethodID(stu_cls, "<init>", "(ILjava/lang/String;)V");
    if (constructMId == NULL) {
        LOGI("getStuInfo getConstruc failed");
        return NULL;
    }
    jstring name = env->NewStringUTF("赵一飞");
    //构造一个对象，调用构造方法并且传入相应参数
    return env->NewObject(stu_cls, constructMId, 26, name);
}
```

##### 4.Native层传递集合对象给Java层

Java层原型方法：

```java
public native ArrayList<Student> getListStudents();
```

Native层方法实现：

```C

//Native层传递集合对象给Java层
JNIEXPORT jobject JNICALL getListStudents(JNIEnv *env, jobject obj) {
    jclass list_cls = env->FindClass("java/util/ArrayList");//获得ArrayList类引用

    if (list_cls == NULL) {
        return NULL;
    }
    jmethodID list_costruct = env->GetMethodID(list_cls, "<init>", "()V"); //获得得构造函数Id

    jobject list_obj = env->NewObject(list_cls, list_costruct); //创建一个Arraylist集合对象
    //或得Arraylist类中的 add()方法ID，其方法原型为： boolean add(Object object) ;
    jmethodID list_add = env->GetMethodID(list_cls, "add", "(Ljava/lang/Object;)Z");

    jclass stu_cls = env->FindClass(
            "com/example/github/jnidemo/MainActivity$Student");//获得Student类引用
    //获得该类型的构造函数  函数名为 <init> 返回类型必须为 void 即 V
    jmethodID stu_costruct = env->GetMethodID(stu_cls, "<init>", "(ILjava/lang/String;)V");

    for (int i = 0; i < 3; i++) {
        jstring str = env->NewStringUTF("Native");
        //通过调用该对象的构造函数来new 一个 Student实例
        jobject stu_obj = env->NewObject(stu_cls, stu_costruct, 10, str);  //构造一个对象

        env->CallBooleanMethod(list_obj, list_add, stu_obj); //执行Arraylist类实例的add方法，添加一个stu对象
    }

    return list_obj;
}
```

这几种实例只是常见的几种数据交互的场景，后面会持续更新，敬请期待！

[完整代码](https://github.com/Jesse505/BlogDemo/tree/master/JniDemo),希望这篇文章能帮助到你，喜欢记得star下，谢谢！




