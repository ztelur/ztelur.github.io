---
title: 'MySQL探秘(八):InnoDB的事务'
tags: MySQL
abbrlink: 6edb90fb
date: 2018-12-10 21:45:39
---


&emsp;事务是数据库最为重要的机制之一，凡是使用过数据库的人，都了解数据库的事务机制，也对ACID四个基本特性如数家珍。但是聊起事务或者ACID的底层实现原理，往往言之不详，不明所以。所以，今天我们就一起来分析和探讨InnoDB的事务机制，希望能建立起对事务底层实现原理的具体了解。

![事务的四大特性](https://upload-images.jianshu.io/upload_images/623378-2e908e18a2de4210.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



&emsp;数据库事务具有ACID四大特性。ACID是以下4个词的缩写：
- 原子性(atomicity) ：事务最小工作单元，要么全成功，要么全失败 。
- 一致性(consistency)： 事务开始和结束后，数据库的完整性不会被破坏 。
- 隔离性(isolation) ：不同事务之间互不影响，四种隔离级别为RU（读未提交）、RC（读已提交）、RR（可重复读）、SERIALIZABLE （串行化）。
- 持久性(durability) ：事务提交后，对数据的修改是永久性的，即使系统故障也不会丢失 。

&emsp;下面，我们就以一个具体实例来介绍数据库事务的原理，并介绍InnoDB是如何实现ACID四大特性的。

### 示例介绍

&emsp;我们首先来看一下具体的示例。大家可以自己亲自试验一下，这样理解和记忆都会更加深刻。
&emsp;首先，使用如下的SQL语句创建两张表，分别是goods和trade，代表货物和交易。并向goods表中插入一条记录，id为1的货物数量为10。

```
CREATE TABLE goods (id INT, num INT, PRIMARY KEY(id));
CREATE TABLE trade (id INT, goods_id INT, user_id INT, PRIMARY KEY(id));
INSERT INTO goods VALUES(1, 10);
```

&emsp;然后打开终端，连接数据库，开启会话一，先用BEGIN显示开启一个事务。会话一先将goods表中id为1的货物的数量减一，然后向trade表中添加一笔交易的记录，最后使用COMMIT显示提交事务。
&emsp;而会话二则先查询goods表中id为1的货物数量，然后向trade表中添加一笔交易记录，接着更新goods表中id为1的货物的数量，最后使用ROLLBACK进行事务的回滚。其中，两个会话中执行的具体语句和先后顺序如下图所示。

![示例具体语句和执行顺序](https://upload-images.jianshu.io/upload_images/623378-e1cb53add98d666d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


&emsp;这个示例可以体现数据库事务的很多特性，我们一一来介绍。首先会话一的操作2更新了id为1的货物的数量，但是会话二的操作5读出来的数量仍然是10，这体现了事务的隔离性，使用InnoDB的[多版本控制机制](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483698&idx=1&sn=3654042755c7ea0922e5d5b462930946&chksm=fc04c552cb734c443f771eb2075cd4d4ce85df85105805667cc4cf51a67ceed2f9ebdf1a8cde&token=1535405475&lang=zh_CN#rd)实现。
&emsp;会话二的操作7也要更新同种货物的数量，此时因为会话一的操作2已经更新了该货物的数量，InnoDB已经锁住了该记录的行锁，所以操作7会被阻塞，直到会话一COMMIT。但是会话一的操作4和会话二的操作7都是向trade表中插入记录，后者却不会因为前者而阻塞，因为二者插入的不是同一行记录。锁机制是一种常见的并发控制机制，它和多版本控制机制一起实现了InnoDB事务的隔离性，关于InnoDB锁相关的具体内容可以参考[InnoDB锁的类型和状态查询](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483694&idx=1&sn=671ad369f67441c7d1572110066d5695&chksm=fc04c54ecb734c58101f8ff020914f4cccaf6660742a6723b431066ca05d5e71365dfd8d4556&token=1535405475&lang=zh_CN#rd)和[InnoDB行锁算法](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483702&idx=1&sn=669fb9f413db0cc744bdb5b9ec8f725e&chksm=fc04c556cb734c40b3c90a55868619b8ca1f3e1f8763be2d6e7f0568d7c992409937c33b7e10&token=1535405475&lang=zh_CN#rd)。

&emsp;会话一事务最终使用COMMIT提交了事务而会话二事务则使用ROLLBACK回滚了整个事务，这体现了事务的原子性。即事务的一系列操作要么全部执行(COMMIT)，要么就全部不执行(ROLLBACK)，不存在只执行一部分的情况。InnoDB使用事务日志系统来实现事务的原子性。这里有的同学就会问了，如果中途连接断开或者Server Crash会怎么样。能怎么样，直接自动回滚呗。

&emsp;一旦会话一使用COMMIT操作提交事务成功后，那么数据一定会被写入到数据库中并持久的存储起来，这体现了事务的持久性。InnoDB使用[redo log机制](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483683&idx=1&sn=5225ab3481c38bb57297a36df8e62bce&chksm=fc04c543cb734c556574f9e5331ab70f0c8239d70197f70015f58d4ac3f5c4d0b1260f0478e3&token=1535405475&lang=zh_CN#rd)来实现事务的持久性。

&emsp;而事务的一致性比较难以理解，简单的讲在事务开始时，此时数据库有一种状态，这个状态是所有的MySQL对象处于一致的状态，例如数据库完整性约束正确，日志状态一致等。当事务提交后，这时数据库又有了一个新的状态，不同的数据，不同的索引，不同的日志等。但此时，约束，数据，索引，日志等MySQL各种状态还是要保持一致性。 也就是说数据库从一个一致性的状态，变到另一个一致性的状态。事务执行后，并没有破坏数据库的完整性约束。

&emsp;下面我们就来详细讲解一下上述示例涉及的事务的ACID特性的具体实现原理。**总结来说，事务的隔离性由多版本控制机制和锁实现，而原子性、一致性和持久性通过InnoDB的redo log、undo log和Force Log at Commit机制来实现**。

### 原子性，持久性和一致性
&emsp;原子性，持久性和一致性主要是通过redo log、undo log和Force Log at Commit机制机制来完成的。redo log用于在崩溃时恢复数据，undo log用于对事务的影响进行撤销，也可以用于多版本控制。而Force Log at Commit机制保证事务提交后redo log日志都已经持久化。
&emsp;开启一个事务后，用户可以使用COMMIT来提交，也可以用ROLLBACK来回滚。其中COMMIT或者ROLLBACK执行成功之后，数据一定是会被全部保存或者全部回滚到最初状态的，这也体现了事务的原子性。但是也会有很多的异常情况，比如说事务执行中途连接断开，或者是执行COMMIT或者ROLLBACK时发生错误，Server Crash等，此时数据库会自动进行回滚或者重启之后进行恢复。

&emsp;我们先来看一下redo log的原理，redo log顾名思义，就是重做日志，每次数据库的SQL操作导致的数据变化它都会记录一下，**具体来说，redo log是物理日志，记录的是数据库页的物理修改操作**。如果数据发生了丢失，数据库可以根据redo log进行数据恢复。

&emsp;InnoDB通过Force Log at Commit机制实现事务的持久性，即当事务COMMIT时，必须先将该事务的所有日志都写入到redo log文件进行持久化之后，COMMIT操作才算完成。
&emsp;当事务的各种SQL操作执行时，即会在缓冲区中修改数据，也会将对应的redo log写入它所属的缓存。当事务执行COMMIT时，与该事务相关的redo log缓冲必须都全部刷新到磁盘中之后COMMIT才算执行成功。

![数据库日志和数据落盘机制](https://upload-images.jianshu.io/upload_images/623378-99cd9e5402f46d87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;redo log写入磁盘时，必须进行一次操作系统的fsync操作，防止redo log只是写入了操作系统的磁盘缓存中。参数innodb_flush_log_at_trx_commit可以控制redo log日志刷新到磁盘的策略，它的具体作用可以查阅[InnoDB的磁盘文件及落盘机制](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483683&idx=1&sn=5225ab3481c38bb57297a36df8e62bce&chksm=fc04c543cb734c556574f9e5331ab70f0c8239d70197f70015f58d4ac3f5c4d0b1260f0478e3&token=1535405475&lang=zh_CN#rd)



&emsp;redo log全部写入磁盘后事务就算COMMIT成功了，但是此时事务修改的数据还在内存的缓冲区中，称其为脏页，这些数据会依据检查点(CheckPoint)机制择时刷新到磁盘中，然后删除相应的redo log，但是如果在这个过程中数据库Crash了，那么数据库重启时，会依据redo log file将那些还在内存中未更新到磁盘上的数据进行恢复。

&emsp;数据库为了提高性能，数据页在内存修改后并不是每次都会刷到磁盘上。而是引入checkpoint机制，择时将数据页落盘，checkpoint记录之前的数据页保证一定落盘了，这样相关的redo log就没有用了(由于InnoDB redo log file循环使用，这时这部分日志就可以被覆盖)，checkpoint之后的数据页有可能落盘，也有可能没有落盘，所以checkpoint之后的redo log file在崩溃恢复的时候还是需要被使用的。InnoDB会依据脏页的刷新情况，定期推进checkpoint，从而减少数据库崩溃恢复的时间。检查点的信息在第一个日志文件的头部。

&emsp;数据库崩溃重启后需要从redo log中把未落盘的脏页数据恢复出来，重新写入磁盘，保证用户的数据不丢失。当然，在崩溃恢复中还需要回滚没有提交的事务。由于回滚操作需要undo日志的支持，undo日志的完整性和可靠性需要redo日志来保证，所以崩溃恢复先做redo恢复数据，然后做undo回滚。

&emsp;在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log记录了数据在每个操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。

![数据和回滚日志的逻辑存储结构.jpg](https://upload-images.jianshu.io/upload_images/623378-6a680cf9597332b4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;undo log的存储不同于redo log，它存放在数据库内部的一个特殊的段(segment)中，这个段称为回滚段。回滚段位于共享表空间中。undo段中的以undo page为更小的组织单位。undo page和存储数据库数据和索引的页类似。因为redo log是物理日志，记录的是数据库页的物理修改操作。所以undo log的写入也会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。如上图所示，表空间中有回滚段和叶节点段和非叶节点段，而三者都有对应的页结构。

&emsp;我们再来总结一下数据库事务的整个流程，如下图所示。

![事务的相关流程](https://upload-images.jianshu.io/upload_images/623378-c46ad59604b75f65.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


&emsp;事务进行过程中，每次sql语句执行，都会记录undo log和redo log，然后更新数据形成脏页，然后redo log按照时间或者空间等条件进行落盘，undo log和脏页按照checkpoint进行落盘，落盘后相应的redo log就可以删除了。此时，事务还未COMMIT，如果发生崩溃，则首先检查checkpoint记录，使用相应的redo log进行数据和undo log的恢复，然后查看undo log的状态发现事务尚未提交，然后就使用undo log进行事务回滚。事务执行COMMIT操作时，会将本事务相关的所有redo log都进行落盘，只有所有redo log落盘成功，才算COMMIT成功。然后内存中的数据脏页继续按照checkpoint进行落盘。如果此时发生了崩溃，则只使用redo log恢复数据。



### 隔离性

&emsp;InnoDB事务的隔离性主要通过多版本控制机制和锁机制实现，具体可以参考[多版本控制](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483698&idx=1&sn=3654042755c7ea0922e5d5b462930946&chksm=fc04c552cb734c443f771eb2075cd4d4ce85df85105805667cc4cf51a67ceed2f9ebdf1a8cde&token=1535405475&lang=zh_CN#rd)，[InnoDB锁的类型和状态查询](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483694&idx=1&sn=671ad369f67441c7d1572110066d5695&chksm=fc04c54ecb734c58101f8ff020914f4cccaf6660742a6723b431066ca05d5e71365dfd8d4556&token=1535405475&lang=zh_CN#rd)和[InnoDB行锁算法](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483702&idx=1&sn=669fb9f413db0cc744bdb5b9ec8f725e&chksm=fc04c556cb734c40b3c90a55868619b8ca1f3e1f8763be2d6e7f0568d7c992409937c33b7e10&token=1535405475&lang=zh_CN#rd)三篇文章。

### 后记
&emsp;本来想一篇文章将MySQL的事务机制讲明白，写完自己读了一遍，还是发现内容有些晦涩难懂，复杂的知识本来就是很难讲明白的，夫夷以近，则游者众；险以远，则至者少，希望读者以本文作为一篇指引性的文章，自己再去更加深入的地方去探秘。不过，能将复杂知识讲解的通俗简单也是一项很大的本领，文字和讲解能力还是需要提示的。

- [Mysql探索(一):B-Tree索引
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483664&idx=1&sn=a4aea45edf13b367ee17539eaff4874b&chksm=fc04c570cb734c66447aec4344288025bfe6ba7d715af31dc6d60d65411cd90a05d9b02e749d&token=451486072&lang=zh_CN#rd)
- [数据库内部存储结构探索
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483669&idx=1&sn=de5770a2c732a688b6377b4201bf1577&chksm=fc04c575cb734c63fb5da0a871c5447c0cbbaea2a0a39d3896058b546e3d3a85575f575faf4b&token=451486072&lang=zh_CN#rd)
- [MySQL探秘(二)：SQL语句执行过程详解
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483673&idx=1&sn=cba5118dd4705035c40089a9e59305a9&chksm=fc04c579cb734c6fbc0e67006493d5727ed62262ac243ec74ad6c088cb4e3bcd53dfad73caaf&token=451486072&lang=zh_CN#rd)
- [MySQL探秘(三):InnoDB的内存结构和特性
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483676&idx=1&sn=b82135c479c806d2b97d026e143f346a&chksm=fc04c57ccb734c6a530b209b3d78de96c30291228e2296179565cc367107df9bc05bcc325c1c&token=451486072&lang=zh_CN#rd)
- [MySQL探秘(四):InnoDB的磁盘文件及落盘机制
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483683&idx=1&sn=5225ab3481c38bb57297a36df8e62bce&chksm=fc04c543cb734c556574f9e5331ab70f0c8239d70197f70015f58d4ac3f5c4d0b1260f0478e3&token=451486072&lang=zh_CN#rd)
- [MySQL探秘(五):InnoDB锁的类型和状态查询
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483694&idx=1&sn=671ad369f67441c7d1572110066d5695&chksm=fc04c54ecb734c58101f8ff020914f4cccaf6660742a6723b431066ca05d5e71365dfd8d4556&token=451486072&lang=zh_CN#rd)

- [MySQL探秘(六):InnoDB一致性非锁定读
](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483698&idx=1&sn=3654042755c7ea0922e5d5b462930946&chksm=fc04c552cb734c443f771eb2075cd4d4ce85df85105805667cc4cf51a67ceed2f9ebdf1a8cde&token=731065842&lang=zh_CN#rd)

![](https://upload-images.jianshu.io/upload_images/623378-7d960275042f309d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 参考

- [MySQL · 引擎特性 · InnoDB 事务系统](http://mysql.taobao.org/monthly/2017/12/01/)
- [MySQL · 引擎特性 · InnoDB 崩溃恢复过程
](http://mysql.taobao.org/monthly/2015/06/01/)