# ConcurrentHashMap

ConcurrentHashMap 是 Java 并发包中提供的一个线程安全且高效的 HashMap 实现。

# ConcurrentHashMap 存储数据


```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
```


```
 /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); // 计算 key 的 hash 值
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) { // 开启一个死循环
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0) // 检查table是否初始化了，如果没有，则调用initTable()进行初始化
                tab = initTable();
            // 根据 key 的 hash 值计算出其应该在 table 中储存的位置 i，取出 table[i] 的节点用f表示
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {// 当 table[i] 为空时，利用 CAS 操作直接存储在该位置
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果 table[i]!=null,则该位置已经有其它节点，发生碰撞
            else if ((fh = f.hash) == MOVED)
            // 检查 table[i] 的节点的 hash 是否等于 MOVED，如果等于，则检测到正在扩容，则帮助其扩容
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {// 链表节点
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 查找链表中是否出现了此 key，如果出现，则更新value，并跳出循环
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 否则将节点加入到链表尾部并跳出循环
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { // 树节点
                            Node<K,V> p;
                            binCount = 2;
                            // 插入到树中
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                // 如果插入的是链表节点，则要判断下该链表是否要转化为树
                // binCount 记录了上面所追加的链表节点数量
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```



```
    // 获取索引　i　处的　Node
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {　
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
    //利用 CAS 算法设置 i 位置上的 Node 节点,即将 c 和 table[i] 比较，相同则插入 v
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {　
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    // 设置节点 i 位置的值，仅在上锁区被调用
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```


```
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 如果 sizeCtl < 0，则说明已经有其它线程正在进行扩容，即正在初始化或初始化完成
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 如果 CAS 成功，则表示正在初始化，设置为 -1，否则说明其它线程已经对其正在初始化或是已经初始化完毕
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {// 再一次检查确认是否还没有初始化
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);// sc = 0.75n
                    }
                } finally {
                    sizeCtl = sc;// sizeCtl = 0.75*Capacity，即为扩容阀值
                }
                break;
            }
        }
        return tab;
    }
```

































