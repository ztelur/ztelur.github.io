title: 图解Android事件传递之View篇
date: 2016-02-04 17:09:01
tags:
	- 事件传递
categories:
	- Android
---
&emsp;最近放假在家里闲着，想好好研究一下android的事件传递机制。于是便抓来`View`,`ViewGroup`这些类的源代码来看；有很多疑惑，又在网上找到了几篇比较好的介绍事件传递机制的文章阅读了一番。然后想着最好把学习到的知识输出一遍，画成视图，写下这篇博文。
&emsp;除了图片，我还在源码上进行了注释，提交到了github上去。[我的github](https://github.com/ztelur/AOSP-analysis/tree/master/%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92)
#### View的dispatchTouchEvent
![View-dispatchTouchEvent.png](http://7xrxif.com1.z0.glb.clouddn.com/touch-view1.png)
#### View的onTouchEvent
![View-onTouchEvent1.png](http://7xrxif.com1.z0.glb.clouddn.com/touch-view2.png)

![View-onTouchEvent2.png](http://7xrxif.com1.z0.glb.clouddn.com/touch-view3.png)

#### 参考的文章
- [一位大神的博文，他的文章系列都值得看一看](http://wangkuiwu.github.io/2015/01/05/TouchEvent-Sample-02-View/)
- [郭大神的博客](http://blog.csdn.net/guolin_blog/article/details/9097463)
- [很详细但有错误的地方的一篇经典文章](http://www.infoq.com/cn/articles/android-event-delivery-mechanism)
