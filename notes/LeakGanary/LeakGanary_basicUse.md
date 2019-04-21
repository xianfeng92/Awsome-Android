
## 使用方法

在 build.gradle 中加入引用，不同的编译使用不同的引用:

```
dependencies {
debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
// Optional, if you use support library fragments:
debugImplementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.3'
}
```
在 Application 中:

```
public class ExampleApplication extends Application {

@Override public void onCreate() {
super.onCreate();
if (LeakCanary.isInAnalyzerProcess(this)) {
// This process is dedicated to LeakCanary for heap analysis.
// You should not init your app in this process.
return;
}
LeakCanary.install(this);
// Normal app init code...
}
}
```
这样，就万事俱备了！ 在 debug build 中，如果检测到某个 activity 有内存泄露，LeakCanary 就是自动地显示一个通知。


## 使用 RefWatcher 监控那些本该被回收的对象

```
...
RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
refWatcher.watch(someObjNeedGced);
```
当 someObjNeedGced 还在内存中时，就会在 logcat 里看到内存泄漏的提示。


## 几个有意思的问题

### 如何导出 hprof 文件
```
File heapDumpFile = new File("heapdump.hprof");
Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
```
可以参阅 AndroidHeapDumper.java 的代码。

### 如何分析 hprof 文件

这是个比较大的话题，感兴趣的可以移步另外一个开源库 [HAHA](https://github.com/square/haha)，它的祖先是 MAT。

### 如何使用 HandlerThread

可以参阅AndroidWatchExecutor.java的代码，特别是关于 Handler, Loop 的使用。

### 怎么知道某个变量已经被 GC 回收

可以参阅 RefWatcher.java 的 ensureGone() 函数。最主要是利用 WeakReference 和 ReferenceQueue 机制。简单地讲，就是当弱引用 WeakReference 所引用的对象被回收后，这个 WeakReference 对象就会被添加到 ReferenceQueue 队列里，我们可以通过其 poll()方法获取到这个被回收的对象的 WeakReference 实例，进而知道需要监控的对象是否被回收了。

### 关于内存泄漏

内存泄漏可能很容易发现，比如 Cursor 没关闭；比如在 Activity.onResume() 里 register 了某个需要监听的事件，但在 Activity.onPause() 里忘记 unregister 了；内存泄漏也可能很难发现，比如 LeakCanary 示例代码，隐含地引用，并且只有在旋转屏幕时才会发生。还有更难发现，甚至无能为力的内存泄漏，比如 Android SDK 本身的 BUG 导致内存泄漏。AndroidExcludedRefs.java 里就记录了一些己知的 AOSP 版本的以及其 OEM 实现版本里存在的内存泄漏。

