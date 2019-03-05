通过 LayoutInflater 可以获取到布局文件中的View，任何一个View都不可能凭空突然出现在屏幕上，它们都是要经过非常科学的绘制流程后才能显示出来的。
每一个视图的绘制过程都必须经历三个最主要的阶段，即onMeasure()、onLayout()和onDraw()。

## 关于Measure几个小结论:

1. View 的 measureSpec 由其父容器的 measureSpec(mode和size) 及自身的 LayoutParams(具体值,wrapContent,matchParent) 共同决定. 对于        ViewRoot其测量模式为Exactly,大小为Window大小.

2. 对于所有的控件,如果在布局中指定了具体数值的高和宽时,系统就会给它指定数值的高和宽的空间.至于屏幕上有没有那么多空间给这些控件那是另外一回事,其实如果我    们尝试在LinearLayout中,将Button A的布局宽度设置为屏幕宽度的1/2,将Button B的布局宽度设置为整个屏幕的宽.假设在系统measure,先测量的Button A,    那么运行时会发现Button B的宽度只会显示出其宽度的1/2.

3. 当一个控件的布局宽度设置为match_parant时,此时传给它的 measureSize 就是其父控件可用的布局宽度值,它的 measureMode为 Exactly.那么该控件实际显    示的宽度还要看其自身 OnMeasure 中在 mode 为 Exactly 时的实现.

4. 当一个控件的布局宽度设置为wrap_content时,如果其父View为Exactly,那么其mode为At_Most,size为父View可用的布局宽度值.如果其父View的mode为        At_most,那么传给子View的mode为At_most,size为父View可用的布局宽度值.

5. 对View进行测量的目的是让View的父控件知道View想要多大的尺寸.当我们的子View在布局文件中,设置具体的布局大小时,其实就是在直接告诉父View,我想要这么    大的布局空间.当我们的子View在布局文件中为match_parent时,其实就是在告诉父View,你有多大布局空间就给我多大的布局空间,贪婪呀~~~~ .当我们的子View    在布局文件中为wrap_content时,其实是在告诉父View,我不贪,我只要够我用的布局空间就可以了.此时父View传给子View的size其实还是父view的可用空间.


## View的MeasureSpec的创建规则

   ![](https://github.com/xianfeng92/android-code-read/blob/master/images/view_measureSpec.png)

   父View传给子View的mode = 父View + 子view的布局参数的设定
   
   我们在布局文件中设置一个View的布局宽高时,有三种选择:具体数值(dp),match_parent,wrap_content,而对于ViewRoot其mode为Exactly,所有一般
   我们只需要考虑mode为Exactly和At_most两种情况.当布局为一个很大的View树时,从根View开始,父View传给子View的一般都是Exactly或At_most.
   
   所有说,我们在自定义View的时候,重点需要考虑,当该View在布局文件中是match_parent 或 wrap_content 时,我们希望它以何种大小显示(这是我们要根据我们    的需要来决定的)
   
   
## onMeasures

__对View进行测量的目的是让View的父控件知道View想要多大的尺寸__。

整个应用测量的起点是 ViewRootImpl 类，从它开始依次对子 View 进行测量，如果子View是一个 ViewGroup，那么又会遍历该 ViewGroup 的子 View 依次进行测量。 __也就是说，测量会从 View 树的根结点，纵向递归进行，从而实现自上而下对View树进行测量，直至完成对叶子节点View的测量__。

那么到底如何对一个View进行测量呢？

Android通过调用View的measure()方法对View进行测量，让该View的父控件知道该View想要多大的尺寸空间。

具体来说，View 的父控件 ViewGroup 会调用 View 的 measure 方法，ViewGroup 会将一些宽度和高度的限制条件传递给 View 的 measure 方法。

在View的measure方法会首先从成员变量中读取以前缓存过的测量结果，如果能找到该缓存值，那么就基本完事了，如果没有找到缓存值，那么measure方法会执行onMeasure回调方法，
measure方法会将上述的宽度和高度的限制条件依次传递给onMeasure方法。onMeasure方法会完成具体的测量工作，并将测量的结果通过调用 View 的 setMeasuredDimension
方法保存到 View 的成员变量 mMeasuredWidth 和 mMeasuredHeight 中。

测量完成之后，View的父控件就可以通过调用 getMeasuredWidth、getMeasuredState、getMeasuredWidthAndState 这三个方法获取View的测量结果。

specMode 一共有三种类型，如下所示：

1. EXACTLY

   该值表示View必须使用其父ViewGroup指定的尺寸

2. AT_MOST

   该值表示View最大可以取其父ViewGroup给其指定的尺寸

3. UNSPECIFIED
   
   该值表示View的父ViewGroup没有给View在尺寸上设置限制条件，这种情况下View可以忽略measureSpec中的size，View可以取自己想要的值作为测量的尺寸。

widthMeasureSpec和heightMeasureSpec这两个值又是从哪里得到的呢？通常情况下，这两个值都是由父视图经过计算后传递给子视图的，说明父视图会在一定程度上决定子视图的大小。
但是最外层的根视图，它的 widthMeasureSpec 和 heightMeasureSpec 又是从哪里得到的呢？ 这就需要去分析ViewRoot中的源码了，观察performTraversals()方法可以发现如下代码：

```
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);

```

可以看到，这里调用了getRootMeasureSpec()方法去获取 widthMeasureSpec 和 heightMeasureSpec 的值，注意方法中传入的参数，其中lp.width和lp.height在创建ViewGroup实例的时候就被赋值了，
它们都等于MATCH_PARENT。

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

可以看到，这里使用了 MeasureSpec.makeMeasureSpec() 方法来组装一个 MeasureSpec，当 rootDimension 参数等于 MATCH_PARENT 的时候，MeasureSpec 的 specMode 就等于 EXACTLY，
当rootDimension等于WRAP_CONTENT的时候，MeasureSpec的specMode就等于AT_MOST。并且MATCH_PARENT和WRAP_CONTENT时的specSize都是等于windowSize的，
也就意味着根视图总是会充满全屏的。

当 View 的父控件 ViewGroup 对View进行测量时，会调用View的measure方法，ViewGroup会传入 widthMeasureSpec 和 heightMeasureSpec ，分别表示父控件对View的宽度和高度的一些限制条件。

view中的measure方法：

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

2. 如果给View设置了背景，那么就取View本身最小宽度mMinWidth和背景的最小宽度的最大值

有两种办法给View设置最小宽度：

第一种情况是，mMinWidth是在View的构造函数中被赋值的，View通过读取XML中定义的minWidth的值来设置View的最小宽度mMinWidth，
以下代码片段是View构造函数中解析minWidth的部分：

```
//遍历到XML中定义的minWith属性
case R.styleable.View_minWidth:
//读取XML中定义的属性值作为mMinWidth，如果XML中未定义，则设置为0
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

* 首先根据 measuredSpec 解析出对应的 specMode 和 specSize

* 当mode是 UNSPECIFIED 时，View就直接用自己想要的尺寸size作为测量的结果

* 当mode是 AT_MOST 或 EXACTLY时，View就用其父ViewGroup指定的尺寸作为测量的结果

最终，View会根据measuredSpec限制条件，得到最终的测量的尺寸。

这样在onMeasure方法中， 当执行getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec)时，我们就得到了最终测量到的宽度值； 
当执行getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)时，我们就得到了最终测量到的高度值。

setMeasuredDimension方法：

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


resolveSizeAndState

resolveSizeAndState除了返回最终尺寸信息还会有可能返回测量的state标志位信息。


View的measure方法还是比较聪明的，知道如何偷懒利用以前测量过的数据，如果情况有变，那么就调用onMeasure方法进行实际的测量工作，在onMeasure中，View要根据父ViewGroup给其传递进来的.widthMeasureSpec和heightMeasureSpec，并结合View自身想要的尺寸，综合考虑，计算出最终的测量的宽度和高度，并存储到相应的成员变量中，这才标志着该View测量有效的完成了，如果没有将值存入到成员变量中，View会抛出异常。在该成员变量中有可能也存储了测量过程中的state信息。由于View的measure已经实现了很多逻辑判断，所以我们在自定义View或ViewGroup时，都不应该重写measure方法，而应该重写onMeasure方法，在其中实现我们自己的测量逻辑。


# 参考
[源码解析Android中View的measure测量过程](https://blog.csdn.net/iispring/article/details/49403315)
