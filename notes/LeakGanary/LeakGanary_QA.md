# LeakGanary QA

## 1. LeakGananry 何时开始监测内存泄露

1. 对于 Activity，当其 onActivityDestroyed 方法被回调时，LeakGananry 就开始尝试分析该 Activity 对象是否存在内存泄露。

2. 对于 Fragment，当其 onFragmentViewDestroyed 和 onFragmentDestroyed 被调用时，LeakGananry 就开始尝试分析该 Fragment 对象是否存在内存泄露。

3. 对于我们使用 RefWatch 手动监测的对象，当我们调用 RefWatch.watch 时，LeakGananry 就开始尝试分析该对象是否存在内存泄露。

4. LeakGananry 在主线程空闲时，才会真正开始执行监测对象内存泄露的分析。

## 2. LeakGanary 是如何判断对象的内存泄露

1. 利用 WeakReference 和 ReferenceQueue 机制来判断对象是否被 GC。
   LeakGananry 会使用一个 WeakReference 来引用所监测的对象，并用唯一的 key 来标识该对象。 WeakReference 所引用的对象被回收后，
   这个 WeakReference 对象就会被添加到 ReferenceQueue 队列里，通过其 poll()方法获取到这个被回收的对象的 WeakReference 实例，
   进而知道需要监控的对象是否被回收了。

2. 如果 WeakReference 所引用的对象没有被 GC，LeakGananry 会触发一次 GC，然后再次查看 WeakReference 所引用的对象是否被 GC。

3. 如果 WeakReference 所引用的对象还没有被 GC，此时 LeakGananry 会去 dump 内存, 分析内存快照并找出泄露对象的引用路径。

## 3. 关于 LeakGananry 运行进程

1. 在监测对象是否内存泄露时，LeakGananry 和应用处于同一个进程中。当然，LeakGananry 只在应用主线程空闲时，才会去监测对象是否内存泄露。

2. 只有 LeakGananry 监测的对象出现内存泄露时，LeakGananry 才会启动一个线程用来分析 heapDump
























