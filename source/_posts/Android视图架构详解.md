title: Android视图架构详解
date: 2016-04-17 14:56:23
tags: 
	- View Architecture
categories:
	- Android
---

作者： [ztelur](http://www.jianshu.com/users/481d9f540fb9/latest_articles)
联系方式：[segmentfault](https://segmentfault.com/u/remcarpediem)，[csdn](http://blog.csdn.net/u012422440)，[github](http://ztelur.github.io/)
>本文仅供个人学习，不用于任何形式商业目的，转载请注明原作者、文章来源，链接，版权归原文作者所有。


&emsp;最近一直在研究`View`的绘制相关的机制，发现需要补充一下Android View Architecture的相关知识，所以就特地研究了一下这方面的代码，写成本篇文章
&emsp;为了节约你的时间，本篇文章内容大致如下:

- `Activity`，`DecorView`，`PhoneWindow`和`ViewRoot`的作用和相关关系

### Android View Architecture
&emsp;先来几张图，大致展现一下Android 视图架构的大概。

> 感谢网友提醒，泛化和实现这两种关系的箭头画反啦。以后要仔细学一遍UML了，平时经常画，如果有错误可真是闹笑话啊。

![View Architecutre类图](http://7xrxif.com1.z0.glb.clouddn.com/2016417-view-View%20Architecutre%E7%B1%BB%E5%9B%BE.jpg)


![视图关系图.png](http://7xrxif.com1.z0.glb.clouddn.com/2016417-view-%E8%A7%86%E5%9B%BE%E5%85%B3%E7%B3%BB%E5%9B%BE.png)


![View树状结构图.png](http://7xrxif.com1.z0.glb.clouddn.com/2016417-view-View%E6%A0%91%E7%8A%B6%E7%BB%93%E6%9E%84%E5%9B%BE.png)
### Activity和Window
&emsp;总所周知,Activity并不负责视图控制，它只是控制生命周期和处理事件，真正控制视图的是`Window`。一个Activity包含了一个Window，Window才是真正代表一个窗口，也就是说Activity可以没有Window，那就相当于是Service了。在`ActivityThread`中也有控制`Service`的相关函数或许正好印证了这一点。
&emsp;`Activity`和`Window`的第一次邂逅是在`ActivityThread`调用`Activity`的`attach()`函数时。
``` java
//[window]:通过PolicyManager创建window,实现callback函数,所以,当window接收到
//外界状态改变时,会调用activity的方法,
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    ....
    mWindow = PolicyManager.makeNewWindow(this);
    //当window接收系统发送给它的IO输入事件时,例如键盘和触摸屏事件,就可以转发给相应的Activity
    mWindow.setCallback(this);
    .....
    //设置本地窗口管理器
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    .....
}
```
&emsp;在`attach()`中，新建一个`Window`实例作为自己的成员变量，它的类型为`PhoneWindow`,这是抽象类`Window`的一个子类。然后设置`mWindow`的`WindowManager`。

### Window,Activity和DecorView
&emsp;`DecorView`是`FrameLayout`的子类，它可以被认为是Android视图树的根节点视图。`DecorView`作为顶级View，一般情况下它内部包含一个竖直方向的`LinearLayout`，在这个LinearLayout里面有上下两个部分（具体情况和Android版本及主体有关），上面的是标题栏，下面的是内容栏。在Activity中通过setContentView所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是content，在代码中可以通过ViewGroup content = （ViewGroup)findViewById(R.android.id.content)来得到content对应的layout。
&emsp;`Window`中有几个视图相关的比较重要的成员变量如下所示:

- `mDecor`:`DecorView`的实例，标示`Window`内部的顶级视图
- `mContentParent`:`setContetView`所设置的布局文件就加到这个视图中
- `mContentRoot`:是`DecorView`的唯一子视图，内部包含`mContentParent`,标题栏和状态栏。

&emsp;Activity中不仅持有一个`Window`实例，还有一个类型为`View`的`mDecor`实例。这个实例和`Window`中的`mDecor`实例有什么关系呢？它又是什么时候被创建的呢？
&emsp;二者其实指向同一个对象，这个对象是在`Activity`调用`setContentView`时创建的。我们都知道`Activity`的`setContentView`实际上是调用了`Window`的`setContentView`方法。
``` java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) { //[window]如何没有DecorView,那么就新建一个
        installDecor(); //[window]
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    ....
    //[window]第二步,将layout添加到mContentParent
    mLayoutInflater.inflate(layoutResID, mContentParent);
    .....
}
```
&emsp;代码很清楚的显示了布局文件的视图是添加到`mContentParent`中，而且`Window`通过`installDecor`来新建`DecorView`。
``` java
//[window]创建一个decorView
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(); //直接new出一个DecorView返回
        ....
    }
    if (mContentParent == null) {
        //[window] 这一步也是很重要的.
        mContentParent = generateLayout(mDecor); //mContentParent是setContentVIew的关键啊
        .....
    }
    ....
}
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.
    .......
    //[window] 根据不同的style生成不同的decorview啊
    View in = mLayoutInflater.inflate(layoutResource, null);
    // 加入到deco中,所以应该是其第一个child
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in; //给DecorView的第一个child是mContentView
    // 这是获得所谓的content 
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    }
    .....
    return contentParent;
}

```
&emsp;从上述的代码中，我们可以清楚的看到`mDecor`和`mContentParent`和`mContentRoot`的关系。
&emsp;那么，`Activity`中的`mDecor`是何时被赋值的？我们如何确定它和`Widnow`中的`mDecor`指向同一个对象呢？我们可以查看`ActivityThread`的`handleResumeActivity`函数，它负责处理`Activity`的`resume`阶段。在这个函数中，Android直接将`Window`中的`DecorView`实例赋值给`Activity`。
``` java
final Activity a = r.activity;
r.window = r.activity.getWindow();
View decor = r.window.getDecorView();
decor.setVisibility(View.INVISIBLE);
ViewManager wm = a.getWindowManager();
WindowManager.LayoutParams l = r.window.getAttributes();
a.mDecor = decor;
```


### Window，DecorView 和 ViewRoot
&emsp;`ViewRoot`对应`ViewRootImpl`类，它是连接`WindowManagerService`和`DecorView`的纽带，View的三大流程(测量（measure），布局（layout），绘制（draw）)均通过ViewRoot来完成。`ViewRoot`并不属于View树的一份子。从源码实现上来看，它既非View的子类，也非View的父类，但是，它实现了`ViewParent`接口，这让它可以作为`View`的名义上的父视图。`RootView`继承了`Handler`类，可以接收事件并分发，Android的所有触屏事件、按键事件、界面刷新等事件都是通过ViewRoot进行分发的。**ViewRoot可以被理解为“View树的管理者”——它有一个mView成员变量，它指向的对象和上文中`Window`和`Activity`的`mDecor`指向的对象是同一个对象**。

&emsp;我们来先看一下`ViewRoot`的创建过程。由于`ViewRoot`作为`WindowMangerService`和`DecorView`的纽带，只有在`WindowManager`将持有`DecorView`的`Window`添加进窗口管理器才创建。我们可以查看`WindowMangerGlobal`中的`addView`函数。对`WindowManager`不太熟悉的同学可以参考[《Window和WindowManager解析》](http://ztelur.github.io/2015/10/28/Window%E5%92%8CWindowManager%E8%A7%A3%E6%9E%90/)
``` java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
        // 创建ViewRootImpl,然后将下述对象添加到列表中
    ....
    root = new ViewRootImpl(view.getContext(), display);

    view.setLayoutParams(wparams);

    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
    ....
    try {
        // 添加啦!!!!!!!!这是通过ViewRootImpl的setView来完成，这个View就是DecorView实例
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
      ....
    }
    ....
}
```
&emsp;那么，`Window`是什么时候被添加到`WindowManager`中的呢？我们回到`ActivityThread`的`handleResumeActivity`函数。我们都知道Activity的resume阶段就是要显示到屏幕上的阶段，在Activity也就是`DecorView`将要显示到屏幕时，系统才会调用`addView`方法。
&emsp;我们在`handleResumeActivity`函数中找到了下面一段代码,它调用了`Activity`的`makeVisible()`函数。
``` java
// ActivityThread
r.activity.makeVisible();

//Activity
    //[windows] DecorView正式添加并显示
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```
&emsp;我们通过源代码发现，正式在`makeVisible`函数中，系统进行了`Window`的添加。


引用
http://wiki.jikexueyuan.com/project/deep-android-v1/surface.html
http://blog.csdn.net/guxiao1201/article/details/41744107
http://forlan.iteye.com/blog/2269381
