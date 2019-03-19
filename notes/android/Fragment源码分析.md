
先上结论：

1. FragmentActivity 是具有支持fragment功能的最底层的 activity, 其他什么 AppCompatActivity 都是他的子类！

2. __FragmentActivity 主要负责就是生命周期的转发__，比如 onCreate onResume onDestroy 等等，这就是为什么 activity 和 fragment 状态能统一的原因了！

当然了，分发的原因就是因为 fragmentActivity 对象持有一个 fragmentController 的实例！

其实讲白了，fragmentController 就是因为它自己有一个 fragmentHostCallback，然后这个 fragmentHostCallback 还持有了 fragmentManagerImpl,

最终由 fragmentManagerImpl 的 moveToState 方法完成 fragment 生命周期的转换。


3. fragementHostCallback 持有了 activity 的很多资源，context，handler 是其中最主要的2个。fragmentManagerImpl 就是因为拿到了 activty 的这2个资源，所以才能和 activty 互相通信的！

   为什么 fragmentManagerImpl 可以拿到 activty 的 context 和 handler？

   因为其为 fragementHostCallback 的内部类

4. fragmentMangerImple 就是 fragmentManger 的具体实现类。moveToState方法就是在这个里面实现的 

5. FragmentTransition 也是个抽象类，他主要就是提供对外的接口函数的 add replace move 这种。BackStackRecord 就是它的具体实现类。还额外实现了 OpGenerator 接口，该接口只有一个
  generateOps(ArrayList<BackStackRecord> var1, ArrayList<Boolean> var2) 方法。

6. BackStackRecord 里面会有个 executeOps() 方法。这个方法就是根据不同的操作(所谓操作就是OP.CMD的那个值)来分发不同的事件，从而调用 fragmentManager的各种转换 fragment 生命周期的方法！


## 源码如下

一般在activity中用如下方式动态添加一个fragment：

```
        FragmentManager fragmentManager = getSupportFragmentManager();
        FragmentTransaction tx = fragmentManager.beginTransaction();
        //当Activity因为配置发生改变（屏幕旋转）或者内存不足被系统杀死，造成重新创建时，我们的fragment会被保存下来
        // 但是会创建新的FragmentManager，新的FragmentManager会首先会去获取保存下来的fragment队列，重建fragment队列，从而恢复之前的状态。
        if (fragmentOne == null){
            fragmentOne = FragmentOne.newInstance("fragment one");
            // 布局的id是告知FragmentManager，此fragment的位置
            // 另一方面是此fragment的唯一标识；就像我们上面通过fm.findFragmentById(R.id.id_fragment_container)查找
            tx.add(R.id.content,fragmentOne,"one");
            tx.commit();
        }
```

这段代码相信大家都很熟悉了，我们就来一步步跟进去看看：

```
    public FragmentManager getSupportFragmentManager() {
        //到这里能发现是 mFragments 返回给我们的 FragmentManager
        return this.mFragments.getSupportFragmentManager();
    }

     //继续往下跟就会发现 mFragments 是由 FragmentController 的 createController 函数构造出来的一个对象，
    //并且这个函数需要传进去一个 HostCallBack 的对象
    final FragmentController mFragments = FragmentController.createController(new FragmentActivity.HostCallbacks());

   //下面的代码就来自于FragmentController 这个类
   private final FragmentHostCallback<?> mHost;

    private FragmentController(FragmentHostCallback<?> callbacks) {
        this.mHost = callbacks;
    }

    //从这个函数就能看出来HostCallbacks 这个类肯定是FragmentHostCallback的子类了
    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }

    //所以这个getSupportFragmentManager返回的就是 FragmentManager这个对象，并且这个对象是 mHost 的 getFragmentManagerImpl 函数返回的。
    //这里结合构造函数一看就明白了，这个mHost就是我们在activity代码里面，传进去的HostCallbacks这个对象来帮助初始化的
    public FragmentManager getSupportFragmentManager() {
        return this.mHost.getFragmentManagerImpl();
    }

      // 下面的代码在FragmentActivity里
     // 这个地方一目了然 果然我们这个 HostCallbacks 这个类是继承自 FragmentHostCallback 的，并且能看出来，我们这里把 activity 的引用也传进去了。
    // 所以能马上得出一个结论就是一个 activity 对应着一个 HostCallback 对象, 这个对象持有本身这个activity的引用。传进去以后就代表FragmentController
    //这个类的成员mHost 也持有了activity的引用
    class HostCallbacks extends FragmentHostCallback<FragmentActivity> {
        public HostCallbacks() {
            super(FragmentActivity.this);
        }

    
    //到这里就能看到FragmentHostCallback 持有了acitivty的引用 并且连activity的handler都一并持有！
    FragmentHostCallback(@Nullable Activity activity, @NonNull Context context, @NonNull Handler handler, int windowAnimations) {
        this.mFragmentManager = new FragmentManagerImpl();
        this.mActivity = activity;
        this.mContext = (Context)Preconditions.checkNotNull(context, "context == null");
        this.mHandler = (Handler)Preconditions.checkNotNull(handler, "handler == null");
        this.mWindowAnimations = windowAnimations;
    }
```

上面初步分析了 getSupportFragmentManager 这个方法的由来。那继续看这个方法到底是返回的什么？

```
    //下面的代码来源自抽象类FragmentHostCallback
    FragmentManagerImpl getFragmentManagerImpl() {
        return this.mFragmentManager;
    }

    //所以就能看出来 我们在activity中调用的 getSupportFragmentManager 这个方法最终返回的是 FragmentManagerImpl 这个类的对象了
    FragmentHostCallback(@Nullable Activity activity, @NonNull Context context, @NonNull Handler handler, int windowAnimations) {
        this.mFragmentManager = new FragmentManagerImpl();
        this.mActivity = activity;
        this.mContext = (Context)Preconditions.checkNotNull(context, "context == null");
        this.mHandler = (Handler)Preconditions.checkNotNull(handler, "handler == null");
        this.mWindowAnimations = windowAnimations;
    }

```

由上可知，FragmentManagerImpl 是在 FragmentHostCallback 中实例化的，且 FragmentHostCallback 持有对 fragmentActivity 以及其对应的 Handler 和 Context 的引用。


再进去看看 这个对象的 beginTransaction 方法返回的是什么:

```
    // 下面的代码来源自 FragmentManagerImpl
    public FragmentTransaction beginTransaction() {
        //可以看出来 返回的是BackStackRecord 这个类的对象
        return new BackStackRecord(this);
    }

```

```
// 可以看一下BackStackRecord 是 FragmentTransaction 的子类，并且实现了 BackStackEntry, OpGenerator 这两个接口
final class BackStackRecord extends FragmentTransaction implements BackStackEntry, OpGenerator {
    static final String TAG = "FragmentManager";
    final FragmentManagerImpl mManager;
    ...


// Op 为 BackStackRecord 的一个静态内部类

    static final class Op {
        int cmd;
        Fragment fragment;
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

所以 begintranscation 返回的最终就是 backstackrecord 对象了。

我们继续看看这个对象的操作:

```
    //以下代码来源自 backstackrecord 源码
    public FragmentTransaction replace(int containerViewId, Fragment fragment) {
        return this.replace(containerViewId, fragment, (String)null);
    }

    public FragmentTransaction replace(int containerViewId, Fragment fragment, @Nullable String tag) {
        //你如果没有传进去一个有效的id的话 异常就会在这里出现了
        if (containerViewId == 0) {
            throw new IllegalArgumentException("Must use non-zero containerViewId");
        } else {
            //最终都是调用的 doAdddop 这个方法来完成操作的
            this.doAddOp(containerViewId, fragment, tag, 2);
            return this;
        }
    }

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


    void addOp(BackStackRecord.Op op) {
        this.mOps.add(op);
        op.enterAnim = this.mEnterAnim;
        op.exitAnim = this.mExitAnim;
        op.popEnterAnim = this.mPopEnterAnim;
        op.popExitAnim = this.mPopExitAnim;
    }

    ArrayList<BackStackRecord.Op> mOps = new ArrayList();

```

最终其实就是将对应的 fragment 及相关信息（containerViewId，opcmd）封装后，放进 backstackrecord 对象的一个 mOps（数组列表） 中，类似，add 和 remove 方法也是对这个mOps的一些操作。


再来看看commit方法：

```
    //以下代码来源自 backstackrecord 源码
    public int commit() {
        return this.commitInternal(false);
    }

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
                this.mIndex = this.mManager.allocBackStackIndex(this);
            } else {
                this.mIndex = -1;
            }

             // 这个对象就是 final FragmentManagerImpl mManager
            // 我们在调用begin函数的时候传进去一个this指针就是用来初始化它的
            this.mManager.enqueueAction(this, allowStateLoss);
            return this.mIndex;
        }
    }

```


```
     // 下面是 FragmentManagerImpl 的源码了
    // 这个 mHost 前文介绍过持有了activity的引用，所以这里你看就是用 activity 的 handler 去执行了 mExecCommit 
    // 注意是在activity的主线程去执行的 mExecCommit
    public void enqueueAction(FragmentManagerImpl.OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            this.checkStateLoss();
        }

        synchronized(this) {
            if (!this.mDestroyed && this.mHost != null) {
                if (this.mPendingActions == null) {
                    this.mPendingActions = new ArrayList();
                }

                this.mPendingActions.add(action);
                this.scheduleCommit();
            } else if (!allowStateLoss) {
                throw new IllegalStateException("Activity has been destroyed");
            }
        }
    }

    void scheduleCommit() {
        synchronized(this) {
            boolean postponeReady = this.mPostponedTransactions != null && !this.mPostponedTransactions.isEmpty();
            boolean pendingReady = this.mPendingActions != null && this.mPendingActions.size() == 1;
            if (postponeReady || pendingReady) {
                this.mHost.getHandler().removeCallbacks(this.mExecCommit);
                this.mHost.getHandler().post(this.mExecCommit);
            }
        }
    }

    //这个线程执行的 execPendingActions 就是这个方法 这个方法也是在 FragmentManagerImpl 里的。并不在activity里。
    //所以commit操作就是最终让activity的主线程去执行了 FragmentManagerImpl execPendingActions方法
    Runnable mExecCommit = new Runnable() {
        public void run() {
            FragmentManagerImpl.this.execPendingActions();
        }
    };


    public boolean execPendingActions() {
        this.ensureExecReady(true);

        boolean didSomething;
        for(didSomething = false; this.generateOpsForPendingActions(this.mTmpRecords, this.mTmpIsPop); didSomething = true) {
            this.mExecutingActions = true;

            try {
                this.removeRedundantOperationsAndExecute(this.mTmpRecords, this.mTmpIsPop);
            } finally {
                this.cleanupExec();
            }
        }

        this.doPendingDeferredStart();
        this.burpActive();
        return didSomething;
    }

    private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
        if (records != null && !records.isEmpty()) {
            if (isRecordPop != null && records.size() == isRecordPop.size()) {
                this.executePostponedTransaction(records, isRecordPop);
                int numRecords = records.size();
                int startIndex = 0;

                for(int recordNum = 0; recordNum < numRecords; ++recordNum) {
                    boolean canReorder = ((BackStackRecord)records.get(recordNum)).mReorderingAllowed;
                    if (!canReorder) {
                        if (startIndex != recordNum) {
                            this.executeOpsTogether(records, isRecordPop, startIndex, recordNum);
                        }

                        int reorderingEnd = recordNum + 1;
                        if ((Boolean)isRecordPop.get(recordNum)) {
                            while(reorderingEnd < numRecords && (Boolean)isRecordPop.get(reorderingEnd) && !((BackStackRecord)records.get(reorderingEnd)).mReorderingAllowed) {
                                ++reorderingEnd;
                            }
                        }

                        this.executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
                        startIndex = reorderingEnd;
                        recordNum = reorderingEnd - 1;
                    }
                }

                if (startIndex != numRecords) {
                    this.executeOpsTogether(records, isRecordPop, startIndex, numRecords);
                }

            } else {
                throw new IllegalStateException("Internal error with the back stack records");
            }
        }
    }


    private void executeOpsTogether(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        boolean allowReordering = ((BackStackRecord)records.get(startIndex)).mReorderingAllowed;
        boolean addToBackStack = false;
        if (this.mTmpAddedFragments == null) {
            this.mTmpAddedFragments = new ArrayList();
        } else {
            this.mTmpAddedFragments.clear();
        }

        this.mTmpAddedFragments.addAll(this.mAdded);
        Fragment oldPrimaryNav = this.getPrimaryNavigationFragment();

        int postponeIndex;
        for(postponeIndex = startIndex; postponeIndex < endIndex; ++postponeIndex) {
            BackStackRecord record = (BackStackRecord)records.get(postponeIndex);
            boolean isPop = (Boolean)isRecordPop.get(postponeIndex);
            if (!isPop) {
                oldPrimaryNav = record.expandOps(this.mTmpAddedFragments, oldPrimaryNav);
            } else {
                oldPrimaryNav = record.trackAddedFragmentsInPop(this.mTmpAddedFragments, oldPrimaryNav);
            }

            addToBackStack = addToBackStack || record.mAddToBackStack;
        }

        this.mTmpAddedFragments.clear();
        if (!allowReordering) {
            FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, endIndex, false);
        }

        executeOps(records, isRecordPop, startIndex, endIndex);// commit操作 最终执行的实际上是我们 backstackrecord 这个类里的 executeOps 方法
        postponeIndex = endIndex;
        if (allowReordering) {
            ArraySet<Fragment> addedFragments = new ArraySet();
            this.addAddedFragments(addedFragments);
            postponeIndex = this.postponePostponableTransactions(records, isRecordPop, startIndex, endIndex, addedFragments);
            this.makeRemovedFragmentsInvisible(addedFragments);
        }

        if (postponeIndex != startIndex && allowReordering) {
            FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, postponeIndex, true);
            this.moveToState(this.mCurState, true);
        }

        for(int recordNum = startIndex; recordNum < endIndex; ++recordNum) {
            BackStackRecord record = (BackStackRecord)records.get(recordNum);
            boolean isPop = (Boolean)isRecordPop.get(recordNum);
            if (isPop && record.mIndex >= 0) {
                this.freeBackStackIndex(record.mIndex);
                record.mIndex = -1;
            }

            record.runOnCommitRunnables();
        }

        if (addToBackStack) {
            this.reportBackStackChanged();
        }

    }

```

一直到这里 我们就知道，commit 操作最终执行的实际上是我们 backstackrecord 这个类里的 executeOps 方法。

```
 void executeOps() {
        int numOps = this.mOps.size();

        for(int opNum = 0; opNum < numOps; ++opNum) {
            BackStackRecord.Op op = (BackStackRecord.Op)this.mOps.get(opNum);
            Fragment f = op.fragment;
            if (f != null) {
                f.setNextTransition(this.mTransition, this.mTransitionStyle);
            }

            switch(op.cmd) {
            case 1:
                f.setNextAnim(op.enterAnim);
                this.mManager.addFragment(f, false);// 根据对应的 op.cmd, fragmentManager 来管理 fragment 的一些状态。
                break;
            case 2:
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
            case 3:
                f.setNextAnim(op.exitAnim);
                this.mManager.removeFragment(f);
                break;
            case 4:
                f.setNextAnim(op.exitAnim);
                this.mManager.hideFragment(f);
                break;
            case 5:
                f.setNextAnim(op.enterAnim);
                this.mManager.showFragment(f);
                break;
            case 6:
                f.setNextAnim(op.exitAnim);
                this.mManager.detachFragment(f);
                break;
            case 7:
                f.setNextAnim(op.enterAnim);
                this.mManager.attachFragment(f);
                break;
            case 8:
                this.mManager.setPrimaryNavigationFragment(f);
                break;
            case 9:
                this.mManager.setPrimaryNavigationFragment((Fragment)null);
            }

            if (!this.mReorderingAllowed && op.cmd != 1 && f != null) {
                this.mManager.moveFragmentToExpectedState(f);
            }
        }

       //我们也很容易就能看出来 最终都是走的 mManager.moveToState 这个方法
       //同时moveToState 也是 fragment 状态分发最重要的方法了
        if (!this.mReorderingAllowed) {
            this.mManager.moveToState(this.mManager.mCurState, true);
        }

    }

```

到这里应该就差不多了，最终的线索就是 只要搞明白moveToState这个函数就可以了。

```
     // 下面代码来自于fragment
    // 我们先去看看这个函数的参数之一new state是什么
   // 其实new state 就是代表新的状态
    static final int INITIALIZING = 0;
    static final int CREATED = 1;
    static final int ACTIVITY_CREATED = 2;
    static final int STARTED = 3;
    static final int RESUMED = 4;

```
   
下面我们可以模拟一个流程 帮助大家理解这个状态到底是干嘛的,有什么用。

比如我们先看看 fragmentactivity的源码，假设我们想看看 activity 发生 onResumne 事件的时候对 fragment 有什么影响。

```
    protected void onResume() {
        super.onResume();
        this.mHandler.sendEmptyMessage(2);
        this.mResumed = true;
        this.mFragments.execPendingActions();
    }

    final Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch(msg.what) {
            case 2:
                FragmentActivity.this.onResumeFragments();
                FragmentActivity.this.mFragments.execPendingActions();
                break;
            default:
                super.handleMessage(msg);
            }

        }
    };

    protected void onResumeFragments() {
        this.mFragments.dispatchResume();
    }

    public void dispatchResume() {
        this.mStateSaved = false;
        this.mStopped = false;
        this.dispatchStateChange(4);
    }

    //下面的代码来自 FragmentManagerImpl
    //一直追踪到这里就能明白 activity 的声明周期与 fragment 声明周期关联的时候就是通过 moveToState 这个函数来完成。
    private void dispatchStateChange(int nextState) {
        try {
            this.mExecutingActions = true;
            this.moveToState(nextState, false);
        } finally {
            this.mExecutingActions = false;
        }

        this.execPendingActions();
    }
```

__一直追踪到这里就能明白 activity 的声明周期与 fragment 声明周期关联的时候就是通过 moveToState 这个函数来完成__



# 再理一下相关类的关系吧

## FragmentManager 是什么 ？

FragmentActivity 中持有类 FragmentController 的一个对象 mFragments，而在 FragmentController 的 createController 方法中会传入 HostCallbacks（ 其为 FragmentActivity 的一个内部类）。
而该 HostCallbacks（其为接口 FragmentHostCallback 的一个实现类） 在构造时会传入一个当前 activity（FragmentActivity）的引用。

===>

 1 FragmentHostCallback 持有对 fragmentActivity，以及其对应的context和handler的引用

 2 FragmentController 通过 mHost 持有对 FragmentHostCallback的引用。


而当我们调用 getSupportFragmentManager()，其实是 FragmentController 转给 FragmentHostCallback ，并调用其 getFragmentManagerImpl 方法来获取一个 FragmentManagerImpl 对象。

===>
 
 1 __所以说 FragmentManager 其实就是 FragmentManagerImpl 的对象，其为 FragmentManager 的具体实现类__

##  BackStackRecord 是什么

当我们调用 FragmentManagerImpl 的 beginTransaction方法时，返回一个 BackStackRecord 的对象（该对象继承 FragmentTransaction，并持有对 FragmentManagerImpl 对象的引用）。
此时我们可以通过对象 backStackRecord 来将 fragmentA 添加到指定的 container 中。添加 fragmentA 本质就是__将 fragment 和对应的操作（如：add remove replace）封装成一个Op对象，然后将该对象存储在一个数组列表中__。

这里需要注意的是，containerId 为什么不存储 ？

其实 containerId 已经存储到 fragmentA 中了！！！

```
      // BackStackRecord执行add操作时会执行如下代码：
      fragment.mContainerId = fragment.mFragmentId = containerViewId;
```

backStackRecord 对象调用commit时又发生了什么呢？

上面说过，backStackRecord 对象持有对 FragmentManagerImpl 对象的引用，而commit操作，其实就回调了 FragmentManagerImpl 对象的 enqueueAction 方法来对 backStackRecord 对象进行操作。
到这里 backStackRecord 可以说暂时完成使命了。

其实 BackStackRecord 可以理解为 BackStack + Record， Record为记录的意思，记录啥呢？ 其实就是我们要对fragment执行的一些操作。

那么BackStack如何理解呢 ?

## FragmentManagerImpl 到底做了些啥？

我们将 BackStackRecord 对象中记录的一些对fragment的操作当做一个action。 那么当 backStackRecord 调用commit时，FragmentManagerImpl 对象如何处理这些 action 的呢？

1 用一个数组列表（PendingActions）去存储这个 action，然后 post 一个runnable（mExecCommit），到主线程去执行。

核心代码如下：

```
  public boolean execPendingActions() {
        this.ensureExecReady(true);

        boolean didSomething;
        for(didSomething = false; this.generateOpsForPendingActions(this.mTmpRecords, this.mTmpIsPop); didSomething = true) {
            this.mExecutingActions = true;

            try {
                this.removeRedundantOperationsAndExecute(this.mTmpRecords, this.mTmpIsPop);
            } finally {
                this.cleanupExec();
            }
        }

        this.doPendingDeferredStart();
        this.burpActive();
        return didSomething;
    }
```
该方法的执行，会调用到 BackStackRecord对象的 executeOps，这个方法就是根据不同的操作(所谓操作就是OP.CMD的那个值)来分发不同的事件，
从而调用 FragmentManagerImpl 来转换 fragment 生命周期的方法！




一直分析到这里，相信大家就对fragment的源码基础知识有一个不错的理解了，在这里就简单总结一下上面的分析：

1. FragmentActivity 是具有支持 fragment 功能的最底层的 activity, 其他什么 AppCompatActivity 都是他的子类！

2. __FragmentActivity 主要负责就是生命周期的转发__，比如 onCreate onResume onDestroy 等等，这就是为什么 activity 和 fragment 状态能统一的原因了！

当然了，分发的原因就是因为 fragmentActivity 对象持有一个 fragmentController 的实例！

其实讲白了，fragmentController 就是因为它自己有一个 fragmentHostCallback，然后这个 fragmentHostCallback 还持有了 fragmentManagerImpl,

最终由 fragmentManagerImpl 的 moveToState 方法完成 fragment 生命周期的转换。


3. fragementHostCallback 持有了 activity 的很多资源，context，handler 是其中最主要的2个。fragmentManagerImpl 就是因为拿到了 activty 的这2个资源，所以才能和 activty 互相通信的！

   为什么 fragmentManagerImpl 可以拿到 activty 的 context 和 handler？

   因为其为 fragementHostCallback 的内部类

4. fragmentMangerImple 就是 fragmentManger 的具体实现类。moveToState方法就是在这个里面实现的 

5. FragmentTransition 也是个抽象类，他主要就是提供对外的接口函数的 add replace move 这种。BackStackRecord 就是它的具体实现类，还额外实现了 OpGenerator 接口。

6. BackStackRecord 里面会有个 executeOps() 方法。这个方法就是根据不同的操作(所谓操作就是OP.CMD的那个值)来分发不同的事件，从而调用 fragmentManager的各种转换 fragment 生命周期的方法！














