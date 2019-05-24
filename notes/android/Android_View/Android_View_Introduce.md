
## 什么是View

View是Android中所有控件的基类，不光是简单的 Button 和 TextView 还是复杂的 RelativeLayout 和 Listview ,它们的共同基类都是View。
__所以说，View是一种界面层的控件的一种抽象，它代表了一个控件__。除了View，还有 ViewGroup，ViewGroup 内部包含了许多个控件，即一组View。
在Android的设计中，ViewGroup 也继承了View，这就意味着 View 本身就可以是单个控件也可以是由多个控件组成的一组控件，通过这种关系就形成了View树的结构。
根据这个概念，我们知道，Button 显然是个 View，而 LinearLayout 不但是一个 View 而且还是一个 ViewGroup，而 ViewGroup 内部是可以有子View的，这个子View同样还可以是ViewGroup。


## View的位置参数

View 的位置主要由它的四个顶点来决定，分别对应于 View 的四个属性：top、left、right，bottom，其中top是左上角纵坐标，left是左上角横坐标，right是右下角横坐标，bottom是有下角纵坐标。
需要注意的是，这些坐标都是相对于View的父容器来说的，因此它是一种相对坐标。在Android中，x轴和y轴的正别为右和下。


![view_location.png]()


从图中的关系我们很容易得到宽高的关系:

```
width= right- left
height = bottom - top

```

那么如何得到View的四个参数呢？也很简单，在对应的源码众有这四个方法:

```
Left = getLeft();
Right = getRight();
Top = getTop();
Bottom = getBottom():

```

从Android3.0开始，View增加了额外的几个参数，x,y，translationX,translationY,其中x，y是View左上角的图标，而 translationX,translationY 是左上角相对父容器的偏移角量，这几个参数也是相对于父容器的坐标。
并且 translationX,translationY 的默认值为 0；和 View 的四个基本位置参数一样，View也为我们提供了get/set方法这几个换算关系：


```
x = left + translationX
y = top + translationY

```

需要注意的是： __View在平移的过程中，top和left表示在原始左上角的位置信息，其值并不会发生什么，此时发生改变的是 x,y,translationX,translationY 这四个参数。__



## MotionEvent和TouchSlop

### MotionEvent

在手指接触屏幕后所产生的一系列事件中，典型的事件类型有如下几种：

1. ACTION_DOWN一手指刚接触屏幕

2. ACTION_MOVE一—手指在屏幕上移动

3. ACTION_UP——手机从屏幕上松开的一瞬间


正常情况下，一次手指触摸屏幕的行为会触发一系列点击事件，考虑如下几种情况:

1. 点击屏幕后离开松开，事件序列为DOWN->UP

2. 点击屏幕滑动一会再松开，事件序列为DOwN > MOVE >…..>MOVE-Up


上述三种情况是典型的事件序列，同时通过 MotionEvent 对象我们可以得到点击事件发生的x和y坐标。为此，系统提供了两组方法：getX/gety和 getRawX/getRawY。
它们的区别其实很简单，getX/getY返回的是相对于当前View左上角的x和y坐标，而geiRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。


### TouchSlop

__TouchSlop是系统所能识别出的被认为是滑动的最小距离__，换句话说，当手指在屏慕上滑动时，如果两次滑动之间的距离小于这个常量，那么系统就不认为你是在进行滑动操作。
原理很简单，滑动的距离太短，系统不认为他在滑动，这是一个常量，和设备无关，在不同的设备下这个值可能不同，通过如下方式即可获取这个常量：
ViewConfigurtion.get(getContext()).getScaledTouchSlop,这个常量有什么意义呢?当我们在处理滑动时，可以利用这个常量来做一些过滤，比如当两次滑动事件的滑动距离小于这个值，
我们就可以认为未达到常动距离的临界值，因此就可以认为它们不是滑动，这样做可以有更好的用户体验在fraweworks/base/core/res/va;ues/config.xml中，就有这个常量的定义



## VelocityTracker,GestureDetector 和 Scroller

### VelocityTracker

速度追踪，用于追踪手指在屏幕上滑动的速度，包括水平和竖直方向上的速度使用过程很简单，首先，在View的onTouchEvent方法里追踪：

```
  VelocityTracker velocityTracker = VelocityTracker.obtain();
  velocityTracker.addMovement(event);
```

接着，当我们想知道当前的滑动速度时，这个时候可以采用如下的方式得到当前的速度：

```
velocityTracker.computeCurrentVelocity(1000);
int xVelocity = (int) velocityTracker.getXVelocity();
int yVelocity = (int) velocityTracker.getYVelocity();
```

1. 第一点获取速度的之前必须先计算速度，即 getXVelocity 和 getYVelocity 这两个方法前面一定要调用 computeCurrentVelocity 方法

2. 第二点，这里的速度是指一段时间内手指滑动的屏幕像素，比如将时间设置为1000ms时，在1s内，手指在水平方向手指滑动100像素，那么水平速度就是100。
   注意速度可以为负数，当手指从右向左滑动的时候为负，这个需要理解一下，速度的计算公式如下表示：

			```
			速度 = （终点位置 -  起点位置）/时间段

			```

根据上面的公式再加上Android系统的坐标系，可以知道，手指逆着坐标系的正方向滑动， 所产生的速度为赋值。

最后，当不需要使用它的时候，需要调用clear方法来重置并回收内存:

```
velocityTracker.clear();
velocityTracker.recycle();

```


### GestureDetector


手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。要使用 GestureDetector 也不复杂参考如下过程。

首先，需要创建一个 GestureDetector 对象并实现 OnGestureListener 接口，根据需要我们还可以实现 OnDoubleTapListener 从而能够监听双击行为;

			```
			GestureDetector mGestureDetector = new GestureDetector(this);
			//解决长按屏幕后无法拖动的现象
			mGestureDetector.setIsLongpressEnabled(false);
			```

接着，接管目标View的onTouchEvent方法，在待监听View的 onTouchEvent 方法中添加如下实现：

			```
			boolean consum = mGestureDetector.onTouchEvent(event);
			return consum;
			```



做完了上面两步，我们就可以有选择地实现 OnGestureListener 和 OnDoubleTapListener 中的方法了，这两个接口中的方法介绍如下图：


![gestureDetector]()


### Scroller

弹性滑动对象，用于实现 View 的弹性滑动，我们知道，当使用 View 的 scrollTo/scrollBy 方法来进行滑动的时候，其过程是瞬间完成的，这个没有过度效果的滑动用户体验肯定是不好的。
这个时候就可以用Scroller来实现过度效果的滑动，其过程不是瞬间完成的，而是在一定的时间间隔去完成的，Scroller本身是无法让 View 弹性滑动。
它需要和 view 的 computScrioll 方法配合才能完成这个功能。

```
scroller = new Scroller(getContext());

private void smoothScrollTo(int destX,int destY){
        int scrollX = getScrollX();
        int delta = destX - scrollX;
        //1000ms内滑向destX,效果就是慢慢的滑动
        scroller.startScroll(scrollX,0,delta,0,1000);
        invalidate();
    }

@Override
public void computeScroll() {
        if(scroller.computeScrollOffset()){
            scrollTo(scroller.getCurrX(),scroller.getCurrY());
            postInvalidate();
        }
 }
```



## View的滑动

在Android设备上，滑动几乎是应用的标配，不管是下拉刷新还是 SlidingMenu，它们的基础都是滑动。从另外一方面来说，Android手机由于屏幕比较小，为了给用户呈现更多的内容，
就需要使用滑动来隐藏和显示一些内容。基于上述两点，可以知道，滑动在Android开发中具有很重要的作用，不管一些滑动效果多么绚丽，归根结底，它们都是由不同的滑动外加一些特效所组成的。
因此，掌握滑动的方法是实现绚丽的自定义控件的基础。

通过三种方式可以实现View的滑动：

###  第一种是通过View本身提供的 scrollTo/scrollBy 方法来实现滑动

为了实现View的滑动，View提供了专门的方法来实现这个功能，那就是scrollTo/scrollBy，我们先来看看这两个方法的实现，如下所示：

```
  /**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
```

从上面的源码可以看出，scrollBy 实际上也是调用了 scrolrTo 方法，它实现了基于当前位置的相对滑动，而 scrollTo 则实现了基于所传递参数的绝对滑动。
利用 scrollTo 和 scrollBy 来实现 View 的滑动，这不是一件困难的事，但是我们要明白滑动过程，View 内部的两个属性 mScrollX 和 mScrollY 的改变规则，
这两个属性可以通过getScrollX和getScrollY方法分别得到。

这里先简要概况一下：

1. 在滑动过程中，__mScrollX的值总是等于 View 左边缘和 View 内容左边缘在水平方向的距离，而 mScrollY 的值总是等于 View 上边缘和 View 内容上边缘在竖直方向的距离__。

2. View 边缘是指View的位置，由四个顶点组成，而 View 内容边缘是指View中的内容的边缘，scrolTo 和 scrollBy 只能改变View内容的位置而不能变View在布局中的位置。

3. mScrollX和mscrollY的单位为像素，并且当View左边缘在Veiw内容左边缘的右边时，mScrolX为正值，反之为负值， 当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。

4. __如果从左向右滑动，那么mScrollX负值，反之为正值__。如果从上往下滑动，那么mScrollY为负值，反之为正值。


### 第二种是通过动画给 View 施加平移效果来实现滑动

通过动画，我们来让一个View移动，而平移就是一种滑动，使用动画来移动View，主要是操作View的translationX，translationY属性，即可以采用传统的View动画，也可以采用属性动画。

采用View动画的代码，如下所示，此动画可以在100ms里让一个View从初始的位置向右下角移动100个像素:

```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:fillAfter="true"
     android:zAdjustment="normal">

    <translate
        android:duration="100"
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:interpolator="@android:anim/linear_interpolator"
        android:toXDelta="100"
        android:toYDelta="100"
        />

</set>
```
如果采用属性动画的话，那就更简单了，我们可用这样：

```
ObjectAnimator.ofFloat(testButton,"translationX",0,100).setDuration(100).start();

```

使用动画来做 View 的滑动要注意一点，View动画是对 View 的影像做操作，它并不能真正改变 View 的位置参数，包括高宽，并且如果希望动画后的状态得以保存还必须将 fillAfter 属性设置为true。
否则动画完成之后就会消失，比如我们要把View向右移动100个像素，如果 fillAfter 为 false，那么动画完成的一刹那，View就会恢复之前的状态，fillAfter为true的话就会停留在最终点，
这是视图动画，属性动画不会有这样的问题。

上面提到的View的动画并不能真正改变View的位置，这会带来一个很严重的后果，试想一下，比如我们通过一个View动画将一个button向右移动100px，并且这个View设置点击事件，然后你会发现，
在新位置无法触发，而在老位置可以触发点击事件，所以，这只是视图的变化，在系统眼里，这个button并没有发生任何改变。他的真生仍然在原始的位置，在这种情况下，单击新位置当然不会触发点击事件了。


### 第三种是通过改变Viev的LayoutParams使得View重新布局从而实现滑动


这个比较好理解了，比如我们想把一个Button向右平移100px，我们只需要将这个 Bution 的 LayoutParams 里的 marginLeft 参数的值增加 100px 即可。

还有一种情形，view的默认宽度为0，当我们需要向右移动Button时，只需要重新设置空View的宽度即可，就自动被挤向右边，即实现了向右平移的效果。

				```
				ViewGroup.MarginLayoutParams layoutParams = (ViewGroup.MarginLayoutParams) testButton.getLayoutParams();
				layoutParams.width +=100;
				layoutParams.leftMargin +=100;
				testButton.requestLayout(); //testButton.setLayoutParams(layoutParams);
				```

通过改变LayoutParams的方式去实现View的滑动同样是一种很灵活的方法，需要根据不同情况去做不同的处理。



## 各种滑动方式的对比

* scrollTo/scrollBy：操作简单，适合对View内容的滑动

* 动画：操作简单，主要适用于没有交互的 View 和实现复杂的动画效果

* 改变布局参数：操作稍微复杂，适用于有交互的View



## 弹性滑动

知道了View的滑动，我们还要知道如何实现View的弹性滑动，比较生硬地滑动过去这种用户体验实在是太差了，因此我们要实现渐进式滑动，那么如何实现弹性滑动呢？
其实实现方法也是有很多，但都有一个共同的思想：__将一次大的滑动分成若干个小的滑动，并且在一个时间段完成__。
实现方式很多，比如Scroller，Handler#PostDelayed以及Thread#Sleep。


### Scroller

Scroller的使用方法在之前就已经介绍了，我们来分析一下他的源码，从而探索为什么能实现View的弹性滑动：

```
   Scroller scroller = new Scroller(getContext());

    private void smootthScrollTo(int destX,int destY){
        int scrollX = getScrollX();
        int deltaX = destX - scrollX;
        //1000ms内滑向destX，效果是慢慢滑动
        scroller.startScroll(scrollX,0,deltaX,0,1000);
        invalidate()；
    }

    @Override
    public void computeScroll() {
        if(scroller.computeScrollOffset()){
            scrollTo(scroller.getCurrX(),scroller.getCurrY());
            postInvalidate();
        }
    }
```

上面是Scroller的典型用法，当我们构建一个 scroller 对象并且调用它的 startScroll 方法，scroller内部其实并没有做什么，它只是保存了我们传递的参数。
这几个参数从 startScroll 的原型就可以看出，如下的代码：

```
 public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }

```

这个方法的参数含义很清楚，startX 和 startY 表示的是滑动的起点，dx 和 dy 表示的是要滑动的距离，而duration表示的是滑动时间，即整个滑动过程完成所需要的时间。
注意这里的滑动是指 View 内容的滑动而非 View 本身位置的改变。仅仅调用startScroll方法是无法让View滑动的，因为它内部并没有做滑动相关的事。

那么Scroller到底是如何让View弹性滑动的呢?

是startScroll 方法下面的 invalidate 方法。invalidate 方法会导致 View 重绘，在 View 的 draw 方法中又会调用 computeScroll方法，
computeScroll 方法在 View 中是一个空实现。因此需要我们自己去实现，上面的代码已经实现了 computeScroll 方法。正是因为这个computeScroll方法，View才能实现弹性滑动。
这看起来还是很抽象，其实这样的：当View重绘后会在draw方法中调用 computescroll，而computeScroll又会去向Scroller获取当前的scrollX 和ScrollY。
然后通过 scrolrTo 方法实现滑动；接着又调用postlnvalidate方法来进行第二次重绘，这一次重绘的过程和第一次重绘一样，还是会导致computeScroll方法被调用；然后继续向
Scroller获取当前的 scrollX 和 scrollY，并通过 scrolTTo 方法滑动到新的位置，如此反复，直到整个滑动过程结束。


我们来看下 Scroller 的 computeScrollOffset 方法的实现：

```
 public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);

        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                final float t = (float) timePassed / mDuration;
                final int index = (int) (NB_SAMPLES * t);
                float distanceCoef = 1.f;
                float velocityCoef = 0.f;
                if (index < NB_SAMPLES) {
                    final float t_inf = (float) index / NB_SAMPLES;
                    final float t_sup = (float) (index + 1) / NB_SAMPLES;
                    final float d_inf = SPLINE_POSITION[index];
                    final float d_sup = SPLINE_POSITION[index + 1];
                    velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                    distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                }

                mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;

                mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                // Pin to mMinX <= mCurrX <= mMaxX
                mCurrX = Math.min(mCurrX, mMaxX);
                mCurrX = Math.max(mCurrX, mMinX);

                mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                // Pin to mMinY <= mCurrY <= mMaxY
                mCurrY = Math.min(mCurrY, mMaxY);
                mCurrY = Math.max(mCurrY, mMinY);

                if (mCurrX == mFinalX && mCurrY == mFinalY) {
                    mFinished = true;
                }

                break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
```

这个方法会根据时间的流逝来计算当前的 scrollX 和 scrollY 的值，计算方法也很简单，__根据时间流逝的百分比来计算 scrollX 和 scrollY__。
__改变的百分比值这个过程相当于动画的插值器的概念__。这个方法的返回值也很重要，它返回true表示滑动还未结束，false表示结束。

Scroller的本身并不会滑动，需要配合 computeScroll 方法才能完成弹性滑动的效果，不断的让View重绘，而每次都有一些时间间隔。
通过这个时间间隔就能得到它的滑动位置，这样就可以用 ScrollTo 方法来完成View的滑动了，就这样，View 的每一次重绘都会导致 View 进行小幅度的滑动，
而多次的小幅度滑动形成了弹性滑动。整个过程他对于View没有丝毫的引用，甚至在他内部连计时器都没有。


## 通过动画

动画本身就是一种渐进的过程，因此通过他来实现滑动天然就具有弹性效果，比如以下代码让一个view在100ms内左移100像素：

```
ObjectAnimator.ofFloat(testView, "translationX", 0, 100).setDuration(100).start();
```


我们可用利用动画的特性来实现一些动画不能实现的效果，我们想模仿scroller来实现View的弹性滑动，那么利用动画的特性我们可用这样做：

```
        final int startX = 0;
        final  int deltaX =100;
        final ValueAnimator animator = ValueAnimator.ofInt(0,1).setDuration(1000);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                float fraction = animator.getAnimatedFraction();
                testView.scrollTo(startX + (int)(deltaX * fraction),0);
            }
        });
```

我们的动画本质上没有作用于任何对象上，它只是在1000ms内完成了整个动画过程。利用这个特性，可以在动画的每一帧到来时获取动画完成的比例，
再根据这个比例计算出当前 View 所要滑动的距离。这里的滑动针对的是 View 的内容而非 View 本身。这个方法的思想其实和 Scroller比较类似，
都是通过改变百分比配合 scrolITo 方法来完成View的滑动。需要说明一点，采用这种方法除了能够完成弹性滑动以外，还可以实现其他动画效果，
我们完全可以在 onAnimationUpdate 方法中加上我们想要的其他操作。



## 使用延时策略

另外一种实现弹性滑动的方法，那就是延时策略。它的核心思想是通过发送一系列延时消息从而达到一种渐近式的效果，具体来说可以使用 Handle r或 View 的 postDelayed方法。
也可以使用线程的sleep方法。对于postDelayed方法来说，我们可以通过它来延时发送一个消息，然后在消息中来进行View的滑动，如果接连不断地发送这种延时消息，
那么就可以实现弹性滑动的效果。对于sleep方法来说，通过在while循环中不断的滑动View和sleep，就可以实现弹性滑动的效果。


```
    private static final  int MESSAGE_SCROLL_TO = 1;
    private static final  int FRAME_COUNT = 30;
    private static final  int DELAYED_TIME = 33;

    private int count = 1;

    private Handler handler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case MESSAGE_SCROLL_TO:
                    count++;
                    if(count <= FRAME_COUNT){
                        float fraction = count / (float)FRAME_COUNT;
                        int scrollX = (int)(fraction * 100);
                        testButton.scrollTo(scrollX,0);
                        handler.sendEmptyMessageDelayed(MESSAGE_SCROLL_TO,DELAYED_TIME);
                    }
                    break;
            }
        }
    };

```















































