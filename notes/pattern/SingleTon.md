
# 单例模式

1. 有些实例需要在整个系统环境中以单例的形式存在，或者多次创建该实例需要耗费很多内存资源。

## 几种常见的单例模式

### 懒汉式

```
public class SingleTon {

    private static SingleTon instance;

    private SingleTon(){
    }

    // 懒汉式，使用时才创建
    public static SingleTon getInstance(){
        if(instance == null){
            instance = new SingleTon();
        }
        return instance;
    }
}
```

### 饿汉式

```
public class SingleTon {
    // 饿汉式
    private static SingleTon instance = new SingleTon();

    private SingleTon(){
    }

    public static SingleTon getInstance(){
        return instance;
    }
}
```


### 双重线程锁

```
public class SingleTon {

    // volatile　保证内存的可见性，并不保证操作的原子性
    private static volatile SingleTon instance;

    private SingleTon(){
    }

    public static SingleTon getInstance(){
        if(null == instance){ // code 1
            synchronized (SingleTon.class){// code 2
                if(null == instance){ // code 3
                    instance = new SingleTon();// code 4
                }
            }
        }
        return instance;
    }
}
```

1. 在多线程中，两个线程可能同时执行到 code 1, synchronize　保证只有一个线程能进入 code 2 下面的代码,此时一个线程 A 进入,一个线程B在外等待。
   当线程A完成 code 3 和　code 4 之后,线程B进入 synchronized code 2 下面的方法, 线程 B 在 code 3 的时候判断 instance 已经实例化　,从而保
   证了多线程下单例模式的线程安全。

2. synchronized 能保证每次只有一个线程访问代码块来创建 instance，volatile 保证了 instance 内存可见性。

### 静态内部类

```
public class SingleTon {

    private SingleTon(){
    }
    // 1. 调用该方法时，才会访问 LazyHolder.INSTANCE 这个静态类的静态变量
    public static SingleTon getInstance(){
        return LazyHoler.INSTANCE;
    }

    private static class LazyHoler{
   // 2. 访问 LazyHolder.INSTANCE 才会触发 Singleton 的初始化
        static final SingleTon INSTANCE = new SingleTon();
    }
}
```

1. 类初始化是线程安全的，并且只会执行一次。因此在多线程环境下，依然能保证只有一个 Singleton 实例。

2. 类加载规范的：访问静态字段时，才会初始化静态字段所在的类

### map 实现单例

```
public class SingleTon {
     private static Map<String,Object> objectMap = new HashMap<>();

     private SingleTon(){}

     public static void putObject(String key,String instance){
         if(!objectMap.containsKey(key)){
             objectMap.put(key,instance);
         }
     }

     public static Object getObject(String key){
         return objectMap.get(key);
     }
}
```






















