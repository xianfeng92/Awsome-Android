
#  Context

Context 提供了对应用程序状态信息的访问。如: 在 Activity 、Fragment 和 Service 中对资源文件、图像、主题/样式和外部目录位置的访问。
它还允许访问Android的内置服务，如布局加载器、键盘和查找 content providers。

## Context 具体可以用于做什么?

### 显式的启动组件

    ```
    //如果 myActivity 是一个内部的Activity, 传入 context
    Intent intent = new Intent(context, MyActivity.class);
    startActivity(intent);
    ```

当我们显式启动一个组件时, 需要提供两个信息:
     * package name, 用于标识包含所需启动组件的应用程序
     * 组件的全限定Java类名
     
如果启动是一个内部组件, 则可以传入 context, 因为可以通过 context.getPackageName（）提取当前应用程序的 package name

### 创建视图

    ```
    TextView textView = new TextView(context);
    ```
context 中包含视图所需要的以下信息:
    
    * 用于将dp、sp转换为 px 的设备屏幕大小和尺寸

    * 样式化属性(styled attributes)

    * onclick属性对 Activity 引用

### 加载布局文件

    ```
    // A context is required when creating views.
    LayoutInflater inflater = LayoutInflater.from(context);
    inflater.inflate(R.layout.my_layout, parent);
    ```

### 发送本地广播

当发送或注册广播的接收器时, 我们使用 Context 来获取 LocalBroadcastManager:

    ```
    Intent broadcastIntent = new Intent("custom-action");
    LocalBroadcastManager.getInstance(context).sendBroadcast(broadcastIntent);
    ```

### 检索系统服务

要从应用程序发送通知时, 需要使用 NotificationManager 系统服务:

    ```
    // Context 对象能够获取或启动系统服务
    NotificationManager notificationManager = 
    (NotificationManager) getSystemService(NOTIFICATION_SERVICE);

    int notificationId = 1;

    // 构建 RemoteViews 对象时需要 Context
    Notification.Builder builder = 
    new Notification.Builder(context).setContentTitle("custom title");
    
    notificationManager.notify(notificationId, builder.build());
    ```

## Application vs Activity Context

虽然主题和样式通常在 Application 级别应用, 但也可以在 Activity 级别指定。通过这种方式, Activity 可以具有与应用程序其余部分不同的主题或样式集。
在androidmanifest.xml文件中，应用程序通常有一个android:theme。当然我们还可以为 Activity 指定一个不同的 theme:
    
    ```
    <application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme" >
    <activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:theme="@style/MyCustomTheme">
    ```
因此,重要的是要知道有一个Application Context 和一个 Activity Context, 它们在各自的生命周期中持续存在。大多数视图(View)都应该传递一个 Activity Context,
以便访问应用程序的 themes, style 以及 dimensions。如果没有为 Activity 显式指定 themes, 则默认情况下使用应用程序指定的 themes。
在大多数情况下, 应该使用 Activity Context。通常, 使用 this 关键字来引用类的实例在 Activity 中需要 Context 时使用.
下面的示例显示了如何使用此方法显示Toast消息:
    
    ```
    public class MainActivity extends AppCompatActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);  
    Toast.makeText(this, "hello", Toast.LENGTH_SHORT).show();
    }
    }
    ```
    
## 匿名函数
    
由于 Java 中的 this 关键字适用于被直接声明的类,当我们使用一个匿名内部类来实现一个 Listener 时, 必须指定外部类 mainActivity 来引用 Activity 的实例。
       
       ```
       public class MainActivity extends AppCompatActivity {
       
       @Override
       protected void onCreate(Bundle savedInstanceState) {
       
       TextView tvTest = (TextView) findViewById(R.id.abc);
       tvTest.setOnClickListener(new View.OnClickListener() {
       @Override
       public void onClick(View view) {
       Toast.makeText(MainActivity.this, "hello", Toast.LENGTH_SHORT).show();
       }
       });
       }
       }
       }
       ```
       
## Adapters

### Array Adapter

在为 ListView 构造 Adapter 时, 通常在布局加载中调用 getContext(), 此方法使用的是实例化 ArrayAdapter 的Context:

    ```
    if (convertView == null) {
    convertView = 
    LayoutInflater
    .from(getContext())
    .inflate(R.layout.item_user, parent, false);
    }
    ```
如果用 Application Context 来实例化 ArrayAdapter, 可能会发现到主题/样式没有被应用。因此,在这些情况下,请确保传递的是 Activity Context。

###  RecyclerView Adapter
    ```
    public class MyRecyclerAdapter extends RecyclerView.Adapter<MyRecyclerAdapter.ViewHolder> {

    @Override // parent 即与 RecyclerView.Adapter 所关联的 RecyclerView
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    View v = 
    LayoutInflater
    .from(parent.getContext())
    .inflate(R.layout.itemLayout, parent, false);

    return new ViewHolder(v);
    }

    @Override
    public void onBindViewHolder(ViewHolder viewHolder, int position) {
    // If a context is needed, it can be retrieved 
    // from the ViewHolder's root view.
    Context context = viewHolder.itemView.getContext();

    // Dynamically add a view using the context provided.
    if(position == 0) {
    TextView tvMessage = new TextView(context);
    tvMessage.setText("Only displayed for the first item.")

    viewHolder.customViewGroup.addView(tvMessage);
    }
    }

    public static class ViewHolder extends RecyclerView.ViewHolder {
    public FrameLayout customViewGroup;

    public ViewHolder(View imageView) {
    // Very important to call the parent constructor
    // as this ensures that the imageView field is populated.
    super(imageView);

    // Perform other view lookups.
    customViewGroup = (FrameLayout) imageView.findById(R.id.customViewGroup);
    }
    }
    }
    ```
RecyclerView.Adapter 中可以通过其 the parent view 来获取一个正确的 Context. 与 RecyclerView.adapter 所关联的 RecyclerView 会始终将自身作为父视图
(parent)传递到 RecyclerView.adapter.onCreateViewHolder()中。如果在 onCreateViewholder()
方法之外需要 Context, 只要有可用的 Viewholder 实例，就可以通过: viewholder.itemview.getContext() 来检索Context。

## 避免内存泄漏

Application Context 通常在需要创建单例实例时使用,例如需要 Context 信息访问系统服务,在多个 Activity 中重用的自定义管理器类。
由于保留对 Activity Context 的引用会导致内存在不再运行后无法回收, 因此使用应用程序上下文非常重要。

在下面的示例中，如果存储的 Context 是一个 Activity 或 Service , 并且被 Android 系统所 destroy, 那么它将无法被垃圾收集, 因为 CustomManager 类持有对它的静态引用。

    ```
    public class CustomManager {
    private static CustomManager sInstance;

    public static CustomManager getInstance(Context context) {
    if (sInstance == null) {

    // This class will hold a reference to the context
    // until it's unloaded. The context could be an Activity or Service.
    sInstance = new CustomManager(context);
    }

    return sInstance;
    }

    private Context mContext;

    private CustomManager(Context context) {
    mContext = context;
    }
    }
    ```

## 存储 context:使用 application context

为了避免内存泄漏，不要在 Contect 的生命周期之外保存对其的引用。应当检查我们的后台线程、挂起的处理程序或可能保存在 Context 对象上的内部类。正确的使用方法是
将 application context 存储在 customManager.getInstance()中。application context 是与应用程序进程的生命周期相关联, 这样可以安全地
存储对它的引用。
    
当需要的 Context 的引用超过了组件的寿命时, 或者 Context 引用应该独立于所传入的 Context 的生命周期, 应该使用 application context。
