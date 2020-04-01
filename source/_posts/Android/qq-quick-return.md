---
title: 仿QQ手机锁屏回复功能
date: 2016-06-20 22:53:30
tags:
- Android
---
在现在版本手Q的功能当中，可以提供在锁屏时直接回复的功能。如图所示
![](/images/qq_quickreturn/qq_quickreturn1.png)
QQ实现了在收到消息的时候，自动把屏幕置亮，并提供快捷回复的操作。
那么，我们就研究下该功能是如何实现的。

<!--more-->
## 实现方案
要产生这种场景，其原理就是在屏幕显示一个弹窗样式的Activity，同时置亮屏幕。当关闭时，自动跳过解锁步骤。
假设我们要显示DialogActivity，把目标Activity设为`__Wallpaper__`样式（确保锁屏时能置顶显示）：
```xml
<activity android:name=".DialogActivity"
    android:theme="@android:style/Theme.Wallpaper.NoTitleBar" />
```
在弹框的时机，使用Intent跳转页面，需要添加`Intent.FLAG_ACTIVITY_NEW_TASK`标记
```java
Intent i = new Intent(mContext, DialogActivity.class);
i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(i);
```
在DialogActivity启动的时候，需要对`Window`添加标记如下：
```java
super.onCreate(savedInstanceState);
getWindow().addFlags(WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
        | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON
        | WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);
setContentView(R.layout.activity_dialog);
```
我们来看下这三个标记的作用：
```java
/** Window flag: special flag to let windows be shown when the screen
 * is locked. This will let application windows take precedence over
 * key guard or any other lock screens. Can be used with
 * {@link #FLAG_KEEP_SCREEN_ON} to turn screen on and display windows
 * directly before showing the key guard window.  Can be used with
 * {@link #FLAG_DISMISS_KEYGUARD} to automatically fully dismisss
 * non-secure keyguards.  This flag only applies to the top-most
 * full-screen window.
 */
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;         // Added in API level 5
  
/** Window flag: when set as a window is being added or made
 * visible, once the window has been shown then the system will
 * poke the power manager's user activity (as if the user had woken
 * up the device) to turn the screen on. */
public static final int FLAG_TURN_SCREEN_ON = 0x00200000;           // Added in API level 5
 
/** Window flag: when set the window will cause the keyguard to
 * be dismissed, only if it is not a secure lock keyguard.  Because such
 * a keyguard is not needed for security, it will never re-appear if
 * the user navigates to another window (in contrast to
 * {@link #FLAG_SHOW_WHEN_LOCKED}, which will only temporarily
 * hide both secure and non-secure keyguards but ensure they reappear
 * when the user moves to another UI that doesn't hide them).
 * If the keyguard is currently active and is secure (requires an
 * unlock pattern) than the user will still need to confirm it before
 * seeing this window, unless {@link #FLAG_SHOW_WHEN_LOCKED} has
 * also been set.
 */
public static final int FLAG_DISMISS_KEYGUARD = 0x00400000;         // Added in API Level 5
```
我们可以看到，这些属性在 `API 5` 的时候就已经添加进去了。
当锁屏的级别不是保护性（不需要密码）时，FLAG_DISMISS_KEYGUARD 标记可以自动跳过解锁屏操作。该方式__不需要添加任何权限__。
------
另外，还有另一种方式跳过锁屏。
```java
KeyguardManager keyguardManager = (KeyguardManager) getSystemService(KEYGUARD_SERVICE);
KeyguardManager.KeyguardLock keyguardLock = keyguardManager.newKeyguardLock("");
keyguardLock.disableKeyguard();
```
并且需要添加权限：
```xml
<uses-permission android:name="android.permission.DISABLE_KEYGUARD"/>
```


## Demo实现：（minSdkVersion : 5）
Manifest配置文件：
```xml
<activity android:name=".DialogActivity"
    android:theme="@android:style/Theme.Wallpaper.NoTitleBar" />
```

MainActivity.java
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    mContext = this;
 
    // 延时操作。10秒后弹框
    new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            Intent i = new Intent(mContext, DialogActivity.class);
            i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(i);
            return false;
        }
    }).sendEmptyMessageDelayed(0, 10000);
}
```

DialogActivity.java
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
            | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON
            | WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);
    setContentView(R.layout.activity_dialog);
    mText = (TextView) findViewById(R.id.test);
 
    String devicename = android.os.Build.MODEL;
    String brand = android.os.Build.BRAND;
    int version = Build.VERSION.SDK_INT;
 
    String logText = "devicename: " + devicename + "; api levvel: " + version + "; brand: " + brand;
    Log.d("test", logText);
    mText.setText(logText);
 
    mText.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            finish();
        }
    });
}
```

demo执行后可关闭屏幕，10秒后便会弹出目标界面。
![](/images/qq_quickreturn/qq_quickreturn2.png)