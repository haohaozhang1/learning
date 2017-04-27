上一篇[Okhttp3源码分析（一）](http://blog.csdn.net/android_text/article/details/70810322)讲了Request、OkHttpClient、RealCall类，下面继续往深层次讲述Okhttp3的奥秘

**一、核心类Dispatcher**

```
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

 
```
分析一下：

 1. maxRequests ：最大同时请求数
 2. maxRequestsPerHost ：同时最大的相同Host的请求数
 3. executorService ： 线程池
 4. readyAsyncCalls ：异步的等待队列
 5. runningAsyncCalls ： 异步的执行对列
 6. runningSyncCalls ：同步的执行队列

```
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
分析一下这个线程池，对ThreadPoolExecutor不了解的可以去看[ThreadPoolExecutor源码分析](http://blog.csdn.net/android_text/article/details/70318780)

 1. 此线程数是核心线程为0
 2. 最大线程数为Interger.MAX_VALUE = 0x7FFFFFFF
 3. 非核心线程的KeepLive空闲时间为60，单位为TomeUnit.SECONDS
 4. 任务队列为SynchronousQueue<Runnable>
 5. 创建Thread的工厂类为Util.threadFactory("OkHttp Dispatcher", false)

另外可以设置最大请求数
```
public synchronized void setMaxRequests(int maxRequests) {
    if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
    }
    this.maxRequests = maxRequests;
    promoteCalls();
  }
```

设置最大同时Host的请求数
```
public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
    if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
    }
    this.maxRequestsPerHost = maxRequestsPerHost;
    promoteCalls();
  }
```

**1、异步执行**

```
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```
分析一下：
当异步执行队列runningAsyncCalls<maxRequests &&和该任务的Host一样的任务的总数<maxRequestsPerHost；才会把该任务加入异步执行队列，然后交给线程池去执行；
反之加入异步等待队列readyAsyncCalls

```
ublic synchronized void cancelAll() {
    for (AsyncCall call : readyAsyncCalls) {
      call.cancel();
    }

    for (AsyncCall call : runningAsyncCalls) {
      call.cancel();
    }

    for (RealCall call : runningSyncCalls) {
      call.cancel();
    }
  }
```
取消所有的请求任务

异步的finished
```
synchronized void finished(AsyncCall call) {
    if (!runningAsyncCalls.remove(call)) throw new AssertionError("AsyncCall wasn't running!");
    promoteCalls();
  }
```
当一个任务执行完以后就会调用Dispatcher的finish（AsyncCall ），**线程池里的线程循环利用就是靠promoteCalls();**

```
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

 - 判断是否正在执行数>=maxRequests ;return;
 - 异步等待队列.isEmpty; return
 - 循环等待队列readyAsyncCalls，不断的从队列里取出任务交给线程池执行；此时也要符合最大执行数和相同HOST的最大执行数


----------

**2、同步执行**
先看一下RealCall的同步执行

```
@Override
  public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain(false);
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

 - 首先同步关键字synchronized ，设置标志位executed，如果为true代表已经执行了，throw
 - Dispatcher分发器把该同步任务加入同步执行队列里
 - 执行该同步任务
 - 执行完，调用Dispatcher的finished（Call）
 
```
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  synchronized void finished(Call call) {
    if (!runningSyncCalls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
  }
```
上面很简单，只是对同步队列的入队和出队操作


----------
**二、简要介绍一下SynchronousQueue**<**Runnable**>

此队列是一个不会储存任何元素的队列，如果想offer/put入队列，首先得有一个pull/take阻塞等待。
如何没有pull/take阻塞等待取元素，offer/put入队不成功。反之亦然。
此队列是一个安全阻塞队列。





 
 
