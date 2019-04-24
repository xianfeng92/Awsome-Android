# FragmentRefWatcher

FragmentRefWatcher 为一个接口，接口中定义了一个 watchFragments(Activity activity) 方法来监测 Fragment 的内存泄露。
FragmentRefWatcher 还有一个内部类 Helper。在 LeakGananry 中监测 Fragment 内存泄露调用的是如下方法：

```
FragmentRefWatcher.Helper.install(context, refWatcher)

```

其中 Helper 为 FragmentRefWatcher 的一个内部类，其 install 方法如下所示：

```
    // FragmentRefWatcher#Helper
    public static void install(Context context, RefWatcher refWatcher) {
      List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();
      // 如果当前 sdk 版本大于26时，实例化 AndroidOFragmentRefWatcher 对象并添加到 fragmentRefWatchers
      if (SDK_INT >= O) {
        fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
      }
      // 尝试通过反射去实例化一个 supportFragment
      try {
        Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
        Constructor<?> constructor =
            fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
        FragmentRefWatcher supportFragmentRefWatcher =
            (FragmentRefWatcher) constructor.newInstance(refWatcher);
        fragmentRefWatchers.add(supportFragmentRefWatcher);
      } catch (Exception ignored) {
      }

      if (fragmentRefWatchers.size() == 0) {
        return;
      }
      // 实例化 Helper 对象
      Helper helper = new Helper(fragmentRefWatchers);

      Application application = (Application) context.getApplicationContext();
      // 监听 Activity 的生命周期
      application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
    }
```

看看 FragmentRefWatcher 是如何监听 Activity 的生命周期：

```
    // helper#activityLifecycleCallbacks
    private final Application.ActivityLifecycleCallbacks activityLifecycleCallbacks =
        new ActivityLifecycleCallbacksAdapter() {
          @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            for (FragmentRefWatcher watcher : fragmentRefWatchers) {
              watcher.watchFragments(activity);
            }
          }
        };

```

由上可知：当 onActivityCreated 时，每个 Fragment 都会去调用其对应的 watchFragments 方法。注意，watchFragments 方法中
传入的 activity。fragment 是依赖于 Activity 而存在的，其生命周期也是受宿主 Activity 所影响。

AndroidOFragmentRefWatcher 对象实现了 FragmentRefWatcher 接口，其 watchFragments 方法如下所示：

```
  @Override public void watchFragments(Activity activity) {
    // 通过 Activity 获取到对应的 fragmentManager
    FragmentManager fragmentManager = activity.getFragmentManager();
   // 在 fragmentManager 上注册监听 fragment 的生命周期回调
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
  }
```

看看 fragmentLifecycleCallbacks 中是如何处理 Activity 的相关回调：

```
  private final FragmentManager.FragmentLifecycleCallbacks fragmentLifecycleCallbacks =
      new FragmentManager.FragmentLifecycleCallbacks() {

        @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
          View view = fragment.getView();
          if (view != null) {
            refWatcher.watch(view); // 监测 fragment 的 view 是否内存泄露
          }
        }

        @Override
        public void onFragmentDestroyed(FragmentManager fm, Fragment fragment) {
          refWatcher.watch(fragment); // 监测 fragment 是否内存泄露
        }
      };
```

# 小结

1. FragmentRefWatcher 为一个接口，该接口的内部类 Helper 中会实例化对应的 FragmentRefWatcher 的实现类，该
   实现类中进行 fragment 的内存泄露监测。

2. 对 fragment 对象的监测会涉及其 view 和 fragment 对象本身的内存泄露























