# Create a manager for multiple threads

如果我们执行的是“一次性”的任务, 可以通过实现一个 Runnable 接口,在其 run 方法中定义任务,然后交给Thread去 start; 如果我们执行的是在不同的数据集( different sets of data)上需要重复执行的(repeat)任务,但一次只需要执行一个任务,此时可以使用 IntentService; 如果我们想要在资源可用时自动执行任务，或允许多个任务同时执行,此时可以使用 ThreadPoolExecutor 实例,想要执行一个任务,只需将其添加到队列(queue)中,ThreadPoolExecutor 会有空闲的线程时,从队列中取出任务并执行。

线程池(ThreadPool)可以并行的执行任务,因此应该确保代码是线程安全的。一般需要将可由多个线程访问的变量放入同步块中。此方法会阻止一个线程在另一个线程向其写入变量时读取该变量。

## Define the thread pool class

实例化 ThreadPoolExecutor:

#### Use static variables for thread pools

一般建议使用单例模式来实例化一个线程池,以便为有限的CPU或网络资源提供单个控制点。如果我们有不同的可运行(Runnable)类型, 想要为每个 Runnable 类型都提供一个线程池,这里的每个线程池都可以用一个单例模式来实例化。例如,可以将此添加为全局字段声明的一部分:


```
public class PhotoManager {
    ...
    static  {
        ...
        // Creates a single static instance of PhotoManager
        sInstance = new PhotoManager();
    }
    ...
```


#### Use a private constructor

将构造函数设置为 private,这样可以确保其不会被外部类所实例化.

```
public class PhotoManager {
    ...
    /**
     * Constructs the work queues and thread pools used to download
     * and decode images. Because the constructor is marked private,
     * it's unavailable to other classes, even in the same package.
     */
    private PhotoManager() {
    ...
    }
```


#### Start your tasks by calling methods in the thread pool class.

在线程池类中定义将任务添加到线程池队列的方法,例如:

```
public class PhotoManager {
    ...
    // Called by the PhotoView to get a photo
    static public PhotoTask startDownload(
        PhotoView imageView,
        DownloadTask downloadTask,
        boolean cacheFlag) {
        ...
        // Adds a download task to the thread pool for execution
        sInstance.
                downloadThreadPool.
                execute(downloadTask.getHTTPDownloadRunnable());
        ...
    }
```

#### Instantiate a Handler in the constructor and attach it to your app's UI thread.

通过 Handler 将线程池中的任务执行结果回传到 UI 线程, 可用于对UI进行更新.

```
    private PhotoManager() {
    ...
        // Defines a Handler object that's attached to the UI thread
        handler = new Handler(Looper.getMainLooper()) {
            /*
             * handleMessage() defines the operations to perform when
             * the Handler receives a new Message to process.
             */
            @Override
            public void handleMessage(Message inputMessage) {
                ...
            }
        ...
        }
    }
```


### Determine the thread pool parameters

#### Initial pool size and maximum pool size

线程池初始化时的线程数(Initial pool size)和可分配的最大线程数(maximum pool size)。线程池中可以拥有的线程数主要取决于设备可用的cpu核心数。

```
public class PhotoManager {
...
    /*
     * Gets the number of available cores
     * (not always the same as the maximum number of cores)
     */
    private static int NUMBER_OF_CORES =
            Runtime.getRuntime().availableProcessors();
}
```

NUMBER_OF_CORES 可能不反映设备中物理 cpu 核心的实际数量.因为某些设备的CPU根据系统负载停用一个或多个核心。对于这些设备, availableProcessors()
返回活动着的核心数量,该数量可能小于核心的总数。


#### Keep alive time and time unit

线程在关闭前保持空闲状态的持续时间,时间单位由 TimeUnit 来定义。

#### A queue of tasks


线程池执行器(ThreadPoolExecutor)会从 Queue 中取出 Runnable 对象, 并将其关联到一个线程上并执行.在创建线程池时,可以使用实现 BlockingQueue 接口的任何队列类来提供此队列对象。

```
public class PhotoManager {
    ...
    private PhotoManager() {
        ...
        // A queue of Runnables
        private final BlockingQueue<Runnable> decodeWorkQueue;
        ...
        // Instantiates the queue of Runnables as a LinkedBlockingQueue
        decodeWorkQueue = new LinkedBlockingQueue<Runnable>();
        ...
    }
    ...
}
```


BlockingQueue 主要作用为存储 Runnable 对象,这样当线程池有空闲的线程时,ThreadPoolExecutor就可以从 BlockingQueue 取出 Runnable ,并将其放到一个线程上执行. 具体使用哪种
BlockingQueue 取决于具体的使用场景和需求.


### Create a thread pool

通过调用 ThreadPoolExecutor 的构造函数来实例化线程池管理器, ThreadPoolExecutor 将创建和管理一组受约束的线程。在上面的代码中,由于 Initial pool size 和
Max pool size 都为 NUMBER_OF_CORES, 所以 ThreadPoolExecutor 在实例化时已经将所有的线程对象都创建出来了。

```
    private PhotoManager() {
        ...
        // Sets the amount of time an idle thread waits before terminating
        private static final int KEEP_ALIVE_TIME = 1;
        // Sets the Time Unit to seconds
        private static final TimeUnit KEEP_ALIVE_TIME_UNIT = TimeUnit.SECONDS;
        // Creates a thread pool manager
        decodeThreadPool = new ThreadPoolExecutor(
                NUMBER_OF_CORES,       // Initial pool size
                NUMBER_OF_CORES,       // Max pool size
                KEEP_ALIVE_TIME,
                KEEP_ALIVE_TIME_UNIT,
                decodeWorkQueue);
    }
```


如果我们并不需要并发处理任务, 此时可以使用 Executors 工厂方法去创建一个 single thread executor,该 executor 会将所有的任务都放在一个线程中处理.


[Create a manager for multiple threads](https://developer.android.google.cn/training/multiple-threads/create-threadpool.html)