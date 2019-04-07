# Adapters

## 作用

       将一个类的接口转换为客户端所期望的接口, Adapter可以让原本不兼容的接口可以一起工作.

## 解释

     🌰现实生活中的栗子:
     当我们想将外置存储卡中的一些图片拷贝到笔记本上.为了传输这些图片, 我们需要某种与计算机端口兼容的适配器,以便将存储卡连接到计算机。在这种情况下,读卡器是一个适配器。
     另一个栗子是著名的电源适配器,三腿的插头不能连接到两脚插座上,它需要使用一个电源适配器, 使其与两脚插座兼容。翻译员的作用也可以看作是一个适配器,即把一个人说的话翻译
     给另外一个人听。

     简单的说,适配器模式将不兼容的对象包装在适配器中, 使其与另一个类兼容。

    Wikipedia 解释:
    
    在软件工程中, 适配器模式是一种软件设计模式, 允许将现有类的接口用作另一个接口,它通常用于在不修改源代码的情况下使现有类与其他类一起工作.

## 程序示例

     假设有一个船长只会划艇,不会驾驶渔船航行. 我们有 RowingBoat 和 FishingBoat 两个接口:
     
     ```
     // 代表划艇的能力
     public interface RowingBoat {
     void row();
     }
     
     // 代表驾驶渔船的能力
     public class FishingBoat {
     private static final Logger LOGGER = LoggerFactory.getLogger(FishingBoat.class);
     public void sail() {
     LOGGER.info("The fishing boat is sailing");
     }
     }
     ```
     
     船长实现了 RowingBoat 接口, 他可以划艇了:
     
    ```
     public class Captain implements RowingBoat {
     
     private RowingBoat rowingBoat;
     
     public Captain(RowingBoat rowingBoat) {
     this.rowingBoat = rowingBoat;
     }
     
     @Override
     public void row() {
     rowingBoat.row();
     }
     }
     ```
     假设海盗来了, 我们的船长需要逃跑,但只有渔船可用。现在我们需要创建一个适配器,让船长用他的划艇技能来操作渔船。
     
     ```
     // FishingBoatAdapter 实现了 RowingBoat 接口,即 FishingBoatAdapter 也代表一种划艇的能力
     public class FishingBoatAdapter implements RowingBoat {
     
     private static final Logger LOGGER = LoggerFactory.getLogger(FishingBoatAdapter.class);
     // FishingBoatAdapter 内部持有了 FishingBoat, 即可以通过 FishingBoatAdapter 的成员变量 FishingBoat 来操作渔船
     private FishingBoat boat;
     // 注意构建适配器 FishingBoatAdapter 时, 需要传入 FishingBoat
     public FishingBoatAdapter() {
     boat = new FishingBoat();
     }
     // 划艇技能操作渔船
     @Override
     public void row() {
     boat.sail();
     }
     }
     ```
     此时船长就可以用渔船来躲避海盗了:
     
     ```
     Captain captain = new Captain(new FishingBoatAdapter());
     captain.row();//用划艇技能操作渔船,其实内部还是调用 FishingBoat 的 sail 方法
     ```
     上面这个例子中,我们在没有修改 Captain 类的源码下,使得 Captain 具有了 sail 的能力.
     
## 适用性
     
     在以下情况下可以考虑使用Adapter:
     
     1. 我们想使用一个现有的类,当是其接口与我们需要的接口不匹配
        
        Captain 需要 RowingBoat 接口, 与 FishingBoat 类不匹配
        
     2. 当我们想创建一个可重用的类, 它与不相关或不可预见的类（即不一定具有兼容接口的类）之间协作
    
     3. 当需要使用几个现有的子类, 但是通过子类化每个子类来调整它们的接口是不切实际的。Adapter 可以调整其父类的接口
    
     4. 大多数使用第三方库的应用程序会用 Adapter 作为应用程序和第三方库之间的中间层, 以将应用程序与库分离。如果要使用新库,则只需要新库的适配器,而不必更改应用程序的代码

## 结论

    适配器从结构上分为: 类适配器和对象适配器。其中类适配器使用继承关系来对类进行适配, 对象适配器使用对象引用来进行适配。
    
    类适配器:
    
    * 通过构建一个具体的 adaptee 类使 adaptee 适应目标接口(target)。因此, 当我们想要调整一个类及其所有子类时, 类适配器将无法工作
    
    * 让 Adapter 重写 Adaptee 的一些行为, 因为 Adapter 是 Adaptee 的子类
    
    * 只引入一个对象,不需要额外的指针间接指向即可到达 adaptee(多态特性)

    对象适配器:
    
    * 让一个 Adapter 与许多 Adaptee 一起工作, 也就是说, Adaptee 本身及其所有子类（如果有的话)。Adapter 还可以一次为所有 Adaptee 添加功能
    
    * 很难重写 Adaptee 的行为, 它将需要子类化 adaptee, 并使适配器引用子类。

## 使用适配的例子

     * [java.util.Arrays#asList()](https://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#asList%28T...%29)
     
     * [java.util.Collections#list()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#list-java.util.Enumeration-)
     
     * [java.util.Collections#enumeration()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#enumeration-java.util.Collection-)
    
