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


关于事件传递的机制，这里给出一些结论，根据这些结论可以更好地理解整个传递机制，如下所示：

1. 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏慕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最后以up结束

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


11. onclick 会发生的前提实际当前的View是可点击的，并且他收到了down和up的事件

12. 事件传递过程是由外到内的，理解就是事件总是先传递给父元素，然后再由父元素分发给子View，通过 requestDisallowInterptTouchEvent 方法可以再子元素中干预元素的事件分发过程，但是ACTION_DOWN除外
























