# Handler

先上结论：

1 Handler的sendMessage方法做了什么？

  本质上，就是将一个message存储到handler所维护的MessageQueue中。Handler是依附于线程所创建的，在创建时需要先提供一个Looper对象（loop.prepare()）。

2 为什么在UI线程中使用Handler处理消息，不需要调用Looper.prepare()和Looper.loop()

  ActivityThread创建中，已经调用了 Looper.prepareMainLooper() 和 Looper.loop()，来创建 Looper 以及处理主线程Handler MessageQueue中的msg

3 MessageQueue作用

  MessageQueue以msg的time为基准，维护一个优先级队列。

4 Handler何时处理message？

  Handler中的Looper.loop(),会不断的从MessageQueue中取出msg进行处理（当MessageQueue为空时，进入阻塞状态）。取出msg时，会通过msg.target找到是哪个Handler发送这个mgs的，然后调用该Handler的dispatchMessage方法来处理msg。

5 Handler post一个runnable是如何处理的呢？

  Handler会将runnable封装成一个msg对象，并将runnable赋值给msg的一个callback变量。在dispatchMessage，此时检查到callback不为null，会调用callback.run()方法，即runnable的run方法被调用。
  
  
  
  ！[](https://github.com/xianfeng92/android-code-read/blob/master/images/20130817090611984.png)




# Handler 的创建

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
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
   
```

 /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }


 // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();


    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

```

Handler中提供了很多个发送消息的方法，其中除了sendMessageAtFrontOfQueue()方法之外，其它的发送消息方法最终都会辗转调用到sendMessageAtTime()方法中，这个方法的源码如下所示：

```
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
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

```
sendMessageAtTime()方法接收两个参数，其中msg参数就是我们发送的Message对象，而uptimeMillis参数则表示发送消息的时间，它的值等于自系统开机到当前时间的毫秒数再加上延迟时间，如果你调用的不是sendMessageDelayed()方法，延迟时间就为0，然后将这两个参数都传递到MessageQueue的enqueueMessage()方法中。

那么enqueueMessage()方法毫无疑问就是入队的方法了，我们来看下这个方法的源码：

```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;// mMessages对象表示当前待处理的消息,即队列头部
            boolean needWake;
            if (p == null || when == 0 || when < p.when) { //当前队列为空，或者到了msg执行的时间
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {//开启一个死循环
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {//从队列头开始，依据when，将msg插入到队列中
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```
msg.target对应的就是一个发送该消息的Handler，mMessages对象表示当前待处理的消息,即队列头部。enqueueMessage功能就是将msg进行入队操作，所谓的入队其实就是将所有的消息按时间来进行排序。

当然如果你是通过sendMessageAtFrontOfQueue()方法来发送消息的，它也会调用enqueueMessage()来让消息入队，只不过时间为0，这时会把mMessages赋值为新入队的这条消息，然后将这条消息的next指定为刚才的mMessages，这样也就完成了添加消息到队列头部的操作。


现在入队操作我们就已经看明白了，那出队操作是在哪里进行的呢?这个就需要看一看Looper.loop()方法的源码了，如下所示：


```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);// 每当有一个消息出队，就将它传递到dispatchMessage()方法中
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```

可以看到，这个方法会进入了一个死循环，然后不断地调用的MessageQueue的next()方法，我想你已经猜到了，这个next()方法就是消息队列的出队方法。它的简单逻辑就是如果当前MessageQueue中存在mMessages(即待处理消息)，就将这个消息出队，然后让下一条消息成为mMessages，否则就进入一个阻塞状态，一直等到有新的消息入队。每当有一个消息出队，就将它传递到msg.target的dispatchMessage()方法中。

接下来当然就要看一看Handler中dispatchMessage()方法的源码了，如下所示：

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

```

如果mCallback不为空，则调用mCallback的handleMessage()方法，否则直接调用Handler的handleMessage()方法，并将消息对象作为参数传递过去。

另外除了发送消息之外，我们还有以下几种方法可以在子线程中进行UI操作：

1. Handler的post()方法

2. View的post()方法

3. Activity的runOnUiThread()方法


我们先来看下Handler中的post()方法，代码如下所示：

```
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

```
原来这里还是调用了sendMessageDelayed()方法去发送一条消息啊，并且还使用了getPostMessage()方法将Runnable对象转换成了一条消息，我们来看下这个方法的源码：

```
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

```

在这个方法中将消息的callback字段的值指定为传入的Runnable对象。咦？这个callback字段看起来有些眼熟啊，喔！在Handler的dispatchMessage()方法中原来有做一个检查，如果Message的callback等于null才会去调用handleMessage()方法，否则就调用handleCallback()方法。那我们快来看下handleCallback()方法中的代码吧：

```
    private static void handleCallback(Message message) {
        message.callback.run();
    }

```

直接调用了一开始传入的Runnable对象的run()方法。


View的post()方法

```
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // Postpone the runnable until we know on which thread it needs to run.
        // Assume that the runnable will be successfully placed after attach.
        getRunQueue().post(action);
        return true;
    }
```

原来就是调用了Handler中的post()方法，我相信已经没有什么必要再做解释了。

最后再来看一下Activity中的runOnUiThread()方法，代码如下所示：

```
    /**
     * Runs the specified action on the UI thread. If the current thread is the UI
     * thread, then the action is executed immediately. If the current thread is
     * not the UI thread, the action is posted to the event queue of the UI thread.
     *
     * @param action the action to run on the UI thread
     */
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }

```

如果当前的线程不等于UI线程(主线程)，就去调用Handler的post()方法，否则就直接调用Runnable对象的run()方法。还有什么会比这更清晰明了的吗？

# 参考

[Android异步消息处理机制完全解析，带你从源码的角度彻底理解](https://blog.csdn.net/guolin_blog/article/details/9991569)
