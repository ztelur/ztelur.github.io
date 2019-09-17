---
title: Android Scroll详解(二)：OverScroller实战
tags:
  - scroll
categories:
  - Android
abbrlink: e2cc6893
date: 2016-04-07 22:58:49
---
作者： [ztelur](http://www.jianshu.com/users/481d9f540fb9/latest_articles)
联系方式：[segmentfault](https://segmentfault.com/u/remcarpediem)，[csdn](http://blog.csdn.net/u012422440)，[github](http://ztelur.github.io/)
>本文仅供个人学习，不用于任何形式商业目的，转载请注明原作者、文章来源，链接，版权归原文作者所有。

&emsp;本文是android滚动相关的系列文章的第二篇，主要总结一下使用手势相关的代码逻辑。主要是单点拖动，多点拖动，fling和OveScroll的实现。每个手势都会有代码片段。
&emsp;对android滚动相关的知识还不太了解的同学可以先阅读一下文章：

- [《Android-MotionEvent详解》](http://ztelur.github.io/2016/03/16/Android-MotionEvent%E8%AF%A6%E8%A7%A3/)
- [《Android Scroll详解(一)：基础知识》](http://ztelur.github.io/2016/03/27/Android-Scroll%E8%AF%A6%E8%A7%A3-%E4%B8%80-%EF%BC%9A%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)

&emsp;为了节约你的时间，我特地将文章大致内容总结如下：

- 手势Drag的实现和原理
- 手势Fling的实现和原理
- OverScroll效果和EdgeEffect效果的实现和原理。

&emsp;详细代码请查看[我的github](https://github.com/ztelur/ScrollProject)
### Drag
&emsp;Drag是最为基本的手势：用户可以使用手指在屏幕上滑动，以拖动屏幕相应内容移动。实现Drag手势其实很简单，步骤如下：

- 在`ACTION_DOWN`事件发生时，调用`getX`和`getY`函数获得事件发生的x,y坐标值，并记录在`mLastX`和`mLastY`变量中。
- 在`ACTION_MOVE`事件发生时，调用`getX`和`getY`函数获得事件发生的x,y坐标值,将其与`mLastX`和`mLastY`比较，如果二者差值大于一定限制（ScaledTouchSlop）,就执行`scrollBy`函数，进行滚动,最后更新`mLastX`和`mLastY`的值。
- 在`ACTION_UP`和`ACTION_CANCEL`事件发生时，清空`mLastX`，`mLastY`。

``` java
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int actionId = MotionEventCompat.getActionMasked(event);
        switch (actionId) {
            case MotionEvent.ACTION_DOWN:
                mLastX = event.getX();
                mLastY = event.getY();
                mIsBeingDragged = true;
                if (getParent() != null) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }
                break;
            case MotionEvent.ACTION_MOVE:
                float curX = event.getX();
                float curY = event.getY();
                int deltaX = (int) (mLastX - curX);
                int deltaY = (int) (mLastY - curY);
                if (!mIsBeingDragged && (Math.abs(deltaX)> mTouchSlop ||
                                                        Math.abs(deltaY)> mTouchSlop)) {
                    mIsBeingDragged = true;
                    // 让第一次滑动的距离和之后的距离不至于差距太大
                    // 因为第一次必须>TouchSlop,之后则是直接滑动
                    if (deltaX > 0) {
                        deltaX -= mTouchSlop;
                    } else {
                        deltaX += mTouchSlop;
                    }
                    if (deltaY > 0) {
                        deltaY -= mTouchSlop;
                    } else {
                        deltaY += mTouchSlop;
                    }
                }
                // 当mIsBeingDragged为true时，就不用判断> touchSlopg啦，不然会导致滚动是一段一段的
                // 不是很连续
                if (mIsBeingDragged) {
                        scrollBy(deltaX, deltaY);
                        mLastX = curX;
                        mLastY = curY;
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mIsBeingDragged = false;
                mLastY = 0;
                mLastX = 0;
                break;
            default:
        }
        return mIsBeingDragged;
    }
```

### 多触点Drag
&emsp;上边的代码只适用于单点触控的手势，如果你是两个手指触摸屏幕，那么它只会根据你第一个手指滑动的情况来进行屏幕滚动。更为致命的是，当你先松开第一个手指时，由于我们少监听了`ACTION_POINTER_UP`事件，将会导致屏幕突然滚动一大段距离，因为第二个手指移动事件的x,y值会和第一个手指移动时留下的`mLastX`和`mLastY`比较，导致屏幕滚动。

&emsp;如果我们要监听并处理多触点的事件，我们还需要对`ACTION_POINTER_DOWN`和`ACTION_POINTER_UP`事件进行监听，并且在`ACTION_MOVE`事件时，要记录所有触摸点事件发生的x,y值。

- 当`ACTION_POINTER_DOWN`事件发生时，我们要记录第二触摸点事件发生的x,y值为`mSecondaryLastX`和`mSecondaryLastY`，和第二触摸点pointer的id为`mSecondaryPointerId`
- 当`ACTION_MOVE`事件发生时，我们除了根据第一触摸点pointer的x，y值进行滚动外，也要更新`mSecondayLastX`和`mSecondaryLastY`
- 当`ACTION_POINTER_UP`事件发生时，我们要先判断是哪个触摸点手指被抬起来啦，如果是第一触摸点，那么我们就将坐标值和pointer的id都更换为第二触摸点的数据；如果是第二触摸点，就只要重置一下数据即可。

``` java
        switch (actionId) {
            .....
            case MotionEvent.ACTION_POINTER_DOWN:
                activePointerIndex = MotionEventCompat.getActionIndex(event);
                mSecondaryPointerId = MotionEventCompat.findPointerIndex(event,activePointerIndex);
                mSecondaryLastX = MotionEventCompat.getX(event,activePointerIndex);
                mSecondaryLastY = MotionEventCompat.getY(event,activePointerIndex);
                break;
            case MotionEvent.ACTION_MOVE:
                ......
                // handle secondary pointer move
                if (mSecondaryPointerId != INVALID_ID) {
                    int mSecondaryPointerIndex = MotionEventCompat.findPointerIndex(event, mSecondaryPointerId);
                    mSecondaryLastX = MotionEventCompat.getX(event, mSecondaryPointerIndex);
                    mSecondaryLastY = MotionEventCompat.getY(event, mSecondaryPointerIndex);
                }
                break;
            case MotionEvent.ACTION_POINTER_UP:
                //判断是否是activePointer up了
                activePointerIndex = MotionEventCompat.getActionIndex(event);
                int curPointerId  = MotionEventCompat.getPointerId(event,activePointerIndex);
                Log.e(TAG, "onTouchEvent: "+activePointerIndex +" "+curPointerId +" activeId"+mActivePointerId+
                                        "secondaryId"+mSecondaryPointerId);
                if (curPointerId == mActivePointerId) { // active pointer up
                    mActivePointerId = mSecondaryPointerId;
                    mLastX = mSecondaryLastX;
                    mLastY = mSecondaryLastY;
                    mSecondaryPointerId = INVALID_ID;
                    mSecondaryLastY = 0;
                    mSecondaryLastX = 0;
                    //重复代码，为了让逻辑看起来更加清晰
                } else{ //如果是secondary pointer up
                    mSecondaryPointerId = INVALID_ID;
                    mSecondaryLastY = 0;
                    mSecondaryLastX = 0;
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mIsBeingDragged = false;
                mActivePointerId = INVALID_ID;
                mLastY = 0;
                mLastX = 0;
                break;
            default:
        }
```

### Fling
&emsp;当用户手指快速划过屏幕，然后快速立刻屏幕时，系统会判定用户执行了一个Fling手势。视图会快速滚动，并且在手指立刻屏幕之后也会滚动一段时间。Drag表示手指滑动多少距离，界面跟着显示多少距离，而fling是根据你的滑动方向与轻重，还会自动滑动一段距离。Filing手势在android交互设计中应用非常广泛：电子书的滑动翻页、ListView滑动删除item、滑动解锁等。所以如何检测用户的fling手势是非常重要的。
&emsp;在检测Fling时，你需要检测手指在屏幕上滑动的速度，这是你就需要`VelocityTracker`和`Scroller`这两个类啦。

- 我们首先使用`VelocityTracker.obtain()`这个方法获得其实例
- 然后每次处理触摸时间时，我们将触摸事件通过`addMovement`方法传递给它
- 最后在处理`ACTION_UP`事件时，我们通过`computeCurrentVelocity`方法获得滑动速度;
- 我们判断滑动速度是否大于一定数值(MinFlingSpeed),如果大于，那么我们调用`Scroller`的`fling`方法。然后调用`invalidate()`函数。
- 我们需要重载`computeScroll`方法，在这个方法内，我们调用`Scroller`的`computeScrollOffset()`方法啦计算当前的偏移量，然后获得偏移量，并调用`scrollTo`函数,最后调用`postInvalidate()`函数。
- 除了上述的操作外，我们需要在处理`ACTION_DOWN`事件时，对屏幕当前状态进行判断，如果屏幕现在正在滚动(用户刚进行了Fling手势)，我们需要停止屏幕滚动。

&emsp;具体这一套流程是如何运转的，我会在下一篇文章中详细解释，大家也可以自己查阅代码或者google来搞懂其中的原理。
``` java
	@Override
    public boolean onTouchEvent(MotionEvent event) {
        .....
        if (mVelocityTracker == null) {
	        //检查速度测量器，如果为null，获得一个
            mVelocityTracker = VelocityTracker.obtain();
        }
        int action = MotionEventCompat.getActionMasked(event);
        int index = -1;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                ......
                                if (!mScroller.isFinished()) { //fling
                    mScroller.abortAnimation();
                }
                .....
                break;
            case MotionEvent.ACTION_MOVE:
                ......
                break;
            case MotionEvent.ACTION_CANCEL:
                endDrag();
                break;
            case MotionEvent.ACTION_UP:
                if (mIsBeingDragged) {
                //当手指立刻屏幕时，获得速度，作为fling的初始速度     mVelocityTracker.computeCurrentVelocity(1000,mMaxFlingSpeed);
                    int initialVelocity = (int)mVelocityTracker.getYVelocity(mActivePointerId);
                    if (Math.abs(initialVelocity) > mMinFlingSpeed) {
                        // 由于坐标轴正方向问题，要加负号。
                        doFling(-initialVelocity);
                    }
                    endDrag();
                }
                break;
            default:
        }
        //每次onTouchEvent处理Event时，都将event交给时间
        //测量器
        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(event);
        }
        return true;
    }
    private void doFling(int speed) {
        if (mScroller == null) {
            return;
        }
        mScroller.fling(0,getScrollY(),0,speed,0,0,-500,10000);
        invalidate();
    }
    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
            postInvalidate();
        }
    }
```
#### OverScroll
&emsp;在Android手机上，当我们滚动屏幕内容到达内容边界时，如果再滚动就会有一个发光效果。而且界面会进行滚动一小段距离之后再回复原位，这些效果是如何实现的呢？我们需要使用`Scroller`和`scrollTo`的升级版`OverScroller`和`overScrollBy`了，还有发光的`EdgeEffect`类。
&emsp;我们先来了解一下相关的API，理解了这些接口参数的含义，你就可以轻松使用这些接口来实现上述的效果啦。
``` java
protected boolean overScrollBy(int deltaX, int deltaY,
            int scrollX, int scrollY,
            int scrollRangeX, int scrollRangeY,
            int maxOverScrollX, int maxOverScrollY,
            boolean isTouchEvent)
```

- int deltaX,int deltaY : 偏移量，也就是当前要滚动的x，y值。
- int scrollX,int scrollY : 当前的mScrollX和mScrollY的值。
- int scrollRangeX,int scrollRangeY: 标示可以滚动的最大的x,y值，也就是你视图真实的长和宽。也就是说，你的视图可视大小可能是100,100,但是视图中的内容的大小为200,200,所以，上述两个值就为200,200
- int maxOverScrollX,int maxOverScrollY：允许超过滚动范围的最大值，x方向的滚动范围就是0~maxOverScrollX,y方向的滚动范围就是0~maxOverScrollY。
- boolean isTouchEvent:是否在`onTouchEvent`中调用的这个函数。所以，当你在`computeScroll`中调用这个函数时，就可以传入false。

``` java
protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY)
```
- int scrollX,int scrollY:就是x，y方向的滚动距离，就相当于`mScrollX`和`mScrollY`。你既可以直接把二者赋值给相应的成员变量，也可以使用`scrollTo`函数。
- boolean clampedX,boolean clampY:表示是否到达超出滚动范围的最大值。如果为true，就需要调用`OverScroll`的`springBack`函数来让视图回复原来位置。

``` java
public boolean springBack(int startX, int startY, int minX, int maxX, int minY, int maxY)
```

- int startX,int startY:标示当前的滚动值，也就是`mScrollX`和`mScrollY`的值。
- int minX,int maxX:标示x方向的合理滚动值
- int minY,int maxY:标示y方向的合理滚动值。

&emsp;相信看完上述的API之后，大家会有很多的疑惑，所以这里我来举个例子。
&emsp;假设视图大小为100*100。当你一直下拉到视图上边缘，然后在下拉，这时，`mScrollY`已经达到或者超过正常的滚动范围的最小值了，也就是0，但是你的maxOverScrollY传入的是10,所以，`mScrollY`最小可以到达-10,最大可以为110。所以，你可以继续下拉。等到`mScrollY`到达或者超过-10时，clampedY就为true，标示视图已经达到可以OverScroll的边界，需要回滚到正常滚动范围，所以你调用springBack(0,0,0,100)。

&emsp;然后我们再来看一下发光效果是如何实现的。
&emsp;使用`EdgeEffect`类。一般来说，当你只上下滚动时，你只需要两个`EdgeEffect`实例，分别代表上边界和下边界的发光效果。你需要在下面两个情景下改变`EdgeEffect`的状态，然后在`draw()`方法中绘制`EdgeEffect`

- 处理`ACTION_MOVE`时，如果发现y方向的滚动值超过了正常范围的最小值时，你需要调用上边界实例的`onPull`方法。如果是超过最大值，那么就是调用下边界的`onPull`方法。
- 在`computeScroll`函数中，也就是说Fling手势执行过程中，如果发现y方向的滚动值超过正常范围时的最小值时，调用`onAbsorb`函数。

&emsp;然后就是重载`draw`方法，让`EdgeEffect`实例在画布上绘制自己。你会发现，你必须对画布进行移动或者旋转来让`EdgeEffect`绘制出上边界或者下边界的发光的效果，因为`EdgeEffect`对象自己是没有上下左右的概念的。


``` java
    @Override
    public void draw(Canvas canvas) {
        super.draw(canvas);
        if (mEdgeEffectTop != null) {
            final int scrollY = getScrollY();
            if (!mEdgeEffectTop.isFinished()) {
                final int count = canvas.save();
                final int width = getWidth() - getPaddingLeft() - getPaddingRight();
                canvas.translate(getPaddingLeft(),Math.min(0,scrollY));
                mEdgeEffectTop.setSize(width,getHeight());
                if (mEdgeEffectTop.draw(canvas)) {
                    postInvalidate();
                }
                canvas.restoreToCount(count);
            }

        }
        if (mEdgeEffectBottom != null) {
            final int scrollY = getScrollY();
            if (!mEdgeEffectBottom.isFinished()) {
                final int count = canvas.save();
                final int width = getWidth() - getPaddingLeft() - getPaddingRight();
                canvas.translate(-width+getPaddingLeft(),Math.max(getScrollRange(),scrollY)+getHeight());
                canvas.rotate(180,width,0);
                mEdgeEffectBottom.setSize(width,getHeight());
                if (mEdgeEffectBottom.draw(canvas)) {
                    postInvalidate();
                }
                canvas.restoreToCount(count);
            }

        }
    }
    
 @Override
    public boolean onTouchEvent(MotionEvent event) {
            ......
            case MotionEvent.ACTION_MOVE:
                .....
                if (mIsBeingDragged) {
                    overScrollBy(0,(int)deltaY,0,getScrollY(),0,getScrollRange(),0,mOverScrollDistance,true);
                    final int pulledToY = (int)(getScrollY()+deltaY);
                    mLastY = y;
                    if (pulledToY<0) {
                        mEdgeEffectTop.onPull(deltaY/getHeight(),event.getX(mActivePointerId)/getWidth());
                        if (!mEdgeEffectBottom.isFinished()) {
                            mEdgeEffectBottom.onRelease();
                        }
                    } else if(pulledToY> getScrollRange()) {
                        mEdgeEffectBottom.onPull(deltaY/getHeight(),1.0f-event.getX(mActivePointerId)/getWidth());
                        if (!mEdgeEffectTop.isFinished()) {
                            mEdgeEffectTop.onRelease();
                        }
                    }
                    if (mEdgeEffectTop != null && mEdgeEffectBottom != null &&(!mEdgeEffectTop.isFinished()
                                        || !mEdgeEffectBottom.isFinished())) {
                        postInvalidate();
                    }
                }
                .....
        }
        ....
    }
    @Override
    protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY) {
        if (!mScroller.isFinished()) {  
            int oldX = getScrollX();
            int oldY = getScrollY();
            scrollTo(scrollX,scrollY);
            onScrollChanged(scrollX,scrollY,oldX,oldY);
            if (clampedY) {
                Log.e("TEST1","springBack");
                mScroller.springBack(getScrollX(),getScrollY(),0,0,0,getScrollRange());
            }
        } else {
            // TouchEvent中的overScroll调用
            super.scrollTo(scrollX,scrollY);
        }
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            int oldX = getScrollX();
            int oldY = getScrollY();
            int x = mScroller.getCurrX();
            int y = mScroller.getCurrY();

            int range = getScrollRange();
            if (oldX != x || oldY != y) {
                overScrollBy(x-oldX,y-oldY,oldX,oldY,0,range,0,mOverFlingDistance,false);
            }
            final int overScrollMode = getOverScrollMode();
            final boolean canOverScroll = overScrollMode == OVER_SCROLL_ALWAYS ||
                    (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);
            if (canOverScroll) {
                if (y<0 && oldY >= 0) {
                    mEdgeEffectTop.onAbsorb((int)mScroller.getCurrVelocity());
                } else if (y> range && oldY < range) {
                    mEdgeEffectBottom.onAbsorb((int)mScroller.getCurrVelocity());
                }
            }
        }
    }

```

### 后记
&emsp;本篇文章是系列文章的第二篇，大家可能已经知道如何实现各类手势，但是对其中的机制和原理还不是很了解，之后的第三篇会讲解从本篇代码的视角讲解一下android视图绘制的原理和Scroller的机制，希望大家多多关注。

### 参考文章
http://stackoverflow.com/questions/22843671/android-swipe-vs-fling

https://www.google.com/design/spec/patterns/gestures.html#gestures-drag-swipe-or-fling-details

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1212/2145.html
