---
title: MySQL死锁系列-加锁场景分析
tags: MySQL
abbrlink: 5a35e6d9
date: 2020-05-21 22:25:30
---

在上一篇文章[《锁的类型以及加锁原理》](https://mp.weixin.qq.com/s/QVEUIfD0RBbtvUDORaz2vQ)主要总结了 MySQL 锁的类型和模式以及基本的加锁原理，今天我们就从原理走向实战，分析常见 SQL 语句的加锁场景。了解了这几种场景，相信小伙伴们也能举一反三，灵活地分析真实开发过程中遇到的加锁问题。

如下图所示，**数据库的隔离等级，SQL 语句和当前数据库数据会共同影响该条 SQL 执行时数据库生成的锁模式，锁类型和锁数量**。

![](http://cdn.remcarpediem.net/2020-05-26-121712.png)

下面，我们会首先讲解一下隔离等级、不同 SQL 语句 和 当前数据库数据对生成锁影响的基本规则，然后再依次具体 SQL 的加锁场景。

###  隔离等级对加锁的影响

MySQL 的隔离等级对加锁有影响，所以**在分析具体加锁场景时，首先要确定当前的隔离等级**。

- 读未提交（Read Uncommitted 后续简称 RU）：可以读到未提交的读，基本上不会使用该隔离等级，所以暂时忽略。
- 读已提交（Read Committed 后续简称 RC）：存在幻读问题，**对当前读获取的数据加记录锁**。
- 可重复读（Repeatable Read 后续简称 RR）：不存在幻读问题，对当前读获取的数据加记录锁，同时对涉及的范围加间隙锁，防止新的数据插入，导致幻读。
- 序列化（Serializable）：从 MVCC 并发控制退化到基于锁的并发控制，不存在快照读，都是当前读，并发效率急剧下降，不建议使用。

这里说明一下，**RC 总是读取记录的最新版本，而 RR 是读取该记录事务开始时的那个版本**，虽然这两种读取的版本不同，但是都是快照数据，并不会被写操作阻塞，所以这种读操作称为 **快照读（Snapshot Read）**

MySQL 还提供了另一种读取方式叫**当前读（Current Read）**，它读的不再是数据的快照版本，而是数据的最新版本，并会对数据加锁，根据语句和加锁的不同，又分成三种情况：

- SELECT ... LOCK IN SHARE MODE：加共享(S)锁
- SELECT ... FOR UPDATE：加排他(X)锁
- INSERT / UPDATE / DELETE：加排他(X)锁

当前读在 RR 和 RC 两种隔离级别下的实现也是不一样的：**RC 只加记录锁，RR 除了加记录锁，还会加间隙锁，用于解决幻读问题**。

### 不同 SQL 语句对加锁的影响

不同的 SQL 语句当然会加不同的锁，总结起来主要分为五种情况：

- SELECT ... 语句正常情况下为快照读，不加锁；
- SELECT ... LOCK IN SHARE MODE 语句为当前读，加 S 锁；
- SELECT ... FOR UPDATE 语句为当前读，加 X 锁；
- 常见的 DML 语句（如 INSERT、DELETE、UPDATE）为当前读，加 X 锁；
- 常见的 DDL 语句（如 ALTER、CREATE 等）加表级锁，且这些语句为隐式提交，不能回滚。

其中，当前读的 SQL 语句的 where 从句的不同也会影响加锁，包括是否使用索引，索引是否是唯一索引等等。


### 当前数据对加锁的影响

SQL 语句执行时数据库中的数据也会对加锁产生影响。

比如一条最简单的根据主键进行更新的 SQL 语句，如果主键存在，则只需要对其加记录锁，如果不存在，则需要在加间隙锁。

至于其他非唯一性索引更新或者插入时的加锁也都不同程度的受到现存数据的影响，后续我们会一一说明。

### 具体场景分析

具体 SQL 场景分析主要借鉴何登成前辈的《MySQL 加锁处理分析》文章和 aneasystone 的系列文章，在他们的基础上进行了总结和整理。


我们使用下面这张 book 表作为实例，其中 id 为主键，ISBN（书号）为二级唯一索引，Author（作者）为二级非唯一索引，score（评分）无索引。

![](http://cdn.remcarpediem.net/2020-05-26-121732.png)

### UPDATE 语句加锁分析

下面，我们先来分析 UPDATE 相关 SQL 在使用较为简单 where 从句情况下加锁情况。其中的分析原则也适用于 UPDATE,DELETE 和 SELECT ... FOR UPDATE等当前读的语句。

#### 聚簇索引，查询命中

聚簇索引就是 InnoDB 存储引擎下的主键索引，具体可参考[《MySQL索引》](https://mp.weixin.qq.com/s/FX0bPiBIWWjSI9yRy715FQ)。

下图展示了使用 UPDATE book SET score = 9.2 WHERE ID = 10 语句命中的情况下在 RC 和 RR 隔离等级下的加锁，两种隔离等级下没有任何区别，都是对 ID = 10 这个索引加排他记录锁。


![](https://upload-images.jianshu.io/upload_images/623378-9aa0c26774b0ed32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 聚簇索引，查询未命中


下图展示了 UPDATE book SET score = 9.2 WHERE ID = 16 语句未命中时 RR 隔离级别下的加锁情况。

在 RC 隔离等级下，不需要加锁；而在 RR 隔离级别会在 ID = 16 前后两个索引之间加上间隙锁。

![](https://upload-images.jianshu.io/upload_images/623378-a71c4e540a4c9939.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

值得注意的是，间隙锁和间隙锁之间是互不冲突的，间隙锁唯一的作用就是为了防止其他事务的插入新行，导致幻读，所以加间隙 S 锁和加间隙 X 锁没有任何区别。

#### 二级唯一索引，查询命中

下图展示了 UPDATE book SET score = 9.2 WHERE ISBN = 'N0003' 在 RC 和 RR 隔离等级下命中时的加锁情况。

在 InnoDB 存储引擎中，二级索引的叶子节点保存着主键索引的值，然后再拿主键索引去获取真正的数据行，所以在这种情况下，二级索引和主键索引都会加排他记录锁。

![](https://upload-images.jianshu.io/upload_images/623378-f2df4b02fa76e924.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 二级唯一索引，查询未命中

下图展示了 UPDATE book SET score = 9.2 WHERE ISBN = 'N0008' 语句在 RR 隔离等级下未命中时的加锁情况，RC 隔离等级下该语句未命中不会加锁。

因为 N0008 大于 N0007，所以要锁住 (N0007,正无穷)这段区间，而 InnoDB 的索引一般都使用 Suprenum Record 和 Infimum Record 来分别表示记录的上下边界。Infimum 是比该页中任何记录都要小的值，而 Supremum 比该页中最大的记录值还要大，这两条记录在创建页的时候就有了，并且不会删除。

所以，在 N0007 和 Suprenum Record 之间加了间隙锁。


![](https://upload-images.jianshu.io/upload_images/623378-16d854a96f4275d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为什么不在主键上也加 GAP 锁呢？欢迎留言说出你的想法。

#### 二级非唯一索引，查询命中

下图展示了 UPDATE book SET score = 9.2 WHERE Author = 'Tom' 语句在 RC 隔离等级下命中时的加锁情况。

我们可以看到，在 RC 等级下，二级唯一索引和二级非唯一索引的加锁情况是一致的，都是在涉及的二级索引和对应的主键索引上加上排他记录锁。

![](https://upload-images.jianshu.io/upload_images/623378-f897052c7564c31a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是在 RR 隔离等级下，加锁的情况产生了变化，它不仅对涉及的二级索引和主键索引加了排他记录锁，还在非唯一二级索引上加了三个间隙锁，锁住了两个 Tom 索引值相关的三个范围。

那为什么唯一索引不需要加间隙锁呢？间隙锁的作用是为了解决幻读，防止其他事务插入相同索引值的记录，而唯一索引和主键约束都已经保证了该索引值肯定只有一条记录，所以无需加间隙锁。



![](https://upload-images.jianshu.io/upload_images/623378-92ce2a5dc5a2f787.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是，上图虽然画着 4 个记录锁，三个间隙锁，但是实际上间隙锁和它右侧的记录锁会合并成 Next-Key 锁。

所以实际情况有两个 Next-Key 锁，一个间隙锁(Tom60,正无穷)和两个记录锁。

#### 二级非唯一索引，查询未命中

下图展示了 UPDATE book SET score = 9.2 WHERE Author = 'Sarah' 在 RR 隔离等级下未命中的加锁情况，它会在二级索引 Rose 和 Tom 之间加间隙锁。而 RC 隔离等级下不需要加锁。

![](https://upload-images.jianshu.io/upload_images/623378-c137f7789e60a986.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 无索引

当 Where 从句的条件并不使用索引时，则会对全表进行扫描，在 RC 隔离等级下对所有的数据加排他记录锁。在RR 隔离等级下，除了给记录加锁，还会对记录和记录之间加间隙锁。和上边一样，间隙锁会和左侧的记录锁合并成 Next-Key 锁。

下图就是 UPDATE book SET score = 9.2 WHERE score = 22 语句在两种隔离等级下的加锁情况。


![](https://upload-images.jianshu.io/upload_images/623378-08c428ebfebf3bdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 聚簇索引，范围查询

上面介绍的场景都是 where 从句的等值查询，而范围查询的加锁又是怎么样的呢？我们慢慢来看。

下图是 UPDATE book SET score = 9.2 WHERE ID <= 25 在 RC 和 RR 隔离等级下的加锁情况。

RC 场景下与等值查询类似，只会在涉及的 ID = 10，ID = 18 和 ID = 25 索引上加排他记录锁。

![](https://upload-images.jianshu.io/upload_images/623378-c68266e9ee9da19e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


而在 RR 隔离等级下则有所不同，它会加上间隙锁，和对应的记录锁合并称为 Next-Key 锁。除此之外，它还会在(25, 30] 上分别加 Next-Key 锁。这一点是十分特殊的，具体原因还需要再探究。

#### 二级索引，范围查询


下图展示了 UPDATE book SET ISBN = N0001 WHERE score <= 7.9 在 RR 级别下的加锁情况。

![](https://upload-images.jianshu.io/upload_images/623378-71b90974da56dffe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 修改索引值

UPDATE 语句修改索引值的情况可以分开分析，首先 Where 从句的加锁分析如上文所述，多了一步 Set 部分的加锁。

下图展示了 UPDATE book SET Author = 'John' WHERE ID = 10 在 RC 和 RR 隔离等级下的加锁情况。除了在主键 ID 上进行加锁，还会对二级索引上的 Bob(就值) 和 John(新值) 上进行加锁。

![](https://upload-images.jianshu.io/upload_images/623378-4bcbe899fee57785.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### DELETE 语句加锁分析

一般来说，DELETE 的加锁和 SELECT FOR UPDATE 或 UPDATE 并没有太大的差异。

因为，在 MySQL 数据库中，执行 DELETE 语句其实并没有直接删除记录，而是在记录上打上一个删除标记，然后通过后台的一个叫做 purge 的线程来清理。从这一点来看，DELETE 和 UPDATE 确实是非常相像。事实上，DELETE 和 UPDATE 的加锁也几乎是一样的。

### INSERT 语句加锁分析

接下来，我们来看一下 Insert 语句的加锁情况。 

Insert 语句在两种情况下会加锁：

- 为了防止幻读，如果记录之间加有间隙锁，此时不能 Insert；
- 如果 Insert 的记录和已有记录造成唯一键冲突，此时不能 Insert；

除了上述情况，Insert 语句的锁都是隐式锁。隐式锁是 InnoDB 实现的一种延迟加锁的机制来减少加锁的数量。

隐式锁的特点是只有在可能发生冲突时才加锁，减少了锁的数量。另外，隐式锁是针对被修改的 B+Tree 记录，因此都是记录类型的锁，不可能是间隙锁或 Next-Key 类型。

具体 Insert 语句的加锁流程如下：

*   首先对插入的间隙加插入意向锁（Insert Intension Locks）

    *   如果该间隙已被加上了间隙锁或 Next-Key 锁，则加锁失败进入等待；
    *   如果没有，则加锁成功，表示可以插入；
*   然后判断插入记录是否有唯一键，如果有，则进行唯一性约束检查

    *   如果不存在相同键值，则完成插入
    *   如果存在相同键值，则判断该键值是否加锁

        *   如果没有锁， 判断该记录是否被标记为删除

            *   如果标记为删除，说明事务已经提交，还没来得及 purge，这时加 S 锁等待；
            *   如果没有标记删除，则报 duplicate key 错误；
        *   如果有锁，说明该记录正在处理（新增、删除或更新），且事务还未提交，加 S 锁等待；
*   插入记录并对记录加 X 记录锁；

### 后记

本文中讲解的 SQL 语句都是十分简单的，当 SQL 语句包含多个查询条件时，加锁的分析过程就往往更加复杂。我们需要使用 MySQL 相关的工具进行分析，并且有时甚至需要查询 MySQL 相关的日志信息来了解到底语句加了什么锁或者为什么产生死锁，下篇文章中我们就主要了解一下这些内容，请大家持续关注。



[个人博客，欢迎来玩](http://remcarpediem.net/)

![](http://cdn.remcarpediem.net/2020-05-26-144752.png)