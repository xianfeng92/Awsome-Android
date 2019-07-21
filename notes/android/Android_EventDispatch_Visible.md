## View的事件分发

当一个 MotionnEvent 产生了以后, 系统需要把这个事件传递给一个具体的 View，而这个传递的过程就是分发过程。事件从 Activity到 ViewGroup,再从 ViewGroup 分发到具体的View.

先简单看看下面几个结论,稍后用 AndroidStudioProfiler 来验证:

1. 一个应用 View 树的实际的根节点是 PhoneWindow$DecorView。而 DecorView 内的 invalidate() 或 requestLayout() 等操作是通过 ViewRootImpl 来开始的。ViewRootImpl 内有一个叫 mView 的成员变量，这个变量就是 DecorView。

2. ViewRootImpl 中最重要的方法是 performTraversals(), 在这个方法里面执行整个 View 树的 measure、layout 和 draw 操作。

3. 在实现上，ViewRootImpl 监听 Choreographer，Choreographer 监听 VSync。

4. 当某个子 View 调用 invalidate() 或 requestLayout() 方法时，说明视图树需要更新，ViewRootImpl 会监听 Choreographer，而 Choreographer 则请求 VSync 信号。当 VSync 信号到来时，执行 Choreographer 的相关方法，处理 3 种事件：Input、Animation 和 Traversal。 其中，Traversal 也就是 ViewRootImpl#performTraversals(), 然后 View 树就会被更新，进而显示在屏幕上。


下面使用 AndroidStudioProfiler 来分析一下,点击事件发生后是如何传递的：

打开在 [Ganks]() 项目中打开 AndroidStudioProfiler,进入 HomeFragment 画面, 如下图所示:

![initScreenUseAndroidProfier]()

我们在 HomeFragment 画面中, 使用 AndroidStudioProfiler 来 Record 记录在 HomeFragment 的点击和滑动事件,如下图所示:

![clickEventHappen]()

先看看点击事件发生前的 call Chart:

![BeforeClick]()

即,点击事件发生前,主线程的Looper.loop()方法一直在等待着系统的命令(传递来的需要处理的事件) 。这也验证了整个Android系统是基于消息循环的，无论是组件生命周期的回调还是相关的
input事件，都是通过 message 传递个主线程的MessageQueue，然后由主线程的Looper取出并分发相关事件。

当我们点击屏幕时,AMS 会将其传递给主线程的 MessageQueue,此时主线程的Looper就可以从 MessageQueue poll 出该点击事件:

![AfterClick]()

当主线程 Looper 从 MessageQueue 中 poll 出点击事件后,接下来就是事件的分发了。上图中，可以看到首先是 Choreographer 的 FrameDisplayEventReceiver 监听到点击事件，
然后由其交给 ViewRootImpl。

这里值得关注的是，Activity 的 dispatchTouchEvent，是不是很熟悉：

![MethodCalls]()

先看：

Activity -->PhoneWindow(DecoverView)--->ViewGroup---->SwipeRefreshLayout

注意这里的 SwipeRefreshLayout，这是我们当前显示在屏幕上的画面布局文件的根 View：

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.SwipeRefreshLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/refreshLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</android.support.v4.widget.SwipeRefreshLayout>

```

接下来，点击事件从 SwipeRefreshLayout ---> RecyclerView--->LinearLayoutManager。LinearLayoutManager 是管理 RecycleView 的布局以及 view 的回收策略。

到这里，我们知道点击事件最终到了 LinearLayoutManager 中。

在我们点击屏幕16ms后，系统触发一个vSync信号来绘制画面，相关的方法调用如下图所示：

![After16msOfClick]()


在 ViewRootImpl 中调用了 performTraversals()，在这个方法里面执行整个 View 树的 measure、layout 和 draw 操作。





