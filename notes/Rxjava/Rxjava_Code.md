
# 一个简单的例子

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

## 从create开始

```
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

```

返回值是 Observable,参数是 ObservableOnSubscribe ,定义如下：

```
public interface ObservableOnSubscribe<T> {
    void subscribe(ObservableEmitter<T> e) throws Exception;
}
```

ObservableOnSubscribe 是一个接口，里面就一个方法,也是我们实现的那个方法：该方法的参数是 ObservableEmitter，它是关联起 Disposable 概念的一层：

```
public interface ObservableEmitter<T> extends Emitter<T> {
    void setDisposable(Disposable d);
    void setCancellable(Cancellable c);
    boolean isDisposed();
    ObservableEmitter<T> serialize();
}
```

ObservableEmitter 也是一个接口。里面方法很多，它继承了 Emitter<T> 接口。

```
public interface Emitter<T> {
    void onNext(T value);
    void onError(Throwable error);
    void onComplete();
}
```

Emitter<T>定义了我们在 ObservableOnSubscribe 中实现 subscribe() 方法里最常用的三个方法。


ObservableCreate 算是一种适配器的体现，create()需要返回的是 Observable,而我现在有的是（即 方法传入的参数）ObservableOnSubscribe 对象，ObservableCreate 
将 ObservableOnSubscribe 适配成 Observable。 其中 subscribeActual()方法表示的是被订阅时真正被执行的方法。

```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```

OK,至此，创建流程结束，我们得到了 Observable<T> 对象，其实就是 ObservableCreate<T>。

## 到 subscribe 结束

   上面在代码在 create，返回 ObservableCreate 对象，然后调用该对象的 subscribe 完成订阅，我们知道 ObservableCreate 继承 Observable，
当调用 subscribe 方法时，会调 Observable 的 subscribe 方法：

```
    public final void subscribe(Observer<? super T> observer) {
            ...
             // 真正的订阅处
            subscribeActual(observer);
            ...
    }

```

上面代码的 subscribeActual 调用的是 ObservableCreate 中的方法。

```
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        //1 创建 CreateEmitter，也是一个适配器，可以将 Observer -> Disposable，CreateEmitter 中主要持有 observer 对象的引用，并且维护了 dispose 变量。
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        //2 onSubscribe（）参数是 Disposable。还有一点要注意的是 onSubscribe() 是在我们执行 subscribe() 这句代码的那个线程回调的，并不受线程调度影响。
        // 给 observer 的一个回调，告诉它是否 dispose
        observer.onSubscribe(parent);

        try {
            //3 将 ObservableOnSubscribe（源头）与 CreateEmitter（Observer，终点）联系起来，即完成订阅，此时 ObservableOnSubscribe 会向 observer 传送事件
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

```

source 即 ObservableOnSubscribe 对象，在本文中是：

```
new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("Android");
                emitter.onNext("ios");
                emitter.onNext("Other");
                emitter.onComplete();
            }
        }
```

ObservableOnSubscribe#subscribe 中会调用 parent.onNext() 和 parent.onComplete()，parent 是 CreateEmitter 对象，如下：

```

        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
           //如果没有被 dispose，会调用 Observer 的 onNext()方法
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }
```

## 总结

1. Observable 和 Observer 的关系没有被 dispose，才会回调 Observer 的 onXXXX()方法

2. Observer 的 onComplete() 和 onError() 互斥只能执行一次，因为 CreateEmitter 在回调他们两中任意一个后，都会自动dispose()

3. Observable 和 Observer关联时（订阅时），Observable 才会开始发送数据

4. ObservableCreate 将 ObservableOnSubscribe(真正的源)->Observable

5. ObservableOnSubscribe(真正的源)需要的是发射器 ObservableEmitter

6. CreateEmitter 将 Observer->ObservableEmitter,同时它也是 Disposable

7. source.subscribe(parent)
   这句代码执行时，才开始从发送 ObservableOnSubscribe 中利用 ObservableEmitter 发送数据给 Observer。即数据是从源头 push 给终点。


## map操作符


```
        Observable.create(new ObservableOnSubscribe<String>() { // return ObservableCreate

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("Android");
                emitter.onNext("ios");
                emitter.onNext("Other");
                emitter.onComplete();
            }
        })
                .map(new Function<String, String>() { // return ObservableMap, 并且ObservableMap持有对 ObservableSubscribeOn 的引用
                    @Override
                    public String apply(String s) {
                        return s+s;
                    }
                })
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

我们看一下map函数的源码：

```
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```

```
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        // super()将上游的Observable保存起来 ，用于subscribeActual()中用。
        super(source);
        // 将function变换函数类保存起来
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

```

ObservableMap 继承自 AbstractObservableWithUpstream,该类继承自 Observable，很简单，就是将上游的 ObservableSource 保存起来，做一次 wrapper，
所以它也算是装饰者模式的体现，如下:

```
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    // 将上游的 ObservableSource 保存起来
    protected final ObservableSource<T> source;

    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}
```

关于 ObservableSource，代表了一个标准的无背压的源数据接口，可以被 Observer 消费（订阅），如下：

```
public interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}
```

所有的 Observable 都已经实现了它,所以我们可以认为 Observable 和 ObservableSource 是相等的：

```
public abstract class Observable<T> implements ObservableSource<T> {
```

所以我们得到的 ObservableMap 对象也很简单，就是将上游的 Observable 和变换函数类Function保存起来。 
Function的定义超级简单，就是一个接口，给我一个T，还你一个R。

```
public interface Function<T, R> {
    R apply(T t) throws Exception;
}
```

subscribeActual()是订阅真正发生的地方，就是用 MapObserver 订阅上游 Observable。

```
    @Override
    public void subscribeActual(Observer<? super U> t) {
    //用 MapObserver 订阅上游 Observable。
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

MapObserver 也是装饰者模式，对终点（下游）Observer修饰。

```
    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            // super()将actual保存起来
            super(actual);
           // 保存Function变量
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            //done在onError 和 onComplete以后才会是true，默认这里是false，所以跳过
            if (done) {
                return;
            }
            //默认sourceMode是0，所以跳过
            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }

            U v;

            try {
                //这一步执行变换,将上游传过来的 T，利用 Function 转换成下游需要的 V
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
           //变换后传递给下游Observer
            downstream.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public U poll() throws Exception {
            T t = qd.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
```

订阅的过程，是从下游到上游依次订阅的:

* 即终点 Observer 订阅了 map 返回的 ObservableMap

* 然后 map 的 Observable(ObservableMap)在被订阅时，会订阅其内部保存上游 Observable，用于订阅上游的 Observer 是一个装饰者(MapObserver)，
  内部保存了下游（本例是终点）Observer，以便上游发送数据过来时，能传递给下游。

* 以此类推，直到源头 Observable 被订阅，它开始向 Observer 发送数据。数据传递的过程，当然是从上游push到下游的，

* 源头 Observable 传递数据给下游 Observer（本例就是MapObserver）,然后MapObserver接收到数据，对其变换操作后(实际的function在这一步执行)，
  再调用内部保存的下游 Observer 的 onNext() 发送数据给下游,以此类推，直到终点 Observer 。

