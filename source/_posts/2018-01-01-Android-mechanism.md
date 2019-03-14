---
title: Android的消息机制
date: 2018-01-01 21:11:15
tags: Handler
categories: [Android,基础]
---

## 一、内容概览

说到Android中的消息机制，你会想到什么，Handler? Message?没错在开发中确实是是handler和Message用的最多了，最常见的就是子线程通知UI线程更新UI。但是Android中的消息机制并不仅仅这么简单，我们从以下几点开始分析：

* 消息机制概述
* MessageQueue的原理
* Looper的工作原理
* Handler的工作原理

## 二、消息机制概述

消息机制主要包含：MessageQueue，Handler和Looper这三大部分，以及Message，下面我们一一介绍。

**Message：**需要传递的消息，可以传递数据；

**MessageQueue：**消息队列，但是它的内部实现并不是用的队列，实际上是通过一个单链表的数据结构来维护消息列表，因为单链表在插入和删除上比较有优势。主要功能向消息池投递消息\(MessageQueue.enqueueMessage\)和取走消息池的消息\(MessageQueue.next\)；

**Handler：**消息辅助类，主要功能向消息池发送各种消息事件\(Handler.sendMessage\)和处理相应消息事件\(Handler.handleMessage\)；

**Looper：**不断循环执行\(Looper.loop\)，从MessageQueue中读取消息，按分发机制将消息分发给目标处理者。

下面给出一张来自网络的消息机制的流程图：

![](http://upload-images.jianshu.io/upload_images/3985563-d7da4f5ba49f6887.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

<!--more-->

## 三、MessageQueue的原理

MessageQueue主要包含两个操作，插入和读取，而读取也是伴随着删除操作，插入和读取对应的方法是`enqueueMessage`和`next`，都说源码是最好的书籍，直接看源码：

```java
boolean enqueueMessage(Message msg, long when) {
    // 每一个Message必须有一个target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //正在退出时，回收msg，加入到消息池
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; 
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

MessageQueue是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) { //当消息循环已经退出，则直接返回
        return null;
    }
    int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                //当消息Handler为空时，查询MessageQueue中的下一条异步消息msg，为空则退出循环。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;   //成功地获取MessageQueue中的下一条即将要执行的消息
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }
         //消息正在退出，返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            ...............................
    }
}
```

nativePollOnce是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。  
可以看出`next()`方法根据消息的触发时间，获取下一条需要执行的消息,队列中消息为空时，则会进行阻塞操作。

说完了MessageQueue的两个方法后，那到底这两个方法是谁在调用呢？Looper和Handler就应该上场了

## 四、Looper的工作原理

就是Looper在调用MessageQueue中的next的，到底怎么调用的呢，我们接着看，首先要创建一个looper:

**初始化Looper**  
无参情况下，默认调用`prepare(true);`表示的是这个Looper可以退出，而对于false的情况则表示当前Looper不可以退出。。

```java
 public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

这里看出，不能重复创建Looper，只能创建一个。创建Looper,并保存在TheadLocal中。

**开启Looper**

```java
public static void loop() {
    final Looper me = myLooper();  //获取TLS存储的Looper对象 
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;  //获取Looper对象中的消息队列

    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) { //进入loop的主循环方法
        Message msg = queue.next(); //可能会阻塞
        if (msg == null) { //消息为空，则退出循环
            return;
        }

        Printer logging = me.mLogging;  //默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg); //获取msg的目标Handler，然后用于分发Message 
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
         
        }
        msg.recycleUnchecked(); 
    }
}
```

loop\(\)进入循环模式，不断调用queue的next方法，直到消息为空时退出循环，拿到message后，在通过msg.target.dispatchMessage把Message分发给相应的target，这里的target其实就是我们接下来要讲的Handler。


## 五、Handler的工作原理

Handler的工作主要包含消息的发送和接收。消息的发送都是通过send和post的一系列方法来实现的，其实最终调用的都是`sendMessageAtTime`

```java
 public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
       //其中mQueue是消息队列，从Looper中获取的
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //调用enqueueMessage方法
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //调用MessageQueue的enqueueMessage方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

可以看到sendMessageAtTime\(\)\`方法的作用很简单，就是调用MessageQueue的enqueueMessage\(\)方法，往消息队列中添加一个消息。MessageQueue的next方法会返回这条消息给Looper，Looper收到消息后分发给Handler处理，即Handler的dispatchMessage方法会被调用：

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当Message存在回调方法，回调msg.callback.run()方法；
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //当Handler存在Callback成员变量时，回调方法handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //Handler自身的回调方法handleMessage()
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
        message.callback.run();
    }
```

可以看到消息的处理很简单，我们都知道handleMeaage()是我们经常实例化一个handler对象的时候需要重写的方法，msg.callback是调用post系列方法传过来的runnable对象，那mCallback是个什么鬼？

Callback是个接口，定义如下：

```java
    public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        public boolean handleMessage(Message msg);
    }
```

*通过Callback可以创建Handler对象：Handler handler = new Handler(callback);看到这里我们就明白了，如果我们不想派生一个Handler子类的话，就可以用这种方式新建Handler对象*

**这里有个需要注意的地方，在Activity退出的时候，记得调用下removeCallbacksAndMessages(null)，移除掉所有的callback和message，防止内存泄漏。**

## 六、总结

总结的话再次盗用网上的一张图，毕竟图形化的东西更容易理解。

![](http://upload-images.jianshu.io/upload_images/3985563-b3295b67a2b0477f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

最后感谢《Android开发艺术探索》这本书的作者玉刚！

