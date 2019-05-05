# SubscriberMethodFinder

SubscriberMethodFinder 类就是查看订阅者对象里面有没有 @Subscribe 注解的方法。怎么做到的？当然是反射。而且这个类用了大量的反射去查找类中方法名。

先看它的变量声明：

```
/**
 * 在较新的类文件，编译器可能会添加一些被 BRIDGE 或 SYNTHETIC 所修饰方法。
 * EventBus 必须忽略这些修饰符所修饰的方法
 */
private static final int BRIDGE = 0x40;
private static final int SYNTHETIC = 0x1000;

//需要忽略的修饰符
private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE |
        SYNTHETIC;

// key:订阅者的类名, value:该类中的事件响应函数集合
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

private static final int POOL_SIZE = 4;
// 对象池
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

```
METHOD_CACHE 可以提供 app 在运行时的缓存，意味着，整个库在运行期间所有遍历的方法都会存在这个 map 中，而不必每次都去做耗时的反射取方法了。


findSubscriberMethods 方法根据订阅者的 Class 对象来解析其所有的事件响应函数，源码如下：

```
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            // 使用反射生成 subscriberMethods
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
           // 使用 apt 获取 subscriberMethods
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

有两种方法获取订阅者的事件响应函数：反射 和 apt。默认情况下会执行上面的 findUsingReflection()，即通过反射来解析注解。

```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        // FindState 用来做订阅方法的校验和保存
        FindState findState = prepareFindState(); 
        findState.initForSubscriber(subscriberClass);
        // 此处为一个循环，目的是找出订阅者以及其父类中的所有的事件响应函数
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            // 获取订阅者的父类的 Class 对象
            findState.moveToSuperclass();
        }
           // 获取findState中的SubscriberMethod(也就是订阅方法List)并返回
        return getMethodsAndRelease(findState);
    }
```

从上面的代码块，我们可以看到，第一步准备 FindState，我们点进去看看:

```
    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```
该方法中使用对象池来提供 FindState 对象，FindState 中维护的就是我们对订阅方法查找结果的封装。当获取到 FindState
对象后，使用 initForSubscriber 方法将订阅者的 Class 对象传给 FindState 对象。后面 FindState 就可以从订阅者的 Class 对象
中解析出使用 Subscribe 注解的函数。

```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // //通过反射得到订阅者的方法数组
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            // NoClassDefFoundError错误的发生，是因为Java虚拟机在编译时能找到合适的类，而在运行时不能找到合适的类导致的错误。
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //遍历 Method
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                 // 保证必须只有一个事件参数
                if (parameterTypes.length == 1) {
                    // 获取方法的注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                    // 提取出注解中的参数：threadMode，eventType等
                        Class<?> eventType = parameterTypes[0];
                     // 校验是否添加该方法
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            // 使用反射获取到的订阅者事件响应函数的相关信息来构建一个 SubscriberMethod 对象，并将其添加到 subscriberMethods 列表中。
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

该方法主要做了如下三件事：

1. 使用反射来获取订阅者中的所有方法。

2. 循环订阅者的方法，找出使用 Subscribe 注解的方法。

3. 解析使用 Subscribe 注解的方法，然后用其构造出一个 SubscriberMethod 对象。

#### 补充

1. 关于 Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149

这是由于某些Android版本在调用GetDeclaredMethods或GetMethods时出现的bug。如果类的方法的参数对设备的 API 级别不可用，则会抛出 NoClassDefFoundError。

EventBus 给出了一些解决这个bug的建议，[A java.lang.NoClassDefFoundError is throw when a subscriber class is registered. What can I do?](http://greenrobot.org/eventbus/documentation/faq/)

其中一个建议是将出现该Error的方法改写成 non-public，这样当 getDeclaredMethods 调用抛出异常时，会在 catch 中尝试调用 getMethods，该方法不会调用 non-public 的方法，
这样也就不会报错了。

2. Replace `getMethods()` with `getDeclaredMethods()`

public Method[] getMethods() 获取某个类的所有公用（public）方法包括其继承类的公用方法。

public Method[] getDeclaredMethods() 获取当前类中所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。

EventBus 的[e47442b](https://github.com/greenrobot/EventBus/commit/e47442b684f04b4d346bb1d0af526908fda7cc1c) 提交中，Replace `getMethods()` with 
`getDeclaredMethods()`。使用 getDeclaredMethods 可以很高效的获取“胖”类的方法，例如 Activity。

findUsingReflection 方法返回的是一个 getMethodsAndRelease(findState)，其源码如下：

```
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        // 重置 findState 变量的状态，然后将其放入对象池中
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

该方法主要做了两件事：

1. 取出findState对象搜索出订阅者的事件响应函数

2. 重置 findState 对象的相关状态，然后将其放入对象池以便下次的复用


