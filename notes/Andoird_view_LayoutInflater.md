## Android LayoutInflater

LayoutInflater 主要是用于加载布局的。在Activity onCreate中调用的 setContentView() 其实也是用 LayoutInflater 来加载布局的。

先来看一下LayoutInflater的基本用法吧，它的用法非常简单，首先需要获取到 LayoutInflater 的实例，有两种方法可以获取到，第一种写法如下：

```
LayoutInflater layoutInflater = LayoutInflater.from(context);

```
当然，还有另外一种写法也可以完成同样的效果：

```
LayoutInflater layoutInflater = (LayoutInflater) context
		.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

其实第一种就是第二种的简单写法，只是Android给我们做了一下封装而已。得到了LayoutInflater的实例之后就可以调用它的inflate()方法来加载布局了，如下所示：

```
layoutInflater.inflate(resourceId, root);

```

inflate()方法一般接收两个参数，第一个参数就是要加载的布局id，第二个参数是指给该布局的外部再嵌套一层父布局，如果不需要就直接传null。
这样就成功成功创建了一个布局的实例，之后再将它添加到指定的位置就可以显示出来了。

不管使用的哪个inflate()方法的重载，最终都会辗转调用到LayoutInflater的如下代码中：

```
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml 用于根据节点名来创建View对象的
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context. 
                    // rInflate()方法来循环遍历这个根布局下的子元素
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }

```
上面代码那么多，其实 LayoutInflater 主要就是使用Android提供的pull解析方式来解析布局文件的,最终会把整个布局文件都解析完成后就形成了一个完整的DOM结构，最终会把最顶层的根布局返回。
即上面那一顿操作以后，返回一个根View（DOM树的顶部），__此时我们就可以通过跟View的findViewById从DOM中找到里面的相应子布局(控件)__。

对于每个View的layout_width和layout_height是__设置该控件在View布局中的大小__。也就是说，首先View必须存在于一个布局中，之后如果将layout_width
设置成match_parent表示让View的宽度填充满布局。

而对于平时在 Activity 中指定布局文件的时候，最外层的那个布局是可以指定大小，layout_width和layout_height都是有作用的。确实，这主要是因为，
在 setContentView()方法中，Android 会自动在布局文件的最外层再嵌套一个 FrameLayout，所以 layout_width 和 layout_height 属性才会有效果。


任何一个Activity中显示的界面其实主要都由两部分组成，标题栏和内容布局。标题栏就是在很多界面顶部显示的那部分内容，比如刚刚我们的那个例子当中就有标题栏，
可以在代码中控制让它是否显示。而内容布局就是一个 FrameLayout，这个布局的id叫作content，我们调用setContentView()方法时所传入的布局其实就是放到这个 FrameLayout 中的，
这也是为什么这个方法名叫作 setContentView()，而不是叫 setView()。


附上一张Activity窗口的组成图：

![](https://github.com/xianfeng92/android-code-read/blob/master/images/activity_window.png)



所以 LayoutInflater 主要作用是加载和解析我们定义的布局文件，并且能够返回一个DOM树的顶部（最顶层的根View），然后当我们要引用布局文件中的view时，就是通过DOM
树这个结构去查找和获取的。



# 参考

[Android LayoutInflater原理分析](https://blog.csdn.net/guolin_blog/article/details/12921889)

