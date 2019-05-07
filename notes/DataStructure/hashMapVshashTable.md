# hashMap VS hashTable

1. hashMap 中使用位运算替代取余操作，更加高效

2. 对 hash 值进行扰动处理，减少 hash 冲突

3. 扩容时对原有的 hash 冲突形成的链表进行拆分，进而提高查询效率

4. 增加红黑树，当 hash 冲突形成的链表长度大于 8 时，树形化存储，提高增删改查的效率

5. hashMap 线程不安全,hashTable 为线程安全
