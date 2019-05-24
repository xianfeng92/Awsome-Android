# ThreadLocal 的工作原理

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


这段代码中，主线程设置了 ThreadLocal 为true。而在子线程1中设置了 false 然后分别获取他们的值。这个时候主线程为true,子线程1中为false，而子线程2中由于没有设置，是 null。

虽然在不同线程中访问的是同一个 ThreadLocal 对象，但是他们通过 ThreadLocal 获取到的值确实不一样的，这就是ThreadLocal的奇妙之处。


ThreadLocal 之所以有这么奇妙的效果，是因为不同线程访问同一个 ThreadLocal 的 get，ThreadLocal 内部会从各自的线程中取出一个数组，然后从数组中根据当前的 ThreadLocal 索引
查出对应的 value 值。很显然，不同线程中的数组是不同的，这就是为什么通过 ThreadLocal 可以在不同的线程中维护一套数据的副本并且彼此互不干扰。

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

可以发现 ThreadLocal 的 get 方法逻辑还算是比较清晰，他同样是取当前线程的 localValues 对象，如果这个对象为 null 那么就返回初始值，初始值由 ThreadLocal 的 initialValue 
方法来描述。默认情况下为null，当然也可以重写这个方法，它的默认实现是：

```
    protected T initialValue() {
        return null;
    }

```

如果localValues对象不为null，那就取出它的table数组并找出 ThreadLocal 的 rederence 对象在 table 数组中的位置，然后 table 数组的下一个位置所存储的数据就是 ThreadLocal 的值。

从 ThreadLocal 的 set 和 get 方法可以看出，它们所操作的对象都是当前线程的 localValues 对象的 table 数组，因此在不同线程中访问同一个 ThreadLocal 的set和get方法，它们对 
ThreadLocal 所做的读写操作仅限于各自线程的内部，这就是为什么 ThreadLocal 可以在多个线程互不干扰的存储和修改数据。
