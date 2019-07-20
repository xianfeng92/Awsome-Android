# Guide to background processing

每个Android应用程序都有一个主线程(UI Thread)来负责处理用户界面（包括测量和绘图视图）、协调用户交互和接收生命周期事件。如果在这个线程上做太多的工作,  应用程序就会挂起或变慢，从而导致用户体验变差。任何长时间运行的计算和操作，如: 解码 bitmap、访问磁盘或执行网络请求,都应该在单独的后台线程上完成。
一般来说，任何超过几毫秒的事情都应该委托(delegate)给后台线程,其中一些任务可能需要在用户主动与应用程序交互时执行。

即使用户没有使用应用程序时，应用程序也可能需要运行一些任务，例如定期与后端服务器同步或定期获取应用程序中的新内容。即使在用户完成与应用程序的交互之后,可能也需要服务运行，直到其完成任务，

## Choosing the right solution for your work

* 这个任务能可以延时(deferred)运行吗,还是需要立马去完成?

如果需要从网络中获取一些数据以响应用户单击按钮，那么必须立即完成该任务。但是，如果您想将日志上传到服务器，那么可以推迟该任务，而不会影响应用程序的性能或用户期望。

* 任务的执行是否取决于系统条件？

指定任务仅在设备满足某些条件时运行,例如连接到电源、具有Internet连接等。如,应用程序可能需要定期压缩其存储的数据。为了避免影响用户,可以在设备充电和空闲时执行此作业。

* 任务是否需要在指定的时间(precise time)运行？

一个日历应用程序可能允许用户在特定时间为事件设置提醒,用户希望在正确的时间看到提醒通知。还有一些情况,应用程序不要求任务在特定的时间运行,只需要任务可以满足一些条件,比如“任务A必须先运行，然后是任务B，然后是任务C”。

![bg-job-choose](https://github.com/xianfeng92/android-code-read/blob/master/images/bg-job-choose.svg)


### WorkManager

使用 WorkManager 时, 当任务的执行条件（如网络可用性和电源）满足时,即使设备发生了重启,它也可以优雅地运行延迟的后台任务。

#### Foreground services

对于需要立即运行并且必须执行完成的用户启动(user-initiated)的任务,可以使用 Foreground services. 使用 Foreground services 时,应用程序会告诉系统, 其正在做一些重要的事情,不应该杀死它。前台服务通过通知托盘中不可拒绝的通知对用户可见。

#### AlarmManager

如果想要在一个精确的时间点上运行任务, 此时可以使用 AlarmManager。AlarmManager 将启动应用程序以在指定的时间执行该任务。如果我们的任务不需要在
精确的时间运行,那么WorkManager是一个更好的选择.WorkManager 能够更好地平衡系统资源.例如,如果我们需要每小时左右运行一个作业, 但不需要在特定时间运行该作业,
则应使用 WorkManager 设置定期任务(recurring job)。

#### DownloadManager

如果应用程序正在长时间执行的HTTP下载任务(long-running HTTP downloads), 此时可以考虑使用DownloadManager.客户端可能会请求将URI下载到应用程序进程之外的特定目标文件。

DownloadManager 将在后台进行下载，处理HTTP交互，并在失败后或在连接更改和系统重新启动时重试下载。

[Guide to background processing](https://developer.android.google.cn/guide/background/)

