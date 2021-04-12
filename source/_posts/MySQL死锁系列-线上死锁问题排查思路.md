---
title: MySQL死锁系列-线上死锁问题排查思路
tags: mysql
abbrlink: 2245b524
date: 2020-09-28 23:18:42
---





### 前言

MySQL 死锁异常是我们经常会遇到的线上异常类别，一旦线上业务日间复杂，各种业务操作之间往往会产生锁冲突，有些会导致死锁异常。这种死锁异常一般要在特定时间特定数据和特定业务操作才会复现，并且分析解决时还需要了解 MySQL 锁冲突相关知识，所以一般遇到这些偶尔出现的死锁异常，往往一时没有头绪，不好处理。



本篇文章会讲解一下如果线上发生了死锁异常，如何去排查和处理。除了系列前文讲解的有关加锁和锁冲突的原理还，还需要对 MySQl 死锁日志和 binlog 日志进行分析。

![线上死锁异常分析](http://cdn.remcarpediem.net/2020-09-28-151927.png)

### 正文 

**日常工作中，应对各类线上异常都要有我们自己的 SOP (标准作业流程) ** ，这样不仅能够提高自己的处理问题效率，也有助于将好的处理流程推广到团队，提高团队的整体处理异常能力。

所以，面对线上偶发的 MySQL 死锁问题，我的排查处理过程如下：

1. 线上错误日志报警发现死锁异常

2. 查看错误日志的堆栈信息

3. 查看 MySQL 死锁相关的日志

4. 根据 binlog 查看死锁相关事务的执行内容

5. 根据上述信息找出两个相互死锁的事务执行的 SQL 操作，根据本系列介绍的锁相关理论知识，进行分析推断死锁原因

6. 修改业务代码



**根据1，2步骤可以找到死锁异常时进行回滚事务的具体业务，也就能够找到该事务执行的 SQL 语句。然后我们需要通过 3，4步骤找到死锁异常时另外一个事务，也就是最终获得锁的事务所执行的 SQL 语句，然后再进行锁冲突相关的分析。**

第一二步的线上错误日志和堆栈信息一般比较容易获得，第五步的分析 SQL 锁冲突原因中涉及的锁相关的理论在系列文章中都有介绍，没有了解的同学可以自行去阅读以下。

下面我们就来重点说一下其中的第三四步骤，也就是如何查看死锁日志和 binlog 日志来找到死锁相关的 SQL 操作。




### 死锁日志的获取



发生死锁异常后，我们可以直接使用 `show engine innodb status` 命令获取死锁信息，但是该命令只能获取最近一次的死锁信息。所以，我们可以通过开启 InnoDB 的监控机制来获取实时的死锁信息，它会周期性（每隔 15 秒）打印 InnoDb 的运行状态到 mysqld 服务的错误日志文件中。



InnoDb 的监控较为重要的有标准监控（Standard InnoDB Monitor）和 锁监控（InnoDB Lock Monitor），通过对应的系统参数可以将其开启。

```
-- 开启标准监控
set GLOBAL innodb_status_output=ON;
-- 关闭标准监控
set GLOBAL innodb_status_output=OFF;
-- 开启锁监控
set GLOBAL innodb_status_output_locks=ON;
-- 关闭锁监控
set GLOBAL innodb_status_output_locks=OFF;
```

另外，MySQL 提供了一个系统参数 `innodb_print_all_deadlocks` 专门用于记录死锁日志，当发生死锁时，死锁日志会记录到 MySQL 的错误日志文件中。

```
set GLOBAL innodb_print_all_deadlocks=ON;
```

### 死锁日志的分析

通过上述手段，我们可以拿到死锁日志，下图是我做实验触发死锁异常时获取的日志(省略的部分信息)。

![](http://cdn.remcarpediem.net/2020-10-01-012810.png)

该日志会列出死锁发生的时间，死锁相关的事务，并显示出两个事务(可惜，多事务发生死锁时，也只显示两个事务)在**发生死锁时执行的 SQL 语句、持有或等待的锁信息和最终回滚的事务**。

下面，我们来一段一段的解读该日志中给出的信息，我们按照图中标注的顺序来介绍：

```bash
TRANSACTION 2078, ACTIVE 74 sec starting index read // -1 事务一的基础信息，包括事务ID、活跃时间，当前运行状态
```

表示的是 ACTIVE 74 sec 表示事务活动时间，starting index read 为事务当前正在运行的状态，可能的事务状态有：fetching rows，updating，deleting，inserting, starting index read 等状态。

```bash
mysql tables in use 1, locked 1  // -2 使用一个table，并且有一个表锁
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1  // -3 涉及的锁结构和内存大小 
```

tables in use 1 表示有一个表被使用，locked 1 表示有一个表锁。LOCK WAIT 表示事务正在等待锁，3 lock struct(s) 表示该事务的锁链表的长度为 3，每个链表节点代表该事务持有的一个锁结构，包括表锁，记录锁或 autoinc 锁等。heap size 1136 为事务分配的锁堆内存大小。

2 row lock(s) 表示当前事务持有的行锁个数，通过遍历上面提到的 11 个锁结构，找出其中类型为 LOCK_REC 的记录数。undo log entries 1 表示当前事务有 1 个 undo log 记录，说明该事务已经更新了 1条记录。

下面就是死锁日志中最为重要的持有或者待获取锁信息，如图中-5和-6行所示，**通过它可以分析锁的具体类型和涉及的表，这些信息能辅助你按照系列文章的锁相关的知识来分析 SQL 的锁冲突**。

```
RECORD LOCKS space id 2 page no 4 n bits 80 index PRIMARY of table `test`.`t` trx id 2078 lock_mode X locks rec but not gap  // -5 具体持有锁的信息
RECORD LOCKS space id 2 page no 4 n bits 80 index PRIMARY of table `test`.`t` trx id 2078 lock_mode X locks rec but not gap waiting // -6 等待获取锁的信息
```

在 [《锁类型和加锁原理》](https://mp.weixin.qq.com/s/QVEUIfD0RBbtvUDORaz2vQ) 一文中我们说过，一共有四种类型的行锁：记录锁，间隙锁，Next-key 锁和插入意向锁。这四种锁对应的死锁日志各不相同，如下：

- 记录锁（LOCK_REC_NOT_GAP）: lock_mode X locks rec but not gap
- 间隙锁（LOCK_GAP）: lock_mode X locks gap before rec
- Next-key 锁（LOCK_ORNIDARY）: lock_mode X
- 插入意向锁（LOCK_INSERT_INTENTION）: lock_mode X locks gap before rec insert intention

所以，按照死锁日志，我们发现事务一持有了 test.t 表上的记录锁，并且等待另一个记录锁。



通过死锁日志，我们可以找到最终获得锁事务最后执行的 SQL，但是如果该事务执行了多条 SQL，这些信息就可能不够用的啦，我们需要完整的了解该事务所有执行的 SQL语句。这时，我们就需要从 binlog 日志中获取。

### binlog的获取和分析

**binlog 日志会完整记录事务执行的所有 SQL，借助它，我们就能找到最终获取锁事务所执行的全部 SQL。**然后再进行具体的锁冲突分析。



我们可以使用 MySQL 的命令行工具 `Mysqlbinlog` 远程获取线上数据库的 binlog 日志。具体命令如下所示：

```bash
Mysqlbinlog -h127.0.0.1 -u root -p --read-from-remote-server binlog.000001 --base64-output=decode-rows -v
```

其中 `--base64-output=decode-rows` 表示 row 模式 binlog日志，所以该方法只适用于 row 模式的 binlog日志，但是目前主流 MySQL 运维也都是把 binlog 日志设置为 row 模式，所以这点限制也就无伤大雅。`-v` 则表示将行事件重构成被注释掉的伪SQL语句。



我们可以**通过死锁日志中死锁发生的具体事件和最终获取锁事务正在执行的SQL的参数信息找到 binlog 中该事务的对应信息**，比如我们可以直接通过死锁日志截图中的具体的时间 10点57分和 Tom1、Teddy2 等 SQL 的具体数据信息在 binlog 找到对应的位置，具体如下图所示。



![](http://cdn.remcarpediem.net/2020-09-28-151942.png)

根据 binlog 的具体信息，我们可以清晰的找到最终获取锁事务所执行的所有 SQL 语句，也就能找到其对应的业务代码，接下来我们就能进行具体的锁冲突分析。



### 小节

死锁系列终于告一段落，如果大伙有什么疑问或者文中有什么错误，欢迎在下方留言讨论。也希望大家继续持续关注。


[个人博客，欢迎来玩](http://remcarpediem.net/)

![](http://cdn.remcarpediem.net/2020-05-26-144752.png)