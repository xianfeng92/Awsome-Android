# Dagger2

Dagger 使用注解来实现依赖注入, 它利用 APT(Annotation Process Tool) 在编译时生成辅助类，这些类继承特定父类或实现特定接口，程序在运行时 Dagger 加载这些辅助类，调用相应接口完成依赖生成和注入。Dagger 对于程序的性能影响非常小，因此更加适用于 Android 应用的开发。

## 依赖注入相关概念

### 依赖(Dependency)
如果在 Class A 中，有个属性是 Class B 的实例，则称 Class B 是 Class A 的依赖，我们可以将 Class A 称为宿主(Host)；Class B 称为依赖(Dependency)。一个 Host 可能是另外一个类的 Dependency。

### 宿主(Host)
如果 Class B 是 Class A 的 Dependency，则称 Class A 是 Class B 的宿主(Host)。

### 依赖注入
如果 Class B 是 Class A 的 Dependency，B 的赋值不是写死在了类或构造函数中，而是通过构造函数或其他函数的参数传入，这种赋值方式我们称之为依赖注入。

## Dagger注入规则

### 通过 @Inject去注解一个A类的构造函数时, Dagger 就会在需要获取 A 对象时，调用这个被标记的构造函数，从而生成一个 A 对象

如果A构造函数含有参数，Dagger 会在构造A对象的时候先去获取这些参数，所以我们要保证A构造函数的参数也提供了可被 Dagger 调用到的生成函数。
Dagger 可调用的对象生成方式有两种:一种是用 @Inject 修饰的构造函数,另外一种是用 @Provides 修饰的函数

### 在属性前添加的 @Inject 注解的目的是告诉 Dagger 哪些属性需要被注入

对构造函数进行注解是很好用的依赖对象生成方式，然而它并不适用于所有情况。例如:

1. 接口(Interface)是没有构造函数的，当然就不能对构造函数进行注解
2. 第三方库提供的类，我们无法修改源码，因此也不能注解它们的构造函数
3. 有些类需要提供统一的生成函数(一般会同时私有化构造函数)或需要动态选择初始化的配置，而不是使用一个单一的构造函数

### 使用 @Provides 注解来标记自定义的生成函数，从而被 Dagger 调用

和构造函数一样，@Provides 注解修饰的函数如果含有参数，它的所有参数也需要提供可被 Dagger 调用到的生成函数。需要注意的是，所有 @Provides 注解的生成函数都需要在Module中定义实现


## @Inject 和 @Provide 两种依赖生成方式区别

1.  @Inject 用于注入可实例化的类，@Provides 可用于注入所有类
2.  @Inject 可用于修饰属性和构造函数，可用于任何非 Module 类，@Provides 只可用于用于修饰非构造函数，并且该函数必须在某个Module内部
3. @Inject 修饰的函数只能是构造函数，@Provides 修饰的函数必须以 provide 开头

### 在相应函数添加 @Singleton 注解，依赖的对象就只会被初始化一次，之后的每次都会被直接注入相同的对象。

## 编译时检查

Dagger 会在编译时对代码进行检查，并在检查不通过的时候报编译错误，具体原因请看下面的详细原理介绍。检查内容主要有三点：


[Dagger 源码解析](http://a.codekk.com/detail/Android/%E6%89%94%E7%89%A9%E7%BA%BF/Dagger%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)


