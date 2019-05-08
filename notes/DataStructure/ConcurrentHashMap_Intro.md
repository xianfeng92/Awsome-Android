# ConcurrentHashMap

ConcurrentHashMap 是 Java 并发包中提供的一个线程安全且高效的 HashMap。ConcurrentHashMap 底层数据结构与 HashMap 相同，仍然采用 table 数组+链表+红黑树结构。

## 基本数据结构

```
// 键值对桶数组
transient volatile Node<K,V>[] table;
// resizing 扩容时用到的新键值对数组
private transient volatile Node<K,V>[] nextTable;
// 记录当前键值对总数，通过 CAS 更新，对所有线程可见
private transient volatile long baseCount;
// 控制标识符
private transient volatile int sizeCtl;
// 自旋锁
private transient volatile int cellBusy；
// counter cell表，长度总为2的幂次
private transient volatile CounterCell[] counterCells;

// 视图
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
```

sizeCtl 为控制标识符：

* 当 sizeCtl < 0　时，代表正在对表进行初始化或扩容操作
  其中　-1 代表初始化，-N 表示有　N-1　个线程正在进行扩容操作

* 当 sizeCtl = 0　时，默认值

* 当 sizeCtl > 0　时，表示下一次的扩容阈值

## 重要的内部类

###  Node

Node 是最核心的内部类，它包装了 key-value 键值对，所有插入 ConcurrentHashMap 的数据都包装在这里面。它对 value 和 next 属性设置了 volatile 同步锁，
它不允许调用 setValue 方法直接改变 Node 的 value 域，它增加了 find 方法辅助 map.get() 方法。 


```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;// 带有同步锁的 value
        volatile Node<K,V> next;// 带有同步锁的 next 指针

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()     { return key; }
        public final V getValue()   { return val; }
        public final int hashCode() { return key.hashCode() ^ val.hashCode(); }
        public final String toString() {
            return Helpers.mapEntryToString(key, val);
        }
        // 不允许调用 setValue 方法直接改变 Node 的 value 域
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```

## 构造函数

```
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```















































