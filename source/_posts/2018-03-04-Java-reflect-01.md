---
title: Java进阶 - 反射总结
date: 2018-03-04 22:01:15
tags: 反射
categories: [Java,进阶]
---

## 使用反射获取类的信息

### 获取类的所有变量信息

调用getFields()输出类以及父类所有的public声明的变量；  
调用getDeclaredFields()输出类的所有成员变量，不问访问权限；

### 获取类的所有方法信息

调用getMethods()输出类以及父类所有的public声明的方法；  
调用getDeclaredMethods()输出类的所有成员方法，不问访问权限；

<!--more-->

## 访问或操作类的私有变量和私有方法

新建一个测试类

```java
public class TestClass {

    private String MSG = "Original";
    
    private final String FINAL_VALUE = "hello";

    private void privateMethod(String head , int tail){
        System.out.print(head + tail);
    }

    public String getMsg(){
        return MSG;
    }
    
    publiv String getFinalValue() {
    	 return FINAL_VALUE;
	 }
```

### 访问私有方法

访问TestClass中privateMethod()私有方法,直接看代码，注释写的很清楚：

```java
/**
 * 访问对象的私有方法
 * 为简洁代码，在方法上抛出总的异常，实际开发别这样
 */
private static void getPrivateMethod() throws Exception{
    //1. 获取 Class 类实例
    TestClass testClass = new TestClass();
    Class mClass = testClass.getClass();

    //2. 获取私有方法
    //第一个参数为要获取的私有方法的名称
    //第二个为要获取方法的参数的类型，参数为 Class...，没有参数就是null
    //方法参数也可这么写 ：new Class[]{String.class , int.class}
    Method privateMethod =
            mClass.getDeclaredMethod("privateMethod", String.class, int.class);

    //3. 开始操作方法
    if (privateMethod != null) {
        //获取私有方法的访问权
        //只是获取访问权，并不是修改实际权限
        privateMethod.setAccessible(true);

        //使用 invoke 反射调用私有方法
        //privateMethod 是获取到的私有方法
        //testClass 要操作的对象
        //后面两个参数传实参
        privateMethod.invoke(testClass, "Java Reflect ", 666);
    }
}
```

### 访问私有变量

访问TestClass中的MSG变量，其初始值为 "Original" ，我们要修改为 "Modified",直接看代码：

```java
/**
 * 修改对象私有变量的值
 * 为简洁代码，在方法上抛出总的异常
 */
private static void modifyPrivateFiled() throws Exception {
    //1. 获取 Class 类实例
    TestClass testClass = new TestClass();
    Class mClass = testClass.getClass();

    //2. 获取私有变量
    Field privateField = mClass.getDeclaredField("MSG");

    //3. 操作私有变量
    if (privateField != null) {
        //获取私有变量的访问权
        privateField.setAccessible(true);

        //修改私有变量，并输出以测试
        System.out.println("Before Modify：MSG = " + testClass.getMsg());

        //调用 set(object , value) 修改变量的值
        //privateField 是获取到的私有变量
        //testClass 要操作的对象
        //"Modified" 为要修改成的值
        privateField.set(testClass, "Modified");
        System.out.println("After Modify：MSG = " + testClass.getMsg());
    }
}
```

### 修改私有常量

补充一个知识点：  
Java 虚拟机（JVM）在编译 .java 文件得到 .class 文件时，会优化我们的代码以提升效率。其中一个优化就是：JVM 在编译阶段会把引用常量的代码替换成具体的常量值，**所以就算我们修改了私有常量也没有多大的意义**。为了修改有意义，我们可以尝试如下修改：

#### 方法一

在声明常量时不赋值，在构造函数里赋值，因为构造函数是在new对象时才会调用，所以就避免了JVM编译阶段的优化

#### 方法二

使用三目运算符赋值，因为三目运算符是在运行时刻计算的，所以也避免了编译时候被优化：

```java
private final String FINAL_VALUE
        = null == null ? "FINAL" : null;
```

最后总结一下，私有常量通过反射肯定是可以修改的，只是修改以后有没有意义还需要再确定。