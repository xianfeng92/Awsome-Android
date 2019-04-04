# Intents and Intent Filters

Intent 是一个消息传递对象(messaging object)，可用于从另一个应用程序组件中请求操作。 尽管 intent 有多种方式可以完成组件之间的通信，但有三个基本用例：

* Starting an activity

  一个 Activity 表示应用中的一个屏幕。通过将 Intent 传递给 startActivity（）来启动 Activity 的新实例，其 Intent 描述了要启动的 Activity 并可以携带任何必要的数据。

* Starting a service

  __service 是在没有用户界面的情况下在后台执行操作的组件__。对于 Android 5.0（API级别21）及更高版本，可以使用 JobScheduler 启动服务。对于 Android 5.0（API级别21）之前那的版本，
可以使用 Service 类的方法启动服务。将 Intent 传递给 startService（）来启动服务以执行一次性操作（例如下载文件）。Intent描述了要启动的服务并可以携带任何必要的数据。如果 Service 
是使用client-server 接口设计的，则可以通过将 Intent 传递给 bindService（）来绑定到另一个组件的服务

* Delivering a broadcast

broadcast 是任何应用都可以接收的消息。系统为系统事件提供各种广播，例如系统启动或设备开始充电时。通过将 Intent 传递给 sendBroadcast（）或 sendOrderedBroadcast（）可以向其他应用程序广播。

## Intent types

有两种类型的 intent：

* Explicit intents 

Explicit intents 通过提供目标应用程序的包名称或完全限定的组件类名称来指定哪个应用程序将满足 intent。__通常会使用 Explicit intents 在自己的应用中启动组件__，因为此时我们知道要启动
的 Activity 或Service 的类名。

例如，在应用中启动新 Activity 以响应用户操作，或启动 Service 以在后台下载文件。

* Implicit intents

__Implicit intents 不命名特定组件，而是声明要执行的常规操作，这允许来自另一个应用程序的组件处理它__。例如，如果要向用户显示地图上的位置，则可以使用 Implicit intents 请求另一个有
能力的应用程序在地图上显示指定位置。

下图显示了在启动 Activity 时如何使用intent。当Intent对象显式指定特定 Activity 组件时，系统立即启动该组件。当使用 Implicit intents 时，Android 系统会通过将 intent 的内容与设备上其他应
用程序的 manifest 文件中声明的 intent filters 进行比对来找到适当的组件。如果 intent 与 intent filter 匹配，则系统启动该组件并将 Intent 对象传给它。如果 intent 与多个 intent filter匹配，
系统将显示一个对话框，以便用户可以选择要使用的应用程序。

![intent-filters_2x.png]()


intent filters 是应用程序 manifest 中的定义的，用于指定组件要接收的 intent 类型。例如，通过为 Activity 声明一个 intent filters，可以让其他应用程序以某种 intent 来启动我们的 Activity。
如果没有为Activity 声明任何 intent filters，则只能使用 Explicit intents 启动它。

Caution：为确保应用程序的安全，在启动 Service 时应该使用 Explicit intents，并且不要为其声明 intent filters。使用 Implicit intents 启动 Service 存在安全隐患，因为我们无法确定哪些
Service 将响应 intent并且用户无法查看启动的是哪个 Service。从Android 5.0（API级别21）开始，如果使用 Implicit intents 调用 bindService（），系统将抛出异常。

## Building an intent

Intent 对象携带 Android 系统用于确定要启动的组件的信息（例如应该接收intent的组件名称或组件类别），以及为了正确执行 Action 而需要使用的信息。

Intent中包含的主要信息如下：

### Component name
  
 要启动的组件的名称。这是可选的，但它是 Explicit intents 的关键信息，意味着 intent 仅传递给该 component name 所定义的app组件。如果没有 component name，则 intent 是 Implicit，系统根
据其他 intent 信息（例如 action, data, and category）决定哪个组件应该接收该 intent。如果需要在应用程序中启动特定组件，则应指定组件名称。

Note：启动 Service 时，需要指定组件　Component name。否则，无法确定哪些 Service 将响应 intent，并且用户无法查看哪个　Service 被启动。

Intent　的　ComponentName　字段可以使用目标组件的完全限定类名指定该对象，例如com.example.ExampleActivity。还可以使用setComponent（），setClass（），setClassName（）或Intent构造函数设置
组件名称。

### Action

一个字符串，指定要执行的操作（例如 view or pick）。

在 broadcast intent 的情况下，intent 则表示正在发生并被报告的一个　Action。该　Action　很大程度上决定了 intent 的其余部分是如何构建的-特别是数据和附加内容中包含的信息。

我们指定自己的 Action 以供应用程序中的 intent 使用，但通常指定由 Intent 类或其他框架类定义的 action constants。以下是启动 Activity 的一些常见操作：

* ACTION_VIEW

当 Activity 可以向用户显示的某些信息时，例如要在图库应用中查看照片或要在地图应用中查看地址，可以在 intent 中使用 ACTION_VIEW。

* ACTION_SEND

也称为共享 intent，当用户想向其他应用程序（例如电子邮件应用程序或社交共享应用程序）共享的某些数据时，应该在 startActivity（）的 intent 中使用此 ACTION_SEND。


可以使用 setAction（）或Intent构造函数为intent指定 Action。

如果定义自己的Action，请确保使用应用程序的包名称作为前缀，如以下示例所示：

```
static final String ACTION_TIMETRAVEL = "com.example.action.TIMETRAVEL";

```

### Data

URI（一个Uri对象），它引用要作用的数据或该数据的MIME类型。提供的数据类型通常由 intent 的 action 决定。 例如，如果操作是 ACTION_EDIT，则数据应包含要编辑的文档的URI。

在创建 intent 时，除了 URI 之外，通常还需要指定数据类型（其MIME类型）。例如，即使URI格式可能类似，但能够显示图像的 Activity 可能无法播放音频文件。指定数据的MIME类型
有助于Android系统找到接收意图的最佳组件。

如果只要设置数据URI，调用setData(),只要设置MIME类型，请调用setType()如有都需要设置，使用setDataAndType()。


### Category

一个包含有关应处理 intent 的组件类型的其他信息的字符串。可以在 intent 中放置任意数量的 Category，但大多数 intent 不需要 Category。以下是一些常见 Category：

* CATEGORY_BROWSABLE

  The target activity allows itself to be started by a web browser to display data referenced by a link, such as an image or an e-mail message.

* CATEGORY_LAUNCHER

  The activity is the initial activity of a task and is listed in the system's application launcher.


上面列出的这些属性（Component name，Action，Data 和 Category）表示 intent 的定义特征。__通过这些属性，Android系统能够解析应该要启动的应用程序组件__。intent 可以携带其他信息，这些
信息不会影响应用程序组件的解析方式。intent 还可以提供以下信息：

* Extras

 Key-value pairs 带有完成请求的 Action 所需的附加信息。使用 putExtra（）方法添加额外数据，每个方法都接受两个参数：键名和值。还可以通过创建 Bundle 对象来添加额外数据，然后使用 putExtras（）
 将 Bundle 插入 Intent 中。

Caution：向其他应用程序发送 intent 时，请勿使用 Parcelable 或 Serializable 数据。当该应用程序尝试访问 Bundle对象中的数据但无法访问 parceled 或 serialized 类时，则会引发系统的 RuntimeException。


* Flags


Flags 在Intent类中定义，作为intent的元数据。__Flags可以指示Android系统如何启动 Activity__（例如，Activity 应该属于哪个 task）以及如何在启动 Activity 后对其进行处理。
例如，它是否属于 recent activities 的列表。


## Example explicit intent


explicit intent 用于启动特定应用程序组件的 intent，例如应用程序中的特定Activity 或 Service。要创建 explicit intent，component name 属性是必须要提供的 - 所有其他 intent 属性都是可选的。


例如，如果在应用程序中构建了一个名为 DownloadService 的服务，旨在从Web下载文件，则可以使用以下代码启动它：


```
// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
Intent downloadIntent = new Intent(this, DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
```

## Example implicit intent

implicit intent 指定了一个 Action，其可以调用设备上能够执行该 Action 的应用程序。当我们的应用无法执行 Action 时，使用 implicit intent 很有用。

例如，如果用户希望与其他人共享内容，通过使用 ACTION_SEND 操作创建 intent，并添加上要共享的内容，使用该 intent 来调用 startActivity（）时，用户可以
选择通过其共享内容的应用程序。

Caution：当系统中没有一个应用程序可以处理我们的 implicit intent 时，此时我们的应用程序就会 crash。可以通过在 Intent 对象上调用 resolveActivity()来验证是否有Activity可以处理该 intent,
如果结果为非 null，则至少有一个应用程序可以处理该 intent，此时可以安全地调用 startActivity()。


```
// Create the text message with a string
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
sendIntent.setType("text/plain");

// Verify that the intent will resolve to an activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(sendIntent);
}
```

当调用 startActivity() 时，系统会检查所有已安装的应用程序，以确定哪些应用程序可以处理此类 intent。如果只有一个应用程序可以处理它，那么该应用程序会立即打开并获得 intent。
如果多个 Activity 可以处理 intent，系统将显示一个对话框，由用户来选择要使用的应用程序。


## Forcing an app chooser

当我们始终要使用 chooser 时，可以通过 createChooser（）来创建一个 Intent 并将其传递给 startActivity()。下面的例子就是要显示一个对话框，其中包含响应传递给 createChooser() 
的 intent 的应用程序列表，并使用提供的文本作为对话框标题。

```
Intent sendIntent = new Intent(Intent.ACTION_SEND);
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show the chooser dialog
Intent chooser = Intent.createChooser(sendIntent, title);

// Verify the original intent will resolve to at least one activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```

## Receiving an implicit intent


































































