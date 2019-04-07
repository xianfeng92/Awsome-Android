
##  setResult 的调用时机:

1. Activity 返回 result 是在被 finish 的时候，也就是说调用 setResult() 方法必须在finish()之前.
 
2. onActivityResult 是在前一个 Activity onRestart 时被调用,所以想要在前一个 Activity 中获取当前 Activty 的Result,
   必须在前一个Activity的 onRestart 方法调用之前,已经设置了 result.

### 实际应用中,可以这么调用

1. 按 BACK 键从一个 Activity 退出来的，一按BACK，android 就会自动调用 Activity 的finish()方法.

```
    @Override
    public void onBackPressed() {
        Intent intent = getIntent();
        Bundle bundle = intent.getExtras();
        bundle.putString("msg","I am XFORG");
        intent.putExtras(bundle);
        setResult(111,intent);
        finish();
        super.onBackPressed();
    }
```

2. 在点击事件中显式的调用 finish()

 ```
  Intent intent = getIntent();
        Bundle bundle = intent.getExtras();
        bundle.putString("msg","I am XFORG");
        intent.putExtras(bundle);
        setResult(111,intent);
        finish();
 ```
 
 ps:
 
  Activity.finish() Call this when your activity is done and should be closed. 
  在你的activity动作完成的时候，或者Activity需要关闭的时候，调用此方法。
  当你调用此方法的时候，系统只是将最上面的Activity移出了栈，并没有及时的调用onDestory（）方法，
  其占用的资源也没有被及时释放。因为移出了栈，所以当你点击手机上面的“back”按键的时候，也不会再找到这个Activity。
  
## onSaveInstanceState() 和 onRestoreInstanceState()
  
  Android系统的回收机制会在未经用户主动操作的情况下销毁 activity，而为了避免系统回收 activity 导致数据丢失，Android为我们提供了onSaveInstanceState(Bundle outState)和onRestoreInstanceState(Bundle savedInstanceState)用于保存和恢复数据。
  
   ![](https://github.com/xianfeng92/android-code-read/blob/master/images/saveAndRestoreInstancState.png)
 
### onSaveInstanceState 调用时机
 
onSaveInstanceState(Bundle outState)会在以下情况被调用:

1. 当用户按下HOME键时

2. 从最近应用中选择运行其他的程序时

3. 按下电源按键（关闭屏幕显示）时

4. 从当前 activity 启动一个新的 activity 时

5. 屏幕方向切换时(无论竖屏切横屏还是横屏切竖屏都会调用)

在前4种情况下，当前activity的生命周期为：onPause -> onSaveInstanceState -> onStop。 

### onRestoreInstanceState 调用时机

   onRestoreInstanceState(Bundle savedInstanceState) 只有在 activity 确实是被系统回收，重新创建 activity 的情况下才会被调用。
   比如第5种情况屏幕方向切换时，activity生命周期如下： onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart      -> onRestoreInstanceState -> onResume 
   
   而按 HOME 键返回桌面，又马上点击应用图标回到原来页面时，activity生命周期如下: onPause -> onSaveInstanceState -> onStop -> onRestart -     > onStart -> onResume 因为activity没有被系统回收，因此 onRestoreInstanceState 没有被调用。如果 onRestoreInstanceState 被调用了，则    页面必然被回收过，则 onSaveInstanceState 必然被调用过。

### onCreate()里也有Bundle参数，可以用来恢复数据，它和 onRestoreInstanceState 有什么区别？

    因为 onSaveInstanceState 不一定会被调用，所以 onCreate() 里的 Bundle 参数可能为空，如果使用onCreate()来恢复数据，一定要做非空判断。
    而 onRestoreInstanceState 的 Bundle 参数一定不会是空值，因为它只有在上次 activity 被回收了才会调用。
   
### 保存 Activity 状态  
  
   当 Activity 开始停止时，系统会调用 onSaveInstanceState() 以便可以使用一组键值对来保存当前 Activity 的状态信息。此方法的默认实现保存有关Activity 视图层次结构状态的信息，例如: EditText 中已经输入的文本信息或 ListView 当前的滚动位置。当然,我们也可以通过添加添加键值对的方式,让 Activity 在 destroy 前保存更多的信息,具体方法如下:

```
static final String STATE_SCORE = "playerScore";
static final String STATE_LEVEL = "playerLevel";
...

@Override
public void onSaveInstanceState(Bundle savedInstanceState) {
    // 保存用户自定义的状态
    savedInstanceState.putInt(STATE_SCORE, mCurrentScore);
    savedInstanceState.putInt(STATE_LEVEL, mCurrentLevel);
    
    // 调用父类交给系统处理，这样系统能保存视图层次结构状态
    super.onSaveInstanceState(savedInstanceState);
}
```
这样在 Activity 由于异常的情况而 destroy 后, 系统会额外保存关于 mCurrentScore 和 mCurrentLevel 的信息,而当此 Activity 被重建时,我们可以在恢onCreate()进行一些状态数据的恢复:

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); // 记得总是调用父类
   
    // 检查是否正在重新创建一个以前销毁的实例
    if (savedInstanceState != null) {
        // 从已保存状态恢复成员的值
        mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
        mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
    } else {
        // 可能初始化一个新实例的默认值的成员
    }
    ...
}
```
上面的代码中 savedInstanceState != null 时,正表明当前创建的 activity 之前被异常状态而销毁过.
   
   
## 关于onConfigurationChanged方法

   当系统的配置信息发生改变时，系统会回调此方法。注意，只有在配置文件 AndroidManifest 中处理了 configChanges属性对应的设备配置，该方法才会被调用。如果发生设备配置与在配置文件中设置的不一致，则Activity会被销毁并使用新的配置重建。
   
   例如：
   
   当屏幕方向发生改变时，Activity会被销毁重建，如果在AndroidManifest文件中处理屏幕方向配置信息如下:
```
 <activity android:name=".MainActivity"
            android:configChanges="orientation|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
 ```
 
  则 Activity 不会被销毁重建，而是会回调 onConfigurationChanged 方法。如果configChanges只设置了orientation，则当其他设备配置信息改变时，     Activity依然会销毁重建，且不会调用 onConfigurationChanged。
  例如，在上面的配置的情况下，如果语言改变了，Acitivity 就会销毁重建，且不会调用 onConfigurationChanged 方法。

  注意：横竖屏切换的属性是orientation。如果targetSdkVersion的值大于等于13，则如下配置才会回调onConfigurationChanged方法:
  
 ```
  android:configChanges="orientation|screenSize"
 ```
 
 如果targetSdkVersion的值小于13，则只要配置:
 
 ```
 android:configChanges="orientation"
 ```
 
扩展：

当用户接入一个外设键盘时，默认软键盘会自动隐藏，系统自动使用外设键盘。这个过程Activity的销毁和隐藏执行了两次。并且onConfigurationChanged()周期不会调用。

但是在配置文件中设置android:configChanges="keyboardHidden|keyboard"。当接入外设键盘或者拔出外设键盘时，调用的周期是先调用onConfigurationChanged()周期后销毁重建。

在这里有一个疑点，为什么有两次的销毁重建？

其中一次的销毁重建可以肯定是因为外设键盘的插入和拔出。当设置android:configChanges="keyboardHidden|keyboard"之后，就不会销毁重建，而是调用onConfigurationChanged()方法。

但是还有一次销毁重建一直存在。

__当接入外设键盘时，除了键盘类型的改变，触摸屏也发生了变化。__ 因为使用外设键盘，触摸屏不能使用了。这里，我接入的是键盘，所以触摸屏不能使用了。
 
如果是键盘类型发生了改变，则configChanges属性配置如下Activity才不会销毁重建，且回调onConfigurationChanged方法:

```
<activity android:name=".MainActivity"
            android:configChanges="keyboard|keyboardHidden|touchscreen">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```


















  
  
