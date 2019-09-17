---
title: Android  Span 进阶
tags:
  - Span
categories:
  - Android
abbrlink: 283058df
date: 2016-08-08 21:25:54
---

&emsp;在上篇文章[《Android Span 架构介绍》](http://blog.csdn.net/u012422440/article/details/52133037)，我们讲述了Android Span的基本概念和用法，这篇文章我们就来扩展一下我们对Android Span的了解，这一定会使你感到惊奇的，惊叹Android Span竟然还能完成这些的效果，让你在Android自定义View和动画方面有更加深刻的理解，可能会帮助你你想出更加简洁的实现方式。
&emsp;本篇文章主要讲述一下两个方面的内容:

- 自定义Android Span
- 使用Android Span实现动画

&emsp;先贴一下本篇文章实现的自定义Span和动画的效果图
![demo](http://7xrxif.com1.z0.glb.clouddn.com/20160808-Span-FireWorkSpan.gif)

![demo](http://7xrxif.com1.z0.glb.clouddn.com/20160808-Span-MutiableForegroundSpan.gif)

#### 自定义Span
&emsp;我们都知道，自定义View有两种方式，一种是继承特定的视图类,比如你希望修改`TextView`的行为，所以继承了`TextView`;另一种就是直接继承`View`或者`ViewGroup`,这样可以实现全新的视图和行为。如同自定义View一样，你有两种自定义Span的方法，一种直接继承特定类型的Span类，比如`ForegroundColorSpan`等，这样你可以在这些类的基础上进行修改；另一种就是继承`ReplaceSpan`这样的抽象类或者实现`LetterLineBackgroundSpan`这样的接口，你只要实现它给出的接口，就可以实现新的效果。
&emsp;我们先来讲解第一种方式。直接继承现有的`Span`。文章开头时展示的`ActionBar`动画就是通过继承`ForegroundSpan`来实现的。
&emsp;我们主要重载了`updateDrawsState`和`getForegroundColor`,这样就可以通过改变`setAlpha`函数来改变颜色，让字体从透明(alpha为0)到某个特定颜色。
```
public class MutableForegroundColorSpan extends ForegroundColorSpan {
    private int mAlpha = 255;
    private int mForegroundColor;
    public MutableForegroundColorSpan(int alpha,int color) {
        super(color);
        mAlpha = alpha;
        mForegroundColor = color;
    }
    @Override
    public void updateDrawState(TextPaint ds) {
        ds.setColor(getForegroundColor());
    }
    public void setAlpha(int alpha) {
        mAlpha = alpha;
    }
    public void setForegroundColor(int foregroundColor) {
        mForegroundColor = foregroundColor;
    }
    public float getAlpha() {
        return mAlpha;
    }
    @Override
    public int getForegroundColor() {
        return Color.argb(mAlpha,Color.red(mForegroundColor),Color.green(mForegroundColor),Color.blue(mForegroundColor));
    }
}
```
&emsp;第二种方法是继承`Span`架构中的抽象类或者是实现特定接口。需要注意的是，在上一篇文章中说的`CharacterStyle`,`ParagraphStyle`.`UpdateAppearance`和`UpdateLayout`都是没有函数的，所以，我们无法直接继承或者实现它们。除了第一篇文章中所介绍的那些`Span`可以使用第一种方法进行继承。我们一般都继承或者实现`MetricAffectingSpan`,`ReplacementSpan`或者`LineBackgroundSpan`。
&emsp;比如我们想给每个字都添加一个不同颜色的背景，我们就可以继承`ReplacementSpan`

![demo](http://7xrxif.com1.z0.glb.clouddn.com/20160808-Span-BuddleSpan.png)

```
public class BubbleSpan extends ReplacementSpan {
    private Paint mPaint;
    static Random random = new Random();
    private int mWidth = -1;
    private RectF mRectF = new RectF();

    private int[] mColors = new int[20];

    public BubbleSpan() {
        initPaint();
        initColors();
    }

    private void initPaint() {
        mPaint = new Paint();
        mPaint.setColor(Color.rgb(random.nextInt(255), random.nextInt(255), random.nextInt(255)));
        mPaint.setAntiAlias(true);
    }

    private void initColors() {
        for(int index = 0 ; index < mColors.length ; index++) {
            mColors[index] = Color.rgb(random.nextInt(255), random.nextInt(255), random.nextInt(255));
        }
    }

    @Override
    public int getSize(Paint paint, CharSequence text, int start, int end, Paint.FontMetricsInt fm) {
        //return text with relative to the Paint
		mWidth = (int) paint.measureText(text, start, end);
        return mWidth;
    }

    @Override
    public void draw(Canvas canvas, CharSequence text, int start, int end, float x, int top, int y, int bottom, Paint paint) {
        float charx = x;
        for(int i = start ; i<end; i++) {
            String charAt = extractText(text, i, i + 1);
            float charWidth = paint.measureText(charAt);
            mRectF.set(charx, top, charx += charWidth, bottom);
            mPaint.setColor(mColors[i % mColors.length]);
            //根据每个字的位置绘制背景
            canvas.drawOval(mRectF, mPaint);
        }
        //绘制字体，如果不掉用这个函数，就不会显示字体啦。
        canvas.drawText(text, start, end, x, y, paint);
    }

    private String extractText(CharSequence text, int start, int end) {
        return text.subSequence(start, end).toString();
    }
}
```
&emsp;我们可以看到，我们要实现两个函数:`getSize`和`draw`。`getSize`是获得字体的长度的，所以一般都是直接使用`paint.measureText`,然后`draw`中进行绘制，你可以在这里把每个字的背景绘制出来，而且你必须也要把字体给绘制出来。如果你只想绘制背景，不想涉及字体的绘制，那么就可以直接实现`LineBackgroundSpan`接口。
```
public class RectSpan extends ReplacementSpan {
    private final Paint mPaint;
    private int mWidth;

    public RectSpan() {
        mPaint = new Paint();
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setColor(Color.BLUE);
        mPaint.setAntiAlias(true);
    }

    @Override
    public int getSize(Paint paint, CharSequence text, int start, int end, Paint.FontMetricsInt fm) {
        //return text with relative to the Paint
        mWidth = (int) paint.measureText(text, start, end);
        return mWidth;
    }

    @Override
    public void draw(Canvas canvas, CharSequence text, int start, int end, float x, int top, int y, int bottom, Paint paint) {
        //只绘制了外围矩形，没有绘制文字。
        canvas.drawRect(x, top, x + mWidth, bottom, mPaint);
    }
}
```
![demo](http://7xrxif.com1.z0.glb.clouddn.com/20160808-Span-RectSpan.png)
#### 通过Span实现动画
&emsp;在上一节中，我们已经实现了`MutableForegroundColorSpan`类，那么如何使用它来实现动画呢？这里我们就要使用到`ObjectAnimator`和`Property<T1,T2>`。我们首先定义`MutableForegroundColorSpan`使用的`property`。
```
private static final Property<MutableForegroundColorSpan, Integer> MUTABLE_FOREGROUND_COLOR_SPAN_PROPERTY =
        new Property<MutableForegroundColorSpan, Integer>(Integer.class, "MUTABLE_FOREGROUND_COLOR_SPAN_FC_PROPERTY") {

        @Override
                public void set(MutableForegroundColorSpan alphaForegroundColorSpanGroup, Integer value) {
                        alphaForegroundColorSpanGroup.setForegroundColor(value);
                }

        @Override
                public Integer get(MutableForegroundColorSpan span) {
                        return span.getForegroundColor();
                }
};

```
&emsp;然后我们就可以使用`ObjectAnimator`对`MutableForegroundColorSpan`实现属性动画了。
```
MutableForegroundColorSpan span = new MutableForegroundColorSpan(255, Color.BLUE);
final SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span, 0,4, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
ObjectAnimator objectAnimator = ObjectAnimator.ofInt(span, MUTABLE_FOREGROUND_COLOR_SPAN_PROPERTY, Color.BLACK, Color.RED);
objectAnimator.setEvaluator(new ArgbEvaluator());
objectAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
        //refresh
                mTvTextView.setText(spannableString);
        }       
});
objectAnimator.setInterpolator(mSmoothInterpolator);
objectAnimator.setDuration(600);
objectAnimator.start();
```
&emsp;这里还只是单独一个`Span`实例的动画效果，你可以对多个Span实例进行属性动画，从而实现更加复杂的动画效果。就比如文章开始时的文字逐渐显示的动画效果。
&emsp;我们可以给每个字都设置一个`MutableForegroundColorSpan`实例,并将这些实例都添加到一个对象中，然后在属性动画过程中，乱序设置每个实例的`alpha`的值，从而达到文字逐渐显现的动画。
```
public class FireWorkGroup {
    private final float mProgress;
    private final ArrayList<MutableForegroundColorSpan> mSpans;
    private final ArrayList<Integer> mSpanIndexes;

    public FireWorkGroup() {
        mProgress = 0;
        mSpans  = new ArrayList<>();
        mSpanIndexes = new ArrayList<>();
    }

    public void addSpan(MutableForegroundColorSpan span) {
        span.setAlpha(0);
        mSpanIndexes.add(mSpans.size());
        mSpans.add(span);
    }
    public void init() {
        Collections.shuffle(mSpans);
    }
    public void setProgress(float progress) {
        int size = mSpans.size();
        float total  = 1.0f * size * progress;
        for (int index = 0 ; index < size ; index++) {
            MutableForegroundColorSpan span = mSpans.get(index);
            if (total > 1.0f) {
                span.setAlpha(255);
                total -= 1.0f;
            } else {
                span.setAlpha((int)(total * 255));
                total = 0.0f;
            }
        }
    }
    public float getProgress() {
        return mProgress;
    }
    public static final Property<FireWorkGroup, Float> FIREWORKS_GROUP_PROGRESS_PROPERTY =
            new Property<FireWorkGroup, Float>(Float.class, "FIREWORKS_GROUP_PROGRESS_PROPERTY") {

                @Override
                public void set(FireWorkGroup spanGroup, Float value) {
                    spanGroup.setProgress(value);
                }

                @Override
                public Float get(FireWorkGroup spanGroup) {
                    return spanGroup.getProgress();
                }
            };
}
```

#### 后记
&emsp;上述的这些动画都是我在第一篇文章提到的那篇博文中实现过的效果，我在学习过程中又发现了一个实现了很多`TextView`相关动画的开源项目[HTextView](https://github.com/hanks-zyh/HTextView)。所以接下来的任务就是想使用`Span`机制去实现这个项目中的一些动画效果。希望大家继续支持我的文章，并积极指出文中的错误。
&emsp;!!!源码都在[我的github](https://github.com/ztelur/AndroidSpan)里。

