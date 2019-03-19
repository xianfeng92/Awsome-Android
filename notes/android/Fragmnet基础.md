
## 基本概念

Fragment，简称碎片，是 Android 3.0（API 11）提出的，为了兼容低版本，support-v4库中也开发了一套Fragment API，最低兼容Android 1.6。
而如果要使用 support库的 Fragment，Activity 必须要继承 FragmentActivity（AppCompatActivity是FragmentActivity的子类）。

* Fragment是依赖于Activity的，不能独立存在的。

* 一个Activity里可以有多个Fragment。

* 一个Fragment可以被多个Activity重用。

* Fragment有自己的生命周期，并能接收输入事件。

* 我们能在Activity运行时动态地添加或删除Fragment。


### Fragment的优势有以下几点：

* 模块化（Modularity）：我们不必把所有代码全部写在Activity中，而是把代码写在各自的Fragment中。

* 可重用（Reusability）：多个Activity可以重用一个Fragment。

* 可适配（Adaptability）：根据硬件的屏幕尺寸、屏幕方向，能够方便地实现不同的布局，这样用户体验更好。


### Fragment家族常用的API

* Fragment：Fragment的基类，任何创建的Fragment都需要继承该类。

* FragmentManager：管理和维护Fragment。它是抽象类，具体的实现类是FragmentManagerImpl。

  FragmentManager栈视图如下：


![](https://github.com/xianfeng92/android-code-read/blob/master/images/fragmentManager%E6%A0%88%E8%A7%86%E5%9B%BE.png)



1. 对于宿主 fragmentActivity，getSupportFragmentManager()获取的 FragmentActivity 的 FragmentManager 对象

2. 对于 Fragment，getFragmentManager()是获取的是父 Fragment(如果没有，则是FragmentActivity)的FragmentManager对象

3. getChildFragmentManager()是获取自己的 FragmentManager 对象。


* FragmentTransaction：对Fragment的添加、删除等操作都需要通过事务方式进行。它是抽象类，具体的实现类是BackStackRecord。

* Nested Fragment（Fragment内部嵌套Fragment的能力）是Android 4.2提出的，support-fragment库可以兼容到1.6。通过getChildFragmentManager()能够获得管理子Fragment的FragmentManager，在子Fragment中可以通过getParentFragment()

* 获取FragmentManage的方式：

 getFragmentManager() // v4中，getSupportFragmentManager

* 主要的操作都是 FragmentTransaction 的方法:

FragmentTransaction transaction = fm.benginTransatcion();//开启一个事务

## 生命周期

Fragment必须是依存与Activity而存在的，因此Activity的生命周期会直接影响到Fragment的生命周期。官网这张图很好的说明了两者生命周期的关系：


![](https://github.com/xianfeng92/android-code-read/blob/master/images/2952813-0f4f821975d72317.png)


解释如下：

* onAttach()：Fragment和Activity相关联时调用。可以通过该方法获取Activity引用，还可以通过getArguments()获取参数

* onCreate()：Fragment被创建时调用

* onCreateView()：创建Fragment的布局

* onActivityCreated()：当Activity完成onCreate()时调用

* onStart()：当Fragment可见时调用

* onResume()：当Fragment可见且可交互时调用

* onPause()：当Fragment不可交互但可见时调用

* onStop()：当Fragment不可见时调用

* onDestroyView()：当Fragment的UI从视图结构中移除时调用

* onDestroy()：销毁Fragment时调用

* onDetach()：当Fragment和Activity解除关联时调用

上面的方法中，只有onCreateView()在重写时不用写super方法，其他都需要。


FragmentTransaction有一些基本方法，下面给出调用这些方法时，Fragment生命周期的变化：

* add(): onAttach()->…->onResume()

* remove(): onPause()->…->onDetach()

* replace(): 相当于旧 Fragment 调用remove()，新Fragment调用add()

* show(): 不调用任何生命周期方法，调用该方法的前提是要显示的 Fragment已经被添加到容器，只是纯粹把Fragment UI的setVisibility为true

* hide(): 不调用任何生命周期方法，调用该方法的前提是要显示的Fragment已经被添加到容器，只是纯粹把Fragment UI的setVisibility为false

* detach(): onPause()->onStop()->onDestroyView()。UI从布局中移除，但是仍然被 FragmentManager 管理

* attach(): onCreateView()->onStart()->onResume()



## Back Stack 的实现原理

我们知道Activity有任务栈，用户通过startActivity将Activity加入栈，点击返回按钮将Activity出栈。__Fragment也有类似的栈，称为回退栈（Back Stack）__，
回退栈是由 FragmentManager 管理的。默认情况下，Fragment事务是不会加入回退栈的，如果想将Fragment事务加入回退栈，则可以加入addToBackStack("")。
如果没有加入回退栈，则用户点击返回按钮会直接将Activity出栈；如果加入了回退栈，则用户点击返回按钮会回滚Fragment事务。

下面这个代码的功能就是将Fragment加入Activity中，内部实现为：__创建一个BackStackRecord对象，该对象记录了这个事务的全部操作轨迹__（这里只做了一次add操作，并且加入回退栈），
随后将该对象提交到FragmentManager的执行队列中，等待执行。

```
getSupportFragmentManager().beginTransaction()
    .add(R.id.container, f1, "f1")
    .addToBackStack("")
    .commit();

    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }

```

BackStackRecord类的定义如下:

```
final class BackStackRecord extends FragmentTransaction implements BackStackEntry, OpGenerator 

```

从定义可以看出，BackStackRecord 有三重含义：

* 继承了FragmentTransaction，即是事务，保存了整个事务的全部操作轨迹。

* 实现了BackStackEntry，作为回退栈的元素，正是因为该类拥有事务全部的操作轨迹，因此在popBackStack()时能回退整个事务。

* 继承了Runnable，即被放入FragmentManager执行队列，等待被执行。

先看第一层含义，getSupportFragmentManager.beginTransaction()返回的就是BackStackRecord对象。

BackStackRecord类包含了一次事务的整个操作轨迹，是以数组列表形式存在的，元素是Op类，表示其中某个操作，定义如下：

```
    static final class Op {
        int cmd;//操作是add或remove或replace或hide或show等
        Fragment fragment;////对哪个Fragment对象做操作
        int enterAnim;
        int exitAnim;
        int popEnterAnim;
        int popExitAnim;

        Op() {
        }

        Op(int cmd, Fragment fragment) {
            this.cmd = cmd;
            this.fragment = fragment;
        }
    }

```

```
    public FragmentTransaction add(int containerViewId, Fragment fragment, @Nullable String tag) {
        this.doAddOp(containerViewId, fragment, tag, 1);
        return this;
    }

```

```

    private void doAddOp(int containerViewId, Fragment fragment, @Nullable String tag, int opcmd) {
        Class fragmentClass = fragment.getClass();
        int modifiers = fragmentClass.getModifiers();
        if (fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers) || fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers)) {
            throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName() + " must be a public static class to be  properly recreated from" + " instance state.");
        } else {
            fragment.mFragmentManager = this.mManager;
            if (tag != null) {
                if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                    throw new IllegalStateException("Can't change tag of fragment " + fragment + ": was " + fragment.mTag + " now " + tag);
                }

                fragment.mTag = tag;
            }

            if (containerViewId != 0) {
                if (containerViewId == -1) {
                    throw new IllegalArgumentException("Can't add fragment " + fragment + " with tag " + tag + " to container view with no id");
                }

                if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                    throw new IllegalStateException("Can't change container ID of fragment " + fragment + ": was " + fragment.mFragmentId + " now " + containerViewId);
                }

                fragment.mContainerId = fragment.mFragmentId = containerViewId;
            }

            this.addOp(new BackStackRecord.Op(opcmd, fragment));
        }
    }

```

addOp()是将创建好的Op对象加入数组列表（mOps），定义如下

```
    void addOp(BackStackRecord.Op op) {
        this.mOps.add(op);
        op.enterAnim = this.mEnterAnim;
        op.exitAnim = this.mExitAnim;
        op.popEnterAnim = this.mPopEnterAnim;
        op.popExitAnim = this.mPopExitAnim;
    }

```


addToBackStack(“”)会是将mAddToBackStack变量记为true，在commit()中会用到该变量。

```
    public FragmentTransaction addToBackStack(@Nullable String name) {
        if (!this.mAllowAddToBackStack) {
            throw new IllegalStateException("This FragmentTransaction is not allowed to be added to the back stack.");
        } else {
            this.mAddToBackStack = true;//将mAddToBackStack变量记为true
            this.mName = name;
            return this;
        }
    }

```

commit()是异步的，即不是立即生效的，但是后面会看到整个过程还是在主线程完成，只是把事务的执行扔给主线程的Handler，commit()内部是commitInternal()，实现如下：

```
    int commitInternal(boolean allowStateLoss) {
        if (this.mCommitted) {
            throw new IllegalStateException("commit already called");
        } else {
            if (FragmentManagerImpl.DEBUG) {
                Log.v("FragmentManager", "Commit: " + this);
                LogWriter logw = new LogWriter("FragmentManager");
                PrintWriter pw = new PrintWriter(logw);
                this.dump("  ", (FileDescriptor)null, pw, (String[])null);
                pw.close();
            }

            this.mCommitted = true;
            if (this.mAddToBackStack) {
                this.mIndex = this.mManager.allocBackStackIndex(this); //mAddToBackStack为true时，给该BackStackRecord分配一个index值。
            } else {
                this.mIndex = -1;
            }

            this.mManager.enqueueAction(this, allowStateLoss);
            return this.mIndex;
        }
    }

```

如果mAddToBackStack为true，则调用 allocBackStackIndex(this)将事务添加进回退栈，__FragmentManager类的变量 ArrayList<BackStackRecord> mBackStackIndices 就是回退栈__。

```
    public int allocBackStackIndex(BackStackRecord bse) {
        synchronized(this) {
            int index;
            if (this.mAvailBackStackIndices != null && this.mAvailBackStackIndices.size() > 0) {
                index = (Integer)this.mAvailBackStackIndices.remove(this.mAvailBackStackIndices.size() - 1);
                if (DEBUG) {
                    Log.v("FragmentManager", "Adding back stack index " + index + " with " + bse);
                }

                this.mBackStackIndices.set(index, bse);
                return index;
            } else {
                if (this.mBackStackIndices == null) {
                    this.mBackStackIndices = new ArrayList();
                }

                index = this.mBackStackIndices.size();
                if (DEBUG) {
                    Log.v("FragmentManager", "Setting back stack index " + index + " to " + bse);
                }

                this.mBackStackIndices.add(bse);//将BackStackRecord添加到回退栈中
                return index;
            }
        }
    }

```

与addToBackStack()对应的是popBackStack()，有以下几种变种：

* popBackStack()：将回退栈的栈顶弹出，并回退该事务。

* popBackStack(String name, int flag)：name为addToBackStack(String name)的参数，通过name能找到回退栈的特定元素，flag可以为0或者FragmentManager.POP_BACK_STACK_INCLUSIVE，
0表示只弹出该元素以上的所有元素，POP_BACK_STACK_INCLUSIVE表示弹出包含该元素及以上的所有元素。这里说的弹出所有元素包含回退这些事务。

* popBackStack()是异步执行的，是丢到主线程的MessageQueue执行，popBackStackImmediate()是同步版本。


## Fragment通信

### Fragment向Activity传递数据

* 在 Fragment 中可以通过 getActivity 得到当前绑定的 Activity 的实例，然后进行操作。

* 通过接口回调的方式。

首先，在Fragment中定义接口，并让Activity实现该接口：

```
public interface OnFragmentInteractionListener {    void onItemClick(String str);  //将str从Fragment传递给Activity}

```

在Fragment的onAttach()中，将参数Context强转为OnFragmentInteractionListener对象：

```
public void onAttach(Context context) {
    super.onAttach(context);
        if (context instanceof OnFragmentInteractionListener) {
        mListener = (OnFragmentInteractionListener) context;
    } else {
                throw new RuntimeException(context.toString()
                + " must implement OnFragmentInteractionListener");
    }
}

```

并在Fragment合适的地方调用mListener.onItemClick("hello")，将”hello”从Fragment传递给Activity。

上面两种通信方式都是值得推荐的，随便选择一种自己喜欢的。虽然Fragment和Activity可以通过getActivity与findFragmentByTag或者findFragmentById，进行任何操作，
甚至在Fragment里面操作另外的Fragment，但是没有特殊理由是绝对不提倡的。__Activity担任的是Fragment间类似总线一样的角色，应当由它决定Fragment如何操作__。
另外虽然Fragment不能响应Intent打开，但是Activity可以，Activity可以接收Intent，然后根据参数判断显示哪个Fragment。


## Activity向Fragment传递数据

Activity向Fragment传递数据比较简单，获取Fragment对象，并调用Fragment的方法即可，比如要将一个字符串传递给Fragment，则在Fragment中定义方法：

```
public void setString(String str) { 
    this.str = str;
}
```

并在Activity中调用fragment.setString("hello")即可。

## Fragment之间通信

由于Fragment之间是没有任何依赖关系的，因此如果要进行Fragment之间的通信，建议通过Activity作为中介，不要Fragment之间直接通信。

## Fragment重叠问题

相信大家在使用Fragment的过程中，肯定碰到过Fragment重叠的问题，重启应用就好了。然而原因是什么呢？

首先，Android管理Fragment有两种方式，使用add、hide、show的方式和replace方式，两种方式各有优缺点。

* replace方式 

如果使用这种方式，是可以避免重叠的问题，但是每次replace会把生命周期全部执行一遍，如果在这些生命周期函数里拉取数据的话，就会不断重复的加载刷新数据，所以我们并不推荐使用这种方式。

* add、hide、show的方式 

虽然这种方式避免了可能重复加载刷新数据的问题，但是会出现重叠的问题。

### 为什么会出现Fragment的重叠？

当系统内存不足，Fragment 的宿主 Activity 回收的时候，Fragment 的实例并没有随之被回收。Activity 被系统回收时，会主动调用 onSaveInstance() 方法来保存视图层（View Hierarchy），
所以当 Activity 再次被重建时，之前被实例化过的 Fragment 依然会出现在 Activity 中(而Fragment默认的是show状态)，此时的 FragmentTransaction 中的相当于又再次 add 了 fragment 进去的，所以就出现了重叠。

```
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        this.markFragmentsCreated();
        Parcelable p = this.mFragments.saveAllState();//保存fragment状态
         ...省略
    }


    protected void onCreate(@Nullable Bundle savedInstanceState) {
        this.mFragments.attachHost((Fragment)null);
        super.onCreate(savedInstanceState);
        FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
        if (nc != null && nc.viewModelStore != null && this.mViewModelStore == null) {
            this.mViewModelStore = nc.viewModelStore;
        }

        if (savedInstanceState != null) {
            Parcelable p = savedInstanceState.getParcelable("android:support:fragments");
            this.mFragments.restoreAllState(p, nc != null ? nc.fragments : null);// 恢复fragment的状态
           ...省略
        this.mFragments.dispatchCreate();
    }
```

从上面源码可以看出，FragmentActivity确实是帮我们保存了Fragment的状态，并且在页面重启后会帮我们恢复！

其中的mFragments是FragmentController，它是一个Controller，内部通过FragmentHostCallback间接控制FragmentManagerImpl。

相关代码如下：

```
public class FragmentController {
    private final FragmentHostCallback<?> mHost;

    public Parcelable saveAllState() {
        return mHost.mFragmentManager.saveAllState();
    }

    public void restoreAllState(Parcelable state, List<Fragment> nonConfigList) {
        mHost.mFragmentManager.restoreAllState(state, nonConfigList);
    }
}

public abstract class FragmentHostCallback<E> extends FragmentContainer {
    final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();
}

```

通过上面代码可以看出FragmentController通过FragmentHostCallback里的FragmentManagerImpl对象来控制恢复工作。

那么FragmentManagerImpl到底做了什么？

```
final class FragmentManagerImpl extends FragmentManager {
    Parcelable saveAllState() {
        ...省略 详细保存过程
        FragmentManagerState fms = new FragmentManagerState();
        fms.mActive = active;
        fms.mAdded = added;
        fms.mBackStack = backStack;
        return fms;
    }

    void restoreAllState(Parcelable state, List<Fragment> nonConfig) {
        // 恢复核心代码
        FragmentManagerState fms = (FragmentManagerState)state;
        FragmentState fs = fms.mActive[i];
        if (fs != null) {
            Fragment f = fs.instantiate(mHost, mParent);
        ｝
    }
}
```

我们通过saveAllState()看到了关键的保存代码，原来是是通过FragmentManagerState来保存Fragment的状态、所处Fragment栈下标、回退栈状态等。

而在restoreAllState()恢复时，通过FragmentManagerState里的FragmentState的instantiate()方法恢复了Fragment。

我们看下FragmentManagerState：

```
final class FragmentManagerState implements Parcelable {
    FragmentState[] mActive;           // Fragment状态
    int[] mAdded;                      // 所处Fragment栈下标
    BackStackState[] mBackStack;       // 回退栈状态
    ...
}

```
我们只看FragmentState，它也实现了Parcelable，保存了Fragment的类名、下标、id、Tag、ContainerId以及Arguments等数据：

```
final class FragmentState implements Parcelable {
    final String mClassName;//保存了Fragment的类名
    final int mIndex; //下标
    final boolean mFromLayout;
    final int mFragmentId;//id
    final int mContainerId; //ContainerId
    final String mTag;
    final boolean mRetainInstance;
    final boolean mDetached;
    final Bundle mArguments;
    ...

    //  在FragmentManagerImpl的restoreAllState()里被调用
    public Fragment instantiate(FragmentHostCallback host, Fragment parent) {
        ...省略
        mInstance = Fragment.instantiate(context, mClassName, mArguments);
    }
}

```

__为什么在页面重启后会发生Fragment的重叠？ 其实答案已经很明显了，根据上面的源码分析，我们会发现FragmentState里没有Hidden状态的字段！__

而Hidden状态对应Fragment中的mHidden，该值默认false...

```
public class Fragment ... {
    boolean mHidden;
}
```

我想你应该明白了，在以add方式加载Fragment的场景下，系统在恢复Fragment时，mHidden＝false，即show状态，这样在页面重启后，Activity内的Fragment都是以show状态显示的，
而如果你不进行处理，那么就会发生Fragment重叠现象！


### 为什么重复replace|add Fragment 或者 使用show , hide控制Fragment会导致重叠？

* 重复replace|add Fragment

我们知道加载Fragment有2种方式：replace()和add()。不管哪种方式，重复加载Fragment都会导致重叠，这个很好理解，你加载同一个Fragment 2次当然会重叠了；
问题是我们在哪里重复加载了Fragment？

一般情况下，我们会在Activity的onCreate()里或者Fragment的onCreateView()里加载根Fragment，如果在这里没有进行页面重启的判断的话，就可能导致重复加载Fragment引起重叠。
正确的写法应该是：

```
public class MainActivity ... {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        ...
        // 这里一定要在save为null时才加载Fragment，Fragment中onCreateView等生命周里加载根子Fragment同理
        // 因为在页面重启时，Fragment会被保存恢复，而此时再加载Fragment会重复加载，导致重叠
        if(findFragmentByTag(rootFragmentTag) == null){
              // 正常情况下去 加载根Fragment 
        }
    }
}
```
这里一定要在saveInstanceState==null时才加载Fragment，因为经过上面的分析，在页面重启时，Fragment的状态会被保存恢复，而此时再加载Fragment会重复加载，
就导致栈已经有该Fragment的情况下，再加载一该Fragment，从而导致重叠！


# 参考

[Android Fragment 真正的完全解析](https://blog.csdn.net/lmj623565791/article/details/37992017)
[Android基础：Fragment，看这篇就够了](https://mp.weixin.qq.com/s/dUuGSVhWinAnN9uMiBaXgw)
[关于Fragment重叠问题分析和解决](https://blog.csdn.net/whitley_gong/article/details/51987911)
[Fragment全解析系列（一）：那些年踩过的坑](https://www.jianshu.com/p/d9143a92ad94)
[从源码角度分析，为什么会发生Fragment重叠？](https://www.jianshu.com/p/78ec81b42f92)
[9行代码让你App内的Fragment对重叠说再见](https://www.jianshu.com/p/c12a98a36b2b)
