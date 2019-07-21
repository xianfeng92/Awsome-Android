# Awsome-Android

------------------------------------------

# Open Source Code Read

-----------------------------------------
## EventBus

* [EventBus 思维导图](https://github.com/xianfeng92/Awsome-Android/blob/master/images/EventBus_mind.png)

* [EventBus_register](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_register.md)

* [EventBus_post](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_post.md)

* [EventBus_ThreadMode](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_ThreadMode.md)

* [BackgroundPoster](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_BackgroundPoster.md)

* [PendingPostPool](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_PendingPostPool.md)

* [SubscriberMethodFinde](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_SubscriberMethodFinder_Reflection.md)

* [HandlerPoster](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_HandlerPoster.md)

* [AsyncPoster](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/EventBus/EventBus_AsyncPoster.md)

* [野生 EventBus 文档](https://blog.boileryao.com/2018/02/eventbus-doc-cn/)

----------------------------------------------

## RxJava

* [Rxjava 基本使用](https://github.com/xianfeng92/android-code-read/blob/master/notes/Rxjava/Rxjava.md)

* [Rxjava 源码解读](https://github.com/xianfeng92/android-code-read/blob/master/notes/Rxjava/Rxjava_Code.md)

* [Rxjava 线程调度](https://github.com/xianfeng92/android-code-read/blob/master/notes/Rxjava/Rxjava_Scheduler.md)

* [Rxjava 设计模式](https://github.com/xianfeng92/android-code-read/blob/master/notes/Rxjava/Rxjava_design.md)

----------------------------------------

## Dagger

* [DAGGER基础](https://github.com/xianfeng92/android-code-read/blob/master/notes/Dagger2/Dagger_base.md)

* [使用 @Inject 注入过程分析](https://github.com/xianfeng92/android-code-read/blob/master/notes/Dagger2/Dagger_Analysis_Inject.md)

* [使用 @Provides 注入过程分析](https://github.com/xianfeng92/android-code-read/blob/master/notes/Dagger2/Dagger_Analysis_Provider.md)

-----------------------------------------
## Retrofit

--------------------------------------

## Okhttp

----------------------------------------

## Glide

-----------------------------------------

## LeakCanary

* [用 LeakCanary 检测内存泄漏](https://academy.realm.io/cn/posts/droidcon-ricau-memory-leaks-leakcanary/)

* [LeakGanary_basicUse](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_basicUse.md)

* [LeakGanary](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary.md)

* [LeakGanary_WorkFlow](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_WorkFlow.md)

* [LeakGanary_FragmentRefWatcher](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_FragmentRefWatcher.md)

* [LeakGanary_HeapAnalyzerService](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_HeapAnalyzerService.md)

* [LeakGanary_PatternDesign](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_PatternDesign.md)

-------------------------------------------
## fastjson

* [fastjson](https://github.com/alibaba/fastjson)

* [JSON最佳实践](http://kimmking.github.io/2017/06/06/json-best-practice/)

---------------------------------------------

## Design Patterns
----------------
### 创建型模式

#### 构建者模式

##### EventBus 实例的构建
* [EventBusBuilder](https://github.com/greenrobot/EventBus/blob/master/EventBus/src/org/greenrobot/eventbus/EventBusBuilder.java)

##### Dagger 中构建者模式

* [Dagger中的构建者模式](https://github.com/xianfeng92/android-code-read/blob/master/notes/Dagger2/Pattern_BuilderInDagger.md)

#### 单例模式
##### 双重锁检查
* [EventBus getDefault实现](https://github.com/greenrobot/EventBus/blob/2e7c0461106a3c261859579ec6fdcb5353ed31a4/EventBus/src/org/greenrobot/eventbus/EventBus.java#L80)

#### 工厂方法模式

#### 抽象工厂模式

##### Dagger中的简单工厂模式

* [Pattern ProviderInDagger](https://github.com/xianfeng92/android-code-read/blob/master/notes/Dagger2/Pattern_ProviderInDagger.md)

----------
### 结构型模式

#### 适配器模式

##### Rxjava 中使用 ObservableCreate 适配 ObservableOnSubscribe 和 Observable

*[ObservableCreate](https://github.com/ReactiveX/RxJava/blob/cc7fce89c46e67c4a124da9e206e4c770b6f8850/src/main/java/io/reactivex/internal/operators/observable/ObservableCreate.java)

#### 装饰器模式

#### Rxjava 中使用 MapObserver 去装饰 Observer

* [MapObserver](https://github.com/ReactiveX/RxJava/blob/cc7fce89c46e67c4a124da9e206e4c770b6f8850/src/main/java/io/reactivex/internal/operators/observable/ObservableMap.java#L35)

#### 外观模式

--------
### 行为型模式

#### 观察者模式

##### 在一个类中注册 EventBus 实例,也就是在订阅 EventBus 的相关事件

* [register](https://github.com/greenrobot/EventBus/blob/2e7c0461106a3c261859579ec6fdcb5353ed31a4/EventBus/src/org/greenrobot/eventbus/EventBus.java#L140)

#### 策略模式

##### AndroidAutoSize 中屏幕适配

* [AutoAdaptStrategy](https://github.com/JessYanCoding/AndroidAutoSize/blob/b1518770bf537eba26488ba4f6e548bca77ece3e/autosize/src/main/java/me/jessyan/autosize/AutoAdaptStrategy.java)

#### 责任链模式

----------------------------------------------

# Java

* [hashMap 思维导图](https://github.com/xianfeng92/Awsome-Android/blob/master/images/hashMap_Mind.png)
* [强引用、弱引用、软引用、虚引用](https://www.jianshu.com/p/1fc5d1cbb2d4)
* [ReferenceQueue的使用](https://www.jianshu.com/p/73260a46291c)
* [SparseArray](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/SparseArray.md)
* [ArrayList](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/ArrayList.md)
* [LinkedList](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/LinkedList.md)
* [ConcurrentHashMap_Intro](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/ConcurrentHashMap_Intro.md)
* [ConcurrentHashMap_get](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/ConcurrentHashMap_get.md)
* [ConcurrentHashMap_put](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/ConcurrentHashMap_put.md)
* [BinarySearchTree](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/BinarySearchTree.md)
* [LinkedHashMap](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/LinkedHashMap.md)
* [HashSet](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/HashSet.md)
* [hashTable](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/DataStructure/hashTable.md)

---------------------------------------------

# Android

## Android 消息处理机制

* [Handler_Analysis](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_Messaging_Mechanism/Handler_Analysis.md)

* [Looper_Analysis](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_Messaging_Mechanism/Looper_Analysis.md)

* [MessageQueue_Analysis](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_Messaging_Mechanism/MessageQueue_Analysis.md)

* [ThreadLocal_Analysis](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_Messaging_Mechanism/ThreadLocal_Analysis.md)

* [MainThreadMessageLooper](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/MainThreadMessageLooper.md)

* [AIDL_Analysis](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/AIDL/AIDL_Analysis.md)

## 四大组件

### Activity相关

* [Activity启动流程](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Activty_Start.md)

* [Task And BackStack](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Activity_Task.md)

* [Activty 中的一些重要知识点](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Activity_matter.md)

* [Fragment](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Fragment%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

* [Android Fragments developer](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Fragments_developer.md)

* [Context基本使用](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/ContextBaseUse.md)

* [Context分析](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Context.md)

### Service相关

* [Service](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Service.md)

* [Android_Service_Demo](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Service_Demo.md)

* [Service 中的一些重要知识点](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Service_matter.md)

* [BroadCastRecevier](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Broadcasts.md)

## View 事件体系

### View相关

* [Android_View_Introduce](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_View/Android_View_Introduce.md)

* [Andoird_view_LayoutInflater](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_View/Andoird_view_LayoutInflater.md)

* [Android_view_Measure](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_View/Android_view_Measure.md)

* [Android_view_Layout](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_View/Android_view_Layout.md)

* [Android_view_Draw](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_view_Draw.md)

* [Android_window](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_view_Draw.md)

* [Android_view_customView](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_view_customView.md)

### 事件

* [MotionEvent](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Andoird_view_event.md)

* [Android_EventDispatch_Visible](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_EventDispatch_Visible.md)

* [EventBus 源码解析](http://a.codekk.com/detail/Android/Trinea/EventBus%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

## Android 内存

* [Android_MemoryManager](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_MemoryManager.md)

* [Android_OOM](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_OOM.md)

* [Android_memoryLeak](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_memoryLeak.md)

* [Android_GC](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_GC.md)

* [内存溢出和内存泄漏的区别、产生原因以及解决方案](https://www.cnblogs.com/Sharley/p/5285045.html)

## Android 异步

* [Android_ThreadBase](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_ThreadBase.md)

* [Android_Demo_Thread](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Demo_Thread.md)

* [Android_Guide_background_processing](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Guide_background_processing.md)

* [Android_ManagerForMultipleThreads](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_ManagerForMultipleThreads.md)

* [Android_Communicate_with_UIThread](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_Communicate_with_UIThread.md)

* [Android_RunCode_Threadpool](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/Android_RunCode_Threadpool.md)

* [MultiThread_start](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/MultiThread_start.md)

* [AsyncTask](https://github.com/xianfeng92/android-code-read/blob/master/notes/android/AsyncTask%E5%88%86%E6%9E%90.md)

* [Android_Thread 小结](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/android/Android_Thread.md)

* [handlerThread:看看在 LeakGananry 中的使用](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_AndroidWatchExecutor.md)

* [intentService:看看在 LeakGananry 中的使用](https://github.com/xianfeng92/Awsome-Android/blob/master/notes/LeakGanary/LeakGanary_HeapAnalyzerService.md)
----------------------------------------------
# SOLID

* [单一职责原则](https://github.com/xianfeng92/Awsome-Android/blob/master/SOLID/SingleResponsibilityPrinciple.md)

* [开-闭原则](https://github.com/xianfeng92/Awsome-Android/blob/master/SOLID/OpenClosedPrinciple.md)

* [里氏替换原则](https://github.com/xianfeng92/Awsome-Android/blob/master/SOLID/LiskovSubstitutionPrinciple.md)

* [接口隔离原则](https://github.com/xianfeng92/Awsome-Android/blob/master/SOLID/InterfaceSegregationPrinciple.md)

* [依赖倒置原则](https://github.com/xianfeng92/Awsome-Android/blob/master/SOLID/DependencyInversionPrinciple.md)
