---
title: Android MotionEvent详解
tags:
  - MotionEvent
categories:
  - Android
abbrlink: e89a231a
date: 2016-03-16 13:54:44
---
&emsp;在前边几篇博文中（[《图解Android事件传递之ViewGroup篇》](http://ztelur.github.io/2016/02/11/%E5%9B%BE%E8%A7%A3Android%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92%E4%B9%8BViewGroup%E7%AF%87/)，[《图解Android事件传递之View篇》](http://ztelur.github.io/2016/02/04/%E5%9B%BE%E8%A7%A3Android%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92%E4%B9%8BView%E7%AF%87/)），我们已经了解了android触摸事件传递机制，接着我们再来研究一下与触摸事件传递相关的几个比较重要的类，比如`MotionEvent`。我们今天就来详细说明一下这个类的各方面用法。
#### 事件坐标的含义
 我们都知道，每个触摸事件都代表用户在屏幕上的一个动作，而每个动作必定有其发生的位置。在`MotionEvent`中就有一系列与标触摸事件发生位置相关的函数：
- `getX()`和`getY()`：由这两个函数获得的x,y值是相对的坐标值，相对于消费这个事件的视图的左上点的坐标。
- `getRawX()`和`getRawY()`:有这两个函数获得的x,y值是绝对坐标，是相对于屏幕的。
 在之前的文章中，我们曾经分析过事件如何通过层层分发，最终到达消费它的视图手中。其中`ViewGroup`的`dispatchTransformedTouchEvent`函数有如下一段代码:
```java
	final float offsetX = mScrollX - child.mLeft;
    final float offsetY = mScrollY - child.mTop;
    event.offsetLocation(offsetX, offsetY);
	handled = child.dispatchTouchEvent(event);
	event.offsetLocation(-offsetX, -offsetY);
```
 这段代码清晰展示了父视图把事件分发给子视图时，`getX()`和`getY`所获得的相关坐标是如何改变的。当父视图处理事件时，上述两个函数获得的相对坐标是相对于父视图的，然后通过上边这段代码，调整了相对坐标的值，让其变为相对于子视图啦。

![绝对坐标和相对坐标示意图](http://upload-images.jianshu.io/upload_images/623378-f45e4c2e22f0e8aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 事件类型
 涉及`MotionEvent`使用的代码一般如下:
``` java
	int action = MotionEventCompat.getActionMasked(event);
	switch(action) {
		case MotionEvent.ACTION_DOWN:
			break;
		case MotionEvent.ACTION_MOVE:
			break;
		case MotionEvent.ACTION_UP:
			break;
	}
```
 这里就引入了关于`MotionEvent`的一个重要概念，事件类型。事件类型就是指`MotionEvent`对象所代表的动作。比如说，当你的**一个**手指在屏幕上滑动一下时，系统会产生一系列的触摸事件对象,他们所代表的动作有所不同。有的事件代表你手指**按下**这个动作,有的事件代表你手指在屏幕上**滑动**,还有的事件代表你手指**离开**屏幕。这些事件的事件类型就分别为`ACTION_DOWN`,`ACTION_MOVE`,和`ACTION_UP`。上述这个动作所产生的一系列事件，被称为一个事件流，它包括一个`ACTION_DOWN`事件，很多个`ACTION_MOVE`事件，和一个`ACTION_UP`事件。


![单个手指动作.gif](http://7xrxif.com1.z0.glb.clouddn.com/2016316-%E5%8D%95%E4%B8%AA%E6%89%8B%E6%8C%87%E5%8A%A8%E4%BD%9C.gif)

 当然，除了这三个类型外，还有很多不同的事件类型,比如`ACTION_CANCEL`。它代表当前的手势被取消。要理解这个类型，就必须要了解`ViewGroup`分发事件的机制。一般来说，如果一个子视图接收了父视图分发给它的`ACTION_DOWN`事件，那么与`ACTION_DOWN`事件相关的事件流就都要分发给这个子视图，但是如果父视图希望拦截其中的一些事件，不再继续转发事件给这个子视图的话，那么就需要给子视图一个`ACTION_CANCEL`事件。
 其他的类型会在接下来的博文中一一解释。

#### Pointer
 细心的同学会发现，在上一节我描述用户手指在屏幕上滑动的例子时，特地说明了手指的数量为一个。那么当用户两个或者多个手指在屏幕上滑动时，系统又会产生怎样的事件流呢？
 为了可以表示多个触摸点的动作，`MotionEvent`中引入了`Pointer`的概念，一个pointer就代表一个触摸点，每个pointer都有自己的事件类型，也有自己的横轴坐标值。**一个`MotionEvent`对象中可能会存储多个pointer的相关信息，每个pointer都会有一个自己的id和index。pointer的id在整个事件流中是不会发生变化的，但是index会发生变化。**
 `MotionEvent`类中的很多方法都是可以传入一个int值作为参数的，其实传入的就是pointer的index值。比如`getX(pointerIndex)`和`getY(pointerIndex)`，此时，它们返回的就是index所代表的触摸点相关事件坐标值。
 由于pointer的index值在不同的`MotionEvent`对象中会发生变化，但是id值却不会变化。所以，当我们要记录一个触摸点的事件流时，就只需要保存其id,然后使用`findPointerIndex(int)`来获得其index值，然后再获得其他信息。
``` java
	private final static int INVALID_ID = -1;
    private int mActivePointerId = INVALID_ID;
    private int mSecondaryPointerId = INVALID_ID;
    private float mPrimaryLastX = -1;
    private float mPrimaryLastY = -1;
    private float mSecondaryLastX = -1;
    private float mSecondaryLastY = -1;
    public boolean onTouchEvent(MotionEvent event) {
        int action = MotionEventCompat.getActionMasked(event);

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                int index = event.getActionIndex();
                mActivePointerId = event.getPointerId(index);
                mPrimaryLastX = MotionEventCompat.getX(event,index);
                mPrimaryLastY = MotionEventCompat.getY(event,index);
                break;
            case MotionEvent.ACTION_POINTER_DOWN:
                index = event.getActionIndex();
                mSecondaryPointerId = event.getPointerId(index);
                mSecondaryLastX = event.getX(index);
                mSecondaryLastY = event.getY(index);
                break;
            case MotionEvent.ACTION_MOVE:
                index = event.findPointerIndex(mActivePointerId);
                int secondaryIndex = MotionEventCompat.findPointerIndex(event,mSecondaryPointerId);
                final float x = MotionEventCompat.getX(event,index);
                final float y = MotionEventCompat.getY(event,index);
                final float secondX = MotionEventCompat.getX(event,secondaryIndex);
                final float secondY = MotionEventCompat.getY(event,secondaryIndex);
                break;
            case MotionEvent.ACTION_POINTER_UP:
                xxxxxx(涉及pointer id的转换，之后的文章会讲解)
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                mActivePointerId = INVALID_ID;
                mPrimaryLastX =-1;
                mPrimaryLastY = -1;
                break;
        }
        return true;
    }
```

 除了pointer的概念，`MotionEvent`还引入了两个事件类型：
- `ACTION_POINTER_DOWN`:代表用户**又**使用一个手指触摸到屏幕上，也就是说，在已经有一个触摸点的情况下，有新出现了一个触摸点。
- `ACTION_POINTER_UP`:代表用户的一个手指离开了触摸屏，但是还有其他手指还在触摸屏上。也就是说，在多个触摸点存在的情况下，其中一个触摸点消失了。它与`ACTION_UP`的区别就是，它是在多个触摸点中的一个触摸点消失时（此时，还有触摸点存在，也就是说用户还有手指触摸屏幕）产生，而`ACTION_UP`可以说是最后一个触摸点消失时产生。

 那么，用户先两个手指先后接触屏幕，同时滑动，然后在先后离开这一套动作所产生的事件流是什么样的呢？
 它所产生的事件流如下：
- 先产生一个`ACTION_DOWN`事件，代表用户的第一个手指接触到了屏幕。
- 再产生一个`ACTION_POINTER_DOWN`事件，代表用户的第二个手指接触到了屏幕。
- 很多的`ACTION_MOVE`事件，但是在这些`MotionEvent`对象中，都保存着两个触摸点滑动的信息，相关的代码我们会在文章的最后进行演示。
- 一个`ACTION_POINTER_UP`事件，代表用户的一个手指离开了屏幕。
- 如果用户剩下的手指还在滑动时，就会产生很多`ACTION_MOVE`事件。
- 一个`ACTION_UP`事件，代表用户的最后一个手指离开了屏幕


![两个手指的动作.gif](http://7xrxif.com1.z0.glb.clouddn.com/2016316-%E4%B8%A4%E4%B8%AA%E6%89%8B%E6%8C%87%E7%9A%84%E5%8A%A8%E4%BD%9C.gif)

#### getAction 和 getActionMasked
&emsp;这一段之前有问题，多谢马同学指出其中的错误。
 看到文章开头那段代码的同学可能会有点疑问：好像在很多代码里，大家都是通过`getAction`获得事件类型的，那么它和`getActionMasked`又有什么不同呢？
 从上一节我们可以得知，一个`MotionEvent`对象中可以包含多个触摸点的事件。当`MotionEvent`对象只包含一个触摸点的事件时，上边两个函数的结果是相同的，但是当包含多个触摸点时，二者的结果就不同啦。
 `getAction`获得的int值是由pointer的index值和事件类型值组合而成的，而`getActionWithMasked`则只返回事件的类型值
 举个例子（注:假设了int中不同位所代表的含义，可能不是例子所中的前8位代表index,后8位代表事件类型）:
```
getAction() returns 0x0105.
getActionMasked() will return 0x0005
其中0x0100就是pointer的index值。
```
 一般来说，`(getAction() & ACTION_POINTER_INDEX_MASK)>>ACTION_POINTER_INDEX_SHIFT`就获得了pointer的index,等同于`getActionIndex`函数;`getAction()& ACTION_MASK `就获得了pointer的事件类型，等同于`getActionMasked`函数。

#### 批处理
 为了效率，Android系统在处理`ACTION_MOVE`事件时会将连续的几个多触点移动事件打包到一个`MotionEvent`对象中。我们可以通过`getX(int)`和`getY(int)`来获得最近发生的一个触摸点事件的坐标，然后使用`getHistorical(int,int)`和`getHistorical(int,int)`来获得时间稍早的触点事件的坐标，二者是发生时间先后的关系。所以，我们应该先处理通过`getHistoricalXX`相关函数获得的事件信息，然后在处理当前的事件信息。
 下边就是Android Guide中相关的例子:
``` java
 void printSamples(MotionEvent ev) {
     final int historySize = ev.getHistorySize();
     final int pointerCount = ev.getPointerCount();
     for (int h = 0; h < historySize; h++) {
         System.out.printf("At time %d:", ev.getHistoricalEventTime(h));
         for (int p = 0; p < pointerCount; p++) {
             System.out.printf("  pointer %d: (%f,%f)",
                 ev.getPointerId(p), ev.getHistoricalX(p, h), ev.getHistoricalY(p, h));
         }
     }
     System.out.printf("At time %d:", ev.getEventTime());
     for (int p = 0; p < pointerCount; p++) {
         System.out.printf("  pointer %d: (%f,%f)",
             ev.getPointerId(p), ev.getX(p), ev.getY(p));
     }
 }
```
#### 后续
 之后的博文会继续分析关于触摸处理的几个比较重要的类，比如`OverScroller`和`EdgeEffect`;然后会是一篇关于滑动手势处理代码分析的文章。请大家继续关注。

--
参考文章:
- [何时产生EVENT_CANCEL事件](http://stackoverflow.com/questions/11960861/what-causes-a-motionevent-action-cancel-in-android)
- [getAction和getActionMasked的区别](http://stackoverflow.com/questions/13710967/android-multitouch-and-getactionmasked)
- [MotionEvent的API](http://developer.android.com/intl/zh-cn/reference/android/view/MotionEvent.html)
- [Android Gesture Guide](http://developer.android.com/intl/zh-cn/training/gestures/index.html)
