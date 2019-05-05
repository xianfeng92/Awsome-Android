# HandlerPoster 源码分析

HandlerPoster 的构造方法如下：

```
    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }
```

在 HandlerPoster 的构造方法中会传入 eventBus，looper 以及 maxMillisInsideHandleMessage。eventBus 是用来处理 pendingPost。
looper 决定了 eventBus 在哪个线程中处理 pendingPost。而 maxMillisInsideHandleMessage 为处理 pendingPost 耗时的阀值,关于
为什么要有这个时间阀值，可以参考：[EventBus/issues/369](https://github.com/greenrobot/EventBus/issues/369)。


接着是 HandlerPoster 的入队方法 enqueue()：

```
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

```

入队方法会根据参数创建待处理对象 pendingPost 并加入队列,如果此时 handleMessage() 没有在运行中，即 handlerActive 标志为false,则发送一条空消息让 
handleMessage 响应，接着是 handleMessage() 方法。

```
    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        // 双重校验,类似单例中的实现
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                // 如果订阅者没有取消注册,则处理 pendingPost
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                // 如果在一定时间内仍然没有处理完队列中所有的 pendingPost,则重新发生一个空消息，并退出
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```

1. 关于临时变量 rescheduled

   在 handleMessage 方法中，如果处理 queue 中的 pendingPost 耗时超过 10ms，则会重新发一个空 message 来调用 handleMessage，同时也会将 rescheduled 置为 true，并中断对
   queue 中的 pendingPost 的处理。

2. 关于 handlerActive 标志位

   handlerActive 用来标识当前的 Handler 是否正在处理 pendingPost。当 handlerActive 为 true 时，往 HandlerPoster 中 enqueue 一个 pendingPost 时就不会再次发送空 message
   来调用 handleMessage 方法了。

3. 使用 maxMillisInsideHandleMessage

   使用 maxMillisInsideHandleMessage 可以很好的切割 queue 中的 pendingPost 的处理耗时，即每一次在主线程中 handleMessage 的执行耗时不会超过 10 ms。
   

# 小结

事件主线程处理，对应ThreadMode.MainThread。继承自 Handler，enqueue 函数将事件放到队列中，并利用 handler 发送 message，
handleMessage 函数从队列中取事件，invoke 事件响应函数处理。

![HandlerPoster]()

值得注意的一个细节是：在主线程中执行 Event 事件时，如果事件的执行耗时超过 10ms，此时就会暂停在主线程中执行剩余的其他事件，避免阻塞主线程的正常运行。
