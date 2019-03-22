# 主线程的消息循环机制

当Android应用程序启动后，系统会创建一个叫做“main”的线程。它就是主线程，也叫UI线程，非常重要。在Android系统中，主线程主要负责执行四大组件的执行。
负责分发事件给构建，包括绘制事件。

Android中规定访问UI只能在主线程进行，如果在子线程中访问UI，那么程序就会抛出异常。ViewRootImpl中对UI的操作进行了验证，由它的checkThread()方法来完成的。
代码如下：

```
 void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

那么为什么安卓系统不允许在子线程中访问UI呢？这是因为Android的UI控件不是线程安全的，如果在多线程中并发访问可能会导致UI控件处于不可预期的状态。
如果在UI控件的访问加上锁机制的话，由于锁机制会阻塞线程的执行，会降低UI访问的效率。


主线程的主要责任:

1. 快速的处理UI事件。Android希望UI线程能快速响应用户操作，如果UI线程花太多时间处理后台的工作，会让用户有非常糟糕的体验。当UI事件发生时，
   让用户等待时间超过5秒而未处理，Android系统就会给用户显示ANR提示信息。

2. 快速的处理Broadcast消息。在BroadcastReceiver的onReceive()函数中，不宜占用太长的时间，否则会导致主线程无法处理其它的Broadcast消息或UI事件。
   如果占用时间超过10秒， Android系统就会给用户显示ANR提示信息。


Android 的主线程的入口在ActivityThread#main方法。下面我们来看一下:


```
public static void main(String[] args) {
        ....

        //创建Looper和MessageQueue对象，用于处理主线程的消息
        Looper.prepareMainLooper();

        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread(); 

        //建立Binder通道 (创建新线程)
        thread.attach(false);

        Looper.loop(); //消息循环运行
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

Activity的生命周期都是依靠主线程的 Looper.loop，当收到不同Message时则采用相应措施。一旦退出消息循环，那么你的程序也就可以退出了。 

从消息队列中取消息可能会阻塞，取到消息会做出相应的处理。如果某个消息处理时间过长，就可能会影响UI线程的刷新速率，造成卡顿的现象。

```
thread.attach(false)方法函数中便会创建一个Binder线程（具体是指ApplicationThread，Binder的服务端，用于接收系统服务AMS发送来的事件），
该Binder线程通过Handler将Message发送给主线程。比如收到msg=H.LAUNCH_ACTIVITY，则调用ActivityThread.handleLaunchActivity()方法，最终会通过反射机制，
创建Activity实例，然后再执行Activity.onCreate()等方法；再比如收到msg=H.PAUSE_ACTIVITY，则调用ActivityThread.handlePauseActivity()方法，最终会执行Activity.onPause()等方法。
```


主线程的消息又是哪来的呢？

当然是App进程中的其他线程通过 Handler 发送给主线程。

## system_server进程

system_server进程是系统进程，java framework框架的核心载体，里面运行了大量的系统服务，比如这里提供 ApplicationThreadProxy（简称ATP），ActivityManagerService（简称AMS），
这个两个服务都运行在system_server进程的不同线程中，由于ATP和AMS都是基于IBinder接口，都是binder线程，binder线程的创建与销毁都是由binder驱动来决定的。


## App进程

App进程则是我们常说的应用程序，主线程主要负责Activity/Service等组件的生命周期以及UI相关操作都运行在这个线程； 另外，每个App进程中至少会有两个binder线程 
ApplicationThread(简称AT)和ActivityManagerProxy（简称AMP）。


## Binder

Binder用于不同进程之间通信，由一个进程的Binder客户端向另一个进程的服务端发送事务。handler用于同一个进程中不同线程的通信。


![]()

ActivityThread 通过 ApplicationThread 和 AMS 进行进程间通讯，AMS 以进程间通信的方式完成 ActivityThread 的请求后会回调 ApplicationThread 中的 Binder方法，
然后 ApplicationThread 会向 H 发送消息，H 收到消息后会将 ApplicationThread 中的逻辑切换到 ActivityThread 中去执行，即切换到主线程中去执行，
这个过程就是__主线程的消息循环模型__。


另外，ActivityThread 实际上并非线程，不像 HandlerThread 类，ActivityThread 并没有真正继承Thread类


那么问题又来了，既然 ActivityThread 不是一个线程，那么 ActivityThread 中 Looper 绑定的是哪个 Thread？


## ActivityThread 的动力是什么？

### 进程

每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，
让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。 

### 线程

线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，
都是一个task_struct结构体，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片。



__其实承载 ActivityThread 的主线程就是由Zygote fork而创建的进程__。



