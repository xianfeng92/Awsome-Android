# EventBus

## post 发布事件

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

其中, PostingThreadState 的结构如下:

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
利用 PostingThreadState 来对 event 发送前的状态进行检查, 如: 事件是否正在分发中,以及 event 事件是否被取消. 而真正进行 post 事件的是 postSingleEvent 方法,其源码如下:

```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
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

在 subscriptionsByEventType 中查找该事件的订阅者队列, 然后调用 postToSubscription 函数向每个订阅者发布事件。

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


