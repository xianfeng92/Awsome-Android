# EventBus

## post 流程

![post](https://github.com/xianfeng92/Awsome-Android/blob/master/images/post.png)

## post 源码分析

post 函数用于发布事件，cancel 函数用于取消某订阅者订阅的所有事件类型、removeStickyEvent 函数用于删除 sticky 事件。

```
/** Posts the given event to the event bus. */
public void post(Object event) {
PostingThreadState postingState = currentPostingThreadState.get();
List<Object> eventQueue = postingState.eventQueue;
eventQueue.add(event);

if (!postingState.isPosting) {
postingState.isMainThread = isMainThread();
postingState.isPosting = true;
if (postingState.canceled) {
throw new EventBusException("Internal error. Abort state was not reset");
}
try {
while (!eventQueue.isEmpty()) {
postSingleEvent(eventQueue.remove(0), postingState);
}
} finally {
postingState.isPosting = false;
postingState.isMainThread = false;
}
}
}
```

post 函数会首先得到当前线程的 post 信息 PostingThreadState，其中包含事件队列，将当前事件添加到其事件队列中，然后循环调用 
postSingleEvent 函数发布队列中的每个事件。值得注意的是，PostingThreadState 是存储在 ThreadLocal 中的，即每个线程都会有
一个不同 PostingThreadState。PostingThreadState 的结构如下:

```
/** For ThreadLocal, much faster to set (and get multiple values). */
final static class PostingThreadState {
final List<Object> eventQueue = new ArrayList<>();
boolean isPosting;
boolean isMainThread;
Subscription subscription;
Object event;
boolean canceled;
}
```
PostingThreadState 中会有一个 eventQueue 列表来存储当前线程中需要 post 的所有事件。使用 isPosting 来标记当前线程是否正在 post 事件，
使用 isMainThread 标记当前线程是否为主线程，使用 canceled 标记当前线程的 post 操作是否被取消。



循环调用 postSingleEvent 方法来分发当前线程的 eventQueue 列表中的事件,其源码如下:

```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
// 获取当前 post 的事件的 Class 对象
Class<?> eventClass = event.getClass();
boolean subscriptionFound = false;
if (eventInheritance) {
List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
int countTypes = eventTypes.size();
for (int h = 0; h < countTypes; h++) {
Class<?> clazz = eventTypes.get(h);
subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
}
} else {
subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
}
if (!subscriptionFound) {
if (logNoSubscriberMessages) {
logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
}
if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
eventClass != SubscriberExceptionEvent.class) {
post(new NoSubscriberEvent(this, event));
}
}
}
```


### postSingleEventForEventType 

在 subscriptionsByEventType 中查找该事件的订阅者列表 然后调用 postToSubscription 函数向每个订阅者发布事件。

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
CopyOnWriteArrayList<Subscription> subscriptions;
synchronized (this) {
// 找出订阅 eventClass 类型事件的 Subscription 列表
subscriptions = subscriptionsByEventType.get(eventClass);
}
if (subscriptions != null && !subscriptions.isEmpty()) {
// 循环遍历每个 subscription, 依次将事件 post 到 subscription
for (Subscription subscription : subscriptions) {
postingState.event = event;
postingState.subscription = subscription;
boolean aborted = false;
try {
postToSubscription(subscription, event, postingState.isMainThread);
aborted = postingState.canceled;
} finally {
postingState.event = null;
postingState.subscription = null;
postingState.canceled = false;
}
if (aborted) {
break;
}
}
return true;
}
return false;
}
```

### postToSubscription

向每个订阅者发布事件, postToSubscription 函数中会判断订阅者的 ThreadMode，从而决定在什么 Mode 下执行事件响应函数。

```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
switch (subscription.subscriberMethod.threadMode) {
case POSTING:
invokeSubscriber(subscription, event);
break;
case MAIN:
if (isMainThread) {
invokeSubscriber(subscription, event);
} else {
mainThreadPoster.enqueue(subscription, event);
}
break;
case MAIN_ORDERED:
if (mainThreadPoster != null) {
mainThreadPoster.enqueue(subscription, event);
} else {
// temporary: technically not correct as poster not decoupled from subscriber
invokeSubscriber(subscription, event);
}
break;
case BACKGROUND:
if (isMainThread) {
backgroundPoster.enqueue(subscription, event);
} else {
invokeSubscriber(subscription, event);
}
break;
case ASYNC:
asyncPoster.enqueue(subscription, event);
break;
default:
throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
}
}
```

### ThreadMode 共有四类

#### 1.POSTING

默认的 ThreadMode，表示在执行 Post 操作的线程直接调用订阅者的事件响应方法，不论该线程是否为主线程（UI 线程）。当该线程为主线程时，
响应方法中不能有耗时操作，否则有卡主线程的风险。适用场景：对于是否在主线程执行无要求，但若 Post 线程为主线程，不能耗时的操作。

#### 2.MAIN

在主线程中执行响应方法。如果发布线程就是主线程，则直接调用订阅者的事件响应方法，否则通过主线程的 Handler 发送消息在主线程中处理——调用
订阅者的事件响应函数。显然，MainThread类的方法也不能有耗时操作，以避免卡主线程。适用场景：必须在主线程执行的操作。


#### 3.BACKGROUND

在后台线程中执行响应方法。如果发布线程不是主线程，则直接调用订阅者的事件响应函数，否则启动唯一的后台线程去处理。由于后台线程是唯一的，
当事件超过一个的时候，它们会被放在队列中依次执行，因此该类响应方法虽然没有PostThread类和MainThread类方法对性能敏感，但最好不要有重度耗
时的操作或太频繁的轻度耗时操作，以造成其他操作等待。适用场景：操作轻微耗时且不会过于频繁，即一般的耗时操作都可以放在这里。

#### 4.ASYNC

不论发布线程是否为主线程，都使用一个空闲线程来处理。和 BackgroundThread 不同的是，Async 类的所有线程是相互独立的，因此不会出现卡线程的问题。
适用场景：长耗时操作，例如网络访问。






























