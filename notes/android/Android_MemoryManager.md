## Android的内存管理机制

### 共享内存

Android系统通过下面几种方式来实现共享内存:

* Android应用的进程都是从一个叫做Zygote的进程fork出来的。Zygote进程在系统启动并且载入通用的framework的代码与资源之后开始启动。为了启动一个新的程序进程，系统会fork Zygote进程生成一个新的进程，
  然后在新的进程中加载并运行应用程序的代码。这使得大多数的RAM pages被用来分配给framework的代码，同时使得RAM资源能够在应用的所有进程之间进行共享。

* 大多数static的数据被mmapped到一个进程中。这不仅仅使得同样的数据能够在进程间进行共享，而且使得它能够在需要的时候被paged out。常见的static数据包括Dalvik Code，app resources，so文件等。

### 分配与回收内存

* 每一个进程的 Dalvik heap 都反映了使用内存的占用范围。这就是通常逻辑意义上提到的 Dalvik Heap Size，它可以随着需要进行增长，但是增长行为会有一个系统为它设定的上限。

* Android系统并不会对Heap中空闲内存区域做碎片整理。系统仅仅会在新的内存分配之前判断Heap的尾端剩余空间是否足够，如果空间不够会触发gc操作，从而腾出更多空闲的内存空间。
  在Android的高级系统版本里面针对Heap空间有一个Generational Heap Memory的模型，最近分配的对象会存放在 Young Generation区域,当这个对象在这个区域停留的时间达到一定程度，它会被移动到
  Old Generation, 最后累积一定时间再移动到 Permanent Generation 区域。系统会根据内存中不同的内存数据类型分别执行不同的 gc 操作。例如,刚分配到 Young Generation 区域的对象通常更容易被销毁回收，
  同时在 Young Generation 区域的 gc 操作速度会比 Old Generation 区域的 gc 操作速度更快。如下图所示:

![android_memory_gc_mode]()

__每一个Generation的内存区域都有固定的大小,随着新的对象陆续被分配到此区域,当这些对象总的大小快达到这一级别内存区域的阀值时,会触发GC的操作,以便腾出空间来存放其他新的对象__。

补充：

#### 内存抖动

##### 什么是内存抖动呢？

Android里内存抖动是指内存频繁地分配和回收，而频繁的gc会导致卡顿，严重时还会导致OOM。一个很经典的案例是string拼接创建大量小的对象(比如在一些频繁调用的地方打字符串拼接的log的时候)。

##### 内存抖动为什么会引起 OOM 呢？

因为__大量小的对象频繁创建，导致内存碎片，从而当需要分配内存时，虽然总体上还是有剩余内存可分配，而由于这些内存不连续，导致无法分配，系统直接就返回OOM了__。

如下图所示:

![gc_threshold]()

通常情况下，GC发生的时候，所有的线程都是会被暂停的。执行GC所占用的时间和它发生在哪一个Generation也有关系,Young Generation中的每次GC操作时间是最短的，Old Generation其次,Permanent Generation最长。
执行时间的长短也和当前Generation中的对象数量有关,遍历树结构查找20000个对象比起遍历50个对象自然是要慢很多的。

ps:当 Generation　的内存区域达到一定阀值时，会触发垃圾回收。如果 Activity 相互切换时，需要 new 很多的新对象，此时会触发垃圾回收！！！！！　频繁的触发垃圾回收，可能会造成
   Activity切换时的不顺滑，用户体验就会不理想。解决办法～～__延迟一些对象的实例化__。

### 限制应用的内存

* __为了整个Android系统的内存控制需要, Android 系统为每一个应用程序都设置了一个硬性的 Dalvik Heap Size 最大限制阈值__,这个阈值在不同的设备上会因为RAM大小不同而各有差异。

* 如果你的应用占用内存空间已经接近这个阈值, 此时再尝试分配内存的话很容易引起 OutOfMemoryError 错误。ActivityManager.getMemoryClass()　可以用来查询当前应用的Heap 
  Size 阈值, 这个方法会返回一个整数,表明你的应用的 Heap Size 阈值是多少Mb。

  这里也诞生了一些比较”黑科技”的内存优化方案，比如将耗内存的操作放到Native层，或者使用分进程的方式突破每个进程的Dalvik Heap内存限制。

### 应用切换操作

* Android系统并不会在用户切换应用的时候做交换内存的操作。Android 会把那些不包含 Foreground 组件的应用进程放到 LRU Cache 中。例如,当用户开始启动了一个应用,系统会为它创建了一个
  进程但是当用户离开这个应用，此进程并不会立即被销毁，而是会被放到系统的Cache当中，如果用户后来再切换回到这个应用，此进程就能够被马上完整的恢复，从而实现应用的快速切换。
  
* 如果你的应用中有一个被缓存的进程，这个进程会占用一定的内存空间，它会对系统的整体性能有影响。因此当系统开始进入 LowMemory 的状态时，它会由系统根据LRU的规则与应用的优先级，内存占用
  情况以及其他因素的影响综合评估之后决定是否被杀掉。

### Android 进程的优先级

 进程的优先级反应了系统对于进程重要性的判定。在Android系统中，进程的优先级影响着以下三个因素:

1. 当内存紧张时，系统对于进程的回收策略

2. 系统对于进程的CPU调度策略

3. 虚拟机对于进程的内存分配和垃圾回收策略

Anroid基于进程中运行的组件及其状态规定了默认的五个回收优先级：

* Empty process(空进程)

* Background process(后台进程)

* Service process(服务进程)

* Visible process(可见进程)

* Foreground process(前台进程)

系统需要进行内存回收时最先回收空进程,然后是后台进程，以此类推最后才会回收前台进程（一般情况下前台进程就是与用户交互的进程了,如果连前台进程都需要回收那么此时系统几乎不可用了）。

由此也衍生了很多进程保活的方法（提高优先级，互相唤醒，native 保活等等），出现了国内各种全家桶，甚至各种杀不死的进程。

### 优先级的依据

下面来简单列一下应用组件与进程的相关信息：

* 每一个Android的应用进程中, 都可能包含四大组件中的一个/种或者多个/种。

* 对于运行中的 Service 和 ContentProvider 来说，可能有若干个客户端进程正在对其使用。

* __应用进程是由 ActivityManagerService　发送请求让　zygote 创建的, 并且 ActivityManagerService 中对于每一个运行中的进程都有一个 ProcessRecord 对象与之对应__。

ProcessRecord 的简化图如下所示:

![ProcessRecord_and_Components](https://github.com/xianfeng92/android-code-read/blob/master/images/ProcessRecord_and_Components.png)

在　ProcessRecord　中，详细记录了应用组件的相关信息，相关代码如下:

// all activities running in the process---指定进程中运行的所有 Activity
final ArrayList<ActivityRecord> activities = new ArrayList<>();
// all ServiceRecord running in this process---指定进行中运行的所有 service
final ArraySet<ServiceRecord> services = new ArraySet<>();
// services that are currently executing code (need to remain foreground).
final ArraySet<ServiceRecord> executingServices = new ArraySet<>();
// All ConnectionRecord this process holds
final ArraySet<ConnectionRecord> connections = new ArraySet<>();---记录了对于Service连接
// all IIntentReceivers that are registered from this process.--进程中运行的BroadcastReceiver
final ArraySet<ReceiverList> receivers = new ArraySet<>();
// class(String) -> ContentProviderRecord
final ArrayMap<String, ContentProviderRecord> pubProviders = new ArrayMap<>();
// All ContentProviderRecord process is using --记录了对于ContentProvider的连接
final ArrayList<ContentProviderConnection> conProviders = new ArrayList<>();

当一个后台的Service正在被一个前台的Activity使用,那么这个后台的Service就需要设置一个较高的优先级以便不会被回收。（否则后台Service进程一旦被回收，便会对前台的Activity造成影响。）

而所有这些组件的状态就是其所在进程优先级的决定性因素,组件的状态是指：

* Activity是否在前台，用户是否可见

* Service正在被哪些客户端使用

* ContentProvider正在被哪些客户端使用

* BroadcastReceiver 是否正在接受广播

进程是由 ActivityManagerService 发送请求让 zygote 创建，然后由 ActivityManagerService 来管理其要创建的每个进程，如：进程中组件的生命周期回调呀～进程优先级的评定呀～


### 优先级的更新

系统会对处于不同状态的进程设置不同的优先级。但实际上,进程的状态是一直在变化中的。例如：用户可以随时会启动一个新的Activity,或者将一个前台的Activity切换到后台。在这个时候,发生状态变化的Activity
的所在进程的优先级就需要进行更新。并且，Activity可能会使用其他的Service或者ContentProvider。当Activity的进程优先级发生变化的时候,它所使用的Service或者ContentProvider的优先级也应当发生变化。

系统在判定优先级的时候，应当做到公平公正，并且不能让开发者有机可乘。

“公平公正”是指系统需要站在一个中间人的状态下,不偏倚任何一个应用, 公正的将系统资源分配给真正需要的进程。并且在系统资源紧张的时候,回收不重要的进程。

Android系统认为“重要”的进程主要有三类：

* 系统进程

* 前台与用户交互的进程

* 前台进程所使用到的进程

不过对于这一点是有改进的空间的,例如,可以引入用户习惯的分析：如果是用户频繁使用的应用,可以给予这些应用更高的优先级以提升这些应用的响应速度。目前,国内一些Android定制厂商已经开始做这类功能的支持。

“不能让开发者有机可乘”是指: 系统对于进程优先级的判定的因素应当是不能被开发者利用的。因为一旦开发者可以利用,每个开发者都肯定会将自己的设置为高优先级,来抢占更多的资源。

## 总结

### 每一个Generation的内存区域都有固定的大小,随着新的对象陆续被分配到此区域,当这些对象总的大小快达到这一级别内存区域的阀值时,会触发GC的操作,以便腾出空间来存放其他新的对象

1. 当 Generation　的内存区域达到一定阀值时，会触发垃圾回收。如果 Activity 相互切换时，需要 new 很多的新对象，此时会触发垃圾回收！！！！！　频繁的触发垃圾回收，可能会造成
   Activity切换时的不顺滑，用户体验就会不理想。解决办法～～__延迟一些对象的实例化__。

2. 要想办法避免一次创建很多个对象。


### 为了整个Android系统的内存控制需要, Android 系统为每一个应用程序都设置了一个硬性的 Dalvik Heap Size 最大限制阈值。

1. 避免内存泄露。

2. 占用内存多的资源，考虑是不是可以压缩处理。如：bitMap,可以对其采样压缩。

3. 应用很大，真的需要很多内存，可以考虑多进程

### 应用进程是由 ActivityManagerService　发送请求让　zygote 创建的, 并且 ActivityManagerService 中对于每一个运行中的进程都有一个 ProcessRecord 对象与之对应

1. ActivityManagerService 是应用程序进程请求创建和管理的大管家

2. 当应用程序进行中相关组件的状态变化时，ActivityManagerService 会对其记录，并据此来调整其进程优先级。




