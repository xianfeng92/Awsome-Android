
## 吃个栗子

拿下面这个栗子说事：

```
        Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("Android");
                emitter.onNext("ios");
                emitter.onNext("Other");
                emitter.onComplete();
            }
        })
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        
                    }

                    @Override
                    public void onNext(String s) {
                        Log.d(TAG, "onNext: "+s);
                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });

```

## subscribeOn

 subscribeOn()源码如下：

```
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
```

ObservableSubscribeOn 类是一个装饰者，对下游的 Observer 进行装饰，很明显这里主要增加了一个 Scheduler 来做线程切换。
ObservableSubscribeOn 已经不再是从前的那个__下游的Observer__，Scheduler 的能力让它可以牛逼轰轰的了。


```
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    // 保存线程调度器,该栗子中即Schedulers.io()，Schedulers.io()生成的一个线程调度对象,此对象是维护这一个线程池,让操作在io线程池中执行
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
   // 保存 ObservableSource，该栗子中即 observableCreate 类
        super(source);
        this.scheduler = scheduler;
    }
    // 订阅时调用
    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        // 对 Observer 进行包装
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
         // 调用 下游（终点）Observer.onSubscribe()方法,所以onSubscribe()方法执行在订阅处所在的线程
        observer.onSubscribe(parent);
        // setDisposable()是为了将子线程的操作加入 Disposable 管理中
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            // 此时已经运行 io 线程中
            source.subscribe(parent);
        }
    }
```

上面的代码中，scheduler.scheduleDirect 将 SubscribeTask 加入到线程池中（此处为IO线程）执行。所以subscribeOn对线程切换是在
Observer 对 ObservableSubscribeOn 类进行 subscribe 时发生的。即，当一个 Observer 对 ObservableSubscribeOn 进行订阅时，后面
所有的操作都会在我们所指定线程中（此处为io）执行。

再来看看 SubscribeOnObserver 如何包装 observer:

```
    static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {
        // 真正的下游（终点）观察者
        final Observer<? super T> downstream;
        // 用于保存上游的Disposable，以便在自身 dispose 时，连同上游一起 dispose
        final AtomicReference<Disposable> upstream;

        SubscribeOnObserver(Observer<? super T> downstream) {
            this.downstream = downstream;
            this.upstream = new AtomicReference<Disposable>();
        }
        // onSubscribe()方法由上游调用，传入 Disposable,在本类中赋值给 this.upstream，加入管理。
        @Override
        public void onSubscribe(Disposable d) {
            DisposableHelper.setOnce(this.upstream, d);
        }
        // 直接调用下游观察者的对应方法
        @Override
        public void onNext(T t) {
            downstream.onNext(t);
        }

        @Override
        public void onError(Throwable t) {
            downstream.onError(t);
        }

        @Override
        public void onComplete() {
            downstream.onComplete();
        }
        // 取消订阅时，连同上游 Disposable 一起取消
        @Override
        public void dispose() {
            DisposableHelper.dispose(upstream);
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
        // 这个方法在subscribeActual()中被手动调用，为了将Schedulers返回的Worker加入管理
        void setDisposable(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }
    }
```
SubscribeOnObserver 对象会持有一个observer对象，同时也会维护一个 Disposable 对象，用于保存上游的Disposable，
以便在自身 dispose 时，连同上游一起 dispose。这里可以看到 SubscribeOnObserver 其实是 observer 对象的一个装饰者。


我们总结一下subscribeOn(Schedulers.xxx())的过程：

* 返回一个 ObservableSubscribeOn 包装类对象

* 上一步返回的对象被订阅时，回调该类中的 subscribeActual() 方法，在其中会立刻将线程切换到对应的 Schedulers.xxx() 线程

* 在切换后的线程中，执行 source.subscribe(parent)，对上游(终点)Observable订阅

* 上游(终点) Observable 开始发送数据，上游发送数据仅仅是调用下游观察者对应的 onXXX() 方法而已，所以此时操作是在切换后的线程中进行


为什么 subscribeOn(Schedulers.xxx())切换线程N次，总是以第一次为准，或者说离源 Observable 最近的那次为准？

* 订阅流程从下游往上游传递

* 在subscribeActual()里开启了 Scheduler 的工作，source.subscribe(parent)。从这一句开始切换了线程，所以在这之上的代码都是在切换后的线程里的了

* 但如果连续切换，最上面的切换最晚执行,此时线程变成了最上面的 subscribeOn(xxxx) 指定的线程

* 而数据 push 时，是从上游到下游的，所以会在离源头最近的那次 subscribeOn(xxxx) 的线程里 push 数据（onXXX()）给下游


## 线程调度 observeOn

增加一个 observeOn (AndroidSchedulers.mainThread())，就完成了观察者线程的切换。

```
                .subscribeOn(Schedulers.newThread())
                .map(new Function<String, String>() {
                    @Override
                    public String apply(String s) {
                        return s+s;
                    }
                })
                .observeOn(Schedulers.io())
```


```
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {\
   //本例是 AndroidSchedulers.mainThread()
    final Scheduler scheduler;
    //默认false
    final boolean delayError;
    //默认128
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            // 创建出一个主线程的 Worker
            Scheduler.Worker w = scheduler.createWorker();
           // 订阅上游数据源，
            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
```

subscribeActual 主要做了两件事：

* 创建一个AndroidSchedulers.mainThread()对应的 Worker

* 用 ObserveOnObserver 订阅上游数据源。这样当数据从上游 push 下来，会由 ObserveOnObserver 对应的 onXXX() 处理,
  而 ObserveOnObserver 中会保存下游的Observer，所以它可以对事件进行一些处理后（此处为线程切换）交给Observer。

```
 static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
        // 下游的观察者
        final Observer<? super T> downstream;
        // 对应Scheduler里的Worker
        final Scheduler.Worker worker;
        final boolean delayError;
        final int bufferSize;
        // 上游被观察者 push 过来的数据都存在这里
        SimpleQueue<T> queue;

        Disposable upstream;
        // 如果onError了，保存对应的异常
        Throwable error;
        //是否完成
        volatile boolean done;
        //是否取消
        volatile boolean disposed;
        // 代表同步发送 异步发送 
        int sourceMode;

        boolean outputFused;

        ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.downstream = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.validate(this.upstream, d)) {
                this.upstream = d;
               //创建一个queue 用于保存上游 onNext() push的数据
                queue = new SpscLinkedArrayQueue<T>(bufferSize);
               //回调下游观察者onSubscribe方法
                downstream.onSubscribe(this);
            }
        }

        @Override
        public void onNext(T t) {
            // 执行过error / complete 会是true
            if (done) {
                return;
            }
            // 如果数据源类型不是异步的， 默认不是
            if (sourceMode != QueueDisposable.ASYNC) {
           // 将上游push过来的数据 加入 queue里
                queue.offer(t);
            }
          // 开始进入对应 Workder 线程，在线程里将 queue里的 t 取出发送给下游Observer
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
                return;
            }
            error = t;
            done = true;
            //开始调度
            schedule();
        }

        @Override
        public void onComplete() {
            if (done) {
                return;
            }
            done = true;
            //开始调度
            schedule();
        }

        @Override
        public void dispose() {
            if (!disposed) {
                disposed = true;
                upstream.dispose();
                worker.dispose();
                if (getAndIncrement() == 0) {
                    queue.clear();
                }
            }
        }

        @Override
        public boolean isDisposed() {
            return disposed;
        }

        void schedule() {
            if (getAndIncrement() == 0) {
                //该方法需要传入一个线程， 注意看本类实现了Runnable的接口，所以查看对应的run()方法
                worker.schedule(this);
            }
        }

         //从这里开始，这个方法已经是在Workder对应的线程里执行的了
        @Override
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                // 取出 queue 里的数据发送
                drainNormal();
            }
        }
    }

```

上面代码主要功能如下：

1. 将上游push过来的数据加入 ObserveOnObserver 对象的一个队列中（SpscLinkedArrayQueue）

2. ObserveOnObserver 的Scheduler.Worker worker 变量保存着我们所指定的线程，此处为 AndroidSchedulers.mainThread()。

3. 然后 worker 将 ObserveOnObserver 对象放入指定的线程中（此处为 AndroidSchedulers.mainThread()），去取出其队列中
   存储的来自上游的事件。


还需要关注几点:

* ObserveOnObserver实现了 Observer 和 Runnable 接口,所以可以将其传入 worker 中，执行其 run 方法。

* 在onNext()里，先不切换线程，将数据加入队列 queue 。然后开始切换线程，在另一线程中，从queue里取出数据，push 给下游 Observer。

* 所以 observeOn() 影响的是其下游的代码，且多次调用仍然生效。因为其切换线程代码是在Observer里onXXX()做的。在上游 push 数据过程中，总要经过 onXXX()。

关于多次调用生效问题。对比 subscribeOn() 切换线程是在 subscribeActual()里做的，只是主动切换了上游的订阅线程，从而影响其发射数据时所在的线程。
而直到真正发射数据之前，任何改变线程的行为，都会生效（影响发射数据的线程）。所以 subscribeOn() 只生效一次。observeOn()是一个主动的行为，并且切换线程后会立刻发送数据，所以会生效多次。

## 小结

### 线程调度subscribeOn()：

* 先切换线程，在切换后的线程中对上游 Observable 进行订阅，这样上游发送数据时就是处于被切换后的线程里了。也因此多次切换线程，最后一次切换（离源数据最近）的生效。

* 内部订阅者（SubscribeOnObserver）接收到数据后，直接发送给下游 Observer,引入内部订阅者是为了控制线程（dispose）


### 线程调度 observeOn()：

* 使用装饰的 ObserveOnObserver 对上游 Observable 进行订阅

* 在 ObserveOnObserver 中onXXX()方法里，将待发送数据存入队列，同时请求切换线程处理，并将数据从队列中取出进而push给下游

* 多次使用 observeOn 切换线程，都会影响下游所执行的线程


