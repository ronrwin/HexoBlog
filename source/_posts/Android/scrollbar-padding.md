---
title: 对ScrollBar设置padding
date: 2016-07-19 20:09:25
tags:
- Android
---

开发的过程中，遇到了一个问题：如何使ScrollBar往里面缩10dp，而不影响下面内容的展示？

在官方的控件中没有可以使用的api，那只能看源码了。

主要的绘制放在View的onDrawScrollBars方法里面，主要是下面的方法调用：
```java
onDrawVerticalScrollBar(canvas, scrollBar, left, top, right, bottom);
```

然后我们发现，这个方法是protected的：
```java
/**
     * <p>Draw the vertical scrollbar if {@link #isVerticalScrollBarEnabled()}
     * returns true.</p>
     *
     * @param canvas the canvas on which to draw the scrollbar
     * @param scrollBar the scrollbar's drawable
     *
     * @see #isVerticalScrollBarEnabled()
     * @see #computeVerticalScrollRange()
     * @see #computeVerticalScrollExtent()
     * @see #computeVerticalScrollOffset()
     * @see android.widget.ScrollBarDrawable
     * @hide
     */
    protected void onDrawVerticalScrollBar(Canvas canvas, Drawable scrollBar,
            int l, int t, int r, int b) {
        scrollBar.setBounds(l, t, r, b);
        scrollBar.draw(canvas);
    }
```

那么，一个简单的方式，就是自定义一个子类继承ScrollView或ListView。重写onDrawVerticalScrollBar方法：
```java
@Override
    protected void onDrawVerticalScrollBar(Canvas canvas, Drawable scrollBar, int l, int t, int r, int b) {
        canvas.save();
        canvas.translate(-mScrollBarPaddingRight, 0);
        super.onDrawVerticalScrollBar(canvas, scrollBar, l, t, r, b);
        canvas.restore();
    }
```

然后使用这个自定义控件，那么ScrollBar就能如我们所愿缩进啦。`记得要设ScrollBarStyle为insideOverlay`。