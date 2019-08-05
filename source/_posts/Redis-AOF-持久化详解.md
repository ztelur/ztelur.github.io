---
title: Redis AOF 持久化详解
date: 2019-07-28 22:36:28
tags:
---

Redis 是一种内存数据库，将数据保存在内存中，读写效率要比传统的将数据保存在磁盘上的数据库要快很多。但是一旦进程退出，Redis 的数据就会丢失。

为了解决这个问题，Redis 提供了 RDB 和 AOF 两种持久化方案，将内存中的数据保存到磁盘中，避免数据丢失。RDB的介绍在这篇文章中[《Redis RDB 持久化详解》](https://mp.weixin.qq.com/s/NpUV-7bvXTD3iu0_2aRssQ)，今天我们来看一下 AOF 相关的原理。


AOF( append only file )持久化以独立日志的方式记录每次写命令，并在 Redis 重启时在重新执行 AOF 文件中的命令以达到恢复数据的目的。AOF 的主要作用是解决数据持久化的实时性。

### RDB 和 AOF
antirez 在《Redis 持久化解密》一文中讲述了 RDB 和 AOF 各自的优缺点：
- RDB 是一个紧凑压缩的二进制文件，代表 Redis 在某个时间点上的数据备份。非常适合备份，全量复制等场景。比如每6小时执行 bgsave 备份，并把 RDB 文件拷贝到远程机器或者文件系统中，用于灾难恢复。
- Redis 加载 RDB 恢复数据远远快于 AOF 的方式
- RDB 方式数据没办法做到实时持久化，而 AOF 方式可以做到。

下面，我们就来了解一下 AOF 是如何做到实时持久化的。

### AOF 持久化的实现

![示意图](https://upload-images.jianshu.io/upload_images/623378-bcabcd38e2b747b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，AOF 持久化功能的实现可以分为命令追加( append )、文件写入( write )、文件同步( sync )、文件重写(rewrite)和重启加载(load)。其流程如下：
- 所有的写命令会追加到 AOF 缓冲中。
- AOF 缓冲区根据对应的策略向硬盘进行同步操作。
- 随着 AOF 文件越来越大，需要定期对 AOF 文件进行重写，达到压缩的目的。
- 当 Redis 重启时，可以加载 AOF 文件进行数据恢复。

#### 命令追加
当 AOF 持久化功能处于打开状态时，Redis 在执行完一个写命令之后，会以协议格式(也就是RESP，即 Redis 客户端和服务器交互的通信协议 )将被执行的写命令追加到 Redis 服务端维护的 AOF 缓冲区末尾。

比如说 SET mykey myvalue 这条命令就以如下格式记录到 AOF 缓冲中。
```
"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
```

Redis 协议格式本文不再赘述，AOF之所以直接采用文本协议格式，是因为所有写入命令都要进行追加操作，直接采用协议格式，避免了二次处理开销。


#### 文件写入和同步

Redis 每次结束一个事件循环之前，它都会调用 `flushAppendOnlyFile` 函数，判断是否需要将 AOF 缓存区中的内容写入和同步到 AOF 文件中。

`flushAppendOnlyFile` 函数的行为由 redis.conf 配置中的 `appendfsync` 选项的值来决定。该选项有三个可选值，分别是 `always`、`everysec` 和 `no`：
- `always`：Redis 在每个事件循环都要将 AOF 缓冲区中的所有内容写入到 AOF 文件，并且同步 AOF 文件，所以 `always` 的效率是 `appendfsync` 选项三个值当中最差的一个，但从安全性来说，也是最安全的。当发生故障停机时，AOF 持久化也只会丢失一个事件循环中所产生的命令数据。
- `everysec`：Redis 在每个事件循环都要将 AOF 缓冲区中的所有内容写入到 AOF 文件中，并且每隔一秒就要在子线程中对 AOF 文件进行一次同步。从效率上看，该模式足够快。当发生故障停机时，只会丢失一秒钟的命令数据。
- `no`：Redis 在每一个事件循环都要将 AOF 缓冲区中的所有内容写入到 AOF 文件。而 AOF 文件的同步由操作系统控制。这种模式下速度最快，但是同步的时间间隔较长，出现故障时可能会丢失较多数据。

Linux 系统下 `write` 操作会触发延迟写( delayed write )机制。Linux 在内核提供页缓存区用来提供硬盘 IO 性能。`write` 操作在写入系统缓冲区之后直接返回。同步硬盘操作依赖于系统调度机制，例如：缓冲区页空间写满或者达到特定时间周期。同步文件之前，如果此时系统故障宕机，缓冲区内数据将丢失。

而 `fsync` 针对单个文件操作，对其进行强制硬盘同步，`fsync` 将阻塞直到写入磁盘完成后返回，保证了数据持久化。

`appendfsync `的三个值代表着三种不同的调用 `fsync`的策略。调用`fsync`周期越频繁，读写效率就越差，但是相应的安全性越高，发生宕机时丢失的数据越少。

有关 Linux 的I/O和各个系统调用的作用如下图所示。具体内容可以查看[《聊聊 Linux I/O》](https://mp.weixin.qq.com/s/3mKxTH2pfXFpDvvJnDtgEQ)一文。

![示意图](https://upload-images.jianshu.io/upload_images/623378-a099d000d47ca0ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### AOF 数据恢复

AOF 文件里边包含了重建 Redis 数据所需的所有写命令，所以 Redis 只要读入并重新执行一遍 AOF 文件里边保存的写命令，就可以还原 Redis 关闭之前的状态。

![示意图](https://upload-images.jianshu.io/upload_images/623378-7b17a4463deec765.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Redis 读取 AOF 文件并且还原数据库状态的详细步骤如下：

- 创建一个不带网络连接的的伪客户端( fake client)，因为 Redis 的命令只能在客户端上下文中执行，而载入 AOF 文件时所使用的的命令直接来源于 AOF 文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行 AOF 文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样的。
- 从 AOF 文件中分析并取出一条写命令。
- 使用伪客户端执行被读出的写命令。
- 一直执行步骤 2 和步骤3，直到 AOF 文件中的所有写命令都被处理完毕为止。

当完成以上步骤之后，AOF 文件所保存的数据库状态就会被完整还原出来。



### AOF 重写

因为 AOF 持久化是通过保存被执行的写命令来记录 Redis 状态的，所以随着 Redis 长时间运行，AOF 文件中的内容会越来越多，文件的体积也会越来越大，如果不加以控制的话，体积过大的 AOF 文件很可能对 Redis 甚至宿主计算机造成影响。

为了解决 AOF 文件体积膨胀的问题，Redis 提供了 AOF 文件重写( rewrite) 功能。通过该功能，Redis 可以创建一个新的 AOF 文件来替代现有的 AOF 文件。新旧两个 AOF 文件所保存的 Redis 状态相同，但是新的 AOF 文件不会包含任何浪费空间的荣誉命令，所以新 AOF 文件的体积通常比旧 AOF 文件的体积要小得很多。

![示意图](https://upload-images.jianshu.io/upload_images/623378-f4a19a6b0e3532de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，重写前要记录名为`list`的键的状态，AOF 文件要保存五条命令，而重写后，则只需要保存一条命令。


AOF 文件重写并不需要对现有的 AOF 文件进行任何读取、分析或者写入操作，而是通过读取服务器当前的数据库状态来实现的。首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令，这就是 AOF 重写功能的实现原理。


在实际过程中，为了避免在执行命令时造成客户端输入缓冲区溢出，AOF 重写在处理列表、哈希表、集合和有序集合这四种可能会带有多个元素的键时，会先检查键所包含的元素数量，如果数量超过 REDIS_AOF_REWRITE_ITEMS_PER_CMD ( 一般为64 )常量，则使用多条命令记录该键的值，而不是一条命令。

rewrite的触发机制主要有一下三个：

- 手动调用 bgrewriteaof 命令，如果当前有正在运行的 rewrite 子进程，则本次rewrite 会推迟执行，否则，直接触发一次 rewrite。
- 通过配置指令手动开启 AOF 功能，如果没有 RDB 子进程的情况下，会触发一次 rewrite，将当前数据库中的数据写入 rewrite 文件。
- 在 Redis 定时器中，如果有需要退出执行的 rewrite 并且没有正在运行的 RDB 或者 rewrite 子进程时，触发一次或者 AOF 文件大小已经到达配置的 rewrite 条件也会自动触发一次。

### AOF 后台重写

AOF 重写函数会进行大量的写入操作，调用该函数的线程将被长时间阻塞，所以 Redis 在子进程中执行 AOF 重写操作。

- 子进程进行 AOF 重写期间，Redis 进程可以继续处理客户端命令请求。
- 子进程带有父进程的内存数据拷贝副本，在不适用锁的情况下，也可以保证数据的安全性。

但是，在子进程进行 AOF 重启期间，Redis接收客户端命令，会对现有数据库状态进行修改，从而导致数据当前状态和 重写后的 AOF 文件所保存的数据库状态不一致。

为此，Redis 设置了一个 AOF 重写缓冲区，这个缓冲区在服务器创建子进程之后开始使用，当 Redis 执行完一个写命令之后，它会同时将这个写命令发送给 AOF 缓冲区和 AOF 重写缓冲区。

![示意图](https://upload-images.jianshu.io/upload_images/623378-fc2a68fc8c8ec78b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当子进程完成 AOF 重写工作之后，它会向父进程发送一个信号，父进程在接收到该信号之后，会调用一个信号处理函数，并执行以下工作：

- 将 AOF 重写缓冲区中的所有内容写入到新的 AOF 文件中，保证新 AOF 文件保存的数据库状态和服务器当前状态一致。
- 对新的 AOF 文件进行改名，原子地覆盖现有 AOF 文件，完成新旧文件的替换
- 继续处理客户端请求命令。

在整个 AOF 后台重写过程中，只有信号处理函数执行时会对 Redis 主进程造成阻塞，在其他时候，AOF 后台重写都不会阻塞主进程。

![示意图](https://upload-images.jianshu.io/upload_images/623378-6c78b84c7c03cd57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 后记
后续将会继续学习 Redis 复制和集群相关的知识，希望大家持久关注。
