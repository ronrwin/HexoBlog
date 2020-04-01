---
title: 使用AccessibilityService自动授予权限
date: 2016-06-20 23:06:54
tags:
- Android
---
AccessibilityService是Android中的无障碍功能，可以用来监听屏幕事件。
关于AccessibilityService的使用：参考 [Building Accessibility Services](http://developer.android.com/intl/zh-cn/guide/topics/ui/accessibility/services.html)

在Android6.0（API 23）以上，会要求动态权限的授予，此时系统会弹出提示框，我们如何自动跳过授予这些权限呢，一种方式使模拟点击，即本文的方法。

<!--more-->

## 实现
在Manifest.xml中，添加权限：BIND_ACCESSIBILITY_SERVICE.，在Android 4.0以上，我们可以声明<meta-data>使用配置文件xml/serviceconfig.xml，如下：


```xml
<uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE" />

<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">

    <service
        android:name=".MyAccessibilityService"
        android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
        <intent-filter>
            <action android:name="android.accessibilityservice.AccessibilityService" />
        </intent-filter>

        <meta-data
            android:name="android.accessibilityservice"
            android:resource="@xml/serviceconfig" />

    </service>
</application>
```

xml/serviceconfig.xml的内容如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackAllMask"
    android:accessibilityFlags=""
    android:canRetrieveWindowContent="true"
    android:description="@string/description"
    android:notificationTimeout="0"
    android:packageNames="com.google.android.packageinstaller" />
```
主要是packageName，是监听的对象。这里直接定义权限弹框的控制程序。监听事件EventType是屏幕状态变换typeWindowStateChanged。即当屏幕状态变化时，并且属于com.google.android.packageinstaller控制的事件发生时，该service可以监听到该对象，并做相应的处理。

在MyAccessibilityService中实现onAccessibilityEvent()和onInterrupt()
```java
public class MyAccessibilityService extends AccessibilityService {
    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        final int eventType = event.getEventType();

        AccessibilityNodeInfo node = event.getSource();
        List<AccessibilityNodeInfo> list = node.findAccessibilityNodeInfosByText("允许");
        List<AccessibilityNodeInfo> list2 = node.findAccessibilityNodeInfosByText("ALLOW");
        List<AccessibilityNodeInfo> list3 = node.findAccessibilityNodeInfosByText("允許");
        List<AccessibilityNodeInfo> list4 = node.findAccessibilityNodeInfosByText("allow");

        for (AccessibilityNodeInfo info : list) {
            info.performAction(AccessibilityNodeInfo.ACTION_CLICK);
        }
        for (AccessibilityNodeInfo info : list2) {
            info.performAction(AccessibilityNodeInfo.ACTION_CLICK);
        }
        for (AccessibilityNodeInfo info : list3) {
            info.performAction(AccessibilityNodeInfo.ACTION_CLICK);
        }
        for (AccessibilityNodeInfo info : list4) {
            info.performAction(AccessibilityNodeInfo.ACTION_CLICK);
        }

    }

    @Override
    public void onInterrupt() {

    }

    @Override
    protected void onServiceConnected() {
        Log.d("service", "onServiceConnected");
        super.onServiceConnected();
    }
}
```
安装程序之后，打开手机“设置”----“无障碍”
![](/images/accessibility_service/accessibility_service2.png)
把NTest的开关打开：
![](/images/accessibility_service/accessibility_service1.png)
这样，程序就可以监听到权限弹框，从中获取到“允许”按钮，对其执行点击操作，完成允许操作。


> 当然监听屏幕事件变化，模拟点击等操作还能应用在其他场景，熟练掌握会十分有用。
