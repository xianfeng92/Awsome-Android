# layout

Layout的作用是 ViewGroup 用来确定子元素的位置，__当 ViewGroup 的位置被确认之后，他的 layout 就会去遍历所有子元素并且调用 onLayout 方法__
，在layout方法中 onLayou 又被调用。__layout 方法确定了 View 本身的位置，而 onLayout 方法则会确定所有子元素的位置。__

先看 View 的layout方法，如下所示：

```
 public void layout(int l, int t, int r, int b) {
        // 成员变量mPrivateFlags3中的一些比特位存储着和layout相关的信息
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
             // 如果在mPrivateFlags3的低位字节的第4位（从最右向左数第4位）的值为1，
            // 那么就表示在layout布局前需要先对View进行测量，
            // 这种情况下就会执行View的onMeasure方法对View进行测量
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
           // 移除掉标签 PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        // 如果isLayoutModeOptical()返回true，那么就会执行setOpticalFrame()方法，
        // 否则会执行setFrame()方法。并且setOpticalFrame()内部会调用setFrame()，
        // 所以无论如何都会执行setFrame()方法。
        // setFrame()方法会将View新的 left、top、right、bottom 存储到View的成员变量中
        // 并且返回一个boolean值，如果返回true表示View的位置或尺寸发生了变化，
        // 否则表示未发生变化
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
             // 如果View的布局发生了变化，或者mPrivateFlags有需要LAYOUT的标签PFLAG_LAYOUT_REQUIRED，那么就会执行以下代码
            // 首先会触发onLayout方法的执行，View中默认的onLayout方法是个空方法
            // 不过继承自ViewGroup的类都需要实现onLayout方法，从而在 onLayout 方法中依次循环子View，
            // 并调用子View的layout方法
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }
            // 在执行完onLayout方法之后，从mPrivateFlags中移除标签PFLAG_LAYOUT_REQUIRED
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            //我们可以通过View的addOnLayoutChangeListener(View.OnLayoutChangeListener listener)方法
            //向View中添加多个Layout发生变化的事件监听器
            //这些事件监听器都存储在mListenerInfo.mOnLayoutChangeListeners这个ArrayList中
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                //首先对mOnLayoutChangeListeners中的事件监听器进行拷贝
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    // 遍历注册的事件监听器，依次调用其onLayoutChange方法，这样Layout事件监听器就得到了响应
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        final boolean wasLayoutValid = isLayoutValid();
        // 从mPrivateFlags中移除强制Layout的标签PFLAG_FORCE_LAYOUT
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        // 向mPrivateFlags3中加入Layout完成的标签PFLAG3_IS_LAID_OUT
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        if (!wasLayoutValid && isFocused()) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            if (canTakeFocus()) {
                // We have a robust focus, so parents should no longer be wanting focus.
                clearParentsWantFocus();
            } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
                // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
                // layout. In this case, there's no guarantee that parent layouts will be evaluated
                // and thus the safest action is to clear focus here.
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                clearParentsWantFocus();
            } else if (!hasParentWantsFocus()) {
                // original requestFocus was likely on this view directly, so just clear focus
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
            // otherwise, we let parents handle re-assigning focus during their layout passes.
        } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            View focused = findFocus();
            if (focused != null) {
                // Try to restore focus as close as possible to our starting focus.
                if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                    // Give up and clear focus once we've reached the top-most parent which wants
                    // focus.
                    focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                }
            }
        }

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    }

```

### setFrame

setFrame()方法是具体用来完成给View分配尺寸以及位置工作的，在layout()方法中会调用setFrame()方法。其源码如下所示：

```
protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d(VIEW_LOG_TAG, this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
             // 将新旧left、right、top、bottom进行对比，只要不完全相对就说明View的布局发生了变化，
            // 则将changed变量设置为true
            changed = true;

            //先保存一下mPrivateFlags中的PFLAG_DRAWN标签信息
            int drawn = mPrivateFlags & PFLAG_DRAWN;

            // 分别计算View的新旧尺寸
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;

           // 比较View的新旧尺寸是否相同，如果尺寸发生了变化，那么sizeChanged的值为true
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);
            // 将新的left、top、right、bottom存储到View的成员变量中
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;

             // mRenderNode.setLeftTopRightBottom()方法会调用RenderNode中原生方法的nSetLeftTopRightBottom()方法，
            // 该方法会根据 left、top、right、bottom 更新用于渲染的显示列表
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
            // 向mPrivateFlags中增加标签PFLAG_HAS_BOUNDS，表示当前View具有了明确的边界范围
            mPrivateFlags |= PFLAG_HAS_BOUNDS;


            if (sizeChanged) {
                 // 如果View的尺寸和之前相比发生了变化，那么就执行sizeChange()方法，
                // 该方法中又会调用onSizeChanged()方法，并将View的新旧尺寸传递进去
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {

                 //有可能在调用setFrame方法之前，invalidate方法就被调用了，
                //这会导致mPrivateFlags移除了PFLAG_DRAWN标签。
                //如果当前View处于可见状态就将mPrivateFlags强制添加PFLAG_DRAWN状态位，
                //这样会确保下面的invalidate()方法会执行到其父控件级别。
                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(sizeChanged);
                // invalidateParentCaches()方法会移除其父控件的PFLAG_INVALIDATED标签，
                // 这样其父控件就会重建用于渲染的显示列表
                invalidateParentCaches();
            }

            // 重新恢复mPrivateFlags中原有的PFLAG_DRAWN标签信息
            mPrivateFlags |= drawn;

            mBackgroundSizeChanged = true;
            mDefaultFocusHighlightSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
        return changed;
    }

```

layout的方法的大致流程如下：

1. 首先会通过一个 setFrame 方法来设定 View 的四个顶点的位置，即初始化 mLeft,mTop,mRight,mBottom 这四个值。

2. View的四个顶点一旦确定，那么View在父容器的位置也就确定了，接下来会调用 onLayout 方法，这个方法的用途是调用父容器确定子元素的位置，和onMeasure类似。

3. onLayout的具体位置实现同样和具体布局有关，所有 View 和 ViewGroup 均没有真正的实现onLayout方法。




View的测量宽高和最终宽高有什么区别，即 View的 getMeasureWidth 和 getWidth 这两个方法有什么区别？


在View的默认实现中，View的测量宽高和最终的是一样的，只不过一个是measure过程，一个是layout过程。而最终形成的是layout过程，即两者的赋值时机不同，测量宽高的赋值时机，
稍微早一些。因此，在日常开发中，我们可用认为他们是相等的


但是还是有些不相同的，我们可用重写View的layout方法：


```
  public void layout(int l,int t,int r, int b){
        super.layout(l,t,t+100,b+100);
    }

```

上述代码会导致在任何平台下 View 的最终宽高总是比测量大于100px,虽然这样这样会导致View显示不正常和没什么意义，但是这证明了测量不等于最终。
另一种情况是在某种情况下，View 需要多次 measure 才能确定自己的测量宽高，在前几次的测量过程中，其得出的测量宽高可能和最终宽高是不一致的。



# 参考

[源码解析Android中View的layout布局过程](https://blog.csdn.net/iispring/article/details/50366021)
[Android艺术开发探索第四章——View的工作原理]
