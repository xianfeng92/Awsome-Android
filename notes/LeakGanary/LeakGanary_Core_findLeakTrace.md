
## checkForLeak

使用 referenceKey 从 heapDump 中找到对应 KeyedWeakReference 引用所指向的实例,即存在内存泄露的对象。

```
  /**
   * Searches the heap dump for a {@link KeyedWeakReference} instance with the corresponding key,
   * and then computes the shortest strong reference path from that instance to the GC roots.
   */
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
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      listener.onProgressUpdate(PARSING_HEAP_DUMP);
      // 使用 haha 将 heapDump 转换为 snapshot
      Snapshot snapshot = parser.parse();
      listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
      deduplicateGcRoots(snapshot);
      listener.onProgressUpdate(FINDING_LEAKING_REF);
      // 从 snapshot 中找到内存泄露对象的引用
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        String className = leakingRef.getClassObj().getClassName();
        return noLeak(className, since(analysisStartNanoTime));
      }
      // 寻找内存泄露对象引用 leakingRef 到 GCRoots 的最短路径
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
```

使用 haha 解析得到的 Snapshot 会罗列出内存中各个对象，它通过对象之间的引用链关系构成了一棵树，在这个树中可以查询到各个对象的信息，包括它的Class对象信息、
内存地址、持有的引用及被持有的引用关系等。得到 Snapshot 对象后，根据 referenceKey 在 Snapshot 中搜索（__findLeakingReference__）出内存泄露对象的位置。

## findLeakingReference

从 snapshot 中找到 key 所对应的内存泄露对象。

```
  private Instance findLeakingReference(String key, Snapshot snapshot) {
    // 从 snapshot 中找出 KeyedWeakReference Class
    ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
    if (refClass == null) {
      throw new IllegalStateException(
          "Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
    }
    List<String> keysFound = new ArrayList<>();
    // 遍历 KeyedWeakReference Class 所对应的所有 KeyedWeakReference 对象
    for (Instance instance : refClass.getInstancesList()) {
      List<ClassInstance.FieldValue> values = classInstanceValues(instance);
      Object keyFieldValue = fieldValue(values, "key");
      if (keyFieldValue == null) {
        keysFound.add(null);
        continue;
      }
      String keyCandidate = asString(keyFieldValue);
      if (keyCandidate.equals(key)) {
        // 比对 snapshot 中 KeyedWeakReference 对象的 key 值和我们需要搜索的 key。如果一致，则返回该对象。
        return fieldValue(values, "referent");
      }
      keysFound.add(keyCandidate);
    }
    throw new IllegalStateException(
        "Could not find weak reference with key " + key + " in " + keysFound);
  }
```
为了能够准确找到被泄漏对象，LeakCanary 通过被泄漏对象的弱引用来在 Snapshot 中定位它。因为，如果一个对象被泄漏，一定也可以在内存中找到这个对象的弱引用，
再通过弱引用对象的 referent 就可以直接定位被泄漏对象。下一步的工作就是找到一条有效的到被泄漏对象的最短的引用，这通过__findLeakTrace__来实现。


## findLeakTrace

找到 snapshot 中，从 leakingRef 到 GCRoots 的最短路径：

```
  private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
      Instance leakingRef, boolean computeRetainedSize) {

    listener.onProgressUpdate(FINDING_SHORTEST_PATH);
    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);

    String className = leakingRef.getClassObj().getClassName();

    // False alarm, no strong reference path to GC Roots.
    if (result.leakingNode == null) {
      return noLeak(className, since(analysisStartNanoTime));
    }

    listener.onProgressUpdate(BUILDING_LEAK_TRACE);
    LeakTrace leakTrace = buildLeakTrace(result.leakingNode);

    long retainedSize;
    if (computeRetainedSize) {

      listener.onProgressUpdate(COMPUTING_DOMINATORS);
      // Side effect: computes retained size.
      snapshot.computeDominators();

      Instance leakingInstance = result.leakingNode.instance;

      retainedSize = leakingInstance.getTotalRetainedSize();

      // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
      if (SDK_INT <= N_MR1) {
        listener.onProgressUpdate(COMPUTING_BITMAP_SIZE);
        retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
      }
    } else {
      retainedSize = AnalysisResult.RETAINED_HEAP_SKIPPED;
    }

    return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
        since(analysisStartNanoTime));
  }
```

在 findLeakTrace 方法中，主要通过 ShortestPathFinder#findPath 方法，在 snapshot 中，找到一条从 
leakingRef 到 GCRoot 对象的最短路径。


































