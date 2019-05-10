# SparseArray

SparseArray(稀疏数组),它是 Android 内部特有的 api,标准的 jdk 是没有这个类的。在 Android 内部用来替代 HashMap<Integer,E> 这种形式
,使用 SparseArray 更加节省内存空间的使用, SparseArray 也是以 key 和 value 对数据进行保存的。使用的时候只需要指定 value 的类型即可，
并且 key 不需要封装成对象类型。

# 向 SparseArray 添加数据
 
```
    public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);// 在数组 mKeys 对 key 进行二分查找。数组 mKeys 为有序的。
        if (i >= 0) {
            mValues[i] = value;// key 已存在, 更新 value
        } else { // 如果数组内部不存在 key 的话,那么返回的数值是负数
            i = ~i;// 取反后的 i 即为 key 在数组 mKeys 应该存放的位置
            // i 值小于 mSize 表示在这之前. mKey 和 mValue 数组已经被申请了空间，只是键值被删除了。
           // 那么当再次保存新的值的时候.不需要额外的开辟新的内存空间，直接对数组进行赋值即可.
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
           // 当内存不够,但是 mKey 中存在无用的数值,那么调用 gc() 函数进行无用数值的回收
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
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




























