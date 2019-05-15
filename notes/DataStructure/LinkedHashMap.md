# LinkedHashMap

LinkedHashMap 继承自 HashMap，在 HashMap 基础上，__通过维护一条双向链表，解决了 HashMap 不能随时保持遍历顺序和插入顺序一致的问题__。
在实现上，LinkedHashMap 很多方法直接继承自 HashMap，仅为维护双向链表覆写了部分方法。LinkedHashMap 是 HashMap 的一个 Wrapper。

![LinkedHashMap]()

LinkedHashMap 在 HashMap 的基础上，采用双向链表（doubly-linked list）的形式将所有 entry 连接起来，这样是为保证元素的迭代顺序跟插入顺序相同。
上图给出了 LinkedHashMap 的结构图，主体部分跟 HashMap 完全一样，多了header指向双向链表的头部（dummy 节点），该双向链表的迭代顺序就是 entry 的
插入顺序。除了可以确保迭代顺序，这种结构还有一个好处：迭代 LinkedHashMap 时不需要像 HashMap 那样遍历整个table，而只需要直接遍历 header 指向的
双向链表即可，也就是说 LinkedHashMap 的迭代时间就只跟　entry　的个数相关，而跟　table　的大小无关。


有两个参数可以影响 LinkedHashMap 的性能：初始容量（inital capacity）和负载系数（load factor）。初始容量指定了初始table的大小，负载系数用来
指定自动扩容的临界值。当　entry　的数量超过　capacity*load_factor　时，容器将自动扩容并重新哈希。对于插入元素较多的场景，将初始容量设大可以减少重新
哈希的次数。

将对象放入到 LinkedHashMap 或 LinkedHashSet 中时，有两个方法需要特别关心：hashCode()和equals()。__hashCode() 方法决定了对象会被放到哪个 bucket 里__，
__当多个对象的哈希值冲突时，equals() 方法决定了这些对象是否是“同一个对象”__。所以，如果要将自定义的对象放入到 LinkedHashMap 或 LinkedHashSet 中，
需要重写 hashCode() 和 equals() 方法。


## 重要属性

```
    // HashMap.Node<K,V> 基础上增加了 before 和 after，用以实现双向链表
    static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    //双向链表的头结点
   transient LinkedHashMapEntry<K,V> head;
   //双向链表的尾节点
   transient LinkedHashMapEntry<K,V> tail;

  // 节点元素的访问顺序，默认为 false
  // 为 true 时， 迭代时输出的顺序是按照访问节点的顺序
  // 为 false 时，迭代时输出的顺序是插入节点的顺序
  final boolean accessOrder;
```


## 链表的建立过程

LinkedHashMap 重写了构建新节点的 newNode()方法，其源码如下：

```
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        // 实例化一个新的节点
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p); // 将新节点链接到双向链表的尾部
        return p;
    }

```

在链表尾部添加节点：

```
    private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
        LinkedHashMapEntry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else { // 将新节点链接到双向链表的尾部
            p.before = last;
            last.after = p;
        }
    }
```

由上述代码可知，在每次构建新节点时，LinkedHashMap 通过 linkNodeLast(p) 将新节点链接到内部双向链表的尾部。

1. 初始情况下，让 LinkedHashMap 的 head 和 tail 引用同时指向新节点 p，链表就算建立起来了

2. 随后不断有新节点插入，通过将新节点接在 tail 引用指向节点的后面，即可实现链表的更新

## LinkedHashMap 添加数据

put(K key, V value) 方法是将指定的 key, value　对添加到 map　里。该方法首先会对map做一次查找，看是否包含该元组。如果已经包含则
直接返回，查找过程类似于 get() 方法；如果没有找到，则会通过 addEntry(int hash, K key, V value, int bucketIndex)
方法插入新的 entry。

注意，这里的插入有两重含义：

1. 从 table 的角度看，新的 entry 需要插入到对应的 bucket 里，当有哈希冲突时，采用头插法将新的 entry 插入到冲突链表的头部。

2. 从 header 的角度看，新的 entry 需要插入到双向链表的尾部。

![LinkedHashMapaddEntry]()



## 链表节点的删除过程

LinkedHashMap 删除操作相关的代码也是直接用父类的实现，删除节点后父类会回调 afterNodeRemoval 方法。LinkedHashMap
通过该方法在其维护的链表中移除被删除节点。

```
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null; // 将 p 节点置为 null，便于 GC
        if (b == null)
            head = a;// 如果 p 的前驱节点为 null，就直接将 head 作为其前驱节点
        else
            b.after = a; // 如果 p 的前驱节点不为 null，将其指向 p 的后继节点
        if (a == null)
            tail = b; // 如果 p 的后继节点为 null，直接将 tail 指向 p 的前驱节点
        else
            a.before = b; // 如果 p 的后继节点不为 null ，将 a 的前驱节点指向 b
    }
```

链表中删除节点操作需注意边界值情况，即所要移除的节点为头节点或尾节点。


## 访问顺序的维护过程

默认情况下，LinkedHashMap 是按插入顺序维护链表。不过我们可以在初始化 LinkedHashMap，指定 accessOrder 参数为 true，即可让它按访问
顺序维护链表。访问顺序的原理上并不复杂，当我们调用 get/getOrDefault/replace 等方法时，只需要将这些方法访问的节点移动到链表的尾部即可。
相应的源码如下:

```
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

将最近一次访问的节点 e 移动到双向链表的尾部：

```
   void afterNodeAccess(Node<K,V> e) {
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

该方法的执行逻辑如下:

1. 当节点 p 为头节点时，将 head 引用指向 p 的后继节点 a。否则，将 p 的前驱节点的 after 引用指向 p 的后继节点

2. 当节点 p 为尾节点时，直接将 last 引用指向 p。否则，将 p 的后继节点的 before 引用指向 p 的前驱节点。步骤1、2 
   主要讲节点 p 从双向链表中拆解出来。

3. 将节点 p 链接到双向链表的尾部






























































