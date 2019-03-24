# Android中常见的内存泄露

  内存泄露就是指该被GC垃圾回收的，由于有另外一个对象仍然在引用它，导致无法回收，造成内存泄露，过多的内存泄露会导致OOM.Android中的内存泄露通常是Activity或者Fragment的泄露。

主要有下面几种情况:

## 单例模式导致内存泄漏（实质是静态变量引用Activity）

单例由于它的静态特性使得其生命周期跟应用一样长, 如果我们把上下文context（比如说一个Activity）传入到了单例类中的执行业务逻辑，这时候静态变量就引用了我们的Activity,如果没有及时置空,就会在这个Activity finish的时候，导致该Activty一直驻留在内存中,并发生内存泄漏。

解决方案：

在单例中我们尽可能的引用生命周期较长的对象即可。

mInstance = new SingleUtils(context.getApplicationContext());


## 匿名内部类导致内存泄漏

匿名内部类同样会持有一个外部类的引用，比如说在Activity的onCreate()方法中定义了一个匿名的 AsyncTask 对象。如果 Activity 被销毁之后 AsyncTask 仍然在执行, 那就会阻止垃圾回收器回收 Activity 对象,进而导致内存泄漏,直到执行结束才能回收 Activity。

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
 
    new AsyncTask<Void, Void, Void>() {
        @Override
        protected Void doInBackground(Void... params) {
            //子线程中持有Activity的引用
            //子线程在Activity销毁后依然会继续执行，导致该Activity内存泄漏
            while (true) ;
        }
    }.execute();
}
```

解决方案：

1. 在onDestroy中中断子线程的运行。

2. 使用全局的线程池代替在类中创建子线程。


## 非静态内部类

 非静态内部类会持有外部类的一个引用,如果这时候有一个静态变量引用了非静态内部类,导致非静态内部类的生命周期比外部类（Activity）长，就会导
 致外部类在该被回收的时候,无法被回收掉,引起内存泄露.

 这种情况属于静态变量间接引用了 Activity 或者 Fragment

 解决办法:

  将非静态内部类改成静态内部类，或者直接抽离成一个外部类。

如果在静态内部类中，需要引用外部类对象，那么可以将这个引用封装在一个WeakReference中。如下面代码所示:

```
    static class MyHandler extends Handler{
        private WeakReference<Activity> mWeakReference;
        public MyHandler(Activity activity){
            mWeakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            // do something
            Log.d(TAG, "handleMessage: "+mWeakReference.get().getLocalClassName());
        }
    }
```

PS: 在WeakReference弱引用,它的特点是GC在回收时会忽略掉弱引用,即就算有弱引用指向某对象,但只要该对象没有被强引用指向（实际上多数时候还要求没有软引用，但此处软引用的概念可以忽略），该对象就会在被GC检查到时回收掉。对于上面的代码,用户在关闭Activity之后，就算后台线程还没结束,但由于仅有一条来自Handler的弱引用指向Activity，所以GC仍然会在检查的时候把Activity回收掉。这样,内存泄露的问题就不会出现了.

## 静态的View

有时，当一个 Activity 经常启动，但是对应的 View 读取非常耗时，我们可以通过静态 View 变量来保持对该 Activity 的 rootView 引用。这样就可以不用每次启动 Activity 都去读取并渲染View了。这确实是一个提高Activity启动速度的好方法! 但是要注意，一旦 View attach 到我们的 Window 上, 就会持有一个Context(即Activity)的引用。而我们的 View 又是一个静态变量，所以导致Activity不被回收。


解决办法:

  在使用静态View时,需要确保在资源回收时,将静态 View detach 掉。


## Handler

如下所示我们在 Activity 中定义了一个 handler,然后在 Activity 的 onCreate() 方法中，发送了一条延迟消息。
当 Activity finish 的时候，延时消息还保存在主线程的消息队列里。而且这条消息持有对 handler 的引用,而 handler 又持有对 Activity 引用。这条引用关系会保持到消息被处理,从而这就阻止了 Activity 被垃圾回收器回收, 造成内存泄漏。

```
private final Handler handler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
 
    }
};
 
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
 
    handler.postDelayed(new Runnable() {
        @Override
        public void run() { /* ... */ }
    }, Integer.MAX_VALUE); 
}
```



## 资源对象没关闭造成内存泄漏

当我们打开资源时,一般都会使用缓存。比如读写文件资源、打开数据库资源、使用Bitmap资源等等。当我们不再使用时,应该关闭它们,使得缓存内存区域及时回收。虽然有些对象，如果我们不去关闭，它自己在finalize()函数中会自行关闭。但是这得等到GC回收时才关闭，这样会导致缓存驻留一段时间。如果我们频繁的打开资源，内存泄漏带来的影响就比较明显了。

解决办法:

  及时关闭资源


## 总结

android中的很多内存泄露都是由于在Activity中使用了非静态内部类导致的，我们在使用非静态内部类一定要格外注意，如果该非静态内部类的实例对象的生命周期大于外部对象,那么就有可能导致内存泄露.
























