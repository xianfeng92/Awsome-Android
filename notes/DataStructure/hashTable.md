# Hashtable

## Hashtable 简介

和 HashMap 一样，Hashtable 也是一个散列表，它存储的内容是键值对(key-value)映射。Hashtable 继承于 Dictionary，实现了 Map、Cloneable、
java.io.Serializable 接口。Hashtable 的函数都是同步的，这意味着它是线程安全的。它的 key、value 都不可以为 null。此外，Hashtable 中
的映射不是有序的。

## Hashtable 构造函数

```
// 默认构造函数。
public Hashtable()

// 指定“容量大小”的构造函数
public Hashtable(int initialCapacity)

// 指定“容量大小”和“加载因子”的构造函数
public Hashtable(int initialCapacity, float loadFactor) 

// 包含“子Map”的构造函数
public Hashtable(Map<? extends K, ? extends V> t)
```

## Hashtable 添加数据

```
    // 方法为同步的，确保了线程安全
    public synchronized V put(K key, V value) {
        // hashTable 中的 value 不可为 null ，否则会报 NullPointerException
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();// 只是简单计算 key 的 hashCode，没有进行扰动处理
        int index = (hash & 0x7FFFFFFF) % tab.length; // 通过对 hash 值的取余来获取 key 的存储位置。没有 hashMap 的位操作高效
        @SuppressWarnings("unchecked")
        HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
        // 遍历链表，查看 hashTable 中是否已经有该 key。如果有，则直接更新 value
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        // 如果没有，则将该键值对追加到链表上
        addEntry(hash, key, value, index);
        return null;
    }
```

添加键值对的逻辑很简单:

1. Hashtable 如果已经存在该 key 了，直接更新其 value 值

2. 否则，直接在 Hashtable 中追加该键值对（Entry）

具体追加键值对 addEntry 方法如下：

```
  private void addEntry(int hash, K key, V value, int index) {
        modCount++;
        HashtableEntry<?,?> tab[] = table;
        // 超过 hashTable 的阀值，则需要进行扩容... 暂时不讨论
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        // 保存 HashtableEntry 数组的 index 位置的元素
        HashtableEntry<K,V> e = (HashtableEntry<K,V>) tab[index];
        // 创建一个新的 HashtableEntry，直接挂载到 HashtableEntry 数组的 index 位置
        // 注意了，在构建 HashtableEntry 时，传入了原本 index 位置的元素 e，这样可以将新的 HashtableEntry 的 next 指向 e
        tab[index] = new HashtableEntry<>(hash, key, value, e);
        count++;
    }
```

1. 往 hashTable 中添加元素时，先检测是否需要扩容，然后再追加元素

   这一点和 hashMap 刚好相反～～～

2. 当存在 hash 冲突时，hashTable 中添加元素是从链表头追加的

   这一点和 hashMap 刚好相反～～～

3. 扩容后，index = (hash & 0x7FFFFFFF) % tab.length

   hashMap 在扩容后会对链表上的键值对的 index 重新计算，进而提高查询效率

## Hashtable 获取数据

```
    // 方法为同步的，确保了线程安全
    public synchronized V get(Object key) {
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        // 根据 key 的 hash 值，计算出其在 hashTable 中的位置
        int index = (hash & 0x7FFFFFFF) % tab.length;
        // 获取 hashTable 中指定位置是否存在键为 key 的元素，然后存在则直接返回其 value
        for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
            // 用 hash 值 和 equals() 来判断
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```

1. (e.hash == hash) && e.key.equals(key)
   
   先判断 hash 值，再判断 key 值。


## hashTable 扩容

```
    protected void rehash() {
        int oldCapacity = table.length;
        HashtableEntry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;// 扩容 2 倍
        // 边界检测
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        HashtableEntry<?,?>[] newMap = new HashtableEntry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;
        // 将旧 hashTable 中的元素转移到扩容后的 hashTable 中
        for (int i = oldCapacity ; i-- > 0 ;) {
            for (HashtableEntry<K,V> old = (HashtableEntry<K,V>)oldMap[i] ; old != null ; ) {
                HashtableEntry<K,V> e = old;
                old = old.next;
                // 从链头插入元素
                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (HashtableEntry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```

































