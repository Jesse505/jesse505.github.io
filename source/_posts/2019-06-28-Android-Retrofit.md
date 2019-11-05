---
title: Retrofit源码解析	
date: 2019-06-28 23:17:15
tags: 开源框架
categories: [Android,开源框架]
---



## 前言

- 在Android开发中网络请求很常见，而在网络请求库中，retrofit因为他巧妙的封装和解耦思想，成为了Android最热门的网络请求框架。
- 准确来说，**Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装。**
- 网络请求的工作本质上是 `OkHttp` 完成，而 Retrofit 仅负责 网络请求接口的封装。Okhttp源码分析，请[移步](https://jesse505.github.io/2018/02/08/2018-02-08-Understand-OkHttp/)

  


## 基本使用

首先看看Retrofit的使用，对Retrofit有个直观的感受，便于下面的源码分析。

**1、 定义http请求接口**

```java
public interface GitHubService {

     @GET("users/{user}/repos")
     Call<List<Repo>> listRepos(@Path("user") String user);
}
```

**2、构建retrofit的实例**

```java
Retrofit retrofit = new Retrofit.Builder()
.baseUrl("https://api.github.com/")
.addConverterFactory(GsonConverterFactory.create())
.build();
```

**3、创建http请求接口的实例**

```
GitHubService service = retrofit.create(GitHubService.class);
```

**4、调用接口实例的方法，生成Call对象，执行请求**

```java
Call<List<Repo>> repos = service.listRepos("websocket");
repos.execute() or repos.enqueue()
```

<!--more-->

## 源码分析

文章分析的源码版本是2.4.0

### 1、创建Retrofit实例

retrofit实例的创建使用了建造者模式，所以开发者并不需要关心配置细节就可以创建好Retrofit实例。在创建Retrofit对象时，你可以通过更多更灵活的方式去处理你的需求，如使用不同的Converter、使用不同的CallAdapter，这也就提供了你使用RxJava来调用Retrofit的可能。这部分源码比较简单，可以自行查看[源码](https://github.com/square/retrofit)。

**builder 模式，外观模式（门面模式）**，这就不多说了，可以看看 [stay 的 Retrofit分析-经典设计模式案例](http://www.jianshu.com/p/fb8d21978e38)这篇文章。

### 2、创建http请求接口的实例

```java
GitHubService service = retrofit.create(GitHubService.class);
```

通过create()方法创建请求接口的实例，我们来看看create()方法是怎么实现的

```java
public <T> T create(final Class<T> service) {
  ...
  return (T) Proxy.newProxyInstance(service.getClassLoader(), 
      new Class<?>[] { service },
      new InvocationHandler() {
        @Override 
        public Object invoke(Object proxy, Method method, Object... args)
            throws Throwable {
          //省略invoke方法的实现，在下面会详细讲解
        }
      });
}
```

创建 请求接口的实例使用的是**动态代理技术**。

简而言之，就是动态生成接口的实现类（当然生成实现类有缓存机制），并创建其实例（称之为代理），代理把对接口的调用转发给 `InvocationHandler` 实例，而在 `InvocationHandler` 的实现中，除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

### 3、调用接口实例的方法

当我们调用接口实例的方法listRepos()时，会调用自身的InvocationHandler#invoke()，得到最终的Call对象，我们接着看invoke()方法：

```java
    public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
        ...
      	// (1)读取网络请求接口里的方法，解析注解，并根据前面配置好的属性配置ServiceMethod对象
        ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
      	// (2)根据配置好的serviceMethod对象创建okHttpCall对象 
        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
      	// (3)调用OkHttp，并根据okHttpCall返回返回Call或者rxjava的Observe对象
        return serviceMethod.adapt(okHttpCall);
    }
```

#### 2.1、ServiceMethod

我们首先来分析**第（1）代码**

```java
ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
```

`ServiceMethod`是个什么鬼??? 不着急，我们先看`loadServiceMethod()`方法的实现：

```java
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    //从缓存里拿ServiceMethod
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        //如果缓存取不到，就创建ServiceMethod对象
        result = new ServiceMethod.Builder<>(this, method).build();
        //加入缓存
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

看到这里可能你会有疑问，同步代码里为什么还要再取一次缓存？

> 因为可能两个线程的请求都走到了synchronized之前，一个线程继续往下走，解析注解配置获得了serviceMethod对象并放到了缓存中，这个时候其实缓存里已经有serviceMethod对象了，另外一个线程走进来不需要再去解析注解配置了，所有需要再获取一遍。

接着我们继续看`ServiceMethod`的构造方法，看完你就知道他是个什么鬼了，哈哈

```java
  ServiceMethod(Builder<R, T> builder) {
    //网络请求工厂
    this.callFactory = builder.retrofit.callFactory();
    //网络请求适配器
    this.callAdapter = builder.callAdapter;
    //网络请求的地址
    this.baseUrl = builder.retrofit.baseUrl();
    //内容转换器，例如把服务器返回的内容可以转为Json的数据格式
    this.responseConverter = builder.responseConverter;
    //网络请求的http方法
    this.httpMethod = builder.httpMethod;
    //网络请求的相对地址
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    //方法参数处理器，负责解析 API 定义时每个方法的参数，并在构造 HTTP 请求时设置参数；
    this.parameterHandlers = builder.parameterHandlers;
  }
```

可以看到ServiceMethod类主要是网络请求配置的封装，这个里面我们重点关注`callFactory`，`callAdapter`，`responseConverter`，`parameterHandlers`这四个对象

##### 2.1.1 callFactory

通过上面的代码我们可以看到`callFactory`主要是通过`Retrofit`对象拿到的，而我们在构造 `Retrofit` 对象时，可以指定 `callFactory`，如果不指定，将默认设置为一个 `okhttp3.OkHttpClient`

##### 2.1.2 callAdapter

`callAdapter`是在build方法里设置的，在`build()`方法里调用了`createCallAdapter()`方法：

```java
private CallAdapter<?> createCallAdapter() {
  // 省略检查性代码
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { 
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```

可以看到，`callAdapter` 还是由 `Retrofit` 类提供。在 `Retrofit` 类内部，将遍历一个 `CallAdapter.Factory` 列表，让工厂们提供，如果最终没有工厂能（根据 `returnType` 和 `annotations`）提供需要的 `CallAdapter`，那将抛出异常。而这个工厂列表我们可以在构造 `Retrofit` 对象时进行添加。

##### 2.1.3 responseConverter

```java
private Converter<ResponseBody, T> createResponseConverter() {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { 
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create converter for %s", responseType);
  }
}
```

同样，`responseConverter` 还是由 `Retrofit` 类提供，而在其内部，逻辑和创建 `callAdapter`基本一致，通过遍历 `Converter.Factory` 列表，看看有没有工厂能够提供需要的 responseBodyConverter。工厂列表同样可以在构造 `Retrofit` 对象时进行添加。

##### 2.1.4 parameterHandlers

每个参数都会有一个 `ParameterHandler`，由 `ServiceMethod#parseParameter` 方法负责创建，其主要内容就是解析每个参数使用的注解类型（诸如 `Path`，`Query`，`Field` 等），对每种类型进行单独的处理。构造 HTTP 请求时，我们传递的参数都是字符串，那 Retrofit 是如何把我们传递的各种参数都转化为 String 的呢？还是由 `Retrofit` 类提供 converter！

`Converter.Factory` 除了提供上一小节提到的 responseBodyConverter，还提供 requestBodyConverter 和 stringConverter，API 方法中除了 `@Body` 和 `@Part` 类型的参数，都利用 stringConverter 进行转换，而 `@Body` 和 `@Part` 类型的参数则利用 requestBodyConverter 进行转换。

这三种 converter 都是通过“询问”工厂列表进行提供，而工厂列表我们可以在构造 `Retrofit` 对象时进行添加。

**上面提到了三种工厂：`okhttp3.Call.Factory`，`CallAdapter.Factory` 和 `Converter.Factory`，分别负责提供不同的模块，至于怎么提供、提供何种模块，统统交给工厂，Retrofit 完全不掺和，它只负责提供用于决策的信息，例如参数/返回值类型、注解等。**

除了上面重点分析的这四个成员，`ServiceMethod` 中还包含了 API 方法的 url 解析等逻辑，包含了众多关于泛型和反射相关的代码，例如`parseMethodAnnotation()`方法，里面主要是解析http的请求方法和地址以及请求头。

#### 2.2、OkHttpCall

接着我们来分析**第（2）代码**

```java
OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
```

`OkHttpCall` 实现了 `retrofit2.Call`，我们通常会使用它的 `execute()` 和 `enqueue(Callback<T> callback)` 接口。

我们看同步执行 `execute()` 方法：

```java
//OkHttpCall.java
@Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
			...
      call = rawCall;
      if (call == null) {
        try {
          //创建okhttp3.Call对象
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
          creationFailure = e;
          throw e;
        }
      }
    }
    if (canceled) {
      call.cancel();
    }
		//执行网络请求解析返回结果，其实就是把okhttp中的Response转化为Retrofit中的Response
    return parseResponse(call.execute());
  }

	//OkHttpCall.java
  private okhttp3.Call createRawCall() throws IOException {
    //调用serviceMethod中的toCall方法并传入方法的参数
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

	//ServiceMethod.java
	//根据方法的参数和ServiceMethod的配置创建http的请求
  okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return callFactory.newCall(requestBuilder.build());
  }


```

在`toCall()`方法中，通过`ServiceMethod`中的`httpMethod`，`baseUrl`，`parameterHandlers`等等来创建`RequestBuilder`对象，最后调用`callFactory.newCall()`来创建okhttp3.Call对象。接着我们看返回值的解析：

```java
	//OkHttpCall.java  
	Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
	  ....
    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      catchingBody.throwIfCaught();
      throw e;
    }
  }
	
	//ServiceMethod.java
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

在 `toResponse` 函数中，就是调用我们设置的内容转换器或者是默认的内容转换器完成响应的数据转换。

看到这里，我们发现，在`ServiceMethod`中的相关配置在`OkHttpCall`中基本都用到了，但是我们发现`callAdapter`还没使用到，接下来就轮到他登场了。

#### 2.3 CallAdapter

最后我们来分析**第（3）代码**

```java
serviceMethod.adapt(okHttpCall);
```

`CallAdapter<T>#adapt(Call<R> call)` 函数负责把 `retrofit2.Call<R>` 转为 `T`。这里 `T` 当然可以就是 `retrofit2.Call<R>`，这时我们直接返回参数就可以了，实际上这正是 `DefaultCallAdapterFactory` 创建的 `CallAdapter` 的行为。在Android平台上默认就是`ExecutorCallAdapterFactory`



到这里，一个基本的网络请求的过程就算分析完成了。