## View的事件分发

### 点击事件的传递规则

当一个 MotionnEvent 产生了以后，系统需要把这个事件传递给一个具体的 View，而这个传递的过程就是分发过程。

点击事件的分发过程由三个很重要的方法来完成:

#### dispatchTouchEvent

用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的 onTouchEvent 和下级View的 dispatchTouchEvent 方法的影响，
表示是否消耗当前事件。


#### onInterceptTouchEvent

dispatchTouchEvent方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，
返回结果表示是否拦截当前事件。


#### onTouchEvent

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件。如果不消耗，则在同一个事件序列中，当前 View 无法再次接收到事件。

上述三个方法到底有什么区别呢?其实它们的关系可以用如下伪代码表示：

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean consume = false;
        if(onInterceptTouchEvent(ev)){
            consume = onTouchEvent(ev);
        }else {
            consume = child.dispatchTouchEvent(ev);
        }
        return consume;

    }
```
对于一个根 ViewGroup 来说，点击事件产生以后，首先传递给它，这时它的 dispatchTouchEvent 就会被调用。如果这个 ViewGroup 的 onIntereptTouchEvent 
方法返回 true 就表示它要控截当前事件，接着事件就会交给这个 ViewGroup 处理，此时它的 onTouchEvent 方法就会被调用。

如果这个 ViewGroup 的 onIntereptTouchEvent 方法返回 false 就表示不需要拦截当前事件，这时当前事件就会继续传递给它的子元素，
接着子元素的 onIntereptTouchEvent 方法就会被调用，如此反复直到事件被最终处理。

当一个View需要处理事件时，如果它设置了 OnTouchListener，那么 OnTouchListener 中的 onTouch 方法会被回调。这时事件如何处理还要看 onTouch 的返回值，
如果返回false,那当前的 View 的方法 OnTouchListener 会被调用。如果返回true，那么 onTouchEvent 方法将不会被调用。由此可见，给View设置的 OnTouchListener，
其优先级比 onTouchEvent 要高，在 onTouchEvent 方法中，如果当前设置的有 OnClickListener，那么它的 onClick 方法会被调用。可以看出，
平时我们常用的 OnClickListener，其优先级最低，即处于事件传递的尾端。

当一个点击事件产生后，它的传递过程遵循如下顺序：Activity>Window-View，即事件总是先传递给Activity,Activity再传递给Window，最后Window再传递给顶级View。
顶级View接收到事件后，就会按照事件分发机制去分发事件。


考虑一种情况，如果一个view的 onTouchEvent 返回false，那么它的父容器的 onTouchEvent 将会被调用，依此类推,如果所有的元素都不处理这个事件，那么这个事件将会最终传递给 Activity 处理，
即 Activity 的 onTouchEvent 方法会被调用。这个过程其实也很好理解，我们可以换一种思路，假如点击事件是一个难题，这个难题最终被上级领导分给了一个程序员去处理（这是事件分发过程），
结果这个程序员搞不定(onTouchEvent返回false)，现在该怎么办呢？难题必须要解决，那只能交给水平更高的上级解决（上级的onTouchEvent被调用)，如果上级再搞不定，那只能交给上级的上级去解决，
就这样将难题一层层地向上抛，这是公司内部一种很常见的处理问题的过程。从这个角度来看，View的事件传递过程还是很贴近现实的，毕竟程序员也生活在现实中。


## 关于事件传递的机制，这里给出一些结论，根据这些结论可以更好地理解整个传递机制，如下所示：

1. 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏慕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的 move 事件，最后以 up 结束

2. 正常情况下，一个事件序列只能被一个 View 拦截且消耗

3. 因为一旦一个元素拦截了某此事件，那么同一个事件序列内的所有事件都会直接交给它处理，__因此同一个事件序列中的事件不能分别由两个View同时处理__。
   但是通过特殊手段可以做到，比如一个Vew将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。

4. 某个 View 一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话)，并且它的onInterceprTouchEvent不会再被调用。
   这条也很好理解，就是说当一个 View 决定拦截一个事件后，那么系统会把同一个事件序列内的其他事件都直接交给它来处理，因此就不用再调用这个 View 的 onInterceptTouchEvent 去询问它是否要拦截了。

5. 某个View一旦开始处理事件，如果它不消耗 ACTON_DOWN 事件(onTouchEvent返回了false)，那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，
   即父元素的 onTouchEvent 会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了，这就好比上级交给程序员一件事。
   如果这件事没有处理好，短期内上级就不敢再把事情交给这个程序员做了，二者是类似的道理。


6. 如果View不消耗除 ACTION_DOWN 以外的其他事件，那么这个点击事件会消失，此时父元素的 onTouchEvent 并不会被调用，并且当前View可以持续收到后续的事件，
   最终这些消失的点击事件会传递给Activity处理。


7. ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouchEvent方法默认返回false


8. View没有 onInterceptTouchEvent 方法，一旦有点击事件传递给它，那么它的 onTouchEvent 方法就会被调用。


9. view的 onTouchEvent 默认都会消耗事件（返回true)，除非它是不可点击的(clickable和longClickable同时为false)，View的 longClickable 属性默认都为false, clickable 属性要分情况，
   比如Button的clickable属性默认为true，而 TextView 的 clickable 属性默认为false。


10. view 的 enable 属性不影响 onTouchEvent 的默认返回值。哪怕一个View是 disable 状态的，只要它的 clickable 或者 longclickable 有一个为true，那么它的onTouchEvent就返会true。


11. onclick 会发生的前提实际当前的 View 是可点击的，并且他收到了 down 和 up 的事件

12. 事件传递过程是由外到内的，理解就是事件总是先传递给父元素，然后再由父元素分发给子 View，通过 requestDisallowInterptTouchEvent 方法可以再子元素中干预元素的事件分发过程，但 ACTION_DOWN 除外


## 事件分发的源码解析

### 事件从 Activity到 ViewGroup

点击事件用 MotionEvent 来表示，当一个点击操作发生的时候，事件最先传递给 Activity，由 Activity 的 dispatchTouchEvent 来进行事件的派发。
具体的工作是由内部的 window 来完成的，window 会将事件传递给 DecorView , DecorView 是当前界面的底层容器（setContentView 所设置的父容器），
可以通过 Activity.getWindow.getDecorView() 获得，我们可用先从 dispatchTouchEvent 的源码看起：

```
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

```

首先事件交给 Activity 所依附的 window 进行分发，如果返回 true, 整个事件循环就结束了，返回 false 意味着事件没人处理。所有 View 的 onTouchEvent 都返回false，
那么 Activity 的 onTouchEvent 就会被调用。

接下来我们看下 window 是如何将事件传递给 ViewGroup，通过源码我们知道，window时一个抽象类，而 window 的 super.dispatchTouchEvent(ev) 方法也是抽象的，
因此我们必须找到window的实现类。

```
public abstract boolean superDispatchTouchEvent(MotionEvent event);

```
那么 window 的实现类是什么呢？就是 phoneWindow，这点源码中有一段注释就说明了，意思大概就是 winodw 类控制顶级的 View 的外观和行为机制。

由于 Window 的唯一实现是 PhoneWindow，因此接下来看一下 PhoneWindow 是如何处理点击事件。

```
public boolean superDispatchTouchEvent(MotionEvent ev){
        return mDecor.superDispatchTouchEvent(ev);
    }

```

phoneWindow 传递给了 DecorView。

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker {

    private DecorView mDecor;

    @Override
    public final View getDecorView(){
        if(mDecor == null){
            installDesor():
        }0
        return  mDecor;
    }
}
```


下面是 DecorView 的superDispatchTouchEvent 源码：

```
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }

```

对于 DecorView 的 super.dispatchTouchEvent(event) 其实调用的时 ViewGroup 的 dispatchTouchEvent 方法。即 DecorView 将 event 传给了 ViewGroup，
下面就交由ViewGroup 来对event进行分发了。

小结一下： 

1. 在一个界面上，当一个点击操作发生的时候，事件最先传递给 Activity。

2. Activity 内部持有 PhoneWindow 引用，PhoneWindow 内部持有 DecorView 的引用。所以当 Activity 收到事件后，就可以将事件派发给 DecorView。

3. DecorView 是 FrameLayout 的一个子类，DecorView 接收到事件后，直接将事件传给其父类（super.dispatchTouchEvent(event)）。
   到此事件传递到了 ViewGroup 的 dispatchTouchEvent 方法中了。下面就是 ViewGroup 来操作了。



__这里的ViewGroup可以看做我们的布局文件的根布局,所以接下来，事件将从根部局开始往子布局传递__


### ViewGroup 来分发事件

先看看 ViewGroup 的 dispatchTouchEvent 的完整源码：

```
 @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // 检查是否需要拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

```

代码很长，一段一段来分析吧。


先看下面这一段源码，它描述的是 ViewGroup 是否拦截点击事情这个逻辑。

```
            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                // 子view中可以设置，一旦 disallowIntercept 为 true，那么ACTION MOVE 和 UP 时，
                // ViewGroup 的 onInterceptTouchEvent 将不会调用
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

```

从上面代码我们可以看出，ViewGroup 在如下两种情况下会判断是否要拦截当前事件：事件类型为 ACTION_DOWN 或者 mFirstTouchTarget!=null 。

其中 mFirstTouchTargetl=null 是什么意思呢？

这个从后面的代码逻辑可以看出来，__当事件由 ViewGroup 的子元素成功处理时，mFirstTouchTarget 会被赋值并指向子元素__。

1. 当 ACTION_DOWN 事件传递给 ViewGroup 时，ViewGroup 通过 onInterceptTouchEvent 拦截事件，那么此时 mFirstTouchTarget 是为null，
   所以对于后面的 ACTION_MOVE 和 ACTION_UP 事件到来时，由于 actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null
   这个条件为false，将导致 ViewGroup 的 onInterceptTouchEvent 不会再被调用。

2. 当 ACTION_DOWN 事件传递给 ViewGroup 时，ViewGroup 的 onInterceptTouchEvent 不拦截该事件，那么该事件就会交由 ViewGroup 的子元素
   来处理，当子元素成功处理该事件时，mFirstTouchTarget 会被赋值并指向子元素。此时 mFirstTouchTarget != null。那么此时 ACTION_MOVE 和 ACTION_UP 事件到来时，
   由于 mFirstTouchTarget != null这个条件为 true，导致 ViewGroup 的 onInterceptTouchEvent 会被调用。

3. 对于 FLAG_DISALLOW_INTERCEPT 标记位，这个标记位是通过 requestDisallowInterceptTouchEvent 方法来设置的，一般用于子View中。
    FLAG_DISALLOW_INTERCEPT 一旦设置后，ViewGroup 将无法拦截除了 ACTION_DOWN 以外的其他点击事件。

为什么说是除了 ACTION_DOWN 以外的其他事件呢?

这是因为 ViewGroup 在分发事件时，如果是 ACTION_DOWN 就会重置 FLAG_DISALLOW_INTERCEPT 这个标记位，将导致子View中设置的这个标记位无效。
因此，__当面对 ACTION_DOWN 事件时，ViewGroup 总是会调用自己的 onInterceptTouchEvent 方法来询问自己是否要拦截事件__。

在下面的代码中，ViewGroup 会在 ACTION_DOWN 事件到来时做重置的作用。而在 requestTouchState 方法中会对 FLAG_DISALLOW_INTERCEPT 进行重置，
因此子View调用 requestDisallowInterceptTouchEvent 方法并不能影响 ViewGroup 对 ACTION_DOWN 事件的处理：


```
             // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
```

从上面的源码分析，我们可用得出结论：

* 当 ViewGroup 决定拦截事件之后，那么后续的点击事件，将会默认交给它处理并且不再调用他的 onInterceptTouchEvent 方法

* FLAG_DISALLOW_INTERCEPT 这个标志的作用是让 ViewGroup 不在拦截事件，当然前提是 ViewGroup 不拦截 ACTION_DOWN 事件


当 ViewGroup 不拦截事件的时候，事件就会向下分发交由它的子 View （对应于布局文件中的子布局）进行处理：

```
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            // 判断 child 能否接收点击事件以及点击事件是否在 child 的坐标范围
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // dispatchTransformedTouchEvent 中其实就是调用 child（child 不为null时） 的 dispatchTouchEvent
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
```


首先遍历 ViewGroup 的所有子元素，然后判断子元素是否能接受这个点击事件。

能否接收点击事件主要是两点来衡量：

1. 子元素是否在播放动画

2. 点击事件的坐标是否落在子元素的区域内

__如果某子元素满足这两个条件，那么事件就会传递给它处理__。


即：父布局不拦截传递来的事件时，其会将事件传递给其中的子布局。父布局会根据__事件产生的坐标__以及__子布局是否在动画__来寻找合适的子布局，将事件传给它。

这里的事件传递，其实就是通过 dispatchTransformedTouchEvent 来调用的就是子元素的 dispatchTouchEvent 方法。这样事件就交由子元素处理处理，从而完成了一轮事件分发。

```
        if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
```

如果子元素的 dispatchTouchEvent 返回true，那么 mFirstTouchTarget 就会被赋值，同时跳出for循环。


```
// addTouchTarget内部完成对 mFirstTouchTarget 的赋值
newTouchTarget = addTouchTarget(child, idBitsToAssign);
alreadyDispatchedToNewTouchTarget = true;
break;
```
这几行代码就完成了 mFirstTouchTarget 的赋值并终止对子元素的遍历，如果子元素的 dispatchTouchEvent 返回 false，ViewGroup 就会把事件分给下一个子元素。

其实 mFirstTouchTarget 真正的赋值过程是在 addTouchTarget 内部完成的，从下面的 addTouchTarget 的内部结构就可以看出，mFirstTouchTarget 其实是一种单链表的结构，
mFirstTouchTarget 是否被赋值，将直接影响到 ViewGroup 对事件的拦截机制。

如果 mFirstTouchTarget 为 null，那么 ViewGroup 的默认拦截下来同一序列中所有的点击事件，其实这个时候 ViewGroup 的子 View 没有处理事件，这个时候 ViewGroup 会尝试
来处理该事件。

		```
		private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
			final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
			target.next = mFirstTouchTarget;
			mFirstTouchTarget = target;
			return target;
		    }

		```


如果遍历所有的子元素后，事件都没有被合适的处理，这包含两种情况：

1. 第一是 Viewgroup 没有子元素

2. 第二是子元素处理了该点击事件，但是在 dispatchTouchEvent 中返回 false，这一般是因为子元素在 onTouchEvent 返回了false

在这两种情况下，ViewGroup 会自己处理点击事件，看代码：

```
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } 
```


这里的第三个参数 child 为 null，从上面的分析我们可用知道，它会调用 supe.dispatchTouchEvent(event)，
很显然，这里就转到了 View 的 dispatchTouchEvent 方法，就是点击事件传递给 View 来处理了。


### View 对点击事件的处理

View 对点击事件的处理稍微有点简单（这里的 View 不包含 ViewGroup），先看它的 dispatchTouchEvent 方法：

```
 public boolean dispatchTouchEvent(MotionEvent event) {
        ...

        boolean result = false;

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        ...

        return result;
    }
```

View 点击事件的处理，因为它只是一个 View，它没有子元素所以无法向下传递，所以只能自己处理点击事件。

从上面的源码可以看出 View 对点击事件的处理过程：

1. 首选会判断你有没有设置 onTouchListener，如果 onTouchListener 中的 onTouch 为true，那么 onTouchEvent 就不会被调用。
   可见 onTouchListener 的优先级高于 onTouchEvent，这样做到好处就是方便在外界处理点击事件。即：我们如果在一个 Activity中设置了一个
   onTouchListener 之后，当 Event 传递到 View 时，是会先调用 onTouch 方法的，只有 OnTouch 方法返回 false，才会去调用 onTouchEvent。


2. 看看 onTouchEvent 的实现 

* 先看当 View 处于不可用的状态(DISABLE)下点击事件的处理过程。如下，很显然，不可用状态下的 View 照样会消耗点击事件，尽管它看起来不可用：

```
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
```

* 如果View设置有代理，那么还会执行 TouchDelegate 的 onTouchEvent 方法，这个 onTouchEvent 的工作机制看起来和 onTouchListener 类似：

```
 if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

```

* 最后 onTouchEvent 中对点击事件的具体处理，如下所示：

```
 if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
                 black;
             }
        ....
    return true;
    }
```

从上面的代码来看，只要 View 的 CLICKABLE 和 LONG_CLICKABLE 有一个为 true，那么它就会消耗这个事件，即 onTouchEvent 返回true，不管它是不是（不可用）DISABLE 状态。
然后就是__当 ACTION_UP 事件发生之后，会触发 performClick 方法__。如果 View 设置了 onClickListener，那么 performClick 方法内部就会调用它的onClick方法。


```
 public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

View 的 LONG_CLICKABLE 属性默认为 false，而 CLICKABLE 属性是否为false和具体的View有关，确切的说是可点击的 View 其 CLICKABLE 为true，不可点击的为false。
比如 button 是可点击的，textview 是不可点击的。

通过 setOnClickListener 和 setOnLongClickListener 是改变这些状态的：

```
 public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }

    public void setOnLongClickListener(@Nullable OnLongClickListener l) {
        if (!isLongClickable()) {
            setLongClickable(true);
        }
        getListenerInfo().mOnLongClickListener = l;
    }
```

整个事件分发流程的源码到此分析结束，最后看个事件分发的流程图来回味一下吧。

![事件分发的流程图]()



图分为3层，从上往下依次是 Activity、ViewGroup、View


* 事件从左上角那个白色箭头开始，由 Activity 的 dispatchTouchEvent 做分发


* 箭头的上面字代表方法返回值，（return true、return false、return super.xxxxx(), super 的意思是调用父类实现


* dispatchTouchEvent 和 onTouchEvent的框里有个【true---->消费】的字，表示的意思是如果方法返回true，那么代表事件就此消费，不会继续往别的地方传了，事件终止。



从上图可以看出：

1. 事件是由 Activity -------> ViewGroup-------->View。相应的在我们写的布局文件中，事件是由 Activity 传给根ViewGroup控件，然后由根ViewGroup控件传给其包含的子控件。

2. 根ViewGroup控件，如果拦截事件，那么其 onTouchEvent 方法会被调用。如果 onTouchEvent 返回 true，那么事件被消耗掉了，否则事件会传给 Activity，由 Activity 的 
   onTouchEvent 方法来处理

3. 如果ViewGroup控件不拦截该事件，就会交给其子控件来处理，调用子控件 onTouchEvent 来处理。补充一下：ViewGroup控件根据Event的坐标位置找到对应的子控件。
   如果子控件的 onTouchEvent 返回 true，则代表该事件被处理了，否则，该事件会转回到 ViewGroup控件，并由其 onTouchEvent 来处理，如果还是返回 false，则会继续回传给
   activity，让其 onTouchEvent 来处理该事件，到这里整个事件已经分发和处理结束。


## 参考

* Android开发艺术探索

* [图解 Android 事件分发机制](https://www.jianshu.com/p/e99b5e8bd67b)
