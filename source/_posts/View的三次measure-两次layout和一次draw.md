---
title: 'View的三次measure,两次layout和一次draw'
tags:
  - View
  - ViewRoot
categories:
  - 源码
  - 视图
abbrlink: 251abaf9
date: 2016-10-30 16:37:22
---


&emsp;我在[《Android视图结构》](http://ztelur.github.io/2016/04/17/Android%E8%A7%86%E5%9B%BE%E6%9E%B6%E6%9E%84%E8%AF%A6%E8%A7%A3/)这篇文章中已经描述了`Activity`,`Window`和`View`在视图架构方面的关系。前天，我突然想到为什么在`setContentView`中能够调用`findViewById`函数？`View`那时不是还没有被加载，测量，布局和绘制啊。然后就搜索了相关的条目，发现`findViewById`只需要在`inflate`结束之后就可以。于是，我整理了Activity生命周期和View的生命周期的关系，并再次做一下总结。

 为了节约你的时间，本篇文章的主要内容为：
- Activity的生命周期和它包含的View的生命周期的关系
- Activity初始化时View为什么会三次measure，两次layout但只一次draw？
- ViewRoot的初始化过程

#### Activity的生命周期和View的生命周期
 我通过一个简单的demo来验证`Activity`生命周期方法和`View`的生命周期方法的调用先后顺序。请看如下截图
![截图](http://7xrxif.com1.z0.glb.clouddn.com/20161030-111.png)

&emsp;在`onCreate`函数中，我们通常都调用`setContentView`来设置布局文件，此时Android系统就会读取布局文件,但是视图此时并没有加载到`Window`上，并且也没有进入自己的生命周期。
 只有等到`Activity`进入resume状态时，它所拥有的`View`才会加载到`Window`上，并进行测量，布局和绘制。所以我们会发现相关函数的调用顺序是：

- onResume(Activity)
- onPostResume(Activity)
- onAttachedToWindow(View)
- onMeasure(View)
- onMeasure(View)
- onLayout(View)
- onSizeChanged(View)
- onMeasure(View)
- onLayout(View)
- onDraw(View)

 大家会发现，为什么`onMeasure`先调用了两次，然后再调用`onLayout`函数，最后还有在依次调用`onMeasure`,`onLayout`和`onDraw`函数呢？

#### ViewGroup的measure
 大家应该都知道，有些`ViewGroup`可能会让自己的子视图测量两次。比如说，父视图先让每个子视图自己测量，使用`View.MeasureSpec.UNSPECIFIED`，然后在根据每个子视图希望得到的大小不超过父视图的一些限制，就让子视图得到自己希望的大小，否则就用其他尺寸来重新测量子视图。这一类的视图有`FrameLayout`,`RelativeLayout`等。
 在《Android视图结构》中，我们已经知道Android视图树的根节点是`DecorView`,而它是`FrameLayout`的子类，所以就会让其子视图绘制两次，所以`onMeasure`函数会先被调用两次。

``` java
// FrameLayout的onMeasure函数，DecorView的onMeasure会调用这个函数。
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    .....
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            ......
        }
    }
    ........
    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            ........
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```
&emsp;你以为到了这里就能解释通View初始化时的三次measure,两次layout却只一次draw吗？那你就太天真了！我们都知道，视图结构中不仅仅是`DecorView`是`FrameLayout`,还有其他的需要两次measure子视图的`ViewGroup`,如果每次都导致子视图两次measure,那效率就太低了。所以`View`的`measure`函数中有相关的机制来防止这种情况。

``` java 

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
  ......
  // 当FLAG_FORCE_LAYOUT位为１时，就是当前视图请求一次布局操作
  //或者当前当前widthSpec和heightSpec不等于上次调用时传入的参数的时候
  //才进行从新绘制。
    if (forceLayout || !matchingSize &&
            (widthMeasureSpec != mOldWidthMeasureSpec ||
                    heightMeasureSpec != mOldHeightMeasureSpec)) {
            ......
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            ......
    }
    ......
}
```
&emsp;源码看到这里，我几乎失望的眼泪掉下来！没办法，只能再想其他的方法来分析这个问题。
#### 断点调试大法好
&emsp;为了分析函数调用的层级关系，我想到了断点调试法。于是，我果断在`onMeasure`和`onLayout`函数中设置了断点，然后进行调试。



![函数调用帧](http://7xrxif.com1.z0.glb.clouddn.com/20161030-blog-measure1.png)

&emsp;在《Android视图架构》一文中，我们知道`ViewRoot`是`DecorView`的父视图，虽然它自己并不是一个`View`，但是它实现了`ViewParent`的接口，Android正是通过它来实现整个视图系统的初始化工作。而它的`performTraversals`函数就是视图初始化的关键函数。
&emsp;对比两次`onMeasure`被调用时的函数调用帧，我们可以轻易发现`ViewRootImpl`的`performTraversals`函数中直接和间接的调用了两次`performMeasure`函数，从而导致了`View`最开始的两次measure过程。
&emsp;然后在相同的`performTraversals`函数中会调用`performLayout`函数，从而导致`View`进行一轮layout过程。
&emsp;但是为什么这次`performTraversals`并没有触发`View`的draw过程呢?反而是`View`又将重新进行一轮measure,layout过程之后才进行draw。
#### 两次performTraversals
&emsp;通过断点调试，我们发现在`View`初始化的过程中，系统调用了两次`performTraversals`函数，第一次`performTraversals`函数导致了View的前两次的`onMeasure`函数调用和第一次的`onLayout`函数调用。后一次的`performTraversals`函数导致了最后的`onMeasure`,`onLayout`,和`onDraw`函数的调用。但是，第二次`performTraversals`为什么会被触发呢？我们研究一下其源码就可知道。



``` java 

private void performTraversals() {
    ......
    boolean newSurface = false;
    //TODO:决定是否让newSurface为true,导致后边是否让performDraw无法被调用，而是重新scheduleTraversals
    if (!hadSurface) {
        if (mSurface.isValid()) {
            // If we are creating a new surface, then we need to
            // completely redraw it.  Also, when we get to the
            // point of drawing it we will hold off and schedule
            // a new traversal instead.  This is so we can tell the
            // window manager about all of the windows being displayed
            // before actually drawing them, so it can display then
            // all at once.
            newSurface = true;
                    .....
        }
    }
            ......
    if (!cancelDraw && !newSurface) {
        if (!skipDraw || mReportNextDraw) {
            ......
            performDraw();
        }
    } else {
        if (viewVisibility == View.VISIBLE) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }
    ......
}
```
&emsp;由源代码可以看出，当`newSurface`为真时，`performTraversals`函数并不会调用`performDraw`函数，而是调用`scheduleTraversals`函数，从而再次调用一次`performTraversals`函数，从而再次进行一次测量，布局和绘制过程。
&emsp;我们由断点调试可以轻易看到，第一次`performTraversals`时的`newSurface`为真，而第二次时是假。


![断点调试截图](http://7xrxif.com1.z0.glb.clouddn.com/20161030-%E7%AC%AC%E4%B8%80%E6%AC%A1.png)

#### 总结
&emsp;虽然我已经通过源码得知View初始化时measure三次，layout两次，draw一次的原因，但是Android系统设计时，为什么要将整个初始化过程设计成这样？我却还没有明白，为什么当`Surface`为新的时候，要推迟绘制，重新进行一轮初始化，这些可能都要涉及到`Surface`的相关内容，我之后要继续学习相关内容！

