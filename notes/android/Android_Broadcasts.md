# Broadcasts

Android 应用可以从 Android 系统和其他 Android 应用发送或接收广播消息，类似于发布-订阅设计模式。These broadcasts are sent when an event of interest occurs
例如，Android系统在发生各种系统事件时会发送广播，例如系统启动或设备开始充电时。当然，应用程序也可以发送自定义广播，以通知其他应用程序他们可能感兴趣的内容（例如，已下载了一些新数据）。

应用可以注册以接收特定广播。当发送广播时，系统自动将广播路由到已订阅接收该特定类型广播的应用。__广播可以用作跨应用程序和普通用户流程之外的消息传递系统__。但是，不要滥用响应广播的机会
并在后台运行可能导致系统性能降低的任务。

## About system broadcasts

当发生各种系统事件时，系统会自动发送广播。例如当系统进出飞机模式时,系统广播将发送到订阅接收事件的所有应用程序。广播消息被　Intent　对象所包装，Intent 的 action 字符串标识所发生的事件（例如android.intent.action.AIRPLANE_MODE）。intent 中还会有一些额外字段标识附加的信息。例如，飞行模式 intent 包括一个额外的布尔值，用于代表飞行模式是否打开。

## Changes to system broadcasts

随着Android平台的发展，它会定期更改系统广播的行为方式。如果应用是针对Android 7.0（API级别24）或更高版本，或者如果它安装在运行Android 7.0或更高版本的设备上，请留意以下更改。


### Android 9

从 Android 9（API级别28）开始，NETWORK_STATE_CHANGED_ACTION 广播不会接收有关用户位置或个人身份识别数据的信息。此外，如果应用安装在运行 Android 9或更高版本的设备上，
则来自 Wi-Fi 的系统广播不包含 SSID，BSSID，连接信息或扫描结果。要获取此信息，需调用 getConnectionInfo（）。

### Android 8.0

从 Android 8.0（API级别26）开始，系统对 manifest-declared 的接收器施加了额外的限制。如果应用面向 Android 8.0 或更高版本，则无法使用 manifest 为大多数隐式广播声明 receiver
（broadcasts that don't target your app specifically）。当用户主动使用应用时，可以使用 context-registered receiver。

### Android 7.0

Android 7.0（API级别24）及更高版本不发送以下系统广播：

* ACTION_NEW_PICTURE

* ACTION_NEW_VIDEO

此外，针对 Android 7.0及更高版本的应用必须使用 registerReceiver（BroadcastReceiver，IntentFilter）注册 CONNECTIVITY_ACTION 广播,在 manifest 中声明 receiver 不起作用。


## Receiving broadcasts


应用程序可以通过两种方式接收广播：manifest-declared receivers and context-registered receivers

### Manifest-declared receivers

__在 Manifest 中声明了广播接收器，系统会在发送广播时启动该应用（如果应用尚未运行）__。

Note：如果应用程序的 API 为 26或更高，则不能使用 Manifest 来声明隐式广播的接收者（broadcasts that do not target your app specifically），除了
[一些免于该限制的隐式广播](https://developer.android.com/guide/components/broadcast-exceptions.html)。在大多数情况下，可以使用 scheduled jobs 来替代 。

要在 Manifest 中声明广播接收器，请执行以下步骤：


1. 在应用程序 Manifest 中指定<receiver>元素

```
<receiver android:name=".MyBroadcastReceiver"  android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.INPUT_METHOD_CHANGED" />
    </intent-filter>
</receiver>
```
__intent filters 用来指定 receiver 所订阅的广播事件__。


2. 继承 BroadcastReceiver 并重写其 onReceive（Context，Intent）方法，以下示例中的广播接收器记录并显示广播的内容：

```
public class MyBroadcastReceiver extends BroadcastReceiver {
        private static final String TAG = "MyBroadcastReceiver";
        @Override
        public void onReceive(Context context, Intent intent) {
            StringBuilder sb = new StringBuilder();
            sb.append("Action: " + intent.getAction() + "\n");
            sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
            String log = sb.toString();
            Log.d(TAG, log);
            Toast.makeText(context, log, Toast.LENGTH_LONG).show();
        }
    }
```

The system package manager  在安装应用程序时注册 receiver。该 receiver 成为了应用程序的一个单独的入口点，这意味着如果应用程序当前未运行,系统可以启动应用程序并
发送广播。

系统创建一个新的 BroadcastReceiver 组件对象来处理它接收的每个广播。此对象仅在调用onReceive（Context，Intent）期间有效。一旦代码从此方法返回，系统会认为该组件
不再处于活动状态。


### Context-registered receivers

要使用 Context-registered receivers，请执行以下步骤：

1. 创建 BroadcastReceiver 的实例

```
BroadcastReceiver br = new MyBroadcastReceiver();
```

2. 创建一个 IntentFilter 并通过调用 registerReceiver（BroadcastReceiver，IntentFilter）来注册 Receiver：

```
IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
    filter.addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED);
    this.registerReceiver(br, filter);
```

Note: To register for local broadcasts, call LocalBroadcastManager.registerReceiver(BroadcastReceiver, IntentFilter) instead.

只要注册的 context 有效，Context-registered receivers 就会接收广播。如果在 Activity context 中注册，则只要 Activity 未被销毁，就会接收到广播。如果在 Application context
注册，则只要应用程序正在运行，就会接收到广播。

3. 要停止接收广播，调用 unregisterReceiver（android.content.BroadcastReceiver），当不再需要 Receiver或 Context 不再有效时，请务必 unregister the receiver。

要留意广播接收器的注册和取消注册位置。如果使用 Activity Context 在 onCreate（Bundle）中注册接收器，则应在 onDestroy（）中取消注册，以防止 receiver 的内存泄露。 
如果在 onResume（）中注册接收器，则应在 onPause（）中注销它以防止多次注册。不能在 onSaveInstanceState（Bundle）中取消注册，因为如果用户执行的是 Activity 的出栈操作，则就不会调用
该方法。

## Effects on process state

__BroadcastReceiver 的状态（无论是否在 Running）会影响其所在进程的状态，进而影响其被系统 Kill 的可能性__。例如，当进程正在执行 BroadcastReceiver 的onReceive（）中的
代码时，该进程会被认为是一个前台进程。除极端的内存压力外，系统都会保持其运行状态。一旦代码从 onReceive() 中返回，BroadcastReceiver 就不再处于活动状态。比如：一个进程中只有
一个manifest-declared receiver(可以看做用户从未或最近未与之交互的进程)。当其从 onReceive() 中返回，系统会降低该进程的优先级，以便其他更重要的进程需要资源时可以 kill 该进程。

因此，不应该在 broadcast receiver 中执行长时间运行的后台线程。在 onReceive（）之后，系统可以随时终止进程以回收内存，并且这样做会终止在进程中开启的线程。要避免这种情况，可以调用
goAsync()或使用 JobScheduler，来告诉系统该进程还会继续执行一些工作。

以下代码段显示了一个 BroadcastReceiver，它使用 goAsync（）标记在 onReceive() 返回后还需要一些时间（> 16ms）才能完成任务。


```
public class MyBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "MyBroadcastReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        final PendingResult pendingResult = goAsync();
        Task asyncTask = new Task(pendingResult, intent);
        asyncTask.execute();
    }

    private static class Task extends AsyncTask {

        private final PendingResult pendingResult;
        private final Intent intent;

        private Task(PendingResult pendingResult, Intent intent) {
            this.pendingResult = pendingResult;
            this.intent = intent;
        }

        @Override
        protected String doInBackground(String... strings) {
            StringBuilder sb = new StringBuilder();
            sb.append("Action: " + intent.getAction() + "\n");
            sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
            String log = sb.toString();
            Log.d(TAG, log);
            return log;
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            // Must call finish() so the BroadcastReceiver can be recycled.
            pendingResult.finish();
        }
    }
}
```

## Sending broadcasts

Android为应用发送广播提供了三种方式：

* sendOrderedBroadcast method sends broadcasts to one receiver at a time

当每个 receiver 依次接收和响应时，它可以将结果传播到下一个 receiver，或中止广播，以便它不会传递给其它 receiver。使用 intent-filter 的 android：priority 属性来控制 receivers 接收
广播的优先级,具有相同优先级的 receiver 接收广播的顺序是任意的。


* sendBroadcast method sends broadcasts to all receivers in an undefined order


* LocalBroadcastManager.sendBroadcast method sends broadcasts to receivers that are in the same app as the sender

__如果不需要跨应用程序（IPC）发送广播，请使用本地广播__。其效率更高，且无需担心与其它应用程序能够接收或发送该广播相关的任何安全问题。


以下代码段演示了如何通过创建 Intent，并调用 sendBroadcast（Intent）来发送广播。

```
Intent intent = new Intent();
intent.setAction("com.example.broadcast.MY_NOTIFICATION");
intent.putExtra("data","Notice me senpai!");
sendBroadcast(intent);
```
使用一个 intent 对象来封装 broadcast message， The intent's action string must provide the app's Java package name syntax and uniquely identify the broadcast event. 
You can attach additional information to the intent with putExtra(String, Bundle). You can also limit a broadcast to a set of apps in the same organization 
by calling setPackage(String) on the intent.


Note: intent 可以用来 sending broadcasts 和 startActivity（Intent），但这些操作完全不相关。Broadcast receivers  无法查看或捕获用于启动 Activity 的 intent。
同样，broadcast 的 intent 也无法启动一个 Activity 。

## Restricting broadcasts with permissions

权限允许将广播限制为具有特定权限的应用程序集，即可以对广播的发送者或接收者实施限制。

### Sending with permissions

当调用 sendBroadcast（Intent，String）或 sendOrderedBroadcast（Intent，String，BroadcastReceiver，Handler，int，String，Bundle）时，可以指定权限参数。
只有那些已经在其 manifest 中申请了相关权限的接收者可以接收广播。例如，以下代码发送广播：

```
sendBroadcast(new Intent("com.example.NOTIFY"),
              Manifest.permission.SEND_SMS);
```

要接收该广播，应用必须请求权限，如下所示：

```
<uses-permission android:name="android.permission.SEND_SMS"/>
```

可以指定现有系统权限（如SEND_SMS）或使用<permission>元素定义自定义权限。


### Receiving with permissions

如果在注册广播接收器时指定了权限参数（使用 registerReceiver（BroadcastReceiver，IntentFilter，String，Handler）或清单中的<receiver>标记），
则只有使用 <uses-permission> 请求权限的广播可以向接收者发送 intent。

如下所示：

```
<receiver android:name=".MyBroadcastReceiver"
          android:permission="android.permission.SEND_SMS">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE"/>
    </intent-filter>
</receiver>
```

Or your receiving app has a context-registered receiver as shown below:

```
IntentFilter filter = new IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED);
registerReceiver(receiver, filter, Manifest.permission.SEND_SMS, null );

```


为了能够向 MyBroadcastReceiver 发送广播，发送应用必须请求权限，如下所示：

```
<uses-permission android:name="android.permission.SEND_SMS"/>

```

## Security considerations and best practices


以下是发送和接收广播的一些安全注意事项和最佳实践：

* 如果无需向应用程序外部的组件发送广播，则使用 Support Library 中提供的 LocalBroadcastManager 发送和接收本地广播。LocalBroadcastManager 的效率更高（无需进程间通信），
  且无需考虑与其他应用程序能够接收或发送该广播相关的任何安全问题。本地广播可以在应用程序中用作通用发布/子事件总线，而无需系统范围广播的任何开销。

* 如果许多应用已注册在其 manifest 中接收相同的广播，则可能导致系统启动大量应用，从而对设备性能和用户体验产生重大影响。为避免这种情况，请优先使用 context-registered receivers 
  有时，Android 系统本身会强制使用 context-registered receivers 的接收器。例如，CONNECTIVITY_ACTION 广播仅传递给 context-registered receivers。

* 不要使用隐式 intent 广播敏感信息。任何注册接收广播的应用都可以读取该信息。有三种方法可以控制谁可以接收广播：

  1. 在发送广播时指定权限

  2. 在 Android 4.0及更高版本中，在发送广播时指定包含 setPackage（String）的包。系统将广播限制为与包匹配的应用程序集。
 
  3. 使用 LocalBroadcastManager 发送本地广播。




* 当注册 receiver 时，任何应用都可以向该 receiver 发送潜在的恶意广播。有三种方法可以限制应用收到的广播：

  1. 可以在注册广播接收器时指定权限
  
  2. 对于 manifest-declared receivers，可以在 manifest 中将android：exported 属性设置为“false”，即不接收来自该应用程序之外的广播。

  3. 将自己限制为仅使用 LocalBroadcastManager 进行本地广播。



* 广播操作的命名空间是全局的。确保操作名称和其他字符串都写在命名空间中，否则可能会无意中与其他应用程序发生冲突。

* 因为接收者的 onReceive（Context，Intent）方法在主线程上运行，所以它应该快速执行并返回。如果需要执行长时间运行的工作，请注意生成线程或启动后台服务，因为系统可能会在 onReceive（）
  返回后终止整个进程。


* 不要从 broadcast receivers 启动 Activity，因为用户体验很不稳定。



[broadcasts](https://developer.android.com/guide/components/broadcasts)
