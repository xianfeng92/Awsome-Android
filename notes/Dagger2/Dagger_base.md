# Dagger2

Dagger 使用注解来实现依赖注入, 它利用 APT(Annotation Process Tool) 在编译时生成辅助类，这些类继承特定父类或实现特定接口，程序在运行时 Dagger 加载这些辅助类，调用相应接口完成依赖生成和注入。
Dagger 对于程序的性能影响非常小，因此更加适用于 Android 应用的开发。

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

　　和构造函数一样，@Provides 注解修饰的函数如果含有参数，它的所有参数也需要提供可被 Dagger 调用到的生成函数。

### @Inject 和 @Provide 两种依赖生成方式区别

1. @Inject 用于注入可实例化的类，@Provides 可用于注入所有类
2. @Inject 可用于修饰属性和构造函数，可用于任何非 Module 类，@Provides 只可用于修饰非构造函数，并且该函数必须在某个Module内部
3. @Inject 修饰的函数只能是构造函数，@Provides 修饰的函数必须以 provide 开头

###　Lazy （延迟注入）

有时我们想注入的依赖在使用时再完成初始化，加快加载速度，就可以使用注入Lazy<T>。只有在调用 Lazy<T> 的 get() 方法时才会初始化依赖实例注入依赖。

	```
	public class Man {
	    @Inject
	    Lazy<Car> lazyCar;

	    public void goWork() {
		...
		lazyCar.get().go(); // lazyCar.get() 返回 Car 实例
		...
	    }
	}
	```

### 在相应函数添加 @Singleton 注解，依赖的对象就只会被初始化一次，之后的每次都会被直接注入相同的对象。
   
    Dagger 2 中 @Singleton 的 Component 不能依赖其他的 Component。

### Scope（作用域）

Scope 是用来确定注入的实例的生命周期。__如果没有使用 Scope 注解，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数生成的工厂时都会创建一个新的实例，而使用 Scope 后可以复用之前的依赖实例__。

@Scope是元注解，是用来标注自定义注解的，如下：

	```
	@Documented
	@Retention(RUNTIME)
	@Scope
	public @interface MyScope {}
	```

当 Component 与 Module、目标类（需要被注入依赖）使用 Scope 注解绑定时，意味着 Component 对象持有绑定的依赖实例的一个引用直到 Component 对象本身被回收。也就是作用域的原理，其实是让生成的依赖实例的生命周期与 Component 绑定，__Scope 注解并不能保证生命周期，要想保证依赖实例的生命周期，需要确保 Component 的生命周期__。

### Binding Instances

通过 Scope（作用域）可以让 Component 间接持有 Module 或 Inject 目标类构造函数提供的依赖实例。除此之外，Component 还可以在创建 Component 的时候绑定依赖实例，用以注入。这就是@BindsInstance注解的作用，只能在 Component.Builder 中使用。



### Qualifier（限定符）

Qualifier 限定符用来解决依赖迷失问题，可以依赖实例起个别名用来区分。

### Component 的组织关系

Component 管理着依赖实例，根据依赖实例之间的关系就能确定 Component 的关系。这些关系可以用 object graph描述，我称之为依赖关系图。在 Dagger 2 中 Component 
的组织关系分为两种:

* 依赖关系，一个 Component 依赖__其他 Compoent 公开的依赖实例__，用 Component 中的dependencies声明。
  __被依赖的 Component 需要把暴露的依赖实例用显式的接口声明__。

	```
	@ManScope
	@Component(modules = CarModule.class)
	public interface ManComponent {
	    void inject(Man man);

	    Car car();  //必须向外提供 car 依赖实例的接口，表明 Man 可以借 car 给别人
	}

	@FriendScope
	@Component(dependencies = ManComponent.class)
	public interface FriendComponent {
	    void inject(Friend friend);
	}
	```


因为 FriendComponent 和 ManComponent 是依赖关系，如果其中一个声明了作用域的话，另外一个也必须声明，ManComponent 的生命周期 >= FriendComponent 的。FriendComponent 的 Scope 不能是 @Singleton，
因为 Dagger 2 中 @Singleton 的 Component 不能依赖其他的 Component。


* 继承关系，一个 Component 继承（也可以叫扩展）某 Component 提供更多的依赖，SubComponent 就是继承关系的体现。 __继承关系中不用显式地提供暴露依赖实例的接口__

	```
	@ManScope
	@Component(modules = CarModule.class)
	public interface ManComponent {
	    void inject(Man man);   // 继承关系中不用显式地提供暴露依赖实例的接口
	}

	@SonScope
	@SubComponent(modules = BikeModule.class)
	public interface SonComponent {
	    void inject(Son son);

	    @Subcomponent.Builder
	    interface Builder { // SubComponent 必须显式地声明 Subcomponent.Builder，parentComponent 需要用该　Builder 来创建 SubComponent
		SonComponent build();
	    }
	}
	```

__那么如何确定一个 SubComponent 是继承哪个 parentComponent 的呢？__

如下所示，只需要在 parentComponent 依赖的 Module 中添加 subcomponents＝SubComponent.class，然后就可以在 parentComponent 中请求 SubComponent.Builder。

```
@Module(subcomponents = SonComponent.class)
public class CarModule {
    @Provides
    @ManScope
    static Car provideCar() {
        return new Car();
    }
}

@ManScope
@Component(modules = CarModule.class)
public interface ManComponent {
    void injectMan(Man man);

    SonComponent.Builder sonComponent();    // 在 parentComponent 中通过 Builder 来创建 Subcomponent。
}
```

此时　SonComponent　中就有　ManComponent　中的所有的依赖。

SubComponent 编译时不会生成 DaggerXXComponent，需要通过 parent Component 的获取 SubComponent.Builder 方法获取 SubComponent 实例。

	```
	ManComponent manComponent = DaggerManComponent.builder().build();
	SonComponent sonComponent = manComponent.sonComponent().build();
	sonComponent.inject(son);
	```

### 依赖关系 vs 继承关系

#### 相同点：

1. 两者都能复用其他 Component 的依赖

2. 依赖关系和继承关系的 Component 不能有相同的 Scope

#### 区别：

1. 依赖关系中被依赖的 Component 必须显式地提供公开依赖实例的接口，而 SubComponent 默认继承 parent Component 的依赖

2. 依赖关系会生成两个独立的 DaggerXXComponent 类，而 SubComponent 不会生成 独立的 DaggerXXComponent 类


__在 Android 开发中，Activity 是 App 运行中组件，Fragment 又是 Activity 一部分，这种组件化思想适合继承关系，所以在 Android 中一般使用 SubComponent。__

一般会以 AppComponent 作为 rootComponent，AppComponent 一般还会使用 @Singleton 作用域，而 ActivityComponent 为 SubComponent。

# 参考

[Dagger 2 完全解析（三），Component 的组织关系与 SubComponent](https://www.jianshu.com/p/2ac2f39cb25f)
[Dagger 源码解析](http://a.codekk.com/detail/Android/%E6%89%94%E7%89%A9%E7%BA%BF/Dagger%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
