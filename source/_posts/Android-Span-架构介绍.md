---
title: Android Span 架构介绍
date: 2016-08-05 21:48:15
tags: 
	- Span
categories:
	- Android
---

&emsp;前几天在微博上看到一个人评论Android Span机制相当强大，有必要细心研究一下，于是就google了一下，发现了一篇很好的文章[Spans, a Powerful Concept.](http://flavienlaurent.com/blog/2014/01/31/spans/)，然后以这篇文章为基础研究了一下Android Span的用法。于是决定写几篇文章来总结一下。

### Android Span 介绍
&emsp;对于`Span`，大家首先想起的应用场景就是同一个`TextView`中显示不同颜色的字体。对于这种需求，我们会用到`ForegroundColorSpan`
``` java
ForegroundColorSpan span = new ForegroundColorSpan(Color.RED);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
&emsp;上面这段代码可能就是大家对Android Span的全部了解了，但是Android Span有着和Drawable同样强大的能力，希望我的这几篇文章可以带领大家初探Android Span的强大能力。
### Android Span 框架
&emsp;Android Span 框架涉及的类遵循一下四个定义规则:

- 如果一个`Span`影响字符层次上的文字格式，那么它继承`CharacterStyle`类。
- 如果一个`Span`影响段落层次上的文字格式，那么它实现`ParagraphStyle`接口。
- 如果一个`Span`修改字符层次上的文字外观，那么它实现`UpdateAppearance`接口。
- 如果一个`Span`修改字符层次上的度量或者尺寸，那么它实现`UpdateLayout`接口。
&emsp;虽然有以上四个规则，但是我们自定义Span时并不需要直接继承或者实现这些类或者接口，而是实现其他更加具体的类。
&emsp;遵循这些规则，Android Span框架相关类图如下：
![CharacterStyle相关类图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-cdcharacterstyle.png)

![ParagraphStyle相关类图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-cdparagraphstyle.png)

![UpdateAppearance相关类图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-cdupdateappearance.png)

![UpdateLayout相关类图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-cdupdatelayout.png)


### Android Span类型介绍
&emsp;Android Span的相关原理和机制以后再讲，我们先来了解一下当前Android预设的`Span`类型的用法吧。

####　**android.text.style.BulletSpan**
&emsp;`BulletSpan`影响**段落层次**文字的格式，它让你在段落开头添加一个黑圆点。
```
BulletSpan span = new BulletSpan(15,Color.RED);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,1,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![BulletSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-BulletSpan.png)

####　**android.text.style.QuoteSpan**

&emsp;`QuoteSpan`影响**段落层次**文字的格式，它可以在段落前边添加一个竖直的引用线。

```
QuoteSpan span = new QuoteSpan(Color.RED);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![QuoteSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-QuoteSpan.png)

####　**android.text.style.AlignmentSpan**

&emsp;`AlignSpan.Standard`影响**段落层次**文字的格式，它允许你控制段落的对齐方式，有居中对齐，右侧对齐和左侧对齐。
```
AlignmentSpan.Standard span = new AlignmentSpan.Standard(Layout.Alignment.ALIGN_CENTER);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,0,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);

```
![AlignSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-AliginSpan.png)


####　**android.text.style.UnderlineSpan**

&emsp;`UnderlineSpan`影响**字符层次**文字的格式，它可以在文字下方添加下划线。其内部实现其实就是使用`Paint.setUnderlineText(true)`实现的。

```
UnderlineSpan span = new UnderlineSpan();
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length(),Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![UnderlineSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-UnderlineSpan.png)


####　**android.text.style.StrikethroughSpan**

&emsp;`StrikethroughSpan`影响**字符层次**上的文字的格式，它允许你在文字上添加删除线。它内部使用` Paint.setStrikeThruText(true))`来实现。

```
StrikethroughSpan span = new StrikethroughSpan();
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length(),Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![StrikethroughSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-StrikeSpan.png)

####　**android.text.style.SubscriptSpan**

&emsp;`SubscriptSpan`影响**字符层次**上的文字的格式，它允许你把文字作为下标进行显示。

```
SubscriptSpan span = new SubscriptSpan();
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);

```
![SubscriptSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-SubscriptSpan.png)


####　**android.text.style.SuperscriptSpan**

&emsp;``影响**字符层次**上的文字的格式，它允许你把文字作为上标进行显示。

```
SuperscriptSpan span = new SuperscriptSpan();
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);

```
![SuperscriptSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-SupscriptSpan.png)

####　**android.text.style.BackgroundColorSpan**

&emsp;`BackgroundColorSpan`影响**字符层次**上的文字的格式，你可以使用它设置文字的背景颜色。

```
BackgroundColorSpan span = new BackgroundColorSpan(Color.RED);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![BackgroundColorSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-BackgroundSpan.png)

####　**android.text.style.ForegroundColorSpan**

&emsp;`ForegroundColorSpan`影响**字符层次**上的文字的格式，你可以使用它设置文字的自己的颜色。

```
ForegroundColorSpan span = new ForegroundColorSpan(Color.RED);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```

![ForegroundColorSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-ForegroundSpan.png)

####　**android.text.style.ImageSpan**

&emsp;`ImageSpan`影响**字符层次**上的文字的格式，你可以使用它在文字中间添加图片。

```
ImageSpan span = new ImageSpan(this,R.mipmap.ic_launcher);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,1,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```

![ImageSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-ImageSpan.png)

####　**android.text.style.StyleSpan**

&emsp;`StyleSpan`影响**字符层次**上的文字的格式。它允许你设置文字的类型(bold, italic, normal)

```
StyleSpan span = new StyleSpan(Typeface.BOLD);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![StyleSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-StyleSpan.png)

####　**android.text.style.TypefaceSpan**

&emsp;`TypefaceSpan`影响**字符层次**上的文字的格式。它允许你设置文字的字体族(monospace, serif等)

```
TypefaceSpan span = new TypefaceSpan("serif");
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```

![TypefaceSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-TypefaceSpan.png)

####　**android.text.style.AbsoluteSizeSpan**

&emsp;`AbsoluteSizeSpan`影响**字符层次**上的文字的格式。它允许你设置文字的绝对字体大小。

```
AbsoluteSizeSpan span = new AbsoluteSizeSpan(24,true);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![AbsoluteSizeSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-AbsolutSizeSpan.png)

####　**android.text.style.RelativeSizeSpan**

&emsp;`RelativeSizeSpan`影响**字符层次**上的文字的格式。它允许你设置文字的相对字体大小。

```
RelativeSizeSpan span = new RelativeSizeSpan(2.0f);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![RelativeSizeSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-RelativeSizeSpan.png)

####　**android.text.style.ScaleXSpan**

`&emsp;`ScaleXSpan`影响**字符层次**上的文字的格式。它让你让文字在ｘ方向上进行缩放。
```
ScaleXSpan span = new ScaleXSpan(2.0f);
SpannableString spannableString = new SpannableString(CONTENT);
spannableString.setSpan(span,0,spannableString.length()/2,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTvTextView.setText(spannableString);
```
![ScaleXSpan效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-ScaleSpan.png)

####　**android.text.style.MaskFilterSpan**

&emsp;`MaskFilterSpan`影响**字符层次**上的文字的格式。它让你在文字上设置`android.graphics.MaskFilter`

```
//Blur a character
span = new MaskFilterSpan(new BlurMaskFilter(density*2, BlurMaskFilter.Blur.NORMAL));
//Emboss a character
span = new MaskFilterSpan(new EmbossMaskFilter(new float[] { 1, 1, 1 }, 0.4f, 6, 3.5f));

```

![BlurMaskFilter效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-maskfilterblur.png)

![EmbossMaskFilter效果图](http://7xrxif.com1.z0.glb.clouddn.com/20160805-Span-maskfilteremboss.png)


### 后记
&emsp;本篇文章只是简单介绍Android Span的使用，接下来的文章会介绍Span更加高级的用法和它的原理。希望大家继续支持。
