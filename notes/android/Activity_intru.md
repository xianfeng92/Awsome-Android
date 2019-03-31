## What is Activity ?

Activity 是用户可以做的单一，专注的事情。几乎所有 Activity 都与用户交互，因此 Activity 类负责创建一个窗口，在其中可以使用setContentView（View）放置UI。
Activity 通常以全屏窗口的形式呈现给用户，它也可以以其他方式使用：作为浮动窗口（通过具有R.attr.windowIsFloating集的主题）或嵌入到另一个活动内部（使用ActivityGroup）。

有两种方法几乎所有Activity的子类都会实现：

* onCreate（Bundle）是初始化 Activity 的地方。最重要的是，在这里通常会调用 setContentView（int）来定义UI的布局资源，并使用findViewById（int）检索该UI中需要以
  编程方式(programmatically)进行交互的 widgets

* onPause（）是处理用户离开活动时（Activity不可见）地方。此时应该提交用户所做的任何更改（通常是提交数据的 ContentProvider）


所有 Activity 类必须在其包的 AndroidManifest.xml 中具有相应的<activity>声明，才能使用 Context.startActivity() 来启动一个 Activity。

PS： 当一个 Activity 没有在 AndroidManifest.xml 中声明时，Context 是不认识这个 Activity 的，自然无法对其进行相关操作。


## Activity Lifecycle

系统是使用 Activity Stack来对 Activity进行管理。当一个新 Activity 开始时，它会被系统置于 Activity Stack 顶部并成为当前运行且可见的 Activity。

Activity 主要有四种状态：

* 如果 Activity 正在屏幕上显示（位于 Activity Stack 顶部），则它处于 active or running 状态

* 如果 Activity 失去焦点但仍然可见（有一个新的透明的或者非全屏的 Activity 位于该 Activity 之上），此时该 Activity 处于 Pause 状态。处于 Pause 状态的 Activity 
  也是存活着的（它仍然维护着所有状态和成员信息并仍然 attach 窗口管理器），但其在极低内存情况下会被系统 kill。

* 如果 Activty 被另一个 Activity 完全遮挡住，则该 Activity 会处于 Stop 状态。此时它仍然保留所有状态和成员信息，但是，它不再对用户可见，因此其窗口被隐藏，
  并且当其它地方需要内存时，它通常会被系统 kill。

* 如果一个 Activity 处于 Pause 或者 Stop 状态，系统可以通过 finish 该 Activity 或 kill 该 Activity所在的进程来将 Activity 从内存中移除。当它再次显示给用户时，
  必须重新启动（restarted）并恢复到（restored）之前的状态。

下图显示了 Activity 的重要状态路径。方形矩形表示当 Activity 在状态之间转移时会回调的方法。彩色椭圆是 Activity 所处的状态。

![activity_lifecycle.png](https://github.com/xianfeng92/android-code-read/blob/master/images/activity_lifecycle.png)


There are three key loops you may be interested in monitoring within your activity:


* Activity 的整个生命周期（entire lifetime ）是从调用 onCreate(Bundle) 开始，到 onDestroy() 结束。在 onCreate(Bundle) 做一些全局（global）状态的设置，
  在 onDestroy() 释放所有的剩余资源。例如，如果 Activity 有一个在后台运行的线程从网络下载数据，它可以在 onCreate（）中创建该线程，然后在onDestroy（）中停止该线程。

* Activity的可见生命周期(The visible lifetime ) 是从 onStart（）的调用开始，直到对 onStop（）调用结束。在此期间，用户可以在屏幕上看到该 Activity，尽管它可能不在前台
  和用户交互。在这两个方法之间，我们可以维护向用户显示活动所需的资源。例如，可以在onStart（）中注册BroadcastReceiver以监视影响 UI 的更改，并在 onStop（）中取消注册。
  当 Activity 对在可见和不可见状态切换时，onStart（）和onStop（）方法会被多次调用。

* Activity 的前台生命周期（The foreground lifetime）是从调用 onResume（）方法开始，直到调用 onPause（）方法结束。在此期间，Activity 位于所有其他 Activity 之上并与用户交互。
  Activity 可以经常在Resume 和 Stop 状态之间切换。例如，当设备进入休眠状态，当有新的 intent 传递来时，因此这些方法中的代码应当尽量轻量级。


Activity 的整个生命周期由以下方法定义。可以将这些方法看成 hooks，通过重写这些方法可以在 Activity 状态变化时执行相应的工作。如：在 onCreate（Bundle）方法中进行一些初始设置，
在 onPause（）方法中提交用户对数据的更改，并准备停止与用户交互。在实现这些方法时，应该始终调用超类的相关实现。

```
 public class Activity extends ApplicationContext {
     protected void onCreate(Bundle savedInstanceState);

     protected void onStart();

     protected void onRestart();

     protected void onResume();

     protected void onPause();

     protected void onStop();

     protected void onDestroy();
 }
 
```

In general the movement through an activity's lifecycle looks like this:



* onCreate()
  
  在第一次创建 Activity 时调用。在该方法中主要进行一些常规静态设置（static set up）：创建视图，将数据绑定到列表等。
  此方法还可以提供 Activity 先前冻结状态的 Bundle。

* onRestart()

  在 Activity 被 Stop 后再次 start 之前调用

* onStart()

  当 Activity 变得对用户可见时调用。如果活动到达前台，则接着回调 onResume（）；如果隐藏，则接着回调onStop（）。


* onResume()

  当 Activity 开始可以和用户交互时回调，此时该 Activity 处于 Activity Stack 栈顶

* onPause()

  当系统即将开始恢复先前的 Activity 时回调该方法，该方法中通常会做一些用户相关数据保存，停止动画以及可能消耗CPU等事情。
  此方法的实现必须非常快，因为在此方法返回之前，不会 resume 下一个 Activity。

* onStop()

  当 Activity 不再对用户可见时调用，因为另一个 Activity 已 Resume 并且正在覆盖此 Activity。如果此 Activity 返回与用户交互，则执行onRestart（），
  如果此 Activity 消失，则执行onDestroy（）。


* onDestroy()

  这是 Activity 被销毁之前的最后一个回调。该方法的调用可能是因为 Activity 被主动 finish，或者因为系统暂时销毁此 Activity 实例以节省空间。
  可以使用Activity＃isFinishing方法区分这两种情况。


## Configuration Changes

如果设备的配置（由Configuration类定义）发生更改，则显示用户界面的任何内容都需要更新以匹配该配置。由于 Activity 是与用户交互的主要机制，
因此它包含对处理配置更改的特殊支持。当设备的配置发生更改（例如屏幕方向，语言，输入设备等的更改）将导致当前 Activity 被 Destroy 如果此 Activity 
位于前台或对用户可见，则在该实例中调用onDestroy（）后，将创建 Activity 的新实例，并通过上一个实例 Destroy前 onSaveInstanceState（Bundle）生成的
savedInstanceState，来恢复 Activity 销毁前的一些状态。

这样做是因为任何应用程序资源（包括布局文件）都可以根据配置值进行更改。因此，处理配置更改的唯一安全方法是重新检索所有资源，包括布局，drawable和字符串。
因为 Activity 已经知道如何保存其状态并从该状态重新创建自己，所以这是使用新配置重新启动活动的便捷方式。

在某些特殊情况下，在清单中配置 Activity 的android：configChanges属性可以绕过一些配置的更改所需要的 Activity重启。For any types of configuration changes 
you say that you handle there, you will receive a call to your current activity's onConfigurationChanged(Configuration) method instead of being restarted. 
If a configuration change involves any that you do not handle, however, the activity will still be restarted and onConfigurationChanged(Configuration) 
will not be called.


## Saving Persistent State

Activity 通常会处理两种持久状态：共享文档类数据（通常使用 content provider 存储在SQLite数据库中）和内部状态（如用户首选项）。对于 content provider 数据，我们建议Activity
使用“就地编辑”用户模型（edit in place）。 也就是说，用户进行的任何编辑都可以立即有效地进行，而无需额外的确认步骤。支持此模型通常只需遵循以下两条规则：

1. 创建新文档时，会立即创建后备数据库条目或文件。例如，如果用户选择编写新电子邮件，则会在他们开始输入数据时立即创建该电子邮件的新条目，
   这样，如果他们此时跳转到任何其他 Activity，则此电子邮件现在将显示在草稿中。

2. 当调用活动的onPause（）方法时，应该将用户所做的任何更改通过 content provider 来保存。这可确保任何其他即将运行的 Activity 都可以看到这些更改。
   当然我们也可以在 Activity 生命周期的关键时刻做一些数据的保存：例如，在开始新活动之前，完成自己的 Activity 之前，用户在输入字段之间切换时。

此模型旨在防止用户在 Activity 之间导航时数据丢失，并允许系统在暂停（onPause）后的任何时间安全地终止 Activity（因为其他地方需要系统资源）。
值得注意的是，__当用户从当前 Activity 中按 BACK 并不意味着“取消” ,这意味着将活动与其当前内容保存在一起__。取消 Activity 中的编辑必须通过
其他一些机制提供，例如显式的“恢复”或“撤消”选项。

Activity 类还提供用于管理与 Activity 关联的内部持久状态的API。例如，可用其记住用户在日历（日视图或周视图）中的首选初始显示或用户在Web浏览器中的默认主页。

使用 getPreferences（int）管理 Activity 持久状态，允许检索和修改与 Activity 关联的一组 name/value pairs。要使用跨多个应用程序组件（活动，接收器，服务，提供程序）共享的首选项，
可以使用基础Context＃getSharedPreferences方法检索以特定名称存储的首选项对象。 （请注意，无法跨应用程序包共享设置数据 - 因此您需要内容提供程序。）



[Activity](https://developer.android.com/reference/android/app/Activity)
