---
title: Property Animation框架详解(一)
tags:
  - animation
categories:
  - Android
abbrlink: 2f8becff
date: 2016-05-09 14:14:19
---

作者： [ztelur](http://ztelur.github.io/)
联系方式：[segmentfault](https://segmentfault.com/u/remcarpediem)，[csdn](http://blog.csdn.net/u012422440)，[简书](http://www.jianshu.com/users/481d9f540fb9/latest_articles)
>本文转载请注明原作者、文章来源，链接，版权归原文作者所有。

&emsp;前段时间阅读了一篇关于Android动画学习的文章《如何学习 Android Animation？》，深感Android动画种类繁复，类型多遍，虽然自己实现过很多动画效果，但是对Android动画的整体领悟还有所欠缺，所以决定最近好好研究一下Android动画的相关内容。
&emsp;我首先决定好好看一下Android属性动画的相关源码，因为平时属性动画用的最多而且涉及的知识点也很多。于是写下这个系列的总结笔记吧，预计一共两篇文章。
&emsp;为了节约你的时间，本文主要内容如下:
- `ValueAnimator`,`PropertyValuesHolder`和`Keyframe`之间的关系
- 使用`ValueAnimator`实现动画的背后原理，本篇不包括`Choreographer`原理的分析。
- `Keyframe`的动画数据计算原理。
- `ObjectAnimator`的属性赋值问题。

&emsp;先上一张Android Animation相关的UML图吧。
![Android Property Animation 类图](http://7xrxif.com1.z0.glb.clouddn.com/2016510-animation-Animation.jpg)
#### **使用ValueAnimator实现动画**
&emsp;我们先来看`ValueAnimator`实现一个Android动画的小例子吧。
``` java
ValueAnimator animation = ValueAnimator.ofFloat(0f, 1f);
animation.setDuration(1000);
animation.start();
```
&emsp;当然，上述并不是全部的代码，你需要使用`AnimatorUpdateListener`来监控`ValueAnimator`来自己实视图属性设置来完成动画。
&emsp;下面，我们就以上述实例为基础，来讲述一下Android属性动画原理。

#### **ValueAnimator的初始化**
&emsp;相信大家在使用`ValueAnimator`或者`ObjectAnimator`的使用，一般都是直接使用其工厂方法来获得对象实例，而不是直接new出来。我们现在就从`ofFloat`这个工厂方法入手，来分析一下`ValueAnimator`的初始化过程。
``` java
public static ValueAnimator ofFloat(float... values) {
    ValueAnimator anim = new ValueAnimator();
    anim.setFloatValues(values);
    return anim;
}
```
&emsp;首先这个函数的参数是一个可变长参数，也就是说你既可以想第一节的例子一样传入两个值，也可以传入三个，四个值，也就是说传入的数值个数是不限的。传入数值个数不同，对其内部初始化过程有很大的影响，具体影响之后会讲到。

&emsp;`ValueAnimator`的构造函数是一个空函数，没有任何内容，所以我们直接调到`setFloatValues`函数。大家要注意的是，这个函数有对应的一系列函数，比如`setIntValues`，但是`setIntValues`会在`ofInt`这个工厂方法中被调用。
``` java
public void setFloatValues(float... values) {
    if (values == null || values.length == 0) {
        return;
    }
    // mValues如果没有初始化或者无值的话
    if (mValues == null || mValues.length == 0) {
        setValues(PropertyValuesHolder.ofFloat("", values));
    } else {
        //否则直接取出第一个位置的对象，进行复制。
        PropertyValuesHolder valuesHolder = mValues[0];
        valuesHolder.setFloatValues(values);
    }
    // New property/values/target should cause re-initialization prior to starting
    mInitialized = false;
}
```
&emsp;上述代码主要是进行`mValues`的初始化或者赋值，并涉及到`PropertyValuesHolder`的初始化过程了，相应得，`PropertyValuesHolder`的`setFloatValues`和`ofFloat`函数也是一系列函数中的一个，用于Float数据的初始化。
&emsp;`PropertyValuesHolder`这个类包含了属性(`Property`)和在动画期间属性的相关数值，它是属性动画中很重要的一个类，基本上属性的反射赋值，属性数值随时间变化的原理都在其中。它的`ofFloat`方法中就是实例化了一个`FloatPropertyValuesHolder`对象，它是`PropertyValuesHolder`的子类。所以我们直接看对应的构造函数及其相关函数。
``` java
private PropertyValuesHolder(String propertyName) {
    mPropertyName = propertyName;
}
public FloatPropertyValuesHolder(String propertyName, float... values) {
    super(propertyName);
    setFloatValues(values);
}

```
&emsp;从代码中可以看出，构造函数只是将传入的属性名赋值给`mPropertyName`,并且调用`setFloatValues`函数，仔细的读者可能会发现发现在`ValueAnimator`的`setFloatValues`函数中也调用了`PropertyValueHoler`的这个函数。
``` java
public void setFloatValues(float... values) {
   //做了两个事情,一是设置了类型,另外一个是生成了Keyframe,关键的类就是Keyframe啦
    mValueType = float.class;
    mKeyframes = KeyframeSet.ofFloat(values);
}
```
&emsp;在`setFloatValues`中，函数记录了数值的类型`float.class`，这个在属性赋值时需要用到，并且使用`KeyframeSet`的`ofFloat`方法实例化了一个`Keyframe`。我们之前说`PropertyValueHoler`负责动画期间数值的变化，其实真正负责的对象是`Keyframe`。
``` java
public static KeyframeSet ofFloat(float... values) {
    int numKeyframes = values.length;
    FloatKeyframe keyframes[] = new FloatKeyframe[Math.max(numKeyframes,2)];
    if (numKeyframes == 1) {
        keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f);
        keyframes[1] = (FloatKeyframe) Keyframe.ofFloat(1f, values[0]);
        ....
    } else {
        keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f, values[0]);
        for (int i = 1; i < numKeyframes; ++i) {
            // ofFloat的第一个参数会从小逐渐变大，最后为1f
            keyframes[i] = (FloatKeyframe) Keyframe.ofFloat(
                                      (float) i / (numKeyframes - 1), values[i]);
            ....
        }
    }
    .....
    return new FloatKeyframeSet(keyframes);
}
```
&emsp;说实话，刚看到这里，我并不是很明白`Keyframe`初始化的传参问题，为什么第一个参数fraction是从小到大最终为1呢？我们可以往看一下。`FloatKeyframe`的工厂方法和构造函数都很简单，只是将传入的参数赋值给成员变量。`FloatKeyframeSet`的构造函数也只是调用了`KeyframeSet`的构造函数。
``` java
public KeyframeSet(Keyframe... keyframes) {
    mNumKeyframes = keyframes.length;
    mKeyframes = Arrays.asList(keyframes);
    mFirstKeyframe = keyframes[0];
    mLastKeyframe = keyframes[mNumKeyframes - 1];
    mInterpolator = mLastKeyframe.getInterpolator();//以最后一个为准
}
```
&emsp;看到这里大家可能感到疑惑，我们好像没有看到`Keyframe`中的`Interpolator`初始化啊，这是怎么回事呢？我检查了代码，它在`Keyframe`中确实没有初始化，而且必须通过`setInterpolator`，而且`KeyframeSet`中也没有进行相应的初始化。
&emsp;至此，`ValueAnimator`的整个初始化就结束了，它将`ValueAnimator`,`PropertyValueHoler`还有`Keyframe`连接起来，三者的职责划分十分清楚明了。

#### **动画start之后**

&emsp;接着，我们就来看start函数调用之后的源码吧。我们直接来看`start(boolean)`这个函数。

``` java
private void start(boolean playBackwards) {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    .....
    //这边是重点，涉及到ThreadLocal的使用
    AnimationHandler animationHandler = getOrCreateAnimationHandler();
    animationHandler.mPendingAnimations.add(this);
    ....
    animationHandler.start();
}
```
&emsp;省略的内容中是一些成员变量的赋值，比如`mPlayingBackwards`，`mSeekFraction`，和`mPlayingState`。大家从字面上就能理解它们的作用，这里就不细说。
&emsp;`getOrCreateAnimationHandler`中涉及到`ThreadLocal`的使用，相比了解`Handler`,`Looper`相关原理的同学都比较了解，这里也不具体讲解，感兴趣的同学可以去学习一下，可以简单的理解为每个线程都可以保存一个单例的一种实现方法。
&emsp;我们着重的了解一下`AnimationHandler`,它是`Runnable`的子类。它的`start`函数中直接调用了`scheduleAnimation`函数。
``` java
private void scheduleAnimation() {
    if (!mAnimationScheduled) {
        mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, this, null);
        mAnimationScheduled = true;
    }
}
```
&emsp;这里涉及了另外一个比较重要的类`Choreographer`，我们将在写一篇文章中详细说明这个类的代码，现在大家只要知道使用`postCallback`注册了这个类之后，它会协调不同动画或者其他活动的时间，并每隔一定时间之后调用该`Runnable`的`run`方法,并且`getFrameTime`可以获得调用时的时间，需要注意的是这个时间不一定是`System.nanoTime`。
&emsp;在`run()`函数里调用了`doAnimationFrame`并将`Choreographer`的`getFrameTime`获得的时间值传递进去。
``` java
final boolean doAnimationFrame(long frameTime) {
    if (mPlayingState == STOPPED) {
        mPlayingState = RUNNING;
        ...
    }
    if (mPaused) {
        ....
        return false;
    } else if (mResumed) {
      ....
    }
    final long currentTime = Math.max(frameTime, mStartTime);
    return animationFrame(currentTime);
}
```
&emsp;在`doAnimationFrame`中，程序会根据`mPlayingState`做一些相应的操作，并改变状态值，最后调用`animationFrame`。
```java
boolean animationFrame(long currentTime) {
    boolean done = false;
    switch (mPlayingState) {
    case RUNNING:
    case SEEKED:
        float fraction = mDuration > 0 ? (float)(currentTime - mStartTime) / mDuration : 1f;
        ....
        animateValue(fraction);
        break;
    }
    return done;
}
```
&emsp;我们可以看到，这里通过开始时间，当前事件和每帧动画间隔事件求出了一个`fraction`,然后代码中的省略部分主要是处理动画重复播放和倒着播发时的`fraction`处理，最后调用`animateValue`函数。
``` java
    void animateValue(float fraction) {
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);//ViewPropertyViewHolder
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                //这边就结束啦，直接传递给AnimationUpdateListener
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```
&emsp;这个函数是理解动画流程的关键之一，你首先可以看到`Interpolator`的使用，这个参数传入的`fraction`是通过动画开始时间和当前时间计算出来的，而通过不同`Interpolator`进行转换，无论是`BounceInterpolator`还是`LinerInterpolator`。接着，`ValueAnimator`调用了`mValues`保存的所有`ViewPropertyViewHolder`的`calculateValue`方法，让其去计算动画需要的数据。最后通过`AnimatorUpdateListenters`通知用户，然后我们会在回调函数中进行响应的操作。`ObjectAnimator`会override这个函数，然后加上属性反射赋值的相关逻辑。我们先来看如何计算动画所需的数值吧。
#### **动画所需数值计算**
``` java
@Override
void calculateValue(float fraction) {
    mFloatAnimatedValue = mFloatKeyframes.getFloatValue(fraction);
}
```
&emsp;从代码上来看，`FloatViewPropertyViewHolder`将计算动画所需的数值的任务又交给了`Keyframe`,这也符合二者的职责分配。我们按照之前的分析，来看一下`FloatKeyframeSet`的响应操作。`getValue`中直接调用了`getFloatValue`函数，而后者是了解动画相关数值计算的关键函数。
``` java
@Override
public float getFloatValue(float fraction) {
  //分为两种情况,一个是2 Keyframe时一般就是你给出两个值进行初始化,另外一个是多个keyframe时
    if (mNumKeyframes == 2) {
        if (firstTime) {
            firstTime = false;
            firstValue = ((FloatKeyframe) mKeyframes.get(0)).getFloatValue();
            lastValue = ((FloatKeyframe) mKeyframes.get(1)).getFloatValue();
            deltaValue = lastValue - firstValue;
        }
        //如果KeyframeSet也有Interpolator
        if (mInterpolator != null) {
            fraction = mInterpolator.getInterpolation(fraction);
        }
        //如果有Evaluator，那么使用它计算，如果没有就使用默认计算方法。
        if (mEvaluator == null) {
            return firstValue + fraction * deltaValue;
        } else {
            return ((Number)mEvaluator.evaluate(fraction, firstValue, lastValue)).floatValue();
        }
    }
    //处理多个Keyframe的情况
    ..... //省略代码处是处理fraction<=0和fraction>=1的特殊情况
    FloatKeyframe prevKeyframe = (FloatKeyframe) mKeyframes.get(0);
    for (int i = 1; i < mNumKeyframes; ++i) {
        FloatKeyframe nextKeyframe = (FloatKeyframe) mKeyframes.get(i);
        //找到第一个可以处理这个fraction的keyframe。
        if (fraction < nextKeyframe.getFraction()) {
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            //在构造Keyframe时，我们知道第i个Keyframe的fraction就是i/(总个数)
            //所以下边这个计算公式就是将整个动画区间的fraction转变成prevKeyframe到nextKeyframe
            //这个小区间中的fraction
            float intervalFraction = (fraction - prevKeyframe.getFraction()) /
                (nextKeyframe.getFraction() - prevKeyframe.getFraction());
            //用于计算动画数值的值区间是前一个Keyframe的value和当前的value
            float prevValue = prevKeyframe.getFloatValue();
            float nextValue = nextKeyframe.getFloatValue();
            //默认计算方式或者使用Evaluator进行计算
            return mEvaluator == null ?
                    prevValue + intervalFraction * (nextValue - prevValue) :
                    ((Number)mEvaluator.evaluate(intervalFraction, prevValue, nextValue)).
                        floatValue();
        }
        prevKeyframe = nextKeyframe;
    }
    return ((Number)mKeyframes.get(mNumKeyframes - 1).getValue()).floatValue();
}
```
&emsp;代码中有很详细的注释，大家自己阅读吧，理解这段代码，感觉对Android动画框架的理解十分的重要，你会对其中动画数值的计算有更加深刻的理解。

#### **ObjectAnimator的属性赋值**
&emsp;如果你使用`ObjectAnimator`来实现动画的话，我们可以验证上述关于`ValueAnimator`的梳理路线继续走下去，动画所需的数值已经计算完全，之后就是对属性继续赋值了。我们来看`ObjectAnimator`的`animateValue`函数。
``` java
@Override
void animateValue(float fraction) {
    final Object target = getTarget();
    //target是一个WeakReference，使用前需要检查。
    if (mTarget != null && target == null) {
        cancel();
        return;
    }
    super.animateValue(fraction);
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        //这是使用反射将值赋给对象的啦.
        mValues[i].setAnimatedValue(target);
    }
}
```
&emsp;这个函数首先调用了父函数，进行计算数值，然后遍历`mValues`来设置对象的属性值。我们直接看`FloatPropertyValuesHolder`的`setAnimatedValue`函数。当然不同的`PropertyValuesHolder`子类的这个函数的实现都是差不多，只是涉及的数据类型不同。
``` java
void setAnimatedValue(Object target) {
    if (mFloatProperty != null) {
        mFloatProperty.setValue(target, mFloatAnimatedValue);
        return;
    }
    if (mProperty != null) {
        mProperty.set(target, mFloatAnimatedValue);
        return;
    }
    if (mJniSetter != 0) {
        nCallFloatMethod(target, mJniSetter, mFloatAnimatedValue);
        return;
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = mFloatAnimatedValue;
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```
&emsp;通过上述代码，`ObjectAnimator`就将每一次计算而来的数值赋值给View的对象的相应属性上，形成了动画。

#### **后记**
&emsp;本文介绍了Android属性动画的基本框架流程，为的是让大家更好的理解Andorid动画。但是Android动画这一类的知识还是需要多多实践和练习才能发挥其正真的价值，所以，希望自己之后写出几个比较好的动画效果在来和大家进行分享吧。

