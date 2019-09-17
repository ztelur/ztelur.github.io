---
title: OkHttp解析系列-开篇
tags:
  - OkHttp
categories:
  - Android
  - NetWork
abbrlink: 1f02db9c
date: 2015-11-07 11:55:04
---
#### 背景
&emsp;前几天使用`react-native`遇到了底层`okhttp`库`cookie`无法保存的问题，由于自己对http和`okhttp`也不是很了解.所以想开一个系列的博文，借助详细解析`okhttp`的详细解析来梳理一下http相关的知识。
#### Okhttp
&emsp;`Okhttp`是很火而且效率很好的一个android的网络库，被很多app或者开源库使用或者集成，比如`react-native`,官网地址如下[戳我](http://square.github.io/okhttp/).
#### UML类图
&emsp;事先声明，我画的这张UML类图不够标准，正方形的虚线边框是我自己添加上去的，只是逻辑上或者概念上的分类，标示这些类大致是属于哪个模块的，而且也没有添加各个类之间的依赖关系。之所以使用这张图，主要是希望表面`okhttp`代码的不同模块吧。也算为以后的博客进行内容区分。
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/okhttpOkHttp%E5%8D%9A%E5%AE%A2.jpg)

#### Okhttp模块
&emsp;之后的博客，就会按照上图的不同模块来进行，首先是`OkHttp`的主要框架模块，然后是请求和响应相关的模块，然后是关于http机制的模块，最后是关于http报文格式的模块。现在计划是如此，可能在博文之间会添加一些http知识。就这么愉快的决定啦。
