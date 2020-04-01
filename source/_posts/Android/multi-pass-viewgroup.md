---
title: Multi-pass Viewgroup性能表现
date: 2016-04-27 10:18:17
description: 多通道ViewGroup的表现研究
tags: 
- Android
---
原文链接：[Multi-pass ViewGroup and performance](http://periplanisi.com/android/2013/11/multi-pass-viewgroup-and-performance/)

2009年Romain Guy写了关于使用**_RelativeLayout_**替代两个**_LinearLayout_**的好处的[文章](http://www.curious-creature.com/2009/02/22/android-layout-tricks-1/)，不幸的是，很多人阅读那篇文章以后便假设RelativeLayout对比LinearLeyout来说，在所有的场景下会更好用。这是完全**错误**的假设。

那篇文章写得很好，讲解了应该减少视图层级和尽量少使用对象。当然第一原则是使视图尽量**扁平化**，避免不必要的**容器**，使用**merge**标签等。

然而，这个例子并没有使得RelativeLayout比LinearLayout要好。实际上如果你查看RelativeLayout的实现，你会发现它的孩子会调用**_onMeasure_**两次！这样会降低效率。如果我们有很复杂的页面结构，而RelativeLayout处于它的根节点，那将会是很大的性能开销。**所以最好不要把RelativeLayout放在视图树的顶层**。

当使用weight的时候，LinearLayout的子控件也会调用两次，为了计算所有控件的正确的大小。

在性能考虑上，还需要注意ListView的情况。由于多通道的计算和行控件的创建，ListView的布局如果使用wrap_content也会降低性能。

我们举个简单例子。

我们有下面这样的xml布局：
使用RelativeLayout作为父容器。我们将会观察RelativeLayout的性能。并且ListView设为wrap_content。

```xml
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <ListView
        android:id="@+id/mylist"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</RelativeLayout>
```

然后把item的布局如下：

```xml
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    >
    
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/some_text"
        />
    
</RelativeLayout>
```

我们运行一下。假设有20行，那么TextView就会调用**320**次onMeasure！！！

这里我们有3点可以提升的地方，减少一半的计算次数。
- 把RelativeLayout转换成LinearLayout
- 把item的wrap_content改为match_parent
- 把每一行的item容器由RelativeLayout转为LinearLayout

我们把次数降低到了40次（减少了8倍），加载时间加快了100ms，在复杂的布局上，提升会更显著。



