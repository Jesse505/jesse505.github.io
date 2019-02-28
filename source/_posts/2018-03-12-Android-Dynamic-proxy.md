---
title: 静态代理与动态代理的区别和使用场景
date: 2018-03-12 21:11:15
tags: 设计模式
categories: [Android,设计模式]
---

在讲解静态代理与动态代理的区别之前，我们先来回顾代理模式相关的知识点，往下看。

## 代理模式的定义

>为其他对象提供一种代理以控制对这个对象的访问。

说白了，我们可以理解为生活中常见的中介或者明星经纪人，我们买房一般都会通过中介，但是最后卖房的却是开发商，可以认为中介就是开发商的代理。

## 代理模式的使用场景
>当不能访问或不想直接访问或访问某个对象存在困难时，我们可以通过一个代理对象来间接访问，为了客户端使用的透明性，我们应该保证代理对象和被代理对象应该实现同一个接口。

## 静态代理
理论知识讲了一堆，有点绕口，我们直接讲静态代理吧，我们以买房为例定义接口：

```java
public interface Buy {
    void buyHouse(long money);
}
```
<!--more-->

小明去买房，小明也就是被代理对象：

```java
public class Xiaoming implements Buy {
    @Override
    public void buyHouse(long money) {
        System.out.println("我买房了，用了"+money+" 钱 ");
    }
}
```
中介，也就是代理对象：

```java
public class UserProxy implements Buy {
    /**
     *这个是真实对象，买房一定是真实对象来买的，中介只是跑腿的
     */
    private Buy mBuy;
    public UserProxy(Buy mBuy) {
        this.mBuy = mBuy;
    }

    @Override
    public void buyHouse(long money) {
        /**
         * 这里是我们出钱去买房,中介只是帮忙
         */
        mBuy.buyHouse(newMoney);
    }
}
```

如上就是静态代理的实现，我们可以看到代理类是在编译期间就已经存在的，而且UserProxy代理类也只能代理实现了Buy接口的类，而动态代理与静态代理相反，通过反射机制动态的生成代理者的对象，也就是说我们在code阶段根本不需要知道代理谁，代理谁将会在执行阶段决定。

## 动态代理

动态代理是运行时的代理，Java实现动态代理需要实现InvocationHandler接口：

```java
public class DynamiclProxy implements InvocationHandler {
    //被代理的类引用
    private Object mObject;

    public DynamiclProxy(Object mObject) {
        this.mObject = mObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //调用被代理类对象的方法
    Object result = method.invoke(mObject, args);
    return result;
    }
}
```

如上代码所述，我们声明一个Object引用指向被代理的类，而我们调用被代理类的方法则是在invoke()方法里进行，是不是很简洁？也就是说我们本来交给代理类完成的事情现在完全由InvocationHandler来处理，不再关心需要代理谁，下面我们修改一下客户类的逻辑：

```java
public class ProxyClient {
    public static void main(String[] args){
        System.out.println("静态代理测试");
        Buy buy=new Xiaoming();
        UserProxy proxy=new UserProxy(buy);
        proxy.buyHouse(1000000);

        System.out.println("动态代理测试");
        Buy dynamicProxy= (Buy) Proxy.newProxyInstance(buy.getClass().getClassLoader(),
                buy.getClass().getInterfaces(),new DynamiclProxy(buy));
        dynamicProxy.buyHouse(1000000);
    }
}

```

运行一下，可以发现打印是一样的。  

通过上面的分析，我们可以总结下：**静态代理与动态代理的区别在于代理类生成的时间不同，如果需要对多个类进行代理，并且代理的功能都是一样的，用静态代理重复编写代理类就非常的麻烦，可以用动态代理动态的生成代理类**。

## 应用场景

动态代理不仅在Java中有重要的作用，特别是AOP编程方面，更是在Android的插件话发挥了不可或缺的作用，我们前面说过Java层的Hook一般有反射和动态代理2个方面，一般情况下是成对出现的，反射是负责找出隐藏的对象，而动态代理则是生成目标接口的代理对象，然后再由反射替换掉，一起完成有意思的事情。  
下面我们来Hook Android中NotificationManagerService,直接上代码：  

```java
public static void hookNotificationManager(Context context) {
        try {
            NotificationManager notificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
            Method method = notificationManager.getClass().getDeclaredMethod("getService");
            method.setAccessible(true);
            //获取代理对象
            final Object sService = method.invoke(notificationManager);
            Log.d("[app]", "sService=" + sService);
            Class<?> INotificationManagerClazz = Class.forName("android.app.INotificationManager");
            Object proxy = Proxy.newProxyInstance(INotificationManagerClazz.getClassLoader(),
                    new Class[]{INotificationManagerClazz},new NotifictionProxy(sService));
            //获取原来的对象
            Field mServiceField = notificationManager.getClass().getDeclaredField("sService");
            mServiceField.setAccessible(true);
            //替换
            mServiceField.set(notificationManager, proxy);
            Log.d("[app]", "Hook NoticeManager成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

public class NotifictionProxy implements InvocationHandler {
    private Object mObject;

    public NotifictionProxy(Object mObject) {
        this.mObject = mObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Log.d("[app]", "方法为:" + method.getName());
        /**
         * 做一些业务上的判断
         * 这里以发送通知为准,发送通知最终的调用了enqueueNotificationWithTag
         */
        if (method.getName().equals("enqueueNotificationWithTag")) {
            //具体的逻辑
            for (int i = 0; i < args.length; i++) {
                if (args[i]!=null){
                    Log.d("[app]", "参数为:" + args[i].toString());
                }
            }
            //做些其他事情，然后替换参数之类
            return method.invoke(mObject, args);
        }
        return null;
    }
}

```

这里面有反射相关的知识，可以参考我的[反射总结](https://jesse505.github.io/2018/03/04/2018-03-04-Java-reflect-01/)。  
好了，我们在Application里面的attachBaseContext()方法里面注入就好，为什么要在这里注入呢，因为attachBaseContext()在四大组件中的方法是最先执行的，比ContentProvider的onCreate()方法都先执行，而ContrentProvider的onCreate()方法比Application的onCreate()都先执行，大家可以去测试一下,因此如果Hook的地方是涉及到ContentProvider的话，那么最好在这个地方执行，我们在页面发送通知试试:,代码如下:

```java
Intent intent=new Intent();
                Notification build = new NotificationCompat.Builder(MotionActivity.this)
                        .setContentTitle("测试通知")
                        .setContentText("测试通知内容")
                        .setAutoCancel(true)
                        .setDefaults(Notification.DEFAULT_SOUND|Notification.DEFAULT_VIBRATE)
                        .setPriority(NotificationCompat.PRIORITY_MAX)
                        .setSmallIcon(R.mipmap.ic_launcher)
                        .setWhen(System.currentTimeMillis())
                        .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
                        .setContentIntent(PendingIntent.getService(MotionActivity.this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT))
                        .build();
                NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
                manager.notify((int) (System.currentTimeMillis()/1000L), build);

```

好了，我们看一下结果：

<img src="/img/201803/java_dynamic_proxy.png" alt="java_dynamic_proxy" style="width: 600px;">

看到结果了吧，已经成功检测到被Hook的方法了，而具体如何执行就看具体的业务了。

## 参考
[Java代理以及在Android中的一些简单应用](https://juejin.im/post/5a2e4e9a51882559e2259ad3)  
书籍 - Android设计模式 
