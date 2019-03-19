## 适配器模式

   将一个接口转换成客户希望的另一个接口，使接口不兼容的那些类可以一起工作，其别名为包装器(Wrapper)。

## Rxjava 中适配器模式

拿下面一段代码作为栗子：

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
                .map(new Function<String, String>() {
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

Observable.create 返回的是一个 ObservableCreate 对象，看看该对象的源码：

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

从上面的代码可知，ObservableCreate 通过一个 source 变量保存了对 ObservableOnSubscribe 对象的引用。

ObservableOnSubscribe 其实就是一个接口：

```
public interface ObservableOnSubscribe<T> {
    void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
}
```

而所有的 Observable 其实都是 ObservableSource 接口的子类，再看看 ObservableSource 接口的源码：

```
public interface ObservableSource<T> {
    void subscribe(@NonNull Observer<? super T> observer);
}
```

当我们在 Observable.create 时，我们希望返回一个实现 ObservableSource 接口的对象（因为我们想要链式调用），
然而输入的却是一个实现 ObservableSource 接口的对象，怎么办呢？

__我们通过增加一个新的适配器类来解决接口不兼容的问题，使得原本没有任何关系的类可以协同工作__。

此处这个新的适配器类就是 ObservableCreate，其通常继承抽象目标类（ObservableSource），并通过组合适配者（ObservableOnSubscribe），
从而使得目标类与适配者之间形成关联。



## 观察者模式

   观察者模式(Observer Pattern)：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。
   Observable 就是一个被观察者，而 Observer 为观察者，当 Observer 订阅 Observable 时，Observer 就可以收到 Observable 的事件通知。



## 装饰者模式

装饰者模式：在不改变原类文件以及不使用继承的情况下，动态地将责任附加到对象上，从而实现动态拓展一个对象的功能。它是通过创建一个包装对象，
也就是装饰来包裹真实的对象。

## Rajava 中的装饰者模式


继续看上面的栗子，对 ObservableOnSubscribe 完成适配后，我们得到一个 ObservableCreate 对象，来看看在 ObservableCreate.map 发生了什么？

```
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }

```

通过上面代码可知，生成了一个 ObservableMap 对象，看看该对象的源码：

```
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

```

ObservableMap 中会对 ObservableCreate 以及 function 对象进行保存，__将 ObservableCreate 包装成了 ObservableMap 对象__。

最后当我们调用 ObservableMap.subscribe 时，其实是会调用其 subscribeActual 方法，源码如下：

```
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

其中 source 为 ObservableCreate 对象，即当 Observer 订阅 ObservableMap 时，ObservableMap 同时会去订阅 ObservableCreate，即一层一层地向上订阅。

再看看 MapObserver：

```
    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            downstream.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Override
        public U poll() throws Exception {
            T t = qd.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
```

MapObserver 把我们原来的 Observer 对象包装成 MapObserver 对象，__MapObserver 对象对 Observer 对象的功能进行了增强__。如何增强的呢？
我们可以看其 onNext 函数，即 MapObserver 在 onNext 中对上游push下来的数据先进行 mapper.apply(t) 转换，然后将结果 downstream.onNext(v) 回调
给 Observer。

即 在不改变原类文件（Observer）以及不使用继承的情况下，动态地将责任附加到对象上（mapper.apply(t)），从而实现动态拓展一个对象的功能。
它是通过创建一个包装对象（MapObserver），也就是装饰来包裹真实的对象（Observer）。


















































































































