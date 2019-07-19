## AIDL

Android Interface Definition Language(AIDL),即 Android 接口定义语言。服务端负责实现这些接口, 而客户端是来调用这些接口。

## 服务端实现 Binder 对象

在服务端实现一个特殊的对象(Binder对象),这个对象有两个特性:

1. 完成特定任务的能力,即服务  -----> AIDL 接口的实现

2. 跨进程传输的能力,即 Binder 对象  ------> 进程间数据传输

Binder 具有跨进程传输的能力是因为它实现了 IBinder 接口,系统会为每个实现了该接口的对象提供跨进程传输。

Binder 具有的完成特定任务的能力是通过它的 attachInterface 方法,该方法会将（descriptor，"服务"）作为（ key,value ）存入 Binder 对象中的一个 Map<String,IInterface> 对象中。Binder 对象可通过 attachInterface 方法持有一个 IInterface 对象的引用，并依靠它获得完成特定任务的能力。

queryLocalInterface 方法是根据 key 值（即参数 descriptor）查找相应的 IInterface 对象。

这里我们可以将 descriptor 看作服务的唯一标识, “服务”是存储在 Binder 的一个Map<String,IInterface>, 所以“服务”是要实现 IInterface 接口。

ps:为什么Binder的Map<String,IInterface>对象,存储一个 IInterface 而不是具体服务呢？

面向接口编程~~~~~~~~~

## AIDL 调用流程

![aidl.png](https://github.com/xianfeng92/Awsome-Android/blob/master/images/aidl.png)

假设客户端已经 bind 服务端,并且客户端的已持有系统所提供的服务端的 Binder 对象(mRemote), 服务端提供 addBook 和 getBook 的服务。

1. 当客户端想要调用服务端的 addBook 服务时, 是通过 mRemote 的 transact 方法将调用的接口(此处即addBook)和参数通过系统底层传给服务端。

2. 服务端的 onTransact 方法接收到请求后,解析所要调用的接口和参数后,去调用服务端本地的对应的接口,并将返回值回传给客户端。

3. 上图中可以看到,当客户端发起一次服务端的调用请求时,客户端会挂起,直到服务端给出返回数据。所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求。

4. 服务端是通过线程池来处理客户端的请求,所以 AIDL 调用是多线程的。

## 代码实现和解析

### 客户端代码

```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private static final String TAG = "MainActivity";
    private IBookManager mService;
    private ArrayList<Book> list;
    private Button button1;
    private Button button2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        button1 = findViewById(R.id.button1);
        button1.setOnClickListener(this);
        button2 = findViewById(R.id.button2);
        button2.setOnClickListener(this);
        Intent intent = new Intent(MainActivity.this,BookManagerService.class);

        // 调用bindService时，当绑定成功时，会调用 BookManagerService的onBind方法，
        bindService(intent,serviceConnection,BIND_AUTO_CREATE);
    }

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程.
            // 那么此方法返回的就是服务端的Stub对象本身,否则返回的是系统封装后的Stub.proxy
            // 这里由于 BookManagerService 为remote service，故获取到的是 Stub.proxy
            // 客户端利用服务端返回的Binder构建一个Proxy。
            mService = IBookManager.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    public void onClick(View v) {
        switch (v.getId()){
            case R.id.button1:
                try {
                    // 利用Proxy调用服务端相关方法,即 mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    mService.addBook(new Book(1,"Happy"));
                    mService.addBook(new Book(2,"New"));
                    mService.addBook(new Book(3,"Year"));
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            case R.id.button2:
                try {
                    list = (ArrayList<Book>) mService.getBook();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                for(int i=0;i<list.size();i++){
                    Log.d(TAG, "getBook: "+list.get(i).getName());
                }
                break;
                default:
                    break;
        }
    }
}

```

### 服务端代码

```
public class BookManagerService extends Service {

    private ArrayList<Book> list;

    //Binder 实体是 Server进程 在 Binder 驱动中的存在形式
    //该对象保存 Server 和 ServiceManager 的信息
    //Binder 驱动通过 内核空间的Binder 实体 找到用户空间的Server对象
    IBookManager.Stub mBinder = new IBookManager.Stub() {

        @Override
        public List<Book> getBook() throws RemoteException {
            return list;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            if (list == null){
                list = new ArrayList<>();
            }
            list.add(book);
        }
    };

    // 当另外一个进程A调用bindService方法去绑定在进程B中的 service 时，系统收到进程A的bindService请求后，会调用进程B中相应service的onBind方法
    // 该方法会返回一个Binder对象，相应的在进程A的调用处，通过 ServiceConnection 的 onServiceConnected 回调，可以拿到该Binder
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

### AIDL文件

```
//Binder的唯一标识，一般用当前Binder的类名表示
private static final java.lang.String DESCRIPTOR = "com.example.zhongxianfeng.demo_aidl.IBookManager";


// 服务端实现 Stub 类,从而可以为客户端提供“服务”, 服务端是通过继承 android.os.IInterface 接口,来定义一些“服务”,并将其实现
public static abstract class Stub extends android.os.Binder implements com.example.zhongxianfeng.demo_aidl.IBookManager
{
private static final java.lang.String DESCRIPTOR = "com.example.zhongxianfeng.demo_aidl.IBookManager";
/** Construct the stub at attach it to the interface. */
public Stub()
{
this.attachInterface(this, DESCRIPTOR);
}

/**
 * Cast an IBinder object into an com.example.zhongxianfeng.demo_aidl.IBookManager interface,
 * generating a proxy if needed.
 */
   // 将Binder对象转换成 IBookManager 接口,分两种情况:
   // 1 服务端: 其实是可以通过 queryLocalInterface ,查到对应的 IBookManager 接口的,那么此时就会直接返回 IBookManager 接口
  //  注意,在服务端我们已经实现了 IBookManager的Stub. 面向对象的多态特性决定了在服务端调用 IBookManager 相关的getBook或者addBook
  //  方法时,其实是在调用服务端的getBook或者addBook的具体实现.

 // 2 客户端: 客户端返回的将是一个 IBookManager 的 proxy 对象, 当我们在客户端调用 proxy的 getBook或者addBook 时
 //  其实是通过一个调用一个 mRemote 的 transact 方法,来远程调用服务端相应方法的具体实现. 
 // 这里的 mRemote 是当客户端 bind (绑定) 服务端时,系统给我们返回的一个 binder 对象. 我们用这个 binder 就可以实现远程调用服务端的服务了
public static com.example.zhongxianfeng.demo_aidl.IBookManager asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof com.example.zhongxianfeng.demo_aidl.IBookManager))) {
return ((com.example.zhongxianfeng.demo_aidl.IBookManager)iin);
}
return new com.example.zhongxianfeng.demo_aidl.IBookManager.Stub.Proxy(obj);
}
@Override public android.os.IBinder asBinder()
{
return this;
}

// 这个方法运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。服务端通过code可以确定客户端所请求的    目标方法是什么，
// 接着从data中取出目标方法所需要的参数（如果目标方法中有参数的话），然后执行目标方法，当目标方法执行完毕后，就向reply中写入返回值（如果目标方法有返回    值的话)
@Override 
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
java.lang.String descriptor = DESCRIPTOR;
switch (code)
{
case INTERFACE_TRANSACTION:
{
reply.writeString(descriptor);
return true;
}
case TRANSACTION_getBook:
{
data.enforceInterface(descriptor);
java.util.List<com.example.zhongxianfeng.demo_aidl.Book> _result = this.getBook();
reply.writeNoException();
reply.writeTypedList(_result);
return true;
}
case TRANSACTION_addBook:
{
data.enforceInterface(descriptor);
com.example.zhongxianfeng.demo_aidl.Book _arg0;
if ((0!=data.readInt())) {
_arg0 = com.example.zhongxianfeng.demo_aidl.Book.CREATOR.createFromParcel(data);
}
else {
_arg0 = null;
}
this.addBook(_arg0);
reply.writeNoException();
return true;
}
default:
{
return super.onTransact(code, data, reply, flags);
}
}
}

// 服务端BookManager的代理对象,其也实现了IBookManager接口
private static class Proxy implements com.example.zhongxianfeng.demo_aidl.IBookManager
{
// mRemote 为服务端返回的Binder对象
private android.os.IBinder mRemote;

Proxy(android.os.IBinder remote)
{
 mRemote = remote;
}

@Override 
public android.os.IBinder asBinder()
{
return mRemote;
}

public java.lang.String getInterfaceDescriptor()
{
return DESCRIPTOR;
}

// 这个方法运行在客户端，当客户端远程调用此方法时,它的内部实现是这样的：
// 首先创建该方法所需要的输入型 Parcel对象_data、输出型Parcel对象_reply和返回值对象List,然后把该方法的参数信息写入_data中（如果有参数的话)
// 接着调用 transact 方法来发起 RPC（远程过程调用)请求, 同时当前线程挂起,然后服务端的 onTransact 方法会被调用。
// 直到RPC过程返回后, 当前线程继续执行, 并从 _reply 中取出 RPC 过程的返回结果,最后返回_reply中的数据。
@Override 
public java.util.List<com.example.zhongxianfeng.demo_aidl.Book> getBook() throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
java.util.List<com.example.zhongxianfeng.demo_aidl.Book> _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
// 通过Binder远程调用服务端的getBook方法
mRemote.transact(Stub.TRANSACTION_getBook, _data, _reply, 0);
_reply.readException();
_result = _reply.createTypedArrayList(com.example.zhongxianfeng.demo_aidl.Book.CREATOR);
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}

// 这个方法路行在客户端，它的执行过程和getBook是一样的，addBook没有返回值，所以它不需要从.reply中取出返回值。
// 通过上面分析，应该已经理解到Binder工作机制，但是有两点还是需要额外说明一下,
// 首先，当客户端发起远程请求时，由于当前线程会被挂起直至服务器进程返回数据。
// __所以如果一个远程方法是很耗时的，那么不能再UI线程中发起此远程请求。__

@Override 
public void addBook(com.example.zhongxianfeng.demo_aidl.Book book) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
if ((book!=null)) {
_data.writeInt(1);
book.writeToParcel(_data, 0); // 将book参数存储到_data中
}
else {
_data.writeInt(0);
}
// 通过Binder远程调用服务端的addBook方法
mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
_reply.readException();
}
finally {
_reply.recycle();
_data.recycle();
}
}

}
```

### 案例代码

具体案例代码 

[Demo_AIDL](https://github.com/xianfeng92/Demo_AIDL)

# 参考

[Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析](https://blog.csdn.net/Luoshengyang/article/details/6642463)

[可能是讲解Binder机制最好的文章](https://www.jianshu.com/p/1eff5a13000d)

[Android开发艺术探索——第二章：IPC机制](https://blog.csdn.net/qq_26787115/article/details/52664405)

[Android跨进程通信：图文详解 Binder机制 原理](https://blog.csdn.net/carson_ho/article/details/73560642)
