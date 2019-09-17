---
title: 'Android性能优化(一):TraceView'
tags:
  - 工具
categories:
  - Android性能
abbrlink: e21717d9
date: 2016-12-31 16:18:03
---


&emsp;最近，我准备好好研究一下Android性能优化方面的相关知识，准备从应用流畅度开始，边看《移动App性能评测与优化》边自己实践，希望可以补足一下自己在优化这方面的空白。
&emsp;工欲善其事必先利其器，我先学习了TraceView这个大神器的使用方法。下面就来总结一下。
&emsp;为了节省你的时间，本文主要内容如下：
- TraceView两种使用方法
- TraceView各项数据的含义

### TraceView使用方法
&emsp;TraceView有两种使用方法，一种是在DDMS上手动操作进行使用，还有一种方法是使用代码生成trace文件，然后在使用DDMS打开文件。我们以检测一个启动很慢的Activity为例来介绍。在应用中点击个人中心按钮，跳转到个人中心页面，但是这个过程有明显的卡顿。我们分别使用两个方法来检测这个过程。
&emsp;第一种方法是最简单也是最常用的使用方法。点击DDMS左侧的Start Method Profiling按钮(具体位置如图1,三个小箭头和红点的那个按钮)。

![图1:TraceView启动按钮位置](http://7xrxif.com1.z0.glb.clouddn.com/20161231-tools-TraceView.png)

&emsp;然后点击个人应用中心按钮，等待个人中心页面显示出来，然后再次点击那个按钮，DDMS就会自动显示出如图2的界面。

&emsp;第一种方法很方便，但是精准度不够，当你希望有更加精确的检测时，你就可以使用第二种方法。在你要检测的代码过程中开始调用`android.os.Debug.startMethodTracing()`方法,然后在过程的末尾调用`android.os.Debug.stopMethodTracing()`方法。等到这段代码执行完之后，系统就会在手机的`/sdcard`文件夹下生成一个trace文件。你可以使用DDMS或者adb将其复制到你的电脑上，然后用DDMS工具打开。
![图2:TraceView数据展示](http://7xrxif.com1.z0.glb.clouddn.com/20161231-tools-data.png)

### TraceView各项数据分析
&emsp;我很久之前就接触过TraceView，但是当时并没有完全理解它所展示的所有数据的具体含义，没有感受到它的强大之处。下面，我总结一下它的各项数据的含义和自己的理解。
&emsp;首先，我们可以看到一行表头，其中不同的项代表不同的内容。我们接下来一一进行解释。

#### Incl Cpu Time
`Incl Cpu Time `表示这个函数从开始被执行到执行结束的时间。这个时间包括这个函数中调用其他函数所花费的时间。也就是说，这个函数和函数中所有子函数的一共执行时间。举个例子：
```
  public void example() {
    a();
    b();
  }
```
&emsp;`Incl Cpu Time`就表示`example`函数执行的总时间，假如说`example`方法执行的时间为10ms,函数a执行了3ms,方法b执行了６ms，调用`example`方法的函数执行的总时间为100ms。最终计算出来的`Incl Cpu Time`比率为：
- example 10%
- a 30%
- b 60%

&emsp;通过这个指标，我们可以看到一个函数的所有时间的占比，这个指标一般不会用到，因为我们一般都先查看所有函数的`Excl Cpu Time`指标。但是当你需要精确了解各个函数总执行时间并优化时，你还是需要先从这个函数入手。比如说优化应用冷启动时间，这个时候并没有明显的有较长`Excl Cpu Time`的函数，你需要对目标函数进行一点一点的分析和优化。而不是像寻找明显卡顿原因时的直接找`Excl Cpu Time`指标最大的函数。

#### Excl Cpu Time
&emsp;理解了`Incl Cpu Time`，我们就可以轻易理解这个指标的含义啦。还是上边那个例子。我们可以计算出各个函数的`Incl Cpu Time`比例。
- example (10 - 3 -6)/100 = 1%
- a 30%
- b 60%
&emsp;从例子上来看，一个方法的`Excl Cpu Time`等于它的`Incl Cpu Time`减去它所有子函数的`Incl Cpu Time`。

#### Incl Real Time 和 Excl Real Time
&emsp;对这两条指标的理解主要涉及到对`Cpu time`和`Real time`的理解。Cpu Time 应该是某个方法占用CPU的时间;而Real Time 应该是这个方法的实际运行时间。因为Cpu的上下文切换，阻塞，等待等原因，Real Time 一般长于Cpu Time。
#### Calls+Recur Calls/Total
&emsp;这个指标表示这个方法执行的总次数。Calls表示这个函数的调用次数，而Recur Calls表示递归调用次数。

#### Cpu Time/Calls Real Time/Calls
&emsp;这两个指标表示方法的每次调用的平均执行时间。


### 后记
&emsp;有关TraceView的内容就介绍到这里，了解工具的使用只是开始，我们还需多在实践中应用这些工具，比如开发过程中发现有页面比较卡顿，那么我们就可以使用TraceView去检测一下！！！
