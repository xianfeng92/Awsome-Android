# BackgroundPoster 源码分析

BackgroundPoster 中有一个待处理任务队列,PendingPostQueue queue，进到队列里面再看 定义了两个节点,从字面上理解就是
队列的头节点和尾节点。

```
private PendingPost head; //待处理对象队列 头节点
private PendingPost tail;//待处理对象队列 尾节点
```

再看这个 PendingPost 类的实现:

```
//单例池,复用对象
private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

Object event; // 事件类型
Subscription subscription; // 代表一个订阅，即订阅者和其事件响应函数的封装
PendingPost next; // 队列中下一个待处理对象
```

首先是提供了一个池的设计，类似于我们的线程池，目的是为了减少对象创建的开销。当一个对象不用了，我们可以留着它，下次再需要的时候返回这个
保留的而不是再去创建。


BackgroundPoster 的入队方法enqueue()：

```
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
    }
```

入队方法会根据参数创建 待处理对象 pendingPost 并加入队列。如果此时后台线程没有在处理 PendingPostQueue 中的待处理任务，即 executorRunning 为false。则
会调用 CachedThreadPool 去处理 BackgroundPoster，即调用 BackgroundPoster#run 方法。

BackgroundPoster#run 源码如下：

```
    public void run() {
        try {
            try {
                // 开启一个死循环
                while (true) {
                    // 不断的从 queue 中取出待处理的任务
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                    // 同步代码块
                        synchronized (this) {
                            // Check again, this time in synchronized
                            // 如果取出的 pendingPost 为 null，会再次同步尝试一个
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    // 执行 pendingPost 任务
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            // 重置 executorRunning 状态
            executorRunning = false;
        }
    }
```

# 小结

事件 Background 处理，对应 ThreadMode.BACKGROUND，继承自 Runnable。enqueue 函数将事件放到队列中，
并调用线程池执行当前任务，在 run 函数从队列中取事件，invoke 事件响应函数处理。与 AsyncPoster.java 不同的是，
BackgroundPoster 中的任务只在同一个线程中依次执行，而不是并发执行。执行流程图如下所示：

![BackgroundPoster](https://github.com/xianfeng92/Awsome-Android/blob/master/images/BackgroundPoster.png)

适用场景：操作轻微耗时且不会过于频繁，即一般的耗时操作都可以放在这里。




























