# 自定义View 

## 自定义View的分类


1. 继承View重写onDraw方法

   这种方法主要用于实现一些不规则的效果，即这种效果不方便通过布局组合来达到，往往需要静态或者动态地显示一些不规则的图形。
采用这个方式需要自身支持 warp_content，并且 pading 也要自己处理，比较考验功底。


2. 继承ViewGroup派生出来的Layout

   这种方法主要用于实现自定义布局。采用这种方式稍微复杂一些，需要合适的处理ViewGroup的测量和布局过程，并处理子元素的测量和布局过程。


3. 继承特定的 View

   比如TextView，就是重写原生的View，比如你想让 TextView 默认有颜色之类的，有一些小改动，这个就可以用它的，它相对来说比较简单。
   这个就不需要自己支持 wrap_content 和 pading 了


4. 继承特定的 ViewGroup

   这个和上述一样，只不过是重写容器而已，这个也比较常见，事件分发的时候用的也多。


## 自定义View的须知


1. 让 View 支持 warp_content

   直接继承 View 或者 ViewGroup 的控件，如果不在 onMeasure 中对 wrap_content 做特殊处理，那么当外界在布局中使用 wrap_content 时
   就无法达到预期效果。

   
2. 如果有必要，让你的 View支持 padding

   这是因为直接继承View的控件，如果不在draw方法中处理padding，那么padding属性是无法起作用的。另外，直接继承自 ViewGroup 的控件需要考虑在

   onMeasure 和 onLayout 中考虑 padding 和子元素的 margin 对其造成的影响，不然将会导致 padding 和子元素的 margin 失效。


3. 尽量不要在 View 中使用Handler

   为什么不能用，是因为没有必要，View本身就有一系列的post方法，当然，你想用也没人拦着你，我倒是觉得handler写起来代码简洁很多


4. View中如果有线程或者动画，需要及时停止

   这个问题那就更好理解了，你要是不停止这个线程或者动画，容易导致内存溢出的，所以你要在一个合适的机会销毁这些资源。在View中，当View被remove的时候，
   onDetachedFromWindow会被调用，和此方法对应的是onAttachedToWindow


5. View带有滑动嵌套时，需要处理好滑动冲突


## 自定义View的实例


### 继承View重写onDraw方法 


```
public class CycleView extends View {

    private int mColor = Color.RED;
    private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);

    public CycleView(Context context) {
        super(context);
        init();
    }

    public CycleView(Context context,  AttributeSet attrs) {
        super(context, attrs);
        TypedArray a = context.obtainStyledAttributes(attrs,R.styleable.CycleView);
        mColor = a.getColor(R.styleable.CycleView_circle_color,Color.BLACK);
        a.recycle();
        init();
    }

    public CycleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init(){
        mPaint.setColor(mColor);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(200,200);
        }else if (widthMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(200,heightSize);
        }else if (heightMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(widthSize,200);
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        final int paddingLeft = getPaddingLeft();
        final int paddingRight = getPaddingRight();
        final int paddingTop = getPaddingTop();
        final int paddingBottom = getPaddingBottom();

        int width = getWidth() - paddingLeft - paddingRight;
        int height = getHeight() - paddingBottom - paddingTop;
        int radius = Math.min(width,height) / 2;
        canvas.drawCircle(width/2+paddingLeft,height/2+paddingTop,radius,mPaint);
    }
}

```


```
   <!--name为声明的"属性集合"名，可以随便取，但是最好是设置为跟我们的View一样的名称-->
    <declare-styleable name="CycleView">
        <!--声明我们的属性，名称为default_size,取值类型为尺寸类型（dp,px等）-->
        <attr name="circle_color" format="color">
        </attr>
    </declare-styleable>

```

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical"
    android:id="@+id/linear_layout"
    tools:context=".MainActivity">

    <com.example.zhongxianfeng.demo_view.CycleView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="10dp"
        android:padding="10dp"
        app:circle_color="@color/colorPrimaryDark"/>

</LinearLayout>
```


补充： 

1. 自定义View，layout_margin 属性是由父容器控制的，因此 CycleView 中不需要处理

2. padding 为View的内边距，CycleView 中需要处理，处理的时候考虑到 View 四周的空白即可，其中圆心和半径都会考虑到 View 四周
   的 padding，从而做出相应的调整。

3. 自定义属性，我们可以在values文件夹下创建自定义属性xml，然后在 View的构造函数中解析自定义属性并做相应处理。
   为了使用自定义属性，必须在布局文件中添加 schemas声明： xmlns:app="http://schemas.android.com/apk/res-auto"


### 继承ViewGroup派生出来的Layout 


## 自定义View的思想

首先要掌握基本功，比如 view 的弹性滑动，滑动冲突，绘制原理等，这些都是自定义 View 所必须的，尤其是那些看起来很炫酷的自定义 View，它
往往对这些技术点要求更高；熟练掌握基本功以后，在面对新的自定义 View 时，要能够对其分类并选择合适的思路。另外，平时还需要多积累
一些自定义 View 的相关经验，并做到融会贯通，通过这种思想慢慢地就可以提高自定义 View 的水平。




























