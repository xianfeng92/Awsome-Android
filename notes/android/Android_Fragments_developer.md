# Fragments

## Creating a Fragment

我们可以通过继承 Fragment 来创建一个 fragment, fragment类的代码和Activity非常相似。它包含着类似于Activity的回调方法,如:onCreate（）、onStart（）、
onPause（）和onStop（）。事实上, 如果想要将现有的Android应用程序转换为使用 Fragment, 只需要简单地将代码从 Activity 的回调方法移动到 Fragment 的相应
的回调方法中。

通常, 应该至少实现以下生命周期方法:

* onCreate

  系统在创建 Fragment 时调用此函数。在我们的实现中, 在该方法中初始化 Fragment 的基本组件, 该组建在 Fragment 暂停或停止时仍然会继续保留。

* onCreateView

当 Fragment 第一次绘制其用户界面时, 系统会调用此函数。要为 Fragment 绘制UI, 必须从该方法返回一个视图(View), 该方法是 Fragment 布局的根。
如果 Fragment 不提供UI, 则可以返回空值。

* onPause

系统调用此方法作为用户离开 Fragment 的第一个指示（尽管它并不总是意味着 Fragment 正在被销毁). 该方法通常会将一些需要持久化的用户数据进行保存。


![fragment_lifecycle](https://github.com/xianfeng92/android-code-read/blob/master/images/fragment_lifecycle.png)


如下是一些 Fragment 的子类, 我们可以直接继承这些子类来实现特定的需求:

* DialogFragment

显示浮动对话框, 使用这个类来创建一个对话框可以很的的替代 直接在Activity中使用dialog helper methods, 因为可以将 DialogFragment 放入 Activity 所管理的Fragment的栈中 ( back stack), 从而允许用户返回到一个已解除的 fragment。

* ListFragment

显示由适配器（如SimpleCursorAdapter）管理的项目列表, 类似于ListActivity。它提供了几种管理列表视图的方法,例如处理单击事件的 OnListItemClick（）回调。


* PreferenceFragmentCompat

Displays a hierarchy of Preference objects as a list. This is used to create a settings screen for your application.



## Adding a user interface


Fragment 通常用作 Activity 用户界面的一部分, 并为 Activity 提供自己的布局。要为 Fragment 提供布局, 必须重写其 onCreateView（）回调方法, 当 Fragment 绘制布局时, Android系统将调用该方法。此方法的实现必须返回一个视图(View), 该视图是 Fragment 布局的根。

Node: 如果我们实现的 fragment 是 ListFragment 的子类, 则默认实现从 onCreateView（）返回ListView, 因此不需要重写该方法.


要从 onCreateView（）返回布局, 可以从 XML 中定义的布局资源对其进行加载。因此在 onCreateView（）也提供了一个layoutInflator对象,用于加载fragment的布局。


例如，下面是fragment的子类, 它从 example_fragment.xml 文件中加载布局:


```
public static class ExampleFragment extends Fragment {
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        return inflater.inflate(R.layout.example_fragment, container, false);
    }
}
```

onCreateView() 中的参数 container 是 fragment 布局所要插入的父布局( Activity 的布局中的一个 viewGroup). savedinstanceState 参数是一个bundle, 它提供有关 Fragment 的上一个实例的数据(如果 Fragment 正在 Resume).

The inflate() method takes three arguments:

* 所要加载的布局资源的id

* 所要加载布局的父布局, 系统在该父布局中添加 fragment 的布局.

* 一个布尔值, 指示布局在加载期间是否应附加到 ViewGroup（第二个参数）. 应该为 false, 因为系统已经将加载的布局插入到容器中, 传递 true 将在最终布局中创建冗余的 ViewGroup。


接下是, 是将 fragment 添加到 Activity 中:

## Adding a fragment to an activity


有两种方法可以将 Fragment 添加到活动布局中:

1. Declare the fragment inside the activity's layout file.

在 Activity 的布局中直接声明一个 fragment, 如下所示:

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment android:name="com.example.news.ArticleListFragment"
            android:id="@+id/list"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
    <fragment android:name="com.example.news.ArticleReaderFragment"
            android:id="@+id/viewer"
            android:layout_weight="2"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
</LinearLayout>
```


 <fragment> 中的 android:name 属性指定要在布局中实例化的 fragment 类。

当系统创建上面的 Activity 布局时, 它会实例化布局中指定的每个 fragment, 并调用它们的 onCreateView（）方法, 以检索每个 fragment 的布局。然后系统将 fragment 返回的视图(View)直接插入到<fragment>元素的位置。


Note: 每个 fragment 都需要一个唯一的标识符, 如果 Activity 重新启动, 系统可以使用该标识符来还原fragment ,并且可以使用该标识符捕获片段以执行事务,例如删除它。

有如下两种方法,可以为 fragment 提供一个唯一标识符:

* 为android:id 属性提供唯一的id

* 为android:tag 属性提供唯一的字符串


2. programmatically add the fragment to an existing ViewGroup.

在 Activity 运行的任何时候, 都可以将 Fragment 添加到 activity 布局中。我们只需要指定一个 View Group 来放置Fragment。

要在 Activity 中创建 Fragment 事务, 例如添加、删除或替换片段, 必须使用 FragmentTransaction 中的API。我们可以从 fragmentActivity 中获取 fragmentTransaction的实例, 如下所示:

```
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```


然后, 可以使用add（）方法来添加 Fragment,需要指定要添加的 Fragment 以及插入Fragment的视图(View)。例如:

```
ExampleFragment fragment = new ExampleFragment();
fragmentTransaction.add(R.id.fragment_container, fragment);
fragmentTransaction.commit();
```

传递给add（）的第一个参数是应该放置 Fragment 的 ViewGroup, 由资源ID指定，第二个参数是要添加的 Fragment。使用 fragmentTransaction 进行add后, 必须调用 commit() 才能使本次 add 动作生效.


## Managing Fragments

要管理 Activity 中的 Fragment, 需要使用 FragmentManager。在 Activity 中调用 getSupportFragmentManager() 可以获取一个 FragmentManager.

使用 FragmentManager 可以执行以下操作:

* 使用 findFragmentByID() 或 findFragmentBytag(), 来从Activity 中获取一个指定的 fragment.

  Note:

  findFragmentByID() ---for fragments that provide a UI in the activity layout

  findFragmentBytag()---for fragments that do or don't provide a UI


* 使用 popBackStack 将 fragments 从回退栈(the back stack)中 Pop 出来


* 使用 addOnBackStackChangedListener 来监听回退栈中的变化


## Performing Fragment Transactions

在 Activity 中使用 Fragment 的一个重要特性是能够根据用户的交互来添加、删除、替换和执行其他操作。提交给 Activity 的每一个更改都可以称为一个事务,通过 FragmentTransaction 中的Api来执行更改.

我们还可以将每个事务保存到 Activity 管理的回退栈中, 允许用户可以 back 回前一个事务（类似于在 Activity 中向后导航)

可以从 FragmentManager 获取 FragmentTransaction 的实例, 如下所示:

```
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```


__每个事务都是我们希望可以同时执行的一组更改, 可以使用add（）、remove（）和replace（）等方法设置要为给定事务执行的所有更改__。要将事务应用到 Activity, 必须调用commit（）。

在调用commit（）之前, 我们可以调用 addToBackStack（）来将事务添加到回退栈中。这个回退栈是由 Activity 所管理, 并允许用户按后退按钮(back)返回到前一个 Fragment 状态。

Note: 用户按后退按钮(back),相当于将回退栈顶部的 transaction, 移出回退栈, 此时前一个 transaction 将会位于回退栈顶部, Activity 中也会通过  reverse the transaction 来返回上一个 fragment.


例如, 下面介绍如何用另一个 fragment 替换另一个 fragment,并在回退栈中保留以前的状态:

```
// Create new fragment and transaction
Fragment newFragment = new ExampleFragment();
FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();

// Replace whatever is in the fragment_container view with this fragment,
// and add the transaction to the back stack
transaction.replace(R.id.fragment_container, newFragment);
transaction.addToBackStack(null);

// Commit the transaction
transaction.commit();
```

在本例中, newfragment 将会替换由 r.id.fragment_container id 标识的布局容器中当前存在的任何片段（如果有）。通过调用 addToBackStack(), replace transaction 将会被保存到回退栈中, 这样用户可以通过按back按钮反转事务并返回上一个片段。

因此, 用户可以通过按后退按钮来 reverse the transaction, 并返回上一个 fragment。

FragmentActivity 通过 onBackpressed() 自动从回退栈中检索 fragment。

如果向事务添加多个更改, 如再次 add（）或remove() , 然后调用 addToBackStack(), 最后调用commit(), 之前应用的所有更改都将作为单个事务添加到回退栈中, 而 back 按钮会将它们全部反转(reverses)。


向 FragmentTransaction 添加更改的顺序并不重要, 但是:

* 必须是在最后调用commit()

* 如果要将多个 Fragment 添加到同一容器中, 则添加它们的顺序将决定它们在视图层次结构中的显示顺序

如果在执行删除 fragment 的事务时不调用addToBackStack（）, 那么在提交事务后该 fragment 将会被销毁, 且用户无法导航回该事务. If necessary, however, you may call executePendingTransactions() from your UI thread to immediately execute transactions submitted by commit(). Doing so is usually not necessary unless the transaction is a dependency for jobs in other threads.


Caution: 

You can commit a transaction using commit() only prior to the activity saving its state (when the user leaves the activity). If you attempt to commit after that point, an exception is thrown. This is because the state after the commit can be lost if the activity needs to be restored. For situations in which it's okay that you lose the commit, use commitAllowingStateLoss().

即在 Activity 中调用 commit 时序必须在Activity saveInstanceState 之前. 很简单, 如果在 saveInstanceState 之后调用, 该 commit 多对应的事务也就不会被 Activity 的回退栈所保存.

__Activity 以事务为基本元素来管理其 Fragment.__


## Communicating with the Activity

Although a Fragment is implemented as an object that's independent from a FragmentActivity and can be used inside multiple activities, 
__a given instance of a fragment is directly tied to the activity that hosts it.__

即, fragment的一个实例与承载该fragment的Activity是有关联的. 也就是说, 我们在一个 Activity 中使用fragment, 该 fragment 就会被 Activity 所管理.

如何管理呢? 

back Stack!!!!!

具体来说, fragment 可以使用 getActivity（）来访问 fragmentActivity 实例, 并轻松执行诸如在 Activity 布局中查找 View 之类的任务:

```
View listView = getActivity().findViewById(R.id.list);

```

同样, Activty 可以通过使用 findFragmentByID（）或findFragmentBytag（）从 FragmentManager 获取对 fragment 的引用来调用 fragmnet 中的方法。例如:

```
ExampleFragment fragment = (ExampleFragment) getSupportFragmentManager().findFragmentById(R.id.example_fragment);
```


## Creating event callbacks to the activity

In some cases, you might need a fragment to share events or data with the activity and/or the other fragments hosted by the activity. To share data, create a shared ViewModel. If you need to propagate events that cannot be handled with a ViewModel, you can instead define a callback interface inside the fragment and require that the host activity implement it. When the activity receives a callback through the interface, it can share the information with other fragments in the layout as necessary.

例如, 如果一个新闻应用程序在一个 Activity 中有两个fragment, 一个显示文章列表（fragmentA）,另一个显示文章（fragmentB）, 那么fragmentA 必须告诉该 Activity 选择了哪一个列表项, 以便它可以让 fragmentB 显示相关的列表项所对应的内容。在这种情况下, 可以在 fragmentA 中声明一个 OnArticleSelectedListener 接口:

```
public static class FragmentA extends ListFragment {
    ...
    // Container Activity must implement this interface
    public interface OnArticleSelectedListener {
        public void onArticleSelected(Uri articleUri);
    }
    ...
}
```

然后由 host Activity 来实现 OnArticleSelectedListener 接口, 并重写 OnArticleSelected() 方法以通知 fragmentB.

为了确保 host Activity 实现此了接口, fragmentA 的 onAttach() 回调方法（系统在将 fragment 添加到 Activity 时调用该方法). 通过将传递到
 onAttach（）的 Activity 强制转换为 onArticleSelectedListener 的实例:

```
public static class FragmentA extends ListFragment {
    OnArticleSelectedListener listener;
    ...
    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        try {
            listener = (OnArticleSelectedListener) context;
        } catch (ClassCastException e) {
            throw new ClassCastException(context.toString() + " must implement OnArticleSelectedListener");
        }
    }
    ...
}
```

如果 Activity 没有实现此接口, 那么 fragment 将抛出 ClassCastException. 如果 Activity 实现了此接口, mlistener 成员就持有对 Activity 实现 OnArticleSelectedListener 的引用, 此时 FragmentA 就通过调用 OnArticleSelectedListener 接口定义的方法与 Activity 共享事件。例如, 如果 fragmentA 是 ListFragment 的实现类, 则每次用户单击列表项时, 系统都会在片段中调用 OnListItemClick（）, 然后调用 OnArticleSelected（）与 Activity 共享事件:

```
public static class FragmentA extends ListFragment {
    OnArticleSelectedListener listener;
    ...
    @Override
    public void onListItemClick(ListView l, View v, int position, long id) {
        // Append the clicked item's row ID with the content provider Uri
        Uri noteUri = ContentUris.withAppendedId(ArticleColumns.CONTENT_URI, id);
        // Send the event and Uri to the host activity
        listener.onArticleSelected(noteUri);
    }
    ...
}
```


传递给 OnListItemClick() 的ID参数是被单击项的ID, Activity （或其他Fragment）使用该 ID 从应用程序的 ContentProvider 获取项目。



## Handling the Fragment Lifecycle


管理 Fragment 的生命周期非常类似于管理 Activity 的生命周期。与 Activity 一样, Fragment可以有三种状态:

* Resumed

The fragment is visible in the running activity.

* Paused

Another activity is in the foreground and has focus, but the activity in which this fragment lives is still visible (the foreground activity is partially transparent or doesn't cover the entire screen).

* Stopped

Fragment 不可见, host Activtity 已经 stoped, 或者Fragment 已从Activity 中删除, 但是被添加到回退栈中。A stopped fragment is still alive (all state and member information is retained by the system). However, it is no longer visible to the user and is killed if the activity is killed.

与 Activity 一样, 可以使用 OnSaveInstanceState（bundle）、ViewModel和持久本地存储(persistent local storage)来保存 fragment 的状态.

Activity 和 Fragment 在生命周期中最显著的区别是如何将其存储在各自的回退栈中。默认情况下, Activity 会被放置到由系统在停止时管理的Activity 回退栈中。对于 Fragment, 只有我们在执行事务期间显示的调用 addToBackStack() 来保存实例时, Host Activity 才会将 Fragment 放入 Activity 管理的回退栈中。


Caution: If you need a Context object within your Fragment, you can call getContext(). However, be careful to call getContext() only when the fragment is attached to an activity. When the fragment isn't attached yet, or was detached during the end of its lifecycle, getContext() returns null.

更好的方法就是在 fragment Attach Activity 时去存储该 Activity 引用; 在Dettach时, 去移除该引用. 中间使用该引用, 要提前判空.


## Coordinating with the activity lifecycle


 Fragment 所在的 Activity 的生命周期直接影响 Fragment 的生命周期, 因此 Activity 的每个生命周期回调都会导致对每个 Fragment 的类似回调。例如, 当 Activity 收到onPause()时, Activity 中的每个 Fragment 都会收到 onPause()回调。

然而, Fragment 有一些额外的生命周期回调, 它们处理与 Activity 的独特交互, 以便执行诸如构建和销毁 Fragment 的UI之类的操作。这些额外的回调方法是:

* onAttach()

 Called when the fragment has been associated with the activity (the Activity is passed in here)

* onCreateView()

 Called to create the view hierarchy associated with the fragment

* onActivityCreated()

 Called when the activity's onCreate() method has returned

* onDestroyView()

 Called when the view hierarchy associated with the fragment is being removed

* onDetach()

 Called when the fragment is being disassociated from the activity.


![activity_fragment_lifecycle](https://github.com/xianfeng92/android-code-read/blob/master/images/activity_fragment_lifecycle.png)


The flow of a fragment's lifecycle, as it is affected by its host activity, is illustrated by figure 3. In this figure, you can see how each successive state of the activity determines which callback methods a fragment may receive. For example, when the activity has received its onCreate() callback, a fragment in the activity receives no more than the onActivityCreated() callback.

Once the activity reaches the resumed state, you can freely add and remove fragments to the activity. Thus, only while the activity is in the resumed state can the lifecycle of a fragment change independently.

只有在 Activity Resume 状态时, fragment 的生命周期的回调才能说是独立的.但是, 当 Activity 离开 Resume 状态时,  Activity 会再次推动 Fragment 的生命周期(Fragment 的生命周期会受host Activity的影响)。


[Fragments](https://developer.android.google.cn/guide/components/fragments)
