## Thread

### Define a class that implements Runnable

1. 实现一个 Runnable 接口

```
public class PhotoDecodeRunnable implements Runnable {
    ...
    @Override
    public void run() {
        /*
         * Code you want to run on the thread goes here
         */
        ...
    }
    ...
}
```

2. 重写 run 方法

对于那些耗时的操作就可以放在下面的 run 方法中处理了,如下所示:

```
class PhotoDecodeRunnable implements Runnable {
...
    /*
     * Defines the code to run for this task.
     */
    @Override
    public void run() {
        // Moves the current Thread into the background
        android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_BACKGROUND);
        ...
        /*
         * Stores the current Thread in the PhotoTask instance,
         * so that the instance
         * can interrupt the Thread.
         */
        photoTask.setImageDecodeThread(Thread.currentThread());
        ...
    }
...
}
```

有两点值得注意的是:

1. PhotoDecodeRunnable 线程优先级的设置, 将子线程的优先级设置为 THREAD_PRIORITY_BACKGROUND,这样可以减少 UI Thread 和子线程之间的系统资源争    夺.让系统尽可能的处理 UI Thread 中相关操作.

2. PhotoTask 存储了当前线程的引用,这样在必要的时候进行中断线程的操作.
