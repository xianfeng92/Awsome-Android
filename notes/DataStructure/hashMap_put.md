# hashMap

## HashMap 重要参数

```
 /** 
   * 即：容量、加载因子、扩容阈值（要求、范围均相同）
   */
   // 1.容量（capacity)：必须是 2 的幂 & 最大容量（2 的 30 次方）
  // 默认容量 = 16 = 1<<4 = 00001中的1向左移4位 = 10000 = 十进制的2^4=16
  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
  // 最大容量 =  2的30次方（若传入的容量过大，将被最大值替换）
  static final int MAXIMUM_CAPACITY = 1 << 30;

  // 2.加载因子(Load factor)：HashMap 在其容量扩容前可达到多满的一种尺度
  final float loadFactor; // 实际加载因子
  static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认加载因子 = 0.75

  //3.扩容阈值（threshold）：当哈希表非空 bucket 的数量 ≥ 扩容阈值时，就会扩容哈希表（即扩充 HashMap 的容量） 
  // a.扩容 = 对哈希表进行 resize 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数 
  // b.扩容阈值 = 容量 x 加载因子
  int threshold;

  // 4. 其他
  transient Node<K,V>[] table;  // 存储数据的 Node 类型数组，长度 = 2的幂；数组的每个元素 = 1个单链表
  transient int size;// HashMap的大小，即 HashMap 中存储的键值对的数量
```

## hashMap 构造函数

```
  /**
     * 构造函数1：默认构造函数（无参）
     * 加载因子为 0.75
     * 容量为 16
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

```
    /**
     * 构造函数2：指定“容量大小”的构造函数
     * 加载因子 = 默认 = 0.75 、容量 = 指定大小 initialCapacity
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

```
    /**
     * 构造函数3：指定“容量大小”和“加载因子”的构造函数
     * 加载因子 & 容量 = 自己指定
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // 设置 扩容阈值
        // 注：此处不是真正的阈值，仅仅只是将传入的容量大小转化为：> 传入容量大小的最小的2的幂，该阈值后面会重新计算
        this.threshold = tableSizeFor(initialCapacity);
    }
```

```
    /**
     * 构造函数4：包含“子Map”的构造函数
     * 即 构造出来的HashMap包含传入Map的映射关系
     * 加载因子 & 容量 = 默认
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        // 设置容量大小 & 加载因子 = 默认
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

1. 此处仅用于接收初始容量大小（capacity）、加载因子(Load factor)，但仍无真正初始化哈希表，即初始化存储数组table

2. 真正初始化哈希表（初始化存储数组table）是在第1次添加键值对时，即第1次调用put（）时。

## 向HashMap添加数据

```
    public V put(K key, V value) {
        // 1. 计算传入的键 key 的 hash 值 ->>分析1
        // 2. 再调用putVal（）添加数据进去 ->>分析2
        return putVal(hash(key), key, value, false, true);
    }
```

下面，将详细分析上面的2个主要分析点：

### hash(key)

```
    // 分析1：hash(key)
    // 作用：计算传入 key 的哈希码（哈希值、Hash值）
    static final int hash(Object key) {
        int h;
        // 将键 key 转换成 hash 值操作 = key.hashCode() + 1次位运算(无符号右移) + 1次异或运算  2次扰动
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

补充：

1. 当key = null时，hash值 = 0，所以 HashMap 的 key 可为 null。对比 HashTable，HashTable 对 key 直接 hashCode（），若 key 为 null 时，
   会抛出异常，所以 HashTable 的 key 不可为 null。

2. 当key ≠ null时，则通过先计算出 key 的 hashCode()（记为h），然后对 h 进行扰动处理：(h) ^ (h >>> 16)。(h) ^ (h >>> 16) 就是为了加大
   哈希码低位的随机性，使得分布更均匀，从而提高对应数组存储下标位置的随机性 & 均匀性，最终减少Hash冲突。

4. hash(key) & (length-1) 即为该 key 在 hashMap 的 table 数组中位置。
   length 为 hashMap 容量（capacity)：是 2 的幂，那么 length -1 的二进制表示各位全为 1。这样 hash(key) & (length-1)，即为取 hash(key) 值的
   低几位（具体位数由 length 大小决定）来做 key 在 hashMap 中的数组下标。这里使用了__位运算替代取余操作，更加高效__。


### putVal(hash(key), key, value, false, true)

此处有2个主要讲解点：

1. 计算完存储位置后，具体该如何 存放数据 到哈希表中

2. 具体如何扩容，即 扩容机制


#### 1：计算完存储位置后，具体该如何存放数据到哈希表中

由于数据结构中加入了红黑树，所以在存放数据到哈希表中时，需进行多次数据结构的判断：数组、红黑树、链表。

```
   /**
     * 分析2：putVal(hash(key), key, value, false, true)
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //1. 若哈希表的数组tab为空，则通过 resize() 创建
        // 所以，初始化哈希表的时机 = 第1次调用 put 函数时，即调用 resize() 初始化创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 2. 计算插入存储的数组索引i：根据键值 key 计算的 hash 值得到
        // 3. 插入时，需判断是否存在 Hash 冲突：
        if ((p = tab[i = (n - 1) & hash]) == null)
         // 若不存在（即当前tab[i = (n - 1) & hash] == null），则直接在该数组位置新建节点，插入完毕
         // 否则，代表存在 Hash 冲突，即当前存储位置已存在节点，则依次往下判断:
         // a. 当前位置的 key 是否与需插入的 key 相同
        // b. 判断需插入的数据结构是否为红黑树 or 链表
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // a. 判断 table[i] 的元素的 key 是否与需插入的 key 一样。若一样，则直接用新 value 覆盖旧 value
           // 判断原则：equals（）
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // b. 继续判断：需插入的数据结构为红黑树 or 链表
           // 若是红黑树，则直接在树中插入 or 更新键值对
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 若是链表,则在链表中插入 or 更新键值对
                // i. 遍历table[i]，判断 Key是否已存在：采用equals（） 对比当前遍历节点的key 与 需插入数据的key：若已存在，则直接用新value 覆盖 旧value 
                // ii. 遍历完毕后仍无发现上述情况，则直接在链表尾部插入数据 
                // 注：新增节点后，需判断链表长度是否 >8（8 = 桶的树化阈值）：若是，则把链表转换为红黑树
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {// 遍历到尾部，追加新节点到尾部
                        p.next = newNode(hash, key, value, null);
                        // 如果追加节点后，链表数量 >= 8，则转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 在遍历链表过程中如果出现 key 值相等的情况，则直接退出。此时 e 即为需要覆盖值的节点，后面会将其 value 值覆盖
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 更新 p 指向下一个节点，继续遍历
                    p = e;
                }
            }
            // 对 i 情况的后续操作：发现 key 已存在，直接用新 value 覆盖旧 value & 返回旧 value
            if (e != null) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        // 修改 modCount
        ++modCount;
        // 更新 size，并判断是否需要扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

#### 2：扩容机制（即 resize（）函数方法）

```
 // 该函数有2种使用情况：
    // 1.初始化哈希表
    // 2.当前数组容量过小，需扩容
    final Node<K,V>[] resize() {
        // 扩容前的 table 数组（当前数组）
        Node<K,V>[] oldTab = table;
        // 扩容前的 table 数组的容量 = 长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 扩容前的 table 数组的阈值
        int oldThr = threshold;
        // 初始化新的容量和阈值为0
        int newCap, newThr = 0;
        // 如果当前容量大于0
        if (oldCap > 0) {
            // 针对情况2：若扩容前的数组容量超过最大值，则不再扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 则设置阈值是2的31次方-1
                threshold = Integer.MAX_VALUE;
                // 同时返回当前的哈希桶，不再扩容
                return oldTab;
            }// 否则新的容量为旧的容量的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY) // 如果旧的容量大于等于默认初始容量16
                newThr = oldThr << 1; // double threshold 新的阈值也等于旧的阈值的两倍
        }// 如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;//此时新表的容量为默认的容量 16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//新的阈值为默认容量16 * 默认加载因子0.75f = 12
        }
        if (newThr == 0) {//如果新的阈值是0，对应的是当前表是空的，但是有阈值的情况
            float ft = (float)newCap * loadFactor;
            // 进行越界修复
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        // 更新阈值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            // 根据新的容量构建新的哈希桶
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //更新哈希桶引用
        table = newTab;
        //如果以前的哈希桶中有元素
        if (oldTab != null) {
            // 下面开始将当前哈希桶中的所有节点转移到新的哈希桶中
            for (int j = 0; j < oldCap; ++j) {
                // 取出当前的节点 e
                Node<K,V> e;
                // 如果当前桶中有元素,则将链表赋值给 e
                if ((e = oldTab[j]) != null) {
                    // 将原哈希桶置空以便 GC
                    oldTab[j] = null;
                    // 如果当前链表中就一个元素，（没有发生哈希碰撞）
                    if (e.next == null)
                        // 直接将这个元素放置在新的哈希桶里
                        // 注意这里取下标是用 哈希值 与 桶的长度-1.由于桶的长度是 2 的n次方，这么做其实是等于一个模运算,但是效率更高.
                        newTab[e.hash & (newCap - 1)] = e;
                        // 如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                        // 如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置
                    else {
                        // 因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即low位，或者扩容后的下标，即 high 位。high位 =  low位 + 原哈希桶容量
                        // 低位链表的头结点、尾节点
                        Node<K,V> loHead = null, loTail = null;
                        // 高位链表的头节点、尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        // 临时节点存放 e 的下一个节点
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 将扩容前的链表拆分为两条链，一条链位于原 index 处，另一条链位于原 index + oldCap
                           //  拆分规则是：当前节点的 hash 值的第 N 位为 0 或 1。其中 N 为 oldCap 的二进制的位数。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }//循环直到链表结束
                        } while ((e = next) != null);
                        // 将低位链表存放在原 index 处
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 将高位链表存放在新index处，即 原index + oldCap
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

