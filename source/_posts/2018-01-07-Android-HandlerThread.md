---
title: HandlerThread使用场景和源码解析
date: 2018-01-07 22:11:15
tags: HandlerThread
categories: [Android,基础]
---

## 一、内容概览

上一章总结完消息机制，我们接着看HandlerThread，这是谷歌把消息机制和Thread相结合的类，我们主要说说使用场景，如何使用，以及源码解析。

## 二、使用场景，如何使用

### 使用场景
想象一下，如果你有多个串行的耗时任务需要执行，最简单方法就是开启好多线程，线程的开启和销毁是很消耗性能的，当然你也可以使用线程池，当然最好的方案还是使用HandlerThread。

### 如何使用呢？

1. 创建HandlerThread的实例对象

```java
HandlerThread handlerThread = new HandlerThread("myHandlerThread");
```

该参数表示线程的名字，可以随便选择。 

2. 启动我们创建的HandlerThread线程

```java
handlerThread.start();
```

1. 将我们的handlerThread与Handler绑定在一起。其实很简单，就是将线程的looper与Handler绑定在一起，代码如下：

``` java
mThreadHandler = new Handler(mHandlerThread.getLooper()) {
    @Override
    public void handleMessage(Message msg) {
        checkForUpdate();
        if(isUpdate){
            mThreadHandler.sendEmptyMessage(MSG_UPDATE_INFO);
        }
    }
};
```

**注意必须按照以上三个步骤来，如果没有start就调用getLooper，getLooper获取的会是一个null值。**

<!--more-->

## 三、源码解析

既然是一个线程，我们首先看run方法：

```java
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        //持有锁机制来获得当前线程的Looper对象
        synchronized (this) {
            mLooper = Looper.myLooper();
            //发出通知，当前线程已经创建mLooper对象成功，这里主要是通知getLooper方法中的wait
            notifyAll();
        }
        //设置线程的优先级别
        Process.setThreadPriority(mPriority);
        //这里默认是空方法的实现，我们可以重写这个方法来做一些线程开始之前的准备，方便扩展
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

你会发现把Looper消息循环的一套流程拿到run方法里了，最终分发给Handler处理的消息就是在这run方法里执行的，所以这里实现了异步操作。

我们再看这里为什么要加锁，要想知道为什么，我们接着看getLooper方法：

```java
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        // 直到线程创建完Looper之后才能获得Looper对象，Looper未创建成功，阻塞
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```

可以看到这里涉及到多线程的同步问题，在调用getLooper方法的时候，run方法还没有执行的话，那么getLooper方法返回的就是一个null值，这明显不是我们想要的，所以这里要加锁。

由于run方法是一个无限循环，当我们确定不需要HandlerThread时，我们应该通过quit或者quitSafely终止线程，我们看源码：

```java
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
```

跟踪这两个方法，我们发现最终调用的是MessageQueue中的`quit(boolean safe)`，如下：

```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;
        //安全退出调用这个方法
        if (safe) {
            removeAllFutureMessagesLocked();
        } else {//不安全退出调用这个方法
            removeAllMessagesLocked();
        }
        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

不安全的会调用`removeAllMessagesLocked();`这个方法，我们来看这个方法是怎样处理的，其实就是遍历Message链表，移除所有信息的回调，并重置为null。

```java
private void removeAllMessagesLocked() {
    Message p = mMessages;
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
```

安全地会调用`removeAllFutureMessagesLocked();`这个方法，它会根据Message.when这个属性，判断我们当前消息队列是否正在处理消息，没有正在处理消息的话，直接移除所有回调，正在处理的话，等待该消息处理处理完毕再退出该循环。因此说`quitSafe()`是安全的，而`quit()`方法是不安全的，因为quit方法不管是否正在处理消息，直接移除所有回调。

```java
private void removeAllFutureMessagesLocked() {
    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;
    if (p != null) {
        //判断当前队列中的消息是否正在处理这个消息，没有的话，直接移除所有回调
        if (p.when > now) {
            removeAllMessagesLocked();
        } else {//正在处理的话，等待该消息处理处理完毕再退出该循环
            Message n;
            for (;;) {
                n = p.next;
                if (n == null) {
                    return;
                }
                if (n.when > now) {
                    break;
                }
                p = n;
            }
            p.next = null;
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}
```


