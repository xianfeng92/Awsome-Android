# Parcelables and Bundles

Parcelable 和 Bundle 对象可用于跨进程通信(IPC/BInder 事务), Activity 之间的数据传递以及配置更改时存储一些数据.

Note: Parcel 不是通用序列化机制, 不应将任何 Parcel数据存储在磁盘上或通过网络发送。


## Sending data between activities

当应用程序在 startActivity（android.content.Intent）中使用 Intent 对象来启动新 Activity 时, 可以使用 putExtra（java.lang.String，java.lang.String）
方法传入参数。

以下代码段显示了如何执行此操作的示例:

```
Intent intent = new Intent(this, MyActivity.class);
intent.putExtra("media_id", "a1b2c3");
// ...
startActivity(intent);
```

在 intent 调用 putExtra, 其实就是往一个 Bundle 对象中去存储该数据:

```
    public @NonNull Intent putExtra(String name, String value) {
        if (mExtras == null) {
            mExtras = new Bundle();
        }
        mExtras.putString(name, value);
        return this;
    }
```

当调用 startActivity 时, 操作系统会将先 Bundle 序列化,然后再创建新Activity，最后将 Bundle 进行反序列化传递给新的 Activity。

Bundle 类针对使用 parcel 的 marshalling 和 unmarshalling 进行了高度优化, 所以建议利用它来在 Activity 之间传递数据。

In some cases, you may need a mechanism to send composite or complex objects across activities. In such cases, the custom class should implement Parcelable, 
and provide the appropriate writeToParcel(android.os.Parcel, int) method. It must also provide a non-null field called CREATOR that implements the Parcelable.
Creator interface, whose createFromParcel() method is used for converting the Parcel back to the current object. 

通过 intent 发送数据时，应该需要主要所要传输数据大小(最多为1Mb, 发送太多数据可能导致系统抛出 TransactionTooLargeException 异常。


## Sending data between processes

在进程之间发送数据类似于在 Activity 之间执行此操作。将自定义 Parcelable 对象从一个应用程序发送到另一个应用程序，需要确保在发送和接收应用程序上都存在完全
相同的自定义 Parcelable 的版本。通常，这可能是在两个应用程序中使用的公共库.如果应用程序向系统发送的自定义 parcelable 则会发生错误,因为系统无法 unmarshal 
不知道的类。

例如, 应用程序可能使用 AlarmManager 类设置 alarm, 并在 alarm 的 intent 上使用自定义 Parcelable. 当 alarm 响起时, 系统会修改 intent 的附加内容以添加重复次数。

此修改可能导致系统从附加功能中剥离自定义 Parcelable。这种剥离可能导致应用程序在收到修改后的 alarm intent 时崩溃, 因为自定义 Parcelable已经不在存在了。

__Binder 事务缓冲区的固定大小是有限的(目前为 1MB), 该缓冲区由进程中的所有正在进行的事务共享__。这些事务包括应用程序中的所有 Binder 事务, 例如:
onSaveInstanceStateSince, startActivity 以及应用程序和系统的交互. 当这些总大小超过 1 MB 时,会抛出一个 TransactionTooLargeException.

对于 savedInstanceState, 其应该存储尽量少的数据量, 一般数据量应该建议少于50k。

Note: 在Android 7.0（API级别24）及更高版本中，系统会抛出TransactionTooLargeException作为运行时异常。在较低版本的Android中，系统仅在logcat中显示警告。

























































