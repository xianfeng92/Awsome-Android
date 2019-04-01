# java多线程开启的三种方式

先直接上代码：

```
 public void test(){

        //开启线程方式1
        myThread.start();

        //开启线程方式2
        new Thread(new MyRunnable()).start();

        //开启线程方式3
        FutureTask futureTask = new FutureTask(myCallbale);
        new Thread(futureTask).start();
    }

    MyCallbale myCallbale = new MyCallbale();

    class MyCallbale implements Callable<Integer>{

        @Override
        public Integer call() throws Exception {
            int result = 0;
            for (int i = 0; i < 10; i++){
                result = result + i;
                System.out.println("run:"+i);
            }
            return result;
        }
    }

    class MyRunnable implements Runnable{

        @Override
        public void run() {
            for (int i = 0; i < 10; i++){
                System.out.println("run:"+i);
            }
        }
    }

    MyThread myThread = new MyThread();

    class MyThread extends Thread{
        @Override
        public void run() {
            for (int i = 0; i < 10; i++){
                System.out.println("run:"+i);
            }
        }

    }
    ```

1、继承Thread类，新建一个当前类对象，并且运行其start()方法

   先看看调用Threa类的start方法到底做了什么？

```
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        // Android-changed: throw if 'started' is true
         // 对线程状态和是否启动的判断
        // 状态校验  0：NEW 新建状态
        if (threadStatus != 0 || started)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);//添加进线程组

        started = false;
        try {
            nativeCreate(this, stackSize, daemon);//调用native方法执行线程run方法
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);//启动失败，从线程组中移除当前前程
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

在start（）方法中，首先进行的线程状态（threadStatus）的判断，如果是一个JVM新启动的线程，那么threadStatus 的状态是为0的，如果线程不为0 将报出异常， 然后将线程添加到group中， group.add(this)方法中执行的结果是，通知group， 这个线程要执行了，所以可以添加进group中，__调用native方法执行线程run方法__。

对于第一种线程启动方式，由于多态特性，navive方法执行的run方法其实也就是MyThread类的run方法。

2、实现Runnable接口，然后新建当前类对象，接着新建Thread对象时把当前类对象传进去，最后运行Thread对象的start()方法
 
   Thread类的start方法已经分析，再看看其run方法吧。

```
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }

```

Thread的run方法其实就是调用target的run方法，target是谁呢？

```
    /**
     * Initializes a Thread.
     *
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     */
    private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
        Thread parent = currentThread();
        if (g == null) {
            g = parent.getThreadGroup();
        }

        g.addUnstarted();
        this.group = g;

        this.target = target;//Thread的target赋值，其实就是我们传进去的runnable对象
        this.priority = parent.getPriority();
        this.daemon = parent.isDaemon();
        setName(name);

        init2(parent);

        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;
        tid = nextThreadID();
    }
```

我们发现这个target其实就是我们在初始化Thread类时传进去的runnable对象，即最终调用的时runnable的run方法。

3、实现Callable接口，新建当前类对象，在新建FutureTask类对象时传入当前类对象，接着新建Thread类对象时传入FutureTask类对象，最后运行Thread对象的start()方法

   当我们初始化Thread时，传进去的是一个FutureTask时，同理也会调用到FutureTask的run方法。

   ```
    public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
最后就是执行了当前类的call()方法，线程执行结束后，将其结果返回。


   


























