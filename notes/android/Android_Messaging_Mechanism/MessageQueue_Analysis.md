# MessageQueue 工作原理

消息队列在 Android 中指的是 MessageQueue，MessageQueue 主要包含两个操作:插入和读取，插入和读取对应的方法分别为 enqueueMessage 和 next，
其中 enqueueMessage 的作用是往消息队列中插入一条消息，而 next 的作用是从消息队列中取出一条消息并将其从消息队列中移除。MessageQueue 的底层
采用单链表来实现一个消息队列，单链表的插入和删除上比较有优势。

## enqueueMessage

```
 boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) { // 确保当前的 msg 不在使用中
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) { // 确保指定了处理该 msg 的 Handler
            throw new AndroidRuntimeException("Message must have a target.");
        }

        synchronized (this) { // 线程同步
            if (mQuitting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            /*
             * 当出现下面三种情况，直接将当前入队的 msg 作为队列的头部：
	     * 1. 当前 MessageQueue 为空
	     * 2. 当前入队的 msg 的 when 为 0
	     * 3. 当前入队的 msg 的 when 最小
             * */
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 将 msg 插入到 MessageQueue 中
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {// 遍历当前 MessageQueue，找到 msg 需要插入的位置
                    prev = p; // 存储待插入结点的前一个结点
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                // msg 插入到 MessageQueue 中
                msg.next = p;
                prev.next = msg;
            }
            if (needWake) {
                nativeWake(mPtr); // 唤醒队列
            }
        }
        return true;
    }
```

1. enqueueMessage 的入队操作主要以 msg 的 when 为基准，将 msg 插入到消息队列中。其中 when 为 SystemClock.uptimeMillis() + delayMillis。

2. 当出现下面三种情况，直接将当前入队的 msg 作为队列的头部：
	* 当前 MessageQueue 为空
	* 当前入队的 msg 的 when 为 0 时 --- 对应调用 sendMessage 
	* 当前入队的 msg 的 when 为 MessageQueue 中最小

##　next

```
  Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        // 开启一个死循环
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            // We can assume mPtr != 0 because the loop is obviously still running.
            // The looper will not call this method after the loop quits.
            nativePollOnce(mPtr, nextPollTimeoutMillis);
            // 进行同步操作
            synchronized (this) {
                // 尝试检索下一条消息，如果找到则返回
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {// 如果当前队列的头的 msg 还未达到执行时间
                        // 设置超时等到队列头 msg 到达执行时间,将其唤醒
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else { // 取出 MessageQueue 的头 msg
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

                if (mQuitting) { // 当 mQuitting 为 true 时，next 方法返回的是 null
                    dispose();
                    return null;
                }

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null;

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

            pendingIdleHandlerCount = 0;

            nextPollTimeoutMillis = 0;
        }
    }
```

next 是一个死循环方法，主要操作如下:

1. 如果消息队列中没有消息，那么 next 方法会一直阻塞。nativePollOnce 保证在消息队列为空时，next 方法可以处于休眠状态，不会占用 cpu。

2. 当有新消息到来时，next 方法会检查该 msg 是否达到执行时间，如果是，则从 MessageQueue 中移除该 mgs，并执行。
   如果否，next 会继续等待该 msg 到达其执行时间。

