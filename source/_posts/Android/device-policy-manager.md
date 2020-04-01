---
title: DevicePolicyManager防卸载
date: 2016-06-15 21:25:16
tags:
- Android
---
## 设备管理器？
Android 2.2 介绍了通过提供设备管理器来支持企业级应用的开发。设备管理器API提供了系统级别的一些功能，允许企业使用一些安全敏感的应用来开发更多内部的应用。
官方介绍：https://developer.android.com/guide/topics/admin/device-admin.html#overview

## 示例
首先，我们需要在配置文件中声明一个DeviceAdminReceiver的继承类，这里命名为MyDeviceReceiver。
```java
<!-- 把receiver去掉覆盖安装即可正常删除apk -->
<receiver
    android:name=".MyDeviceReceiver"
    android:label="@string/app_name"
    android:permission="android.permission.BIND_DEVICE_ADMIN">  <!-- 需要声明权限 -->
 
    <meta-data
        android:name="android.app.device_admin"
        android:resource="@xml/device_admin_sample" />
 
    <intent-filter>
        <!--此处必须设定该Action，不设定则无法启动设备管理器-->
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
    </intent-filter>
 
</receiver>
```
<!--more-->
+ 需要声明android.permission.BIND_DEVICE_ADMIN权限，确保系统可以与Receiver正常交互。
+ android.app.action.DEVICE_ADMIN_ENABLED是Receiver必须设定的。否则无法提示用户打开设备管理器开关。可以在代码中通过监听onEnabled()获取用户的操作。
+ android:resource="@xml/device_admin_sample"声明需要使用到的属性。可以从DeviceAdminInfo中获取相关的信息。

我们看下xml/device_admin_sample.xml里面的声明（这些声明可以按需选择，也可以不要）：
```java
<device-admin xmlns:android="http://schemas.android.com/apk/res/android">
  <uses-policies>
    <limit-password />
    <watch-login />
    <reset-password />
    <force-lock />
    <wipe-data />
    <expire-password />
    <encrypted-storage />
    <disable-camera />
  </uses-policies>
</device-admin>
```
之后看看MyDeviceReceiver：
```java
public class MyDeviceReceiver extends DeviceAdminReceiver {
    @Override
    public CharSequence onDisableRequested(Context context, Intent intent) {
//        Intent outOfDialog = context.getPackageManager().getLaunchIntentForPackage("com.android.settings");
//        outOfDialog.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
//        context.startActivity(outOfDialog);
 
        // 取消授权时的弹框提示文字
        return "请不要关闭";
    }
 
    @Override
    public void onEnabled(Context context, Intent intent) {
        // 激活监听
        Log.d("MyDeviceReceiver", intent != null ? intent.toString() : "null intent");
        super.onEnabled(context, intent);
    }
}
```
在自定义的Receiver中可以通过监听用户操作来添加我们需要的处理逻辑。

最后，在启动页中引导用户跳转到激活页面：
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_what_the_hell);
 
    // 获取设备管理器
    DevicePolicyManager devicePolicyManager = (DevicePolicyManager) getSystemService(Context.DEVICE_POLICY_SERVICE);
    // 指定授权对象，设置为我们的Receiver
    ComponentName deviceComponentName = new ComponentName("com.uc.ronrwin.devicepolicymanager",
            "com.uc.ronrwin.devicepolicymanager.MyDeviceReceiver");
 
    // 判断是否已授权
    boolean isAdminActivie = devicePolicyManager.isAdminActive(deviceComponentName);
    if (!isAdminActivie) {
        Intent intent = new Intent(DevicePolicyManager.ACTION_ADD_DEVICE_ADMIN);
        intent.putExtra(DevicePolicyManager.EXTRA_DEVICE_ADMIN, deviceComponentName);
        intent.putExtra(DevicePolicyManager.EXTRA_ADD_EXPLANATION,
                "（自定义区域2）");
        startActivity(intent);
    }
}
```
这样我们的demo就完成了。

启动时关于属性的描述：
![](/images/DevicePolicyManager/1.png)

当激活之后，该应用将不能被操作：
![](/images/DevicePolicyManager/2.png)

如果一定要卸载，则需要到"Security"->"Device administrators"中取消激活：
![](/images/DevicePolicyManager/3.png)
![](/images/DevicePolicyManager/4.png)

如果点击“DEACTIVATE”，则会出现我们之前定义的弹框内容：
![](/images/DevicePolicyManager/5.png)
点击OK之后，该应用就可以卸载了。


## 设备策略
我们通过下面的方式获取DevicePolicyManager对象：
```java
DevicePolicyManager mDPM =
    (DevicePolicyManager)getSystemService(Context.DEVICE_POLICY_SERVICE);
```
以前一篇文章所述，我们还是原来的demo定义：
```java
// 指定授权对象，设置为我们的Receiver
ComponentName deviceComponentName = new ComponentName("com.uc.ronrwin.devicepolicymanager",
        "com.uc.ronrwin.devicepolicymanager.MyDeviceReceiver");
```

#### 设置密码
下面的代码提示用户设置密码：
```java
Intent intent = new Intent(DevicePolicyManager.ACTION_SET_NEW_PASSWORD);
mDPM.setPasswordQuality(deviceComponentName, DevicePolicyManager.PASSWORD_QUALITY_UNSPECIFIED);
startActivity(intent);
```
#### 设置密码质量
有多个选项可供选择：
+ PASSWORD_QUALITY_ALPHABETIC：密码至少包含字母(Password)
+ PASSWORD_QUALITY_ALPHANUMERIC：密码至少包含数字和字母(Password)
+ PASSWORD_QUALITY_NUMERIC：密码至少包含数字(PIN、Password)
+ PASSWORD_QUALITY_COMPLEX：密码至少包含数字、字母和特殊字符
+ PASSWORD_QUALITY_SOMETHING：至少包含一类密码(Pattern、PIN、Password)
+ PASSWORD_QUALITY_UNSPECIFIED：默认方式，所有方式可用(None、Swipe、Pattern、PIN、Password)

设置密码要求包含内容
+ setPasswordMinimumLetters()
+ setPasswordMinimumLowerCase()
+ setPasswordMinimumUpperCase()
+ setPasswordMinimumNonLetter()
+ setPasswordMinimumNumeric()
+ setPasswordMinimumSymbols()

#### 设置密码最小长度
```java
int pwLength = 8;
mDPM.setPasswordMinimumLength(deviceComponentName, pwLength);
```

#### 设置最大密码错误重试次数
```java
int maxFailedPw = 5;
mDPM.setMaximumFailedPasswordsForWipe(deviceComponentName, maxFailedPw);
```

#### 设置密码失效时间
```java
long pwExpiration;
mDPM.setPasswordExpirationTimeout(deviceComponentName, pwExpiration);
```

#### 禁止使用旧密码个数
```java
int pwHistoryLength = 5;    // 假设为5，则新密码不能与之前的5个密码相同
mDPM.setPasswordHistoryLength(deviceComponentName, pwHistoryLength);
```

#### 锁住设备
```java
long timeMs = 1000L*Long.parseLong(mTimeout.getText().toString());
mDPM.setMaximumTimeToLock(deviceComponentName, timeMs);
```
或者可以直接锁屏
```java
mDPM.lockNow();
```

#### 恢复出厂设置（慎用）
```java
mDPM.wipeData(0);
```

#### 禁止使用摄像头
```java
mDPM.setCameraDisabled(deviceComponentName, true);
```

#### 存储加密
```java
mDPM.setStorageEncryption(deviceComponentName, true);
```


## 总结
该功能一定程度上影响了安卓的安全，建议开发者慎用。
> 如果我们强制不让卸载，则需要在Receiver的onDisableRequested中作一些处理。可以参考http://2bab.me/2015/02/09/app-cannot-be-uninstalled/这篇文章的方式（强制锁屏绕过弹框操作）。




