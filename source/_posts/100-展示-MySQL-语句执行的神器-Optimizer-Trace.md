---
title: 100% 展示 MySQL 语句执行的神器-Optimizer Trace
tags: MySQL
abbrlink: '49488729'
date: 2020-08-02 21:23:58
---



在上一篇文章[《用Explain 命令分析 MySQL 的 SQL 执行》](https://mp.weixin.qq.com/s/88sGSpVYfGBREH-vZkl_jg)中，我们讲解了 Explain 命令的详细使用。但是它只能展示 SQL 语句的执行计划，无法展示为什么一些其他的执行计划未被选择，比如说明明有索引，但是为什么查询时未使用索引等。为此，MySQL 提供了 Optimizer Trace 功能，让我们能更加详细的了解 SQL 语句执行的所有分析，优化和选择过程。



如果您想更深入地了解为什么选择某个查询计划，那么优化器跟踪非常有用。虽然 EXPLAIN 显示选定的计划，但Optimizer Trace 能显示为什么选择计划：您将能够看到替代计划，估计成本以及做出的决策。本篇文章会详细讲解 Optimizer Trace 展示的所有相关信息，并且会辅之一些具体使用案例。



### 基于成本的执行计划

在了解 Optimizer Trace 的之前，我们先来学习一下 MySQL 是如何选择众多执行计划的。

MySQL 会使用一个基于成本(cost)的优化器对执行计划进行选择。每个执行计划的成本大致反应了该计划查询所需要的资源，主要因素是计算查询时将要访问的行数。优化器主要根据从存储引擎获取数据的统计数据和数据字典中元数据信息来做出判断。它会决定是使用全表扫描或者使用某一个索引进行扫描，也会决定表 join的顺序。优化器的作用如下图所示。



![_images/optimizer-overview.png](http://cdn.remcarpediem.net/2020-08-02-131930.png)

优化器会为每个操作标上成本，这些成本的基准单位或最小值是从磁盘读取随机数据页的成本，其他操作的成本都是它的倍数。所以优化器可以根据每个执行计划的所有操作为其计算出总的成本，然后从众多执行计划中，选取成本最小的来最终执行。

既然是基于统计数据来进行标记成本，就总会有样本无法正确反映整体的情况，这也是 MySQL 优化器有时做出错误优化的重要原因之一。

### Optimizer Trace 的基本使用

首先，我们来看一下具体如何使用 Optimizer Trace。默认情况下，该功能是关闭的，大家可以使用如下方式打开该功能，然后执行自己需要分析的 SQL 语句，然后再从 INFORMATION_SCHEMA 的 OPTIMIZER_TRACE中查找到该 SQL 语句执行优化的相关信息。

```sql
# 1. 打开optimizer trace功能 (默认情况下它是关闭的):
SET optimizer_trace="enabled=on";
SELECT ...; # 这里输入你自己的查询语句
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
# 当你停止查看语句的优化过程时，把optimizer trace功能关闭
SET optimizer_trace="enabled=off";
```

这个 OPTIMIZER_TRACE 表有4个列，如下所示：

- `QUERY`：表示我们的查询语句。
- `TRACE`：表示优化过程的JSON格式文本。
- `MISSING_BYTES_BEYOND_MAX_MEM_SIZE`：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
- `INSUFFICIENT_PRIVILEGES`：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是`1`，我们暂时不关心这个字段的值。

其中，信息最多也最为重要的就是第二列 TRACE，它也是我们后续分析的重点。

### TRACE 列的基本格式

TRACE 列的内容是一个超级大的 JSON 数据，直接展开然后一条一条解析估计能看到大伙脑壳疼。

![OIP 2](http://cdn.remcarpediem.net/2020-08-02-131653.png)

所以，我们先来看一下这坨大 JSON 的骨架。它有三大块内容，也代表着 SQL 语句处理的三个阶段，分别为准备阶段，优化阶段和执行阶段。

![opt_three_step](http://cdn.remcarpediem.net/2020-08-02-131701.png)



接下来，我们详细介绍一个案例，在案例中介绍涉及到的具体字段和含义。



### 为什么查询未走索引而是全表扫描

首先，SQL 语句查询不使用索引的情况有很多，我们这里只讨论因为基于成本的优化器认为全表查询执行计划的成本低于走索引执行计划的情况。

如下图这个场景，明明 val 列上有索引，并且 val 现存值也有一定差异性，为什么没有使用索引进行查询呢？

![](http://cdn.remcarpediem.net/2020-08-02-131719.png)



我们按照上文使用 Optimizer Trace 找到其 join_optimization 中 range_analysis 相关数据，它会展示 where 从句范围查询过程中索引的选择情况



![](http://cdn.remcarpediem.net/2020-08-02-131730.png)

由上图可以看出，MySQL 对比了全表扫描和使用 val 作为索引两个方案的成本，最后发现虽然全表扫描需要扫描更多的行，但是成本更低。所以选择了全表扫描的执行方案。

这是为什么呢？明明使用 val 索引可以少扫描 4 行。这其实涉及 InnoDB 中使用索引查询数据行的原理。



Innodb引擎查询记录时在无法使用索引覆盖(也就是需要查询的数据多与索引值，比如该例子中，我要查name，而索引列是 val)的场景下，需要做回表操作获取记录的所需字段，也就是说，通过索引查出主键，再去查数据行，取出对应的列，这样势必是会多花费成本的。

所以在回表数据量比较大时，经常会出现 Mysql 对回表操作查询代价预估代价过大而导致不使用索引的情况。


一般来说，当SQL 语句查询超过表中超过大概五分之一的记录且不能使用覆盖索引时，会出现索引的回表代价太大而选择全表扫描的现象。且这个比例随着单行记录的字节大小的增加而略微增大。



通过 range_analysis 中的相关数据也可以对 where 从句使用多个索引列，如何选择执行时使用的索引的情况进行分析。



### 小节

终于，介绍了有关于 MySQL 语句执行分析的 explain 和 Optimizer Trace，下一篇，我们将分析具体的死锁场景。



[个人博客，欢迎来玩](http://remcarpediem.net/)

![](http://cdn.remcarpediem.net/2020-05-26-144752.png)