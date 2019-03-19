---
title: JsBridge源码解析
date: 2019-01-18 21:11:15
tags: 开源框架
categories: [Android,开源框架]
---

## 内容概览

现在的app基本都是H5混合开发了，这样就涉及到原生代码与Js代码的交互，JsBridge就这样产生了，下面我们分几方面讲解JsBridge是如何实现Native代码和Js的互调：

* JsBridge原理
* 核心文件说明
* Native调用Js流程
* Js调用Native流程

## JsBridge原理

**Native调用Js**


webview可以通过loadUrl()的方法直接调用。在4.4以上还可以通过evaluateJavascript()方法获取js方法的返回值。JsBridge就是通过loadUrl的方式实现Native调用Js的。

<!--more-->

**Js调用Native**

目前有三种方案：

1. API注入。通过webview.addJavascriptInterface()的方法实现。
2. 拦截Js的alert/confirm/prompt/console等事件。由于prompt事件在js中很少使用，所以一般是拦截该事件。这些事件在WebChromeClient都有对应的方法回调（onConsoleMessage，onJsPrompt，onJsAlert，onJsConfirm）
3. url跳转拦截，对应WebViewClient的shouldOverrideUrlLoading()方法

第一种方法，由于webview在4.2以下的安全问题，所以有版本兼容问题。JsBridge就是通过url跳转拦截，自定义协议，实现Js调用Native的。


## 核心文件说明

**BridgeWebViewClient.java** ：WebViewClient的子类，重写了ShouldOverrideUrlLoading，onPageFinish，onPageStart等方法

**BridgeWebView.java** ：WebView的子类，提供了注册Handler，调用Handler等方法

**BridgeWebView.java** ：bridge接口文件,定义了发送信息的方法，由BridgeWebView来实现

**BridgeHandler.java** ：作为Java与Js交互的载体。Java&Js通过Handler的名称来找到响应的Handler来操作

**DefaultBridgeHandler.java** ：BridgeHandler的子类，不做任何操作。仅为Java提供默认的接收数据的Handler

**Message.java** ：消息对象，用来封装与js交互时的json数据，callbackId，responseId等

**CallBackFunction.java** ：回调接口，用于Handler处理完后，返回消息的入口

**BridgeUtil.java** ：工具类，提供从Url中提取数据，获取传递信息等方法

**WebViewJavascriptBridge.js** ：被注入到各个页面的js文件；提供初始化，注册Handler，调用Handler等方法


## Native调用Js

首先举个Native调用Js的例子：

```java
webView.callHandler("functionInJs", "data from Java", new CallBackFunction() {
				@Override
				public void onCallBack(String data) {
					// TODO Auto-generated method stub
					Log.i(TAG, "reponse data from js " + data);
				}

});
```

我们一步步来跟踪，接着看callHandler方法：

```java
//java调用Js
//handlerName是Js提前注册的方法名 data是方法的参数
//callBack 是java的回调对象
public void callHandler(String handlerName, String data,
                            CallBackFunction callBack) {
   doSend(handlerName, data, callBack);
}
```

这里handlerName就是在js里提前注册好的方法的名称，如果还没注册好会导致调用不成功，我们接着看doSend方法：

```java
private void doSend(String handlerName, String data,
                        CallBackFunction responseCallback) {
        //将要调用的js方法名和参数封装成Message
        Message m = new Message();
        if (!TextUtils.isEmpty(data)) {
            m.setData(data);
        }
        //如果java需要回调，生成唯一的回调id，放到message中
        //并且将对应的java回调保存到 responseCallbacks中，以callbackId为键
        if (responseCallback != null) {
            String callbackStr = String.format(
                    BridgeUtil.CALLBACK_ID_FORMAT,
                    ++uniqueId
                            + (BridgeUtil.UNDERLINE_STR + SystemClock
                            .currentThreadTimeMillis()));
            responseCallbacks.put(callbackStr, responseCallback);
            m.setCallbackId(callbackStr);
        }

        if (!TextUtils.isEmpty(handlerName)) {
            m.setHandlerName(handlerName);
        }

        queueMessage(m);
}
```

可以看到doSend方法就是将参数data,回调id,方法名handlerName封装成Message对象，然后调用queueMessage:

```java

final static String JS_HANDLE_MESSAGE_FROM_JAVA  
    = "javascript:WebViewJavascriptBridge._handleMessageFromNative('%s');";
    
private void queueMessage(Message m) {
  //startupMessage在页面第一次加载完成就会置空，所以这里一定会走到dispatchMessage中
   if (startupMessage != null) {
       startupMessage.add(m);
    } else {
       dispatchMessage(m);
    }
}

 private void dispatchMessage(Message m) {
    //message转成json,并进行转义
    String messageJson = m.toJson();
    messageJson = messageJson.replaceAll("\\\\/", "/");
    messageJson = messageJson.replaceAll("(\\\\)([^utrn])", "\\\\\\\\$1$2");
    messageJson = messageJson.replaceAll("(?<=[^\\\\])(\")", "\\\\\"");
    //将转义好的消息json串传递到js的_handleMessageFromNative方法中
    //并在主线程中调用
    String javascriptCommand = String.format(
            BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
    messageJson = null;
    if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
        this.loadUrl(javascriptCommand);
    }
}
```
可以看到最终调用LoadUrl将Message对象转成的json数据传给js的_handleMessageFromNative方法了

我们接着看js中_handleMessageFromNative：

```js
function _handleMessageFromNative(messageJSON) {
    //receiveMessageQueue在页面加载完成后就赋值为null了
    //所以最后都会到_dispatchMessageFromNative中
    if (receiveMessageQueue) {
        receiveMessageQueue.push(messageJSON);
    } else {
        _dispatchMessageFromNative(messageJSON);
    }
}

 function _dispatchMessageFromNative(messageJSON) {
    setTimeout(function() {
    //将传递的消息json串转为message对象    
    var message = JSON.parse(messageJSON);
    var responseCallback;

     //这里是js调用java并且js有回调，这里表明java将js回调需要的数据传过来了
     //此时再进行js 回调处理
      if (message.responseId) {
                responseCallback = responseCallbacks[message.responseId];
                if (!responseCallback) {
                    return;
                }
                responseCallback(message.responseData);
                delete responseCallbacks[message.responseId];
            } else {
                //直接发送 
                ////这里如果有回调id， 说明前面java需要js回传数据
                //构造回调对象
                if (message.callbackId) {
                    var callbackResponseId = message.callbackId;
                    responseCallback = function(responseData) {
                        _doSend({
                            responseId: callbackResponseId,
                            responseData: responseData
                        });
                    };
                }
            //从js预定义的方法集合messageHandlers中，匹配传递过来消息中方法名
                var handler = WebViewJavascriptBridge._messageHandler;
                if (message.handlerName) {
                    handler = messageHandlers[message.handlerName];
                }
             //匹配成功后，调用该方法，传递数据，并且将回调对象作为参数也传递了
                try {
                    handler(message.data, responseCallback);
                } catch (exception) {
                    if(responseCallback){
                        responseCallback({error:404,errorMessage:"JS API not find"})
                    }
                    if (typeof console != 'undefined') {
                        console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                    }
                }
            }
        });
    }
```

可以看到java调用js 并没有responseId 所以走else，接着从预先注册好的方法数组中找到相对应的方法，如下：

```java
//js 注册了一个名为functionInJs的方法 这里的responseCallback就是传递的回调对象
bridge.registerHandler("functionInJs", function(data, responseCallback) {
    document.getElementById("show").innerHTML = ("data from Java: = " + data);
    /因为java不一定需要js回传数据，只有需要的时候，才会传入该对象，所以这里判空
    if (responseCallback) {
         var responseData = {data:'Javascript Says Right back aka!'};
         //使用该回调对象主动调用其方法
          responseCallback(responseData);
     }
});
```

其实到这里就已经完成了Native调用Js了，那么Js是如何把响应数据返回给Native的呢？可以看到js可以通过responseCallback方法把相应数据返回，我们往前看到下面这段代码：

```js
				//构造回调对象
                if (message.callbackId) {
                    var callbackResponseId = message.callbackId;
                    responseCallback = function(responseData) {
                        _doSend({
                            responseId: callbackResponseId,
                            responseData: responseData
                        });
                    };
                }
```

可以看到responseCallback最终调用的是_doSend方法，并且这个时候Message的callbackId变成了reponseId

```js
function _doSend(message, responseCallback) {
   //这时的responseCallback为null
  if (responseCallback) {
     var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
     responseCallbacks[callbackId] = responseCallback;
     message.callbackId = callbackId;
  }
  //将上步的message放入sendMessageQueue发送消息队列中
  sendMessageQueue.push(message);
  //改变iframe的src从而通知native从h5取消息
  messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
 }
```

通过改变iframe的src会触发WebViewClient的shouldOverrideUrlLoading()，通知Native取消息。

```js
//通知Native有消息的协议
yy://__QUEUE_MESSAGE__/
```

我们接着看Native的处理：

```java
@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    WYLogUtils.i(TAG, "shouldOverrideUrlLoading --> " + url);
    //yy://return/{function}/returncontent || yy://"
    //这里开始拦截协定的协议
    if (url.startsWith(BridgeUtil.YY_RETURN_DATA) || url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) {
        //解码
        try {
            url = URLDecoder.decode(url, "UTF-8");
        } catch (IllegalArgumentException e) { //解决未编码同时内容中有%时奔溃的问题
            WYLogUtils.e(TAG, e.getMessage(), e);
            if (url.contains("%%")) {
                try {
                    url = URLDecoder.decode(removeDoublePercent(url), "UTF-8");
                } catch (Exception e1) {
                    WYLogUtils.e(TAG, e1.getMessage(), e1);
                }
            }
        } catch (Exception e) {
            WYLogUtils.e(TAG, e.getMessage(), e);
        }
        //yy://return/{function}/returncontent || yy://
        //这里由于上面是为yy://__QUEUE_MESSAGE__/ 所以会走到flushMessageQueue()中
        if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) {// 如果是返回数据
            handlerReturnData(url);
        } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) {
            flushMessageQueue();
        }
        return true;
    }
    ...
}
```

这里拦截到预定的协议后，调用了flushMessageQueue()

```java
private void flushMessageQueue() {
     //这里在主线程
     if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
        //这里往loadUrl方法中传了一个字符串和一个新的回调对象
        loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA,new CallBackFunction() {
                @Override
                public void onCallBack(String data) {
                ...
            }
}

private void loadUrl(String jsUrl, CallBackFunction returnCallback) {
     //主线程执行了js的_fetchQueue方法
     this.loadUrl(jsUrl);
     responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl),
             returnCallback);
 }
```

```java
BridgeUtil.java
//这是个js方法 
final static String JS_FETCH_QUEUE_FROM_JAVA = "javascript:WebViewJavascriptBridge._fetchQueue();";

//从字符串中解析出js的方法名
public static String parseFunctionName(String jsUrl) {
        return jsUrl.replace("javascript:WebViewJavascriptBridge.",  "").replaceAll("\\(.*\\);", "");
```

可以看到最终调用了js的_fetchQueue方法，并且以_fetchQueue为key,将上面创建的回调对象为值，存到了responseCallbacks中，我们接着看Js中的处理：

```js
function _fetchQueue() {
    //将上面要回传给native的消息sendMessageQueue转为json
    //所有的消息
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    //这里肯定是Android Ios多余 用的不是一个框架
    if(isAndroid()){
     //通知Native过来拿消息。 
        messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + messageQueueString;
    }else if (isIphone()) {
        return messageQueueString;
//android can't read directly the return data, so we can reload iframe src to communicate with java
    }
}
```

这里将要回传给native的消息转为json，放到iframe的src中，从而通知native取消息。

```js
//返回数据的协议
yy:///return/_fetchQueue/+json
```

触发WebViewClient的shouldOverrideUrlLoading()，拦截该url，并调用handlerReturnData(url)

```java
final static String YY_OVERRIDE_SCHEMA = "yy://";    
final static String YY_RETURN_DATA = YY_OVERRIDE_SCHEMA + "return/";/

//shouldOverrideUrlLoading方法中的代码片段
if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) {// 如果是返回数据
    handlerReturnData(url);
} else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) {
    flushMessageQueue();
}
```

handlerReturnData方法中处理回传的数据，并执行前面存入回调对象的方法。

```java
private void handlerReturnData(String url) {
    //functionName: _fetchQueue
    //以_fetchQueue为key从responseCallbacks中取出上面存入的回调对象
    String           functionName = BridgeUtil.getFunctionFromReturnUrl(url);
    CallBackFunction f            = responseCallbacks.get(functionName);
    //从回传数据中解析出回传的数据
    String data = BridgeUtil.getDataFromReturnUrl(url);
    if (f != null) {
        //执行回调对象的方法，并传入数据
        f.onCallBack(data);
        //移除回调
        responseCallbacks.remove(functionName);
    }
}
```

我们接着看之前的回调对象：

```java
			loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {

				@Override
				public void onCallBack(String data) {
					// deserializeMessage 反序列化消息
					List<Message> list = null;
					try {
						list = Message.toArrayList(data);
					} catch (Exception e) {
                        e.printStackTrace();
						return;
					}
					if (list == null || list.size() == 0) {
						return;
					}
					for (int i = 0; i < list.size(); i++) {
						Message m = list.get(i);
						String responseId = m.getResponseId();
						// 这里reponseId不为空，也就是之前的callBackId
						if (!TextUtils.isEmpty(responseId)) {
							CallBackFunction function = responseCallbacks.get(responseId);
							String responseData = m.getResponseData();
							function.onCallBack(responseData);
							responseCallbacks.remove(responseId);
						} else {
							...
						}
					}
				}
			});
```

这里reponseId不为空，也就是之前的callBackId，通过reponseId从responseCallbacks取出开头例子中传入的回调对象，这样就顺利的把相应数据通过回调对象返回了，至此，Native调用js的流程已经全部分析结束，让我们来总结下其中的关键流程：


<img src="/img/201901/Native2JS.png" alt="Native2Js" style="width: 600px;">





## Js调用Native

js调用Native的流程其实和Native调用Js基本是相似的，有兴趣的可以下载[源码](https://github.com/lzyzsd/JsBridge)看看，这里直接给出关键流程图：

<img src="/img/201901/Js2Native.png" alt="Js2Native" style="width: 600px;">

参考文档：https://mp.weixin.qq.com/s/R7GyDuzAHzIYt_91QZLvWw
