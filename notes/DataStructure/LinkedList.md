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

将元素节点插入到 succ 之前的位置：

```
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
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

LinkedList 为双向链表，可以从链表头部或者尾部开始遍历。上述代码通过比较 index 与节点数量 size/2 的大小，决定从头结点还是尾节
点进行查找。

