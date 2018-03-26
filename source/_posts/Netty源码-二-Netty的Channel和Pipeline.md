---
title: 'Netty源码(二):Netty的Channel和Pipeline'
date: 2017-03-27 21:38:03
tags: 
	- Netty
categories:
	- NIO
---

 本文主要讲述Netty框架中Channel相关的知识，Netty通过Channel和Pipeline等一些组件提供了所谓的`Universal Communication API`．与`Channel`相关的知识点比较多，本篇文章就主要讲解一下`Channel`和`Pipeline`的事件处理流原理．`Channel`,`EventLoop`和`ChannelFuture`的相关知识下篇文章中再进行讲述．

### 官方文档上的Channel
 官方文档上给出的解释是Channel是与网络Socket相关的或具有一定I/O能力的组件．一个`Channel`可以给用户提供:

- 当前`Channel`的状态(比如，是否保存Open状态，是否处于连接状态)
- `Channel`的配置参数，比如说buffer的大小
- `Channel`支持的相关I/O操作，比如说`read`,`write`,`connect`和`bind`
- 提供一个`ChannelPipeline`来处理所有与该`Channel`相关的I/O事件和请求

 `Channel`上进行的所有I/O操作都是异步的，也就是说，所有涉及I/O操作的调用都会立刻返回，并不保证操作完成，而是会返回一个`ChannelFuture`对象来通知你操作是否完成．
 Channel是有层级的，这样的话，你就可以很方便的利用其他已有的Channel来构建自己需要的`channel`,比如说基于`SocketChannel`来实现关于`SSH`的`Channel`

 此外，当你完成某些操作之后调用`close()`或在`close(ChannelPromise)`是非常重要的，这样能确保你正确的释放了所有资源．
#### 我眼中的Channel
 首先，我们应该都知道Netty支持很多I/O通信协议:

- 基于TCP的NIO: `NioServerSocketChannel`,`NioSocketChannel`
- 基于UDP的NIO:`NioDatagramChannel`
- 基于TCP的OIO:`OioSocketChannel`和`OioServerSocketChannel`
- 基于UDP的OIO:`OioDatagramChannel`
如果把关于Channel的类图列出来的话，你会发现支持各种协议的`Channel`,不信你就看一下这个类图．

 这样想一下，***`Channel`不就是`Netty`框架用来封装不同协议逻辑的组件吗?***，有了`Channel`的存在，所有于通信协议相关的逻辑都隐藏在不同的`Channel`实现里，然后在对外提供相对统一的API.
 说道这里，你可能还不知道，即使是`OIOChannel`,它提供的I/O操作也是异步的．也就是说***在Netty框架中，不论是OIO还是NIO模型，读写都会阻塞***．这样也体现了`Universal Communication API`的思想，这就使得我们切换Channel非常方便．我们只要初始化不同的Channel即可．
`
ServerBootstrap b = new ServerBootstrap(); 
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
ServerBootstrap b = new ServerBootstrap(); 
            b.group(bossGroup, workerGroup)
             .channel(OioServerSocketChannel.class)
`
 有关`Channel`协议相关的底层知识，我们会在下一篇文章时进行介绍．
### ChannelHandler和ChannelPipeline
 只有`Channel`的支持，还不足以实现`Universal Communication API`,还需要上述两个类来提供`ChannelHandler`的编程模式，基于`ChannelHandler`来开发业务逻辑，而不需要考虑网络通讯方面的事情．
 Netty源码中一张图形象的描述了这个机制
![示意图](http://7xrxif.com1.z0.glb.clouddn.com/2017327-netty-stream.png)

&emsp;Netty中的`ChannelPipeline`包含两条线路：Upstream和`Downstream`．Upstream对应上行，接受信息，被动的状态改变，都属于Upstream．Downstream则对应下行，发送消息，主动状态的改变．Upstream对应InBound Handler,Downstream对应Outbound Handler.从Netty内部IO线程接读到IO数据，依次经过N个Handler到达最内部的逻辑处理单元，这种称之为Inbound Handler；从Channel发出IO请求，依次经过M个Handler到达Netty内部IO线程，这种称之为Outbound Handler
&emsp;需要注意的是，这个Handler链中消息或在事件不会自动的向下或在向上流动或转发，而是需要由上一个Handler显示的调用`ChannelPipeline.sendUp(down)stream`来交给下一个Handler来处理．也就是说，每个Handler接受到一个`ChannelEvent`,处理结束后，如果需要继续处理，那么它需要向下一个或在上一个Handler发起一个事件．




