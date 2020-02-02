---
title: 关于Android MVP模式的思考
tags:
  - 框架
categories:
  - Android
abbrlink: eca06a79
date: 2017-03-12 17:09:22
---

 这一周对现有的Android项目进行了框架重构，使用MVP模式来重新构建整个项目和包结构。今天就来总结一下我在这个过程中理解和实践吧。

### MVP概述
 MVP是指Model，View和Presenter的缩写，是MVC模式的一种改进版。MVP是一种非常适合Android应用的开发模式，它**将把逻辑相关代码从presentation Layer中分离出去**，所以，所有界面应该显示什么和界面如何显示这些是相互分离的，  在理想状态下，MVP模式可以随意切换视图显示的形式。
 但是，需要首先声明的是，MVP并不是**一个框架模式**，它只负责presentation Layer。在Android项目中，还可能存在Domain Layer和Data Layer，它们分别负责业务逻辑和数据存储。关于三个layer的讨论，可以查看[clear Architecture](https://github.com/android10/Android-CleanArchitecture)。


![mvp.png](http://7xrxif.com1.z0.glb.clouddn.com/2017312-mvp-mvp.png)


#### The presenter
 The presenter充当的是view和model模块中间人的角色。它从model模块获得数据并发送给view模块。**但是不同于MVC，presenter模块也决定用户和界面交互时界面如何响应**。

#### The View
 View模块，一般由Activity或者Fragment实现，每个View实例一般都包含一个presenter实例的引用，并负责实例化presenter。理想情况下，你可以使用dagger来实现依赖注入。view的任务就是当用户操作界面时，调用presenter实例的功能来进行响应。

#### The Model
 在设计良好的分层架构中，model应该只是domain layer和业务逻辑的入口，比如说，[clear architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html)。但是你也可以把所有界面显示无关的逻辑都作为model。

### MVP示例
 关于MVP架构的实例有很多，比如[google官方架构](https://github.com/googlesamples/android-architecture)。这里我就给出一下我重构项目一个模块的类图，然后大致说一下实现。

![webwxgetmsgimg.jpg](http://7xrxif.com1.z0.glb.clouddn.com/2017312-mvp-webwxgetmsgimg.jpg)

&emsp;首先定义了`BaseView`和`BasePresenter`两个接口，实现了`BaseActivity`和`BaseFragment`两个基础类。需要注意的是，因为每个View都需要持有一个特定类型的presenter对象，所以，在`BaseView`定义时，使用了范型。除此之外，你可以在`initPresenter`函数中构造出所需要的presenter对象。`showLoading`和`stopLoading`函数是所有视图进行网络数据加载时所需要的方法，其具体使用场景后边可以看到，这里这两个接口是在`BaseActivity`中实现的。

``` java
public interface BaseView<T> {
    void showLoading();
    void stopLoading();
    void initPresnter();
    T getPresenter();
}

public interface BasePresenter {

}

public class BaseActivity extends Activity{
    public void showLoading() {
    }

    public void stopLoading() {
    }
}

public class BaseFragment extends Fragment {
    public void showLoading() {

    }

    public void stopLoading() {

    }
}
```
&emsp;然后是data模块中的相关类和接口的定义。我们在`DataContract`接口中定义所有相关的view和data。
``` java
public interface DataContract {
    interface DataView extends BaseView<DataPresenter> {
        void showFragment(BaseFragment fragment);
    }
    interface DataPresenter extends BasePresenter {
        void switchToDataList();
        void switchToDataTree();
    }
    interface DataListView extends BaseView<DataListPresenter> {
        void showDataList(List<String> data);
    }
    interface DataListPresenter extends BasePresenter {
        void loadDataList();
    }
    interface DataTreeView extends BaseView<DataTreePresenter> {}
    interface DataTreePresenter extends BasePresenter {}
}
```
&emsp;在`DataContract`接口中，我们可以清晰的看到这个模块中所有的view和presenter类型和他们的接口。这样无疑可以帮助其他程序员快速了解本模块的业务逻辑。下面，我们来看一下`DataListFragmnet`和`DataListPresenter`的实现。

``` java
public class DataListFragment extends BaseFragment implements DataContract.DataListView {
    private DataContract.DataListPresenter mPresenter;
    @Override
    public void initPresnter() {
        mPresenter = new DataListPresenter(this);
    }
    @Override
    public DataContract.DataListPresenter getPresenter() {
        return mPresenter;
    }
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        initPresnter();
        //需要加载数据或者接收到界面点击事件时，调用presenter接口进行初始化或者响应。
        mPresenter.loadDataList();
    }
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_data_list, container, false);
    }
    @Override
    public void showDataList(List<String> data) {
        //将数据传给adapter等数据展示的视图
    }
}

public class DataListPresenter implements DataContract.DataListPresenter {
    private DataContract.DataListView mView;
    public DataListPresenter(DataContract.DataListView view) {
        super();
        this.mView = view;
    }

    @Override
    public void loadDataList() {
        //调用baseview的接口，这些接口实现在baseFragment或者BaseActivity中
        mView.showLoading();
        //getdata from network
        mView.stopLoading();
        mView.showDataList(Arrays.asList("a","b","c"));
    }
}

```
&emsp;上述代码中有很多可以好好思考的地方。首先是`showLoading`和`stopLoading`这两个函数。这两个函数接口定义在`BaseView`,实现在`BaseFragment`，所以当`DataListFragment`继承了`BaseFragment`并且实现了`BaseView`时，它并不需要再次定义上述两个接口，毕竟在一个app中，网络加载时的laoding动画一般都是一致的。然后是view和presenter之间函数调用的问题。首先，view在需要初始化或者需要处理点击事件时，需要调用presenter的函数，而不是自己来进行处理。而且二者之间的数据传递必须是presenter调用view的接口传递给view数据，而不是view调用presenter接口获得数据然后自己展示。

### MVP实践中的问题和思考
&emsp;在实现MVP的过程中，我发现有些模块的代码很容易重构为MVP模式，而有些模块的代码却需要花费大量的时间。那些难重构的模块的所有代码都写在了Activity类中，而那些容易重构的模块已经做了逻辑代码和视图代码的分离。可以说，MVP模式不就是将视图和逻辑分离嘛，你不使用MVP模式，也可以按照自己的理解进行良好的设计。但是，当一个团队开发项目时，问题就变得不同了。不同程序员的水平有一定差异，你无法通过他们的水平来确保他们代码都是良好设计的。**你必须通过MVP模式来强制他们去思考如何分离界面和逻辑,否则，他们估计会把所有代码都写在Activity类中**。我认为这是所谓模式和框架的第一层好处，也就是**强迫程序员使用一个种良好的思维方式去思考如何进行代码组织**。
&emsp;第二点的思考是，模式其实就是良好的代码思考的结晶，如何写出内聚的函数？如何写入内聚的类？如何让模块之间耦合小？如何命名？很多模式都是对这些问题思考结果的总和。
&emsp;如果大家对MVP或者移动开发框架有什么思考，欢迎在文章底下进行评论。
