# SubscriberMethodFinder

使用 subscriberMethodFinder#findSubscriberMethods 来找出订阅者所声明的事件响应函数。

```
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
```

findSubscriberMethods 源码如下：

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
        FindState findState = prepareFindState(); 
        findState.initForSubscriber(subscriberClass);
        // 此处为一个循环，目的是找出订阅者以及其父类中的所有的事件响应函数
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            // 获取订阅者的父类的 Class 对象
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

从上面的代码块，我们可以看到，第一步准备 FindState，我们点进去看看：

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
对象后，使用 initForSubscriber方法将订阅者的Class 对象传给 FindState 对象。后面 FindState 就可以从订阅者的 Class 对象中的 Subscribe注解的函数。

```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // 使用反射获取订阅者中所声明的方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    // 获取方法的注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                    // 提取出注解中的参数：threadMode，eventType等
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            // 使用反射获取到的订阅者事件响应函数的相关信息，然后构建一个 SubscriberMethod 方法，并将其添加到 subscriberMethods 列表中。
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

1. 使用反射来获取订阅者中的所有方法（public）。

2. 循环订阅者的方法，找出使用 Subscribe 注解的方法。

3. 解析使用 Subscribe 注解的方法，然后用其构造出一个 SubscriberMethod 对象。


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

1. 取出findState对象搜索出的订阅者事件响应函数

2. 重置findState对象的相关状态，然后将其放入对象池以便下次重复使用，

















