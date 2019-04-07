# 理解Window和WindowManager

Window表示的是一个窗口的概念，在日常生活中使用的并不是很多，但是某些特殊的需求还是需要的，比如悬浮窗之类的，它的具体实现是 PhoneWindow,创建一个 Window 很简单，
只需要 WindowManager 去实现，WindowManager 是外界访问 Window 的入口，Window 的具体实现是在 WindowManagerService 中，它们两个的交互是一个IPC的过程，Android 中的所有
视图都是通过 Window 来呈现的，无论是Activity,Dialog还是Toast,它们的视图都是直接附加在Window上的，因此 Window 是 View 的直接管理者，View 的事件是通过 Window 传递给 DecorView，
然后 DecorView 传递给我们的 View，就连 Activity 的 setContentView,都是由 Window 传递的。

### Window和WindowManager

为了了解 Window 的工作机制，我们首先来看下如何通过 WindowManager 来添加一个 Window：

```
            Button button = new Button(this);
            button.setText("I am Window");
            WindowManager windowManager = (WindowManager)getSystemService(WINDOW_SERVICE);
            WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(WindowManager.LayoutParams.WRAP_CONTENT
                    ,WindowManager.LayoutParams.WRAP_CONTENT,0,0, PixelFormat.TRANSPARENT);

            layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;

            layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;
            layoutParams.x = 200;
            layoutParams.y = 300;
            windowManager.addView(button,layoutParams);
```


上述的代码，其中type和flag是比较重要的，我们来看下：

Flag 参数表示 window 的属性，它有很多选项，通过这些选项可以控制 Window 的显示特性，这里介绍几个比较常用的选项：

* FLAG_NOT_FOCUSABLE

表示 Window 不需要获取焦点，也不需要接收各种事件，这属性会同时启动FLAG_NOT_TOUCH_MODAL，最终的事件会传递给下层的具体焦点的window

* FLAG_NOT_TOUCH_MODAL

在此模式下，系统会将当前 window 区域以外的单击事件传递给底层的 Window，此前的 Window 区域以内的单机事件自己处理，这个标记很重要，
一般来说都需要开启，否则其他 window 将无法获取单击事件

* FLAG_SHOW_WHEN_LOCKED

开启这个属性可以让window显示在锁屏上


Type 参数表示 window 的类型，window 有三种类型，分别是应用Window，子Window，系统Window。应用 window 对应一个 Activity,子 Window 不能单独存在，需要依赖一个父Window，
比如常见的 Dialog 都是子 Window,系统 window 需要声明权限，比如系统的状态栏。


Window 是分层的，每个 Window 对应着 z-ordered,层级大的会覆盖在层级小的Window上面，这和HTML中的z-index的概念是一致的，在这三类中，
应用是层级范围是1-99，子window的层级是1000-1999，系统的层级是2000-2999。这些范围对应着type参数，如果想要window在最顶层，那么层级范围设置大一点就好了，
很显然系统的值要大一些，系统的值很多，我们一般会选择 TYPE_SYSTEM_OVERLAY 和 TYPE_SYSTEM_ERROR，记得要设置权限。


WindowManager 所提供的功能很简单，常用的有三个方法，添加 View,更新 View,删除 View,这三个方法定义在ViewManager中，而 WindowManager继承自ViewManager：

```
   public interface ViewManager {
        public void addView(View view, ViewGroup.LayoutParams params);

        public void updateViewLayout(View view, ViewGroup.LayoutParams params);

        public void removeView(View view);
    }
```


对于开发者来说，WindowManager 常用的就只有这三个功能而已，但这三个功能已经足够我们使用了。它可以创建一个　Window　并向其添加　View。还可以更新和删除Window
中的View。我们常见的那种可以拖动的　Window效果，其实很好实现，只需要根据手指的位置来设定 LayoutParams 中的 x 和 y 的值即可改变 Window 的位置。

```
 button.setOnTouchListener(new View.OnTouchListener() {
                @Override
                public boolean onTouch(View v, MotionEvent event) {
                    int rawX = (int) event.getRawX();
                    int rawY = (int) event.getRawY();
                    switch (event.getAction()){
                        case MotionEvent.ACTION_MOVE:
                            layoutParams.x = rawX;
                            layoutParams.y = rawY;
                            windowManager.updateViewLayout(button,layoutParams);
                            break;
                            default:
                                break;
                    }
                    return false;
                }
```



## Window的内部机制

Window 是一个抽象的概念，每一个 Window 对应着一个 View 和一个 ViewRootImpl, Window 和 View 通过 ViewRootImpl 建立关系。因此 Window 并不是实际存在的，
它是以 View 的形式存在的。这点从WindowManager定义也可以看出，它提供的三个接口方法 addView，UpdateViewLayout以及removeView 都是针对View，这说明 View 才是window的实体。
在实际使用当中我们并不能直接访问 Window，对 WIndow的访问必须通过 WindowManager。


### Window 的添加过程


Window 的添加过程是通过 WindowManager 的 addView 去实现，WindowManager 是一个接口，它的真正实现是 WindowManagerImpl 类。在 WindowManagerImpl 中 Window的三大操作如下：

```
  @Override
    public void addView(View view, ViewGroup.LayoutParams params) {
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }

    @Override
    public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        mGlobal.updateViewLayout(view, params);
    }

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
```

可以发现，WindowManagerImpl 并没有直接去实现一个 Window 的三大操作，而是全部交给了 WindowManagerGlobal 来处理，WindowManagerGlobal 以工厂的形式向外提供自己的实例，
在WindowManagerGlobal中有一段如下的代码：

```
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

```

WindowManagerImpl 这种工作模式就是典型的桥接模式，将所有的操作全部委托给 WindowManagerGlobal 去实现，WindowManagerGlobal 的addView 方法主要分如下几步：


1. 检查参数是否合法，如果是子Window还需要调整一下参数

```
       if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        }
```

2. 创建ViewRootImpl并将View添加到列表中

在WindowManagerGlobal有如下几个列表是比较重要的：

```
   private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

在上面的声明中，mViews 存储所有 window 所对应的View，mRoots 存储是所有 window 所对应的 ViewRootImpl，mParams存储是所对应的布局参数 
，而 mDyingViews 则存储那些正在被删除的对象，在 addView 中通过如下方式将 Window 的一系列对象添加到列表中。


```
   root = new ViewRootImpl(view.getContext(), display);
    view.setLayoutParams(wparams);
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
```

3. 通过 ViewRootImpl 来更新界面并完成 Window 的添加

这个步骤由 ViewRootImpl 的 setView 完成，它内部会通过 requstLayout 来完成异步刷新请求，在下面代码中，scheduleTraversals 实际上就是View绘制的入口:

```
  @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

接下来会通过 WindowSession 最终来完成Window的添加过程，WindowSession 的类型 IWindowSession，它是一个Binder对象，真正的实现类是 Session，
也就是 Window 的添加过程是一次 IPC 调用，最终会交由 WindowManagerService去处理。


## Window的删除过程

Window的删除过程和添加过程一样，都是通过 WindowManagerImpl， 再进一步通过 WindowManagerGlobal来实现了。


## Window的更新过程


# Window的创建过程

通过上面的分析，我们知道，view 是 Android 中视图的呈现方式，但是 view 不能单独存在，它必须依附在 window 这个抽象类中，因此有视图的地方就有 window，Android
中提供视图的地方有 Activity，Dialog，Toast，除此之外，还有一些依托 Window 而实现的视图，比如 PopUpWindow，菜单，它们也是视图，有视图的地方就有Window。


## Activity的Window创建过程

要分析 Activity 的 Window 创建过程就需要去了解activity的启动过程，这里简单概括，activity 的启动最终会由 ActivityThread 中的 perfromLaunchActivity() 来完成
整个启动过程，这个方法内部会通过类加载器创建 Activity 的实例对象，并调用 attach 方法为其关联运行过程中所依赖的一系列上下文环境变量。代码如下：


```
 if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.voiceInteractor);
```

在 Activity 的 attach 方法中，系统会创建 Activity 所属的 Window 对象并为其设置回调接口，Window 对象的创建过程是由 PolicyManager 的 makeNewWindow方法实现的，
由于 Activity 实现了 Window 的 callback 方法接口，因此__当 Window 接受到外界的状态改变的时候就会去调用 Activity 的方法__，callback接口中的方法很多，
但是有几个确实我们非常熟悉的，如 onAttachedToWindow、onDetachFromWindow、dispatchTouchEvent，等等。


当 window 创建完成后，下面分析 Activity 的视图是怎么依附在 Window 上。由于 Activity 的视图是由 setContentView 开始的，所有我们先看下这个方法：

```
    public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

从 Activity 的 setContentView 的实现可以看出，Activity 将具体实现交给了 Window处理，而 Window 的具体实现是 PhoneWindow，所以只要看看 PhoneWindow 的相关逻辑即可。
PhoneWindow 的setContentView 方法大致遵循如下几个步骤：

1. 如果没有 DecorView 就去创建它

  DecorView 是一个 FrameLayout。DecorView是 Activity 中的顶级 View,一般来说它的内部包含标题栏和内部栏，但是这个会随着主题的变化而发生改变的，不管怎么样，内容是一定要存在的，
  并且内容有固定的id，那就是 content,完整的就是android.R.id.content，DecorView 的创建是由 installDecor 方法来完成的，在方法内部会通过 generateDecor 方法来完成创建 DecorView,
  这个时候 DecorView 还只是一个空白的FrameLayout：

```
protected DecorView generateDecor(){
    return new DecorView(getContext(),-1);
}
```

为了初始化 DecorView 的结构，PhoneWindow 还需要通过 generateLayout 方法来加载具体的布局文件到 DecorView 中，这个跟主题有关：

```
View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
```
其中 ID_ANDROID_CONTENT 的定义如下，这个id对应的就是 ViewGroup 的 mContentParent：

```
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```

2. 将 View 添加到 DecorView 的 mContentParent 中

这个过程比较简单，由于在第一步的时候已经初始化了 DecorView，因此这一步就直接将 Activity 的视图添加到 DecorView 的 mContentParent 中既可，mLayoutInflater.inflate(layoutResID, mContentParent)。
到此为止，Activity 的布局文件已经添加到 DecorView 中，由此可以理解 Activity 的 setContentView 的来历了，因为 Activity 的布局文件只是添加到了 DecorView 的 mContentParent 中，
因此叫 setContentView 更加准确。


3. 回调Activity的 onContentChanged 方法来通知Activity视图已经发生改变


经过了上面的三个步骤，到这里为止 DecorView 已经被创建并且初始化完毕了，Activity 的布局文件也已经添加到了 DecorView 的 mContentParent 中，但是这个时候 DecorView 还没有被 windowmanager 
添加到 window 中。这里需要正确的理解 window 的概念，window 更多的是表示一种抽象的功能集合，虽然说早在 Activity 的 attch 中 window 就已经被创建了，但是这个时候由于 DecorView 
还没有被 windowmanager 识别，所有还不能提供具体的功能，因为它还无法接收外界的输入信息。在 ActivityThread 的 handleResumeActivity 方法中，首先会调用 Activity 的 onResume 方法，
接着会调用 Activity 的 makeVisible, 正是在 MakeVisible 方法中，DecorView 真正地完成了添加和显示这两个过程，到这里 Activity 视图才能被用户看到，如下所示：

```
   void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```

到这里，window的创建过程就已经分析完了


[Android开发艺术探索]

