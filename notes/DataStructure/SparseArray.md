# SparseArray

SparseArray(稀疏数组),它是 Android 内部特有的 api。在 Android 内部用来替代 HashMap<Integer,E> 这种形式，它的实现相比于 HashMap 更加节省空间，
而且由于 key 指定为 int 类型，也可以节省 int-Integer 的装箱拆箱操作带来的性能消耗。

# 向 SparseArray 添加数据
 
```
    public void put(int key, E value) {
        // 利用二分查找，找到 待插入key 的 下标index
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        if (i >= 0) {// 如果返回的index是正数，说明之前这个 key 已存在，直接覆盖value即可
            mValues[i] = value;
        } else {  // 若返回的 index 是负数，说明 key不存在。
            i = ~i; // 先对返回的 i 取反，得到应该 key 在数组 mKeys 中应该插入的位置
            //  如果索引 i 小于当前已经存放的长度，并且这个位置上的值为 DELETED(即被标记为删除的值)
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;  // 直接赋值并返回，注意 size 不需要增加
                mValues[i] = value;
                return;
            }
           // 当数组内存不够,但 mKey 中存在无用的数值, 那么调用 gc() 函数进行无用数值的回收
            if (mGarbage && mSize >= mKeys.length) {
                gc();
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }
            // 向数组 mKeys 的 i 位置插入元素 key
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            // 向数组 mValues 的 i 位置，插入元素 value
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```

需要关注的几点：

1. SparseArray 是基于二分法来查找添加数据的应存放的位置 index。

2. 在添加元素的过程中，可能会涉及 gc，insert 以及 SparseArray 的操作，这三个
   操作中都会涉及数组中元素的移动。


#### ContainerHelpers#binarySearch

```
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;// 无符号右移 1 位，即 (lo + hi) / 2
            final int midVal = array[mid]; // 数组中间位的值

            if (midVal < value) {
                lo = mid + 1; // 更新 lo 的值，搜索数组的右半区间
            } else if (midVal > value) {
                hi = mid - 1; // 更新 hi 的值，搜索数组的左半区间
            } else {
                return mid;  // value 值找到
            }
        }
        return ~lo;  // 返回一个负数表示 value 值没找到
    }
```

关注两点：

1. 计算 mid 值时，使用高效的位运算（>>>）

2. 二分查找结果的处理：
   * 返回值大于 0，表示数组中已存在该 value。
   * 返回值小于 0，表示数组中不存在该 value，且该返回值的绝对值即为 value 应该存储在 array 数组的 index 位置。


#### gc 

```
    // 垃圾回收函数,压缩数组
    private void gc() {
        int n = mSize;// 保存 GC 前的集合大小
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;
        // 遍历 values 集合，以下算法 意义为 从 values 数组中，删除所有值为 DELETED 的元素
        for (int i = 0; i < n; i++) {
            Object val = values[i];
        // 如果当前value 没有被标记为已删除
            if (val != DELETED) {
        // 压缩keys、values数组
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
        // 并将当前元素置空，防止内存泄漏
                    values[i] = null;
                }
            // 递增o
                o++;
            }
        }
        // 修改 标识，不需要GC
        mGarbage = false;
        // 更新集合大小
        mSize = o;
    }
```

gc 操作主要是整理 SparseArray 中对数据进行压缩整理：

1. 删除无用的数据

2. 将有用的数据整理到一起。


#### GrowingArrayUtils#insert

在指定索引处向数组中插入元素，如果没有更多空间，则对数组进行扩容。

```
    public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) { // 表示当前数组还能够存放元素
            // 数组的插入操作：将数组从 index 开始的 currentSize - index 个元素依次往后移动一位
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element; // 更新数组 index 位置的值
            return array;
        }
        // 数组扩容
        T[] newArray = (T[]) Array.newInstance(array.getClass().getComponentType(),
                growSize(currentSize));
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index); // 扩容后，对数组执行插入操作
        return newArray;
    }
```


#### GrowingArrayUtils#growSize

数组的扩容算法：

```
   public static int growSize(int currentSize) {
        return currentSize <= 4 ? 8 : currentSize * 2;
    }
```


# 从 SparseArray 删除元素

```
    public void delete(int key) {
        // 二分查找得到要删除的 key 所在 index
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) { // 如果 >=0,表示存在
            if (mValues[i] != DELETED) { // 修改 values 数组对应位置为已删除的标志 DELETED
                mValues[i] = DELETED;
                mGarbage = true;// 修改 mGarbage 为 true,表示稍后需要 GC
            }
        }
    }
```

当删除一个元素时，并不是立即从 value 数组中删除它，并压缩数组，而是将其在 value 数组中标记为已删除。这样当存储相同的
 key 的 value 时，可以重用这个空间。如果该空间没有被重用，随后将在合适的时机里执行gc（垃圾收集）操作，将数组压缩，
以免浪费空间。


# 从 SparseArray 中查询元素


```
    public E get(int key, E valueIfKeyNotFound) {
        // 二分查找得到要删除的 key 所在 index
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {　// 不存在该 key 或 对应的 value 已删除的情况 
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i]; // 返回 value 值
        }
    }
```




















