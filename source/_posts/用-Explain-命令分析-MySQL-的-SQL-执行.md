---
title: 用 Explain 命令分析 MySQL 的 SQL 执行
date: 2020-06-02 22:06:48
tags:
---

大纲

解析为什么需要使用 explain 命令来了解 sql 的执行



【一整段是从其他地方copy的，需要改写，并且加入自己死锁系列的问题】

在实际数据库项目开发中，由于我们不知道实际查询时数据库里发生了什么，也不知道数据库是如何扫描表、如何使用索引的，因此，我们能感知到的就只有SQL语句的执行时间。尤其在数据规模比较大的场景下，如何写查询、优化查询、如何使用索引就显得很重要了。

那么，问题来了，在查询前有没有可能估计下查询要扫描多少行、使用哪些索引呢？

答案是肯定的。以MySQL为例，MySQL通过explain命令输出执行计划，对要执行的查询进行分析。

什么是执行计划呢？

简单来说，就是SQL在数据库中执行时的表现情况，通常用于SQL性能分析、优化等场景。

本文从MySQL的逻辑结构讲解，过渡到MySQL的查询过程，然后给出执行计划的例子并重点介绍执行计划的输出参数，从而理解为什么我们会选择文中建议的方案。



### MySQL 查询过程

如果能搞清楚MySQL是如何优化和执行查询的，对优化查询一定会有帮助。很多查询优化实际上就是遵循一些原则让优化器能够按期望的合理的方式运行。

下图是MySQL执行一个查询的过程。实际上每一步都比想象中的复杂，尤其优化器，更复杂也更难理解。本文只给予简单的介绍。

![mysql_sql_execute](http://cdn.remcarpediem.net/2020-06-02-145336.png)

MySQL查询过程如下：

- 客户端将查询发送到MySQL服务器
- 服务器先检查查询缓存，如果命中，立即返回缓存中的结果；否则进入下一阶段
- 服务器对SQL进行解析、预处理，再由优化器生成对象的执行计划
- MySQL根据优化器生成的执行计划，调用存储引擎API来执行查询
- 服务器将结果返回给客户端，同时缓存查询结果

### 执行计划

MySQL会解析查询，并创建内部数据结构（解析树），并对其进行各种优化，包括重写查询、决定表的读取顺序、选择合适的索引等。

用户可通过关键字提示（hint）优化器，从而影响优化器的决策过程。也可以通过通过优化器解释（explain）优化过程的各个因素，使用户知道数据库是如何进行优化决策的，并提供一个参考基准，便于用户重构查询和数据库表的schema、修改数据库配置等，使查询尽可能高效。



#### select_type

查询数据的操作类型，有如下

- simple 简单查询，不包含子查询或 union

  ![select_type_simple](http://cdn.remcarpediem.net/2020-06-08-142155.png)

- primary 包含复杂的子查询，最外层查询标记为该值

- derived 在 from 列表中包含的子查询被标记为该值，MySQL 会递归执行这些子查询，把结果放在临时表

  ![select_type_primary](http://cdn.remcarpediem.net/2020-06-08-142203.png)

- subquery 在 select 或者 where 里包含子查询，被标记为该值

  ![select_type_subquery](http://cdn.remcarpediem.net/2020-06-08-144233.png)

- dependent subquery：子查询中的第一个 select，取决于外侧的查询

  ![select_type_d_subquery](http://cdn.remcarpediem.net/2020-06-08-144451.png)

- union 如果第二个 select 出现在 union 之后，则被标记为该值；若 union 包含在 from 的子查询中，外层select 被标记为 derived

- union result 从 union 表获取结果的 select

- ![select_type_union](http://cdn.remcarpediem.net/2020-06-08-145001.png)

- dependent union，

  ![select_type_d_union](http://cdn.remcarpediem.net/2020-06-08-145226.png)

#### type

表的连接类型，其值，性能由高到底排列如下。

- system 表只有一行记录，相当于系统表

  ![select_type_primary](http://cdn.remcarpediem.net/2020-06-08-142203.png)

- const 通过索引一次就找到，只匹配一行数据，用于常数值比较PRIMARY KEY 或者 UNIQUE索引。

  ![select_type_simple](http://cdn.remcarpediem.net/2020-06-08-142155.png)

- eq_ref 唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常用于主键或唯一索引扫描。对于每个来自前边的表的行组合，从该表中读取一行。这可能是最好的连接类型，除了 const 类型。它用在一个索引的所有部分被连接并且索引是 UNIQUE或者 PRIMARY KEY时。

- ref 非唯一性索引扫描，返回匹配某个单独值的所有行，用于 = < 或 > 操作符带索引的列。

- range 只检查给定范围的行，使用一个索引来选择行，一般使用 between > 或者 <

- index 只遍历索引树

- ALL 全表扫描

  

  

  前5种情况都是理想的索引的情况。通常优化至少到range级别，最好能优化到ref。 

#### possible_keys

#### key

#### key_len

#### ref

#### rows

#### filtered

#### extra

包含不适合在其他列中显示但十分重要的额外信息。常见的值如下

- using filesort MySQL 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，若出现该值，应该优化 SQL 语句
- using temporary 使用临时表保存中间结果，比如，MySQL 在对查询结果排序时使用临时表，常用于 order by 和 group by，如果出现该值，应该优化 SQL
- using index 表示 select 操作使用了覆盖索引，避免了访问表的数据行，效率不错
- using where where 子句用于限制哪一行
- using join buffer 使用连接缓存
- distinct 发现第一个匹配后，停止为当前的行组合搜索更多的行

### 后记



