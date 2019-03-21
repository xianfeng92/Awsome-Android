
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


![Context]()

我们可以琢磨一下这张图，很明显，在Android平台上，Activity和Service在本质上都是个Context，而代表应用程序的Application对象，也是个Context，
这个对象对应的就是AndroidManifest.xml里的<Application>部分。因为上下文访问应用资源或系统服务的动作是一样的，所以这部分动作被统一封装进一个ContextImpl辅助类里。
__Activity、Service、Application内部都含有自己的ContextImpl，每当自己需要访问应用资源或系统服务时，无非是把请求委托给内部的ContextImpl而已__。



# ContextWrapper

 上图中还有个ContextWrapper，该类是用于表示Context的包装类，它在做和上下文相关的动作时，基本上都是委托给内部mBase域记录的Context去做的。
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


另外，为了便于访问 Android 提供的系统服务，ContextImpl 里还构造了一个小cache，记在 mServiceCache 成员变量里：

```
final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();
```

SystemServiceRegistry是个辅助类，其createServiceCache()函数的代码很简单，只是new了一个Object数组并返回之。也就是说，一开始这个mServiceCache数组，里面是没有内容的。
日后，当我们需要访问系统服务时，会在运行期填写这个数组的某些子项。那么我们很容易想到，一个应用里启动的不同Activity，其内部的ContextImpl所含有的mServiceCache数组内容，
常常也是不同的，而且一般情况下这个数组还是比较稀疏的，也就是说含有许多null。

大家在写代码时，常常会写下类似下面的句子：

```
ActivityManager am = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
```

该函数最终会调用到ContextImpl的同名函数，函数代码如下：

```
【frameworks/base/core/java/android/app/ContextImpl.java】

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```

继续使用着辅助类SystemServiceRegistry。

```
【frameworks/base/core/java/android/app/SystemServiceRegistry.java】

    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```
其中用到的SYSTEM_SERVICE_FETCHERS是SystemServiceRegistry类提供的一张静态哈希表：

```
【frameworks/base/core/java/android/app/SystemServiceRegistry.java】

final class SystemServiceRegistry {
    . . . . . .
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
    private static int sServiceCacheSize;
```

可以看到，这张哈希表的value部分必须实现ServiceFetcher<?>接口，对SystemServiceRegistry而言，应该是个CachedServiceFetcher<T>派生类对象。
CachedServiceFetcher类已经默认实现了getService()函数：

```
【frameworks/base/core/java/android/app/SystemServiceRegistry.java】

    public final T getService(ContextImpl ctx) {
        final Object[] cache = ctx.mServiceCache; // 终于看到ContextImpl的mServiceCache了
        synchronized (cache) {
            Object service = cache[mCacheIndex];
            if (service == null) {
                service = createService(ctx); // 如果cache里没有，则调用createService()
                cache[mCacheIndex] = service;
            }
            return (T)service;
        }
    }
```

简单地说，就是每当getService()时，会优先从ContextImpl的mServiceCache缓存数组中获取，如果缓存里没有，才会进一步createService()，并记入缓存。
而每个不同的CachedServiceFetcher<T>派生类都会实现自己独有的createService()函数，这样就能在缓存里创建多姿多彩的“系统服务访问类”了。

SYSTEM_SERVICE_FETCHERS哈希表会在SystemServiceRegistry类的静态块中初始化，代码截选如下：

```
    . . . . . .
    static {
        registerService(Context.ACCESSIBILITY_SERVICE, AccessibilityManager.class,
                new CachedServiceFetcher<AccessibilityManager>() {
            @Override
            public AccessibilityManager createService(ContextImpl ctx) {
                return AccessibilityManager.getInstance(ctx);
            }});
        . . . . . .
        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), 
                       ctx.mMainThread.getHandler());
            }});
```

这里所谓的registerService()动作，其实主要就是向静态哈希表里插入新创建的CachedServiceFetcher对象：

```
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
```

现在，我们画一张示意图，以一个Activity的ContextImpl为例：

![ContextImpl]()

























[认识一下Android里的Context](https://my.oschina.net/youranhongcha/blog/1807189)

