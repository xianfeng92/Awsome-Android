## 任务和返回栈

一个应用程序当中通常都会包含很多个Activity，每个Activity都应该设计成为一个具有特定的功能，并且可以让用户进行操作的组件。另外，Activity之间还应该是可以相互启动的。
比如，一个邮件应用中可能会包含一个用于展示邮件列表的Activity，而当用户点击了其中某一封邮件的时候，就会打开另外一个Activity来显示该封邮件的具体内容。

除此之外，一个Activity甚至还可以去启动其它应用程序当中的Activity。打个比方，如果你的应用希望去发送一封邮件，你就可以定义一个具有"send"动作的Intent，并且传入一些数据，
如对方邮箱地址、邮件内容等。这样，如果另外一个应用程序中的某个Activity声明自己是可以响应这种Intent的，那么这个Activity就会被打开。在当前场景下，这个Intent是为了要发送邮件的，
所以说邮件应用程序当中的编写邮件Activity就应该被打开。当邮件发送出去之后，仍然还是会回到你的应用程序当中，这让用户看起来好像刚才那个编写邮件的Activity就是你的应用程序当中的一部分。
所以说，__即使有很多个Activity分别都是来自于不同应用程序的，Android系统仍然可以将它们无缝地结合到一起，之所以能实现这一点，就是因为这些Activity都是存在于一个相同的任务(Task)当中的__。

任务是一个Activity的集合，它使用栈的方式来管理其中的Activity，这个栈又被称为返回栈(back stack)，栈中Activity的顺序就是按照它们被打开的顺序依次存放的。

手机的Home界面是大多数任务开始的地方，当用户在Home界面上点击了一个应用的图标时，这个应用的任务就会被转移到前台。如果这个应用目前并没有任何一个任务的话(说明这个应用最近没有被启动过)，
系统就会去创建一个新的任务，并且将该应用的主Activity放入到返回栈当中。

当一个Activity启动了另外一个Activity的时候，新的Activity就会被放置到返回栈的栈顶并将获得焦点。前一个Activity仍然保留在返回栈当中，但会处于停止状态。当用户按下Back键的时候，
栈中最顶端的Activity会被移除掉，然后前一个Activity则会得重新回到最顶端的位置。返回栈中的Activity的顺序永远都不会发生改变，我们只能向栈顶添加Activity，或者将栈顶的Activity移除掉。
因此，返回栈是一个典型的后进先出(last in, first out)的数据结构。下图通过时间线的方式非常清晰地向我们展示了多个Activity在返回栈当中的状态变化：

![backStack_1](https://github.com/xianfeng92/android-code-read/blob/master/images/backStack_1.png)

如果用户一直地按Back键，这样返回栈中的Activity会一个个地被移除，直到最终返回到主屏幕。当返回栈中所有的Activity都被移除掉的时候，对应的任务也就不存在了。

任务除了可以被转移到前台之外，当然也是可以被转移到后台的。当用户开启了一个新的任务，或者点击Home键回到主屏幕的时候，之前任务就会被转移到后台了。当任务处于后台状态的时候，
返回栈中所有的Activity都会进入停止状态，但这些Activity在栈中的顺序都会原封不动地保留着，如下图所示：

![foreandBackGroundTask.png](https://github.com/xianfeng92/android-code-read/blob/master/images/foreandBackGroundTask.png)


这个时候，用户还可以将任意后台的任务切换到前台，这样用户应该就会看到之前离开这个任务时处于最顶端的那个Activity。


举个例子来说，当前任务A的栈中有三个Activity，现在用户按下Home键，然后点击桌面上的图标启动了另外一个应用程序。当系统回到桌面的时候，其实任务A就已经进入后台了，
然后当另外一个应用程序启动的时候，系统会为这个程序开启一个新的任务(任务B)。当用户使用完这个程序之后，再次按下Home键回到桌面，这个时候任务B也进入了后台。
然后用户又重新打开了第一次使用的程序，这个时候任务A又会回到前台，A任务栈中的三个Activity仍然会保留着刚才的顺序，最顶端的Activity将重新变为运行状态。
之后用户仍然可以通过Home键或者多任务键来切换回任务B，或者启动更多的任务，这就是Android中多任务切换的例子。


由于返回栈中的Activity的顺序永远都不会发生改变，所以如果你的应用程序中允许有多个入口都可以启动同一个Activity，那么每次启动的时候就都会创建该Activity的一个新的实例，
而不是将下面的Activity移动到栈顶。这样的话就容易导致一个问题的产生，即同一个Activity有可能会被实例化很多次，如下图所示：

![back_stack](https://github.com/xianfeng92/android-code-read/blob/master/images/back_stack.png)

但是呢，如果你不希望同一个Activity可以被多次实例化，那当然也是可以的，马上我们就将开始讨论如果实现这一功能，现在我们先把默认的任务和Activity的行为简单概括一下：


1. 当Activity A启动Activity B时，Activity A进入停止状态，但系统仍然会将它的所有相关信息保留，比如滚动的位置，还有文本框输入的内容等。如果用户在Activity B中按下Back键，
   那么Activity A将会重新回到运行状态。

2. 当用户通过Home键离开一个任务时，该任务会进入后台，并且返回栈中所有的Activity都会进入停止状态。系统会将这些Activity的状态进行保留，这样当用户下一次重新打开这个应用程序时，
   就可以将后台任务直接提取到前台，并将之前最顶端的Activity进行恢复。

3. 当用户按下Back键时，当前最顶端的Activity会被从返回栈中移除掉，移除掉的Activity将被销毁，然后前面一个Activity将处于栈顶位置并进入活动状态。
    当一个Activity被销毁了之后，系统不会再为它保留任何的状态信息。每个Activity都可以被实例化很多次，即使是在不同的任务当中。



## 管理任务

Android系统管理任务和返回栈的方式，正如上面所描述的一样，就是把所有启动的Activity都放入到一个相同的任务当中，通过一个“后进先出”的栈来进行管理的。这种方式在绝大多数情况下
都是没问题的，开发者也无须去关心任务中的Activity到底是怎么样存放在返回栈当中的。但是呢，__如果你想打破这种默认的行为__，比如说当启动一个新的Activity时，你希望它可以存在于一个独立
的任务当中，而不是现有的任务当中。或者说，当启动一个Activity时，如果这个Activity已经存在于返回栈中了，你希望能把这个Activity直接移动到栈顶，而不是再创建一个它的实例。再或者，
你希望可以将返回栈中除了最底层的那个Activity之外的其它所有Activity全部清除掉。这些功能甚至更多功能，都是可以通过在manifest文件中设置<activity>元素的属性，或者是在启动Activity时
配置Intent的flag来实现的。

在<activity>元素中，有以下几个属性是可以使用的：

* taskAffinity

* launchMode

* allowTaskReparenting

* clearTaskOnLaunch

* alwaysRetainTaskState

* finishOnTaskLaunch

而在Intent当中，有以下几个flag是比较常用的：

* FLAG_ACTIVITY_NEW_TASK

* FLAG_ACTIVITY_CLEAR_TOP

* FLAG_ACTIVITY_SINGLE_TOP


下面我们就将开始讨论，如何通过manifest参数，以及Intent flag来改变Activity在任务中的默认行为。

## 定义启动模式

__启动模式允许你去定义如何将一个Activity的实例和当前的任务进行关联__，你可以通过以下两种不同的方式来定义启动模式：

1. 使用manifest文件

当你在manifest文件中声明一个Activity的时候，你可以指定这个Activity在启动的时候该如何与任务进行关联。

2. 使用Intent flag

当你调用startActivity()方法时，你可以在Intent中加入一个flag，从而指定新启动的Activity该如何与当前任务进行关联。


也就是说，如果Activity A启动了Activity B，Activity B可以定义自己该如何与当前任务进行关联，而Activity A也可以要求Activity B该如何与当前任务进行关联。
如果Activity B在manifest中已经定义了该如何与任务进行关联，而Activity A同时也在Intent中要求了Activity B该怎么样与当前任务进行关联，那么__此时Intent中的定义将覆盖manifest中的定义__。

需要注意的是，有些启动模式在manifest中可以指定，但在Intent中是指定不了的。同样，也有些启动模式在Intent中可以指定，但在manifest中是指定不了的，下面我们就来具体讨论一下。


### 使用manifest文件

当在manifest文件中定义Activity的时候，你可以通过<activity>元素的launchMode属性来指定这个Activity应该如何与任务进行关联。launchMode属性一共有以下四种可选参数：


#### "standard"(默认启动模式)

standard是默认的启动模式，即如果不指定launchMode属性，则自动就会使用这种启动模式。这种启动模式表示每次启动该Activity时系统都会为创建一个新的实例，并且总会把它放入到当前的任务当中。
声明成这种启动模式的Activity可以被实例化多次，一个任务当中也可以包含多个这种Activity的实例。


#### "singleTop"

这种启动模式表示，如果要启动的这个Activity在当前任务中已经存在了，并且还处于栈顶的位置，那么系统就不会再去创建一个该Activity的实例，而是调用栈顶Activity的onNewIntent()方法。
声明成这种启动模式的Activity也可以被实例化多次，一个任务当中也可以包含多个这种Activity的实例。


#### "singleTask"

这种启动模式表示，系统会创建一个新的任务，并将启动的Activity放入这个新任务的栈底位置。但是，如果现有任务当中已经存在一个该Activity的实例了，那么系统就不会再创建一次它的实例，
而是会直接调用它的onNewIntent()方法。声明成这种启动模式的Activity，在同一个任务当中只会存在一个实例。注意这里我们所说的启动Activity，都指的是启动其它应用程序中的Activity，
因为__"singleTask"模式在默认情况下只有启动其它程序的Activity才会创建一个新的任务，启动自己程序中的Activity还是会使用相同的任务__，具体原因会在下面 处理 affinity 部分进行解释。



#### "singleInstance"

这种启动模式和"singleTask"有点相似，只不过__系统不会向声明成"singleInstance"的Activity所在的任务当中再添加其它Activity__。也就是说，这种Activity所在的任务中始终只会有一个Activity，
通过这个Activity再打开的其它Activity也会被放入到别的任务当中。

再举一个例子，Android系统内置的浏览器程序声明自己浏览网页的Activity始终应该在一个独立的任务当中打开，也就是通过在<activity>元素中设置"singleTask"启动模式来实现的。这意味着，
当你的程序准备去打开Android内置浏览器的时候，新打开的Activity并不会放入到你当前的任务中，而是会启动一个新的任务。而如果浏览器程序在后台已经存在一个任务了，则会把这个任务切换到前台。


其实不管是Activity在一个新任务当中启动，还是在当前任务中启动，返回键永远都会把我们带回到之前的一个Activity中的。但是有一种情况是比较特殊的，就是如果Activity指定了启动模式是"singleTask"，
并且启动的是另外一个应用程序中的Activity，这个时候当发现该Activity正好处于一个后台任务当中的话，就会直接将这整个后台任务一起切换到前台。此时按下返回键会优先将目前最前台的任务(刚刚从后台切换到最前台)
进行回退，下图比较形象地展示了这种情况：

![Activity_backStack.png](https://github.com/xianfeng92/android-code-read/blob/master/images/Activity_backStack.png)


### 使用Intent flags

除了使用manifest文件之外，你也可以在调用startActivity()方法的时候，为Intent加入一个flag来改变Activity与任务的关联方式，下面我们来一一讲解一下每种flag的作用：

#### FLAG_ACTIVITY_NEW_TASK

设置了这个flag，新启动Activity就会被放置到一个新的任务当中(与"singleTask"有点类似，但不完全一样)，当然这里讨论的仍然还是启动其它程序中的Activity。这个flag的作用通常是模拟一种Launcher的行为，
即列出一推可以启动的东西，但启动的每一个Activity都是运行在自己独立的任务当中的。


#### FLAG_ACTIVITY_SINGLE_TOP

设置了这个flag，如果要启动的Activity在当前任务中已经存在了，并且还处于栈顶的位置，那么就不会再次创建这个Activity的实例，而是直接调用它的onNewIntent()方法。这种flag和在launchMode中
指定"singleTop"模式所实现的效果是一样的。


#### FLAG_ACTIVITY_CLEAR_TOP

设置了这个flag，如果要启动的Activity在当前任务中已经存在了，就不会再次创建这个Activity的实例，而是会把这个Activity之上的所有Activity全部关闭掉。比如说，一个任务当中有A、B、C、D四个Activity，
然后D调用了startActivity()方法来启动B，并将flag指定成FLAG_ACTIVITY_CLEAR_TOP，那么此时C和D就会被关闭掉，现在返回栈中就只剩下A和B了。

那么此时Activity B会接收到这个启动它的Intent，你可以决定是让Activity B调用onNewIntent()方法(不会创建新的实例)，还是将Activity B销毁掉并重新创建实例。如果Activity B没有在manifest中指定任何启动模式
(也就是"standard"模式)，并且Intent中也没有加入一个FLAG_ACTIVITY_SINGLE_TOP flag，那么此时Activity B就会销毁掉，然后重新创建实例。而如果Activity B在manifest中指定了任何一种启动模式，或者是在Intent
中加入了一个FLAG_ACTIVITY_SINGLE_TOP flag，那么就会调用Activity B的onNewIntent()方法。


FLAG_ACTIVITY_CLEAR_TOP和FLAG_ACTIVITY_NEW_TASK结合在一起使用也会有比较好的效果，比如可以将一个后台运行的任务切换到前台，并把目标Activity之上的其它Activity全部关闭掉。
这个功能在某些情况下非常有用，比如说从通知栏启动Activity的时候。

## 处理affinity

__affinity 可以用于指定一个Activity更加愿意依附于哪一个任务__，在__默认情况下，同一个应用程序中的所有Activity都具有相同的affinity，所以，这些Activity都更加倾向于运行在相同的任务当中__。
当然了，你也可以去改变每个Activity的affinity值，通过<activity>元素的taskAffinity属性就可以实现了。

taskAffinity属性接收一个字符串参数，你可以指定成任意的值(经我测试字符串中至少要包含一个.)，但必须不能和应用程序的包名相同，因为__系统会使用包名来作为默认的affinity值__。


affinity主要有以下两种应用场景：

1. 当调用startActivity()方法来启动一个Activity时，默认是将它放入到当前的任务当中。但是，如果在Intent中加入了一个FLAG_ACTIVITY_NEW_TASK flag的话(或者该Activity在manifest文件中
声明的启动模式是"singleTask")，系统就会尝试为这个Activity单独创建一个任务。但是规则并不是只有这么简单，__系统会去检测要启动的这个Activity的affinity和当前任务的affinity是否相同__，
如果相同的话就会把它放入到现有任务当中，如果不同则会去创建一个新的任务。而同一个程序中所有Activity的affinity默认都是相同的，这也是前面为什么说，同一个应用程序中即使声明成"singleTask"，
也不会为这个Activity再去创建一个新的任务了。

2. __当把Activity的allowTaskReparenting属性设置成true时，Activity就拥有了一个转移所在任务的能力__。具体点来说，就是一个Activity现在是处于某个任务当中的，但是它与另外一个任务具有相同的affinity值，
那么当另外这个任务切换到前台的时候，该Activity就可以转移到现在的这个任务当中。

那还是举一个形象点的例子吧，比如有一个天气预报程序，它有一个Activity是专门用于显示天气信息的，这个Activity和该天气预报程序的所有其它Activity具体相同的affinity值，并且还将allowTaskReparenting属性设置成true了。
这个时候，你自己的应用程序通过Intent去启动了这个用于显示天气信息的Activity，那么此时这个Activity应该是和你的应用程序是在同一个任务当中的。但是当把天气预报程序切换到前台的时候，这个Activity又会被转移到天气预报程序的任务当中，并显示出来，因为它们拥有相同的affinity值，并且将allowTaskReparenting属性设置成了true。


## 清空返回栈

如何用户将任务切换到后台之后过了很长一段时间，系统会将这个任务中除了最底层的那个Activity之外的其它所有Activity全部清除掉。当用户重新回到这个任务的时候，最底层的那个Activity将得到恢复。
这个是系统默认的行为，因为既然过了这么长的一段时间，用户很有可能早就忘记了当时正在做什么，那么重新回到这个任务的时候，基本上应该是要去做点新的事情了。


当然，既然说是默认的行为，那就说明我们肯定是有办法来改变的，在<activity>元素中设置以下几种属性就可以改变系统这一默认行为：

#### alwaysRetainTaskState

如果将最底层的那个Activity的这个属性设置为true，那么上面所描述的默认行为就将不会发生，任务中所有的Activity即使过了很长一段时间之后仍然会被继续保留。



#### clearTaskOnLaunch

如果将最底层的那个Activity的这个属性设置为true，那么只要用户离开了当前任务，再次返回的时候就会将最底层Activity之上的所有其它Activity全部清除掉。简单来讲，就是一种和alwaysRetainTaskState完全相反的
工作模式，它保证每次返回任务的时候都会是一种初始化状态，即使用户仅仅离开了很短的一段时间。



#### finishOnTaskLaunch

这个属性和clearTaskOnLaunch是比较类似的，不过它不是作用于整个任务上的，而是作用于单个Activity上。如果某个Activity将这个属性设置成true，那么用户一旦离开了当前任务，再次返回时这个Activity就会被清除掉。

 
[Android任务和返回栈完全解析，细数那些你所不知道的细节](https://blog.csdn.net/guolin_blog/article/details/41087993)
