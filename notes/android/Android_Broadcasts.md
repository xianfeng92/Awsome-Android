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























































[broadcasts](https://developer.android.com/guide/components/broadcasts)
