---
title: okhttp源码初探 - 基本流程 	
date: 2018-02-08 20:17:15
tags: 开源框架
categories: [Android,开源框架]
---


## 1、整体思路

首先放一张完整流程图（看不懂没关系，慢慢往后看）：

<img src="/img/201712/okhttp_full_process.png" alt="okhttp_full_process" style="width: 600px;">

<!--more-->

## 2、流程分析
基本用法来自 [OkHttp 官方网站](http://square.github.io/okhttp/#examples)

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
### 2.1 创建OkHttpClient对象
首先是创建OkHttpClient对象，支持两种构造方式	

##### 默认方式

```java
public OkHttpClient() {
    this(new Builder());
  }
```
这种方式不需要传入任何参数，也就是说基本参数都是默认的，调用的是如下构造函数

```java
public OkHttpClient(Builder builder) {...}
```

##### builder模式

```java
  public OkHttpClient build() {
    return new OkHttpClient(this);
  }
```
由此我们可以看到okhttp中的**builder设计模式**

### 2.2 创建Request对象
构建完OkHttpClient后就需要构建一个Request对象，查看Request的源码你会发现，你找不到public的构造函数，唯一的一个构造函数是这样的

```java
  private Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tag = builder.tag != null ? builder.tag : this;
  }
```
这就意味着Request对象是通过builder模式创建的

### 2.3 发起 HTTP 请求
构建完Request后，我们就需要构建一个Call，一般都是这样的Call call = mOkHttpClient.newCall(request); 那么我们就返回OkHttpClient的源码看看。

```java
  @Override public Call newCall(Request request) {
    return new RealCall(this, request);
  }
```
可以看到newCall方法是Override的，我们再看看OkHttpClient是否有继承和实现

```java
public class OkHttpClient implements Cloneable, Call.Factory { ... }
```
我们发现`OkHttpClient` 实现了 `Call.Factory`，负责根据请求创建新的 `Call`，这里其实用到了**工厂模式**的思想，将构建的细节交给具体实现，顶层只需要拿到Call对象即可。
另外我们还可以看到newCall返回的是一个RealCall对象，下面我们就一边分析网络请求的过程，一边看看RealCall的具体内容

#### 2.3.1 同步网络请求

我们首先看 `RealCall#execute`：

``` java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");  // (1)
    executed = true;
  }
  try {
    //加入runningSyncCalls队列
    client.dispatcher().executed(this);                                 // (2)
    Response result = getResponseWithInterceptorChain();                // (3)
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    //移出runningSyncCalls队列
    client.dispatcher().finished(this);                                 // (4)
  }
}
```

这里我们做了 4 件事：

1. 检查这个 call 是否已经被执行了，每个 call 只能被执行一次，如果想要一个完全一样的 call，可以利用 `call#clone` 方法进行克隆。
2. 利用 `client.dispatcher().executed(this)` 来进行实际执行，`dispatcher` 是刚才看到的 `OkHttpClient.Builder` 的成员之一，它的文档说自己是异步 HTTP 请求的执行策略，现在看来，同步请求它也有掺和。
3. 调用 `getResponseWithInterceptorChain()` 函数获取 HTTP 返回结果，从函数名可以看出，这一步还会进行一系列“拦截”操作。
4. 最后还要通知 `dispatcher` 自己已经执行完毕。

dispatcher 用三个队列保存call，分别是runningSyncCalls， runningAsyncCalls， readyAsyncCalls，在同步执行的流程中，涉及到 dispatcher 的内容只不过是告知它我们的执行状态，比如开始执行了（调用 `executed`），比如执行完毕了（调用 `finished`），也就是加入队列，移除队列的过程。

真正发出网络请求，解析返回结果的，还是 `getResponseWithInterceptorChain`：

``` java
private Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!retryAndFollowUpInterceptor.isForWebSocket()) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(
      retryAndFollowUpInterceptor.isForWebSocket()));

  Interceptor.Chain chain = new RealInterceptorChain(
      interceptors, null, null, null, 0, originalRequest);
  return chain.proceed(originalRequest);
}
```

在 [OkHttp 开发者之一介绍 OkHttp 的文章里面](https://publicobject.com/2016/07/03/the-last-httpurlconnection/)作者讲到：

> the whole thing is just a stack of built-in interceptors.

可见 `Interceptor` 是 OkHttp 最核心的一个东西，不要误以为它只负责拦截请求进行一些额外的处理（例如 cookie），**实际上它把实际的网络请求、缓存、透明压缩等功能都统一了起来**，每一个功能都只是一个 `Interceptor`，它们再连接成一个 `Interceptor.Chain`，环环相扣，最终圆满完成一次网络请求。

从 `getResponseWithInterceptorChain` 函数我们可以看到，`Interceptor.Chain` 的分布依次是：

<img src="/img/201712/okhttp_interceptors.png" alt="okhttp_interceptors" style="width: 500px;">

1. 在配置 `OkHttpClient` 时设置的 `interceptors`；
2. 负责失败重试以及重定向的 `RetryAndFollowUpInterceptor`；
3. 封装Request和Reponse过滤器`BridgeInterceptor`；
4. 负责读取缓存直接返回、更新缓存的 `CacheInterceptor`；
5. 负责和服务器建立连接的 `ConnectInterceptor`；
6. 配置 `OkHttpClient` 时设置的 `networkInterceptors`；
7. 负责向服务器发送请求数据、从服务器读取响应数据的 `CallServerInterceptor`。为了便于理解各个拦截器的作用，这里盗用一张图说明下：

<img src="/img/201712/okhttp_interceptors_do.png" alt="okhttp_interceptors_do" style="width: 500px;">

添加完过滤器后，就是执行过滤器了,这里也很重要，一开始看比较难以理解。

```java
Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
```
可以看到这里创建了一个RealInterceptorChain，并调用了proceed方法，这里注意一下0这个参数。我们来看看proceed方法的实现

```java
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpStream != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpStream != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpStream, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpStream != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
```
一脸懵逼啊，不要慌，稍微处理下

```java
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

	...

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpStream, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    ...

    return response;
  }
```
这样就很清晰了，这里index就是我们刚才的0，也就是从0开始，如果index超过了过滤器的个数抛出异常，后面会再new一个RealInterceptorChain，而且会将参数传递，并且index+1了，接着获取index的interceptor,并调用intercept方法，传入新new的next对象，这里可能就有点感觉了，这里用了递归的思想来完成遍历，为了验证我们的想法，随便找一个interceptor，看一下intercept方法。

```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    。。。暂时没必要看。。。

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```
可以看到这里我们拿了一个ConnectInterceptor的源码，这里得到chain后，进行相应的处理后，继续调用proceed方法，那么接着刚才的逻辑，index+1,获取下一个interceptor,重复操作，所以现在就很清楚了，这里利用递归循环，也就是okHttp最经典的责任链模式。

责任链模式在安卓系统中也有比较典型的实践，例如 view 系统对点击事件（TouchEvent）的处理，具体可以参考[Android设计模式源码解析之责任链模式](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/chain-of-responsibility/AigeStudio#android源码中的模式实现)中相关的分析。

#### 2.3.2，发起异步网络请求

``` java
client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        System.out.println(response.body().string());
    }
});

// RealCall#enqueue
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}

// Dispatcher#enqueue
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```

这里我们就能看到 dispatcher 在异步执行时发挥的作用了，如果当前还能执行一个并发请求，那就立即执行，否则加入 `readyAsyncCalls` 队列，而正在执行的请求执行完毕之后，会调用 `promoteCalls()` 函数，来把 `readyAsyncCalls` 队列中的 `AsyncCall` “提升”为 `runningAsyncCalls`，并开始执行。

这里的 `AsyncCall` 是 `RealCall` 的一个内部类，它实现了 `Runnable`，所以可以被提交到 `ExecutorService` 上执行，而它在执行时会调用 `getResponseWithInterceptorChain()` 函数，并把结果通过 `responseCallback` 传递给上层使用者。代码如下：

```java
	//RealCall的内部类
  final class AsyncCall extends NamedRunnable {
   		....省略部分代码
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        //还是通过此方法获取网络请求的返回值
        Response response = getResponseWithInterceptorChain();
        ...
      } catch (IOException e) {
        ...
      } finally {
        //通知dispatcher
        client.dispatcher().finished(this);
      }
    }
  }

//继续追踪client.dispatcher().finished(this);发现调用的是如下代码
  private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();
      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }
      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```

这样看来，同步请求和异步请求的原理是一样的，都是在 `getResponseWithInterceptorChain()` 函数中通过 `Interceptor` 链条来实现的网络请求逻辑，而异步则是通过 `ExecutorService` 实现。

### 2.4 返回数据的获取

在上述同步（`Call#execute()` 执行之后）或者异步（`Callback#onResponse()` 回调中）请求完成之后，我们就可以从 `Response` 对象中获取到响应数据了，包括 HTTP status code，status message，response header，response body 等。这里 body 部分最为特殊，因为服务器返回的数据可能非常大，所以必须通过数据流的方式来进行访问（当然也提供了诸如 `string()` 和 `bytes()` 这样的方法将流内的数据一次性读取完毕），而响应中其他部分则可以随意获取。

响应 body 被封装到 `ResponseBody` 类中，该类主要有两点需要注意：

1. 每个 body 只能被消费一次，多次消费会抛出异常；
2. body 必须被关闭，否则会发生资源泄漏；

这里有一点值得一提，OkHttp 对响应的校验非常严格，HTTP status line 不能有任何杂乱的数据，否则就会抛出异。

### 2.5 HTTP 缓存

在 [2.3.1，同步网络请求] 小节中，我们已经看到了 `Interceptor` 的布局，在建立连接、和服务器通讯之前，就是 `CacheInterceptor`，在建立连接之前，我们检查响应是否已经被缓存、缓存是否可用，如果是则直接返回缓存的数据，否则就进行后面的流程，并在返回之前，把网络的数据写入缓存。

这块代码比较多，但也很直观，主要涉及 HTTP 协议缓存细节的实现，而具体的缓存逻辑 OkHttp 内置封装了一个 `Cache` 类，它利用 `DiskLruCache`，用磁盘上的有限大小空间进行缓存，按照 LRU 算法进行缓存淘汰，这里也不再展开。

我们可以在构造 `OkHttpClient` 时设置 `Cache` 对象，在其构造函数中我们可以指定目录和缓存大小：

``` java
public Cache(File directory, long maxSize);
```

而如果我们对 OkHttp 内置的 `Cache` 类不满意，我们可以自行实现 `InternalCache` 接口，在构造 `OkHttpClient` 时进行设置，这样就可以使用我们自定义的缓存策略了。

## 3 总结

在文章最后我们再来回顾一下完整的流程图：

<img src="/img/201712/okhttp_full_process.png" alt="okhttp_full_process" style="width: 600px;">

+ `OkHttpClient` 实现 `Call.Factory`，负责为 `Request` 创建 `Call`；
+ `RealCall` 为具体的 `Call` 实现，其 `enqueue()` 异步接口通过 `Dispatcher` 利用 `ExecutorService` 实现，而最终进行网络请求时和同步 `execute()` 接口一致，都是通过 `getResponseWithInterceptorChain()` 函数实现；
+ `getResponseWithInterceptorChain()` 中利用 `Interceptor` 链条，分层实现缓存、透明压缩、网络 IO 等功能；



感谢： https://dandanlove.com/2018/11/25/okhttp-interceptor/