通过 LayoutInflater 可以获取到布局文件中的 View. 任何一个 View 都不可能凭空突然出现在屏幕上, 它们都是要经过非常科学的绘制流程后才能显示出来的。
每一个视图的绘制过程都必须经历三个最主要的阶段, 即 onMeasure()、onLayout()和onDraw()。

## 关于Measure几个小结论:

1. View 的 measureSpec 由其父容器的 measureSpec(mode和size) 及自身的 LayoutParams(具体值、wrapContent、matchParent) 共同决定. 对于      ViewRoot 其测量模式为 Exactly,大小为 Window 大小.

2. 对于所有的控件,如果在布局中指定了具体数值的高和宽时,系统就会给它指定数值的高和宽的空间.至于屏幕上有没有那么多空间给这些控件那是另外一回事,其实如果    在 LinearLayout中将Button A的布局宽度设置为屏幕宽度的1/2,将Button B的布局宽度设置为整个屏幕的宽.假设在系统measure,先测量的Button A,那么运    行时会发现Button B的宽度只会显示出其宽度的1/2.

3. 当一个控件的布局宽度设置为 match_parant 时,此时传给它的 measureSize 就是其父控件可用的布局宽度值,它的 measureMode为 Exactly.那么该控件实    际显示的宽度还要看其自身 OnMeasure 中在 mode 为 Exactly 时的实现.

4. 当一个控件的布局宽度设置为 wrap_content 时,如果其父View为Exactly,那么其mode为At_Most,size为父View可用的布局宽度值.如果其父View的mode为    At_most,那么传给子View的mode为At_most,size为父View可用的布局宽度值.

5. 对View进行测量的目的是让View的父控件知道View想要多大的尺寸.当我们的子View在布局文件中,设置具体的布局大小时,其实就是在直接告诉父View,我想要这么    大的布局空间.当我们的子View在布局文件中为match_parent时,其实就是在告诉父View,你有多大布局空间就给我多大的布局空间,贪婪呀~~~~ .当我们的子View    在布局文件中为wrap_content时,其实是在告诉父View,我不贪,我只要够我用的布局空间就可以了.此时父View传给子View的size其实还是父view的可用空间.

## View的MeasureSpec的创建规则

   ![](https://github.com/xianfeng92/android-code-read/blob/master/images/view_measureSpec.png)

   父 View 传给子 View 的 mode = 父View + 子view的布局参数的设定
   
   我们在布局文件中设置一个View的布局宽高时,有三种选择:具体数值(dp),match_parent,wrap_content,而对于ViewRoot其mode为Exactly,所有一般
   我们只需要考虑mode为Exactly和At_most两种情况.当布局为一个很大的View树时,从根View开始,父View传给子View的一般都是Exactly或At_most.
  
   我们在自定义View的时候,重点需要考虑,当该 View 在布局文件中是 match_parent 或 wrap_content 时,我们希望它以何种大小显示(这是我们要根据我        们的需要来决定的)
   
## onMeasure

__对 View 进行测量的目的是让 View 的父控件知道 View 想要多大的尺寸__。

整个应用测量的起点是 ViewRootImpl 类, 从它开始依次对子 View 进行测量. 如果子View是一个 ViewGroup, 那么又会遍历该 ViewGroup 的子 View 依次进行测量。 __测量会从 View 树的根结点，纵向递归进行，从而实现自上而下对 View 树进行测量, 直至完成对叶子节点View的测量__。

那么到底如何对一个View进行测量呢？

Android 通过调用 View#measure()方法对View进行测量, 让该 View 的父控件知道该 View 想要多大的尺寸空间。

具体来说，View 的父控件 ViewGroup 会调用 View 的 measure 方法, ViewGroup 会将一些宽度和高度的限制条件传递给 View 的 measure 方法。

在View的measure方法会首先从成员变量中读取以前缓存过的测量结果，如果能找到该缓存值，那么就基本完事了，如果没有找到缓存值，那么measure方法会执行onMeasure回调方法, measure 方法会将上述的宽度和高度的限制条件依次传递给onMeasure方法。onMeasure方法会完成具体的测量工作, 并将测量的结果通过调用 View 的 setMeasuredDimension 方法保存到 View 的成员变量 mMeasuredWidth 和 mMeasuredHeight 中。

测量完成之后，View 的父控件就可以通过调用 getMeasuredWidth、getMeasuredState、getMeasuredWidthAndState 这三个方法获取View的测量结果。

### 理解 MeasureSpec

从名字上看，MeasureSpec 看起来像“测量规格”或者“测量说明书”，不管怎么翻译，他看起来就好像是或多或少的决定了View的测量过程，通过源码可以发现，MeasureSpec的确参与了View的测量过程。

__在测量过程中, 系统会将 View 的 LayoutParams 根据父容器所施加的规则转换成对应的 MeasureSpec, 然后再根据这个 measureSpec 来测量出View的宽高__。

#### MeasureSpec

MeasureSpec代表一个32位int值，高两位代表 SpecMode，低30位代表SpecSize，SpecMode是指测量模式，而SpecSize是指在某个测量模式下的规格大小。

SpecMode 一共有三种类型，如下所示：

1. EXACTLY

   父容器已经检测出View所需要的精度大小, 这个时候View的最终大小就是 SpecSize 所指定的值, 它对应于 LayoutParams 中的 match_parent,和具体的数值这两种模式。

2. AT_MOST

   父容器指定了一个可用大小, 即SpecSize. view的大小不能大于这个值, 具体是什么值要看不同view的具体实现, 它对应于 LayoutParams 中 wrap_content。

3. UNSPECIFIED
   
   父容器不对View有任何的限制, 这种情况一般用于系统内部表示一种测量的状态.

### MeasureSpec 和 LayoutParams 的对应关系

在view测量的时候, 系统会将 layoutparams 在父容器的约束下转换成对应的 MeasureSpec, 然后再根据这个 MeasureSpec 来确定view测量后的宽高. 需要注意的是, MeasureSpec 不是唯一由 layoutparams 决定的，layoutparams 需要和父容器一起决定 view 的 MeasureSpec 从而进一步决定view的宽高。另外，对于顶级View（DecorView）和普通的View来说, MeasureSpec 的转换过程有所不同。

__对于 DecorView，其 MeasureSpec 由窗口尺寸和其自身的 LayoutParams 共同决定__;

对于普通View，其MeasureSpec由父容器的 MeasureSpec 和自身的 LayoutParams 来共通决定。

__MeasureSpec 一旦确定后，onMeasure 中就可以确定 View 的测量宽和高了__。

## DecorView的Measure

对于 DecorView 来说，在 ViewRootImpl 中的 measureHierarchy 方法中有这么一段代码。它展示了DecorViwew的MeasureSpec创建过程，
其中 desiredWindowWidth 和 desiredWindowHeight 是屏幕的尺寸。

```
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
```

可以看到, 这里调用了getRootMeasureSpec()方法去获取 widthMeasureSpec 和 heightMeasureSpec 的值，注意方法中传入的参数，其中lp.width和lp.height在创建ViewGroup实例的时候就被赋值了, 它们都等于MATCH_PARENT。

```
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

通过上述代码，DecorView 的 MesureSpec 的产生过程就已经很明确了，具体来说遵守如下规则，根据它的LayoutParams的宽高的参数来划分：

* LayoutParams.MATCH_PARENT: 精确模式（EXACTLY），大小就是窗口大小

* LayoutParams.WRAP_CONTENT: 最大模式，大小不固定，但是不能超过窗口的大小

* 固定大小（比如100dp）：精确模式，大小为LayoutParams中指定的大小

## 普通View

对应普通View, 这里是指我们布局中的View，View的onMeasure过程是由 ViewGroup 传递而来. 

先看一下 ViewGroup#measureChildWidthMargins 方法:

```
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        // 获取子View的 MarginLayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

上述的方法会对子元素进行 measure, 在调用子元素的measure方法之前会通过 getChildMeasureSpec 方法得到子元素的 MesureSpec，

__从代码上看，很显然，子元素的 MesureSpec 的创建和父容器的 MesureSpec 和子元素的LayoutParams有关，此外，还和view的margin有关__。

具体可以看下ViewGroup的getChildMeasureSpec方法。

```
 public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```


上述方法不难理解，他的主要作用是根据父容器的 MeasureSpec 同时结合 view 本身来 layoutparams 来确定子元素的MesureSpec。参数中的 pading 是指父容器中已占有的控件大小，因此子元素可以用的大小为父容器的尺寸减去 pading，具体代码：

```
int specSize = MesureSpec.getSize(spec);
int size = Math.max(0,specSize - pading);

```

 ![](https://github.com/xianfeng92/android-code-read/blob/master/images/view_measureSpec.png)


针对这张表，这里再做一下说明。前面已经提到，对于普通View，其 MeasureSpec 由父容器的MeasureSpec和自身的LayoutParams来共同决定，那么针对不同的父容器和Viev本身不同的LayoutParams,View就可以有多种MeasureSpec。
这里简单说一下：

1. 当View采用固定宽/高的时候，不管父容器的 MeasureSpec 是什么，View 的 MeasureSpec 都是精确模式，那么View也是精准模式并且其大小是父容器的剩余空间
   如果父容器是最大模式，那么View也是最大模式并且其大小不会超过父容器的剩余空间。

2. 当View的宽/高是wrap_content时，不管父容器的模式是精准还是最大化，View的模式总是最大化,并且大小不能超过父容器的剩余空间

3. 对于 UNSPECIFIED 模式，那是因为这个模式主要用于系统内部多次Measure的情形，一般来说，我们不需要关注此模式。

通过这张表可以看出，只要提供父容器的MeasureSpec和子元素的LayoutParams，就可以快速地确定出子元素的MeasureSpec了，有了 MeasureSpec就可以进一步确定出子元亲测量后的大小了。需要说明的是，表中并非是什么经验总结，它只是getchildMeasureSpec 这个方法以表格的方式呈现出来而已。

## view#measure 过程

当 View 的父控件 ViewGroup 对View进行测量时，会调用View的measure方法，ViewGroup会传入 widthMeasureSpec 和 heightMeasureSpec。

```
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
         //首先判断当前View的layoutMode是不是特例LAYOUT_MODE_OPTICAL_BOUNDS
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            ////LAYOUT_MODE_OPTICAL_BOUNDS是特例情况，比较少见
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // 根据widthMeasureSpec和heightMeasureSpec计算key值，我们在下面用key值作为键，缓存我们测量的结果
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;

        // mMeasureCache是LongSparseLongArray类型的成员变量，
        // 其缓存着View在不同widthMeasureSpec、heightMeasureSpec下测量过的结果
        // 如果mMeasureCache为空，我们就新new一个对象赋值给mMeasureCache
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;


        // mOldWidthMeasureSpec和mOldHeightMeasureSpec分别表示上次对View进行测量时的widthMeasureSpec和heightMeasureSpec
        // 执行View的measure方法时，View总是先检查一下是不是真的有必要费很大力气去做真正的测量工作
        // mPrivateFlags是一个Int类型的值，其记录了View的各种状态位,如果(mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT
        // 那么表示当前View需要强制进行layout（比如执行了View的forceLayout方法），所以这种情况下要尝试进行测量
        // 如果新传入的widthMeasureSpec/heightMeasureSpec与上次测量时的mOldWidthMeasureSpec/mOldHeightMeasureSpec不等
        // 那么也就是说该View的父ViewGroup对该View的尺寸的限制情况有变化，这种情况下要尝试进行测量
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // 通过按位操作，重置View的状态mPrivateFlags，将其标记为未测量状态
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

           //对阿拉伯语、希伯来语等从右到左书写、布局的语言进行特殊处理
            resolveRtlPropertiesIfNeeded();

            // 在View真正进行测量之前，View还想进一步确认能不能从已有的缓存mMeasureCache中读取缓存过的测量结果
            // 如果是强制layout导致的测量，那么将cacheIndex设置为-1，即不从缓存中读取测量结果
            // 如果不是强制layout导致的测量，那么我们就用上面根据measureSpec计算出来的key值作为缓存索引cacheIndex。
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                 // 如果运行到此处，表示我们没有从缓存中找到测量过的尺寸或者是sIgnoreMeasureCache为true导致我们要忽略缓存结果
                // 此处调用onMeasure方法，并把尺寸限制条件 widthMeasureSpec 和 heightMeasureSpec 传入进去
                // onMeasure方法中将会进行实际的测量工作，并把测量的结果保存到成员变量中
                onMeasure(widthMeasureSpec, heightMeasureSpec);

                // onMeasure执行完后，通过位操作，重置View的状态mPrivateFlags，将其标记为在layout之前不必再进行测量的状态
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                 //如果运行到此处，那么表示当前的条件允许View从缓存成员变量mMeasureCache中读取测量过的结果
                //用上面得到的cacheIndex从缓存mMeasureCache中取出值，不必在调用onMeasure方法进行测量了
                long value = mMeasureCache.valueAt(cacheIndex);

                // 一旦我们从缓存中读到值，我们就可以调用setMeasuredDimensionRaw方法将当前测量的结果存储到成员变量中
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

               // 如果我们自定义的View重写了onMeasure方法，但是没有调用setMeasuredDimension()方法，
               // 那么此处就会抛出异常，提醒开发者在onMeasure方法中调用setMeasuredDimension()方法
              // Android是如何知道我们有没有在onMeasure方法中调用setMeasuredDimension()方法的呢？
              // 方法很简单，还是通过解析状态位mPrivateFlags。
             // setMeasuredDimension()方法中会将mPrivateFlags设置为PFLAG_MEASURED_DIMENSION_SET状态，即已测量状态，
            // 此处就检查mPrivateFlags是否含有PFLAG_MEASURED_DIMENSION_SET状态即可判断setMeasuredDimension是否被调用
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        // mOldWidthMeasureSpec和mOldHeightMeasureSpec保存着最近一次测量时的MeasureSpec，
        // 在测量完成后将这次新传入的MeasureSpec赋值给它们
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        //最后用上面计算出的key作为键，测量结果作为值，将该键值对放入成员变量mMeasureCache中，
        //这样就实现了对本次测量结果的缓存，以便在下次measure方法执行的时候，有可能将其从中直接读出，
        //从而省去实际测量的步骤
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

```

这里根据上面的注释简单总结一下measure方法都干了什么事：

1. 首先，我们要知道并不是只要View的measure方法执行的时候View就一定要傻傻的真的去做测量工作，View也喜欢偷懒，如果View发现没有必要去测量的话，那它就不会真的去做测量的工作。

2. 具体来说，View先查看是不是要强制测量以及这次measure中传入的MeasureSpec与上次测量的MeasureSpec是否相同，如果不是强制测量或者MeasureSpec与上次的测量的MeasureSpec相同，那么View就不必真的去测量了。

3. 如果不满足上述条件，View就考虑去做测量工作。但是在测量之前，View还想偷懒，它会以MeasureSpec计算出的key值作为键，去成员变量mMeasureCache中查找是否缓存过对应key的测量结果。
   如果能找到，那么就简单调用一下setMeasuredDimensionRaw方法，将从缓存中读到的测量结果保存到成员变量mMeasuredWidth和mMeasuredHeight中。

4. 如果不能从mMeasureCache中读到缓存过的测量结果，那么这次View就真的不能再偷懒了，只能乖乖地调用onMeasure方法去完成实际的测量工作，并且将尺寸限制条件widthMeasureSpec和
    heightMeasureSpec传递给onMeasure方法。

5. 不论上面代码走了哪个判断的分支，最终View都会得到测量的结果，并且将结果缓存到成员变量mMeasureCache中，以便下次执行measure方法时能够从其中读取缓存值。

   需要说明的是，View有一个成员变量mPrivateFlags，用以保存View的各种状态位，在测量开始前，会将其设置为未测量状态，在测量完成后会将其设置为已测量状态。


这个measure()这个方法是final的，因此我们无法在子类中去重写这个方法，说明 Android 是不允许我们改变 View 的 measure 框架的。
只有 onMeasure 可以被子类重写。onMeasure()方法才是真正去测量并设置View大小的地方。

当View在measure方法中发现不得不进行实际的测量工作时，将会调用onMeasure方法，并且将尺寸限制条件widthMeasureSpec和heightMeasureSpec作为参数传递给onMeasure方法。
View 的onMeasure方法不是空方法，它提供了一个默认的具体实现。 


```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

```

先看看 getSuggestedMinimumWidth 到底干啥的～

````
    protected int getSuggestedMinimumWidth() {
            //如果没有给View设置背景，那么就返回View本身的最小宽度mMinWidth
           //如果给View设置了背景，那么就取View本身最小宽度mMinWidth和背景的最小宽度的最大值
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

1. 如果没有给View设置背景，那么就返回View本身的最小宽度mMinWidth

2. 如果给View设置了背景，那么就取View本身最小宽度 mMinWidth 和背景的最小宽度的最大值

有两种办法给View设置最小宽度：

第一种情况是，mMinWidth是在View的构造函数中被赋值的，View通过读取XML中定义的minWidth的值来设置View的最小宽度mMinWidth，
以下代码片段是View构造函数中解析minWidth的部分：

```
//遍历到XML中定义的 minWith 属性
case R.styleable.View_minWidth:
//读取XML中定义的属性值作为mMinWidth，如果XML中未指定 minWith，则设置为0
mMinWidth = a.getDimensionPixelSize(attr, 0);
break;

```

第二种情况是调用View的setMinimumWidth方法给View的最小宽度mMinWidth赋值，setMinimumWidth方法的代码如下所示：

```
public void setMinimumWidth(int minWidth) {
    mMinWidth = minWidth;
    requestLayout();
}
```


那么mBackground.getMinimumWidth()是什么呢?我们看一下 Drwable 的 getMinimumWidth 方法，如下所示：

```
 public int getMinimumWidth() {
        final int intrinsicWidth = getIntrinsicWidth();
        return intrinsicWidth > 0 ? intrinsicWidth : 0;
    }
```

可以看出，getMinimumWidth返回的就是Drawable的原始宽度，前提是这个Drawable有原始宽度，否则就返回0。那么Drawable在什么情况下有原始宽度呢？
这里先举个例子说明一下，ShapeDrawable 无原始宽/高，而 BitmapDrawable 有原始宽/高(图片的尺寸)。

Android 会将 View 想要的尺寸以及其父控件对其尺寸限制信息 measureSpec 传递给 getDefaultSize 方法，该方法要根据这些综合信息计算最终的测量的尺寸。

```
    public static int getDefaultSize(int size, int measureSpec) {
        //size表示的是View想要的尺寸信息，比如最小宽度或最小高度
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        //如果mode是UNSPECIFIED，表示View的父ViewGroup没有给View在尺寸上设置限制条件
        case MeasureSpec.UNSPECIFIED:
        //此处当mode是UNSPECIFIED时，View就直接用自己想要的尺寸size作为测量的结果
            result = size;
            break;
        //如果mode是 AT_MOST，那么表示View最大可以取其父ViewGroup给其指定的尺寸
        //如果mode是 EXACTLY，那么表示View必须使用其父ViewGroup指定的尺寸
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
        //此处mode是 AT_MOST 或 EXACTLY 时，View就用其父ViewGroup指定的尺寸作为测量的结果
            break;
        }
        return result;
    }

```

从getDefaulSize方法的实现来看，View的宽/高由 specSize 决定，所以我们可以得出如下结论：直接继承View的自定义控件需要重写 onMeasure 方法并设置 wrap_content 时的自身大小，
否则在布局中使用 wrap_content 就相当于使用 match_parent。为什么呢?这个原因需要结合上述代码和之前的表才能更好地理解。从上述代码中我们知道，如果View在布局中使用 wrap_content，
那么它的 specMode 是AT_MOST模式，在这种模式下，它的宽/高等于 specSize；查表4-1可知，这种情况下View 的 specSize 是 parentSize,而 parentSize 是父容器中目前可以使用的大小，
也就是父容器当前剩余的空间大小。很显然，View的宽/高就等于父容器当前剩余的空间大小，这种效果和在布局中使用match_parent完全一致。如何解决这个问题呢?也很简单，代码如下所示:

```
   @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getMode(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getMode(heightMeasureSpec);
        if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWidth, mHeight);
        } else if (widthSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWidth, heightSpecSize);
        } else if (eightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(widthSpecSize, mHeight);
        }
    }

```

在上面的代码中，我们只需要给View指定一个默认的内部宽/高(mWidth和mHeight)),并在wrapcontent时设置此宽/高即可。对于非 wrapcontent 情形，我们沿用系统的测量值即可，至于这个默认的内部宽/高的大小如何指定，这个没有固定的依据，根据需要灵活指定即可。如果查看TextView、Imageview等的源码就可以知道，
针对 wrapcontent情形，它们的onMeasure方法均做了特殊处理。

这样在onMeasure方法中， 当执行getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec)时，我们就得到了最终测量到的宽度值； 
当执行getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)时，我们就得到了最终测量到的高度值。

## ViewGroup#measure过程

对于ViewGroup来说，除了完成自己的measure过程以外，还会遍历去调用所有子元素的measure方法，各个子元素再通归去执行这个过程。和 View 不同的是，ViewGroup 是一个抽象类, 因此它没有重写 View 的 onMeasure 方法, 但是它提供了一个叫 measureChildren:

```
   protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }

```

从上述代码中看到，在 ViewGroup 的 measure 时，会对每一个子元素进行测量，那么这个方法就很好理解了：

```
 protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

```


很显然，measureChild 的思想就是取出子元素的 LayoutParams，然后再通过 getChidMeasureSpec 来创建子元素的 MeasureSpec。接着将 MeasureSpec 直接传递给 View 的 measure 方法来进行测量。

我们知道，ViewGroup 并没有定义其测量的具体过程，这是因为 ViewGroup 是一个抽象类，其测量过程的 onMeasure 方法需要各个子类去具体实现。
比如 LinearLayout，RelativeLayout等，为什么ViewGroup不像 View一样对其 onMeasure方法做统一的实现呢？那是因为不同的ViewGroup子类有不同的布局特性，这导致它们的测量细节各不相同，
比如 Lineartayout 和RelativeLayout 这两者的布局特性显然不同，因此ViewGroup无法做统一实现。下面就通过 LinearLayout 的 onMeasure 方法来分析 ViewGroup 的 measure过程。

首先，我们来看一下LinearLayout的onMeasure方法：

```
   @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }

```

上述的代码很简单我们选择一个来看下，比如选中竖直方向的LinearLayout测量过程，即 measureVertical：

```
    void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
        mTotalLength = 0;
        int maxWidth = 0;
        int childState = 0;
        int alternativeMaxWidth = 0;
        int weightedMaxWidth = 0;
        boolean allFillParent = true;
        float totalWeight = 0;

        final int count = getVirtualChildCount();

        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        boolean matchWidth = false;
        boolean skippedMeasure = false;

        final int baselineChildIndex = mBaselineAlignedChildIndex;
        final boolean useLargestChild = mUseLargestChild;

        int largestChildHeight = Integer.MIN_VALUE;
        int consumedExcessSpace = 0;

        int nonSkippedChildCount = 0;

        // See how tall everyone is. Also remember max width.
        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                mTotalLength += measureNullChild(i);
                continue;
            }

            if (child.getVisibility() == View.GONE) {
               i += getChildrenSkipCount(child, i);
               continue;
            }

            nonSkippedChildCount++;
            if (hasDividerBeforeChildAt(i)) {
                mTotalLength += mDividerHeight;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            totalWeight += lp.weight;

            final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
            if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
                // Optimization: don't bother measuring children who are only
                // laid out using excess space. These views will get measured
                // later if we have space to distribute.
                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
                skippedMeasure = true;
            } else {
                if (useExcessSpace) {
                    // The heightMode is either UNSPECIFIED or AT_MOST, and
                    // this child is only laid out using excess space. Measure
                    // using WRAP_CONTENT so that we can find out the view's
                    // optimal height. We'll restore the original height of 0
                    // after measurement.
                    lp.height = LayoutParams.WRAP_CONTENT;
                }

                // Determine how big this child would like to be. If this or
                // previous children have given a weight, then we allow it to
                // use all available space (and we will shrink things later
                // if needed).
                final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
                measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                        heightMeasureSpec, usedHeight);

                final int childHeight = child.getMeasuredHeight();
                if (useExcessSpace) {
                    // Restore the original height and record how much space
                    // we've allocated to excess-only children so that we can
                    // match the behavior of EXACTLY measurement.
                    lp.height = 0;
                    consumedExcessSpace += childHeight;
                }

                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                       lp.bottomMargin + getNextLocationOffset(child));

                if (useLargestChild) {
                    largestChildHeight = Math.max(childHeight, largestChildHeight);
                }
            }

            /**
             * If applicable, compute the additional offset to the child's baseline
             * we'll need later when asked {@link #getBaseline}.
             */
            if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
               mBaselineChildTop = mTotalLength;
            }

            // if we are trying to use a child index for our baseline, the above
            // book keeping only works if there are no children above it with
            // weight.  fail fast to aid the developer.
            if (i < baselineChildIndex && lp.weight > 0) {
                throw new RuntimeException("A child of LinearLayout with index "
                        + "less than mBaselineAlignedChildIndex has weight > 0, which "
                        + "won't work.  Either remove the weight, or don't set "
                        + "mBaselineAlignedChildIndex.");
            }

            boolean matchWidthLocally = false;
            if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
                // The width of the linear layout will scale, and at least one
                // child said it wanted to match our width. Set a flag
                // indicating that we need to remeasure at least that view when
                // we know our width.
                matchWidth = true;
                matchWidthLocally = true;
            }

            final int margin = lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);
            childState = combineMeasuredStates(childState, child.getMeasuredState());

            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
            if (lp.weight > 0) {
                /*
                 * Widths of weighted Views are bogus if we end up
                 * remeasuring, so keep them separate.
                 */
                weightedMaxWidth = Math.max(weightedMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            } else {
                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            }

            i += getChildrenSkipCount(child, i);
        }

        if (nonSkippedChildCount > 0 && hasDividerBeforeChildAt(count)) {
            mTotalLength += mDividerHeight;
        }

        if (useLargestChild &&
                (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
            mTotalLength = 0;

            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null) {
                    mTotalLength += measureNullChild(i);
                    continue;
                }

                if (child.getVisibility() == GONE) {
                    i += getChildrenSkipCount(child, i);
                    continue;
                }

                final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                        child.getLayoutParams();
                // Account for negative margins
                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                        lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
            }
        }

        // Add in our padding
        mTotalLength += mPaddingTop + mPaddingBottom;

        int heightSize = mTotalLength;

        // Check against our minimum height
        heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

        // Reconcile our calculated size with the heightMeasureSpec
        int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
        heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
        // Either expand children with weight to take up available space or
        // shrink them if they extend beyond our current bounds. If we skipped
        // measurement on any children, we need to measure them now.
        int remainingExcess = heightSize - mTotalLength
                + (mAllowInconsistentMeasurement ? 0 : consumedExcessSpace);
        if (skippedMeasure
                || ((sRemeasureWeightedChildren || remainingExcess != 0) && totalWeight > 0.0f)) {
            float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

            mTotalLength = 0;

            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }

                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                final float childWeight = lp.weight;
                if (childWeight > 0) {
                    final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
                    remainingExcess -= share;
                    remainingWeightSum -= childWeight;

                    final int childHeight;
                    if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                        childHeight = largestChildHeight;
                    } else if (lp.height == 0 && (!mAllowInconsistentMeasurement
                            || heightMode == MeasureSpec.EXACTLY)) {
                        // This child needs to be laid out from scratch using
                        // only its share of excess space.
                        childHeight = share;
                    } else {
                        // This child had some intrinsic height to which we
                        // need to add its share of excess space.
                        childHeight = child.getMeasuredHeight() + share;
                    }

                    final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.max(0, childHeight), MeasureSpec.EXACTLY);
                    final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                            lp.width);
                    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                    // Child may now not fit in vertical dimension.
                    childState = combineMeasuredStates(childState, child.getMeasuredState()
                            & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
                }

                final int margin =  lp.leftMargin + lp.rightMargin;
                final int measuredWidth = child.getMeasuredWidth() + margin;
                maxWidth = Math.max(maxWidth, measuredWidth);

                boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                        lp.width == LayoutParams.MATCH_PARENT;

                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);

                allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                        lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
            }

            // Add in our padding
            mTotalLength += mPaddingTop + mPaddingBottom;
            // TODO: Should we recompute the heightSpec based on the new total length?
        } else {
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                           weightedMaxWidth);


            // We have no limit, so make all weighted views as tall as the largest child.
            // Children will have already been measured once.
            if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
                for (int i = 0; i < count; i++) {
                    final View child = getVirtualChildAt(i);
                    if (child == null || child.getVisibility() == View.GONE) {
                        continue;
                    }

                    final LinearLayout.LayoutParams lp =
                            (LinearLayout.LayoutParams) child.getLayoutParams();

                    float childExtra = lp.weight;
                    if (childExtra > 0) {
                        child.measure(
                                MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                        MeasureSpec.EXACTLY),
                                MeasureSpec.makeMeasureSpec(largestChildHeight,
                                        MeasureSpec.EXACTLY));
                    }
                }
            }
        }

        if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
            maxWidth = alternativeMaxWidth;
        }

        maxWidth += mPaddingLeft + mPaddingRight;

        // Check against our minimum width
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                heightSizeAndState);

        if (matchWidth) {
            forceUniformWidth(count, heightMeasureSpec);
        }
    }
```
从上面的代码可以看出，系统会遍历子元素并对每一个子元素执行 measureChildBeforeLayout 方法，这个方法内部会调用子元素的 measure 方法，这样各个子元素就开始依次进入 measure 过程，
并且系统通过 mTotalLength 这个变量来存储 LinearLayout 在竖直方向上的初步高度，每测量一个子元素，mTotalLength 就会增加，增加的部分主要包括子元素的高度以及竖直方向上的 margin 等。
当子元素测量完毕之后，LinearLayout会根据子元素的情况来测量自己的大小，针对竖直的LinearLayout而言，他的水平方向的测量过程遵循View的测量过程，在竖直方向的测量过程和View有些不同，
具体来说，是指，如果他的布局中高度采用的是match_parent或者具体值，那么他的绘制过程和View一致，即高度为specSize，如果他的布局中高度采用warp_content，那么它的高度是所有的子元素所占用的高度综合，
但是仍然不能超过他的父容器剩余空间，但是他的最终高度还是需要考虑其他的竖直方向上的 pading。

  View的onMeasure是三大流程中最复杂的一个，measure完成以后，通过 getMeasureWidth/Height 就可以正确地获取到View的测量宽/高。需要注意的是，在某些极端情况下measure才能确定最终的测量宽/高，
在这种情形下，系统可能要多次调用measure方法进行测量，在这种情况下，在onMeasure方法中拿到的测量值很可能是不准确的。

__一个比较好的习惯是在onLayout方法中去获取View的测量宽/高或者最终宽/高__。


现在考虑一种情况，比如我们想在 Activity 已启动的时候就做一件任务，但是这一件任务需要获取某个View的宽/高，读者可能会说，这很简单啊，在 onCreate 或者 onResume 里面去获取这个View的宽/高就行了，
实际上在 onCreate、onStart、onResume 中均无法正确得View的宽/高信息，这是因为 View 的 measure 过程和 Activity 的生命周期方法不是同步执行的，因此无法保证 Activiy 执行了 onCreate、onStart、
onResume 时某个Vicw已经测量完毕了。如果View还没有测量完毕，那么获得的宽/高就是0。

有没有什么方法能解决问题呢？

答案是有的，这里给出四种方法来解决这个问题：

1. onWindowFocusChanged这个方法的含义是：__View已经初始化完毕了，宽/高已经准备好了，这个时候去获取宽/高是没问题的__。需要注意的是，onWindowFocusChanged 会被调用多次。
当 Activity 的窗口得到焦点和失去焦点时均会被调用一次。具体来说，当 Activity 继续执行和暂停执行时，onWindowFocusChanged 均会被调用，如果频繁地进行onResume和onPause, 那么onWindowFocusChanged也会被频繁地调用。典型代码如下：

```
 public void onWindowFocusChanged(boolean hasWindowFocus) {
        InputMethodManager imm = InputMethodManager.peekInstance();
        if (!hasWindowFocus) {
            if (isPressed()) {
                setPressed(false);
            }
            if (imm != null && (mPrivateFlags & PFLAG_FOCUSED) != 0) {
                imm.focusOut(this);
            }
            removeLongPressCallback();
            removeTapCallback();
            onFocusLost();
        } else if (imm != null && (mPrivateFlags & PFLAG_FOCUSED) != 0) {
            imm.focusIn(this);
        }
        refreshDrawableState();
    }
```


2. view.post(runnable)

  通过post可以将一个 runnable 投递到消息队列，然后等到 Lopper 调用 runnable 的时候，View 也就初始化好了，典型代码如下:

```
    @Override
    protected void onStart() {
        super.onStart();

        mTextView.post(new Runnable() {
            @Override
            public void run() {
                int width = mTextView.getMeasuredWidth();
                int height = mTextView.getMeasuredHeight();
            }
        });
    }

```

3. ViewTreeObserver

  使用 ViewTreeObserver 的众多回调可以完成这个功能，比如使用 OnGlobalLayoutListener 这个接口，当 View 树的状态发生改变或者 View 树内部的 View 的可见性发生改变，
onGlobalLayout方法就会回调，因此这是获取 View 的宽高一个很好的例子，需要注意的是，伴随着 View 树状态的改变，这个方法也会被调用多次，典型代码如下:

```
@Override
    protected void onStart() {
        super.onStart();

        ViewTreeObserver observer = mTextView.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                mTextView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
                int width = mTextView.getMeasuredWidth();
                int height = mTextView.getMeasuredHeight();
            }
        });
    }
```

4. view.measure(int widthMeasureSpec , int heightMeasureSpec)


   通过手动对 View 进行 measure 来得到View的的宽高，这种方法比较复杂，这里要分情况来处理，根据 View 的 LayoutParams 来分：

   * match_parent


   直接放弃，无法测量出具体的宽高，根据View的测量过程，构造这种 measureSpec 需要知道 parentSize，即父容器的剩下空间。
   而这个时候我们无法知道 parentSize 的大小，所以理论上我们不可能测量出View的大小。

  
   * 具体的数值


     比如宽高都是100dp，那我们可以这样:

	```
		int widthMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY);
		int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY);
		mTextView.measure(widthMeasureSpec,heightMeasureSpec);

	```


## setMeasuredDimension

setMeasuredDimension会调用getDefaultSize方法，会将已经测量到的宽度值和高度值作为参数传递给setMeasuredDimension方法。

```
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        //layoutMode是LAYOUT_MODE_OPTICAL_BOUNDS的特殊情况，我们不考虑
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
      //最终调用setMeasuredDimensionRaw方法，将测量结果传入进去
     setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}
```

setMeasuredDimension方法最后将测量的结果传递给方法setMeasuredDimensionRaw，我们再研究一下setMeasuredDimensionRaw这方法。

```
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    //将测量完成的宽度measuredWidth保存到View的成员变量mMeasuredWidth中
    mMeasuredWidth = measuredWidth;
    //将测量完成的高度measuredHeight保存到View的成员变量mMeasuredHeight中
    mMeasuredHeight = measuredHeight;
    //最后将View的状态位mPrivateFlags设置为已测量状态
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

测量完成的尺寸的state

至此，View的测量过程就完成了，但是View的父ViewGroup如何读取到View测量的结果呢？

为此，View提供了三组方法，分别是： 

1. getMeasuredWidth和getMeasuredHeight方法 

2. getMeasuredWidthAndState和getMeasuredHeightAndState方法 
	
3. getMeasuredState方法

后面那两组方法有啥用？

此处我们要再仔细研究一下 View 中保存测量结果的成员变量 mMeasuredWidth 和 mMeasuredHeight，下面的讨论我们都只讨论宽度，理解了宽度的处理方式，高度也是完全一样的。

mMeasuredWidth是一个Int类型的值，其是由4个字节组成的。

我们先假设mMeasuredWidth只存储了测量完成的宽度信息，而且View的父ViewGroup可以通过相关方法得到该值。但是存在这样一种情况：View在测量时，
父ViewGroup给其传递的widthMeasureSpec中的specMode的值是AT_MOST，specSize是100，但是View的最小宽度是200，显然父ViewGroup指定的specSize不能满足View的大小，
但是由于specMode的值是AT_MOST，View在getDefaultSize方法中不得不妥协，只能含泪将测量的最终宽度设置为100。然后其父ViewGroup通过某些方法获取到该View的测量宽度为100时，
ViewGroup以为子View只需要100就够了，最终给了子View宽度为100的空间，这就导致了在UI界面上View特别窄，用户体验也就不好。


Android为让其View的父控件获取更多的信息，就在mMeasuredWidth上下了很大功夫，虽然是一个Int值，但是想让它存储更多信息，具体来说就是把mMeasuredWidth分成两部分：

* 其高位的第一个字节为第一部分，用于标记测量完的尺寸是不是达到了View想要的宽度，我们称该信息为测量的state信息。

* 其低位的三个字节为第二部分，用于存储实际的测量到的宽度。


View的measure方法还是比较聪明的，知道如何偷懒利用以前测量过的数据，如果情况有变，那么就调用onMeasure方法进行实际的测量工作，在onMeasure中，View要根据父ViewGroup给其传递进来的.widthMeasureSpec和heightMeasureSpec，并结合View自身想要的尺寸，综合考虑，计算出最终的测量的宽度和高度，并存储到相应的成员变量中，这才标志着该View测量有效的完成了，如果没有将值存入到成员变量中，View会抛出异常。在该成员变量中有可能也存储了测量过程中的state信息。由于View的measure已经实现了很多逻辑判断，所以我们在自定义View或ViewGroup时，都不应该重写measure方法，而应该重写onMeasure方法，在其中实现我们自己的测量逻辑。


# 参考

[源码解析Android中View的measure测量过程](https://blog.csdn.net/iispring/article/details/49403315)
