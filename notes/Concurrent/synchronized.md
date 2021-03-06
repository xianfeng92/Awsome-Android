# synchronized

synchronized 可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性。

Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1. 普通同步方法，锁是当前实例对象

2. 静态同步方法，锁是当前类的class对象

3. 同步方法块，锁是括号里面的对象

当一个线程访问同步代码块时，它首先是需要得到锁才能执行同步代码，当退出或者抛出异常时必须要释放锁，那么它是如何来实现这个机制的呢？


### 同步代码块

monitorenter 指令插入到同步代码块的开始位置，monitorexit 指令插入到同步代码块的结束位置，JVM 需要保证每一个 monitorenter 都有一个 monitorexit 
与之相对应。任何对象都有一个 monitor 与之相关联，当且一个 monitor 被持有之后，它将处于锁定状态。线程执行到　monitorenter　指令时，将会尝试获取对象
所对应的 monitor 所有权，即尝试获取对象的锁。

### 同步方法

synchronized 方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被 synchronized 修饰的方法，
而是在 Class 文件的方法表中将该方法的 access_flags 字段中的 synchronized 标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的 Class 在 JVM 
的内部对象表示 Klass 做为锁对象。

### synchronized 和 Volatile

1. volatile 本质是在告诉 jvm 当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取。
　　　synchronized　则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。

2. volatile 仅能使用在变量级别；synchronized 则可以使用在变量、方法、和类级别的。

3. volatile 仅能实现变量的修改可见性，不能保证原子性；而 synchronized 则可以保证变量的修改可见性和原子性。

4. volatile 不会造成线程的阻塞；synchronized 可能会造成线程的阻塞。

5. volatile 标记的变量不会被编译器优化；synchronized 标记的变量可以被编译器优化。
