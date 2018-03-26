---
title: 自定义Switch过程详解
date: 2016-07-15 22:13:49
tags:
	- View
	- Animation
categories:
	- Android
---
作者： [remcarpediem](http://ztelur.github.io/)
联系方式：[segmentfault](https://segmentfault.com/u/remcarpediem)，[csdn](http://blog.csdn.net/u012422440)，[简书](http://www.jianshu.com/users/481d9f540fb9/latest_articles)
>本文转载请注明作者、文章来源，链接，版权归作者所有。

&emsp;前段时间，我看到了一篇关于Android动画的文章[Android View 仿iOS SwitchButton Material Design](http://blog.csdn.net/bfbx5173/article/details/45191147)，十分喜欢文章作者的笔风，可惜每个人的笔风都不同，不过我倒是实现了一个类似的Switch组件,项目地址为[https://github.com/ztelur/FunSwitch](https://github.com/ztelur/FunSwitch)，就用这篇文章来讲述一下实现过程和机制吧。

#### 简介
&emsp;我的自定义Switch是模仿github上的[LLSwitch](https://github.com/lilei644/LLSwitch),其UI设计来源于Dribbble，链接[摸我](https://dribbble.com/shots/2706143-Dribbble-Debut-Boring-Funny-Slider-Animation),其效果图如下

![效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160715-sample.gif)

#### 自定义View需要重载的函数
&emsp;我们都知道以`View`为父类来自定义视图需要重载一系列函数，下面我们就来按照调用顺序来介绍一下这些函数。需要重载的函数列表如下:

- `onMeasure`  
- `onSizeChanged`
- `onDraw`
- `onTouchEvent`
- `onSaveInstanceState`
- `onRestoreInstanceState`

&emsp;首先就是`onMeasure`函数，用于确定自定义视图的长和高。对于本文的Switch,我们让其高为宽的固定比例大小就可以了，所以重构函数实现得十分简单。**这个函数确定的只是测量的长和高，并不是最终视图所显示的长和高**。
``` java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int width = MeasureSpec.getSize(widthMeasureSpec);
    int height = (int) (width * DEFAULT_WIDTH_HEIGHT_PERCENT);
    setMeasuredDimension(width,height);
}
```
&emsp;然后就是视图确定真正大小(`onLayout`)之后要调用的`onSizeChanged`函数了。这个函数调用之后，`draw`函数就可能被调用，所以，一般我们在这个函数中计算绘制时所需要的数据。
&emsp;接着是`draw`函数，在这个函数中，我们绘制各种图像来构成视图的UI。需要注意的是，这个函数会被频繁的调用，所以不要在函数内执行耗时的操作。
&emsp;最后是`onTouchEvent`函数，这个函数是用户触摸屏幕时才会被调用的，主要进行视图的触摸处理，由于我们的自定义Switch支持的触摸事件比较简单，只是支持点击事件，所以此函数的实现也比较简单。
&emsp;最后就是涉及到视图状态保存的两个函数。我们都知道，一定情况下，activity会被销毁，然后重新建立，比如你旋转屏幕时。这个时候，你需要保存视图的一些属性数据，以备重新建立视图时使用，来恢复之前的视图。**你需要注意的是，光重载这两个函数还是不够的，还需要设置View ID和调用setSaveEnabled函数**。
&emsp;我们接下来就一步一步的来实现这个自定义组件吧。
#### 田径场式背景
![田径场背景](http://7xrxif.com1.z0.glb.clouddn.com/20160715-boke1.png)

&emsp;我们先来看一下这个Switch的背景，它是一个形如田径场跑道的形状，由两个半圆和一个矩形组成，我们先来看一下如何来绘制出这样的图案。我们使用`Path`来构造出这样的图案，然后再进行绘制，代码如下所示:
``` java
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        //TODO：还有padding的问题偶！！！
        mWidth = w;
        mHeight = h;
        float top = 0;
        float left = 0;
        float bottom = h*0.8f; //下边预留0.2空间来画阴影
        float right = w;

        RectF backgroundRecf = new RectF(left,top,bottom,bottom);
        mBackgroundPath = new Path();
        //TODO:???????????
        mBackgroundPath.arcTo(backgroundRecf,90,180);

        backgroundRecf.left = right - bottom;
        backgroundRecf.right = right;
        mBackgroundPath.arcTo(backgroundRecf,270,180);
        mBackgroundPath.close();
        ........
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        drawBackground(canvas);
        drawForeground(canvas);
    }

    private void drawBackground(Canvas canvas) {
        mPaint.setColor(mCurrentColor);
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawPath(mBackgroundPath,mPaint);
        mPaint.reset();
    }
```
&emsp;我们使用`arcTo(RectF oval, float startAngle, float sweepAngle)`这儿函数来绘制田径场图案。这个函数，需要传入一个RectF对象，将要绘制的圆是这个对象所代表矩形的内切圆，我们只要计算出来这个矩形的上下左右四点的坐标就可以了。我们先计算绘制左侧半圆所需要的矩形，然后函数后两个参数为90,和180。注意的是，这个函数中，角度的正方向是顺时针的，`startAngle`为90,也就是我们数学坐标系中角度为270所代表的方向。
&emsp;由于Path会自动连接绘制个点之间的连线，所以，我们只需要再绘制出右侧半圆的曲线即可。
&emsp;我们只需要将绘制左侧圆曲线的矩形进行一定距离的平移，就可以绘制出右侧曲线。所以矩形的右边界就等于整个视图的`right`，由于矩形的长为`bottom`,所以矩形的左边界就为`right-bottom`。然后再次调用`arcTo`函数，这次的起始角度就变成270了。
&emsp;最后调用`Path`的`close`函数，让上边画的两段圆弧连接起来，就形成了上述的田径场图案。

#### 绘制脸部图形

![脸部图形](http://7xrxif.com1.z0.glb.clouddn.com/20160715-smileface.png)

&emsp;笑脸图案看似复杂，其实就是几个图形组合在一起。首先是一个大圆，然后是里边的两个椭圆型的眼睛，然后是嘴巴。我们只要在正确的位置将这些图形绘制出来即可。
&emsp;和绘制背景图形的顺序类似，我们首先在`onSizeChanged`函数中进行相关函数的计算。
&emsp;首先是大圆脸的绘制，我们还是使用`drawPath`函数，只不过这次`Path`对象只绘制一个圆；而双眼则是使用`drawOval`函数来花椭圆；最后使用`drawRect`来绘制矩形。

#### Switch动画

&emsp;我们仔细查看自定义Switch的动画效果，可以发现，主要涉及三部分的动画效果：

- 背景颜色动画转变。
- 脸部图形的平移和转动(可以看出相当于脸部水平转动了360度)。
- 脸部表情动画，眨眼睛和嘴巴咧开。

&emsp;由于动画涉及的操作比较多，所以我们选择使用`ValueAnimator`+`AnimatorUpdateListener`的动画实现方式，在`onAnimationUpdate`函数中记录下来当前的`animatedValue`，然后调用`invalidate`函数来让界面重绘，在绘制界面计算数据过程中，使用记录下来的数值，从而产生动画效果。
``` java
    private void startCloseAnimation() {
        mValueAnimator = ValueAnimator.ofFloat(NORMAL_ANIM_MAX_FRACTION,0);
        mValueAnimator.setDuration(mOffAnimationDuration);
        mValueAnimator.addUpdateListener(this);
        mValueAnimator.addListener(this);
        mValueAnimator.setInterpolator(mInterpolator);
        mValueAnimator.start();
        startColorAnimation();
    }
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        mAnimationFraction = (float)animation.getAnimatedValue();
        invalidate(); //产生动画的关键步骤
    }
```
&emsp;所以，最终动画问题又变成了绘制静态图像问题，我们根据不同的`mAnimationFraction`的值来绘制不同的图像。
&emsp;接下来我们就来描述一下几个比较关键的动画的逻辑。

#### 脸部转动动画
&emsp;其实这个脸部动画还是比较难实现的，主要是转动的这个效果没有直接的API可以实现。我们的动画只是让用户产生脸部转动的假象。由于脸部图案就是一个大圆加上充当眼睛和嘴巴的椭圆和矩形，我们可以让眼睛和嘴巴向转动方向平移，让它们平移出大圆，然后在一定时间后从另外一个方向再平移进入大圆，最终回到原来位置。这样就实现了一种脸部转动的效果。
&emsp;如何让眼睛和嘴巴移动到大圆边缘就消失呢？而且是随着移动渐渐的一部分一部分的消失呢？我们这里使用了另外一种思路，使用`clipPath`函数，将画布进行裁剪，只留下大圆范围内的图案。这样的话，当眼睛和嘴巴移动出大圆时，就会逐渐消失。
&emsp;至于眼睛和嘴巴如何平移呢？大家首先想到的方法一定是根据`mAnimationFraction`来计算它们的位置，然后在相应位置上将它们绘制出来，但是这样不是最优的方法，我们可以在绘制这些图像时，对画布进行平移，这样的话，我们绘制眼睛和嘴巴的函数就不会涉及到`mAnimationFraction`，实现比较简单。
``` java
public void drawFace(Canvas canvas,float fraction) {
        mPaint.setAntiAlias(true);
        //面部背景
        mPaint.setColor(mFaceColor);
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawPath(mFacePath,mPaint);
		//先裁剪并平移画布，然后再绘制眼部五官
        translateAndClipFace(canvas,fraction);
        drawEye(canvas,fraction);
        drawMouth(canvas,fraction);
    }

    private void translateAndClipFace(Canvas canvas,float fraction) {
        //截掉超出face的部分。
        canvas.clipPath(mFacePath);

        float faceTransition ;
        //TODO：合理的转动区间，眼睛出现和消失的时间比为1：1,所以当fraction=0.25时，应该只显示侧脸,计算faceTransition。
        if (fraction >=0.0f && fraction <0.5f) {
            faceTransition = fraction * mFaceRadius *4;
        } else if (fraction <=NORMAL_ANIM_MAX_FRACTION){
            faceTransition = - (NORMAL_ANIM_MAX_FRACTION - fraction) * mFaceRadius * 4;
        } else if (fraction <=(NORMAL_ANIM_MAX_FRACTION+FACE_ANIM_MAX_FRACTION)/2) {
            faceTransition =  (fraction - NORMAL_ANIM_MAX_FRACTION) * mFaceRadius * 2;
        } else {
            faceTransition = (FACE_ANIM_MAX_FRACTION - fraction) * mFaceRadius * 2;
        }
        canvas.translate(faceTransition,0);
    }
```

#### 眨眼睛和变笑脸动画
&emsp;眨眼睛动画十分简单，我们只要在绘制眼睛之前对画布进行缩放即可，然后在绘制玩眼睛之后，在将画布转变回来。但是后来我发现，画布缩放的中心点不容易确认，所以，采取了使用`mAnimationValue`计算椭圆数据的方式来进行椭圆大小的缩放。
&emsp;变笑脸动画主要就是嘴巴的动画效果，在静止情况下，我们使用`drawRect`来绘制嘴部图形；但在动画过程中，我们使用`drawPath`和`quadTo`来共同绘制嘴巴形状。
&emsp;`Path`的`quadTo`是用来绘制贝塞尔曲线，具体使用方法请查看[Path之贝塞尔曲线](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/%5B6%5DPath_Bezier.md)。我们主要使用其二阶曲线版本，即两个数据点，一个控制点。我们计算出A,B这两个数据点，也就是静止状态下矩形的左上点和右上点，然后根据`mAnimationValue`来计算控制点c的坐标，然后完成绘制。

![贝塞尔曲线原理图](http://7xrxif.com1.z0.glb.clouddn.com/20160715-quadTo.jpg)

&emsp;嘴部图案的绘制如下所示。
``` java
    private void drawMouth(Canvas canvas,float fraction) {
		.......
        //嘴巴
        if (fraction <=0.75) { //
            canvas.drawRect(mouthLeft, mouthTop, mouthLeft + mouthWidth, mouthTop + mouthHeight, mPaint);
        } else {
            Path path = new Path();
            path.moveTo(mouthLeft,mouthTop);
            float controlX = mouthLeft + mouthWidth/2;
            float controlY = mouthTop + mouthHeight + mouthHeight * 15 * (fraction - 0.75f);
        path.quadTo(controlX,controlY,mouthLeft+mouthWidth,mouthTop);
            path.close();
            canvas.drawPath(path,mPaint);
        }
    }
```

#### 总结
&emsp;其实还有一些细节问题我没有在这篇文章上讲出，一方面是因为讲述起来太过复杂，还是大家自己查阅代码比较好，另一方面是，我觉得自己实现的方式也不是很好，就不在这里献丑了。
&emsp;项目还没有完全完成，比如自定义监听器和自定义属性的相关逻辑都没有添加，希望感兴趣的同学可以自行研究代码并完善它。项目地址摸我[我的github](https://github.com/ztelur/FunSwitch)。

