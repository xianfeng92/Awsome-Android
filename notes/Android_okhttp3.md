
# okhttp3

## 一个简单的okhttp请求

我们用OkHttp3发起一个网络请求一般是这样：

1. 首先要构建一个OkHttpClient：

```
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(15,TimeUnit.SECONDS)
                .writeTimeout(15,TimeUnit.SECONDS)
                .build();
```

2. 然后构建一个 Request

```
        Request request = new Request.Builder()
                .url("https://www.baidu.com/")
                .build();
```


3. 发起请求

```
// 同步请求
okHttpClient.newCall(request).execute();

// 异步请求
okHttpClient.newCall(request).enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    
                }

                @Override
                public void onResponse(Call call, Response response) throws IOException {

                }
            });

```

以上是简略的用OkHttp3请求网络的步骤，下面我们来通过源码分析下。

## 源码分析

我们先来看看OkHttp的newCall方法：

```
  /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

```
可以看见返回的 RealCall，所以我们发起请求无论是调用 execute方法还是 enqueue 方法，实际上调用的都是 RealCall 内部的方法。

```
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      client.dispatcher().executed(this);
      return getResponseWithInterceptorChain();
    } finally {
      client.dispatcher().finished(this);
    }
  }

  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

在 RealCall 内部的 enqueue 方法和 execute 方法中，都是通过 OkHttpClient 的任务调度器 Dispatcher 来完成。这里的 Dispatcher
为一个任务调度器，通过它来调度请求。


## 任务调度器 Dispatcher

Dispatcher中有一些重要的变量:

```
//最大并发请求数
private int maxRequests = 64;
//每个主机的最大请求数
private int maxRequestsPerHost = 5;

//这个是线程池，采用懒加载的模式，在第一次请求的时候才会初始化
private @Nullable ExecutorService executorService;

//将要运行的异步请求任务队列
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

//正在运行的异步请求任务队列
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

//正在运行的同步请求队列
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```
1. Dispatcher 中使用队列这个数据结构来存储请求任务，所以对于最先发起的请求会最先被执行

2. executorService 用来执行异步请求


我们再看看 Dispatcher 的构造方法:

```
  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

1. 可以看到 Dispatcher 有 2 个构造方法，我们如果需要用自己的线程池，可以调用带有线程池参数的构造方法。

2. Dispatcher 中的默认构造方法是个空实现，线程池的加载方式采用的是懒加载，也就是在第一次调用请求的时候初始化。

3. Dispatcher采用的线程池类似于CacheThreadPool，没有核心线程，非核心线程数很大，比较适合执行大量的耗时较少的任务。

4. 当然我们可以通过 setMaxRequests 和 setMaxRequestsPerHost 来修改最大并发请求数和每个主机的最大请求数


先来分析一下同步请求流程：

### 同步请求流程分析

我们通过 okHttpClient.newCall(request).execute() 来发起同步请求，这里实际上调用的是RealCall内部的execute方法：

```
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      client.dispatcher().executed(this);
      return getResponseWithInterceptorChain();
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

RealCall内部的 execute 方法主要做了3件事：

1. 执行 Dispacther 的 executed 方法，将任务添加到同步任务队列中：

```
  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

2. 调用RealCall的 getResponseWithInterceptorChain 方法请求网络，并返回 response。
   从这里可以看到，真正“完成”请求任务的为 getResponseWithInterceptorChain 方法。

3. 无论请求的结果如何，最后都会调用Dispatcher的finished方法，将当前的任务移出同步队列

```
/** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    call.callsPerHost().decrementAndGet();
    finished(runningAsyncCalls, call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call);
  }

  private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      // 将请求移出队列
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }
```

值得关注的一点是，在 finished 一个 runningSyncCalls 时，还会调用 promoteAndExecute 方法，而 promoteAndExecute 主要是对 runningAsyncCalls 的操作。

这里设计的很细致：当同步请求和异步请求数之和达到了最大并发请求数时，当执行完一个同步请求任务时，如果异步请求队列中还有等待执行的任务，
此时就会去取出一个异步任务并执行。除此之外，当 Dispatcher 的 MaxRequests 以及 MaxRequestsPerHost 被改变时，也是会去调用 promoteAndExecute 方法。


#### runningSyncCalls
     
     我们知道 runningSyncCalls 为一个列表，其主要存储客户端发出的所以同步请求（SyncCall），不管是往队列中添加或者移除，都是需要主要状态的同步操作。
其实从相关方法上也可以看到都是 synchronized。


#### 小结

综上流程分析来看，同步请求只是利用了 Dispatcher 的任务队列管理，没有利用 Dispatcher 的线程池，所以 executed 方法是在请求发起的线程中运行的。
所以我们不能直接在 UI 线程中调用 OkHttpClient 的同步请求，否则会报 “NetworkOnMainThread” 错误。


再来看看异步请求的流程吧～


### 异步请求流程分析

我们通过 okHttpClient.newCall(request).enqueue 可以发起一个异步请求，实际上调用的是 RealCall 中的 enqueue 方法。

```
    @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
RealCall 中的 enqueue 方法主要调用 Dispatcher 中的 enqueue 方法，并将传入的 responseCallback 适配成一个 AsyncCall，AsyncCall 可在线程池
中被执行，这里用到了适配器模式。下面通过源码看看 AsyncCall 里主要做了啥：

```
  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    private volatile AtomicInteger callsPerHost = new AtomicInteger(0);

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    AtomicInteger callsPerHost() {
      return callsPerHost;
    }

    void reuseCallsPerHostFrom(AsyncCall other) {
      this.callsPerHost = other.callsPerHost;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    /**
     * Attempt to enqueue this async call on {@code executorService}. This will attempt to clean up
     * if the executor has been shut down by reporting the call as failed.
     */
    void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

1. AsyncCall 继承自 NamedRunnable，而NamedRunnable实现了Runnable接口，在run方法中会调用execute方法，所以当线程池执行AsyncCall时，
   AsyncCall 的 execute 方法就会被调用。

2. AtomicInteger 记录每个 host 的请求数。使用 AtomicInteger，可以确保多线程的情况下（在线程池中执行异步请求），callsPerHost变量（增减操作）的安全性。

3. AsyncCall的 execute 方法通过 getResponseWithInterceptorChain 方法请求网络得到 Response，并将 response 和 RealCall 回调给 responseCallback。
   以上这些方法都是运行在线程池（ExecutorService）中的，不能直接更新UI。

4. AsyncCall的 execute 方法无论请求结果如何，最后都会调用 Dispatcher的finished方法，即将该请求从 runningAsyncCalls 队列中移除。

下面再看看 Dispatcher#finished 具体做了些啥：

```
  void finished(AsyncCall call) {
    call.callsPerHost().decrementAndGet();
    finished(runningAsyncCalls, call);
  }

  private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }
   */
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.
        // 如果当前的请求没有超出每个主机的最大请求数
        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }

```
1. 每次调用finish时，callsPerHost().decrementAndGet()，即每个host的请求数会减一。

2. 将该 asyncCall 添加到 executableCalls 队列中（executableCalls 为 promoteAndExecute 方法内维护的一个队列），然后放到线程池中（executorService）执行。

所以在异步请求中，当达到最大并发请求数时，每执行完（finish）一个任务，就会自动去从待执行的任务队列取出任务来执行。

#### runningAsyncCalls 和 readyAsyncCalls
     
     由于异步请求有最大并发请求数的限制，所以我们需要两个队列来维护异步请求，readyAsyncCalls 为将要运行的异步请求任务队列，而 runningAsyncCalls 为正在运行的异步请求
任务队列，默认情况下，runningAsyncCalls 队列的容量的最大值为64。当有异步请求被处理完成或者并发请求数被修改时，我们需要同时更新 readyAsyncCalls 和 runningAsyncCalls 两个队列的状态。
当然对这两个队列的操作也是需要考虑同步的。


## 小结

1. 综上，异步请求利用了 Dispatcher 的线程池来处理请求。当我们发起一个异步请求时，首先会将我们的请求包装成一个AsyncCall，并加入到 Dispatcher 管理的异步任务队列中，
   如果没有达到最大的请求数量限制(默认值为64)，就会立即调用线程池执行请求。

2. AsyncCall 执行的请求回调方法都是在线程池中调用的，如果想要进行更新UI，可以在回调方法中用 Handler 来切换线程


到了这里，应该对 okhttp 的同步和异步请求流程有了很清晰的认识了，我们知道无论是同步还是异步请求，都是通过 RealCall 内部的
 getResponseWithInterceptorChain 方法来处理具体的网络请求和响应。

那么问题来了，getResponseWithInterceptorChain 是如何处理具体的网络请求和响应的呢？















