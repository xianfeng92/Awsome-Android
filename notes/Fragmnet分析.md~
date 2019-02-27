
## 基本概念

* Fragment是依赖于Activity的，不能独立存在的。

* 一个Activity里可以有多个Fragment。

* 一个Fragment可以被多个Activity重用。

* Fragment有自己的生命周期，并能接收输入事件。

* 我们能在Activity运行时动态地添加或删除Fragment。


### Fragment的优势有以下几点：

* 模块化（Modularity）：我们不必把所有代码全部写在Activity中，而是把代码写在各自的Fragment中。

* 可重用（Reusability）：多个Activity可以重用一个Fragment。

* 可适配（Adaptability）：根据硬件的屏幕尺寸、屏幕方向，能够方便地实现不同的布局，这样用户体验更好。


## 生命周期

Fragment的生命周期和Activity类似，但比Activity的生命周期复杂一些，基本的生命周期方法如下图：


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

因为Fragment是依赖Activity的，因此为了讲解Fragment的生命周期，需要和Activity的生命周期方法一起讲，
即Fragment的各个生命周期方法和Activity的各个生命周期方法的关系和顺序，如图：




FragmentTransaction有一些基本方法，下面给出调用这些方法时，Fragment生命周期的变化：

* add(): onAttach()->…->onResume()

* remove(): onPause()->…->onDetach()

* replace(): 相当于旧Fragment调用remove()，新Fragment调用add()

* show(): 不调用任何生命周期方法，调用该方法的前提是要显示的 Fragment已经被添加到容器，只是纯粹把Fragment UI的setVisibility为true

* hide(): 不调用任何生命周期方法，调用该方法的前提是要显示的Fragment已经被添加到容器，只是纯粹把Fragment UI的setVisibility为false

* detach(): onPause()->onStop()->onDestroyView()。UI从布局中移除，但是仍然被FragmentManager管理

* attach(): onCreateView()->onStart()->onResume()


## Back Stack 的实现原理

我们知道Activity有任务栈，用户通过startActivity将Activity加入栈，点击返回按钮将Activity出栈。__Fragment也有类似的栈，称为回退栈（Back Stack）__，
回退栈是由FragmentManager管理的。默认情况下，Fragment事务是不会加入回退栈的，如果想将Fragment事务加入回退栈，则可以加入addToBackStack("")。
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

从定义可以看出，BackStackRecord有三重含义：

* 继承了FragmentTransaction，即是事务，保存了整个事务的全部操作轨迹。

* 实现了BackStackEntry，作为回退栈的元素，正是因为该类拥有事务全部的操作轨迹，因此在popBackStack()时能回退整个事务。

* 继承了Runnable，即被放入FragmentManager执行队列，等待被执行。

先看第一层含义，getSupportFragmentManager.beginTransaction()返回的就是BackStackRecord对象.
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
addOp()是将创建好的Op对象加入数组列表，定义如下

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
                this.mIndex = this.mManager.allocBackStackIndex(this);
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
在commitInternal()中，mManager.enqueueAction(this, allowStateLoss)，是将 BackStackRecord 加入待执行队列中，定义如下：

```
    public void enqueueAction(FragmentManagerImpl.OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            this.checkStateLoss();
        }

        synchronized(this) {
            if (!this.mDestroyed && this.mHost != null) {
                if (this.mPendingActions == null) {
                    this.mPendingActions = new ArrayList();//mPendingActions就是前面说的待执行队列
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

```
mHost.getHandler()就是主线程的Handler，因此Runnable是在主线程执行的，mExecCommit的内部就是调用了execPendingActions()，即把mPendingActions中所有积压的没被执行的事务全部执行。
执行队列中的事务会怎样被执行呢？就是调用BackStackRecord的run()方法，run()方法就是执行Fragment的生命周期函数，还有将视图添加进container中。


与addToBackStack()对应的是popBackStack()，有以下几种变种：

* popBackStack()：将回退栈的栈顶弹出，并回退该事务。

* popBackStack(String name, int flag)：name为addToBackStack(String name)的参数，通过name能找到回退栈的特定元素，flag可以为0或者FragmentManager.POP_BACK_STACK_INCLUSIVE，0表示只弹出该元素以上的所有元素，POP_BACK_STACK_INCLUSIVE表示弹出包含该元素及以上的所有元素。这里说的弹出所有元素包含回退这些事务。

* popBackStack()是异步执行的，是丢到主线程的MessageQueue执行，popBackStackImmediate()是同步版本。


## Fragment通信

### Fragment向Activity传递数据

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


## DialogFragment

DialogFragment是Android 3.0提出的，代替了Dialog，用于实现对话框。他的优点是：即使旋转屏幕，也能保留对话框状态。



[Android基础：Fragment，看这篇就够了](https://mp.weixin.qq.com/s/dUuGSVhWinAnN9uMiBaXgw)
