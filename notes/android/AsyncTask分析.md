
# AsyncTask

# AsyncTask的基本用法

首先来看一下AsyncTask的基本用法，由于AsyncTask是一个抽象类，所以如果我们想使用它，就必须要创建一个子类去继承它。在继承时我们可以为AsyncTask类指定三个泛型参数，这三个参数的用途如下：

1. Params

在执行AsyncTask时需要传入的参数，可用于在后台任务中使用。

2. Progress

后台任务执行时，如果需要在界面上显示当前的进度，则使用这里指定的泛型作为进度单位。

3. Result

当任务执行完毕后，如果需要对结果进行返回，则使用这里指定的泛型作为返回值类型。

一般需要去重写AsyncTask中的几个方法才能完成对任务的定制。经常需要去重写的方法有以下四个：


1. onPreExecute()

这个方法会在后台任务开始执行之间调用，用于进行一些界面上的初始化操作，比如显示一个进度条对话框等。

2. doInBackground(Params...)

这个方法中的所有代码都会在子线程中运行，我们应该在这里去处理所有的耗时任务。任务一旦完成就可以通过return语句来将任务的执行结果进行返回，如果AsyncTask的第三个泛型参数指定的是Void，就可以不返回任务执行结果。注意，在这个方法中是不可以进行UI操作的，如果需要更新UI元素，比如说反馈当前任务的执行进度，可以调用publishProgress(Progress...)方法来完成。

3. onProgressUpdate(Progress...)

当在后台任务中调用了publishProgress(Progress...)方法后，这个方法就很快会被调用，方法中携带的参数就是在后台任务中传递过来的。在这个方法中可以对UI进行操作，利用参数中的数值就可以对界面元素进行相应的更新。

4. onPostExecute(Result)

当后台任务执行完毕并通过return语句进行返回时，这个方法就很快会被调用。返回的数据会作为参数传递到此方法中，可以利用返回的数据来进行一些UI操作，比如说提醒任务执行的结果，以及关闭掉进度条对话框等。

因此，一个比较完整的自定义AsyncTask就可以写成如下方式：

```
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {
 
	@Override
	protected void onPreExecute() {
		progressDialog.show();
	}
 
	@Override
	protected Boolean doInBackground(Void... params) {
		try {
			while (true) {
				int downloadPercent = doDownload();
				publishProgress(downloadPercent);
				if (downloadPercent >= 100) {
					break;
				}
			}
		} catch (Exception e) {
			return false;
		}
		return true;
	}
 
	@Override
	protected void onProgressUpdate(Integer... values) {
		progressDialog.setMessage("当前下载进度：" + values[0] + "%");
	}
 
	@Override
	protected void onPostExecute(Boolean result) {
		progressDialog.dismiss();
		if (result) {
			Toast.makeText(context, "下载成功", Toast.LENGTH_SHORT).show();
		} else {
			Toast.makeText(context, "下载失败", Toast.LENGTH_SHORT).show();
		}
	}
}
```

当我们 new 一个AsyncTask时做了什么？

```
  /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

```

只是初始化了两个变量，mWorker和mFuture，并在初始化mFuture的时候将mWorker作为参数传入。mWorker是一个Callable对象，mFuture是一个FutureTask对象。

接着如果想要启动某一个任务，就需要调用该任务的execute()方法，因此现在我们来看一看execute()方法的源码，如下所示：

```
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

只有一行代码，仅是调用了executeOnExecutor()方法，那么具体的逻辑就应该写在这个方法里了。

```
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING; // 更新task的状态

        onPreExecute();//

        mWorker.mParams = params;// 给Callable中的参数赋值
        exec.execute(mFuture);

        return this;
    }

```

来看看execute方法里做了啥？

```
    @MainThread
    public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }

   public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

   private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

```

查看上面的execute()方法，原来是传入了一个sDefaultExecutor变量，而这个sDefaultExecutor其实也就是SerialExecutor。

那么我们自然要去看看SerialExecutor的源码了，如下所示：

```
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();//SerialExecutor是使用ArrayDeque这个队列来管理Runnable对象的
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {//首先在第一次运行execute()方法的时候，会调用ArrayDeque的offer()方法将传入的Runnable对象添加到队列的尾部
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {//第一次运行当然是等于null了，于是会调用scheduleNext()方法
                scheduleNext();
            }
        }
       //在这个方法中会从队列的头部取值，并赋值给mActive对象，然后调用THREAD_POOL_EXECUTOR去执行取出的取出的Runnable对象。
       //之后如何又有新的任务被执行，同样还会调用offer()方法将传入的Runnable添加到队列的尾部，但是再去给mActive对象做非空检查的时候就会发现mActive对象已经不再是null了，于是就不会再调用scheduleNext()方法。
       // 也就是说，每次当一个任务执行完毕后，下一个任务才会得到执行，SerialExecutor模仿的是单一线程池的效果，如果我们快速地启动了很多任务，同一时刻只会有一个线程正在执行，其余的均处于等待状态。
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

```

SerialExecutor类中也有一个execute()方法，该方法开启了一个子线程去执行了mFuture对象的run方法，而在该run方法中，其实调用的就是之前在mFuture的构造方法中传进来的mWorker对象的call方法。


再来看看mWorker的call方法做了些啥？

```
 public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);//doInBackground()方法的调用处
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);//将doInBackground()方法返回的结果传递给了postResult()方法
                }
                return result;
            }

```

postResult方法做了些什么了？

```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result)); // 创建一个message对象，消息中携带了MESSAGE_POST_RESULT常量和一个表示任务执行结果的AsyncTaskResult对象
        message.sendToTarget();
        return result;
    }

   private Handler getHandler() {
        return mHandler;
    }

   mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }

```

当我们使用的是AsyncTask的无参构造方法时，mHandler 也就是 InternalHandler 对象。


```
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);//这种消息用于更新当前进度的
                    break;
            }
        }
    }

```

这里对消息的类型进行了判断，如果这是一条MESSAGE_POST_RESULT消息，就会去执行finish()方法，如果这是一条MESSAGE_POST_PROGRESS消息，就会去执行onProgressUpdate()方法。
那么finish()方法的源码如下所示：

```
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;// 更新task的状态
    }

```
可以看到，如果当前任务被取消掉了，就会调用onCancelled()方法，如果没有被取消，则调用onPostExecute()方法，这样当前任务的执行就全部结束了。


[Android AsyncTask完全解析，带你从源码的角度彻底理解](https://blog.csdn.net/guolin_blog/article/details/11711405)
