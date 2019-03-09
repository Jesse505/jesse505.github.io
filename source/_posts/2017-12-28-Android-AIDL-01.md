---
title: AIDL使用小结
date: 2017-12-28 12:11:15
tags: AIDL
categories: [Android,基础]
---

## 一、内容概览
本章内容结合《Android开发艺术探索》一书，给出自己使用AIDL的注意点和总结，主要有几下几个方面： 
 
* AIDL支持的数据结构
* AIDL的单向调用-客户端调用服务端
* AIDL的双向调用

## 二、AIDL支持的数据结构

在AIDL文件中，并不是所有的数据类型都是支持的，那么支持哪些数据类型，如下所示：

* 基本数据类型(int,long,double,char,boolean)
* String和CharSequence
* List:只支持ArrayList，并且里面的元素都能被AIDL支持
* Map:只支持HashMap，并且里面的元素都能被AIDL支持，包括key和value
* Parcelable:所有实现了Parcelable的对象
* AIDL:所有AIDL接口本身也是支持的

其中有几个注意点：  

* 自定义的Parcelable对象和AIDL对象都要显示的import进来，哪怕在同一个包内
* AIDL除了基本数据类型，其他类型必须加上方向：in，out，inout
* AIDL的包结构在服务端和客户端要保持一致

<!--more-->

## 三、AIDL的单向调用-客户端调用服务端

#### 服务端

服务端新建一个service监听客户端连接，再新建AIDL文件暴露给客户端，最终在service中实现AIDL接口。

#### 客户端

绑定服务，在绑定成功后将服务端返回的Binder对象转为AIDL接口对象，这样就可以调用AIDL中的接口了。

这里面有几个需要注意的地方:

* 自定义的Parcelable对象在AIDL文件中如果被加上方向,那她必须实现readFromParcel()方法
* Client调用AIDL接口的时候,会block住,因此clinet调用过程不应该在主线程
* AIDL接口的实现执行是在Binder线程池中,因此不应该去操作UI相关的内容
* 如果多个客户端连接同一个服务并进行数据的操作,可能会造成数据的不同步,这个时候就应该使用CopyOnWriteArrayList和ConcurrentHashMap

## 四、AIDL的双向调用

上一节已经实现了客户端调用服务端,那么服务端调用客户端的话只需要增加一个AIDL接口,也就是所谓的观察者模式,这个时候服务端就是被观察者，客户端是观察者。当然为什么选择AIDL接口，不选择普通接口，上面章节也说过了AIDL只支持AIDL接口不支持普通接口。

下面给出主要代码分析

#### 新建AIDL接口

```
import com.example.github.aidlserver.Book;

interface IOnNewBookArrivedListener {
    void onNewBookArrived(in Book newBook);
}
```

#### 添加注册和注销方法

```
import com.example.github.aidlserver.Book;
import com.example.github.aidlserver.IOnNewBookArrivedListener;

interface BookController {

    void addBook(inout Book book);

    List<Book> getBookList();

    void registerListener(IOnNewBookArrivedListener listener);

    void unRegisterListener(IOnNewBookArrivedListener listener);

}
```

#### 服务端实现

通过RemoteCallbackList保存客户端传过来的Listener对象，调用Listener对象中的方法：

```java
        final int N = mListeners.beginBroadcast();
        for (int i = 0; i < N; i++) {
            IOnNewBookArrivedListener listener = mListeners.getBroadcastItem(i);
            if (null != listener) {
                listener.onNewBookArrived(newBook);
            }
        }
        mListeners.finishBroadcast();
```

#### 客户端实现

新建一个Listener对象并传给服务端：

```java
IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {
        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException {
            //注意这里不能执行UI相关的工作,需要通过handler发送出去
            Log.i(TAG, "来了新书:" + newBook.getName());
        }
    };
```
