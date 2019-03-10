## Android中 startService 和 bindService 的区别

Service属于android四大组件之一，在很多地方经常被用到。开启Service有两种不同的方式：startService和bindService。
不同的开启方式，Service执行的生命周期方法也不同。

1. startService 开启服务时，生命周期执行的方法依次是：

   onCreate() ==> onStartCommand()
   
   之后,调用多次 startService ,onCreate 只有第一次会被执行，而 onStartCommand 会执行多次
   
2. 结束服务时，调用 stopService，生命周期执行 onDestroy 方法，并且多次调用 stopService 时，onDestroy 只有第一次会被执行。


3 bindService 开启服务

```
//开启服务
Intent service = new Intent(this, MyService.class);
MyConnection conn = new MyConnection();
//第一个参数：Intent意图
//第二个参数：是一个接口，通过这个接口接收服务开启或者停止的消息，并且这个参数不能为null
//第三个参数：开启服务时的操作，BIND_AUTO_CREATE 代表自动创建 service
bindService(service, conn, BIND_AUTO_CREATE);
```
bindService 的方法参数需要一个 ServiceConnection 接口的实现类对象，我们自己写一个 MyConnection 类，并实现里面的方法。

```
    class MyServerConnec implements ServiceConnection{

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "onServiceConnected: ");
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "onServiceDisconnected: ");
        }
    }
```
* bingService开启服务时，根据生命周期里onBind方法的返回值是否为空，有两种情况:

  1. onBind返回值是 null
  
  ```
      @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind: ");
        return null;
    }
 ```
 调用bindService开启服务，生命周期执行的方法依次是：
 
 ```
  2019-03-10 10:29:26.056 20362-20362/com.xforg.demo_service D/MyService: onCreate: 
  2019-03-10 10:29:26.058 20362-20362/com.xforg.demo_service D/MyService: onBind: 
 ```
 即使多次调用 bindService，onCreate 和 onBind 也只在第一次调用时被回调。
 
 调用 unbindService 结束服务，生命周期执行 onDestroy 方法，并且 unbindService 方法只能调用一次，多次调用应用会抛出异常
 使用时也要注意调用 unbindService 一定要确保服务已经开启，否则应用会抛出如下异常:
 
```
2019-03-10 10:38:56.759 20828-20828/com.xforg.demo_service E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.xforg.demo_service, PID: 20828
    java.lang.IllegalArgumentException: Service not registered: com.xforg.demo_service.MainActivity$MyServerConnec@33ab3bb
        at android.app.LoadedApk.forgetServiceDispatcher(LoadedApk.java:1562)
        at android.app.ContextImpl.unbindService(ContextImpl.java:1692)
```

2. onBind返回值不为null

   那么我们就在自己写的 MyService 里创建一个内部类 myBinder，让它继承 Binder，并在 onBind 方法里返回 myBinder 的对象。
   
```
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind: ");
        return new myBinder();
    }

    class myBinder extends Binder{
        public int add(int a, int b){
            return a + b;
        }
    }
  ```
  此时我们在 onServiceConnected时,做如下操作:
  
```
class MyServerConnec implements ServiceConnection{
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "onServiceConnected: ");
            myService = (MyService.myBinder) service;
            int result = myService.add(1,2);
            Log.d(TAG, "onServiceConnected: result is "+result);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "onServiceDisconnected: ");
        }
    }
```

这时候调用 bindService 开启服务，生命周期执行的方法依次是：onCreate() ==> onBind() ==> onServiceConnected()

```
2019-03-10 10:53:16.679 21687-21687/com.xforg.demo_service D/MainActivity: onServiceConnected: 
2019-03-10 10:53:16.679 21687-21687/com.xforg.demo_service D/MainActivity: onServiceConnected: result is 3
```
调用多次 bindService ，onCreate、onBind、onServiceConnected 都只在第一次调用时被执行。

PS: 注意这里的 MyService 和 Activity 是在同一个进程中!!!!


### startService 和 bindService 开启服务时，他们与 activity 之间的关系

1. startService 开启服务以后，与activity就没有关联，不受影响，独立运行。

2. bindService 开启服务以后，与 activity 存在关联，退出 activity 时必须调用 unbindService 方法，否则会报 ServiceConnection 泄漏的错误。

```
2019-03-10 11:03:49.074 21687-21687/com.xforg.demo_service E/ActivityThread: Activity com.xforg.demo_service.MainActivity has leaked ServiceConnection com.xforg.demo_service.MainActivity$MyServerConnec@ac35cd1 that was originally bound here
    android.app.ServiceConnectionLeaked: Activity com.xforg.demo_service.MainActivity has leaked ServiceConnection com.xforg.demo_service.MainActivity$MyServerConnec@ac35cd1 that was originally bound here
        at android.app.LoadedApk$ServiceDispatcher.<init>(LoadedApk.java:1610)
```
当同一个服务用 startService 和 bindService 两种方式一同开启，没有先后顺序的要求，MyService 的 onCreate 只会执行一次。关闭服务需要 stopService 和 unbindService 都被调用，
也没有先后顺序的影响，MyService的 onDestroy 也只执行一次。但是如果只用一种方式关闭服务，不论是哪种关闭方式，onDestroy都不会被执行，服务也不会被关闭。

## Service: onStartCommand 的返回值

   Service类有个生命周期方法叫 onStartCommand ,每次启动服务(startService)都会回调此方法。此方法的原型如下:

```
public int onStartCommand(Intent intent, int flags, int startId)
```
需要关注的是这个方法有一个整型的返回值，它有以下选项:

```
START_STICKY_COMPATIBILITY
START_STICKY
START_NOT_STICKY
START_REDELIVER_INTENT
```
它们将影响服务异常终止情况下重启服务时的行为，默认情况下，当我们的服务因为系统内存吃紧或者其他原因被异常终止时，
系统会尝试在某个时刻重新启动服务.

1. START_NOT_STICKY

如果系统在onStartCommand()方法返回之后杀死这个服务，那么直到接受到新的Intent对象，这个服务才会被重新创建。这是最安全的选项，
用来避免在不需要的时候运行你的服务。

2. START_STICKY(系统的默认策略)

如果系统在onStartCommand()返回后杀死了这个服务，系统就会重新创建这个服务并且调用onStartCommand()方法，但是它不会重新传递最后的Intent对象，
系统会用一个null的Intent对象来调用onStartCommand()方法，在这个情况下，除非有一些被发送的Intent对象在等待启动服务。
这适用于不执行命令的媒体播放器（或类似的服务），它只是无限期的运行着并等待工作的到来。

3. START_REDELIVER_INTENT

如果系统在onStartCommand()方法返回后杀死了这个服务，系统就会重新创建了这个服务，并且用发送给这个服务的最后的Intent对象调用了onStartCommand()方法。
任意等待中的Intent对象会依次被发送。这适用于那些应该立即恢复正在执行的工作的服务，如下载文件。



