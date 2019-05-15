# HashSet

对于 HashSet 而言，它是基于 HashMap 实现的，HashSet 底层使用 HashMap 来保存所有元素，因此 HashSet 的实现比较简单，
相关 HashSet 的操作，基本上都是直接调用底层 HashMap 的相关方法来完成。

## 基本属性

```
//　map集合，HashSet 存放元素的容器
private transient HashMap<E, Object> map;
//map　中的 value　值
private static final Object PRESENT = new Object();
```

## 构造方法

无参构造方法，完成map的创建:

```
    public HashSet() {
        this.map = new HashMap();
    }
```

指定集合转化为 HashSet, 完成map的创建：

```
    public HashSet(Collection<? extends E> var1) {
        this.map = new HashMap(Math.max((int)((float)var1.size() / 0.75F) + 1, 16));
        this.addAll(var1);
    }
```

指定初始化大小，和负载因子:
```
    public HashSet(int var1, float var2) {
        this.map = new HashMap(var1, var2);
    }
```

指定初始化大小:

```
    public HashSet(int var1) {
        this.map = new HashMap(var1);
    }
```

通过构造函数，可以发现　HashSet 的底层是采用 HashMap 实现。



## HashSet 添加元素

```
    public boolean add(E var1) {
        return this.map.put(var1, PRESENT) == null;
    }
```

add 方法是将存放的对象当做了　HashMap　的健，value　都是相同的　PRESENT。由于 HashMap 的 key 是不能重复的，所以每当有重复的值写入
到 HashSet 时，value 会被覆盖，但 key 不会受到影响，这样就保证了 HashSet 中只能存放不重复的元素。


## HashSet 删除元素

```
    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
```


## HashSet 查询

```
 public boolean contains(Object o) { 
    return map.containsKey(o); 
    } 
```

底层实际调用 HashMap 的 containsKey 判断是否包含指定 key。


对于 HashSet 中保存的对象，请注意正确重写其 equals 和 hashCode 方法，以保证放入的对象的唯一性。





















