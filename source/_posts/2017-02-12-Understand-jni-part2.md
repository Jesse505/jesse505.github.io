---
title: Android与JNI（二）数据类型映射、描述符说明
date: 2017-02-12 10:44:57
tags: JNI
categories: [Android,NDK]
---

### 数据类型映射
在Java存在两种数据类型： 基本类型 和 引用类型 ，大家都懂的 。  
在JNI的世界里也存在类似的数据类型，与Java比较起来，其范围更具严格性，如下：  

1. primitive types ----基本数据类型，如：int、 float 、char等基本类型
2. reference types----引用类型，如：类、实例、数组。  

特别需要注意：**数组** ------ 不管是对象数组还是基本类型数组，都作为reference types存在。

<!--more-->

#### 基本数据类型映射

<img src="/img/201702/understand-jni-part2-img1.png" alt="understand jni png" style="width: 600px;">

**这些基本数据类型都是可以在Native层直接使用的 。**


#### 引用数据类型映射

<img src="/img/201702/understand-jni-part2-img2.png" alt="understand jni png" style="width: 600px;">

**注意：**  
**1、引用数据类型则不能直接使用，需要根据JNI函数进行相应的转换后，才能使用；**  
**2、多维数组(包括二维数组)都是引用类型，需要使用 jobjectArray 类型存取其值；** 

例如：二维整型数组就是指向一位数组的数组，其声明使用方式如下：

```java
//获得一维数组 的类引用，即jintArray类型
jclass intArrayClass = env->FindClass("[I"); 
//构造一个指向jintArray类一维数组的对象数组，该对象数组初始大小为dimion
jobjectArray obejctIntArray  =  env->NewObjectArray(dimion ,intArrayClass , NULL);
```

另外，关于引用类型的一个**继承关系**如下，我们可以对具有父子关系的类型进行转换：

<img src="/img/201702/understand-jni-part2-img3.png" alt="understand jni png" style="width: 600px;">



### 描述符

#### 类描述符

**类描述符**是类的完整名称（包名+类名）,将原来的 . 分隔符换成 / 分隔符。  
例如：在java代码中的java.lang.String类的类描述符就是java/lang/String

其实，在实践中，我发现可以直接用该类型的域描述符取代，也是可以成功的。  
例如：jclass intArrCls = env->FindClass("java/lang/String")

等同于 jclass intArrCls = env->FindClass("Ljava/lang/String;")

**数组类型**的描述符则为，则为：  [ + 其类型的域描述符        (后文说明)，例如：  
int [ ]     其描述符为[I  
float [ ]   其描述符为[F  
String [ ]  其描述符为[Ljava/lang/String;


#### 域描述符

1、**基本类型的描述符**已经被定义好了，如下表所示：
  
<img src="/img/201702/understand-jni-part2-img4.png" alt="understand jni png" style="width: 600px;">

2、**引用类型的描述符**  
一般引用类型则为 L + 该类型类描述符 + ;   (**注意，这儿的分号“；”只得是JNI的一部分，而不是我们汉语中的分段，下同**)  
例如：String类型的域描述符为 Ljava/lang/String;  

对于**数组**，其为 :  [ + 其类型的域描述符 + ,例如：  
int[ ]     其描述符为[I  
float[ ]   其描述符为[F  
String[ ]  其描述符为[Ljava/lang/String;  
Object[ ]类型的域描述符为[Ljava/lang/Object;

**多维数组**则是 n个[ +该类型的域描述符 , N代表的是几维数组。例如：   
int  [ ][ ] 其描述符为[[I   
float[ ][ ] 其描述符为[[F


#### 方法描述符

将参数类型的域描述符按照申明顺序放入一对括号中后跟返回值类型的域描述符，规则如下：   
1. 参数的域描述符叠加返回  
2. 对于，没有返回值的，用V(表示void型)表示。举例如下：  

<img src="/img/201702/understand-jni-part2-img5.png" alt="understand jni png" style="width: 600px;">

接下来，我们将通过具体的实例来学习JNI。

reference: [http://blog.csdn.net/qinjuning](http://blog.csdn.net/qinjuning)



