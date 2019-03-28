
## 1. 创建操作符

### 1.1 create()

创建一个被观察者:

```
public static <T> Observable<T> create(ObservableOnSubscribe<T> source)

```

下面的代码创建 ObservableOnSubscribe 并重写其 subscribe 方法。当 ObservableOnSubscribe 被 subscribe 时，就可以通过 ObservableEmitter 发射器向其观察者发送事件。

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

```

### 1.2 just()

```
public static <T> Observable<T> just(T item) 
```

创建一个被观察者，并发送事件，发送的事件不可以超过10个以上。


### 1.3 From 操作符

#### 1.3.1 fromArray()

```
public static <T> Observable<T> fromArray(T... items)

```

这个方法和 just() 类似，只不过 fromArray 可以传入多于10个的变量，并且可以传入一个数组。













































