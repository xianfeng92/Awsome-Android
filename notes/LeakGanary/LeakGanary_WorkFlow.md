# LeakGanary Work Flow

![leakcanary](https://github.com/xianfeng92/android-code-read/blob/master/images/leakcanary.png)

### 1. RefWatcher.watch() creates a KeyedWeakReference to the watched object.
         
RefWatcher.watch() 方法创建一个 KeyedWeakReference 引用指向被监测的对象

几点说明:

1. KeyedWeakReference 为一个弱引用

2. 每个 KeyedWeakReference 内部都有一个唯一的 key 来标识

3. 在构建 KeyedWeakReference 对象时, 会将其注册到一个 ReferenceQueue 上

### 2. Later, in a background thread, it checks if the reference has been cleared and if not it triggers a GC.

当应用程序中有activity 或者 fragment 被 destroy 后, LeakGanary 会通过一个后台线程去check 该对象是否为弱可达. 如果不是, Ganary 会触发一次 GC

### 3. If the reference is still not cleared, it then dumps the heap into a .hprof file stored on the file system.

如果弱引用对象仍然没有被清除，说明内存泄漏了，系统就导出 hprof 文件，保存在 app 的文件系统目录下

### 4. HeapAnalyzerService is started in a separate process and HeapAnalyzer parses the heap dump using HAHA.
HeapAnalyzerService 启动一个单独的进程，使用 HeapAnalyzer 来分析 hprof 文件。它使用另外一个开源库 HAHA。

### 5. HeapAnalyzer finds the KeyedWeakReference in the heap dump thanks to a unique reference key and locates the leaking reference.

HeapAnalyzer 通过查找 KeyedWeakReference 弱引用对象来查找内在泄漏

### 6. HeapAnalyzer computes the shortest strong reference path to the GC Roots to determine if there is a leak, and then builds the chain of references causing the leak.

HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄露。如果是的话，建立导致泄露的引用链。

### 7. The result is passed back to DisplayLeakService in the app process, and the leak notification is shown.

内存泄漏信息送回给 DisplayLeakService，它是运行在 app 进程里的一个服务。然后在设备通知栏显示内存泄漏信息。
