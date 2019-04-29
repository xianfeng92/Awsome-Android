# 先上结论

1. Handler的 sendMessage 方法做了什么？

   本质上，就是将一个 message 存储到 Looper 的一个 MessageQueue 中。

2. Handler 创建时需要先提供一个 Looper 对象（loop.prepare()），而 Looper 对象是依附于线程的（存储在 ThreadLocal 中），在 Looper 对象的构造中，会创建一个 MessageQueue 对象。

3. 为什么在 UI 线程中使用 Handler 处理消息，不需要调用 Looper.prepare() 和 Looper.loop()

   ActivityThread 创建中，已经调用了 Looper.prepareMainLooper() 和 Looper.loop()，来创建 Looper 和 MessageQueue。

4. MessageQueue作用

   MessageQueue以 msg 的 time 为基准，维护一个优先级队列。

5. Handler 何时处理 message？

  Looper.loop() 方法会不断的从 MessageQueue 中取出 msg 进行处理（当MessageQueue为空时，进入阻塞状态）。取出msg时，会通过 msg.target 找到是哪个 Handler 发送这个mgs的，
  然后调用该 Handler 的 dispatchMessage 方法来处理 msg。

6. Handler post 一个runnable是如何处理的呢？

  Handler 会将 runnable 封装成一个msg对象，并将 runnable 赋值给 msg 的一个callback变量。在 dispatchMessage，此时检查到 callback 不为null，会调用 callback.run()方法，即 
  runnable的run方法被调用。
  
# Android的消息机制概述

   Android 的消息机制主要是指 Handler 的运行机制，Handler 的运行需要底层的 MessageQueue 和 Looper 的支撑。MessageQueue 即消息队列，它内部存储了一组消息，以队列的形式对外提供插入和删除的工作。
虽然叫做消息队列，但是它的内部存储结构是__采用单链表的数据结构来存储消息列表(基于单链表实现的队列)__。Looper 为消息循环。由于 MessageQueue 只是一个消息的存储单元，并不能
去处理消息，而 Looper 就填补了这个功能，Looper 会以无限循环的形式去查找是否有新的消息，如果有的话就处理，否则就一直等待。Looper 中还有一个特殊的概念，那就是 ThreadLocal, ThreadLocal 并不是线程，
它的作用是可以在每个线程中存储数据。我们知道 Handler 创建的时候会采用当前的 Looper 来构建消息循环系统，那么 Handler 内部是如何获取当前线程的 Looper 呢？ 就是使用 ThreadLocal ，
__ThreadLocal 可以在不同线程中互不干扰地存储并提供数据__，通过 ThreadLocal 可以轻松获取每个线程的 Looper 。


__系统提供 Handler，主要的原因就是解决在子线程中无法访问 UI 的矛盾。__

* 系统为什么不允许在子线程访问 UI 呢？
  这是因为Android的UI控件不是线程安全的，如果在多线程中并发访问可能会导致UI控件处于不可预期的状态

* 那为什么系统不对UI控件访问加上锁机制呢？
  首先加上锁后会让 UI 访问的逻辑变得复杂，其次是会降低 UI 的访问频率。所以__最简单高效的就是采用单线程模型来处理 UI 操作__


Handler 创建完毕后,这个时候其内部的 Lopper 以及 MeaasgeQueue 就可以和 Handler 一起协同工作，然后通过 Handler 的 post 方法将一个 Runnable 投递到 Handler 内部的 Lopper 中去处理，
也可以通过 Handler 的 send 方法发送一个消息，这个消息同样会在 Lopper 中去处理，其实 post 方法最终还是通过 send 方法来完成的。

接下来我们来看下 send 方法的工作过程，当 Handler 的 send 被调用的时候，它会调用 MessageQueu 的 enqueueMessage 方法将这个消息放入消息队列，然后Lopper发现新消息到来时，就会处理这个消息，
最终消息的 Runnable 或者 Handler 的 handlerMessage 方法就被调用，注意 __Lopper 是运行在创建 Handler 所在的线程中__，这样 Handler 中的业务就会被切换到所在线程就执行了，如图：


  
  ![](https://github.com/xianfeng92/android-code-read/blob/master/images/20130817090611984.png)


# Android的消息机制分析

## ThreadLocal 的工作原理

__ThreadLocal 是一个线程内部的数据存储类，通过它可以在执行的线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据__。
一般来说，某一个数据是以线程为作用域并且不同线程具有不同的 Looper，这个时候通过 ThreadLocal 就可以轻松的实现 Looper 在线程中的存取。


下面通过实际的例子来演示 ThreadLocal 的真正含义，首先定义一个 ThreadLocal 对象，这里选择 Boolean 类型的，如下：

```
  private ThreadLocal<Boolean> mBooleanThread = new ThreadLocal<Boolean>();

```

然后分别在主线程，子线程1和2中访问:

```
        mBooleanThread.set(true);
        Log.i(TAG, "主线程:" + mBooleanThread.get());

        new Thread("Thread #1") {
            @Override
            public void run() {
                mBooleanThread.set(false);
                Log.i(TAG, "Thread #1:" + mBooleanThread.get());
            }
        }.start();

        new Thread("Thread #2") {
            @Override
            public void run() {
                Log.i(TAG, "Thread #2:" + mBooleanThread.get());
            }
        }.start();
```


这段代码中，主线程设置了 ThreadLocal 为true。而在子线程1中设置了 false 然后分别获取他们的值。这个时候主线程为true,子线程1中为false，而子线程2中由于没有设置，是null。

虽然在不同线程中访问的是同一个 ThreadLocal 对象，但是他们通过 ThreadLocal 获取到的值确实不一样的，这就是ThreadLocal的奇妙之处。


ThreadLocal 之所以有这么奇妙的效果，是因为不同线程访问同一个 ThreadLocal 的 get，ThreadLocal 内部会从各自的线程中取出一个数组，然后从数组中根据当前的 ThreadLocal 索引查出对应的 value 值。
很显然，不同线程中的数组是不同的，这就是为什么通过 ThreadLocal 可以在不同的线程中维护一套数据的副本并且彼此互不干扰。

ThreadLocal 是一个泛型类，只要弄清楚 ThreadLocal 的 get 和 set 方法就能明白它的工作原理：

首先看ThreadLocal的set方法：

```
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

在上面的 set 方法中会通过 values 方法来获取当前线程中的 ThreadLocal 数组。其实获取的方法也很简单，在 Thread 类的内部有一个成员专门用于存储线程的 ThreadLocal 数据：
ThreadLocal.Values localValues，因此获取当前线程的 ThreadLocal 数据就变成异常简单了。下面看一下 ThreadLocal 的值到底如何在 localValues 中进行存储的。

在 loaclValues 内部有一个数组：private Object[] table，ThreadLocal的值就存在这个 table 数组中。具体看一下 localValues 是如何使用 put 方法将 
ThreadLocal 的值存储到 table 数组中的，如下所示：

```
 void put(ThreadLocal<?> key, Object value) {
            cleanUp();
            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }
```

通过上面的代码，我们可以看出一个存储规则，那就是 ThreadLocal 的值在 table 数组中的存储位置总是为 ThreadLocal 的 reference 字段所标识的对象的下一个位置，
比如 ThreadLocal 的 reference 对象在 table数组中的索引为index，那么 ThreadLocal 的值在 table 数组中的索引就是 index + 1 ,最终 ThreadLocal 的值将会被存
储在table数组中，table[index + 1] = vales。

我们再来看下get方法：

```
public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);
    }
```

可以发现 ThreadLocal 的 get 方法逻辑还算是比较清晰，他同样是取当前线程的 localValues 对象，如果这个对象为 null 那么就返回初始值，初始值由 ThreadLocal 的 initialValue 方法来描述。
默认情况下为null，当然也可以重写这个方法，它的默认实现是：

```
    protected T initialValue() {
        return null;
    }

```

如果localValues对象不为null，那就取出它的table数组并找出 ThreadLocal 的 rederence 对象在 table 数组中的位置，然后 table 数组的下一个位置所存储的数据就是 ThreadLocal 的值。

从 ThreadLocal 的 set 和 get 方法可以看出，它们所操作的对象都是当前线程的 localValues 对象的 table 数组，因此在不同线程中访问同一个 ThreadLocal 的set和get方法，
它们对 ThreadLocal 所做的读写操作仅限于各自线程的内部，这就是为什么 ThreadLocal 可以在多个线程互不干扰的存储和修改数据。


## 消息队列的工作原理

消息队列在 Android 中指的是 MessageQueue，MessageQueue 主要包含两个操作---插入和读取，插入和读取对应的方法分别为 enqueueMessage 和 next，
其中 enqueueMessage 的作用是往消息队列中插入一条消息，而 next 的作用是从消息队列中取出一条消息并将其从消息队列中移除。MessageQueue 的内部
是通过一个单链表的数据结构来维护消息队列，单链表的插入和删除上比较有优势，下面主要看一下它的 enqueueMessage 和 next 方法的实现：

```
 boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        synchronized (this) {
            if (mQuitting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            // MessageQueue 为空时，直接将 msg 作为消息队列的头部
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 将 msg 插入到 MessageQueue 中，一般不必唤醒消息队列，除非该消息是队列中最早的异步消息。
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

从 enqueueMessage 的实现中可以看出，主要是以 msg 的 delayed 为基准，将 msg 插入到消息队列中。


下面看一下 next 方法的实现，next 的主要逻辑：

```
  Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        // 无限循环
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            // We can assume mPtr != 0 because the loop is obviously still running.
            // The looper will not call this method after the loop quits.
            nativePollOnce(mPtr, nextPollTimeoutMillis);
            // 进行同步操作
            synchronized (this) {
                // 尝试检索下一条消息。 如果找到则返回。
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 下一条消息尚未就绪，设置超时以在其准备就绪时将其唤醒
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 获取到一个 message
                        mBlocked = false;
                        // 将 msg 从 MessageQueue 中移除
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // 没有更多的消息
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
可以发现 next 方法是一个无限循环的方法，主要操作如下：

1. 如果消息队列中没有消息，那么 next 方法会一直阻塞在这里

2. 当有新消息到来时，next 方法会检查该 msg 是否达到 delayed 时间，如果有，则从 MessageQueue 中移除该 mgs，并执行。如果没有，next 会继续等待该 msg 达到 delayed 时间。


## Lopper的工作原理

Looper 在消息机制中扮演着__消息循环__的角色，具体来说就是它会不停的从 MessageQueue 中查看是否有可以处理的消息，如果有就会立即处理，否则就会一直阻塞在那里。我们先来看下他的构造方法，
在构造方法里它会创建一个 MessageQueue，然后将当前线程的对象保存起来：

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

```

我们都知道，Handler 的工作需要 Looper，没有Looper的线程就会报错，那么如何为一个线程创建 Looper，其实很简单，就是通过Looper.prepare()就可以为他创建了，
然后通过 Looper.loop 来开启循环。

```
        new Thread("Thread #2") {
            @Override
            public void run() {
                Looper.prepare();
                Handler mHandler = new Handler();
                Looper.loop();
            }
        }.start();
```

Looper 除了 prepare 方法外，还提供了 prepareMainLooper 方法，这个方法主要是给主线程也就是 ActivityThread 创建 Looper 使用的，由于主线程的Looper比较特殊，
所以 Looper 提供了一个 getMainLooper 方法，通过它可以在任何地方获取到主线程的 Looper。Looper 也是可以退出的，提供了 quit 和 quitSafely 来退出一个Looper。
二者的区别是：quit 会直接退出，而 quitSafely 只是设定一个退出标记，然后把消息队列中已有消息处理完毕后才安全退出。Looper 退出后，通过 Handler 发送的消息会失败，
这个时候 Handler 的 send 方法会返回 false。__在子线程中，如果手动为其创建 Looper，那么所有的事情完成以后应该调用 quit 方法来终止循环，否则子线程会一直处于等待状态__，
而如果 Looper 退出以后，这个线程就会立刻终止，因此建议不需要的时候终止 Looper。

Looper 最重要的一个方法是 loop，只有调用 loop，这个消息循环才会真正的起作用:

```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        // 该方法也是个死循环
        for (;;) {
            Message msg = queue.next(); // might block
            //  next方法返回null,退出循环
            //  quit方法调用时，Looper 就会调用 MessageQueue 的 quit 和 quitSafely 方法来通知队列退出
            // 当消息队列被标记为退出状态的时候，它的 next 方法就会返回 null
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycle();
        }
    }
```

Looper的loop方法在工作过程也比较好理解，loop 方法是死循环，唯一跳出循环的方法是 MessageQueue 的 next 方法返回null。

那么何时 next 才会返回 null 呢？

当 Looper 被 quit 方法调用时，Looper 就会调用 MessageQueue 的 quit 或者 quitSafely 方法来通知队列退出。当消息队列被标记为退出状态的时候，它的next方法就会返回null，
也就是说，Looper必须退出，否则 loop 方法就会无限循环下去，loop 方法会调用 MessageQueue 的 next 来获取最新消息，而 next 是一个阻塞操作，当没有消息时，next就会阻塞，
这也就导致 loop 方法一直阻塞在那里。如果 MessageQueue 的 next 返回最新消息，Looper就会处理这条消息：msg.target.dispatchMessage(msg)，
这里的 msg.target 是发送这条消息的 Handler 对象，这样 Handler 所发送的消息最终又交给它的 dispatchMessage 方法来处理了。
但是这里不同的是：Handler 的 dispatchMessage 方法是在创建 Handler 时所使用的 Looper 中执行的，这样就成功的将代码逻辑切换到指定的线程中去执行。


## Handler 的工作原理

Handler 的工作主要是包含消息的发送和接收过程，消息的发送是通过 post 的一系列方法以及 send 的一些列方法来实现的， post 的一系列方法最终是通过 send 的一些列方法来实现的。

发送一条消息的典型过程如下：

```
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }


    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

可以发现， Handler 发消息仅仅是向消息队列里插入一条消息，MessageQueue 的 next 方法就会返回这条消息给  Looper，最终 Handler 的 dispatchMessage 方法就会被调用。

这个时候 Handler 就进入了处理消息的阶段，dispatchMessage的实现如下：

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

首先，它会检查 Message 的 callback 是否为 null，不为null就通过 handlerCallback 来处理消息，Message的 callback 是一个 Runnable 对象。
实际上就是 Handler 的 post 方法所传递 Runnable 参数，handlerCallback 的逻辑也很简单。

```
  private static void handleCallback(Message message) {
        message.callback.run();
    }
```

其次是检查 mCallback 是否为null，不为 null 就调用 mCallback 的 handlerMessage 方法来处理消息，Callback是个接口：

```
    public interface Callback {
        public boolean handleMessage(Message msg);
    }

```

通过 Callback 可以采用如下的方式来创建 Handler 对象，Handler handler = new Handler(callback)。

那么callback的含义在什么呢？

源码里面的注释已经说明，可以用来创建一个 Handler 的实例单并不需要派生 Handler 的子类。在日常开发中，创建 handler 最常见的方式就是派生一个 handler 子类并重写 
handlerMessage 来处理具体的消息，而 Callback 给我们提供了另外一种使用 Handler 的方式，当我们不想派生子类的时候，就可以通过 Callback 来实现。

最后通过调用 Handler 的 handlerMessage 方法来处理消息，Handler 处理消息的过程：

![handler_dispatchMessage]()


Handler 有一个特殊的构造方法，那就是通过一个特定的 Looper 来构造 Handler，它的实现如下：

```
    public Handler(Looper looper) {
        this(looper, null, false);
    }

```

下面再来看下 Handler 的默认构造方法 public Handler，这个构造方法是调用下面的构造方法，很明显，如果当前线程没有Looper的话，
就会抛出Cant create handler inside thread that has not called Looper.prepare 这个异常，这个解释在没有 Looper 的子线程创建 handler 会引发程序异常的原因。

```
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```


另外除了发送消息之外，我们还有以下几种方法可以在子线程中进行UI操作：

1. Handler 的post()方法

2. View 的post()方法

3. Activity 的 runOnUiThread()方法


## 主线程的消息循环

Android 的主线程就是 ActivityThread，主线程的入口为 main，在 mai n方法中系统通过 Looper.prepareMainLooper 来创建主线程的 Looper 和 MessageQueue，
并且通过 Loop.loop() 来开启主线程的消息循环，这个过程如下所示：


```
 public static void main(String[] args) {
        SamplingProfilerIntegration.start();
        ...
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();
        ...
    }
}
```

主线程消息循环开始了以后，ActivityThread 还需要一个 Handler 来和消息队列进行交互，这个 Handler 就是 ActivityThread.H，
它内部定义了一组消息类型，主要包括四大组件的启动和停止过程，如下所示：

```
 private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        public static final int PAUSE_ACTIVITY          = 101;
        public static final int PAUSE_ACTIVITY_FINISHING= 102;
        public static final int STOP_ACTIVITY_SHOW      = 103;
        public static final int STOP_ACTIVITY_HIDE      = 104;
        public static final int SHOW_WINDOW             = 105;
        public static final int HIDE_WINDOW             = 106;
        public static final int RESUME_ACTIVITY         = 107;
        public static final int SEND_RESULT             = 108;
        public static final int DESTROY_ACTIVITY        = 109;
        public static final int BIND_APPLICATION        = 110;
        public static final int EXIT_APPLICATION        = 111;
        public static final int NEW_INTENT              = 112;
        public static final int RECEIVER                = 113;
        public static final int CREATE_SERVICE          = 114;
        public static final int SERVICE_ARGS            = 115;
        public static final int STOP_SERVICE            = 116;
        public static final int REQUEST_THUMBNAIL       = 117;
        public static final int CONFIGURATION_CHANGED   = 118;
        public static final int CLEAN_UP_CONTEXT        = 119;
        public static final int GC_WHEN_IDLE            = 120;
        public static final int BIND_SERVICE            = 121;
        public static final int UNBIND_SERVICE          = 122;
        public static final int DUMP_SERVICE            = 123;
        public static final int LOW_MEMORY              = 124;
        public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
        public static final int RELAUNCH_ACTIVITY       = 126;
        public static final int PROFILER_CONTROL        = 127;
        public static final int CREATE_BACKUP_AGENT     = 128;
        public static final int DESTROY_BACKUP_AGENT    = 129;
        public static final int SUICIDE                 = 130;
        public static final int REMOVE_PROVIDER         = 131;
        public static final int ENABLE_JIT              = 132;
        public static final int DISPATCH_PACKAGE_BROADCAST = 133;
        public static final int SCHEDULE_CRASH          = 134;
        public static final int DUMP_HEAP               = 135;
        public static final int DUMP_ACTIVITY           = 136;
        public static final int SLEEPING                = 137;
        public static final int SET_CORE_SETTINGS       = 138;
        public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
```

ActivityThread 通过 ApplicationThread 和 AMS 进程进程间通信，AMS 以进程间通信的方法完成 ActivityThread 的请求后回调 ApplicationThread 
的 Binder 方法，然后通过 H 发送消息。H 收到消息后将 ApplicationThread 中的逻辑切换到 ActivityThread 去执行，这就是切换到主线程去执行，
这个过程就是主线程的消息循环模型。


#  Looper 死循环为什么不会导致应用卡死？

线程默认没有 Looper 的，如果需要使用 Handler 就必须为线程创建 Looper。我们经常提到的主线程，也叫UI线程，它就是 ActivityThread，ActivityThread
被创建时就会初始化 Looper，这也是在主线程中默认可以使用 Handler 的原因。

我们知道 Looper.loop() 里面维护了一个死循环方法，也就是说循环在Looper.prepare()与Looper.loop()之间。在子线程中，如果手动为其创建了Looper，
那么在所有的事情完成以后应该调用 quit 方法来终止消息循环，否则这个子线程就会一直处于等待（阻塞）状态，而如果退出Looper以后，
这个线程就会立刻（执行所有方法并）终止，因此建议不需要的时候终止Looper。


这时候如果了解了 ActivityThread，并且在 main 方法中我们会看到主线程也是通过 Looper 方式来维持一个消息循环。

```
public static void main(String[] args) {

        ``````
        Looper.prepareMainLooper();//创建Looper和MessageQueue对象，用于处理主线程的消息

        ActivityThread thread = new ActivityThread();
        thread.attach(false);//建立Binder通道 (创建新线程)

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        //如果能执行下面方法，说明应用崩溃或者是退出了...
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

那么回到我们的问题上，这个死循环会不会导致应用卡死，即使不会的话，它会慢慢的消耗越来越多的资源吗？ 

对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出。

那么如何保证能一直存活呢？

简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，
无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，
会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。


主线程的死循环一直运行是不是特别消耗CPU资源呢？ 

其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，
直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，
则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。


##  Handler 是如何能够线程切换

线程间是共享资源的。所以Handler处理不同线程问题就只要注意异步情况即可。__Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，Looper 在哪个线程创建，就跟哪个线程绑定__，
并且__Handler是在它关联的 Looper 对应的线程中处理消息的__。

那么Handler内部如何获取到当前线程的Looper呢—–ThreadLocal。ThreadLocal可以在不同的线程中互不干扰的存储并提供数据，通过ThreadLocal可以轻松获取每个线程的Looper。

当然需要注意的是:

1. 线程是默认没有Looper的，如果需要使用Handler，就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是 ActivityThread

2. ActivityThread 被创建时就会初始化Looper，这也是在主线程中默认可以使用Handler的原因


# 参考

[Android异步消息处理机制完全解析，带你从源码的角度彻底理解](https://blog.csdn.net/guolin_blog/article/details/9991569)
[你真应该再多了解些Handler机制](https://www.jianshu.com/p/8862bd2b6a29)
[Android开发艺术探索]()
