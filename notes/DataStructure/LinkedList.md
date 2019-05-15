# LinkedList

## 重要属性

```
transient Node<E> first; // 指向链表头部节点

transient Node<E> last; // 指向链表尾部节点

transient int size = 0; // 记录链表的节点数

```

```
    private static class Node<E> { // 节点的结构体
        E item; // 节点值
        Node<E> next;//  前驱节点
        Node<E> prev; // 后继节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

## LinkedList 添加元素

```
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
```

在链表尾部插入元素：

```
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```


在链表指定位置插入元素:

```
    public void add(int index, E element) {
        checkPositionIndex(index);
        // 判断 index 是不是链表尾部位置。如果是，直接将元素节点插入链表尾部即可
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```

其中 node(index) 为获取链表 index 位置的节点。


将元素节点插入到 succ 之前的位置：

```
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null) // succ 为链表头节点时，直接将 newNode 更新为头节点
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

## LinkedList 查找元素

```
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
```

查找链表中 index 位置的节点：

```
    Node<E> node(int index) {
        if (index < (size >> 1)) { 
            Node<E> x = first;// 查找位置 index 小于节点数量的一半,则从链表头开始查找
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;// 从链表尾部开始查找
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

使用二分查找来看 index 离 size 中间距离来判断是从头结点正序查还是从尾节点倒序查。node() 会以O(n/2)的性能去获取一个结点。
如果索引值大于链表大小的一半，那么将从尾结点开始遍历。这样的效率是非常低的，特别是当 index 越接近 size 的中间值时。


## LinkedList 移除元素

remove() 方法也有两个版本，一个是删除跟指定元素相等的第一个元素remove(Object o)，另一个是删除指定下标处的元素remove(int index)。

两个删除操作都要：

1. 先找到要删除元素的引用

2. 修改相关引用，完成删除操作


删除跟指定元素相等的第一个元素:

```
    public boolean removeFirstOccurrence(Object o) {
        return remove(o);
    }
```

```
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```

```
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) { // 删除元素是头节点的情况
            first = next;
        } else {
            prev.next = next;
            x.prev = null;　// 将 x 的前驱引用置 null，便于 GC
        }

        if (next == null) { // 删除元素是尾节点的情况
            last = prev;
        } else {
            next.prev = prev;
            x.next = null; // 将 x 的后继引用置 null,便于 GC
        }

        x.item = null;// 将 x 的值置 null
        size--;// 更新链表的节点数
        modCount++;　// 更新链表的修改次数
        return element;
    }
```

在寻找被删元素引用的时候 remove(Object o)调用的是元素的 equals 方法，而 remove(int index)使用的是下标计数，两种方式都是线性时间复杂度。
两个 revome() 方法都是通过 unlink(Node<E> x) 方法完成的。这里需要考虑删除元素是第一个或者最后一个时的边界情况。


## 小结

1. LinkedList　的头节点或尾节点出插入和删除元素只需要更新相关节点的引用，其效率很高。

2. LinkedList 的中间插入或删除元素时，会涉及链表的遍历，其效率会降低，具体受节点在链表中的位置影响。

3. LinkedList　查找操作也会涉及链表的遍历，其效率会降低，具体受节点在链表中的位置影响。


## ArrayList 与 LinkedList

* ArrayList 基于动态数组实现，LinkedList 基于双向链表实现

* ArrayList 支持随机访问，LinkedList 不支持

* LinkedList 在任意位置添加删除元素更快。

