## HandlerPoster

事件主线程处理，对应ThreadMode.MainThread。继承自 Handler，enqueue 函数将事件放到队列中，并利用 handler 发送 message，
handleMessage 函数从队列中取事件，invoke 事件响应函数处理。

![HandlerPoster]()


值得注意的一个细节是：在主线程中执行 Event 事件时，如果事件的执行耗时超过 10ms，此时就会暂停在主线程中执行剩余的其他事件，
避免阻塞主线程的正常运行。
