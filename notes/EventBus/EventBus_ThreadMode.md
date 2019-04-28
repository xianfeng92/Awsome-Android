# BackgroundPoster

事件 Background 处理，对应 ThreadMode.BACKGROUND，继承自 Runnable。enqueue 函数将事件放到队列中，
并调用线程池执行当前任务，在 run 函数从队列中取事件，invoke 事件响应函数处理。与 AsyncPoster.java 不同的是，
BackgroundPoster 中的任务只在同一个线程中依次执行，而不是并发执行。执行流程图如下所示：

![BackgroundPoster](https://github.com/xianfeng92/Awsome-Android/blob/master/images/BackgroundPoster.png)


适用场景：操作轻微耗时且不会过于频繁，即一般的耗时操作都可以放在这里。
