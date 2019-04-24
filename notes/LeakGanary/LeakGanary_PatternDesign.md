# LeakGananry 中的设计模式分析

当在项目中使用 LeakCanary 时，初始化操作流程如下所示：

```
public class ExampleApplication extends Application {
 
    @Override
    public void onCreate() {
        super.onCreate();
        // 判断当前是否在 AnalyzerProcess
        if (LeakCanary.isInAnalyzerProcess(this)) {
            // This process is dedicated to LeakCanary for heap analysis.
            // You should not init your app in this process.
            return;
        }
        // 初始化 LeakCanary。初始化后，LeakGanary 就可以监测 app 中的 Activity 和 Fragment 的内存泄露情况
        LeakCanary.install(this);
    }
}
```

下面看看 LeakCanary.install(this) 到底做了啥～


```
  // LeakCanary#install
  /**
   * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
   * references (on ICS+).
   */
  public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

refWatcher() 为 LeakCanary 的静态方法，该方法中主要返回一个 AndroidRefWatcherBuilder 对象。
AndroidRefWatcherBuilder 就是用来构建 RefWatcher 对象的。

```
  public static @NonNull AndroidRefWatcherBuilder refWatcher(@NonNull Context context) {
    return new AndroidRefWatcherBuilder(context);
  }
```

在 AndroidRefWatcherBuilder 中，我们可以对 LeakGanary 的一些属性进行配置，如：是否需要自动监测Activity，
是否需要自动监测 Fragment以及是否需要将泄露信息显示出来。

```
  /**
   * Whether we should automatically watch activities when calling {@link #buildAndInstall()}.
   * Default is true.
   */
  // 默认情况 watchActivities 为 true，即需要 LeakGananry 为我们自动监测 Activity
  public @NonNull AndroidRefWatcherBuilder watchActivities(boolean watchActivities) {
    this.watchActivities = watchActivities;
    return this;
  }

  /**
   * Whether we should automatically watch fragments when calling {@link #buildAndInstall()}.
   * Default is true. When true, LeakCanary watches native fragments on Android O+ and support
   * fragments if the leakcanary-support-fragment dependency is in the classpath.
   */
  // 默认情况为 true，即需要 LeakGananry 为我们自动监测 Fragment
  public @NonNull AndroidRefWatcherBuilder watchFragments(boolean watchFragments) {
    this.watchFragments = watchFragments;
    return this;
  }

  /**
   * Sets a custom {@link AbstractAnalysisResultService} to listen to analysis results. This
   * overrides any call to {@link #heapDumpListener(HeapDump.Listener)}.
   */
  // 默认情况为 false，即不需要 LeakGananry 为我们将内存泄漏情况显示出来
  public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }
```

当我们用 AndroidRefWatcherBuilder 配置好了 LeakGananry 的相关属性后，调用 buildAndInstall 即可构建出一个 
RefWatcher 对象。

```
  /**
   * Creates a {@link RefWatcher} instance and makes it available through {@link
   * LeakCanary#installedRefWatcher()}.
   *
   * Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
   *
   * @throws UnsupportedOperationException if called more than once per Android process.
   */
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();// 构建 refWatcher 对象,使用 LeakGananry 提供的默认配置就好
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      if (watchActivities) {
        // 如果需要监测 Activity，只需将 refWatcher 传入 ActivityRefWatcher 中
        // ActivityRefWatcher 负责监听应用程序的 Activity 生命周期
        // 当 onActivityDestroyed 后,ActivityRefWatcher 会调用 refWatcher 来监测内存泄露情况
        ActivityRefWatcher.install(context, refWatcher);
      }
      if (watchFragments) {
        // 如果需要监测 Fragment，只需将 refWatcher 传入 FragmentRefWatcher 中
        // FragmentRefWatcher 负责监听应用程序的 fragment 的生命周期
        // 当 onFragmentViewDestroyed 后，FragmentRefWatcher 会调用 refWatcher 来监测内存泄露情况
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }
```

# 小结

1. LeakCanary 使用了门面模式（Facade），将Activity 和 Fragment 生命周期的监听，内存泄露的分析，分析结果的回调等
   这些小模块组合到了一起，最终只提供了一个 install 接口供使用者去调用。本质：就是化零为整；引入一个中介类，把各个
   分散的功能组合成一个整体，只对外暴露一个统一的接口。

2. 进入 LeakCanary 的门面内，使用 AndroidRefWatcherBuilder （构建者模式）来构建具体的 refWatcher 对象。refWatcher 
   对象的职责很单一，就是监测对象的内存泄露情况。

3. refWatcher 监测哪些对象呢？默认情况下，refWatcher 可以监测应用程序中的 Activity 和 Fragment 对象的内存泄露情况。
   当然，我们也可以用 refWatcher 对象来监测任何一个本该被GC的对象。这里也体现了，refWatcher 职责单一的好处了，哪个
   对象需要监测，就可以监测哪个对象。
























