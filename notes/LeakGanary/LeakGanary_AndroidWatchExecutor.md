# AndroidWatchExecutor

AndroidWatchExecutor 的主要作用就是等待主线程的空闲。

## 如何等待主线程空闲呢？

两个字，死等～～

### 在主线程的 MEssageQueue 中注册一个监听

AndroidWatchExecutor#execute

```
  @Override public void execute(@NonNull Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
	    } else {
	      postWaitForIdle(retryable, 0);// 切换到主线程执行 waitForIdle 方法
    }
  }
```

在 AndroidWatchExecutor 的 execute()方法中，如果当前所在线程是主线程就直接调用 waitForIdle() 方法，否则调用 postWaitForIdle() 
方法切换到主线后再执行 waitForIdle() 方法。由此可见 execute() 方法是__保证操作在主线程中进行__。

在主线程中调用 waitForIdle，即等待主线程的空闲：

AndroidWatchExecutor#waitForIdle

```
  private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
```

当 waitForIdle 在主线程中时，通过调用 Looper.myQueue() 拿到当前主线程的 MessageQueue 对象，接着给主线程的 MessageQueue 的添加一个 IdleHandler 实例对象，IdleHandler 
是一个接口，该接口中定义的 queueIdle() 方法的返回值为 boolean 类型，当返回true表示继续保持 IdleHandler实例对象，否则移除该实例对象。

综上，我们指定 AndroidWatchExecutor 通过给主线程的 MessageQueue对象添加一个 IdleHandler 回调接口来监听主线程的状态。当主线程空闲时，就会回调 queueIdle，然后 AndroidWatchExecutor
就可以通过调用 postToBackgroundWithDelay 来进行内存泄露的分析。

### 尝试执行内存泄露检查

下面看看 AndroidWatchExecutor#postToBackgroundWithDelay 的源码：

```
  private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
```

postToBackgroundWithDelay 方法首先会切换到后台线程去执行 retryable.run() 方法，如果run()方法返回值为RETRY就继续调用postWaitForIdle()方法循环以上流程否则本次调用结束，
这里的 Retryable 的 run() 方法中调用的是 RefWatcher 的 ensureGone() 方法，调用该方法后，LeakGanary 会对内存泄露情况进行分析。

## 补充

在 AndroidWatchExecutor 的构造函数中：

```
  public AndroidWatchExecutor(long initialDelayMillis) {
    mainHandler = new Handler(Looper.getMainLooper());
    HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
    handlerThread.start();
    backgroundHandler = new Handler(handlerThread.getLooper());
    this.initialDelayMillis = initialDelayMillis;
    maxBackoffFactor = Long.MAX_VALUE / initialDelayMillis;
  }
```

其中：

* mainHandler 主要用于将任务切换到主线程，在这里是将 waitForIdle 切到主线程中执行。

* backgroundHandler 当 AndroidWatchExecutor 通过 waitForIdle，等待到主线程空闲时，就会切换回一个后台线程去执行操作，这个后台线程就是 handlerThread。

* handlerThread 是一个具有 Handler 的 Thread，启动后会一直在后台等待 message。当其 MessageQueue 为空时，其会阻塞休眠（似不似有点类似主线程的消息处理模型）。
  在构建 backgroundHandler 时，传入的是handlerThread 的 Looper 对象，所以  backgroundHandler post 的消息都会被 handlerThread 的 Looper 取出来进行处理。
  
所以说，在 LeakGananry 中所有的内存泄露检查操作都是在 handlerThread 线程中进行。每当 app 中有 Activity 或者 Fragment destroy 后，并且主线程处理空闲时，都会触发 LeakGanary 
来检查相关对象的内存泄露情况。这里的“触发”也就是一个message，这些 message 都会由一个 handlerThread 来处理。在 LeakGanary 中，handlerThread 也就相当于一个“主线程的消息处理系统”。
不得不说，牛逼呀～～～～











































