# Dagger 中的设计模式分析
## Builder 模式
### DaggerFatherComponent 对象的构建

Dagger 中对 DaggerFatherComponent 对象的构建可以拆分成三部分：

####一 构建使用 @Inject 注解构造函数的对象

如：
```
this.provideCarProvider = FatherModule_ProvideCarFactory.create(builder.fatherModule);
```

####二 构建 FatherComponent 所对应的 FatherModule.class 中提供的依赖对象

1. 对于 FatherModule.class 中提供的所有依赖对象，Dagger 都会生成一个相应的工厂类，可以使用该该工厂类来获取依赖

如：

```
this.provideCarProvider = FatherModule_ProvideCarFactory.create(builder.fatherModule);
```

####三 构建存在依赖或者继承关系的依赖对象

1. FatherComponent 除了可以使用 FatherModule.class 中提供的依赖对象，还可以使用通过依赖或者继承其他Component的依赖。

   * 对于继承（SubComponent）
   Dagger 在构建 FatherComponent 时，同时会构建一个 sonComponentBuilderProvider 对象，该对象主要用来
   构建 sonComponent 对象。即在继承关系中，子 Component 是需要父 Component 来构建的。在构建 子 Component 过程中，父 Component 会将其
   所有依赖都传给 子 Component。


   * 对于依赖（Dependent）
    
    假设 SonComponent 依赖 FatherComponent，我们在构建 SonComponent 时需要传入一个 fatherComponent 对象，表明 SonComponent 中需要使用
    FatherComponent 的一些依赖。而在 FatherComponent 通过接口的形式暴露了 SonComponent 中可以使用的依赖。
   


