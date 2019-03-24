## OOM

Android和java中都会出现由于不良代码引起的内存泄露,为了使Android应用程序能够快速高效的运行,Android每个应用程序都会有专门Dalvik虚拟机实例来运行，也就是每个程序都在属于自己的进程中运行。这样，某个应用程序内存泄露仅仅只会使自己进程被kill掉不会影响其他进程（如果是systemprocess等系统进程出现问题，就会造成系统重启）,另一方面，系统为每一个应用程序分配了不同的内存上限，如果超过这个上限被视为内存泄露，从而被kill掉。Dalvik Heapsize因不同设备的RAM不同而有所差异，应用占用内存接近这个阀值，在尝试分配内存就会引起outofmemoryError的错误。

## 如何避免OOM总结

### 减小对象的内存占用

避免OOM的第一步就是要尽量减少新分配出来的对象占用内存的大小,尽量使用更加轻量的对象。

1. 使用更加轻量的数据结构

例如,我们可以考虑使用ArrayMap/SparseArray而不是HashMap等传统数据结构,下图演示了HashMap的简要工作原理,相比起Android系统专门为移动操作系统编写的ArrayMap容器，在大多数情况下都显示效率低下,更占内存。通常的HashMap的实现方式更加消耗内存,因为它需要一个额外的实例对象来记录Mapping操作。另外,SparseArray更加高效在于他们避免了对key与value的autobox自动装箱，并且避免了装箱后的解箱。

2. 免在Android里面使用Enum

   Android官方培训课程提到过“Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.”

3. 减小Bitmap对象的内存占用

Bitmap是一个极容易消耗内存的大胖子, 减小创建出来的Bitmap的内存占用是很重要的, 通常来说有下面2个措施:

* inSampleSize：缩放比例，在把图片载入内存之前，我们需要先计算出一个合适的缩放比例，避免不必要的大图载入。

* decode format: 解码格式，选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8,存在很大差异。

4. 使用更小的图片

   在设计给到资源图片的时候，我们需要特别留意这张图片是否存在可以压缩的空间,是否可以使用一张更小的图片。尽量使用更小的图片不仅仅可以减少内存的使用,还可以避免出现大量的InflationException。假设有一张很大的图片被XML文件直接引用,很有可能在初始化视图的时候就会因为内存不足而发生InflationException,这个问题的根本原因其实是发生了OOM。

### 内存对象的重复利用

大多数对象的复用，最终实施的方案都是利用对象池技术,要么是在编写代码的时候显式的在程序里面去创建对象池，然后处理好复用的实现逻辑，\,要么就是利用系统框架既有的某些复用特性达到减少对象的重复创建,从而减少内存的分配与回收。   

![android_perf_2_object_pool]()


在Android上面最常用的一个缓存算法是LRU(Least Recently Use)，简要操作原理如下图所示:

![android_perf_2_lru_mode]()

1. 复用系统自带的资源

Android系统本身内置了很多的资源,例如字符串/颜色/图片/动画/样式以及简单布局等等,这些资源都可以在应用程序中直接引用。这样做不仅仅可以减少应用程序的自身负重,减小APK的大小,另外还可以一定程度上减少内存的开销,复用性更好。但是也有必要留意Android系统的版本差异性,对那些不同系统版本上表现存在很大差异,不符合需求的情况,还是需要应用程序自身内置进去。


2. 注意在ListView/GridView等出现大量重复子组件的视图里面对ConvertView的复用

![android_perf_oom_listview_recycle]()


3. Bitmap对象的复用

* 在ListView与GridView等显示大量图片的控件里面需要使用LRU的机制来缓存处理好的Bitmap。

 
4. 避免在onDraw方法里面执行对象的创建

   类似onDraw等频繁调用的方法,一定需要注意避免在这里做创建对象的操作,因为它会迅速增加内存的使用,而且很容易引起频繁的gc,甚至是内存抖动。

5. StringBuilder

  在有些时候，代码中会需要使用到大量的字符串拼接的操作，这种时候有必要考虑使用StringBuilder来替代频繁的“+”。


### 避免对象的内存泄露

内存对象的泄漏,会导致一些不再使用的对象无法及时释放,这样一方面占用了宝贵的内存空间,很容易导致后续需要分配内存的时候,空闲空间不足而出现OOM。显然,这还使得每级Generation的内存区域可用空间变小,gc就会更容易被触发,容易出现内存抖动,从而引起性能问题。

![android_perf_3_leak]()


#### 注意Activity的泄漏

   通常来说，Activity的泄漏是内存泄漏里面最严重的问题,它占用的内存多,影响面广,我们需要特别注意以下两种情况导致的Activity泄漏:

1. 内部类引用导致Activity的泄漏

最典型的场景是Handler导致的Activity泄漏，如果Handler中有延迟的任务或者是等待执行的任务队列过长,都有可能因为Handler继续执行而导致Activity发生泄漏。此时的引用关系链是Looper -> MessageQueue -> Message -> Handler -> Activity。为了解决这个问题,可以在UI退出之前,执行remove Handler消息队列中的消息与runnable对象。或者是使用Static + WeakReference的方式来达到断开Handler与Activity之间存在引用关系的目的。


2. Activity Context被传递到其他实例中,这可能导致自身被引用而发生泄漏

内部类引起的泄漏不仅仅会发生在Activity上,其他任何内部类出现的地方,都需要特别留意！我们可以考虑尽量使用static类型的内部类,同时使用WeakReference的机制来避免因为互相引用而出现的泄露。


#### 考虑使用Application Context而不是Activity Context

对于大部分非必须使用Activity Context的情况（Dialog的Context就必须是Activity Context）,我们都可以考虑使用ApplicationContext而不是Activity 的Context ,这样可以避免不经意的Activity泄露。


#### 注意临时Bitmap对象的及时回收

虽然在大多数情况下,我们会对Bitmap增加缓存机制,但是在某些时候,部分Bitmap是需要及时回收的。例如临时创建的某个相对比较大的bitmap对象,在经过变换得到新的bitmap对象之后,应该尽快回收原始的bitmap,这样能够更快释放原始bitmap所占用的空间。

需要特别留意的是Bitmap类里面提供的createBitmap()方法:

![android_perf_oom_create_bitmap]()

这个函数返回的bitmap有可能和source bitmap是同一个,在回收的时候,需要特别检查source bitmap与return bitmap的引用是否相同,只有在不等的情况下,才能够执行source bitmap的recycle方法。


#### 注意监听器的注销

在Android程序里面存在很多需要register与unregister的监听器,我们需要确保在合适的时候及时unregister那些监听器。自己手动add的listener,需要记得及时remove这个listener。


#### 注意Cursor对象是否及时关闭

在程序中我们经常会进行查询数据库的操作，但时常会存在不小心使用Cursor之后没有及时关闭的情况。这些Cursor的泄露，反复多次出现的话会对内存管理产生很大的负面影响，我们需要谨记对Cursor对象的及时关闭。


## 内存使用策略优化

1. 谨慎使用large heap

在一些特殊的情景下,你可以通过在manifest的application标签下添加largeHeap=true的属性来为应用声明一个更大的heap空间。然后,你可以通过getLargeMemoryClass()来获取到这个更大的heap size阈值。然而,声明得到更大Heap阈值的本意是为了一小部分会消耗大量RAM的应用(例如一个大图片的编辑应用)。不要轻易的因为你需要使用更多的内存而去请求一个大的Heap Size。只有当你清楚的知道哪里会使用大量的内存并且知道为什么这些内存必须被保留时才去使用large heap。因此请谨慎使用large heap属性。使用额外的内存空间会影响系统整体的用户体验,并且会使得每次gc的运行时间更长。在任务切换时,系统的性能会大打折扣。另外, large heap并不一定能够获取到更大的heap 。在某些有严格限制的机器上,large heap的大小和通常的heap size是一样的。因此即使你申请了large heap,你还是应该通过执行getMemoryClass()来检查实际获取到的heap大小。


2. onLowMemory()与onTrimMemory()

Android用户可以随意在不同的应用之间进行快速切换。为了让background的应用能够迅速的切换到forground,每一个background的应用都会占用一定的内存。Android系统会根据当前的系统的内存使用情况,决定回收部分background的应用内存。如果background的应用从暂停状态直接被恢复到forground,能够获得较快的恢复体验.如果background应用是从Kill的状态进行恢复,相比之下就显得稍微有点慢。


![android_perf_3_memory_bg_2_for]()

* onLowMemory()：Android系统提供了一些回调来通知当前应用的内存使用情况,通常来说,当所有的background应用都被kill掉的时候,forground应用会收到onLowMemory()的回调。在这种情况下,需要尽快释放当前应用的非必须的内存资源,从而确保系统能够继续稳定运行。

* onTrimMemory(int)：Android系统从4.0开始还提供了onTrimMemory的回调,当系统内存达到某些条件的时候,所有正在运行的应用都会收到这个回调,同时在这个回调里面会传递以下的参数,代表不同的内存使用情况,收到onTrimMemory回调的时候,需要根据传递的参数类型进行判断,合理的选择释放自身的一些内存占用,一方面可以提高系统的整体运行流畅度,另外也可以避免自己被系统判断为优先需要杀掉的应用。

3. Try catch某些大内存分配的操作

在某些情况下,我们需要事先评估那些可能发生OOM的代码,对于这些可能发生OOM的代码,加入catch机制,可以考虑在catch里面尝试一次降级的内存分配操作。例如decode bitmap 的时候，catch 到OOM,可以尝试把采样比例再增加一倍之后，再次尝试 decode。

4. 谨慎使用static对象

因为static的生命周期过长,和应用的进程保持一致,使用不当很可能导致对象泄漏,在Android中应该谨慎使用static对象。

![android_perf_3_leak_static]()

5. 特别留意单例对象中不合理的持有

虽然单例模式简单实用,提供了很多便利性,但是因为单例的生命周期和应用保持一致,使用不合理很容易出现持有对象的泄漏。


6. 珍惜Services资源

如果你的应用需要在后台使用service,除非它被触发并执行一个任务,否则其他时候Service都应该是停止状态。另外需要注意当这个service完成任务之后因为停止service失败而引起的内存泄漏。 当你启动一个Service,系统会倾向为了保留这个Service而一直保留Service所在的进程。这使得进程的运行代价很高,因为系统没有办法把Service所占用的RAM空间腾出来让给其他组件。这减少了系统能够存放到LRU缓存当中的进程数量,它会影响应用之间的切换效率,甚至会导致系统内存使用不稳定,从而无法继续保持住所有目前正在运行的service。 建议使用IntentService,它会在处理完交代给它的任务之后尽快结束自己。


7. 优化布局层次，减少内存消耗

越扁平化的视图布局，占用的内存就越少，效率越高。我们需要尽量保证布局足够扁平化，当使用系统提供的View无法实现足够扁平的时候考虑使用自定义View来达到目的。

8. 谨慎使用“抽象”编程

很多时候, 开发者会使用抽象类作为”好的编程实践”, 因为抽象能够提升代码的灵活性与可维护性。然而, 抽象会导致一个显著的额外内存开销: 它们需要同等量的代码用于可执行,那些代码会被mapping到内存中,因此如果你的抽象没有显著的提升效率,应该尽量避免它们。


9. 谨慎使用多进程

使用多进程可以把应用中的部分组件运行在单独的进程当中,这样可以扩大应用的内存占用范围,但是这个技术必须谨慎使用,绝大多数应用都不应该贸然使用多进程.一方面是因为使用多进程会使得代码逻辑更加复杂,另外如果使用不当,它可能反而会导致显著增加内存。当你的应用需要运行一个常驻后台的任务,而且这个任务并不轻量,可以考虑使用这个技术。

一个典型的例子是创建一个可以长时间后台播放的MusicPlayer。如果整个应用都运行在一个进程中,当后台播放的时候,前台的那些UI资源也没有办法得到释放。类似这样的应用可以切分成2个进程:一个用来操作UI,另外一个给后台的Service。


10. 使用ProGuard来剔除不需要的代码

ProGuard 能够通过移除不需要的代码,重命名类,域与方法等等对代码进行压缩,优化与混淆。使用ProGuard可以使得你的代码更加紧凑,这样能够减少mapping代码所需要的内存空间。

11. 谨慎使用第三方libraries

很多开源的library代码都不是为移动网络环境而编写的,如果运用在移动设备上,并不一定适合。即使是针对Android而设计的library,也需要特别谨慎,特别是在你不知道引入的library具体做了什么事情的时候。例如,其中一个library使用的是nano protobufs, 而另外一个使用的是micro protobufs。这样一来，在你的应用里面就有2种protobuf的实现方式。这样类似的冲突还可能发生在输出日志，加载图片，缓存等等模块里面。另外不要为了1个或者2个功能而导入整个library,如果没有一个合适的库与你的需求相吻合，你应该考虑自己去实现，而不是导入一个大而全的解决方案。

## 小结

设计风格很大程度上会影响到程序的内存与性能，相对来说，如果大量使用类似Material Design的风格，不仅安装包可以变小,还可以减少内存的占用,渲染性能与加载性能都会有一定的提升。
内存优化并不就是说程序占用的内存越少就越好,如果因为想要保持更低的内存占用,而频繁触发执行gc操作,在某种程度上反而会导致应用性能整体有所下降,这里需要综合考虑做一定的权衡。
Android的内存优化涉及的知识面还有很多:内存管理的细节,垃圾回收的工作原理,如何查找内存泄漏等等。OOM 是内存优化当中比较突出的一点,尽量减少OOM的概率对内存优化有着很大的意义。


[Android内存优化之OOM](http://hukai.me/android-performance-oom/)
