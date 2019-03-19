
# 基本介绍

  RxJava 本质上可以说到它就是一个实现异步操作的库。异步操作很关键的一点是程序的简洁性，因为在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。 Android 创造的 AsyncTask 和 Handler , 其实都是为了让异步代码更加简洁。RxJava 的优势也是简洁，但它的简洁的与众不同之处在于，随着程序逻辑变得越来越复杂，它依然能够保持简洁。

## RxJava 的观察者模式

   RxJava 有四个基本概念：Observable (可观察者，即被观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。

### 基于以上的概念， RxJava 的基本实现主要有三点:

1. 创建 Observer & 定义响应事件的行为

```
  // Observer 即观察者，它决定事件触发的时候将有怎样的行为
    Observer<Integer> observer = new Observer<Integer>() {
        // 在发生订阅的时候会回调此方法
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: ");
        }

        @Override
        public void onNext(Integer integer) {
            Log.d(TAG, "onNext: "+integer);
        }
        // 事件队列异常。在事件处理过程中出异常时，onError() 会被触发，同时队列自动终止，不允许再有事件发出
        @Override
        public void onError(Throwable e) {

        }
        // 事件队列完结时回调，RxJava 不仅把每个事件单独处理，还会把它们看做一个队列
        // RxJava 规定，当不会再有新的 onNext() 发出时，需要触发 onCompleted() 方法作为标志
        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: ");
        }
    };
```


2. 创建 Observable & 定义需发送的事件

```
    // Observable 即被观察者，它决定什么时候触发事件以及触发怎样的事件
    // RxJava 使用 create() 方法来创建一个 Observable ，并为它定义事件触发规则
    Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
            emitter.onNext(1);
            emitter.onNext(2);
            emitter.onNext(3);
            emitter.onComplete();
        }
    });

```

create() 方法是 RxJava 最基本的创造事件序列的方法。基于这个方法， RxJava 还提供了一些方法用来快捷创建事件队列，例如：

```
Observable<String> observable = Observable.just("Android","IOS","Other");

String[] os = new String[]{"Android","IOS","Other"};
Observable<String> observable2 = Observable.fromArray(os);
```

3.  Subscribe (订阅) & 连接观察者和被观察者

    创建了 Observable 和 Observer 之后，再用 subscribe() 方法将它们联结起来，整条链子就可以工作了。

```
observable.subscribe(observer);
```



## 线程控制--Scheduler

   __在不指定线程的情况下， RxJava 遵循的是线程不变的原则__，即：在哪个线程调用 subscribe()，就在哪个线程生产事件; 在哪个线程生产事件，就在哪个线程消费事件。
   如果需要切换线程，就需要用到 Scheduler （调度器）。

   在RxJava 中，Scheduler ——调度器，相当于线程控制器，RxJava 通过它来指定每一段代码应该运行在什么样的线程。RxJava 已经内置了几个 Scheduler ，它们已经适合大多数的使用场景：

1. Schedulers.immediate(): 直接在当前线程运行，相当于不指定线程。这是默认的 Scheduler

2. Schedulers.newThread(): 总是启用新线程，并在新线程执行操作

3. Schedulers.io(): I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io()
   的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。

4. Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 
   使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。

5. Android 还有一个专用的 AndroidSchedulers.mainThread()，它指定的操作将在 Android 主线程运行。


```
       observable
                .subscribeOn(Schedulers.io()) // 在 Schedulers.io() 线程中生产事件
                .observeOn(AndroidSchedulers.mainThread())// 在AndroidSchedulers.mainThread()中消费事件
                .subscribe(observer);
```


PS：

 subscribeOn() 的线程控制可以从事件发出的开端就造成影响， 而 observeOn() 控制的是它后面的线程



## 变换

   RxJava 提供了对事件序列进行变换的支持，这是它的核心功能之一。所谓变换，就是将事件序列中的对象或整个序列进行加工处理，转换成不同的事件或事件序列

### map

事件对象的直接变换:

```
      observable
                .subscribeOn(Schedulers.io())
                // 利用map，将每个事件从 Integer ---> String
                .map(new Function<Integer, String>() {
                    @Override
                    public String apply(Integer integer) throws Exception {
                        return "hello"+integer;
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.d(TAG, "accept: "+s);
                    }
                });

```


### flatMap


```
public class Course {
    private String course;

    public Course(String course){
        this.course = course;
    }

    public String getCourse() {
        return course;
    }
}

public class Student {
    private Course[] course = new Course[2];

    public Student(String subject1, String subject2){
        course[0] = new Course(subject1);
        course[1] = new Course(subject2);
    }

    public Course[] getCourse() {
        return course;
    }
}

protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Student[] students = new Student[]{new Student("英语","数学"),new Student("物理","化学")};
        Observable.fromArray(students)
                .subscribeOn(Schedulers.io())
                .flatMap(new Function<Student, ObservableSource<Course>>() {
                    @Override
                    public ObservableSource<Course> apply(Student student) throws Exception {
                        return Observable.fromArray(student.getCourse());
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<Course>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        Log.d(TAG, "onSubscribe: ");
                    }

                    @Override
                    public void onNext(Course course) {
                        Log.d(TAG, "onNext: "+course.getCourse());
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "onError: ");
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "onComplete: ");
                    }
                });
    }
```

输出如下:

```
2019-03-10 21:00:33.465 9245-9245/com.xforg.demo_rxjava D/MainActivity: onSubscribe: 
2019-03-10 21:00:33.516 9245-9245/com.xforg.demo_rxjava D/MainActivity: onNext: 英语
2019-03-10 21:00:33.516 9245-9245/com.xforg.demo_rxjava D/MainActivity: onNext: 数学
2019-03-10 21:00:33.516 9245-9245/com.xforg.demo_rxjava D/MainActivity: onNext: 物理
2019-03-10 21:00:33.517 9245-9245/com.xforg.demo_rxjava D/MainActivity: onNext: 化学
2019-03-10 21:00:33.517 9245-9245/com.xforg.demo_rxjava D/MainActivity: onComplete: 
```


flatMap() 和 map() 有一个相同点: 它也是把传入的参数转化之后返回另一个对象。但需要注意，和 map() 不同的是， flatMap() 中返回的是个 ObservableSource 对象，并且这个 ObservableSource 对象并不是被直接发送到了 Subscriber 的回调方法中。

 flatMap() 的原理是这样的:

 1. 使用传入的事件对象创建一个 ObservableSource 对象

 2. 并不发送这个 ObservableSource, 而是将它激活，于是它开始发送事件

 3. 每一个创建出来的 ObservableSource 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 
    的回调方法。这三个步骤，把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。



### lift()

   这些变换虽然功能各有不同，但实质上都是针对事件序列的处理和再发送。而在 RxJava 的内部，它们是基于同一个基础的变换方法： lift(Operator)












[友好 RxJava2.x 源码解析](https://juejin.im/post/5a209c876fb9a0452577e830)


