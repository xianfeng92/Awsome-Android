# Android的消息机制概述

Android 的消息机制主要是指 Handler 的运行机制，Handler 的运行需要底层的 MessageQueue 和 Looper 的支撑。MessageQueue 即消息队列，它内部存储了一组消息，以队列的形式对外提供
插入和删除的工作。它的内部存储结构是__采用单链表的数据结构来存储消息列表__。Looper 为消息循环，其会以无限循环的形式去查找和处理 MessageQueue 中的消息。

Looper 中还有一个特殊的概念，那就是 ThreadLocal, ThreadLocal 并不是线程，它的作用是可以在每个线程中存储数据。Handler 创建的时候会采用当前线程的 Looper 来构建消息循环系统，
__ThreadLocal 可以在不同线程中互不干扰地存储并提供数据__，通过 ThreadLocal 可以轻松获取每个线程的 Looper 。


__系统提供 Handler，主要的原因就是解决在子线程中无法访问 UI 的矛盾。__

* 系统为什么不允许在子线程访问 UI 呢？
  这是因为 Android 的 UI 控件不是线程安全的，如果在多线程中并发访问可能会导致 UI 控件处于不可预期的状态

* 那为什么系统不对 UI 控件访问加上锁机制呢？
  首先加上锁后会让 UI 访问的逻辑变得复杂，其次是会降低 UI 的访问频率。所以__最简单高效的就是采用单线程模型来处理 UI 操作__

当 Handler 的 send 被调用的时候，它会调用 MessageQueue 的 enqueueMessage 方法将这个消息插入消息队列，然后 Looper 发现新消息到来时，就会处理这个消息，最终消息的 Runnable 
或者 Handler 的 handlerMessage 方法就被调用，注意 __Looper 是运行在创建 Handler 所在的线程中__，这样 Handler 中的业务就会被切换到所在线程就执行了，如图：

  
  ![](https://github.com/xianfeng92/android-code-read/blob/master/images/20130817090611984.png)

## Handler 的工作原理

Handler 的工作主要是包含消息的发送和接收过程，消息的发送是通过 post 的一系列方法以及 send 的一些列方法来实现的， post 的一系列方法最终是通过 send 的一些列方法来实现的。

发送一条消息的典型过程如下：

```
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }


    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

Handler 发消息仅仅是向消息队列里插入一条消息，MessageQueue 的 next 方法就会返回这条消息给 Looper，最终 Handler 的 dispatchMessage 方法就会被调用。


### 处理消息 dispatchMessage

这个时候 Handler 就进入了处理消息的阶段，dispatchMessage 的实现如下：

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

首先，它会检查 Message 的 callback 是否为 null，不为 null 就通过 handlerCallback 来处理消息，Message 的 callback 是一个 Runnable 对象。
实际上就是 Handler 的 post 方法所传递 Runnable 参数，handlerCallback 的逻辑也很简单。

```
  private static void handleCallback(Message message) {
        message.callback.run();
    }
```

其次是检查 mCallback 是否为null，不为 null 就调用 mCallback 的 handlerMessage 方法来处理消息，Callback是个接口：

```
    public interface Callback {
        public boolean handleMessage(Message msg);
    }

```

通过 Callback 可以采用如下的方式来创建 Handler 对象，Handler handler = new Handler(callback)。

### 那么 callback 的含义在什么呢？

在日常开发中，创建 handler 最常见的方式就是派生一个 handler 子类并重写 handlerMessage 来处理具体的消息，而 Callback 给我们提供了另外一种使用 Handler 的方式，
当我们不想派生子类的时候，就可以通过 Callback 来实现。

最后通过调用 Handler 的 handlerMessage 方法来处理消息，Handler 处理消息的过程：

![handler_dispatchMessage]()


Handler 有一个特殊的构造方法，那就是通过一个特定的 Looper 来构造 Handler，它的实现如下：

```
    public Handler(Looper looper) {
        this(looper, null, false);
    }

```

下面再来看下 Handler 的默认构造方法 public Handler，这个构造方法是调用下面的构造方法，很明显，如果当前线程没有Looper的话，
就会抛出Cant create handler inside thread that has not called Looper.prepare 这个异常，这个解释在没有 Looper 的子线程创建 handler 会引发程序异常的原因。

```
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```


另外除了发送消息之外，我们还有以下几种方法可以在子线程中进行UI操作：

1. Handler 的 post()方法

2. View 的 post() 方法

3. Activity 的 runOnUiThread()方法


## 主线程的消息循环

Android 的主线程就是 ActivityThread，主线程的入口为 main，在 mai n方法中系统通过 Looper.prepareMainLooper 来创建主线程的 Looper 和 MessageQueue，
并且通过 Loop.loop() 来开启主线程的消息循环，这个过程如下所示：


```
 public static void main(String[] args) {
        SamplingProfilerIntegration.start();
        ...
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();
        ...
    }
}
```

主线程消息循环开始了以后，ActivityThread 还需要一个 Handler 来和消息队列进行交互，这个 Handler 就是 ActivityThread.H，
它内部定义了一组消息类型，主要包括四大组件的启动和停止过程，如下所示：

```
 private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        public static final int PAUSE_ACTIVITY          = 101;
        public static final int PAUSE_ACTIVITY_FINISHING= 102;
        public static final int STOP_ACTIVITY_SHOW      = 103;
        public static final int STOP_ACTIVITY_HIDE      = 104;
        public static final int SHOW_WINDOW             = 105;
        public static final int HIDE_WINDOW             = 106;
        public static final int RESUME_ACTIVITY         = 107;
        public static final int SEND_RESULT             = 108;
        public static final int DESTROY_ACTIVITY        = 109;
        public static final int BIND_APPLICATION        = 110;
        public static final int EXIT_APPLICATION        = 111;
        public static final int NEW_INTENT              = 112;
        public static final int RECEIVER                = 113;
        public static final int CREATE_SERVICE          = 114;
        public static final int SERVICE_ARGS            = 115;
        public static final int STOP_SERVICE            = 116;
        public static final int REQUEST_THUMBNAIL       = 117;
        public static final int CONFIGURATION_CHANGED   = 118;
        public static final int CLEAN_UP_CONTEXT        = 119;
        public static final int GC_WHEN_IDLE            = 120;
        public static final int BIND_SERVICE            = 121;
        public static final int UNBIND_SERVICE          = 122;
        public static final int DUMP_SERVICE            = 123;
        public static final int LOW_MEMORY              = 124;
        public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
        public static final int RELAUNCH_ACTIVITY       = 126;
        public static final int PROFILER_CONTROL        = 127;
        public static final int CREATE_BACKUP_AGENT     = 128;
        public static final int DESTROY_BACKUP_AGENT    = 129;
        public static final int SUICIDE                 = 130;
        public static final int REMOVE_PROVIDER         = 131;
        public static final int ENABLE_JIT              = 132;
        public static final int DISPATCH_PACKAGE_BROADCAST = 133;
        public static final int SCHEDULE_CRASH          = 134;
        public static final int DUMP_HEAP               = 135;
        public static final int DUMP_ACTIVITY           = 136;
        public static final int SLEEPING                = 137;
        public static final int SET_CORE_SETTINGS       = 138;
        public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
```

ActivityThread 通过 ApplicationThread 和 AMS 进程进程间通信，AMS 以进程间通信的方法完成 ActivityThread 的请求后回调 ApplicationThread 
的 Binder 方法，然后通过 H 发送消息。H 收到消息后将 ApplicationThread 中的逻辑切换到 ActivityThread 去执行，这就是切换到主线程去执行，
这个过程就是主线程的消息循环模型。


##  Looper 死循环为什么不会导致应用卡死？

线程默认没有 Looper 的，如果需要使用 Handler 就必须为线程创建 Looper。我们经常提到的主线程，也叫 UI 线程，它就是 ActivityThread，ActivityThread
被创建时就会初始化 Looper，这也是在主线程中默认可以使用 Handler 的原因。

Looper.loop() 里面维护了一个死循环方法，也就是说循环在 Looper.prepare() 与 Looper.loop() 之间。在子线程中，如果手动为其创建了 Looper，
那么在所有的事情完成以后应该调用 quit 方法来终止消息循环，否则这个子线程就会一直处于等待（阻塞）状态，而如果退出Looper以后，这个线程就会立刻
（执行所有方法并）终止，因此建议不需要的时候终止 Looper。


这时候如果了解了 ActivityThread，并且在 main 方法中我们会看到主线程也是通过 Looper 方式来维持一个消息循环。

```
public static void main(String[] args) {

        ``````
        Looper.prepareMainLooper();//创建Looper和MessageQueue对象，用于处理主线程的消息

        ActivityThread thread = new ActivityThread();
        thread.attach(false);//建立Binder通道 (创建新线程)

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        //如果能执行下面方法，说明应用崩溃或者是退出了...
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

### 这个死循环会不会导致应用卡死，即使不会的话，它会慢慢的消耗越来越多的资源吗？ 

对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出。

__那么如何保证能一直存活呢？__

简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder 线程也是采用死循环的方法，通过循环方式不同与 Binder 驱动进行读写操作，当然并非简单地死循环，
无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？ 通过创建新线程的方式。真正会卡死主线程的操作是在回调方法 onCreate/onStart/onResume 等操作时间过长，
会导致掉帧，甚至发生 ANR，looper.loop 本身不会导致应用卡死。


#### 主线程的死循环一直运行是不是特别消耗CPU资源呢？ 

其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的 MessageQueue 没有消息时，便阻塞在 loop 的 queue.next() 中的 nativePollOnce() 方法里，此时主线程会释放CPU资源进入休眠状态，
直到下个消息到达或者有事务发生，通过往 pipe 管道写端写入数据来唤醒主线程工作。这里采用的 epoll 机制，是一种 IO 多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，
则立刻通知相应程序进行读或写操作，本质同步 I/O，即读写是阻塞的。所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

##  Handler 是如何能够线程切换

线程间是共享资源的,所以可以在任意线程中调用 Handler 来发送消息。__Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，Looper 在哪个线程创建，就跟哪个线程绑定__，
并且__Handler是在它关联的 Looper 对应的线程中处理消息的__。

那么 Handler 内部如何获取到当前线程的 Looper 呢—–ThreadLocal。ThreadLocal 可以在不同的线程中互不干扰的存储并提供数据，通过 ThreadLocal 可以轻松获取每个线程的 Looper。

