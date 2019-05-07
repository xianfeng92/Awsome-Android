# hashMap 如何减少和处理 Hash 冲突

1. hash 算法确保 hash 值分布的均匀和随机性

   计算 key 的 hashCode 值 h，然后对 h 做扰动处理： h ^ (h >>> 16)。这样可以确保 key 的 hash 值的均匀和随机性，尽可能
   的减少 hash 冲突。


2. 采用扩容机制，减少 hash 冲突的几率

   当哈希表的大小 >= 扩容阀值时，就会扩容哈希表(即，hashMap 的容量)。扩容后的 hashMap 也一定程度上会减小 hash 冲突。


3. 发生 hash 冲突后，采用链表或红黑树来存储

   当发生 hash 冲突后，采用链地址法 + 尾插法 + 红黑树（链长度 > 8 时使用）

# 为什么 HashMap 中 String、Integer 这样的包装类适合作为 key 键

String，Integer 等包装类的特性：

1. final 类型，具体不可变性，保证了 key 的不可更改性，即不会出现放入 & 获取时的哈希值不同的情况。

2. 内部已经重写 equals() 和 hashCode(),严格遵守相关规范，不容易出现 hash 值的计算错误。

这些特性保证了 hash 值的不可更改性和计算准确性，可以有效减少发生 hash 碰撞的几率。

# HashMap 中的 key若 Object类型，则需实现哪些方法？

需要实现如下两个方法：

1. hashCode()

   用于计算存储数据的存储位置，实现不恰当会导致严重的 hash 碰撞 

2. equals()
   
   比较存储位置上是否存在需要存储数据的 key，若存在则直接更新值 value 即可，否则需要插入数据。这样可以
   保证 key 在哈希表中的唯一性。

# HashMap 多线程下可能出现 resize（）死循环

HashMap 线程不安全的其中一个重要原因：多线程下容易出现 resize() 死循环。本质 = 并发执行 put（）操作导致触发扩容行为，从而导致环形链表，
使得在获取数据遍历链表时形成死循环，即 Infinite Loop。
