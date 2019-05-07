# ConcurrentHashMap

ConcurrentHashMap 是 Java 并发包中提供的一个线程安全且高效的 HashMap。ConcurrentHashMap 底层数据结构与 HashMap 相同，仍然采用 table 数组+链表+红黑树结构。

# ConcurrentHashMap 获取数据

```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());// 计算 key 的 hash 值
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {// 检查头结点。如果头结点的 key 即为要获取的 key,则直接返回其 value
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)// table[i]为一颗树
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {// table[i]为一个链表，遍历链表,找出为 key 的结点并返回其 value
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```
 
