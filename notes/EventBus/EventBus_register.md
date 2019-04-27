# EventBus

## register 订阅事件
EventBus 默认可通过静态函数 getDefault 获取单例，当然有需要也可以通过 EventBusBuilder 或 构造函数新建一个 EventBus，每个新建的 EventBus 发布和订阅事件都是相互隔离的，即一个 EventBus 对象中的发布者发布事件，另一个 EventBus 对象中的订阅者不会收到该订阅。

```
EventBus.getDefault.register(this);
```

采用双重锁检查的单例模式:

```
/** Convenience singleton for apps using a process-wide EventBus instance. */
public static EventBus getDefault() {
EventBus instance = defaultInstance;
if (instance == null) {
synchronized (EventBus.class) {
instance = EventBus.defaultInstance;
if (instance == null) {
instance = EventBus.defaultInstance = new EventBus();
}
}
}
return instance;
}
```

当获取到 EventBus 对象后, 我们就可以进行订阅:

```
public void register(Object subscriber) {
Class<?> subscriberClass = subscriber.getClass();
// 根据当前订阅者的 Class 对象,找出其所有的订阅方法
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
synchronized (this) {
// 循环当前订阅者的每个订阅方法,依次执行订阅
for (SubscriberMethod subscriberMethod : subscriberMethods) {
subscribe(subscriber, subscriberMethod);
}
}
}
```
有如下几步:

1. 根据订阅者的 Class 对象, 搜索并存储订阅者中的所有订阅方法(subscriberMethod)

2. 循环所有的订阅方法,调用订阅函数进行订阅

 综上,即订阅者在 EventBus中对其每个订阅方法进行注册.

订阅方法的参数为事件类型(EventType),  而该事件类型也正表明订阅者所关心(订阅)的事件. 当然在订阅方法中, 订阅者可以在收到该类型事件时,
执行一些操作.具体订阅逻辑在 subscribe 方法中,其源码如下:

```
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
// 提取订阅方法中的事件类型
Class<?> eventType = subscriberMethod.eventType;
// 将订阅者和订阅方法封装成一个订阅对象(newSubscription)
Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
// 获取 eventType 的事件的所有订阅对象(Subscription)
CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
if (subscriptions == null) {
subscriptions = new CopyOnWriteArrayList<>();
subscriptionsByEventType.put(eventType, subscriptions);
} else {
// 检查 newSubscription 是否已经注册了 EventBus
if (subscriptions.contains(newSubscription)) {
throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
+ eventType);
}
}

int size = subscriptions.size();
for (int i = 0; i <= size; i++) {
if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
// 根据优先级,将 newSubscription 插入到 subscriptions 列表中.
subscriptions.add(i, newSubscription);
break;
}
}
// 获取订阅者 subscriber 订阅的所有事件类型
List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
if (subscribedEvents == null) {
subscribedEvents = new ArrayList<>();
typesBySubscriber.put(subscriber, subscribedEvents);
}
subscribedEvents.add(eventType);
// 是否为 sticky 事件
if (subscriberMethod.sticky) {
if (eventInheritance) {
// Existing sticky events of all subclasses of eventType have to be considered.
// Note: Iterating over all events may be inefficient with lots of sticky events,
// thus data structure should be changed to allow a more efficient lookup
// (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
for (Map.Entry<Class<?>, Object> entry : entries) {
Class<?> candidateEventType = entry.getKey();
if (eventType.isAssignableFrom(candidateEventType)) {
Object stickyEvent = entry.getValue();
checkPostStickyEventToSubscription(newSubscription, stickyEvent);
}
}
} else {
// 取出 sticky 事件,post 给当前订阅者
Object stickyEvent = stickyEvents.get(eventType);
checkPostStickyEventToSubscription(newSubscription, stickyEvent);
}
}
}
```
首先, 需要了解一下如下三个 hashMap 的作用:

1. subscriptionsByEventType 维护着每一种事件类型(eventType) 到其所有订阅者的一个映射关系.

2. typesBySubscriber 维护着每一个订阅者到其所订阅的事件类型的一个映射关系.

3. stickyEvents 维护着每一种事件类型到其 stickyEvent 的映射关系.

整个 subscribe 方法步骤如下:

1. 将订阅者和订阅方法封装成一个新的订阅对象(newSubscription).

2. 根据订阅方法的事件类型,  获取 eventBus 中关于该事件类型的订阅对象(Subscription)列表.

3. 依据订阅方法的优先级, 将 newSubscription 插入到 eventBus 的订阅对象列表中.

4. 更新订阅者所订阅的事件列表,即 typesBySubscriber

5. 如果订阅方法的事件类型为 stickyEvent, 则取出 stickyEvent,  post给当前订阅者.
