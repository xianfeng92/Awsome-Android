# AsyncPoster

事件异步线程处理，对应 ThreadMode.Async，继承自 Runnable。enqueue 函数将事件放到队列中，并调用线程池执行当前任务，
在 run 函数从队列中取事件，invoke 事件响应函数处理。

## 

![AsyncPoster]()


