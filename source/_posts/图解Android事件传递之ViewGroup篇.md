title: 图解Android事件传递之ViewGroup篇
date: 2016-02-11 17:21:20
tags:
	- 事件传递
categories:
	- Android
---
&emsp;本篇文章主要讲述ViewGroup中关于触摸事件传递的相关逻辑。主要梳理一下`dispatchTouchEvent`函数。
#### 一些知识点

- `FLAG_DISALLOW_INTERCEPT`，可以使用`requestDisallowInterceptTouchEvent`来设置`ViewGroup`的这个标记位，让ViewGroup不拦截事件。
- `ViewGroup`只会将触摸事件转发给那些可见并且触摸事件发生在其可视范围内的子`View`
- 如果一个子`View`没有接收`ACTION_DOWN`事件，那么这个事件系列的`ACTION_MOVE`或者`ACTION_UP`事件根本不会传递给它
- 关于`ViewGroup`拦截与否消费与否的判断，只要记住一点就可以轻易判断：1 `ViewGroup`是否最终没有消费触摸事件（无论是自己自己消费，还是分发给子view消费），决定之后的触摸事件是否会再转发给它。

![dispatchTouchEvent的主流程](http://7xrxif.com1.z0.glb.clouddn.com/201644-toucheventViewGroup.png)
![dispatchTouchEvent中遍历child分发事件的逻辑](http://7xrxif.com1.z0.glb.clouddn.com/201644-toucheventViewGroup%E9%81%8D%E5%8E%86child.png)

![转换触摸事件并分发的过程-dispatchTransformedTouchEvent](http://7xrxif.com1.z0.glb.clouddn.com/viewgroup3.png)

更详细的源代码请查看[我的github](https://github.com/ztelur/AOSP-analysis/tree/master/%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92)
