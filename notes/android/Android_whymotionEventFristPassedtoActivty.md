## 事件如何传递到Activity中

   当一个点击操作发生的时候，事件最先传递给 Activity，由 Activity 的 dispatchTouchEvent 来进行事件的派发。

   那么究竟事件的来源是在哪里呢？

我们知道 Activity 的 dispatchTouchEvent(MotionEvent ev) 是会接受到事件的，所以我们在该方法中调用Thread.dumpStack()来查看调用栈。

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Thread.dumpStack();
        return super.dispatchTouchEvent(ev);
    }
```

运行程序，输出结果为：

```
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err: java.lang.Exception: Stack trace
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at java.lang.Thread.dumpStack(Thread.java:1348)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at com.example.demo_window.MainActivity.dispatchTouchEvent(MainActivity.java:82)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at com.android.internal.policy.DecorView.dispatchTouchEvent(DecorView.java:398)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.View.dispatchPointerEvent(View.java:12752)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:5106)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:4909)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4426)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4479)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4445)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:4585)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4453)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:4642)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4426)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4479)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4445)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4453)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4426)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:7092)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:7061)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:7022)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:7195)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:186)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.os.MessageQueue.nativePollOnce(Native Method)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.os.MessageQueue.next(MessageQueue.java:326)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.os.Looper.loop(Looper.java:160)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.app.ActivityThread.main(ActivityThread.java:6669)
2019-03-22 17:38:23.785 14567-14567/com.example.demo_window W/System.err:     at java.lang.reflect.Method.invoke(Native Method)
2019-03-22 17:38:23.785 14567-14567/com.example.demo_window W/System.err:     at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
2019-03-22 17:38:23.785 14567-14567/com.example.demo_window W/System.err:     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```


## 具体分析

```
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:186)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.os.MessageQueue.nativePollOnce(Native Method)
2019-03-22 17:38:23.784 14567-14567/com.example.demo_window W/System.err:     at android.os.MessageQueue.next(MessageQueue.java:326)
```

在 Android 的主线程消息循环机制，通过将消息封装到 Message 中，再通过 sendMessage 将其存储到 MessageQueue 中，通过 Looper不断调用 MessageQueue 的 next()方法进行消息的处理。

1. 首先，当我们触摸屏幕时，通过Android消息机制，从Looper从MessageQueue中取出该事件，发送给WindowInputEventReceiver

2. WindowInputEventReceiver是ViewRootImpl的内部类，通过enqueueInputEvent方法，将输入事件加入输入事件队列中，并进行处理和转发

3. ViewPostImeInputStage收到输入事件，将事件传递给DecorView的dispatchPointerEvent()方法（是View的方法）

4. dispatchPointerEvent()方法通过DecorView中的dispatchTouchEvent()方法，调用了Activity的dispatchTouchEvent()方法






























































