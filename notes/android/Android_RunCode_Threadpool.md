# Run code on a thread pool thread

## Run a task on a thread in the thread pool

想要在线程池中执行一个任务时,只需要将该任务(Runnable) 传到 ThreadPoolExecutor.execute() 方法中. ThreadPoolExecutor 会将该任务(Runnable)放入线程池
的一个工作队列(work queue)中,当线程池中有空闲(Idle)线程时,ThreadPoolExecutor 会将等待最久的那个任务(Runnable)取出,放入一个线程中执行.

```
public class PhotoManager {
    public void handleState(PhotoTask photoTask, int state) {
        switch (state) {
            // The task finished downloading the image
            case DOWNLOAD_COMPLETE:
            // Decodes the image
                decodeThreadPool.execute(
                        photoTask.getPhotoDecodeRunnable());
            ...
        }
        ...
    }
    ...
}
```


## Interrupt running code

想要停止一个任务(Runnable),就需要中断该任务所在的线程。一般,我们回通过持有线程的引用来方便后面的中断操作,例如:

```
class PhotoDecodeRunnable implements Runnable {
    // Defines the code to run for this task
    public void run() {
        /*
         * Stores the current Thread in the
         * object that contains PhotoDecodeRunnable
         */
        photoTask.setImageDecodeThread(Thread.currentThread());
        ...
    }
    ...
}
```

如果想要中断线程,通过调用 thread.interrupt（）.需要注意的是,线程对象由系统控制,系统可以在应用程序进程之外对其进行修改,所以需要在中断线程之前锁定线程上的访问,
方法是将访问放在同步块中。例如:

```
public class PhotoManager {
    public static void cancelAll() {
        /*
         * Creates an array of Runnables that's the same size as the
         * thread pool work queue
         */
        Runnable[] runnableArray = new Runnable[decodeWorkQueue.size()];
        // Populates the array with the Runnables in the queue
        mDecodeWorkQueue.toArray(runnableArray);
        // Stores the array length in order to iterate over the array
        int len = runnableArray.length;
        /*
         * Iterates over the array of Runnables and interrupts each one's Thread.
         */
        synchronized (sInstance) {
            // Iterates over the array of tasks
            for (int runnableIndex = 0; runnableIndex < len; runnableIndex++) {
                // Gets the current thread
                Runnable runnable = runnableArray[runnableIndex];
                Thread thread = null;
                if (runnable instanceof PhotoDecodeRunnable) {
                    thread = ((PhotoDecodeRunnable)runnable).mThread;
                }
                // if the Thread exists, post an interrupt to it
                if (null != thread) {
                    thread.interrupt();
                }
            }
        }
    }
    ...
}
```

在大多数情况下,thread.interrupt（）会立即停止线程。但是它只停止正在等待的线程,不会中断CPU或网络密集型任务。所以我们可以在进行 CPU-intensive 任务之前, check 一下当前 Thread 是否被中断(Thread.interrupted()).


```
/*
 * Before continuing, checks to see that the Thread hasn't
 * been interrupted
 */
if (Thread.interrupted()) {
    return;
}
...
// Decodes a byte array into a Bitmap (CPU-intensive)
BitmapFactory.decodeByteArray(
        imageBuffer, 0, imageBuffer.length, bitmapOptions);
...
```


[Run code on a thread pool thread](https://developer.android.google.cn/training/multiple-threads/run-code.html)