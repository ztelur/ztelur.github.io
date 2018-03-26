title: Android Scroll详解(三)：Android 绘制过程详解
date: 2016-04-21 22:09:03
tags: 
	- draw
categories:
	- Android
---


作者： [ztelur](http://www.jianshu.com/users/481d9f540fb9/latest_articles)
联系方式：[segmentfault](https://segmentfault.com/u/remcarpediem)，[csdn](http://blog.csdn.net/u012422440)，[github](http://ztelur.github.io/)
>本文转载请注明原作者、文章来源，链接，版权归原文作者所有。

&emsp;本篇为Android Scroll系列文章的最后一篇，主要讲解Android视图绘制机制，由于本系列文章内容都是视图滚动相关的，所以，本篇从视图内容滚动的视角来梳理视图绘制过程。
&emsp;如果没有看过本系列之前文章或者不太了解相关的知识，请大家阅读一下一下的文章：

- [Android MotionEvent详解](http://ztelur.github.io/2016/03/16/Android-MotionEvent%E8%AF%A6%E8%A7%A3/)
- [Android Scroll详解(一)：基础知识](http://ztelur.github.io/2016/03/27/Android-Scroll%E8%AF%A6%E8%A7%A3-%E4%B8%80-%EF%BC%9A%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)
- [Android Scroll详解(二)：OverScroller实战](http://ztelur.github.io/2016/04/07/Android-Scroll%E8%AF%A6%E8%A7%A3-%E4%BA%8C-%EF%BC%9AOverScroller%E5%AE%9E%E6%88%98/)

&emsp;为了节约大家的时间，本文内容主要如下：

- `Scroller`相关机制。
- `mScrollX`和`mScrollY`是如何影响视图内容。
- Android视图绘制逻辑，包括相关API和`Canvas`的相关操作。


### 一切从`Scroller`使用开始
&emsp;使用scroller的实例代码，之后的讲解流程就是scroller和computeScroll是如何调用的啦。
&emsp;在系列文章的第二篇中，我们具体学习了`Scroller`的使用方法。通过`Scroller`的`fling`和`View`的`computeScroll`的配合，实现视图滚动效果。实例代码如下
``` java
    .....
    mScroller.fling(0,getScrollY(),0,speed,0,0,-500,10000)
    invalidate();

    .....
    @Override
    public void computeScroll() {
    		if (mScroller.computeScrollOffset()) {        
    			scrollTo(mScroller.getCurrX(),
    					mScroller.getCurrY());
               postInvalidate();
    	    }
    }

```
&emsp;本篇文章就带大家探究一下这段代码背后的原理和机制。

### Invalidate的寻父之路
&emsp;这一节主要分析在View中调用`invalidate`到`ViewRoot`执行`performTraversals`的原理，对android视图架构不是很熟悉的同学可以先阅读一下[《Android视图架构详解》](http://blog.csdn.net/u012422440/article/details/51173387)。
![invalidate时序图](http://7xrxif.com1.z0.glb.clouddn.com/2016421-view-invalidate%E6%97%B6%E5%BA%8F%E5%9B%BE2.jpg)

&emsp;我们先来看一下`View`中的`invalidate`代码。
```java

    public void invalidate() {
        invalidate(true);
    }
    void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }
        void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
                boolean fullInvalidate) {
    		.....
    		//DRAWN和HAS_BOUNDS是否被设置为１，说明上一次请求执行的UI绘制已经完成，那么可以再次请求执行
            if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                    || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                    || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                    || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
                if (fullInvalidate) {
                    mLastIsOpaque = isOpaque();
                    mPrivateFlags &= ~PFLAG_DRAWN;
                }
    
                mPrivateFlags |= PFLAG_DIRTY;
    
                if (invalidateCache) { //是否让view的缓存都失效
                    mPrivateFlags |= PFLAG_INVALIDATED;
                    mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
                }
    
                // Propagate the damage rectangle to the parent view.
                final AttachInfo ai = mAttachInfo;
                final ViewParent p = mParent;
    			//通过ViewParent来执行操作，如果当前视图是顶层视图也就是DecorView的视图，那么它的
    			//mParent就是ViewRoot对象，所以是通过ViewRoot的对象来实现的。
                if (p != null && ai != null && l < r && t < b) {
                    final Rect damage = ai.mTmpInvalRect;
                    damage.set(l, t, r, b);
                    p.invalidateChild(this, damage);//TODO:这是invalidate执行的主体
                }
                .....
            }
        }

```
&emsp;我们可以看到，调用`invalidate()`会导致整个视图进行刷新，并且会刷新缓存。
&emsp;然后我们再来详细的研究一下`invalidateInternal`中的代码。我们先来着重看一下`if`语句的判断条件把。
``` java

    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque))

```

- 当`mPrivateFlags`的`FLAG_DRAWN`和`FLAG_HAS_BOUNDS`位设置为1时，说明上一次请求执行的UI绘制已经完成，那么可以再次请求重新绘制。`FLAG_DRAWN`位会在`draw`函数中会被置为１，而`FLAG_HAS_BOUNDS`会在`setFrame`函数中被设置为1。
- `mPrivateFlags`的`PFLAG_DRAWING_CACHE_VALID`标示视图缓存是否有效，如果有效并且`invalidateCache`为true,那么可以请求重新绘制。
- 另外两个布尔判断的具体含义并没有分析清楚，大家感兴趣的请自行研究。
&emsp;然后将`mPrivateFlags`的`PFLAG_DIRTY`置为１。并且如果是要刷新缓存的话，将`PFLAG_INVALIDATED`位设置为１，并且将`PFLAG_DRAWING_CACHE_VALID`位设置为0,这一步和之前的`if`判断中后两个布尔判断相对应，可见，如果已经有一个`invalidate`设置了上述两个标志位，那么下一个`invalidate`就不会进行任何操作。
&emsp;接着，调用ViewParent接口的invalidateChild函数，在《Android视图架构详解》，我们已经知道`ViewGroup`和`ViewRoot`都实现了上述接口，那么，根据Android视图树状结构，`ViewGroup`的相应方法会被调用。
``` java

public final void invalidateChild(View child, final Rect dirty) {
    ViewParent parent = this;
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        ....
        // while一直向上递归
        do {
            ......
            parent = parent.invalidateChildInParent(location, dirty);
            ....
        } while (parent != null);
    }
}
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
            (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                    FLAG_OPTIMIZE_INVALIDATE) {
            ......
            return mParent;

        } else {
            .....
            return mParent;
        }
    }
    return null;
}

```

&emsp;通过上述代码我们可以看到`ViewGroup`的`invalidateChild`函数通过循环不断调用其父视图的`invalidateChildInParent`，而且我们知道`ViewRoot`是`DecorView`的父视图，也就是说`ViewRoot`是Android视图树状结构的根。所以，最终`ViewRoot`的`invalidateChildInParent`会被调用。

``` java
  

  //在ViewGroup的invalidateChildInParent中ｗｈｉｌｅ循环，一直调用到这里，然后在调用invalidateChild
    public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        invalidateChild(null, dirty);
        return null;
 }
 public void invalidateChild(View child, Rect dirty) {
    //先检查线程,必须是主线程
    checkThread();
    .....
    //如果mWillDrawSoon为true那么就是消息队列中已经有一个DO_TRAVERSAL的消息啦
    if (!mWillDrawSoon) {
         //直接调用了这个喽
        scheduleTraversals();
    }
}

```
&emsp;最终，在`ViewRoot`的`invalidateChild`函数中，调用了`scheduleTraversals`,开启了视图重绘之旅。

### 我们都被`ViewRoot`骗了
&emsp;`ViewRoot`是Android视图树状结构的根节点，并且它实现了`ViewParent`接口，是`DecorView`的父视图。那么大家一定会认为它就是一个`View`吧。那我们就被它给骗了！！`ViewRoot`本质上是一个`Handler`，我们可以看一下`scheduleTraversals`到`performTraversals`的原理就知道了。
``` java

public void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        sendEmptyMessage(DO_TRAVERSAL);
    }
}

```
&emsp;在`scheduleTraversals`中，`ViewRoot`只是向自己发送了一个`DO_TRAVERSAL`的空信息。
``` java

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
        ....
        case DO_TRAVERSAL:
        //这里就是Handle处理travel信息的地方
            if (mProfile) {
                Debug.startMethodTracing("ViewRoot");
            }
    
            performTraversals();
    
            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
            break;
            .....
        }
    }

```
&emsp;然后我们在查看`handleMessage`方法，发现在处理`DO_TRAVERSAL`时，`ViewRoot`调用了`performTraversals`函数。
&emsp;在`performTraversals`中，视图要进行measure,layout,和draw三大步骤，篇幅有限，我们这里只研究绘制相关的机制。
&emsp;`ViewRoot`在`performTraversals`中调用了自身的`draw`方法，看吧，`ViewRoot`伪装的还挺像，连`draw`方法都有。但是我们会发现，在`draw`方法中，`ViewRoot`实际上只调用了自己的`mView`成员变量的`draw`方法，而且我们都知道的是，`mView`就是`DecorView`，于是，绘制流程来到了真正的View视图的根节点。
### 大家都来画的canvas
&emsp;接下来，我们就正式研究一下Android的绘制机制，我们沿着Android视图的树状结构来分析绘制原理。
![draw 时序图](http://7xrxif.com1.z0.glb.clouddn.com/2016421-view-draw%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)

&emsp;首先是`DecorView`的绘制相关的函数。在`ViewRoot`的`draw`方法中，直接调用了`DecorView`的`draw(Canvas canvas)`函数，我们知道`DecorView`是`FrameLayout`的子类，其`draw(Canvas canvas)`函数是从`View`中继承而来的。所以我们先来看`View`的`draw(Canvas canvas)`方法。
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
&emsp;关于视图的组成部分，我在之前的文章中已经讲述过来，请不太熟悉这部分内容的同学自行查阅文章或者其他资料。通过上述代码我们可以看到，`View`的`dispatchDraw`函数被调用了，它是向子视图分发绘制指令和相关数据的方法。在`View`中，上述函数是一个空函数，但是`ViewGroup`中对这个函数进行了实现。
``` java

    protected void dispatchDraw(Canvas canvas) {
        ....
        final ArrayList<View> preorderedList = usingRenderNodeProperties
                ? null : buildOrderedChildList();
        final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                //在这里drawChild
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ....
    }
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    //这里就调用child的draw方法啦，而不是draw(canvas)方法！！！！！
        return child.draw(canvas, this, drawingTime);
    }

```
&emsp;通过上述代码我们可以看到，`ViewGroup`分别调用了自己的子View的`draw`方法，需要特别注意的是，这个draw和之前draw方法不是同一个方法，他们的参数不同。于是，我们再次转到`View`的源码中，看一下这个draw方法到底做了什么。
``` java

    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        ....
        //进行计算滚动
        if (!hasDisplayList) {
            computeScroll();
            sx = mScrollX;
            sy = mScrollY;
        }
        ...
        //这里进行了平移。
        if (offsetForScroll) {
            canvas.translate(mLeft - sx, mTop - sy);
        }
        ..... 
        if (!layerRendered) {
          if (!hasDisplayList) {
            // Fast path for layouts with no backgrounds
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
              mPrivateFlags &= ~PFLAG_DIRTY_MASK;
              dispatchDraw(canvas);
            } else {
              // 在这里调用了draw
              draw(canvas);
            }
          } 
                ......
        }
        ......
    }

```
&emsp;首先，我们发现`computeScroll`方法是在其中被调用的，从而计算出新的`mScrollX`和`mScrollY`,然后在平移画布，产生内容平移效果。
&emsp;然后我们发现通过`PFLAG_SKIP_DRAW`标志位的判断，有些View是直接调用`dispatchDraw`函数，说明它自己没有需要绘制的内容，而有些View则是调用自己的`draw`方法。我们应该都知道`ViewGroup`默认是不进行绘制内容的吧，我们一般调用`setNotWillDraw`方法来让其可以绘制自身内容，通过调用`setNotWillDraw`方法，会导致`PFLAG_SKIP_DRAW`位被置为１，从而可以绘制自身内容。
&emsp;分析到这里，我们就会发现draw函数沿着Android视图树状结构被不断调用，知道所有视图都完成绘制。
### 把一切连接起来的computeScroll
&emsp;读到这里大家应该对Android视图绘制流程有了基本的了解了吧，那么，我们再来看一下文章开头的例子。在`computeScroll`方法中，我们调用了`postInvalidate`方法，这又是什么用意呢？
&emsp;其实，在`computeScroll`中不掉用`postInvalidate`好像也可以达到正确的效果，具体原因我不太了解，猜测应该是Android自动刷新界面可以代替`postInvalidate`的效果吧。同学们如果知道其中具体原因，请告知我啊。
&emsp;在《Android Scroll详解(一)：基础知识》中，我们已经讲到
`postInvalidate`其实就是调用了`invalidate`，然后整个流程就连接了起来，`mScrollX`和`mScrollY`每个循环都会改变一点，然后导致界面滚动，最终形成界面Scroll效果。

### 后记
&emsp;Android Scroll的系列文章就此结束了，希望大家从中学习到有用的知识。如果其中有任何错误或者容易误解的地方，请大家及时通知我。谢谢各位读者和同学。


http://www.cppblog.com/fwxjj/archive/2013/01/13/197231.html
http://blog.csdn.net/luoshengyang/article/details/8372924


  [1]: /img/bVu4rO
  [2]: /img/bVu4rS

