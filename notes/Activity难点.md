
## setResult的调用时机:

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
  
  
  
  
  
  
