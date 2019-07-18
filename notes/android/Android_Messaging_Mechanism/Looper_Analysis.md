# 消息循环 Looper

Looper 在消息机制中扮演着消息循环的角色, 它会不停的从 MessageQueue 中查看是否有可以处理的消息. 如果有就会立即处理，否则会一直阻塞在那里。

## Looper 构造方法

在 Looper 构造方法里它会创建一个 MessageQueue, 并保存当前的线程对象:

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

即在哪个线程创建 Looper，该 Looper 对象就会与哪个线程相互关联起来。

通过 Looper.prepare() 就可以为一个线程创建 Looper 对象，然后调用 Looper.loop 来开启循环。

```
        new Thread("Thread#2") {
            @Override
            public void run() {
                Looper.prepare();
                Handler mHandler = new Handler();
                Looper.loop();
            }
        }.start();
```

Looper 除了 prepare 方法外，还提供了 prepareMainLooper 方法，这个方法是给主线程也就是 ActivityThread 创建 Looper 使用的，由于主线程的Looper比较特殊，所以 Looper 提供了一个 getMainLooper 方法，通过它可以在任何地方获取到主线程的 Looper。Looper 也是可以退出的，提供了 quit 和 quitSafely 来退出一个Looper。

二者的区别是:

* quit 会直接退出，而 quitSafely 只是设定一个退出标记，然后把消息队列中已有消息处理完毕后才安全退出。

* Looper 退出后，通过 Handler 发送的消息会失败，这个时候 Handler 的 send 方法会返回 false。

__在子线程中，如果手动为其创建 Looper，那么所有的事情完成以后应该调用 quit 方法来终止循环，否则子线程会一直处于等待状态__。

如果 Looper 退出以后，这个线程就会立刻终止，因此建议不需要的时候终止 Looper。

## loop()

Looper 最重要的一个方法是 loop，只有调用 loop()，消息循环才会真正的起作用。

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
        // 开起一个死循环
        for (;;) {
            Message msg = queue.next(); // might block
            //  next 方法返回 null,退出循环
            //  quit 方法调用时，Looper 就会调用 MessageQueue 的 quit 和 quitSafely 方法来通知队列退出
            //  当消息队列被标记为退出状态的时候，它的 next 方法就会返回 null
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);// 处理从消息队列中取出的 msg

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

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

只有当 MessageQueue 的 next 方法返回 null 时才会跳出该死循环。

### 那么何时 next 才会返回 null 呢？

当 Looper 被 quit 方法调用时，Looper 就会调用 MessageQueue 的 quit 或者 quitSafely 方法来通知队列退出。当消息队列被标记为退出状态的时候，它的 next 方法就会返回 null。

如果 MessageQueue 的 next 返回最新消息，Looper 就会处理这条消息：msg.target.dispatchMessage(msg)，这里的 msg.target 是发送这条消息的 Handler 对象，这样 Handler 
所发送的消息最终又交给它的 dispatchMessage 方法来处理了。

值得注意的是：__Handler 的 dispatchMessage 方法是在创建 Handler 时所使用的 Looper 中执行的，这样就成功的将代码逻辑切换到指定的线程中去执行__。
