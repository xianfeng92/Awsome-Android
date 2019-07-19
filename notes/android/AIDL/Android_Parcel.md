# Parcel

Parcel 不是通用序列化机制。此类（以及用于将任意对象放入 Parcel 的相应 Parcelable API）被设计为高性能 IPC 传输。

__Parcel提供了一套机制，可以将序列化之后的数据写入到一个共享内存中，其他进程通过Parcel可以从这块共享内存中读出字节流，并反序列化成对象__.

Parcel可以包含原始数据类型（用各种对应的方法写入，比如 writeInt(),writeFloat()等），可以包含 Parcelable 对象，它还包含了一个活动的 IBinder 对象的引用，这个引用导致另一端接收到一个指向这个 IBinder 的代理 IBinder。

# Parcelable

Parcelable 协议为对象从 Parcel 中自行写入和读取数据提供了一种非常高效的协议。使用方法 writeParcelable（android.os.Parcelable，int）
和 readParcelable（java.lang.ClassLoader）或 writeParcelableArray（T []，int）和 readParcelableArray（java.lang.ClassLoader） 进行写入或读取。
这些方法将类类型及其数据写入 Parcel, 允许在稍后读取时从适当的类加载器重构该类。

## 栗子如下:

···
package com.example.zhongxianfeng.demo_aidl;

import android.os.Parcel;
import android.os.Parcelable;

public class Book implements Parcelable {
    private int id;
    private String name;

    public Book(int id,String name){
        this.id = id;
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public int getId() {
        return id;
    }

    protected Book(Parcel in) { // 反序列化时调用的构造函数
        id = in.readInt();
        name = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }
}
···
