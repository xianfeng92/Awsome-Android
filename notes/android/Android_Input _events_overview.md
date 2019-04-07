# Input events overview

 在Android上, 有多种方法可以拦截用户与应用程序交互的事件(Event), 当考虑我们自己界面中的事件时, 可以从用户与之交互的特定视图(View)对象中捕获事件。视图(View)类提供了相应的方法。
 在组成布局(Layout)的各种视图类中, 我们需要注意几个对拦截 UI 事件可能有用的几个公共回调方法。当相应的操作在该对象上发生时, Android框架将回调这些方法。例如, 当我们 Touch 一个视图（如 Button ）时, 会在该视图对象上调用 onTouchEvent() 方法. 如果我们想要拦截该 Touch 事件, 可以通过继承该 Button 并重写其 onTouchEvent() 方法. 但是,我们不可能为了拦截事件去继承所有 View ,这是不切实际的(特别对于Java这种单继承的特性来说). 这就是为什么视图(View)类还包含一组嵌套(nested)接口,当相应的事件(event)系统就会去回调接口.

 我们可以看看 View 的 ListenerInfo 中定义的一些接口:

```
 static class ListenerInfo {
        /**
         * Listener used to dispatch focus change events.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        protected OnFocusChangeListener mOnFocusChangeListener;

        /**
         * Listeners for layout change events.
         */
        private ArrayList<OnLayoutChangeListener> mOnLayoutChangeListeners;

        protected OnScrollChangeListener mOnScrollChangeListener;

        /**
         * Listeners for attach events.
         */
        private CopyOnWriteArrayList<OnAttachStateChangeListener> mOnAttachStateChangeListeners;

        /**
         * Listener used to dispatch click events.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        public OnClickListener mOnClickListener;

        /**
         * Listener used to dispatch long click events.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        protected OnLongClickListener mOnLongClickListener;

        /**
         * Listener used to dispatch context click events. This field should be made private, so it
         * is hidden from the SDK.
         * {@hide}
         */
        protected OnContextClickListener mOnContextClickListener;

        /**
         * Listener used to build the context menu.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        protected OnCreateContextMenuListener mOnCreateContextMenuListener;

        private OnKeyListener mOnKeyListener;

        private OnTouchListener mOnTouchListener;

        private OnHoverListener mOnHoverListener;

        private OnGenericMotionListener mOnGenericMotionListener;

        private OnDragListener mOnDragListener;

        private OnSystemUiVisibilityChangeListener mOnSystemUiVisibilityChangeListener;

        OnApplyWindowInsetsListener mOnApplyWindowInsetsListener;

        OnCapturedPointerListener mOnCapturedPointerListener;

        private ArrayList<OnUnhandledKeyEventListener> mUnhandledKeyListeners;
    }
```

这些接口也可以称为事件监听器, 用来捕获用户与 UI 交互的凭证(ticket)。此时如果我们想要拦截或者监听 Button 的点击(click)事件时, 只需要简单的为 Button 的 mOnClickListener 设置一个回调对象(OnClickListener). 当点击事件发生时, OnClickListener 的 onClick 方法就会被回调.

大多数情况下, 我们都只需要使用事件监听器来监听用户交互事件. 当自定义 View 的时候, you'll be able to define the default event behaviors for your class using the class event handlers.

## Event listeners

An event listener is an interface in the View class that contains a single callback method,即事件监听器是View中的包含一个回调方法的接口. 当用户与 UI 进行交互时, Android framework 会对相应的方法进行回调.

事件侦听器接口中包括以下回调方法:

* onClick

当用户触摸一个 View 或使用导航键(navigation-keys)或轨迹球(trackball)聚焦到该 View 并按下适当的“回车”键或按下轨迹球时, 会回调此方法。

* onLongClick

当用户触摸并按住 View 或使用导航键或轨迹球聚焦该 View 并按住适当的“回车”键或按下并按住轨迹球（一秒钟）时, 会回调此方法。

* onFocusChange

当用户使用导航键或轨迹球导航到或离开该 View 时, 会回调此函数。

* onKey

当 View 获取到焦点时,用户按下或释放设备上的hardKey时, 会回调此函数。


* onTouch

当用户执行符合触摸事件条件的操作时,会回调此函数. 包括在 View 边界内按下、释放或屏幕上的任何移动手势.	

每个事件监听器(Listener)中都只有一个对应的回调方法. 如果我们想要监听一个事件,只需用 Activity 实现相关的 Listener 接口或者定义一个 Listener 的匿名内部类, 然后将 Listener 的实现类传入想要监听的 View 中即可.


下面的示例演示如何注册按钮的点击事件监听器:

```
// Create an anonymous implementation of OnClickListener
private OnClickListener corkyListener = new OnClickListener() {
    public void onClick(View v) {
      // do something when the button is clicked
    }
};

protected void onCreate(Bundle savedValues) {
    ...
    // Capture our button from layout
    Button button = (Button)findViewById(R.id.corky);
    // Register the onClick listener with the implementation above
    button.setOnClickListener(corkyListener);
    ...
}
```

将 onclickListener 作为 Activity 的一部分来实现更为方便, 这将避免额外的类加载和对象分配。例如:

```
public class ExampleActivity extends Activity implements OnClickListener {
    protected void onCreate(Bundle savedValues) {
        ...
        Button button = (Button)findViewById(R.id.corky);
        button.setOnClickListener(this);
    }

    // Implement the OnClickListener callback
    public void onClick(View v) {
      // do something when the button is clicked
    }
    ...
}
```


上面示例中的 onclick（）回调没有返回值, 但其它的一些事件监听器方法必须返回布尔值:

* onLongClick

This returns a boolean to indicate whether you have consumed the event and it should not be carried further. That is, return true to indicate that you have handled the event and it should stop here; return false if you have not handled it and/or the event should continue to any other on-click listeners.

* onKey

This returns a boolean to indicate whether you have consumed the event and it should not be carried further. That is, return true to indicate that you have handled the event and it should stop here; return false if you have not handled it and/or the event should continue to any other on-key listeners.

* onTouch

This returns a boolean to indicate whether your listener consumes this event. The important thing is that this event can have multiple actions that follow each other. So, if you return false when the down action event is received, you indicate that you have not consumed the event and are also not interested in subsequent actions from this event. Thus, you will not be called for any other actions within the event, such as a finger gesture, or the eventual up action event.


__Remember that hardware key events are always delivered to the View currently in focus__, hardware 事件总是传递给当前获取焦点的View. 它们总是从视图层次结构(View hierarchy)的顶部向下分发(dispatch), 直到到达目的地为止。如果视图（或视图的子级）当前具有焦点, 则可以通过 DispatchKeyEvent（）
方法查看事件的传播。除此之外, 我们还可以在 Activity 中使用 onkeydown（）和 onkeyup（）接收所有事件。

Note: __Android will call event handlers first and then the appropriate default handlers from the class definition second__. As such, returning true from these event listeners will stop the propagation of the event to other event listeners and will also block the callback to the default event handler in the View. So be certain that you want to terminate the event when you return true.



## Event handlers

如果我们是自定义 View 组件, 可以定义几个用作默认事件处理程序的回调方法:

* onKeyDown(int, KeyEvent) - Called when a new key event occurs.

* onKeyUp(int, KeyEvent) - Called when a key up event occurs.

* onTrackballEvent(MotionEvent) - Called when a trackball motion event occurs.

* onTouchEvent(MotionEvent) - Called when a touch screen motion event occurs.

* onFocusChanged(boolean, int, Rect) - Called when the view gains or loses focus.

除此之外,还有一些不是 View 类中的方法也值得我们注意, 这些方法会直接影响我们对事件的相关处理.因此,在管理布局内更复杂的事件时,请考虑以下其他方法:


* Activity.dispatchTouchEvent(MotionEvent)

  大多数情况下,该方法都是返回 false. 如果返回 true, 代表我们的 Activity 将拦截掉所有的 Touch事件,此时 Touch 事件也就不会传递到 Window 中,自然也不会向我们的布局中传递了. Game Over!

* ViewGroup.onInterceptTouchEvent(MotionEvent)

  该方法允许视图组(ViewGroup)在事件分派到子视图时监视事件, 即传递到 ViewGroup 中的子View 的所有事件都是会事先经过 ViewGroup 的 onInterceptTouchEvent 方法的. 如果View Group 不想将一个事件传递到其子View,直接在 onInterceptTouchEvent 返回true即可,Game Over!

* ViewParent.requestDisallowInterceptTouchEvent(boolean)
  
  调用父亲 view 的requestDisallowInterceptTouchEvent 用于告诉父 View 它不应该在 onInterceptTouchEvent 拦截 Touch 事件.


## Touch mode

当用户使用方向键或轨迹球导航到用户界面时, 有必要将焦点放在可操作项（如按钮)上。但是,如果设备具有触摸功能, 并且用户通过触摸它开始与界面交互,
那么就不再需要突出显示项目或将焦点放在特定视图上。因此,有一种称为“触摸模式”(touch mode)的交互模式。对于触摸功能的设备,一旦用户触摸屏幕,设备将进入触摸模式。
__只有 isFocusableInTouchMode() 为真时才是获取到焦点的, 如文本编辑器__。其他可触摸的视图,如: Button在触摸时不会获得焦点,它们只会在按下(press)时回调 onClick Listener。

每当用户点击方向键或使用轨迹球滚动时, 设备将退出触摸模式, 并找到一个视图以获得焦点。此时, 用户可以在不触摸屏幕的情况下恢复与用户界面的交互。
触摸模式状态在整个系统中保持（所有窗口和活动）. 要想要查询当前状态,可以调用 isInTouchMode（）查看设备当前是否处于触摸模式。


## Handling focus

Android framework 会根据用户输入处理常规焦点移动,包括在视图被删除或隐藏或新视图可用时更改焦点。View 通过isFocusable() 方法来表明它们是否愿意获取焦点. 想要更改视图是否可以获得焦点. 可以调用 setFocusable（）。在触摸模式下,通过 isFocusableIntouchmode（）查询视图是否允许焦点。使用setFocusableIntouchmode（）
更改此设置。

Note: 在运行Android 9（API级别28）或更高版本的设备上, Activity 不会分配初始焦点。如果需要, 必须显式地请求初始焦点。

焦点移动规则是基于在给定方向上找到 nearest neighbor。在极少数情况下,默认规则可能与开发人员的预期行为不匹配。此时,可以在布局文件中使用以下XML属性提供显式重写:nextfocusdown、nextfocusleft、nextfocusright和nextfocusup。将这些属性之一添加到焦点离开的视图中。将属性的值定义为应给予焦点的视图的ID。例如:

```
<LinearLayout
    android:orientation="vertical"
    ... >
  <Button android:id="@+id/top"
          android:nextFocusUp="@+id/bottom"
          ... />
  <Button android:id="@+id/bottom"
          android:nextFocusDown="@+id/top"
          ... />
</LinearLayout>
```


Ordinarily, in this vertical layout, navigating up from the first Button would not go anywhere, nor would navigating down from the second Button. Now that the top Button has defined the bottom one as the nextFocusUp (and vice versa), the navigation focus will cycle from top-to-bottom and bottom-to-top.

If you'd like to declare a View as focusable in your UI (when it is traditionally not), add the android:focusable XML attribute to the View, in your layout declaration. Set the value true. You can also declare a View as focusable while in Touch Mode with android:focusableInTouchMode.

To request a particular View to take focus, call requestFocus().

To listen for focus events (be notified when a View receives or loses focus), use onFocusChange(), as discussed in the Event listeners section.


[ui-events](https://developer.android.google.cn/guide/topics/ui/ui-events)