---
title: 当 Redis 发生高延迟时，到底发生了什么
tags: redis
abbrlink: ef4e619
date: 2019-11-10 23:22:19
---
Redis 是一种内存数据库，将数据保存在内存中，读写效率要比传统的将数据保存在磁盘上的数据库要快很多。但是 Redis 也会发生延迟时，这是就需要我们对其产生原因有深刻的了解，以便于快速排查问题，解决 Redis的延迟问题


### 一条命令执行过程

在本文场景下，延迟 (latency) 是指从客户端发送命令到客户端接收到命令返回值的时间间隔。所以我们先来看一下 Redis 一条命令执行的步骤，其中每个步骤出问题都可能导致高延迟。

![](/images/19_1112/image1.webp)

上图是 Redis 客户端发送一条命令的执行过程示意图，绿色的是执行步骤，而蓝色的则是可能出现的导致高延迟的原因。

网络连接限制、网络传输速率和CPU性能等是所有服务端都可能产生的性能问题。但是 Redis 有自己独有的可能导致高延迟的问题：命令或者数据结构误用、持久化阻塞和内存交换。

而且更为致命的是，Redis 采用单线程和事件驱动的机制来处理网络请求，分别有对应的连接应答处理器，命令请求处理器和命令回复处理器来处理客户端的网络请求事件，处理完一个事件就继续处理队列中的下一个。一条命令处理出现了高延迟会影响接下来处于排队状态的其他命令。有关 Redis 事件处理机制的可以参考[本篇文章](http://remcarpediem.net/article/1aa2da89/)。

![](/images/19_1112/image2.webp)


对于高延迟，Redis 原生提供慢查询统计功能，执行 slowlog get {n} 命令可以获取最近的 n 条慢查询命令，默认对于执行超过10毫秒(可配置)的命令都会记录到一个定长队列中，线上实例建议设置为1毫秒便于及时发现毫秒级以上的命令。
```
# 超过 slowlog-log-slower-than 阈值的命令都会被记录到慢查询队列中
# 队列最大长度为 slowlog-max-len
slowlog-log-slower-than 10000
slowlog-max-len 128
```

如果命令执行时间在毫秒级，则实例实际OPS只有1000左右。慢查询队列长度默认128，可适当调大。慢查询本身只记录了命令执行时间，不包括数据网络传输时间和命令排队时间，因此客户端发生阻塞异常 后，可能不是当前命令缓慢，而是在等待其他命令执行。需要重点比对异常和慢查询发生的时间点，确认是否有慢查询造成的命令阻塞排队。

slowlog的输出格式如下所示。第一个字段表示该条记录在所有慢日志中的序号，最新的记录被展示在最前面；第二个字段是这条记录被记录时的系统时间，可以用 date 命令来将其转换为友好的格式第三个字段表示这条命令的响应时间，单位为 us (微秒)；第四个字段为对应的 Redis 操作。
```
> slowlog get
1) 1) (integer) 26
   2) (integer) 1450253133
   3) (integer) 43097
   4) 1) "flushdb"
```

下面我们就来依次看一下不合理地使用命令或者数据结构、持久化阻塞和内存交换所导致的高延迟问题。

### 不合理的命令或者数据结构

一般来说 Redis 执行命令速度都非常快，但是当数据量达到一定级别时，某些命令的执行就会花费大量时间，比如对一个包含上万个元素的 hash 结构执行 hgetall 操作，由于数据量比较大且命令算法复杂度是 O(n)，这条命令执行速度必然很慢。

这个问题就是典型的不合理使用命令和数据结构。对于高并发的场景我们应该尽量避免在大对象上执行算法复杂度超过 O(n) 的命令。对于键值较多的 hash 结构可以使用 scan 系列命令来逐步遍历，而不是直接使用 hgetall 来全部获取。

Redis 本身提供发现大对象的工具，对应命令：redis-cli-h {ip} -p {port} bigkeys。这条命令会使用 scan 从指定的 Redis DB 中持续采样，实时输出当时得到的 value 占用空间最大的 key 值，并在最后给出各种数据结构的 biggest key 的总结报告。

```
> redis-cli -h host -p 12345 --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest hash   found so far 'idx:user' with 1 fields
[00.00%] Biggest hash   found so far 'idx:product' with 3 fields
[00.00%] Biggest hash   found so far 'idx:order' with 14 fields
[02.29%] Biggest hash   found so far 'idx:fund' with 16 fields
[02.29%] Biggest hash   found so far 'idx:pay' with 69 fields
[04.45%] Biggest set    found so far 'indexed_word_set' with 1482 members
[05.93%] Biggest hash   found so far 'idx:address' with 159 fields
[11.79%] Biggest hash   found so far 'idx:reply' with 196 fields

-------- summary -------

Sampled 1484 keys in the keyspace!
Total key length in bytes is 13488 (avg len 9.09)

Biggest    set found 'indexed_word_set' has 1482 members
Biggest   hash found 'idx:的' has 196 fields

0 strings with 0 bytes (00.00% of keys, avg size 0.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
2 sets with 1710 members (00.13% of keys, avg size 855.00)
1482 hashs with 6731 fields (99.87% of keys, avg size 4.54)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

#### 持久化阻塞

对于开启了持久化功能的Redis节点，需要排查是否是持久化导致的阻 塞。持久化引起主线程阻塞的操作主要有：fork 阻塞、AOF刷盘阻塞。

fork 操作发生在 RDB 和 AOF 重写时，Redis 主线程调用 fork 操作产生共享内存的子进程，由子进程完成对应的持久化工作。如果 fork 操作本身耗时过长，必然会导致主线程的阻塞。

![](/images/19_1112/image3.webp)

Redis 执行 fork 操作产生的子进程内存占用量表现为与父进程相同，理论上需要一倍的物理内存来完成相应的操作。但是 Linux 具有写时复制技术 (copy-on-write)，父子进程会共享相同的物理内存页，当父进程处理写请求时会对需要修改的页复制出一份副本完成写操作，而子进程依然读取 fork 时整个父进程的内存快照。所以，一般来说，fork 不会消耗过多时间。


可以执行`info stats`命令获取到 latest_fork_usec 指标，表示 Redis 最近一次 fork 操作耗时，如果耗时很大，比如超过1秒，则需要做出优化调整。

```
> redis-cli -c -p 7000 info | grep -w latest_fork_usec
latest_fork_usec:315
```

当我们开启AOF持久化功能时，文件刷盘的方式一般采用每秒一次，后 台线程每秒对AOF文件做 fsync 操作。当硬盘压力过大时，fsync 操作需要等待，直到写入完成。如果主线程发现距离上一次的 fsync 成功超过2秒，为了数据安全性它会阻塞直到后台线程执行 fsync 操作完成。这种阻塞行为主要是硬盘压力引起，可以查看 Redis日志识别出这种情况，当发生这种阻塞行为时，会打印如下日志：

```
Asynchronous AOF fsync is taking too long (disk is busy). \ 
Writing the AOF buffer without waiting for fsync to complete, \
this may slow down Redis.
```

也可以查看 info persistence 统计中的 aof_delayed_fsync 指标，每次发生 fdatasync 阻塞主线程时会累加。

```
>info persistence
loading:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0
```
### 内存交换

内存交换（swap）对于 Redis 来说是非常致命的，Redis 保证高性能的一个重要前提是所有的数据在内存中。如果操作系统把 Redis 使用的部分内存换出到硬盘，由于内存与硬盘读写速度差几个数量级，会导致发生交换后的 Redis 性能急剧下降。识别 Redis 内存交换的检查方法如下：

```
>redis-cli -p 6383 info server | grep process_id # 查询 redis 进程号
>cat /proc/4476/smaps | grep Swap # 查询内存交换大小
Swap: 0 kB 
Swap: 4 kB 
Swap: 0 kB 
Swap: 0 kB
```
如果交换量都是0KB或者个别的是4KB，则是正常现象，说明Redis进程内存没有被交换。

有很多方法可以避免内存交换的发生。比如说：
- 保证机器充足的可用内存
- 确保所有Redis实例设置最大可用内存(maxmemory)，防止极端情况下 Redis 内存不可控的增长。
- 降低系统使用swap优先级，如`echo10>/proc/sys/vm/swappiness`。

### 参考
- https://redis.io/topics/latency