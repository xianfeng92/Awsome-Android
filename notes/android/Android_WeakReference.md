## WeakReference

在Java里, 当一个对象O被创建时, 它被放在 Heap里, 当GC运行的时候, 如果发现没有任何引用指向O, O就会被回收以腾出内存空间。
一个对象被回收, 必须满足两个条件:

* 没有任何引用指向它 
* GC被运行

在现实情况写代码的时候, 我们往往通过把所有指向某个对象的referece置空来保证这个对象在下次GC运行的时候被回收。

```
Object c = new Car();
c=null;
```

但是, 手动置空对象对于程序员来说, 是一件繁琐且违背自动回收的理念的。

当一个对象仅仅被　weak reference指向, 而没有任何其他strong reference指向的时候, 如果GC运行, 那么这个对象就会被回收. weak reference的语法是：

```
WeakReference<Car> weakCar = new WeakReference(Car)(car);
```
当要获得weak reference引用的object时, 首先需要判断它是否已经被回收:

```
weakCar.get();

```

如果此方法为空, 那么说明weakCar指向的对象已经被回收了。




