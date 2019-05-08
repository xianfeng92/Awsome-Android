# CAS

## Java中的原子操作

原子操作指的是在一步之内就完成而且不能被中断。原子操作在多线程环境中是线程安全的，无需考虑同步的问题。在java中，下列操作是原子操作:

* all assignments of primitive types except for long and double
  除 long 和 double 之外的所有基元类型的赋值

* all assignments of references
  所有引用的赋值

* all operations of java.concurrent.Atomic* classes
　　java 中的Atomic类的操作

* all assignments to volatile longs and doubles
　　volatile 所修饰的　longs 和 doubles　类型的赋值

问题来了，为什么long型赋值不是原子操作呢？例如:

```
long foo = 65465498L;
```

实时上 java 会分两步写入这个 long 变量，先写32位，再写后32位。这样就线程不安全了。

如果改成下面的就线程安全了：

```
private volatile long foo;
```

因为 volatile 内部已经做了 synchronized。


## CAS 无锁算法

要实现无锁（lock-free）的非阻塞算法有多种实现方法，其中 CAS（比较与交换，Compare and swap）是一种有名的无锁算法。CAS 的语义是“我认为 V 的值应该为 A，如果是，
那么将 V 的值更新为 B，否则不修改并告诉 V 的值实际为多少”,CAS 是项乐观锁技术，当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，
而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。CAS 有3个操作数，内存值 V，旧的预期值 A，要修改的新值 B。当且仅当预期值 A
和内存值 V 相同时，将内存值 V 修改为 B，否则什么都不做。


## CAS 优缺点

### 优点

* 高效的解决了原子操作问题


### 缺点

* 循环时间长开销很大

* 只能保证一个共享变量的原子操作

* ABA问题

  如果内存地址V初次读取的值是A，并且在准备赋值的时候检查到它的值仍然为A，那我们就能说它的值没有被其他线程改变过了吗？

  如果在这段期间它的值曾经被改成了B，后来又被改回为A，那 CAS 操作就会误认为它从来没有被改变过。这个漏洞称为 CAS 操作的“ABA”问题。Java并发包为了解决这个问题，
  提供了一个带有标记的原子引用类 “AtomicStampedReference”，它可以通过控制变量值的版本来保证 CAS 的正确性。因此，在使用 CAS 前要考虑清楚“ABA”问题是否会影响程
  序并发的正确性，如果需要解决 ABA 问题，改用传统的互斥同步可能会比原子类更高效。





































