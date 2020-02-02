---
title: 'Netty源码(一):Netty中的Buffer'
tags:
  - Netty
categories:
  - Netty
abbrlink: bf1a30e4
date: 2017-03-22 23:09:21
---

 最近我学习了NIO相关的知识,然后发现了Netty这个基于NIO的网络应用框架,于是就研究起Netty框架源码,来好好体会一下网络框架的设计理念和思想.
 这个系列的文章不仅会总结Netty各个模块的源码原理,也会写出一些自己对这些设计的理解和体会.
 我基本按照并发编程网上[这个系列文章](http://ifeve.com/netty1/)的顺序来进行系列文章的顺序,不同的是我是基于Netty4.1的源码进行分析和讲解.
 为了节约你的时间,本篇文章主要内容如下:
- Netty的Buffer的内存模型,涉及读写指针
- Netty的Buffer框架
- Netty的Pool原理,轻量对象池

### Buffer
 Java NIO中的Buffer用于和NIO通道进行交互,数据可以从通道读入缓冲区,也可以从缓冲区写入到通道中.所以说,Buffer其实就是一块可以读写数据的内存,我们将其包装为一个Java对象来提供一系列读写操作.
 Netty并没有直接使用Java NIO的Buffer实现,而是自己实现了一套Buffer框架来满足自己的业务或者性能需求.

### ByteBuf的基本原理
#### 读写指针的作用
 不同于NIO Buffer的读写指针共用原理,ByteBuf拥有`readerIndex`,`writerIndex`两个指针.下面我们就来详细的讲解一下ByteBuf的内部原理.
```
     +-------------------+------------------+------------------+
     | discardable bytes |  readable bytes  |  writable bytes  |
     |                   |     (CONTENT)    |                  |
     +-------------------+------------------+------------------+
     |                   |                  |                  |
     0      <=      readerIndex   <=   writerIndex    <=    capacity
```
 从示意图中我们可以看出`readerIndex`和`writerIndex`最多可以将整个内容空间划分为三块:`废弃区`,`可读区`和`可写区`.下面我们就来看一下不同操作下的两个指针的变化.
- 在初始化状态下,假设capacity为20,`readerIndex`和`writerIndex`都为0,整个空间中只存在可写区.此时只能写,不能读,进行读操作会抛出异常.
```
       +---------------------------------------------------------+
       |             writable bytes (got more space)             |
       +---------------------------------------------------------+
       |                                                         |
 readerIndex(0)
  writerIndex(0)                   <=                   capacity
```
- 写入10个字节的数据,`writerIndex`指向10,`readerIndex`不会改变,所有内容空间中有可读区和可写区.大小都是10字节.

```
       +-------------------+------------------+------------------+
       |  readable bytes  |  writable bytes                      |
       |     (CONTENT)    |                                      |
       +--------- --------+------------------+------------------
       |                  |                                      |
     readerIndex(0) <= writerIndex(10)           <=        capacity
```

- 读取5个字节的内容,`writerIndex`不变,`readerIndex`加5,指向了5.此时内容空间分为了5字节的废弃区,5字节的可读区和10字节的可写区.

```

 +-------------------+------------------+------------------+
 | discardable bytes |  readable bytes  |  writable bytes  |
 |                   |     (CONTENT)    |                  |
 +-------------------+------------------+------------------+
 |                   |                  |                  |
 0      <=      readerIndex(5)   <=   writerIndex(10)    <=  capacity
```
- 调用`discardReadBytes`方法后,将废弃区的内容舍弃掉,`readerIndex`又指向了0,`writerIndex`指向了5,相当于可读区和可写区整体向左平移了5个字节.
```
       +------------------+--------------------------------------+
       |  readable bytes  |    writable bytes (got more space)   |
       +------------------+--------------------------------------+
       |                  |                                      |
  readerIndex (0) <= writerIndex (5)              <=        capacity
```
#### 零拷贝
 OS层次上`Zero-copy`,就是在操作数据时,不需要将数据buffer从一个内存区域拷贝到另一个内存区域,因为减少了一次内存的拷贝,因此CPU的效率得到了提升.
 `Netty`的`zero-copy`体现在很多方面.比如Buffer的compose,duplicate,slice操作时不会拷贝底层的数据.而是通过ByteBuf对象的组合来实现上述的操作
- Netty提供了`CompositeByteBuf`类,可以将多个`ByteBuf`组合成一个逻辑上的Buffer,避免了各个buffer之间的拷贝,`CompositeByteBuf`并不拥有底层的数据,而是通过拥有两个buffer对象,从这两个buffer对象中获取数据来对外提供看似合并了的数据.比如我们将一份协议数据的头部buffer和消息体buffer合并成一个Buffer.
![compose](http://7xrxif.com1.z0.glb.clouddn.com/2017322-netty-compose.png)
 如上图所示,所有底层的数据还是存储在header和body这两个真实的buffer中.

- 对于`ByteBuf`的`slice`和`duplicate`操作也是如此,不同的buffer共享了相同的底层数据,而不是进行底层数据的拷贝.具体使用到的Buffer类型为`DuplicatedByteBuf`和`SlicedByteBuf`.谁说是共享的底层数据,但是通过对`writerIndex`和`readerIndex`两个指针的操作来实现slice和duplicate的功能.
- Netty使用`wrap`操作将byte数组转化为`ByteBuf`对象时,将byte数组包裹到对象中,而不是拷贝数组存放到对象中.
- Netty 中使用 FileRegion 实现文件传输的零拷贝, 不过在底层 FileRegion 是依赖于 Java NIO FileChannel.transfer 的零拷贝功能.


### Pool和Reference Count
 4.0之后的版本实现了高性能的Buffer池,分配策略则是结合了buddy allocation和slab allocation的jemalloc变种，实现类为`PoolArena`,这样的话,可以在频繁分配和释放Buffer时缓解GC压力,还可以在初始化新buffer时减少内存带宽消耗（初始化时不可避免的要给buffer数组赋初始值).
 `ByteBuf`引入了`Reference Count`机制,你需要在不适用它的时候调用`ReferenceCountUtil.release`方法来减少它的引用.

### 后记
&emsp;感觉自己在研究或在阅读源代码时还是有些问题,起始`ByteBuf`并不是`Netty`的关键所在,不应该花费这么长时间.以后还是要带着目的来看源码,不能把时间浪费在一些代码细节上.

### 引用
- https://segmentfault.com/a/1190000007560884
