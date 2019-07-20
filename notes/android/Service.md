# Service

## 概述

在Android平台上, 那种持续性工作一般都是由service来执行的。在Android平台上已经大幅度地弱化了进程的概念, 取而代之的是一个个有意义的逻辑实体,
比如activity、service等。

Service实体必然要寄身到某个进程里才行, 它也可以再启动几个线程来帮它干活儿。但是, 说到底service只是一个逻辑实体、一个运行期上下文而已。相比 activity 这种“操控UI界面的运行期上下文”, service 一般是没有界面部分的, 不过有些特殊的service还是可以创建自己的界面的，比如当一个service
需要显现某种浮动面板时，就必须自己创建、销毁界面了。

在Android系统内部的 AMS 里, 是利用各种类型的 Record 节点来管理不同的运行期上下文的。比如以 ActivityRecord 来管理 activity, 以 ServiceRecord来管理service。

可是，线程这种东东可没有对应的Record节点喔。一些初学者常常会在activity里启动一个线程，从事某种耗时费力的工作，可是一旦activity被遮挡住，天知道它会在什么时候被系统砍掉, 进而导致连应用进程也退出。从AMS的角度来看，它压根就不知道用户进程里还搞了个工作线程在干活儿，所以当它要干掉用户进程时，是不会考虑用户进程里还有没有工作没干完。

但如果是在 service 里启动了工作线程，那么 AMS 一般是不会随便砍掉 service 所在的进程的, 所以耗时的工作也就可以顺利进行了。

可是，我们也常常遇到那种“一次性处理”的工作，难道就不能临时创建个线程来干活吗？在Android平台上应对这种情况的较好做法是创建一个 IntentService 派生类, 而后覆盖其 onHandleIntent() 成员函数。IntentService 内部会自动为你启动一个工作线程，并在工作线程里回调 onHandleIntent()。当onHandleIntent()返回后, IntentService还会自动执行stopSelf()关闭自己。

## Service机制

我们先来看一下Service类的代码截选:

```
【frameworks/base/core/java/android/app/Service.java】

public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {
    ......
    ......
    private ActivityThread mThread = null;
    private String mClassName = null;
    private IBinder mToken = null;
    private Application mApplication = null;
    private IActivityManager mActivityManager = null;
    private boolean mStartCompatibility = false;
}
```

Service是个抽象类，它间接继承于Context，其继承关系如下图所示：

![Service]()

看了这张图，请大家务必理解，Service只是个“上下文”（Context）对象而已, 它和进程、线程是没什么关系的。

在AMS中，负责管理 service 的 ServiceRecord 节点本身就是个binder实体。当AMS向应用进程发出语义, 要求其创建service对象时，会把ServiceRecord通过binder机制“传递”给应用进程。

这样，应用进程的ActivityThread在处理AMS发来的语义时，就可以得到一个合法的binder代理，这个binder代理最终会被记录在Service对象中，如此一来，Service实体就和系统里的ServiceRecord关联起来了。

假如一个应用进程里启动了两个不同的Service，那么当service创建成功之后，AMS和用户进程之间就会形成如下关系示意图：

![AMSAndApplication]()

我们对图中的ApplicationThread并不陌生，它记录在ActivityThread中mAppThread域中。每当系统新fork一个用户进程后，就会自动执行ActivityThread的attach()动作，里面会调用：

final IActivityManager mgr = ActivityManagerNative.getDefault();
. . . . . .
    mgr.attachApplication(mAppThread);
. . . . . .

将 ApplicationThread 对象远程“传递”给AMS，从而让AMS得到一个合法的代理端。而当系统要求用户进程创建 service 时，就会通过这个合法的代理端向用户进程传递明确的语义。

## startService

我们先来看启动service的流程。要启动一个service，我们一般是调用startService()。说穿了只是向 AMS 发起一个请求, 导致AMS执行如下的startService()动作：

```
【frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java】

public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, int userId) {
    . . . . . .
        ComponentName res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid, userId);
        . . . . . .
        return res;
    }
}
```

其中的mServices域是ActiveServices类型的，其startServiceLocked()函数的代码截选如下：

```
【frameworks/base/services/core/java/com/android/server/am/ActiveServices.java】

ComponentName startServiceLocked(IApplicationThread caller,
        Intent service, String resolvedType,
        int callingPid, int callingUid, int userId) {
. . . . . .
    // 必须先通过 retrieveServiceLocked()找到（或创建）一个 ServiceRecord 节点
    ServiceLookupResult res = retrieveServiceLocked(service, resolvedType,
                callingPid, callingUid, userId, true, callerFg);
    . . . . . .
    ServiceRecord r = res.record;
    . . . . . .
    r.lastActivity = SystemClock.uptimeMillis();
    r.startRequested = true;
    r.delayedStop = false;
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants));

    final ServiceMap smap = getServiceMap(r.userId);
    . . . . . .
    . . . . . .
    return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
}
```
必须先通过 retrieveServiceLocked() 找到（或创建）一个 ServiceRecord 节点, 而后才能执行后续的启动service的动作。

请大家注意，在Android frameworks里有不少这样的动作，基本上都是先创建xxxRecord节点，而后再向目标用户进程传递语义, 创建具体的对象。

在ServiceRecord类中有不少成员变量，其中有一个app域，专门用来记录 Service 对应的用户进程的ProcessRecord。当然, 一开始这个app域是为null的，
也就是说 ServiceRecord 节点还没有和用户进程关联起来, 待Service真正启动之后, ServiceRecord 的 app 域就有实际的值了。

现在我们继续看 startServiceLocked() 函数，它在找到ServiceRecord节点之后，开始调用startServiceInnerLocked()。我们可以绘制一下相关的调用关系：

![startServiceInnerLocked]()


请注意上图中 bringUpServiceLocked() 里的内容。此时大体上分三种情况：

1. 如果ServiceRecord已经和Service寄身的用户进程关联起来了，此时ServiceRecord的app域以及app.thread域都是有值的，那么只调用                    sendServiceArgsLocked()即可。这一步会让Service间接走到大家熟悉的onStartCommand()。

2. 如果尚未启动Service寄身的用户进程，那么需要调用mAm.startProcessLocked()启动用户进程。

3. 如果Service寄身的用户进程已经启动，但尚未和 ServiceRecord 关联起来，那么调用realStartServiceLocked(r, app, execInFg);这种情况下，
   会让Service先走到onCreate()，而后再走到onStartCommand()。

我们重点看第三种情况，realStartServiceLocked()的调用示意图如下：

![realStartServiceLocked]()


看到了吧，无非是利用scheduleXXX()这样的函数，来通知用户进程去做什么事。以后大家看到这种以schedule打头的函数，可以直接打开ActivityThread.java文件去查找其对应的实现函数。

比如scheduleCreateService()的代码如下：

```
【frameworks/base/core/java/android/app/ActivityThread.java】

public final void scheduleCreateService(IBinder token,
        ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
    updateProcessState(processState, false);
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;

    sendMessage(H.CREATE_SERVICE, s);
}

```
它发出的CREATE_SERVICE消息, 会由ActivityThread的内嵌类H处理，H继承于Handler, 而且是在UI主线程里处理消息的。可以看到，处理CREATE_SERVICE消息的函数是handleCreateService()：


case CREATE_SERVICE:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceCreate");
    handleCreateService((CreateServiceData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;

另外，请大家注意realStartServiceLocked()传给scheduleCreateService()函数的第一个参数就是ServiceRecord类型的r，而ServiceRecord本身是个Binder实体噢。待“传到”应用进程时，这个Binder实体对应的Binder代理被称作token，记在了CreateServiceData对象的token域中。现在CreateServiceData对象又经由msg.obj传递到消息处理函数里, 并进一步作为参数传递给handleCreateService()函数。

handleCreateService()的代码如下：

```
private void handleCreateService(CreateServiceData data) {
    . . . . . .
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    . . . . . .
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
    . . . . . .
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
 
        // 注意，ServiceRecord实体对应的代理端，就是此处的data.token。
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
        mServices.put(data.token, service);
    . . . . . .
}
```


简单地说就是，先利用反射机制创建出Service对象：

```
service = (Service) cl.loadClass(data.info.name).newInstance();

```

然后再调用该对象的onCreate()成员函数：

```
service.onCreate();
```


而因为handleCreateService()函数本身是在用户进程的UI主线程里执行的，所以service的onCreate()函数也就是在UI主线程里执行的。常常会有人告诫新手，
不要在onCreate()、onStart()里执行耗时的操作，现在大家知道是为什么了吧，因为在UI主线程里执行耗时的操作不但会引起界面卡顿，严重的还会导致ANR报错。

另外，在执行到上面的service.attach()时，那个和ServiceRecord对应的代理端token也传进来了：

```
public final void attach(
        Context context,
        ActivityThread thread, String className, IBinder token,
        Application application, Object activityManager) {
    attachBaseContext(context);
    mThread = thread;
    mClassName = className;
    mToken = token;     // 注意这个token噢，它的对端就是AMS里的ServiceRecord！
    mApplication = application;
    mActivityManager = (IActivityManager)activityManager;
    mStartCompatibility = getApplicationInfo().targetSdkVersion
            < Build.VERSION_CODES.ECLAIR;
}
```
可以看到，token记录到Service的mToken域了。

最后，新创建出的Service对象还会记录进ActivityThread的mServices表格去，于是我们可以在前文示意图的基础上，再画一张新图：

![fullStartService](https://github.com/xianfeng92/android-code-read/blob/master/images/fullStartService.png)


是startService()启动服务的大体过程。上图并没有绘制“调用startService()的用户进程”，因为当Service启动后，它基本上就和发起方没什么关系了。


## bindService

bindService()，它可比startService()要麻烦一些。当一个用户进程bindService()时，它需要先准备好一个实现了ServiceConnection接口的对象。
ServiceConnection的定义如下：

```
public interface ServiceConnection {
    public void onServiceConnected(ComponentName name, IBinder service);
    public void onServiceDisconnected(ComponentName name);
}
```

在Android平台上，每当用户调用bindService()，Android都将之视作是要建立一个新的“逻辑连接”。而当连接建立起来时，系统会回调ServiceConnection接口的onServiceConnected()。
另一方面，那个onServiceDisconnected()函数却不是在unbindService()时发生的。一般来说，当目标service所在的进程意外挂掉或者被杀掉时，系统才会回调onServiceDisconnected()，
而且，此时并不会销毁之前的逻辑连接，也就是说，那个“逻辑连接”仍然处于激活状态，一旦service后续再次运行，系统会再次回调onServiceConnected()。

在Android平台上，要和其他进程建立逻辑连接往往都需要利用binder机制。那么，发起bindService()的用户进程又是在哪里创建逻辑连接需要的binder实体呢？
我们可以看看bindServiceCommon()函数的代码截选：

```
【frameworks/base/core/java/android/app/ContextImpl.java】

private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
        UserHandle user) {
    IServiceConnection sd;
    . . . . . .
        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                mMainThread.getHandler(), flags);   // 请注意返回的sd！
    . . . . . .
        IBinder token = getActivityToken();
        . . . . . . 
        int res = ActivityManagerNative.getDefault().bindService(
            mMainThread.getApplicationThread(), getActivityToken(),
            service, service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, user.getIdentifier());
    . . . . . .
}
```

请大家注意mPackageInfo.getServiceDispatcher()那一句，它返回的就是binder实体。此处的mPackageInfo是用户进程里和apk对应的LoadedApk对象。
getServiceDispatcher()的代码如下：

```
【frameworks/base/core/java/android/app/LoadedApk.java】

public final IServiceConnection getServiceDispatcher(ServiceConnection c,
        Context context, Handler handler, int flags) {
    synchronized (mServices) {
        LoadedApk.ServiceDispatcher sd = null;
        ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = 
                                                                         mServices.get(context);
        if (map != null) {
            sd = map.get(c);
        }
        if (sd == null) {
            sd = new ServiceDispatcher(c, context, handler, flags);
            if (map == null) {
                map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                mServices.put(context, map);
            }
            map.put(c, sd);
        } else {
            sd.validate(context, handler);
        }
        return sd.getIServiceConnection();  // 注意，返回ServiceDispatcher内的binder实体
    }
}
```
也就是说，先尝试在LoadedApk的mServices表中查询ServiceDispatcher对象，如果查不到，就重新创建一个。ServiceDispatcher对象会记录下从用户处传来的ServiceConnection引用。
而且，ServiceDispatcher对象内部还含有一个binder实体，现在我们可以通过调用sd.getIServiceConnection()一句，返回这个内部的binder实体。

[Android Service演义](https://my.oschina.net/youranhongcha/blog/710046)
