---
title: View的绘制流程总结
date: 2018-12-12 21:11:15
tags: View绘制
categories: [Android,进阶]
---

## 一、内容概览
本文是《Android开发艺术探索》第四章的读书笔记，重在总结，细节问题读者可自行查阅。

View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure，layout，draw三个过程。其中measure用来测量View的宽和高，layout确定View在父容器中的位置，而draw则负责将View绘制到屏幕上。

## 二、理解MeasureSpec

为了更好的理解View的measure过程，我们还需要理解MeasureSpec。 
 
MeasureSpec是一个specSize和specMode信息的32位int值，其中高两位表示specMode，低30位表示specSize。specMode指测量模式，specSize指在某种测量模式下的规格大小。specMode包括： 
 
* UNSPECIFIED
* EXACTLY
* AT_MOST

<!--more-->

View需要MeasureSpec信息，来确定自己能显示多大。顶层View的MeasureSpec由ViewRootImpl#getRootMeasureSpec()方法获得，getRootMeasureSpec()源码如下：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
}

```

子View的MeasureSpec，由父容器和其自身的LayoutParams共同决定，通过ViewGroup提供的getChildMeasureSpec()方法获得，getChildMeasureSpec()源码此处就不贴了。 

这里简单说一下：

* 当View采用固定宽高的时候，不管父容器的MeasureSpec是什么，View的MeasureSpec都是EXACTLY，并且其大小遵循Layoutparams重点额大小。
* 当View的宽高是match_parent的时候，父容器的MeasureSpec是EXACTLY的话，那么View的MeasureSpec也是EXACTLY并且大小就是父容器的剩余空间；父容器的MeasureSpec是AT_MOST的话，View的MeasureSpec也是AT_MOST并且大小不会超过父容器的剩余空间；
* 当View的宽高是wrap_content的时候，不管父容器的MeasureSpec是EXACTLY还是AT_MOST，View的MeasureSpec都是AT_MOST并且大小不会超过父容器的剩余大小。

## 三、measure过程

### View的measure过程

父容器调用子View的measure方法把上一步获得的MeasureSpec信息传递过去，子View的measure方法调用View#onMeasure()，onMeasure调用setMeasuredDimension()设置自身大小。View的onMeasure()方法如下：

```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

```

setMeasuredDimension方法会设置View的宽高，getSuggestedMinimumHeight(),getSuggestedMinimumWidth()源码也很简单，如下：

```java
protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
    }
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

我们主要看getDefaultSize()方法：

```java
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

由前面的分析可知，当View的宽高属性为wrap_content时，其父View通过getChildMeasureSpec方法确定的其测量模式为AT_MOST。再由getDefaultSize()可知，其最终的宽高会被设置为specSize，即父View所剩空间的大小，这也就是为什么自定义View不对AT_MOST模式做处理，其wrap_content和match_parent效果一样。

### ViewGroup的Measure过程

ViewGroup的measure()和onMeasure()方法是从View继承过来的，没有做任何重写（measure方法是final限定，不能重写）。其onMeasure()方法由各个子类各自重写，实现自己的需求。ViewGroup的子类在onMeasure()中做的事其实都差不多:

1. 遍历子View
2. 调用measureChild*让子View确定自己的大小
3. 根据所有子View的大小确定自己的大小

2步骤中*是个通配符，意思是measureChild一类的方法，如measureChildHorizontal，measureChildWithMargins。这些方法内部会调用getChildMeasureSpec确定子View的测量模式，会调用child.measure(childWidthMeasureSpec, childHeightMeasureSpec)触发子View的测量。

## 四、layout过程

View源码中，layout方法中会先调用setFrame给自身的left，top，right，bottom属性赋值,至此自己在父View中的位置就确定了。然后会调用onLayout方法，该方法在View中是一个空实现，具体的实现由其子View（一般指ViewGroup）实现，如LinearLayout。


## 五、draw过程

draw的过程相对简单，直接看源码：

```java
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }
        ...
	}
```

可以看到一般情况下View的draw流程分为四步： 

1. 绘制背景 drawBackground(canvas)
2. 绘制自己 (onDraw)
3. 绘制Children (dispatchDraw)
4. 绘制装饰 (onDrawForeground)

其中步骤二在自定义View的时候经常需要去实现，以绘制自己想要的效果。步骤三dispatchDraw在View中是个空实现。ViewGroup实现了dispatchDraw(),其中调用了ViewGroup#drawChild方法，而drawChild()仅仅是调用了child.draw():

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
}

```

## 六、最后

感谢Android开发艺术探索的作者









