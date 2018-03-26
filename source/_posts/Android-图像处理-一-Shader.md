---
title: 'Android 图像处理(一) : Shader'
date: 2016-08-26 15:09:18
tags: 
	- Shader
	- 图像处理
categories:
	- Android
---

&emsp;之前一段时间，我都在研究Android自定义View的相关知识，随着逐渐的深入，渐渐了解到了一些Android图像处理的知识，主要是`Bitmap`,`Canvas`,`Shader`,`Matric`,`ColorFilter`和`Xfermode`的使用。所以准备写一系列文章介绍一下这些方面的知识。

#### 图像处理相关概念介绍
&emsp;要想了解Shader的概念，首先要了解Android图像处理中几个比较重要的概念：bitmap,canvas,drawing primitive,paint。需要注意的是，上述四个词并不指android中的类，而是四个概念。
&emsp;bitmap指画布。画家画图时都需要一块画布，然后才会在画布上绘制各种形状和颜色，Android中的`Bitmap`就有画布的功能。比如下面这段代码。
``` java

    static Bitmap makeSrc(int w,int h) {
            Bitmap bm = Bitmap.createBitmap(w,h, Bitmap.Config.ARGB_8888);
            Canvas c = new Canvas(bm);
            Paint p = new Paint(Paint.ANTI_ALIAS_FLAG);
    
            p.setColor(0xFF66AAFF);
            c.drawRect(100,100,100,100,p);
            return bm;

}
```
&emsp;Bitmap这里就充当一个画布的作用，之后Canvas的操作都是在这个Bitmap画布进行绘制。
&emsp;canvas是绘制操作的集合，你可以把它理解成画家的手，通过它你可以在画布上绘制各种图像和颜色。相应的在Android的`Canvas`类中就封装了绘制相关的各种操作，比如`drawXXX`系列。
&emsp;drawing primitive指绘制实体或者原始内容，比如画家绘画时可能会画一个苹果，也可能只是绘制一些线段，矩形等。这些都是绘制实体。在Android中，`Path`,`Rect`,`Text`,`Bitmap`都可以充当drawing primitive的角色，因为你都可以把它们绘制到画布上。你也可以这样理解，`Canvas`中的`drawXXX`系列函数中的XXX都是指drawing primitive。
&emsp;paint就是指画家的神奇画笔啦。Android中的`Paint`类就充当这个角色，我们可以通过`Paint`来配置画笔的颜色，粗细，纹理等。我们系列文章主要讲解的`Shader`,`ColorFilter`,`Xfermode`三个类都和`Paint`直接相关。

#### Shader概念介绍
&emsp;Shader,也就是所谓的着色器。它应用于计算机图形学领域，指一组提供计算机图像资源在执行绘制任务时使用的指令，**用于计算图像的颜色和明暗**。从技术角度来看，着色器是渲染器的一部分，它负责计算目标的颜色。以我的理解，`Paint`就是渲染器，`Paint.setShader`这个函数展示了着色器和渲染器之间的关系。下面这段代码显示了`Shader`的使用方式。
``` java

    String text = "Shader";
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.photo);
    BitmapShader bitmapShader = new BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
    Paint paint = new Paint();
    paint.setShader(bitmapShader);
    paint.setTextSize(100);
    paint.setStrokeWidth(50);
    canvas.drawText(text,0,text.length(),150,400,paint);

```
&emsp;上边的代码片段使用了`BitmapShader`，也就是说使用图片上相应位置的颜色进行着色。就相当于使用这个图片作为背景，字体颜色是透明色，字体在(150,400)位置进行绘制，字体最终显示的颜色就是图片中相应位置的颜色。
&emsp;`Shader.TileMode`是指平铺模式，一共有三种类型：`CLAMP`,`MIRROR`,`REPEAT`。构造函数中后两个参数分别规定了当绘制实体的x或ｙ轴范围超过一定范围时(比如`BitmapShader`中`Bitmap`的大小，`LinearGradient`中构造函数传入的x,y值)，着色器填充的行为。我们以`BitmapShader`为例，来讲述三种模式的区别。原图如下所示，来自于[这个博客](http://www.cnblogs.com/tianzhijiexian/p/4298660.html)。

![使用的bitmap原图](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-img.png)

```

    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.img);
    BitmapShader bitmapShader = new BitmapShader(bitmap, Shader.TileMode.XXX, Shader.TileMode.XXXX);
    Paint paint = new Paint();
    paint.setShader(bitmapShader);
    canvas.drawColor(Color.GRAY);
    canvas.drawRect(0,0,800,800,paint);

```

&emsp;`Shader.TileMode.CLAMP`是边缘拉伸模式，它会拉伸边缘的一个像素来填充其他区域。效果图如下所示，我们可以看到，超出bitmap大小的范围都被图片边缘的颜色所填充。

![clamp效果图](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-clamp.png)

&emsp;`Shader.TileMode.MIRROR`是镜像模式，也就是说它通过镜像变化来填充其他区域。需要注意的是，**TileMode是先处理y轴方向的操作，然后在进行x轴方向的操作。也就是说先进行y轴方向的镜像操作，然后在进行x轴方向上的镜像操作**。不过我觉得好像先进行哪个方向上的操作并没有什么区别，最终效果是一样的。其效果图如下所示:
![mirror效果图](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-mirror.png)
&emsp;`Shader.TileMode.REPEAT`是重复模式，通过复制来填充其他区域
![repeat效果图](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-repeta.png)
&emsp;你也可以对x,y轴使用不同的`TileMode`,可以得到不同的效果。
![clamp+mirro效果图](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-clammirror.png)

### Shader类型介绍
&emsp;`BitmapShader`的用法很简单，上边的实例中也有用到，所以我们就不再详细介绍它了。
#### LinearGradient
&emsp;`LinearGradient`是颜色线性渐变的着色器。它有两个构造函数：
&emsp;双色渐变的构造函数是
>`public LinearGradient (float x0, float y0, float x1, float y1, int color0, int color1, Shader.TileMode tile)`

&emsp;参数中,（x0,y0）表示渐变的起点，（x1,y1）表示渐变的终点坐标，这两点都是相对于屏幕坐标系。color0,color1分别表示起点的颜色和终点的颜色。不同于`BitmapShader`需要x,y轴两个平铺模式参数，`LinearGradient`只需要一个`TileMode`，表示**从(x0,y0)到(x1,y1)方向上的平铺模式**。
`LinearGradient`的平铺效果可能比较难以理解，需要大家多做几次实验才能明白。
&emsp;下图是200*200的由红色到黄色的线性渐变效果着色器绘制200*200的矩形的效果图,着色器的`TileMode`为`MIRROR`:
![linegradient1](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-linegradient1.png)
&emsp;下图是上述的着色器绘制400*400的效果图,我们可以看到二者的区别，在屏幕坐标轴的(200,200)方向(第二张图中的黑色线段所代表的方向)上颜色是先从红色到黄色，超过(200,200)这个点之后，由于设置平铺模式为`MIRROR`所以颜色就又从黄色逐渐变成了红色。这个对角线外上的点的颜色是和其在对角线上(-200,200)方向上的投影点的颜色相同，也就是第二张图中蓝色直线上点的颜色都是相同的。
![linegradient2](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-linegradient2.png)
![linegradient5](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-linegradient5.png)

&emsp;多色渐变的构造函数是
>`public LinearGradient (float x0, float y0, float x1, float y1, int[] colors, float[] positions, Shader.TileMode tile)`
&emsp;多色渐变的构造函数中，我们可以传入多个颜色，和每个颜色的占比。而且当positions参数传入null时，代表颜色是均匀的填充整个渐变区域的，显得比较柔和。
![linegradient6](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-linegrandient6.png)

#### SweepGradient
&emsp;`SweepGradient`是梯度渐变，也称为扫描式渐变，效果有点类似与雷达扫描效果。它也有两个构造方法，分别为两色渐变和多色渐变，二者的区别和之前介绍的`LinearGradient`的构造函数类似，我们这里就只介绍双色渐变的构造函数：
> `public SweepGradient(float cx, float cy, int color0, int color1)`
&emsp;在构造函数的参数中，(cx,cy)表示渐变效果的中心点，也就是雷达扫描的圆点。color0和color1表示渐变的起点色和终点色。颜色渐变是顺时针的，从中心点的x轴正方形开始。需要注意的是，这里构造函数并不需要`TileMode`,因为梯度渐变的边界相当于无限大的。
![sweepgradient](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-sweepgradient.png)
#### RadialGradient
&emsp;`RadialGradient`是径向渐变，径向渐变就是从圆的中心点向四周渐变的特效，你可以把它想象成一个圆，从圆点到圆周沿着半径有颜色的渐变效果。它也有两个构造函数，我们只介绍双色渐变的构造函数。
>`public RadialGradient(float centerX, float centerY, float radius,
            int centerColor, int edgeColor, TileMode tileMode)`

&emsp;在构造函数的参数中，(centerX,centerY)表示径向渐变中的圆点，radius表示圆的半径长度，centerColor表示圆点颜色，edgeColor表示圆周颜色，titleMode表示平铺模式，**它的作用方向是沿着半径方向**。下图就是一个有红色到黄色的径向渐变，平铺模式为CLAMP,所以超出圆周的范围的点的颜色就都是圆周的颜色黄色。
![radialsweep](http://7xrxif.com1.z0.glb.clouddn.com/201608026-shader-radialgradient.png)

#### ComposeShader
&emsp;`ComposeShader`用来组合不同的Shader,可以将两个不同的Shader组合在一起，它有两个构造函数:
``` java
ComposeShader (Shader shaderA, Shader shaderB, Xfermode mode)  
ComposeShader (Shader shaderA, Shader shaderB, PorterDuff.Mode mode)  
```
&emsp;二者的区别就是一个使用PorterDuff的模式，另一个可以使用所有的Xfermode。这两个混合模式我们在之后的文章中将会详细讲到。


#### 后记
&emsp;这里只是简单介绍了一下Shader的概念和基本用法，之后会多加一些实际使用的情况，希望大家持续关注。
