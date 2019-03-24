
# 什么是Context？

  在Android平台上，Context是一个基本的概念，它在逻辑上表示一个运行期的“上下文”。在Android平台上，应用里的每个重要UI界面都用一个小型上下文来封装，
  而每个重要的对外服务也都用一个小型上下文封装。这些小型上下文都容身到一个Android大平台上，并由Android统一调度管理，形成一个统一的整体。


# Context的行为

  Context体现到代码上来说，是个抽象类，其主要表达的行为列表如下:

```
Context行为分类         常用函数

使用系统提供的服务      getPackageManager()
                       getSystemService()......




基本功能               startActivity()、sendBroadcast()
                       registerReceiver()、startService()
                       bindService()、getContentResolver()......
                      【内部基本上是和AMS打交道】




访问资源                 getAssets()、getResources()、
                         getString()、getColor()、getClassLoader()......



                            getApplicationContext()......




和信息存储相关           getSharedPreferences()、openFileInput()、
                        openFileOutput()、deleteFile()、openOrCreateDatabase()、
                        deleteDatabase()......

```

既然是抽象类，最终就总得需要实际的派生类才行。在Android上，我们可以画出如下的Context继承示意图：


![Context](https://github.com/xianfeng92/android-code-read/blob/master/images/context.jpg)

我们可以琢磨一下这张图，很明显，在Android平台上，Activity和Service在本质上都是个Context，而代表应用程序的Application对象，也是个Context，
这个对象对应的就是AndroidManifest.xml里的<Application>部分。因为上下文访问应用资源或系统服务的动作是一样的，所以这部分动作被统一封装进一个ContextImpl辅助类里。
__Activity、Service、Application内部都含有自己的ContextImpl，每当自己需要访问应用资源或系统服务时，无非是把请求委托给内部的ContextImpl而已__。

## ContextWrapper

上图中还有个ContextWrapper，该类是用于表示Context的包装类，它在做和上下文相关的动作时，基本上都是委托给内部mBase域记录的Context(即ContextIml)去做的。
如果我们希望子类化上下文的某些行为，可以有针对性地重写ContextWrapper的一些成员函数。

 ContextWrapper的代码截选如下：

```
【frameworks/base/core/java/android/content/ContextWrapper.java】

public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

    public Context getBaseContext() {
        return mBase;
    }
```

代码中的mBase是它的核心。

ContextWrapper只是简单封装了Context，它重写了Context所有成员函数，比如：

```
    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }

    @Override
    public Resources getResources() {
        return mBase.getResources();
    }
```




## ContextImpl

   前文我们已经说过，因为上下文访问应用资源或系统服务的动作是一样的，所以这部分动作被统一封装进一个ContextImpl辅助类里，现在我们就来看看这个类。

ContextImpl的代码截选如下：

```
【frameworks/base/core/java/android/app/ContextImpl.java】

class ContextImpl extends Context {
    private final static String TAG = "ContextImpl";
    private final static boolean DEBUG = false;

    @GuardedBy("ContextImpl.class")
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> 
sSharedPrefsCache;
    @GuardedBy("ContextImpl.class")
    private ArrayMap<String, File> mSharedPrefsPaths;

    final ActivityThread mMainThread;  // 依靠该成员和大系统联系
    final LoadedApk mPackageInfo;

    private final IBinder mActivityToken;
    private final UserHandle mUser;
    private final ApplicationContentResolver mContentResolver;

    private final String mBasePackageName;
    private final String mOpPackageName;

    private final @NonNull ResourcesManager mResourcesManager;
    private final @NonNull Resources mResources;  // 指明自己在用的资源
    private @Nullable Display mDisplay; 
    private final int mFlags;

    private Context mOuterContext;   // 指向其寄身并提供服务的上下文。

    private int mThemeResource = 0;
    private Resources.Theme mTheme = null;
    private PackageManager mPackageManager;
    private Context mReceiverRestrictedContext = null;
```

很明显，作为一个上下文的核心部件，ContextImpl有责任和更大的系统进行通信（我们可以把Android平台理解为一个大系统），所以它会有一个mMainThread成员。
就以启动activity动作来说吧，最后会走到ContextImpl的startActivity()，而这个函数内部大体上是进一步调用 mMainThread.getInstrumentation().execStartActivity()，
从而将语义发送给Android系统。

ContextImpl里的另一个重要方面是关于资源的访问。这就涉及到资源从哪里来。简单地说，当一个APK被加载起来时，系统会创建一个对应的LoadedApk对象，
并通过解码模块将APK里的资源部分加载进LoadedApk。每当我们为一个上下文创建对应的ContextImpl对象时，就会从LoadedApk里获取正确的Resources对象，
并记入ContextImpl的mResources成员变量，以便以后使用。


## 什么时候创建Context实例 


###  创建Application对象的时机
  
 每个应用程序在第一次启动时，都会首先创建Application对象。如果对应用程序启动一个Activity(startActivity)流程比较清楚的话，创建Application 的时机在创建handleBindApplication()方法中，该函数位于 ActivityThread.java类中 ，如下:

```
 //创建Application时同时创建的ContextIml实例
private final void handleBindApplication(AppBindData data){
    ...
    ///创建Application对象
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    ...
}
 
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {
    ...
    try {
        java.lang.ClassLoader cl = getClassLoader();
        ContextImpl appContext = new ContextImpl();    //创建一个ContextImpl对象实例
        appContext.init(this, null, mActivityThread);  //初始化该ContextIml实例的相关属性
        ///新建一个Application对象 
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
       appContext.setOuterContext(app);  //将该Application实例传递给该ContextImpl实例         
    } 
    ...
}
```


### 创建Activity对象的时机

通过startActivity()或startActivityForResult()请求启动一个Activity时，如果系统检测需要新建一个Activity对象时，就会回调handleLaunchActivity()方法，该方法继而调用performLaunchActivity()方法，去创建一个Activity实例，并且回调onCreate()，onStart()方法等， 函数都位于 ActivityThread.java类 ，如下:

```
//创建一个Activity实例时同时创建ContextIml实例
private final void handleLaunchActivity(ActivityRecord r, Intent customIntent) {
    ...
    Activity a = performLaunchActivity(r, customIntent);  //启动一个Activity
}
private final Activity performLaunchActivity(ActivityRecord r, Intent customIntent) {
    ...
    Activity activity = null;
    try {
        //创建一个Activity对象实例
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    }
    if (activity != null) {
        ContextImpl appContext = new ContextImpl();      //创建一个Activity实例
        appContext.init(r.packageInfo, r.token, this);   //初始化该ContextIml实例的相关属性
        appContext.setOuterContext(activity);            //将该Activity信息传递给该ContextImpl实例
        ...
    }
    ...    
}
```

###  创建Service对象的时机
    
通过startService或者bindService时，如果系统检测到需要新创建一个Service实例，就会回调handleCreateService()方法，完成相关数据操作。handleCreateService()函数位于 ActivityThread.java类，如下:

```
//创建一个Service实例时同时创建ContextIml实例
private final void handleCreateService(CreateServiceData data){
    ...
    //创建一个Service实例
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
    } catch (Exception e) {
    }
    ...
    ContextImpl context = new ContextImpl(); //创建一个ContextImpl对象实例
    context.init(packageInfo, null, this);   //初始化该ContextIml实例的相关属性
    //获得我们之前创建的Application对象信息
    Application app = packageInfo.makeApplication(false, mInstrumentation);
    //将该Service信息传递给该ContextImpl实例
    context.setOuterContext(service);
    ...
}
```

需要强调一点的是,ContextImp 的分析可知，其方法的大多数操作都是直接调用其属性mPackageInfo(该属性类型为PackageInfo)的相关方法而来。这说明 ContextImp 是一种轻量级类，而PackageInfo 才是真正重量级的类。而一个App 的所有ContextIml 实例，都对应同一个packageInfo对象。

所有ContextIml实例，都对应同一个packageInfo对象。


## Application中的Context和Activity中的Context的区别 

在需要传递Context参数的时候，如果是在Activity中，我们可以传递this（这里的this指的是Activity.this，是当前Activity的上下文）或者Activity.this。这个时候如果我们传入getApplicationContext()，我们会发现这样也是可以用的。可是大家有没有想过传入Activity.this和传入getApplicationContext()的区别呢？首先Activity.this和getApplicationContext()返回的不是同一个对象，一个是当前Activity的实例，一个是项目的Application的实例，这两者的生命周期是不同的，它们各自的使用场景不同，this.getApplicationContext()取的是这个应用程序的Context，它的生命周期伴随应用程序的存在而存在；而Activity.this取的是当前Activity的Context，它的生命周期则只能存活于当前Activity，这两者的生命周期是不同的。getApplicationContext() 生命周期是整个应用，当应用程序摧毁的时候，它才会摧毁；Activity.this的context是属于当前Activity的，当前Activity摧毁的时候，它就摧毁。

Activity Context 和Application Context两者的使用范围存在着差异，具体如下图所示:

![ContextDiff](https://github.com/xianfeng92/android-code-read/blob/master/images/contextDiff.png)

我们就只看Activity和Application，可以看到前三个操作不在 Application 中出现，也就是Show a Dialog、Start an Activity和Layout Inflation。开发的过程中，我们主要记住一点，凡是跟UI相关的，都用Activity做为Context来处理。



##  Context数量 

在创建Activity、Service、Application时都会自动创建Context，它们各自维护着自己的上下文。在Android系统中Context类的继承结构里面我们讲到Context一共有Application、Activity和Service三种类型，因此如果要统计一个app中Context数量，我们可以这样来表示：

Context数量 = Activity数量 + Service数量 + 1


这里要解释一下，上面的1表示Application数量。一个应用程序中可以有多个Activity和多个Service，但只有一个Application。可能有人会说一个应用程序里面可以有多个Application啊，我的理解是：一个应用程序里面可以有多个Application，可是在配置文件AndroidManifest.xml中只能注册一个，只有注册的这个Application才是真正的Application，才会调用到全部的生命周期，所以Application的数量是1。


## 参考


[认识一下Android里的Context](https://my.oschina.net/youranhongcha/blog/1807189)
[Android Application中的Context和Activity中的Context的异同](https://www.cnblogs.com/ganchuanpu/p/6445251.html)
