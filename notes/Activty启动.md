

# 一切从main()方法开始

Android中，一个应用程序的开始可以说就是从 ActivityThread.java 中的 main() 方法开始的。



main()方法中主要做的事情有：

1. 初始化主线程的Looper、主Handler。并使主线程进入等待接收Message消息的无限循环状态。



2. 调用attach()方法，主要就是为了发送出初始化Application的消息。


## 创建Application的消息是如何发送的呢？


上面提到过，ActivityThread的attach()方法最终的目的是发送出一条创建Application的消息——H.BIND_APPLICATION，到主线程的主Handler中。那我们来看看attach()方法干了啥。
attach()关键代码：

```
    private void attach(boolean system, long startSeq) {
            ...

            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }

           ...

    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    // 这里是通过ServiceManager获取到IBinder实例的。
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };

```

## IActivityManager mgr是个啥？

当我们调用 ActivityManager.getService(); 获得的实际是一个代理类的实例——ActivityManagerProxy，这个东西实现了 IActivityManager 接口。
打开源码你会发现，ActivityManagerProxy 是 ActivityManagerNative 的一个内部类。

1. 先看ActivityManagerProxy的构造函数：

```
public ActivityManagerProxy(IBinder remote) {
        mRemote = remote;
}
```

这个构造函数非常的简单。首先它需要一个IBinder参数，然后赋值给mRemote变量。这个mRemote显然是ActivityManagerProxy的成员变量，
对它的操作是由ActivityManagerProxy来代理间接进行的。这样设计的好处是保护了mRemote，并且能够在操作mRemote前执行一些别的事务，
并且我们是以IActivityManager的身份来进行这些操作的！这就非常巧妙了。


获取IBinder的目的就是为了通过这个 IBinder 和 ActivityManager进行通讯，进而ActivityManager会调度发送H.BIND_APPLICATION即初始化Application的Message消息。


## 再来看看attachApplication(mAppThread)方法

```
	public void attachApplication(IApplicationThread app){
	  ...
	  mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
	  ...
	}

```
这个方法中上面这一句是关键。调用了 IBinder 实例的 tansact() 方法，并且把参数app(这个参数稍后就会提到)放到了data中，最终传递给 ActivityManager。

现在，我们已经基本知道了IActivityManager是个什么东东了。其实最重要的就是它的一个实现类ActivityManagerProxy，
它主要代理了内核中与ActivityManager通讯的Binder实例。

## 再看看ApplicationThread mAppThread。



```
	private class ApplicationThread extends IApplicationThread.Stub

```

ApplicationThread是ActivityThread中的一个内部类，为什么没有单独出来写在别的地方呢？我觉得这也是对最小惊异原则的实践。因为ApplicationThread是专门在这里使用的对象。

那么很明显，ApplicationThread最终也是一个Binder！同时，由于实现了IApplicationThread接口，所以它也是一个 IApplicationThread。

我们在ActivityThread中看到的ApplicationThread使用的构造函数是无参的，所以看上面无参构造函数都干了啥！

好，我们终于知道attach()方法中出现的两个对象是啥了。ApplicationThread 作为 IApplicationThread 的一个实例，承担了最后发送Activity生命周期、及其它一些消息的任务。
也就是说，前面绕了一大圈，最后还是回到这个地方来发送消息！



## ActivityManagerService调度发送初始化消息

经过上面的辗转，ApplicationThread终于到了ActivityManagerService中了。请在上图中找到对应位置！


从上图中可以看到，ActivityManagerService中有一这样的方法：

```
	private final boolean attachApplicationLocked(IApplicationThread thread
	, int pid) {
	    ...
	    thread.bindApplication();
	    //注意啦！
	    ...
	}

```
ApplicationThread 以 IApplicationThread 的身份到了 ActivityManagerService 中，经过一系列的操作，最终被调用了自己的bindApplication()方法，发出初始化Applicationd的消息。

```
  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {

            ...

            sendMessage(H.BIND_APPLICATION, data);
        }

```

但是，这个地方，我们只要知道最后发了一条 H.BIND_APPLICATION 消息，接着程序开始了。


## 收到初始化消息之后的世界

上面我们已经找到初始化Applicaitond的消息是在哪发送的了。现在，需要看一看收到消息后都发生了些什么。


现在上图的H下面找到第一个消息：H.BIND_APPLICATION。一旦接收到这个消息就开始创建Application了。这个过程是在handleBindApplication()中完成的。看看这个方法。在上图中可以看到对应的方法。

```
private void handleBindApplication(AppBindData data) {
    ...
    mInstrumentation = (Instrumentation)
        cl.loadClass(data.instrumentationName.getClassName())
        .newInstance();
    //通过反射初始化一个Instrumentation仪表。
    ...
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    //通过LoadedApp命令创建Application实例
    mInitialApplication = app;
    ...
    mInstrumentation.callApplicationOnCreate(app);
    //让仪器调用Application的onCreate()方法
    ...
}

```

## Instrumentation仪表，什么鬼？

Instrumentation会在应用程序的任何代码运行之前被实例化，它能够允许你监视应用程序和系统的所有交互。

从上面的代码我们可以看出，Instrumentation确实是在Application初始化之前就被创建了。那么它是如何实现监视应用程序和系统交互的呢？

打开这个类你可以发现，最终Apllication的创建，Activity的创建，以及生命周期都会经过这个对象去执行。
简单点说，就是把这些操作包装了一层。通过操作Instrumentation进而实现上述的功能。


那么这样做究竟有什么好处呢？仔细想想。Instrumentation作为抽象，当我们约定好需要实现的功能之后，我们只需要给Instrumentation仪表添加这些抽象功能，
然后调用就好。剩下的，不管怎么实现这些功能，都交给Instrumentation仪器的实现对象就好。
啊！这是多态的运用。啊！这是依赖抽象，不依赖具体的实践。啊！这是上层提出需求，底层定义接口，即依赖倒置原则的践行。呵！抽象不过如此。


从代码中可以看到，这里实例化Instrumentation的方法是反射！而反射的ClassName是来自于从ActivityManagerService中传过来的Binder的。

套路太深！就是为了隐藏具体的实现对象。但是这样耦合性会很低。

既然在说Instrumentation，那就看看最后调的 callApplicationOnCreate()方法。

```
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }

```

## LoadedApk就是data.info

```
public Application makeApplication(boolean forceDefaultAppClass,
    Instrumentation instrumentation) {
    ...
    String appClass = mApplicationInfo.className;
    //Application的类名。明显是要用反射了。
    ...
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread
        , this);
    //留意下Context
    app = mActivityThread.mInstrumentation
        .newApplication( cl, appClass, appContext);
    //通过仪表创建Application
    ...
}

```
在这个方法中，我们需要知道的就是，在取得Application的实际类名之后，最终的创建工作还是交由Instrumentation去完成，就像前面所说的一样。


## 现在把目光移回Instrumentation

看看newApplication()中是如何完成Application的创建的。

```
static public Application newApplication(Class<?> clazz
    , Context context) throws InstantiationException
    , IllegalAccessException
    , ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        //反射创建，简单粗暴
        app.attach(context);
        //关注下这里，Application被创建后第一个调用的方法。
        //目的是为了绑定Context。
        return app;
    }
```



# LaunchActivity

当Application初始化完成后，系统会更具Manifests中的配置的启动Activity发送一个Intent去启动相应的Activity。











* Activity：这个大家都熟悉，startActivity方法的真正实现在Activity中

* Instrumentation：用来辅助Activity完成启动Activity的过程

* ActivityThread（包含ApplicationThread + ApplicationThreadNative + IApplicationThread）：真正启动Activity的实现都在这里


```
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

```

显然，从上往下，最终都是由 startActivityForResult 来实现的,

```

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
         //一般的Activity其mParent为null，mParent常用在ActivityGroup中，ActivityGroup已废弃
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
         //这里会启动新的Activity，核心功能都在mMainThread.getApplicationThread()中完成
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
         //发送结果，即onActivityResult会被调用
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
         //在ActivityGroup内部的Activity调用startActivity的时候会走到这里，内部处理逻辑和上面是类似的
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

可以发现，真正打开 activity 的实现在 Instrumentation 的 execStartActivity 方法中，去看看

code：Instrumentation#execStartActivity

```
    public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, String target,
        Intent intent, int requestCode, Bundle options) {

        //核心功能在这个whoThread中完成，其内部scheduleLaunchActivity方法用于完成activity的打开
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                  //先查找一遍看是否存在这个activity
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                    //如果找到了就跳出循环
                        am.mHits++;
                        if (am.isBlocking()) {
                   //如果目标activity无法打开，直接return
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //这里才是真正打开 activity 的地方，核心功能在 whoThread 中完成。
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target, requestCode, 0, null, options);

                 //这个方法是专门抛异常的，它会对结果进行检查，如果无法打开activity，
		//则抛出诸如ActivityNotFoundException类似的各种异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

再说一下这个方法checkStartActivityResult，它也专业抛异常的，看代码，相信大家对下面的异常信息不陌生吧，就是它干的，
其中最熟悉的非 Unable to find explicit activity class 莫属了，如果你在xml中没有注册目标activity，此异常将会抛出。


再来看看  ActivityManager.getService() 得到的是啥？

```
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };

```

通过上面代码可知得到的是一个 IActivityManager，其为 ActivityManagerService 的远程Binder对象。通过远程Binder对象调用 ActivityManagerService 的 startActivity 方法。

接着startActivity 这个调用了很多层级的函数。。这里省略 。。。到后面调用了一个 Ibind 的接口回到到 app 进程中 ApplicationThread 类的 handleLaunchActivity 方法。

然后调用 H 的sendMessage（） msg.what 为LAUNCH_ACTIVITY，紧接着 去执行performLaunchActivity（），performLaunchActivity里面的动作是 newActivity （真真正正的创建activity）

然后创建application 再去 执行activity 的attach 函数，attach 里面创建了window对象。


接下来我们要去看看 IApplicationThread，因为核心功能由其内部的 scheduleLaunchActivity 方法来完成，由于IApplicationThread是个接口，所以，我们需要找到它的实现类，
它就是ActivityThread中的内部类ApplicationThread，看下它的继承关系：








# 参考

[Android源码分析-Activity的启动过程](https://blog.csdn.net/singwhatiwanna/article/details/18154335)
[3分钟看懂Activity启动流程](https://www.jianshu.com/p/9ecea420eb52)
[Android Launcher 启动 Activity 的工作过程](https://blog.csdn.net/qian520ao/article/details/78156214)
