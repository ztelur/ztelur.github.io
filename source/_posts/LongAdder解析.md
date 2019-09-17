---
title: LongAdder解析
tags:
  - JUC
categories:
  - 并发
abbrlink: d29e90d8
date: 2017-04-21 22:38:05
---

＆emsp;对`LongAdder`的最初了解是从Coolshell上的一篇文章中获得的，但是一直都没有深入的了解过其实现，只知道它相较于`AtomicLong`来说，更加适合读多写少的并发情景。今天，我们就研究一下`LongAdder`的原理，探究一下它如此高效的原因。
### 基本原理和思想
 我们都知道`AtomicLong`是通过无限循环不停的采取CAS的方法去设置value，直到成功为止。那么当并发数比较多或出现更新热点时，就会导致CAS的失败机率变高，重试次数更多，越多的线程重试，CAS失败的机率越高，形成恶性循环，从而降低了效率。而LongAdder的原理就是降低对value更新的并发数，也就是将对单一value的变更压力分散到多个value值上，降低单个value的“热度”
 我们知道`LongAdder`的大致原理之后，再来详细的了解一下它的具体实现，其中也有很多值得借鉴的并发编程的技巧。
### Add操作
``` java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    if ((as = cells) != null || !casBase(b = base, b + x)) { //step1
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 || //step2
            (a = as[getProbe() & m]) == null ||  //step3
            !(uncontended = a.cas(v = a.value, v + x))) //step4
            longAccumulate(x, null, uncontended); // step5
    }
}
```
 `cells`是`LongAdder`的父类`Striped64`中的`Cell`数组类型的成员变量。每个`Cell`对象中都包含一个value值，并提供对这个value值的CAS操作。
 我们来看一下`casBase`函数相关的源码吧。我们可以认为变量`base`就是第一个value值，也是基础value变量。先调用casBase函数来cas一下base变量，如果成功了，就不需要在进行下面比较复杂的算法，
```
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }
```
 然后我们继续看step2第二层条件语句中执行的逻辑。如果cells数组为null或为空,就直接调用`longAccumulate`方法。因为cells为null或在为空，说明cells未完全初始化，所以调用`longAccumulate`进行初始化。否则继续判断。
 如果cells中已经有对象了，那么执行step3。我们先来理解一下`getProbe() & m`的这个操作吧。我们可以首先将这个操作当作一次计算"hash"值,然后将cells中这个位置的Cell对象赋值给变量a。然后判断a是否为null,如果不为null,那么就调用Cell对象自己的cas方法去设置value值。如果a为null,或在cas赋值发生冲突，那么也是开始调用`longAccumulate`方法。

### LongAccumulate方法
 `longAccumulate`函数比较复杂，带有我的注释的代码已经贴在了文章后边，这里我们就只讲一下其中比较关键的一些技巧和思想．
 首先，我们都知道只有当对`base`的cas操作失败之后，`LongAdder`才引入`Cell`数组．所以在`longAccumulate`中就是对`Cell`数组进行操作．分别涉及了数组的初始化，扩容和设置某个位置的Cell对象等操作．
 在这段代码中，关于cellBusy的cas操作构成了一个SpinLock,这就是经典的SpinLock的编程技巧，大家可以学习一下．

``` java
final void longAccumulate(long x, LongBinaryOperator fn,
                             boolean wasUncontended) {
       int h;
       if ((h = getProbe()) == 0) { //获取PROBE变量，探针变量，与当前运行的线程相关，不同线程不同
           ThreadLocalRandom.current(); //初始化PROBE变量，和getProbe都使用Unsafe类提供的原子性操作。
           h = getProbe();
           wasUncontended = true;
       }
       boolean collide = false;
       for (;;) { //cas经典无限循环，不断尝试
           Cell[] as; Cell a; int n; long v;
           if ((as = cells) != null && (n = as.length) > 0) { // cells不为null,并且数组size大于0
           //表示cells已经初始化了
               if ((a = as[(n - 1) & h]) == null) { //通过与操作计算出来需要操作的Cell对象的坐标
                   if (cellsBusy == 0) { //volatile 变量，用来实现spinLock,来在初始化和resize cells数组时使用。
                   //当cellsBusy为0时，表示当前可以对cells数组进行操作。 
                       Cell r = new Cell(x);//将x值直接赋值给Cell对象
                       if (cellsBusy == 0 && casCellsBusy()) {//如果这个时候cellsBusy还是0
                       //就cas将其设置为非０，如果成功了就是获得了spinLock的锁．可以对cells数组进行操作．
                       //如果失败了，就会再次执行一次循环
                           boolean created = false;
                           try {
                               Cell[] rs; int m, j;
                               //判断cells是否已经初始化，并且要操作的位置上没有cell对象．
                               if ((rs = cells) != null &&
                                   (m = rs.length) > 0 &&
                                   rs[j = (m - 1) & h] == null) {
                                   rs[j] = r;　//将之前创建的值为x的cell对象赋值到cells数组的响应位置．
                                   created = true;
                               }
                           } finally {
                               //经典的spinLock编程技巧，先获得锁，然后try finally将锁释放掉
                               //将cellBusy设置为0就是释放锁．
                               cellsBusy = 0;
                           }
                           if (created)
                               break;　//如果创建成功了，就是使用x创建了新的cell对象，也就是新创建了一个分担热点的value
                           continue; 
                       }
                   }
                   collide = false; //未发生碰撞
               }
               else if (!wasUncontended)//是否已经发生过一次cas操作失败
                   wasUncontended = true; //设置成true,以便第二次进入下一个else if 判断
               else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                            fn.applyAsLong(v, x))))
                  　//fn是操作类型，如果是空，就是相加，所以让a这个cell对象中的value值和x相加，然后在cas设置，如果成果
                  //就直接返回
                   break;
               else if (n >= NCPU || cells != as)
               　　//如果cells数组的大小大于系统的可获得处理器数量或在as不再和cells相等．
                   collide = false;            // At max size or stale
               else if (!collide)
                   collide = true;
               else if (cellsBusy == 0 && casCellsBusy()) {
               　　//再次获得cellsBusy这个spinLock,对数组进行resize
                   try {
                       if (cells == as) {//要再次检测as是否等于cells以免其他线程已经对cells进行了操作．
                           Cell[] rs = new Cell[n << 1]; //扩容一倍
                           for (int i = 0; i < n; ++i)
                               rs[i] = as[i];
                           cells = rs;//赋予cells一个新的数组对象
                       }
                   } finally {
                       cellsBusy = 0;
                   }
                   collide = false;
                   continue;
               }
               h = advanceProbe(h);//由于使用当前探针变量无法操作成功，所以重新设置一个,再次尝试
           }
           else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
           //cells数组未初始化，获得cellsBusy　lock,来初始化
               boolean init = false;
               try {                           // Initialize table
                   if (cells == as) {
                       Cell[] rs = new Cell[2];
                       rs[h & 1] = new Cell(x); //设置x的值为cell对象的value值
                       cells = rs;
                       init = true;
                   }
               } finally {
                   cellsBusy = 0;
               }
               if (init)
                   break;
           }//如果初始化数组失败了，那就再次尝试一下直接cas base变量，如果成功了就直接返回
           else if (casBase(v = base, ((fn == null) ? v + x :
                                       fn.applyAsLong(v, x))))
               break;                          // Fall back on using base
       }
   }
```

### 后记
&emsp;本篇文章写的不是很好，我写完之后又看了一遍coolshell上的这篇关于`LongAdder`的[文章](http://coolshell.cn/articles/11454.html)，感觉自己没有人家写的那么简介明了。我对代码细节的注释和投入太多了。其实很多代码大家都可以看懂，并不需要大量的代码片段加注释。以后要注意一下。之后会接着研究一下JUC包中的其他类，希望大家多多关注。
