---
title: TCP/IP的底层队列
date: 2019-03-09 23:24:38
tags: TCP/IP
category: 网络
---
&emsp;自从上次学习了TCP/IP的拥塞控制算法后，我越发想要更加深入的了解TCP/IP的一些底层原理，搜索了很多网络上的资料，看到了陶辉大神关于高性能网络编程的专栏，收益颇多。今天就总结一下，并且加上自己的一些思考。

&emsp;我自己比较了解Java语言，对Java网络编程的理解就止于Netty框架的使用。`Netty`的源码贡献者Norman Maurer对于Netty网络开发有过一句建议，"Never block the event loop, reduce context-swtiching"。也就是尽量不要阻塞IO线程，也尽量减少线程切换。我们今天只关注前半句，对这句话感兴趣的同学可以看一下[蚂蚁通信框架实践
](https://mp.weixin.qq.com/s/JRsbK1Un2av9GKmJ8DK7IQ?)。

&emsp;为什么不能阻塞读取网络信息的IO线程呢？这里就要从经典的网络C10K开始理解，服务器如何支持并发1万请求。C10K的根源在于网络的IO模型。Linux 中网络处理都用同步阻塞的方式，也就是每个请求都分配一个进程或者线程，那么要支持1万并发，难道就要使用1万个线程处理请求嘛？这1万个线程的调度、上下文切换乃至它们占用的内存，都会成为瓶颈。解决C10K的通用办法就是使用I/O 多路复用，Netty就是这样。

![Netty的reactor模型](https://upload-images.jianshu.io/upload_images/623378-778c5e86e4fc9e5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;Netty有负责服务端监听建立连接的线程组(mainReactor)和负责连接读写操作的IO线程组(subReactor)，还可以有专门处理业务逻辑的Worker线程组(ThreadPool)。三者相互独立，这样有很多好处。一是有专门的线程组负责监听和处理网络连接的建立，可以防止TCP/IP的半连接队列(sync)和全连接队列(acceptable)被占满。二是IO线程组和Worker线程分开，双方并行处理网络I/O和业务逻辑，可以避免IO线程被阻塞，防止TCP/IP的接收报文的队列被占满。当然，如果业务逻辑较少，也就是IO 密集型的轻计算业务，可以将业务逻辑放在IO线程中处理，避免线程切换，这也就是Norman Maurer话的后半部分。

&emsp;TCP/IP怎么就这么多队列啊？今天我们就来细看一下TCP/IP的几个队列,包括建立连接时的半连接队列(sync)，全连接队列(accept)和接收报文时的receive、out_of_order、prequeue以及backlog队列。

### 建立连接时的队列

![TCP三次握手和队列示意图](https://upload-images.jianshu.io/upload_images/623378-370a716b31ece06c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;如上图所示，这里有两个队列：syns queue(半连接队列)和accept queue(全连接队列)。三次握手中，服务端接收到客户端的SYN报文后，把相关信息放到半连接队列中，同时回复SYN+ACK给客户端。
&emsp;第三步的时候服务端收到客户端的ACK，如果这时全连接队列没满，那么从半连接队列拿出相关信息放入到全连接队列中，否则按`tcp_abort_on_overflow`的值来执行相关操作，直接抛弃或者过一段时间在重试。

### 接收报文时的队列

&emsp;相比于建立连接，TCP在接收报文时的处理逻辑更为复杂，相关的队列和涉及的配置参数更多。

&emsp;应用程序接收TCP报文和程序所在服务器系统接收网络里发来的TCP报文是两个独立流程。二者都会操控socket实例，但是会通过锁竞争来决定某一时刻由谁来操控，由此产生很多不同的场景。例如，应用程序正在接收报文时，操作系统通过网卡又接收到报文，这时该如何处理？若应用程序没有调用read或者recv读取报文时，操作系统收到报文又会如何处理？

&emsp;我们接下来就以三张图为主，介绍TCP接收报文时的三种场景，并在其中介绍四个接收相关的队列。

#### 接收报文场景一

![场景一](https://upload-images.jianshu.io/upload_images/623378-f25d47544e00ea0b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是TCP接收报文场景一的示意图。操作系统首先接收报文，存储到socket的receive队列，然后用户进程再调用recv进行读取。

1) 当网卡接收报文并且判断为TCP协议时，经过层层调用，最终会调用到内核的`tcp_v4_rcv`方法。由于当前TCP要接收的下一个报文正是S1，所以`tcp_v4_rcv`函数将其直接加入到`receive`队列中。`receive`队列是将已经接收到的TCP报文，去除了TCP头部、排好序放入的、用户进程可以直接按序读取的队列。由于socket不在用户进程上下文中（也就是没有用户进程在读socket），并且我们需要S1序号的报文，而恰好收到了S1报文，因此，它进入了`receive`队列。

2) 接收到S3报文，由于TCP要接收的下一个报文序号是S2，所以加入到`out_of_order`队列，所有乱序的报文会放在这里。

3) 接着，收到了TCP期望的S2报文，直接进入`recevie`队列。由于此时`out_of_order`队列不为空，需要检查一下。

4) 每次向`receive`队列插入报文时都会检查`out_of_order`队列，由于接收到S2报文后，期望的的序号为S3，所以`out_of_order`队列中的S3报文会被移到`receive`队列。

5) 用户进程开始读取socket，先在进程中分配一块内存，然后调用`read`或者`recv`方法。socket有一系列的具有默认值的配置属性，比如socket默认是阻塞式的，它的`SO_RCVLOWAT`属性值默认为1。当然，recv这样的方法还会接收一个flag参数，它可以设置为`MSG_WAITALL`、`MSG_PEEK`、`MSG_TRUNK`等等，这里我们假定为最常用的0。进程调用了`recv`方法。

6) 调用`tcp_recvmsg`方法

7) `tcp_recvmsg`方法会首先锁住socket。socket是可以被多线程使用的，而且操作系统也会使用，所以必须处理并发问题。要操控socket，就先获取锁。

8) 此时，`receive`队列已经有3个报文了，将第一个报文拷贝到用户态内存中，由于第五步中socket的参数并没有带`MSG_PEEK`，所以将第一个报文从队列中移除，从内核态释放掉。反之，`MSG_PEEK`标志位会导致`receive`队列不会删除报文。所以，`MSG_PEEK`主要用于多进程读取同一套接字的情形。

9) 拷贝第二个报文，当然，执行拷贝前都会检查用户态内存的剩余空间是否足以放下当前这个报文，不够时会直接返回已经拷贝的字节数。
10) 拷贝第三个报文。
11) `receive`队列已经为空，此时会检查`SO_RCVLOWAT`这个最小阈值。如果已经拷贝字节数小于它，进程会休眠，等待更多报文。默认的`SO_RCVLOWAT`值为1，也就是读取到报文就可以返回。

12) 检查`backlog`队列，`backlog`队列是用户进程正在拷贝数据时，网卡收到的报文会进这个队列。如果此时`backlog`队列有数据，就顺带处理下。`backlog`队列是没有数据的，因此释放锁，准备返回用户态。

13) 用户进程代码开始执行，此时recv等方法返回的就是从内核拷贝的字节数。


#### 接收报文场景二

&emsp;第二张图给出了第二个场景，这里涉及了`prequeue`队列。用户进程调用recv方法时，socket队列中没有任何报文，而socket是阻塞的，所以进程睡眠了。然后操作系统收到了报文，此时`prequeue`队列开始产生作用。该场景中，`tcp_low_latency`为默认的0，套接字socket的`SO_RCVLOWAT`是默认的1，仍然是阻塞socket，如下图。

![场景二](https://upload-images.jianshu.io/upload_images/623378-524a7634fa58fa48.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;其中1，2，3步骤的处理和之前一样。我们直接从第四步开始。

4) 由于此时`receive`,`prequeue`和`backlog`队列都为空，所以没有拷贝一个字节到用户内存中。而socket的配置要求至少拷贝`SO_RCVLOWAT`也就是1字节的报文，因此进入阻塞式套接字的等待流程。最长等待时间为`SO_RCVTIMEO`指定的时间。socket在进入等待前会释放socket锁，会使第五步中，新来的报文不再只能进入`backlog`队列。
5) 接到S1报文，将其加入`prequeue`队列中。
6) 插入到`prequeue`队列后，会唤醒在socket上休眠的进程。
7) 用户进程被唤醒后，重新获取socket锁，此后再接收到的报文只能进入`backlog`队列。
8) 进程先检查`receive`队列，当然仍然是空的；再去检查`prequeue`队列，发现有报文S1，正好是正在等待序号的报文，于是直接从`prequeue`队列中拷贝到用户内存，再释放内核中的这个报文。
9) 目前已经拷贝了一个字节的报文到用户内存，检查这个长度是否超过了最低阈值，也就是len和`SO_RCVLOWAT`的最小值。
10) 由于`SO_RCVLOWAT`使用了默认值1，拷贝字节数大于最低阈值，准备返回用户态，顺便会查看一下backlog队列中是否有数据，此时没有，所以准备放回，释放socket锁。
11) 返回用户已经拷贝的字节数。



#### 接收报文场景三

&emsp;在第三个场景中，系统参数`tcp_low_latency`为1，socket上设置了`SO_RCVLOWAT`属性值。服务器先收到报文S1，但是其长度小于`SO_RCVLOWAT`。用户进程调用`recv`方法读取，虽然读取到了一部分，但是没有到达最小阈值，所以进程睡眠了。与此同时，在睡眠前接收的乱序的报文S3直接进入`backlog`队列。然后，报文S2到达，由于没有使用`prequeue`队列（因为设置了tcp_low_latency），而它起始序号正是下一个待拷贝的值，所以直接拷贝到用户内存中，总共拷贝字节数已满足`SO_RCVLOWAT`的要求！最后在返回用户前把`backlog`队列中S3报文也拷贝给用户。

![场景三](https://upload-images.jianshu.io/upload_images/623378-2e37fda0937a4a95.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1) 接收到报文S1，正是准备接收的报文序号，因此，将它直接加入到有序的`receive`队列中。
2) 将系统属性`tcp_low_latency`设置为1，表明服务器希望程序能够及时的接收到TCP报文。用户调用的`recv`接收阻塞socket上的报文，该socket的`SO_RCVLOWAT`值大于第一个报文的大小，并且用户分配了足够大的长度为len的内存。
3) 调用`tcp_recvmsg`方法来完成接收工作，先锁住socket。
4) 准备处理内核各个接收队列中的报文。
5) `receive`队列中有报文可以直接拷贝，其大小小于len，直接拷贝到用户内存。
6) 在进行第五步的同时，内核又接收到S3报文，此时socket被锁，报文直接进入`backlog`队列。这个报文并不是有序的。
7) 在第五步时，拷贝报文S1到用户内存，它的大小小于`SO_RCVLOWAT`的值。由于socket是阻塞型，所以用户进程进入睡眠状态。进入睡眠前，会先处理`backlog`队列的报文。因为S3报文是失序的，所以进入`out_of_order `队列。用户进程进入休眠状态前都会先处理一下`backlog`队列。
8) 进程休眠，直到超时或者`receive`队列不为空。
9) 内核接收到报文S2。注意，此时由于打开了`tcp_low_latency`标志位，所以报文是不会进入`prequeue`队列等待进程处理。
10) 由于报文S2正是要接收的报文，同时，一个用户进程在休眠等待该报文，所以直接将报文S2拷贝到用户内存。
11) 每处理完一个有序报文后，无论是拷贝到`receive`队列还是直接复制到用户内存，都会检查`out_of_order`队列，看看是否有报文可以处理。报文S3拷贝到用户内存，然后唤醒用户进程。
12) 唤醒用户进程。
13) 此时会检查已拷贝的字节数是否大于`SO_RCVLOWAT`，以及`backlog`队列是否为空。两者皆满足，准备返回。


&emsp;总结一下四个队列的作用。

- receive队列是真正的接收队列，操作系统收到的TCP数据包经过检查和处理后，就会保存到这个队列中。
- `backlog`是“备用队列”。当socket处于用户进程的上下文时（即用户正在对socket进行系统调用，如recv），操作系统收到数据包时会将数据包保存到`backlog`队列中，然后直接返回。
- `prequeue`是“预存队列”。当socket没有正在被用户进程使用时，也就是用户进程调用了read或者recv系统调用，但是进入了睡眠状态时，操作系统直接将收到的报文保存在`prequeue`中，然后返回。
- `out_of_order`是“乱序队列”。队列存储的是乱序的报文，操作系统收到的报文并不是TCP准备接收的下一个序号的报文，则放入`out_of_order`队列，等待后续处理。


### 后记

&emsp;如果你觉得本篇文章对你有帮助，请点个赞。同时欢迎订阅本人的微信公众号。

![](https://upload-images.jianshu.io/upload_images/623378-7d960275042f309d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 参考

- http://www.voidcn.com/article/p-gzmjmmna-dn.html
- https://blog.csdn.net/russell_tao/article/details/9950615
- https://ylgrgyq.github.io/2017/08/01/linux-receive-packet-3/