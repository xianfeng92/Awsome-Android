# HeapAnalyzerService

HeapAnalyzerService 的静态方法 runAnalysis() 启动了 HeapAnalyzerService 服务：

在 runAnalysis 方法中传入 context，heapDump，以及 listenerServiceClass。当我传入这
三个参数时，就相当于告诉 HeapAnalyzerService 在 context 所在线程上，进行 heapDump 分析，将分析结果回调给 listenerServiceClass。

```
  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
  }
```

HeapAnalyzerService 为一个 intentService，在其启动后会自动调用 onHandleIntentInForeground() 方法：

```
  @Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);
     // 初始化HeapAnalyzer实例对象heapAnalyzer
    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);
    // 调用heapAnalyzer对象的checkForLeak()方法检测内存泄露
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
   // 把检测结果传送给 AbstractAnalysisResultService
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```

在 onHandleIntentInForeground()方法中通过 HeapAnalyzer 对象的 checkForLeak() 方法来检测分析是否发生了内存泄露。

checkForLeak()源码如下：

```
  public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
      Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
      return failure(exception, since(analysisStartNanoTime));
    }

    try {
      listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
       // 根据内存映射文件生成一个HprofBuffer对象
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
       // 根据HprofBuffer对象生成一个解析器parser
      HprofParser parser = new HprofParser(buffer);
       // 解析内存映射文件并生成相应快照
      listener.onProgressUpdate(PARSING_HEAP_DUMP);
      Snapshot snapshot = parser.parse();
      listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
      // 核查引用路径
      deduplicateGcRoots(snapshot);
      listener.onProgressUpdate(FINDING_LEAKING_REF);
      // 找出泄露对象
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        String className = leakingRef.getClassObj().getClassName();
        return noLeak(className, since(analysisStartNanoTime));
      }
      // 返回泄露的路径
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
```

在checkForLeak()方法中主要是利用square的另外一个开源库[HAHA](https://github.com/square/haha)来做内存检测的。HAHA会根据生成的内存快照文件来分析
引用路径从而确定是否发生了内存泄露。最后把分析结果传递给 listenerServiceClass。

在 onHandleIntentInForeground() 方法执行完毕后 HeapAnalyzerService 服务的生命周期也就结束了。


# 小结：

1. HeapAnalyzerService 使用一个静态方法 runAnalysis 来接收需要处理的任务（heapDump），以及处理结果的回调。
   runAnalysis 会将传参封装成一个 intent，然后再去启动 HeapAnalyzerService。

2. HeapAnalyzerService 运行在runAnalysis的参数 context 所在线程

3. HeapAnalyzerService 为一个 intentService，根据接收的intent来处理一些任务，做完就生命周期就结束，活好不粘人呀。





















