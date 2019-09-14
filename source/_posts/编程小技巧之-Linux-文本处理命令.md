---
title: 编程小技巧之 Linux 文本处理命令
date: 2019-09-14 20:47:30
tags: 工具, Linux
---

合格的程序员都善于使用工具，正所谓君子性非异也，善假于物也。合理的利用 Linux 的命令行工具，可以提高我们的工作效率。

本文简单的介绍三个能使用 Linux 文本处理命令的场景，给大家开阔一下思路。希望大家阅读完这篇文章之后，要多加实践，将这些技巧内化到自己的日常工作习惯中，真正的提高效率。内化很重要，就像开玩笑所说的一样，即使我知道高内聚，低耦合的要求，了解 23 种设计模式和 6 大原则，熟读代码整洁之道，却仍然写不出优秀的代码。知道和内化到行为中区别还是很大的。

> 能不能让正确的原则指导正确的行动本身，其实就是区分是否是高手的一个显著标志。

程序员日常工作中往往要处理一些数据和文本，比如说统计一些服务日志文件信息，根据数据库数据生成一些处理数据的SQL和搜索文件内容等。可以直接通过编写代码处理，但不够便捷，因为有时候线上相关的代码环境依赖不一定具备。而直接使用 Linux 的文本处理命令可以很方便地处理这些问题。

### 日志文件捞数据

在工作中，我们往往需要对一些具有固定格式的文件进行信息统计，比如说根据 nginx 的 access.log 文件数据，计算出每个后端 API 接口的调用次数，并且排序。

nginx 的 access.log 文件文件格式配置如下所示，每个字段之间通过空格分隔开来。
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```
上述配置中字段含义如下：
- $remote_addr : 发送请求的源地址
- $remote_user : 发送请求的用户信息
- $time_local : 接收请求的本地时间
- $request : 请求信息，比如说 http 的 method 和 路径。
- $status : 请求状态，比如说 200、401或者 500。
- $body_bytes_sent : 请求 body 字节数。
- $http_referer : 域名。
- $http_user_agent : 用户端 agent 信息，一般就是浏览器信息
- $http_x_forwarded_for : 其他信息。

具体的一段 access.log 内容如下所示。

```
58.213.85.34 - - [11/Sep/2019:03:36:11 +0800] "POST /publish/pending/list HTTP/2.0" 200 1328 "https://remcarpediem.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36"
58.213.85.34 - - [11/Sep/2019:03:36:30 +0800] "GET /publish/search_inner?key=test HTTP/2.0" 200 34466 "https://remcarpediem.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36"
```
那么，我们可以通过下面命令来统计所有接口调用的次数，并且从大到小排序显示。

```
cat /var/log/nginx/access.log | awk '{print $7}' | awk -F'?' '{print $1}' | sort | uniq -c | sort -nr
```

这条命令涉及了 cat、awk、sort ， uniq 四个命令行工具和 | 连接符的含义，我们依次简单讲解一下它们的使用，感兴趣的同学可以自行去全面了解学习。

cat 命令是将文件内容打印到标准输出设备上，可以是终端，也可以是其他文件。比如说：
```
cat /var/log/nginx/access.log # 打印到终端
cat /var/log/nginx/access.log > copy.log # 打印到其他文件中
```

| 符号是管道操作符，它将的一个命令的 stdout 指向第二个命令的 stdin。在这条命令中 | 符号将 cat 命令的输出指向到 awk 命令的输入中。


awk 是贝尔实验室 1977 年搞出来的文本流处理工具，用于对具有固定格式的文件进行流处理。比如说 nginx 的 access.log 文件，它各个字段之间通过空格分隔开来，awk 就很适合处理此类文件。

`'{print $7}'` 就是 awk 的指令声明，表示打印出变量`$7`，`$7`则是 awk 内置的变量，代表按照分隔符分隔开来的第七个文本内容。对于 access.log 文件来说就是 `$request` 代表的路径相关的内容。 `$request` 的全部内容是`POST /publish/pending/list HTTP/2.0`，`$6` 对应 `POST`，而 `$7` 对应的就是 `/publish/pending/list`。

```
awk '{print $7}' Access.log # ''中是命令声明，后边跟着要操作的文件，也就是awk的输入流。
```
但是有些时候我们发现文本内容并不是按照空格进行分隔的，比如说 `$request` 内容可能为 `/publish/search_inner?key=test`，虽然是相同的 path，但是 query 不同，我们统计接口调用量时需要将 query 部分过滤掉。我们可以使用 awk 的 -F 指令指定分隔符。

```
awk -F'?' '{print $1}' 
# 可以将 /publish/search_inner?key=test 处理为 /publish/search_inner
```

sort 是专门用于排序的命令，它有多个参数：
- -n 按数值进行排序，默认是按照字符值排序，按照数值比较 10 > 2 但是按照字符值排序，2 >10 ，因为字符值会先比较首位，也就是 2 > 1。
- -r 默认是升序排列，这个参数指定按照逆序排列。
- -k N 指定按第N列排序，默认是第一个值

```
sort -nr Access.log # 按照数值逆序排序
```
最后一个命令是 uniq，它用于消除重复行，或者统计。

```
sort unsort.txt | uniq #  消除重复行
sort unsort.txt | uniq -c # 统计各行在文件中出现的次数，输入格式是[字数] [内容]
sort unsort.txt | uniq -d # 找出重复行
```
比如说`cat /var/log/nginx/access.log | awk '{print $7}' | awk -F'?' '{print $1}' | sort | uniq -c` 命令的输出如下所示，正好作为 `sort -nr`  的输入。
```
5 /announcement/pending/list
5 /announcement/search_inner
```

利用这些指令，我们可以通过 access.log 统计很多信息，比如下列这些信息( access.log 的信息配置不同，不可以直接照搬 )。

```
cat access.log | awk -F ‘^A’ ‘{if($5 == 500) print $0}’ 
#查找当前日志文件 500 错误的访问：
tail -f access.log | awk -F ‘^A’ ‘{if($6>1) print $0}’ 
#查找耗时超过 1s 的慢请求
```

### 数据库SQL

在业务迭代过程中，有些数据库数据可能需要使用脚本去修改，这是我们可以要根据一些数据生成对应的 SQL 命令，这里我们可以使用命令行工具快速生成。
比如说我们要将一系列订单状态有问题，需要将其恢复成正常的状态。你现在已经收集到了这批订单的信息。
```
oder_id name info good_id
100000  '裤子' '山东' 1000
100001  '上衣' '江苏' 1000
100002  '内衣' '内蒙古' 1000
........
100003  '袜子' '江西' 1000
```
那么你可以使用如下命令直接生成对应的 SQL 语句。
```
cat ErrorOrderIdFile | awk '{print"UPDATE ORDER SET state = 0 WHERE resource_id = "$1}'
```
这里 `''`中都是 awk 的命令内容，而`""`中是打印的纯文本，所以我们可以将需要补充的 SQL 命令打印出来。

### 代码信息统计


在大公司中，各个团队往往会公开出自己的接口给兄弟团队调用，但是随着版本地快速迭代，公开的接口越来越多，想要关闭掉又往往不清楚上游调用方是哪个部门的，轻易不敢关闭或者修改。这时，如果你能访问整个公司的代码库，就可以通过下面的脚本搜索一下项目中是否出现该接口相关的关键词。

笔者公司团队中微服务间通过 FeignClient 相互调用，所以对于这种情况，可以直接将搜索出对应 FeignClient 的函数名出现的文件名称。

下面是一段在多个项目中统计某些关键词出现次数，并打印出文件名的 bash 脚本。

```
#!/bin/bash
keyword=$1 # 将bash命令的第一个参数赋值给 keyword
prefix=`echo $keyword | tr -s '.' '|' | sed 's/$/|/'` # 处理前缀
files=`find services -name "*.java" -or -name "*.js" | xargs grep -il $keyword` 
# 最关键的一条，搜索services文件夹下文件名后缀为.java或者.js并且内容中有关键词的文件名称。
if [ -z "$files" ];then
	echo ${prefix}0
fi
# 打印
for f in $files;do
echo "$prefix$f"
done
```
我们只看一下最关键的 find 命令，其他的命令比如 tr 或者 sed，大家可以自行了解学习。

find 用于查找文件，可以按照文件名称、文件操作权限、文件属主、文件访问时间等条件来查找。

```
find services -name "*.java" -or -name "*.js" # 搜索 services 文件夹下
find . -atime 7 -type f -print 
# -atime是访问时间，-type 是文件类型，区分文件和目录，查找最近7天访问过的文件。
find . -type f -user remcarpediem -print// 找用户 remcarpediem 所拥有的文件
find . ! -name "*.java" -print # !是否定参数，查找所有不是以 .java 结尾的文件。
find . -type f -name "*.java" -delete # find 之后的操作，可以删除当前目录下所有的 java 文件
find . type f -name "*.java" | xargs rm # 上边语句的另外一种写法
```

xargs 命令能够将输入数据转化为特定命令的命令行参数，比如说多行变一行等，串联多个命令行，比如说上边 find 和 rm。

```
> ls 
Sentinel					groovy-engine					spring-cloud-bus-stream-binder-rocketmq
agent-demo					hash						spring-cloud-stream-binder-rabbit
> ls | xargs # 将 ls 的输出内容变成一行。
Sentinel agent-demo groovy-engine  hash spring-cloud-bus-stream-binder-rocketmq spring-cloud-stream-binder-rabbit
> echo "nameXnameXnameXname" | xargs -dX 
name name name name
# -d 选项可以自定义一个定界符，相信你已经了解 xargs 的大致作用了吧，按照分隔符拆分文本到一行，默认分隔符当时是回车了。
```

最后一个命令时 grep，它是文本搜索命令，它可以搜索文本内容的关键词。

```
grep remcarpediem file # 将 file 文件中的带有 remcarpediem 关键词的行。
grep -C10 remcarpediem file # 将 file 文件中的带有 remcarpediem 关键词前后10行的内容。
cat LOG.* | grep "FROM " | grep "WHERE" > b # 将日志中的所有带where条件的sql查找查找出来
grep -li remcarpediem file # 忽略大小写，并且打印出文件名称
```

现在大家在回头看一下这段 bash 脚本，是不是大致了解它执行的过程和原理啦。
```
files=`find services -name "*.java" -or -name "*.js" | xargs grep -il $keyword` 
```

### 后记

本文简单介绍了程序员日常工作中可能用到 Linux 命令的三个场景。大家可以根据自己的实际情况，来判断是否需要继续全面详细地学习相关的知识。毕竟只有能运用于实践，给自己工作产生价值的技术才是真技术。学习一项技术，就要坚持学以致用的目的。

[编程小技巧之 IDEA 的 Live Template](http://remcarpediem.net/2019/06/23/%E7%BC%96%E7%A8%8B%E5%B0%8F%E6%8A%80%E5%B7%A7%E4%B9%8B-IDEA-%E7%9A%84-Live-Template/)

![](http://remcarpediem.net/images/logo.png)


