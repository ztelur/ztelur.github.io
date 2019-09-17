---
title: Android Scroll详解(一)：基础知识
tags:
  - scroll
categories:
  - Android
abbrlink: 1fc40fc1
date: 2016-03-27 18:23:06
---

&emsp;在前边的文章中，我们已经对Android触摸事件处理有了大致的了解，并且详细探讨了`MotionEvent`的相关用法。对之前文章中的知识还不是很了解的同学，请阅读[《Android MotionEvent详解》](http://ztelur.github.io/2016/03/16/Android-MotionEvent%E8%AF%A6%E8%A7%A3/)
&emsp;今天，我们就来探讨一下Android中界面滚动效果的相关机制，本篇文章主要讲解一下滚动相关的知识点，之后的文章会涉及实际的代码和原理。希望大家阅读完这篇文章之后，能够了解或者掌握一下知识:

- Android 视图的组成部分
- `mScrollX`和`mScrollY`对视图显示的影响
- `scrollTo`和`scrollBy`的使用
- `invalidate`和`postInvalidate`的区别

#### View的mScrollX和mScrollY
&emsp;我们都知道，`View`中有两个重要的成员变量，`mScrollX`,`mScrollY`.它们分别代表视图内容(view content)水平方向和竖直方向的滚动距离。我们可以通过`setScrollX`和`setScrollY`来个函数来改变它们的值，从而来滚动视图的内容。
&emsp;**在这里需要强调的是，`mScrollX`和`mScrollY`会导致视图内容(view content)变化,但是不会影响视图背景(background)。**
&emsp;看到这里同学们或许会有写疑问，视图的内容和背景有什么区别呢？视图还有哪些组成部分呢？
&emsp;我们可以从View的`draw`方法中得知View的组成部分。

``` java
// http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/View.java#View
public void draw(Canvas canvas) {
         ........
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
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
        .......

        // Step 2, save the canvas' layers
        .......
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        .......
        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
        .....
        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
        ......
    }
```
&emsp;View显示内容由一下几个部分组成:
- 背景(background)
- 本身的内容(content)
- 子视图
- 边界渐变效果(fade effect),上下左右四个边界都可能会有渐变效果，代码中只显示了上边界的渐变效果绘制。
- 边框或者装饰效果(decorations),比如滚动条

&emsp;举个例子吧，我们都知道在布局文件中，`TextView`有两个比较重要的属性:`background`,`text`。`background`可以设置TextView的背景，而`text`则是设置要绘制字体内容。
``` xml
    <TextView
        android:layout_width="wrap_content"
        android:background="@drawable/ic_launcher"
        android:text="Test"
        android:layout_height="wrap_content" />
```

&emsp;`mScrollX`和`mScrollY`对除了本身内容外的部分的绘制都有影响。只是不会影响视图背景的绘制。

#### 滚动的方向性
&emsp;我们都知道，在Android的视图中，布局相关的数值都是有方向性的，比如`mLeft`,`mTop`。


![View坐标轴.png](http://7xrxif.com1.z0.glb.clouddn.com/2016327-androidscroll-View%E5%9D%90%E6%A0%87%E8%BD%B4.png)

&emsp;由上图我们可以知道，Android视图坐标的原点在屏幕的左上方，x轴正方向是向右，y轴正方向是向下。
&emsp;所以，当你将`mLeft`和`mTop`的数值加10并且重绘视图时，视图会向右下移动。
&emsp;那么`mScrollY`和`mScrollX`也在这样一个坐标域中吗？它们的正方向和`mTop`和`mLeft`是一样的吗？是的，它们属于同一个坐标域，方向性相同。
&emsp;但是如果你将`mScrollX`和`mScrollY`的数值都增大10，然后调用`invalidate()`重新绘制界面的话，你会发现视图中的内容都向左上角移动啦！
&emsp;这是怎么回事呢？从概念上你可以先这样解：`mScrollX`和`mScrollY`改变导致View的可视区域的移动，并不是导致View的视图区域的移动。
&emsp;View的视图区域相当于无限大的，你可以在`onDraw`函数中的`canvas`中绘制任意大的图像，但是你会发现，最终屏幕上显示出来的只会是一部分，因为View自身还有大小概念，也就是`measure`和`layout`时，视图会被设置长宽还有界面中位置，这样的话，视图可视区域就被确定啦。
&emsp;做一个形象的比喻。View的可视区域就是一面墙上的窗户，View的视图区域就相当于墙后边的优美景色。墙外风光无线，但是你只能看到窗户中的景色。如果窗户变大啦，外边风景不变，你看到的景色就大了一点；如果窗户向右下角移动了一段距离，你就会发现外边的景色好像是向左上角"移动"了一段距离。


![view scroll example.png](http://7xrxif.com1.z0.glb.clouddn.com/2016327-androidscroll-view%20scroll%20example.png)


#### ScrollTo 和 ScrollBy
&emsp;这两个函数是用来滚动视图的API
``` java
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
```
&emsp;大家看源代码很容易就理解了二者的作用和区别:`scrollTo`就是直接改变`mScrollX`和`mScrollY`;而`scrollBy`则是给`mScrollX`和`mScrollY`加上增量。

####  invalidate和postInvalidate
&emsp;上边这两个函数都是请求视图重新绘制的API,但是二者的使用有些区别。
&emsp;`invalidate`必须在主线程(UI Thread)中调用，而`postInvalidate`可以在非主线程(Non UI Thread)中调用。
&emsp;除此之外，二者还有点小区别。
&emsp;调用`invalidate`时，它会检查上一次请求的UI重绘是否完成，如果没有完成的话，那么它就什么都不做。
``` java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
            .....
         //DRAWN和HAS_BOUNDS是否被设置为１，说明上一次请求执行的UI绘制已经完成，那么可以再次请求执行
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
                 ......
                 final AttachInfo ai = mAttachInfo;
                final ViewParent p = mParent;
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);//TODO:这是invalidate执行的主体
                .....
        }
    }
```
&emsp;而`postInvalidate`则不会这样，它是向主线程发送个`Message`，然后`handleMessage`时，调用了`invalidate()`函数。

``` java
//View.java
    public void postInvalidateDelayed(long delayMilliseconds) {
    ...	              attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    ...
    }

// ViewRootImpl 发送Message
    public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }

// ViewRootImpl 处理Message
public void handleMessage(Message msg) {
            switch (msg.what) {
            case MSG_INVALIDATE:
                ((View) msg.obj).invalidate();
                break;
            }
}   
```
&emsp;所以，二者的调用时机还是有区别的，就比如使用`Scroller`进行视图滚动时，二者的调用就有所不同。
####  后续
&emsp;之后还有会两篇博文，一篇是《Android Scroll详解(二)：OverScroller实战》讲解具体代码实现，另外一篇是《Android Scroll详解(三)：Android 绘制过程详解》主要是从滚动角度理解Android绘制过程,请大家多多关注啊。


> 参考文章

> http://stackoverflow.com/questions/7596370/what-is-the-difference-between-androids-invalidate-and-postinvalidate-metho

>http://www.programering.com/a/MDN3QDNwATQ.html

> http://blog.csdn.net/xiaanming/article/details/17483273

