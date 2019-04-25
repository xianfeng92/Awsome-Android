# DecoratorPattern

```
  /**
   * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
   * references (on ICS+).
   */
  public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

在构建 RefWatcher 对象中，会传入一个 DisplayLeakService，其作用就是将内存泄露信息展现出来。

下面看看 AndroidRefWatcherBuilder#listenerServiceClass 方法：

```
  /**
   * Sets a custom {@link AbstractAnalysisResultService} to listen to analysis results. This
   * overrides any call to {@link #heapDumpListener(HeapDump.Listener)}.
   */
  public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }
```

我们知道 listenerServiceClass 用来接收内存泄露信息将其展现出来，而 heapDumpListener 用来监听 heapDump，即当出现
内存泄露时，LeakGananry 会 Heap dump，并生成一个 dumpHeap 文件。生成 dumpHeap 后会通知 heapDumpListener 去分析
这个 dumpHeap 文件，即为 ServiceHeapDumpListener 去分析 dumpHeap 文件。

ServiceHeapDumpListener#analyze：
```
  @Override public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    // 启动一个进行进行 heapDump 的分析，将分析结果回传给 listenerServiceClass
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
```

# 小结

1. 上述的 ServiceHeapDumpListener 就是 listenerServiceClass 的一种 Decorator。原本 listenerServiceClass
   只能接收和显示 dumpHeap 文件的分析结果。经过 Decorator后变成 ServiceHeapDumpListener，其可以单独起一个进程对
   dumpHeap 进行分析，找出对象的内存泄露路径，然后将分析结果显示出来。































