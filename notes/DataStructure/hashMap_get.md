# 从 HashMap 中获取数据

```
    // 根据键key，向 HashMap 获取对应的值
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

```
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 1. 计算该 key 存放在数组 table 中的位置
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null){
            // a. 先在数组中找，若存在，则直接返回。即，对 tab[(n - 1) & hash]) 对应的 key 的判断
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // b. 若数组中没有，则到红黑树中寻找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    // c. 若红黑树中也没有，则通过遍历，到链表中寻找。循环该链表，直到找到 key。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

# 对HashMap的其他操作

HashMap除了核心的put（）、get（）函数，还有以下主要使用的函数方法。

```
void clear(); // 清除哈希表中的所有键值对 
int size(); // 返回哈希表中所有 键值对的数量 = 数组中的键值对 + 链表中的键值对 
boolean isEmpty(); // 判断HashMap是否为空；size == 0时 表示为 空 
void putAll(Map<? extends K, ? extends V> m); // 将指定Map中的键值对 复制到 此Map中 
V remove(Object key); // 删除该键值对 
boolean containsKey(Object key); // 判断是否存在该键的键值对；是 则返回true 
boolean containsValue(Object value); // 判断是否存在该值的键值对；是 则返回true
```












