# Parcel


Container for a message (data and object references) that can be sent through an IBinder. A Parcel can contain both flattened data that will 
be unflattened on the other side of the IPC (using the various methods here for writing specific types, or the general Parcelable interface), 
and references to live IBinder objects that will result in the other side receiving a proxy IBinder connected with the original IBinder in the Parcel.


Parcel 不是通用序列化机制。此类（以及用于将任意对象放入 Parcel 的相应 Parcelable API）被设计为高性能IPC传输。因此, 将任何 Parcel 数据放入持久存储中是不合适的,
Parcel 中任何数据的底层实现的更改都可能导致旧数据不可读。

简单来说，__Parcel提供了一套机制，可以将序列化之后的数据写入到一个共享内存中，其他进程通过Parcel可以从这块共享内存中读出字节流，并反序列化成对象__,下图是这个过程的模型。

[parcel_model]()


Parcel可以包含原始数据类型（用各种对应的方法写入，比如writeInt(),writeFloat()等），可以包含Parcelable对象，它还包含了一个活动的IBinder对象的引用，这个引用导致另一端
接收到一个指向这个IBinder的代理IBinder。


## Parcelables

Parcelable 协议为对象从 Parcel 中自行写入和读取数据提供了一种非常高效的协议。使用方法 writeParcelable（android.os.Parcelable，int）
和 readParcelable（java.lang.ClassLoader）或 writeParcelableArray（T []，int）和 readParcelableArray（java.lang.ClassLoader） 进行写入或读取。
这些方法将类类型及其数据写入 Parcel, 允许在稍后读取时从适当的类加载器重构该类。

## Bundles

A special type-safe container, called Bundle, is available for key/value maps of heterogeneous values. This has many optimizations for improved performance
 when reading and writing data, and its type-safe API avoids difficult to debug type errors when finally marshalling the data contents into a Parcel. 
The methods to use are writeBundle(android.os.Bundle), readBundle(), and readBundle(java.lang.ClassLoader).






































