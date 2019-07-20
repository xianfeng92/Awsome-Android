## Andorid中的线程

Android 中线程可分为主线程和子线程, 主线程处理和界面相关的事情,子线程则往往用于执行耗时操作。由于 Andorid 特性, 如果在主线程中执行耗时操作那么就会导致程序无法及时响应，因此耗时操作必须放在子线程中执行。

1. Thread

2. AsyncTask

   封装了线程池和 Handler, 主要为了方便开发者在子线程中更新 UI

3. HandlerThread

   一种具有消息循环的线程, 在它的内部可以使用 Handler

4. IntentService

   采用 HandlerThread 来执行任务，当任务执行完后 IntentService 会自动退出。从任务执行角度来看，IntentService 的作用很像一个后台线程，
   但是 IntentService 也是一种服务, 它不容易被系统杀死从而可以尽量保证任务的执行。
   
   对于一个后台线程, 如果进程中没有活动的四大组件, 那么这个进程的优先级会非常低, 很容易被系统杀死。

在操作系统中，线程是操作系统最小的单元，同时线程又是一种受限的系统资源，即线程不可能无限制的产生，并且线程的创建和销毁都会有相应的开销，
当系统中存在大量的线程时，系统会通过时间片轮转的方式来调度线程，因此线程不可能做到并行，除非线程数量小于等于CPU的核心数，这显然不是高效的做法。
正确的做法是采用线程池，一个线程池可以缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。
Android 中的线程池来源于 Java,主要是通过 Executor 来派生特定类型的线程池，不同种类的线程池又具有各自的特点。

## 主线程和子线程

主线程是指进程所拥有的线程，在 JAVA 中默认情况下一个进程只有一个线程，这个线程就是主线程。主线程主要处理界面交互相关的逻辑，因为用户随时会和界面发生交互，因此主线程在任何时候都必须有较高的响应速度，否则会产生一种界面卡顿的感觉。为了保持较高的响应速度，这就要求主线程中不能执行耗时任务，这个时候子线程就派上用场了。

Android 沿用了 Java 的线程模式，其中的线程也分为主线程和子线程,  __主线程的作用是运行四大组件以及处理他们的用户交互,而子线程的作用则是耗时任务，比如网络请求，IO操作等__。

从Android3.0开始系统要求网络访问必须在子线程中，这是避免主线程由于耗时操作所阻塞出现 ANR 现象。

## Android中的线程形态

除了传统的 Thread 以外, 还包含 AsyncTask、HandlerThread 和 IntentService . 这三者的底层实现也是线程, 但是他们具有特殊的表现形式, 在使用上也各有优缺点。

### AsyncTask

AsyncTask 是一种轻量级的异步任务类, 可在线程池中执行后台任务并把执行的进度和最终结果传递给主线程用于UI的更新。从实现上说，AsyncTask 封装了 Thread 
和 Handler, 通过 AsyncTask 可更加方便的执行后台任务以及访问 UI. 但是 AsyncTask 并不适合进行特别耗时的后台任务。对于特别耗时的任务来说，建议使用线程池。

AsyncTask是一个抽象的泛型类，它提供了Params,Progress和Result这三个泛型参数，其中 Params 表示参数的类型，Progress 表示后台任务的执行进度，Result表示结果的类型。

如果 AsyncTask 确实不需要传递具体的参数可以用 Void 来代替, AsyncTask 的声明如下:

```
public abstract class AsyncTask<Params, Progress, Result> 
```

AsyncTask提供了4个核心方法, 它们的含义如下:

1. onPreExecute 
   
   在主线程执行，在异步任务执行之前被调用，一般可以用于做一些准备工作

2. doInBackground

   在线程池中执行，此方法用于执行异步任务，params 参数表示异步任务的输入参数。在此方法中可以通过 publishProgress 方法来更新任务的进度。
   publishProgress 会调用 onProgressUpdate 方法。另外, 此方法需要返回计算结果给 onPostExecute 方法。

3. onProgressUpdate 

   在主线程执行, 当后台任务的执行进度发生改变时此方法会被调用

4. onPostExecute

   在主线程中执行，在异步任务结束后，此方法被调用. 其中 result 参数是后台任务的返回值, 即 doInBackground 的返回值。

上面几个方法，onPreExecute 先执行，接着是 doInBackground，最后才是 onPostExecute。除了上述四个方法以外，AsyncTask 还提供了一个 onCancelled 方法。它同样在主线程中执行，当异步任务被取消时，onCancelled 方法就会被调用，这个时候 onPostExecute 就不会被调用。

下面提供一个典型的例子：

```
class DownloadTask extends AsyncTask<URL,Integer,Long>{

        @Override
        protected Long doInBackground(URL... urls) {
            int count = urls.length;
            long totalSize = 0;
            for (int i = 0; i < totalSize; i++) {
                totalSize += DownLoad.downloadFile(urls[i]);
                publicProgress((int)((i / (float)count)*100));
                if(isCancelled()){
                    break;
                }
            }
            return totalSize;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
           setProgressPercent(values[0]);
        }

        @Override
        protected void onPostExecute(Long result) {
            showDialog(result);
        }
    }

```

注意到 doInBackground 和 onProgressUpdate 方法的参数中均包含 ... 的字样，在 Java 中 ... 表示参数数量不定，它是一种数组型参数。

当要执行上述下载任务时, 我们可以这样:

			```
			new DownloadTask().execute(url1,url2,url3);
			```
在 DownloadTask 中，doInBackground用来执行具体的下载任务并通过 onProgressUpdate 来更新进度，同时还要判断下载任务是否被外界取消了。
当下载任务完成后，doInBackground 会返回结果，即下载的总字节数。需要注意的是， doInBackground 是在线程池中执行的，onProgressUpdate
用于更新界面中下载的进度，它运行在主线程中，当 publicProgress 被调用时，此方法会被调用。当下载任务完成后，onPostExecute 方法会被调用，
它也是运行在主线程中，这个时候我们就可以在界面上做一些提示，比如弹出一个对话框告知用户下载已经完成。

在 AsyncTask 的使用过程中也是有一些条件限制，主要有如下几点：

1. AsyncTask 的对象必须在主线程创建

2. execute 方法必须在UI线程调用

3. 一个 AsyncTask 对象只能执行一次，即只能调用一次 execute 方法，否则会报运行时异常

4. 在Android1.6之前，AsyncTask 是串行执行任务，之后采用线程池并行处理. 但是从 Android3.0 开始为了避免 AsyncTask 所带来的并发错误,              AsyncTask又采用 一个线程来串行任务. 尽管如此，在 Android3.0 以及以后的版本中，我们也可以通过 AsyncTask 的 executeOnExecutor 方
   法来并行的执行任务。

### AsyncTask 的工作原理

为了分析 AsyncTask 的工作原理，我们从 execute 开始分析：

```
   public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

   public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

sDefaultExecutor 是一个串行的线程池, 一个进程中所有的 AsyncTask 全部在这个串行的的线程池中排队执行.

线程池的执行过程:

```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
从上可以得出 AsyncTask 排队执行的过程:

1. 首先系统会把 AsyncTask 的 Params 参数封装为 FutureTask 对象, FutureTask 是一个并发类，在这里它充当了Runnable的作用

2. 接着这个 FutureTask 会交给 SerialExecutor 的 execute 方法去处理

3. SerialExecutor 的 execute 会将 FutureTask 对象插入到任务列表的 mTask 中

4. 如果此时没有正在活动的 AsyncTask 任务，那么就会调用 SerialExecutor#scheduleNext 来执行下一个 AsyncTask 任务

5. 当一个 AsyncTask 任务执行完成后, AsyncTask 会继续执行其他任务

AsyncTask 中有两个线程池（SerialExecutor 和 THREAD_POOL_EXECUTOR）和一个 Handler, 其中线程池 SerialExecutor 用于任务的排队，
而线程池 THREAD_POOL_EXECUTOR 用于真正的执行任务。IntentalHandler 用于将执行环境从线程池切换到主线程。

AsyncTask 的构造方法中有如下一段代码, 由于 FutureTask 的 run 方法会调用 mWorker#call, 因此 mWorker#call 最终会在线程池中执行。

```
mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                return postResult(doInBackground(mParams));
            }
        };
``

首先将 mTaskInvoked 标志为 true,然后执行 AsyncTask#doInBackground 方法并将其返回值通过 postResult 扔回到主线程:

```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
postResult 会通过 sHandler 发送给一个 MESSAGE_POST_RESULT:

```
 private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
InternalHandler 是一个静态的 Handler 对象, 为了能够将执行环境切换到主线程. 这就是要求 sHandler 这个对象必须在主线程创建。由于静态成员会在加载类的时候进行初始化, 因此这就变相要求 AsyncTask 的类必须在主线程加载, 否则无法工作。

sHandler 收到 MESSAGE_POST_RESULT 这个消息后会调用 AsyncTask 的 finish 方法。

```
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
这个 finish 的逻辑比较简单，如果 AsyncTask 被取消执行了，那么就会调用 onCancelled 方法，否则就会调用 onPostExecute，这样整个工作过程就完毕了。

通过分析 AsyncTask 的源码，可以进一步确认，从 Andorid 3.0 开始，默认情况下 AsyncTask 的确是串形执行的，在这里可以通过一个实验来证明这个判断。

点击按钮同时启动五个 AsyncTask 任务, 每一个会休眠 3s 来模拟耗时操作,然后把每个 AsyncTask 执行结束的时间打印出来：

```
case R.id.btnAsync:
                new MyAsyncTask("#1").execute("");
                new MyAsyncTask("#2").execute("");
                new MyAsyncTask("#3").execute("");
                new MyAsyncTask("#4").execute("");
                new MyAsyncTask("#5").execute("");
                break;
        }
    }

    class MyAsyncTask extends AsyncTask<String,Integer,String>{

        private String sName = "AsyncTask";

        public MyAsyncTask(String sName) {
            this.sName = sName;
        }

        @Override
        protected String doInBackground(String... strings) {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return sName;
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            Log.i(TAG,df.format(new Date())  + "");
        }
    }
}
```
其执行结果为每隔 3s 执行一个AsyncTask。

### HandlerThread

HandlerThread 继承了 Thread, 它是一种可以使用 Handler 的 Thread. 它的实现也很简单, 就是在 run 方法中通过 Looper.prepare 来创建消息队列，
并通过 Loop.loop 来开启消息循环, 这样在实际的使用中就允许在 HandlerThread 中创建 Handler了. 

HandlerThread#run:

```
 @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

普通 Thread 在 run 方法中执行一个耗时操作, 而 HandlerThread 在内部创建了消息队列,外界需要通过 Handler 的消息方式来通知 HandlerThread 执行一个具体的任务. HandlerThread 在 Android 中的一个具体的使用场景是 IntentService.

由于 HandlerThread#run 方法是一个无限循环, 因此当明确不需要再使用 HandlerThread 时，可以通过它的 quit或者quitSafely 方法来终止线程的执行。

### IntentService

IntentService 可用于执行后台耗时的任务. 由于 IntentService 继承自 Service, 这导致它的优先级比单纯的线程要高很多, 所以 IntentService 适合做一些高优先级的后台任务.

从 OnCreate 方法看出 IntentService 封装了 HandlerThread 和 Handler:

```
@Override
    public void onCreate() {
    
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();// 开启一个消息循环线程

        mServiceLooper = thread.getLooper();// 获取消息循环线程的 looper
        mServiceHandler = new ServiceHandler(mServiceLooper); // 使用消息循环线程的 looper 构建一个 handler
    }
```
由上可知, 通过 mServiceHandler 发送的消息会被存储到 HandlerThread 线程的消息队列中并通过其 looper 取出来处理.

从这个角度来看，IntentService 也可以用于执行后台任务，每次启动 IntentService，它的 onStartCommand 方法就会调用一次，IntentService 在 onStartCommand 中处理每个后台任务的 intent。

下面看一下 onStartCommand 如何处理外界的 intent 的，onStartCommand 方法调用 onStart，onStart 方法实现如下：

```
 @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

可以看出，IntentService 仅仅通过 mServiceHandler 发送了一条消息，这个消息会在 HandlerThread 中被处理。mServiceHandler 收到消息后，会将
Intent 对象传递给 onHandleIntent 来处理。

当 onHandleIntent 结束之后, IntentService 会通过 stopSelf(int startId) 方法来尝试停止服务。之所以采用stopSelf（int startId）而不是 stopSelf() 是因为 stopSelf() 会立刻停止服务, 而这个时候可能还有任务没有执行完成. stopSelf（int startId）会等待所有消息都处理完毕才去终止任务。

stopSelf（int startId）在尝试停止服务之前会判断最近启动服务的次数是否和这个 startId 相等, 如果相等就立刻停止服务，不相等则不停止服务。

ServiceHandler 的实现如下所示:

```
 private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

```

IntentService 的 onHandleIntent 方法是一个抽象方法，它需要我们在子类中实现，它的作用是从Intent参数中区分具体的任务并执行这些任务，如果目前只存在一个后台任务, 那么 onHandlerIntent 方法执行完之后，stopSelf(int id)就会直接停止服务; 如果目前存在多个后台任务, 那么当 onHandleIntent 方法执行完最后一个任务时 stopSelf(int id) 才会停止服务。由于每执行一个后台任务就必须启动一次 IntentService，而 IntentService 内部则通过消息的方式向 HandlerThread 请求执行任务，Handler中的 Looper 是顺序处理消息的.

这就意味着 IntentService 也是顺序执行的, 当有多个后台任务时同时存在的, 这些后台任务会按照外界发起的顺序排队执行。

下面通过一个案例来说明 IntentService 的工作方式:

```
public class LocalIntentService extends IntentService {

    private static final String TAG = "LocalIntentService";

    public LocalIntentService() {
        super(TAG);
    }

    @Override
    protected void onHandleIntent( Intent intent) {
        String action = intent.getStringExtra("task_action");
        Log.i(TAG,"receiver action:" + action);
        SystemClock.sleep(3000);
        if("com.lgl.test.ACTION".equals(action)){
            Log.i(TAG,"handler action:" + action);
        }
    }

    @Override
    public void onDestroy() {
        Log.i(TAG,"onDestroy");
        super.onDestroy();
    }
}
```

这里对 LocalIntentService 做一个简单的说明。在 onHandleIntent 方法中会从参数中解析出后台任务的标示，即
task_action 字段所代表的内容，然后根据不同的任务标识来执行具体的后台任务。下面的代码先后发起了三个后台任务的请求：

			```
			Intent Service = new Intent(this,LocalIntentService.class);
			service.putExtra("task_action","TASK1");
			startService(intent);
			Intent Service = new Intent(this,LocalIntentService.class);
			service.putExtra("task_action","TASK2");
			startService(intent);
			Intent Service = new Intent(this,LocalIntentService.class);
			service.putExtra("task_action","TASK3");
			startService(intent);
			```

它们的执行顺序就是它们发起请求的顺序, 即 TASK1，TASK2，TASK3。另外一点就是当 TASK3 执行完毕后，LocalIntentService 才会真正停止, 这也意味着服务正在停止。

### 线程池

线程池的优先有如下:

1. 重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销

2. 能有效控制线程池的最大并发数, 避免大量的线程之间因互相抢占系统资源而导致的阻塞现象

3. 能够对线程进行简单的管理, 并提供定时执行以及指定间隔循环等能力

Android 线程池可分为四类, 都是通过 Executor 提供的工厂方法得到的。

#### ThreadPoolExecutor

ThreadPoolExecutor 是线程池的真正实现, 它的构造方法提供了一系列的参数来配置:

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue
                              ThreadFactory threadFactory) {
```

* corePoolSize 

  默认情况下, 核心线程会在线程池中一直存活, 即使他们处于闲置状态.如果将 ThreadPoolExecutor 的 allowCoreThreadTimeOut 属性设置为true,那么闲置   的核心线程在等待新任务到来时就会有超时策略,当等待时间超过 keepAliveTime 所指的时长后，核心线程就会被终止。

* maximumPoolSize 

  线程池所容纳最大的线程数: 当活动线程数达到这个数值, 后续的新任务将会被阻塞

* keepAliveTime

  非核心线程闲置时的超时时长: 超过这个时长, 非核心线程就会被回收. 当 ThreadPoolExecutor 的 allowCoreThreadTimeOut 属性设置为true时,
  keepAliveTime同样会作用于核心线程

* unit

  用于指定 keepAliveTime 参数的时间单位

* workQueue
 
  线程池的任务队列: 通过线程池的 execute 方法提交的 Runnable 对象会存储在这个队列中

* threadFactory

  线程工厂: 为线程池提供创建新线程的功能. 

除了上面的一些主要参数外，ThreadPoolExecutor 还有一个不常见的参数: RejectedExecutionHandler handler. 当线程池无法执行新任务时, 这可能是由于任务队列已满或者是无法成功执行任务, 这个时候 ThreadPoolExecutor 会调用 handler#RejectedExecutionHandler 通知调用者.

默认情况下, RejectedExecutionHandler方法会直接抛出一个异常。

ThreadPoolExecutor 执行任务时大致遵守如下规则:

1. 如果线程池中的线程数量未达到核心线程的数量，那么直接启动一个核心线程来执行任务

2. 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行

3. 如果在步骤2中无法将任务插入到任务队列中(这往往是由于任务队列已满)，这个时候如果线程数量未达到线程池规定的最大值，那么会立刻启动一个非核心线程来执行任务

4. 如果步骤3中线程数量已经达到线程池规定的最大数量，那么就会拒绝执行此任务，ThreadPoolExecutor 会调用 RejectedExecutionHandler 的    rejectedExecution 方法来通知调用者

ThreadPoolExecutor的参数配置在 AsyncTask 中有明显的体现，下面是 AsyncTask 的线程池的配置情况:

```
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {

        private final AtomicInteger mCount = new AtomicInteger(1);

        @Override
        public Thread newThread(Runnable runnable) {
            return new Thread(runnable, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingDeque<Runnable>(128);
    public static final Executor THREAD_POOL_EXECUTOR =
            new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```


AsyncTask 对 THREAD_POOL_EXECUTOR 的配置如下:

* 核心线程数等于CPU核心数 + 1

* 线程池的最大线程数为CPU核心数的2倍+1

* 核心线程无超时机制，非核心线程的限制时超时时间为1s

* 任务队列的容量为128


### 线程池的分类

Android 中常见的四类具有不同功能特性的线程池，它们都直接或者间接的通过配置 ThreadPoolExecutor 来实现自己的功能，这四类线程池分别是：

1. FixedThreadPool 

通过 Executors 的 newFixedThreadPool 方法来创建,它是一种线程数量固定的线程池. 当线程池处于空闲的状态时, 它们并不会被回收。当所有的线程池都处于活动状态时，新任务都处于等待状态, 直到有线程空闲出来. 由于FixedThreadPool只有核心线程且不会被回收, 这意味着它能够更快加速的响应外界的请求.

newFixedThreadPool 方法的实现如下:

						```
			public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
				return new ThreadPoolExecutor(nThreads, nThreads,
					0L, TimeUnit.MILLISECONDS,
				        new LinkedBlockingQueue<Runnable>(),
					threadFactory);
						    }
						```

可以发现 FixedThreadPool 中只有核心线程并且这些核心线程没有超时机制, 另外任务队列也是没有大小限制的。

2. CachedThreadPool

通过 Executors 的 newCachedThreadPool 来创建, 这是一种线程数量不定的线程池, 它只有非核心线程并且其最大线程数为 Integer.MAX_VALUE.
实际上就相当于最大线程数可以任意大, 当线程池中的线程都处于活动状态的时候，线程池会创建新的线程来处理新任务, 否则就会利用空闲的线程来处理新任务.

线程池的空闲线程都有超时机制, CachedThreadPool 的任务队列其实相当于一个空集合，这就导致任何任务都会立即被执行，因为在这种场景下，
SynchronousQueue 是一个非常特殊的队列，在很多情况下可以把他简单理解为一个无法存储的元素队列. 这类线程池比较适合执行大量的耗时较少的任务, 当整个线程池属于闲置状态的时候, 线程池中的线程都会超时而被停止，这个时候 CachedThreadPool 之中实际上是没有任何线程, 这个时候它几乎是不占用任何资源的。


						```
						 public static ExecutorService newCachedThreadPool() {
							return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
										      60L, TimeUnit.SECONDS,
										      new SynchronousQueue<Runnable>());
						    }
						```



3. ScheduleThreadPool
   
通过 ScheduleThreadPool 方法来创建，它的核心线程数量是固定的，而非核心线程是没有限制，并且当非核心线程闲置时会立即回收, ScheduleThreadPool 主要用于定时任务和具有固定周期的重复任务。



						```
						    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
							return new DelegatedScheduledExecutorService
							    (new ScheduledThreadPoolExecutor(1));
						    }
						```


4. SingleThreadExecutor

 newSingleTheardExecutor 内部只有一个核心线程, 确保所有的任务都在同一个线程按顺序执行. SingleTheardExecutor 的意义是统一所有的外界任务到同一个线程中, 这使得这些任务之间不需要处理线程同步的问题。

						```
		public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
				return new DelegatedScheduledExecutorService
				(new ScheduledThreadPoolExecutor(1, threadFactory));
						    }
						```
 使用示例如下:



                                                 ```
							Runnable common = new Runnable() {
							    @Override
							    public void run() {
								SystemClock.sleep(2000);
							    }
							};

							ExecutorService fix = Executors.newFixedThreadPool(4);
							fix.execute(common);

							ExecutorService cache = Executors.newCachedThreadPool();
							fix.execute(common);

							ScheduledExecutorService schedu = Executors.newScheduledThreadPool(4);
							schedu.schedule(common,2000,TimeUnit.MICROSECONDS);
							schedu.scheduleAtFixedRate(common,10,1000,TimeUnit.MICROSECONDS);

							ExecutorService single = Executors.newSingleThreadExecutor();
							single.execute(common);
					```
