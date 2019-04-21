# LeakGanary

```
LeakCanary.install(this);
```

## 函数入口和常用类

```
public static @NonNull RefWatcher install(@NonNull Application application) {
return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
.excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
.buildAndInstall();
}
```
refWatcher(application) 方法会f生成一个 AndroidRefWatcherBuilder 对象 , 然后通过 AndroidRefWatcherBuilder 来配置相关信息的:

1. listenerServiceClass 方法中传入 DisplayLeakService.class, 主要用来记录内存泄漏分析结果，然后启动 DisplayLeakActivity 将结果显示出来

2. excludedRefs方法是排除一些开发可以忽略的泄漏路径(一般是系统级别BUG)，这些枚举在AndroidExcludedRefs这个类当中定义 

3. buildAndInstall 这才是重点方法，所以看下: 

AndroidRefWatcherBuilder#buildAndInstall
```
/**
* Creates a {@link RefWatcher} instance and makes it available through {@link
* LeakCanary#installedRefWatcher()}.
*
* Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
*
* @throws UnsupportedOperationException if called more than once per Android process.
*/
public @NonNull RefWatcher buildAndInstall() {
if (LeakCanaryInternals.installedRefWatcher != null) {
throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
}
RefWatcher refWatcher = build();
if (refWatcher != DISABLED) {
if (enableDisplayLeakActivity) {
LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
}
if (watchActivities) {
ActivityRefWatcher.install(context, refWatcher);
}
if (watchFragments) {
FragmentRefWatcher.Helper.install(context, refWatcher);
}
}
LeakCanaryInternals.installedRefWatcher = refWatcher;
return refWatcher;
}
```

在 AndroidRefWatcherBuilder 中 watchActivities 和 watchFragments 默认值都为 true .  所以默认情况下, LeakGanary 是可以自动监测 activity 和 fragment 的内存泄露.
当然我们也可以在 AndroidRefWatcherBuilder 对 watchActivities 和  watchFragments 进行重新配置.

下面重点看一下对 Activity 的内存泄露监测:

ActivityRefWatcher#install

```
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
Application application = (Application) context.getApplicationContext();
// 构建一个 activityRefWatcher 对象
ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
// 将 activityRefWatcher.lifecycleCallbacks 注册到 application.registerActivityLifecycleCallbacks
application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
}
```
registerActivityLifecycleCallbacks 是 application 提供的一个方法, 用来统一管理所有 activity 的生命周期.

activityRefWatcher.lifecycleCallbacks:

```
private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
new ActivityLifecycleCallbacksAdapter() {
@Override public void onActivityDestroyed(Activity activity) {
refWatcher.watch(activity);
}
};
```

LeakCanay 是在 onActivityDestroyed 方法实现监控的, 即只要我们的app中每一个 Activity 被 Destroyed 后,都会被 LeakCanay 所监测.

```
/**
* Watches the provided references and checks if it can be GCed. This method is non blocking,
* the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
* with.
*
* @param referenceName An logical identifier for the watched object.
*/
public void watch(Object watchedReference, String referenceName) {
if (this == DISABLED) {
return;
}
checkNotNull(watchedReference, "watchedReference");
checkNotNull(referenceName, "referenceName");
final long watchStartNanoTime = System.nanoTime();
// key 唯一标识我们所监测的 reference, 并将 key 存入集合 retainedKeys 中
String key = UUID.randomUUID().toString();
retainedKeys.add(key);
//将 reference 封装一个 KeyedWeakReference 对象
final KeyedWeakReference reference =
new KeyedWeakReference(watchedReference, key, referenceName, queue);

ensureGoneAsync(watchStartNanoTime, reference);
}
```
其中:
```
final KeyedWeakReference reference =
new KeyedWeakReference(watchedReference, key, referenceName, queue);
```
主要做了如下三件事:

1.  生成一个弱引用( WeakReference) 指向被监测的对象 (watchedReference)

2. 用唯一的 Id (key) 来标识被监测的对象 (watchedReference). 所有的 key 都会被存储在一个 retainedKeys 集合中.

3. 将生成的弱引用注册(registered)到 ReferenceQueue 中
    注意,这里并不是将弱引用存入到 ReferenceQueue, 只有当一个对象变成弱可达时,才会将其存入 ReferenceQueue 中. 简单的说,ReferenceQueue 中存储的
    对象都是可被 GC 回收的.
    
监测机制利用了Java 的 WeakReference 和 ReferenceQueue，将每个被监测的对象(activity 和 fragment )都包装成一个  KeyedWeakReference . 

```
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
watchExecutor.execute(new Retryable() {
@Override public Retryable.Result run() {
return ensureGone(reference, watchStartNanoTime);
}
});
}
```

重点看看 ensureGone 方法:

```
@SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
long gcStartNanoTime = System.nanoTime();
long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
// WeaklyReachable: 只有弱引用可达的对象,即可以 GC 的对象
// 该方法主要是移除弱可达的对象
removeWeaklyReachableReferences();

if (debuggerControl.isDebuggerAttached()) {
// The debugger can create false leaks.
return RETRY;
}

// 此处的 reference 为我们所监测的对象
// 因为上面已经将 WeaklyReachable 对象移除,即集合 retainedKeys 中不会包含 WeaklyReachable 对象的key
// 如果 reference 的 key 还在集合 retainedKeys 中,则可以判断 reference 不为 WeaklyReachable
if (gone(reference)) {
return DONE;
}
// 到此时说明 reference 还不是 WeaklyReachable
// 触发一次系统的GC
gcTrigger.runGc();
// 再一次移除弱可达对象
removeWeaklyReachableReferences();
// 如果此时 reference 还不是 WeaklyReachable,则说明可能有内存泄露了
if (!gone(reference)) {
long startDumpHeap = System.nanoTime();
long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

File heapDumpFile = heapDumper.dumpHeap();
if (heapDumpFile == RETRY_LATER) {
// Could not dump the heap.
return RETRY;
}
long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
// dump 内存
HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
.referenceName(reference.name)
.watchDurationMs(watchDurationMs)
.gcDurationMs(gcDurationMs)
.heapDumpDurationMs(heapDumpDurationMs)
.build();
// 分析内存快照
heapdumpListener.analyze(heapDump);
}
return DONE;
}
```
上面的方法主要做了如下几件事:

1. 如果被监测的对象为 WeaklyReachable , 将其 key 从 retainedKeys 中移除

2. check 一下被监测的对象是否为 WeaklyReachable, 即 retainedKeys 是否还有其 key

3. 如果 retainedKeys 中不包含被监测对象的 key, 则说明被监测对象不存在内存泄露. 

4. 如果 retainedKeys 中还包含被监测对象的 key, 需要执行一个 GC,再次 check 一下被监测的对象是否为 WeaklyReachable

5. 如果此时被监测对象还不是 WeaklyReachable,  则需要 dump 内存, 分析内存快照


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
// 把.hprof 转为 Snapshot，这个 Snapshot 对象包含了对象引用的所有路径
Snapshot snapshot = parser.parse();// 步骤 1
listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
deduplicateGcRoots(snapshot); // 步骤 2
listener.onProgressUpdate(FINDING_LEAKING_REF);
// 找出泄漏的对象
Instance leakingRef = findLeakingReference(referenceKey, snapshot);// 步骤 3

// False alarm, weak reference was cleared in between key check and heap dump.
if (leakingRef == null) {
String className = leakingRef.getClassObj().getClassName();
return noLeak(className, since(analysisStartNanoTime));
}//找出泄漏对象的最短路径
return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize); 步骤 4
} catch (Throwable e) {
return failure(e, since(analysisStartNanoTime));
}
}
```

上面的checkForLeak方法就是输入.hprof，输出分析结果，主要有以下几个步骤：

1.把.hprof转为Snapshot，这个Snapshot对象就包含了对象引用的所有路径

2.精简gcroots,把重复的路径删除，重新封装成不重复的路径的容器

3.找出泄漏的对象

4.找出泄漏对象的最短路径

重点分析放在第3、4步：

```
private Instance findLeakingReference(String key, Snapshot snapshot) {
ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
if (refClass == null) {
throw new IllegalStateException(
"Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
}
List<String> keysFound = new ArrayList<>();
for (Instance instance : refClass.getInstancesList()) {
List<ClassInstance.FieldValue> values = classInstanceValues(instance);
Object keyFieldValue = fieldValue(values, "key");
if (keyFieldValue == null) {
keysFound.add(null);
continue;
}
String keyCandidate = asString(keyFieldValue);
if (keyCandidate.equals(key)) {
return fieldValue(values, "referent");
}
keysFound.add(keyCandidate);
}
throw new IllegalStateException(
"Could not find weak reference with key " + key + " in " + keysFound);
}
```

1.  通过 KeyedWeakReference.class.getName 泄露对象的类名

2. 根据遍历这个类的所有实例, 如果key值和最开始定义封装的key值相同，那么返回这个泄漏对象,即已经在快照中定位到了泄漏对象了。



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


