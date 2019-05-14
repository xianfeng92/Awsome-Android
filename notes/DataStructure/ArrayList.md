# ArrayList

## 重要属性

```
private static final int DEFAULT_CAPACITY = 10; // 默认容量

private static final Object[] EMPTY_ELEMENTDATA = {};// 空的对象数组

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};// 默认的空数组
// 存放数据的数组的缓存变量，不可序列化
transient Object[] elementData;

```

##　ArrayList　查询元素

```
    public E get(int index) {
        if (index >= size) // 判断 index 是否越界
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        return (E) elementData[index];
    }
```
ArrayList　查询元素操作很简单，直接通过索引值 index 去获取对应的元素值，时间复杂度为 O(1)。


## ArrayList　更新元素

```
    public E set(int index, E element) {
        if (index >= size)// 判断 index 是否越界
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        E oldValue = (E) elementData[index];// 缓存旧值
        elementData[index] = element; // 存储新值
        return oldValue;
    }
```
ArrayList　更新元素操作也很简单，直接通过索引值来更新元素的值，时间复杂度为 O(1)。


## ArrayList 添加元素

在 ArrayList 尾部追加元素：

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;// 在数组末尾追加一个元素，并修改 size
        return true;
    }
```

对于数组的追加元素操作，可能会涉及到数组扩容操作，该操作中需要将旧数组的元素移动到新数组中。


在 ArrayList 指定位置插入元素：

```
    public void add(int index, E element) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

对于数组中插入元素的操作，需要整体将插入位置的元素往后移动一位。同时，可能也会涉及到数组的扩容操作。

```

    private void ensureCapacityInternal(int minCapacity) {
         // 利用 == 可以判断数组是否是用默认构造函数初始化的
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

判断是否需要扩容：

```
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

帮助　ArrayList　动态扩容的核心方法：

```
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // 扩容 1.5 倍
        if (newCapacity - minCapacity < 0)// 如果扩容后的长度仍然小于指定的容量
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0) // 如果扩容后的容量大于最大数组大小
            newCapacity = hugeCapacity(minCapacity); // 调整 newCapacity 大小，其最大值为 0x7fffffff
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

其中方法：Arrays.copyOf(elementData, size, Object[].class) 会根据 class 的类型来决定是 new 还是反射去构造一个泛型数组，
同时利用 native 函数，批量赋值元素至新数组中。

```
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        // 根据class的类型来决定是new 还是反射去构造一个泛型数组
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        // 利用native函数，批量赋值元素至新数组中。
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

```
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

## ArrayList 删除元素

```
    public E remove(int index) {
        if (index >= size) //　判断 index 是否越界
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;　// 需要移动的元素数量
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

ArrayList 删除元素的操作中也需要对数组中的元素进行移动, 将删除的 index 之后的元素全部向前移动一位。


## ArrayList　元素迭代

```
    public Iterator<E> iterator() {
        return new Itr();
    }
```

```
  private class Itr implements Iterator<E> {
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < limit;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor;
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
                limit--;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;

            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

## 小结

1. 对 ArrayList 的查询和更新操作的时间复杂度为 O(1),非常高效。

2. 往 ArrayList 末尾追加元素时，时间复杂度为 O(1), 但是需要扩容时，时间复杂度就退化为 O(n)。
   当向 ArrayList 中插入元素时，由于会涉及元素的移动，其效率较低。即，ArrayList　不适合应用在
　　需要频繁插入元素的场合。

3. ArrayList 删除尾部元素，时间复杂度为 O(1), 但是如果删除的是　ArrayList　的中间元素，时间复杂度退化为 O(n)。

4. 因此，结合特点，在使用中，以 Android 中最常用的 listView 为例，列表滑动时需要展示每一个 Item 元素，所以 查 操作是最高频的。
   相对来说，增操作 只有在列表加载更多时才会用到 ，而且是在列表尾部添加，所以也不需要移动数据的操作。而删操作则更低频。故选用　
　　　ArrayList作为保存数据的结构。

