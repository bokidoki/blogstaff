---
title: okhttp原理浅析
date: 2018-12-03 09:04:57
tags: library
categories: Java
img: 
---

## 前言

Okhttp是square出品的替代httpClient和URLConnection的网络连接库，google也将URLConnection的源码替换成了Okhttp的实现方式，Okhttp可以算是Android唯一的网络库了，因此学习Okhttp势在必行，学习Okhttp的原理也对我们深入理解Http1.0/Http2.0的原理有直观的帮助。下面，我会由Okhttp的基础使用方法入手，逐层解析Okhttp的设计理念，以及它是如何发送一个http请求的。

## 使用方法

一般我们通过Okhttp的Builder()构建Okhttp的基础配置参数，并通过newCall()函数创建出了一个RealCall()。

> 目录 okhttp3\OkHttpClient.kt  
> 目录 okhttp3\RealCall.kt

```kotlin
val syncCall = client.newCall()
val asyncCall = client.newCall()

syncCall.execute() // 同步
asyncCall.enqueue() // 异步
```

client.newCall()创建了RealCall分别执行了RealCall的enqueue()和execute()

> 目录 okhttp3\RealCall.kt

```java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    // 通过调度器开启异步请求 dispatcher在OkhttpClient.Builder()初始化时就创建好了，最终走到RealCall的内部类AsyncCall的run()
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}

override fun run() {
  threadName("OkHttp ${redactedUrl()}") {
    var signalledCallback = false
    transmitter.timeoutEnter()
    try {
      // 与同步方法一样调用getResponseWithInterceptorChain
      val response = getResponseWithInterceptorChain()
      signalledCallback = true
      responseCallback.onResponse(this@RealCall, response)
    } catch (e: IOException) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        Platform.get().log(INFO, "Callback failure for ${toLoggableString()}", e)
      } else {
        responseCallback.onFailure(this@RealCall, e)
      }
    } finally {
      client.dispatcher.finished(this)
    }
  }
}

@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } catch (IOException e) {
    eventListener.callFailed(this, e);
    throw e;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

上面代码作用是将Call分别加入到调度器的双向队列中，方便调度器统一调度。另外Dispatcher调度器 enqueue控制正在进行中的异步任务不超过64个, 针对每个域名的请求不超过5个, 满足这个条件则线程池执行AsyncCall任务，若不满足则进入等待的队列。

```kotlin
/** Ready async calls in the order they'll be run. */
private val readyAsyncCalls = ArrayDeque<AsyncCall>()

/** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
private val runningAsyncCalls = ArrayDeque<AsyncCall>()

/** Running synchronous calls. Includes canceled calls that haven't finished yet. */
private val runningSyncCalls = ArrayDeque<RealCall>()

@get:Synchronized var maxRequests = 64
set(maxRequests) {
  require(maxRequests >= 1) { "max < 1: $maxRequests" }
  synchronized(this) {
    field = maxRequests
  }
  promoteAndExecute()
}

@get:Synchronized var maxRequestsPerHost = 5
set(maxRequestsPerHost) {
  require(maxRequestsPerHost >= 1) { "max < 1: $maxRequestsPerHost" }
  synchronized(this) {
    field = maxRequestsPerHost
  }
  promoteAndExecute()
}
```

不管是enquene还是excutor最终执行到RealCall.getResponseWithInterceptorChain()

> 目录okhttp3\RealCall.kt

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

可以看到依次添加了

- RetryAndFollowUpInterceptor
- BridgeInterceptor
- CacheInterceptor
- ConnectInterceptor
- CallServerInterceptor

接下来我们一个一个Interceptor分析，主要分析一下intercept的逻辑，看它们究竟分工做了什么活。

> RetryAndFollowUpInterceptor  
> 负责网络重连工作

![流程图](https://dreamweaver.img.we1code.cn/retryAndFollowUpInterceptor.png)

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  var request = chain.request()
  val realChain = chain as RealInterceptorChain
  // 管理连接池，复用连接
  val transmitter = realChain.transmitter()
  // 超出20此抛出异常
  var followUpCount = 0
  var priorResponse: Response? = null
  while (true) {
    transmitter.prepareToConnect(request)

    if (transmitter.isCanceled) {
      throw IOException("Canceled")
    }

    var response: Response
    var success = false
    try {
      // 交给下一个intercept处理
      response = realChain.proceed(request, transmitter, null)
      success = true
    } catch (e: RouteException) {
      // The attempt to connect via a route failed. The request will not have been sent.
      // 连接失败，判断是否可以恢复连接，若不可恢复，直接抛出异常结束请求。此时还没有发送请求
      if (!recover(e.lastConnectException, transmitter, false, request)) {
        throw e.firstConnectException
      }
      continue
    } catch (e: IOException) {
      // An attempt to communicate with a server failed. The request may have been sent.
      // 尝试与服务器建立连接失败。请求可能已经被发送。
      val requestSendStarted = e !is ConnectionShutdownException
      if (!recover(e, transmitter, requestSendStarted, request)) throw e
      continue
    } finally {
      // The network call threw an exception. Release any resources.
      // 网络请求抛出异常。释放资源。
      if (!success) {
        transmitter.exchangeDoneDueToException()
      }
    }

    // Attach the prior response if it exists. Such responses never have a body.
    // 尝试获取上一个可能存在的响应。这种响应不包含body。
    if (priorResponse != null) {
      response = response.newBuilder()
          .priorResponse(priorResponse.newBuilder()
              .body(null)
              .build())
          .build()
    }

    val exchange = response.exchange
    val route = exchange?.connection()?.route()
    val followUp = followUpRequest(response, route)

    if (followUp == null) {
      if (exchange != null && exchange.isDuplex) {
        transmitter.timeoutEarlyExit()
      }
      return response
    }

    val followUpBody = followUp.body
    if (followUpBody != null && followUpBody.isOneShot()) {
      return response
    }

    response.body?.closeQuietly()
    if (transmitter.hasExchange()) {
      exchange?.detachWithViolence()
    }

    if (++followUpCount > MAX_FOLLOW_UPS) {
      throw ProtocolException("Too many follow-up requests: $followUpCount")
    }

    request = followUp
    priorResponse = response
  }
}
```

> BridgeInterceptor  
> 桥接应用层代码与网络层代码。通过用户请求构造网络请求。然后请求网络。最后通过网络响应构建用户响应。在将请求交付给下一个处理节点前，它的主要作用是在 Request 的 Header 中添加缺失的头部信息（Content-Type、本地存储的 Cookie 信息），以及在接收到服务器返回的 Response 后存储 Header 中的 Cookie，并解压响应实体（如果是 gzip 的话）。  

```kotlin

```

> CacheInterceptor  
> 负责读写缓存，用到LRUDiskCache桥接应用层代码与网络层代码。通过用户请求构造网络请求。然后请求网络。最后通过网络响应构建用户响应。

![流程图](https://dreamweaver.img.we1code.cn/cacheInterceptor.png)

```kotlin

```

> ConnectInterceptor  
> 与服务器创建连接

主要步骤是创建一个Http的编解码器HttpCodec和一个链接对象RealConnection，并将它们加入请求链中，其中HttpCodec主要用于在CallServerInterceptor中发起Request和解析Response。

```kotlin

```

> CallServerInterceptor  
> 拦截器调用链的最后一个节点，向服务器发起最终的请求，完成写入请求，读取响应的工作

执行请求过程主要分为4步

- 写入Request Header
- 写入Request Body
- 读取Response Header
- 读取Response Body

```kotlin

```
