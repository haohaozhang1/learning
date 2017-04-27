**一、首先对Request类做一下分析：**

Request就是组装Http发起的请求；Http发起的请求包含什么可以看一下[HTTP协议格式和header](http://blog.csdn.net/android_text/article/details/70739146)
看一下源码：

```
public final class Request {
  private final HttpUrl url;
  private final String method;
  private final Headers headers;
  private final RequestBody body;
  private final Object tag;

  private volatile URI javaNetUri; // Lazily initialized.
  private volatile CacheControl cacheControl; // Lazily initialized.

  private Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tag = builder.tag != null ? builder.tag : this;
  }
```

看到没？其实就是url、method、headers、requestBody
实例：

```
Request request = new Request.Builder()
                .url("")
                .addHeader("", "")
                .addHeader("", "")
                .get()
                .build();
```

**二、再对OkHttpClient类做下分析**

OKHttpClinet里的内容其实就是Builder内的内容

```
public static final class Builder {
    Dispatcher dispatcher;
    Proxy proxy;
    List<Protocol> protocols;
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();
    final List<Interceptor> networkInterceptors = new ArrayList<>();
    ProxySelector proxySelector;
    CookieJar cookieJar;
    Cache cache;
    InternalCache internalCache;
    SocketFactory socketFactory;
    SSLSocketFactory sslSocketFactory;
    TrustRootIndex trustRootIndex;
    HostnameVerifier hostnameVerifier;
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;
    Authenticator authenticator;
    ConnectionPool connectionPool;
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;

    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
    }
```
对里面几个主要的解释一下：

 1. dispatcher：分发器，用于分发任务的。相当于服务器的nginx
 2. List<Interceptor> interceptors = new ArrayList<>()：应用拦截器
 3. List<Interceptor> networkInterceptors = new ArrayList<>()：网络拦截器
 4. Dns：用于解析dns的
 4. connectTimeout： socket的connectTimeOut
 5. readTimeout：socket的readTimeout
 6. writeTimeout：socket的writeTimeout
 
**三、重点介绍RealCall类**

构造函数

```
final class RealCall implements Call {
  private final OkHttpClient client;

  // Guarded by this.
  private boolean executed;
  volatile boolean canceled;

  /** The application's original request unadulterated by redirects or auth headers. */
  Request originalRequest;
  HttpEngine engine;

protected RealCall(OkHttpClient client, Request originalRequest) {
   this.client = client;
   this.originalRequest = originalRequest;
 }
```
可以看到把OkHttpClient和Request传过来了

再看一下enqueue方法，这个其实就是咱们发起的异步任务

```
@Override
public void enqueue(Callback responseCallback) {
  enqueue(responseCallback, false);
}

void enqueue(Callback responseCallback, boolean forWebSocket) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
}
```
client就是OkHttpClient，client.dispatcher()就是咱们在OkHttpClient里面初始化的dispatcher

所以下一步走到了Dispatcher.enqueue(new AsyncCall(responseCallback, forWebSocket));

**先分析一下AsyncCall类**

```
final class AsyncCall extends NamedRunnable {
  private final Callback responseCallback;
  private final boolean forWebSocket;

  private AsyncCall(Callback responseCallback, boolean forWebSocket) {
    super("OkHttp %s", originalRequest.url().toString());
    this.responseCallback = responseCallback;
    this.forWebSocket = forWebSocket;
  }

  String host() {
    return originalRequest.url().host();
  }

  Request request() {
    return originalRequest;
  }

  Object tag() {
    return originalRequest.tag();
  }

  void cancel() {
    RealCall.this.cancel();
  }

  RealCall get() {
    return RealCall.this;
  }

  @Override
  protected void execute() {
    boolean signalledCallback = false;
    try {
      Response response = getResponseWithInterceptorChain(forWebSocket);
      if (canceled) {
        signalledCallback = true;
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      client.dispatcher().finished(this);
    }
  }
}
```
发现AsyncCall继承了NamedRunnable；分析一下NamedRunnable的源码

```
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = String.format(format, args);
  }

  @Override
  public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```
很简单，NamedRunnable就是一个任务，然后再run（）里调用了execute（）；
所以最终AsyncCall也是一个任务，执行方法在execute（）;
分析一下execute方法：

```
try {
      Response response = getResponseWithInterceptorChain(forWebSocket);
      if (canceled) {
        signalledCallback = true;
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      client.dispatcher().finished(this);
    }
```

 1. 进行网络请求getResponseWithInterceptorChain（）
 2. 是否取消了请求，取消了就回调onFailure
 3. 正常回调onResponse
 4. 出现异常也是回调onFailure
 5. finally移除任务 

```
private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
    Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
  }
```

走到了应用拦截器里面ApplicationInterceptorChain

```
class ApplicationInterceptorChain implements Interceptor.Chain {
   private final int index;
   private final Request request;
   private final boolean forWebSocket;

   ApplicationInterceptorChain(int index, Request request, boolean forWebSocket) {
     this.index = index;
     this.request = request;
     this.forWebSocket = forWebSocket;
   }

   @Override
   public Connection connection() {
     return null;
   }

   @Override
   public Request request() {
     return request;
   }

   @Override
   public Response proceed(Request request) throws IOException {
     // If there's another interceptor in the chain, call that.
     if (index < client.interceptors().size()) {
       Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
       Interceptor interceptor = client.interceptors().get(index);
       Response interceptedResponse = interceptor.intercept(chain);

       if (interceptedResponse == null) {
         throw new NullPointerException("application interceptor " + interceptor
             + " returned null");
       }

       return interceptedResponse;
     }

     // No more interceptors. Do HTTP.
     return getResponse(request, forWebSocket);
   }
 }
```
分析proceed方法：其实相当于一个循环，OkHttpClient.Builder.addInterceptor(Interceptor interceptor) 添加的interceptors循环一个个对Request进行封装，然后把最后封装好的Request扔给**getResponse(request, forWebSocket)**

```
Response getResponse(Request request, boolean forWebSocket) throws IOException {
    // Copy body metadata to the appropriate request headers.
    RequestBody body = request.body();
    if (body != null) {
      Request.Builder requestBuilder = request.newBuilder();

      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }

      request = requestBuilder.build();
    }

    // Create the initial HTTP engine. Retries and redirects need new engine for each attempt.
    engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null);

    int followUpCount = 0;
    while (true) {
      if (canceled) {
        engine.releaseStreamAllocation();
        throw new IOException("Canceled");
      }

      boolean releaseConnection = true;
      try {
        engine.sendRequest();//真正的发送请求
        engine.readResponse();//读取response
        releaseConnection = false;
      } catch (RequestException e) {
        // The attempt to interpret the request failed. Give up.
        throw e.getCause();
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        //出现异常 重新再次请求
        HttpEngine retryEngine = engine.recover(e.getLastConnectException(), null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }
        // Give up; recovery is not possible.
        throw e.getLastConnectException();
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        //出现异常 重新再次请求
        HttpEngine retryEngine = engine.recover(e, null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }

        // Give up; recovery is not possible.
        throw e;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        //没有重连就释放连接
        if (releaseConnection) {
          StreamAllocation streamAllocation = engine.close();
          streamAllocation.release();
        }
      }

      Response response = engine.getResponse();//获取response
      Request followUp = engine.followUpRequest();//是否有重定向

      if (followUp == null) {//没有重定向
        if (!forWebSocket) {
          engine.releaseStreamAllocation();//释放connect
        }
        return response;
      }

      StreamAllocation streamAllocation = engine.close();

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
      // 如果重定向的地址和当前的地址一样，则不需要释放连接
      if (!engine.sameConnection(followUp.url())) {
        streamAllocation.release();
        streamAllocation = null;
      }

      request = followUp;
      //根据重定向重新组装engine
      engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response);
    }
  }
```
分析一下getResponse（）

 1. 组装request
 2. 初始化HttpEngine；engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null);
 3. 发送请求：engine.sendRequest();
 4. 读取响应 ：engine.readResponse();
 5. 是否有异常：是否用重连，发起重连
 5. 获取response：Response response = engine.getResponse();
 6. 是否有重定向：Request followUp = engine.followUpRequest();
 7. 没有重定向，释放connect，返回response：engine.releaseStreamAllocation();return response;
 8. 有重定向：组装engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response)循环执行

 
 

