---
title: 分布式一致性算法Paxos
date: 2017-04-16 14:42:35
tags: 
	- 算法
categories:
	- 分布式
---
 最近在学习zookeeper原理的时候了解到了paxos算法,看了几篇文章之后还是感觉有些迷糊,后来看了知行学社的[paxos视频](http://www.tudou.com/programs/view/e8zM8dAL6hM/)才对这个算法有了一定的了解,这里就做一下总结.

### Paxos简介
 Paxos是Lamport于1990年提出的一种基于消息传递而具有高度容错特性的分布式一致性算法.这个算法是分布式中最为重要的算法,Google Chubby的作者Mike Burrows说过这个世界上只有一种一致性算法,那就是Paxos,其他算法都是残次品.具体Paxos算法的详细内涵和故事背景大家可以参考知乎上的[回答](https://www.zhihu.com/question/19787937);

### Paxos的使用场景和假设
 我们都知道基于消息传递通信模型的分布式系列,不可避免的会发生以下错误:进程可能会慢,被杀死或在重启,消息可能会有延迟,丢失和重复.Paxos算法解决的问题就是在一个可能发生上述异常的分布式系统中如何就某个值达成一致,保证无论发生以上任何异常,都不会破坏决议的一致性。但是Paxos算法也有一定的使用假设。一个假设是在消息传递的过程中不会出现拜占庭将军问题：即虽然有可能一个消息被传递两次，但是绝对不会出现错误的消息。另一个假设是提议不会被反对，只能被同意或在被更新的提议替换。
 Paxos协议中有三种角色，每个节点可以扮演多个角色：
- 倡议者(Proposer):提议者可以提出提议(数值或在操作命令)以供投票表决。
- 接受者(Acceptor):接受者可以对提议者提出的提议进行投票表决，提议有超过半数的接收者投票即被选中。
- 学习者(Learner):学习者无投票者，只是从接收者那里获取哪个提议被选中。


 在Paxos算法中，一个或在多个Proposer都可以并发的提出提议；系统必须针对所有提议中的某个提议达成一致（超过半数大的接受者选中）；最多只能对一个确定的提议达成一致；只要超过半数的节点存活且可以互相通信，整个系统一定可以达成一致，即选择一个确定的提议。
 如果直接讲解Paxos算法，大家可能会有些难以理解，这里我们就按着视频里的顺序，先从简单的分布式一致性算法开始，然后不断进行优化，最后将其演变成Paxos算法。

### 图解Paxos主要流程
 Paxoso算法分为两个的阶段，我们就将其分别记为Phase1和Phase2.每个Proposer都持有一个独有的变量epoch,每个Acceptor都保存三个状态：lastest_prepared__epoch,accepted_epoch和accepted_value.lastest_prepared_epoch是指Acceptor授予访问权的Proposer的epoch值，accepted_epoch是Acceptor接受提议的Proposer的epoch值，而accepted_value就是Acceptor接受的提议值喽，他们的初始值都为null。
#### 阶段一
 Phase1过程中，Proposer向Acceptor发起`Prepare(epoch)`请求来获取访问权。将自己的epoch发送给Acceptor.而Acceptor只会接受比lastest_prepared_epoch更大的epoch,并给予访问权，并将epoch记录到lastest_prepared_epoch的值中，返回当前的accepted_epoch和accepted_value的值。在初始化状态下，二者都是null,所以返回的是<null,null>。如果epoch小于lastest_prepared_epoch则不授予访问权，并返回<error>。
![phase1](http://7xrxif.com1.z0.glb.clouddn.com/2017416-paxos-paxos1.png)
 如上图所示，Proposer1向5个Acceptor发送了Prepare(#1)的请求，其中前三个请求顺利到达，Acceptor授予访问权，返回<null,null>，并修改lastest_prepared_epoch为1。而后两个请求发生了网络延迟,一直未到达相应的Acceptor。
 在阶段一中，Proposer需要获得半数以上的Acceptor的访问权和对应的一组value的取值才会进行第二阶段，这样才会确保，一个Proposer提出的确定的议案会被另外一个Proposer发现，从而在阶段二中会进行正确的操作。
#### 阶段二
 第二阶段采取“后者认同前者”的原则进行。在肯定旧epoch无法生成确定性取值时，新的epoch会提交自己的取值，不会冲突；一旦旧epoch形成了确定性取值，那么该proposer一定可以获得该取值，并且会认同该取值，不会破坏。
 如果Proposer在第一阶段获取的value值都是null,则旧epoch无法形成确定性取值，此时让自己的<epoch,V>成为确定性取值：
- 向epoch对应的所有acceptor提交取值<epoch,V>
- 如果收到半数以上的成功应答，则返回<ok,V>
- 否则返回<error>

 如果value的取值不为null,则认同最大accepted_epoch对应的取值f,使<epoch，f>成为确定性取值，其中epoch是自己的epoch.
- 如果f出现半数以上，则说明f已经是确定性取值了，直接返回<ok,f>
- 否则，向epoch所对应的acceptor提交取值<epoch,f>

 Acceptor在接收到accept(epoch,V)的请求之后，先查看epoch是不是自己记录的lastest_prepared_epoch，如果是则设置<accepted_epoch,accepted_value> = <prepared_epoch,v> 。否在则会返回error


![paxos2](http://7xrxif.com1.z0.glb.clouddn.com/2017416-paxos-paxos2.png)

 如上图所示，由于在阶段一中Proposer1接受到的<accepted_epoch>和<accepted_value>值都为null,所以，决定将自己的值设置为确定值，于是发送accept(1,V1)请求。Acceptor1接受到了这个请求，检查lastest_prepared_epoch也等于1,所以将自己存储的<accepted_epoch,accepted_value>设置为<1,V1>。而Proposer1的另外两个accept请求发生了网络延迟。
 如果此时，Proposer2向Acceptor进行propose会怎么样呢？我们来模拟propose来分析一下。


![paxos3](http://7xrxif.com1.z0.glb.clouddn.com/2017416-paxos-paxos3.png)

 如上图所示，Proposer2向Acceptor发送了prepare(#2)的请求，Acceptor1先检测一下发现2大于现在的lastest_prepared_epoch,所以同意发送访问权，将lastest_prepared_epoch修改为2，并将自己保存的accepted_epoch和acceped_value返回给Proposer2；Acceptor3的操作也是类似，只不过因为Proposer1发送的accept请求发生了延迟，所以Acceptor3返回的是<null,null>；而Acceptor5的操作和我们在文章第一张图中的Acceptor1的操作相同，他们都是第一次接收到prepare请求。
 然后Proposer2进行第二阶段的操作，从所有的返回数据中，找到accepted_epoch最大的那个accepted_value.这里就是Acceptor返回的<1,V1>，所以，Proposer2会尽力让V1成为确定值，所以它向Acceptor发送accept(2,V1)的请求。然后Acceptor1,Acceptor3,Acceptor5三个Acceptor接受了这个accept请求，更新自己的<accepted_epoch,accepted_value>。此时，已经有三个acceptor形成了一致性的值，所以V1就成了整个系统的确定性取值。
![paxos7.png](http://7xrxif.com1.z0.glb.clouddn.com/2017416-paxos-paxos7.png)

 那么Proposer1对Acceptor3发送的accept请求在此时达到Acceptor3会怎么样呢？Acceptor3发现当前lastest_prepared_epoch是2,所以直接拒绝了这个请求。

### 后记
 不清楚大家现在对Paxos算法的过程是否已经有了清楚的了解啊？那么我来问几个问题，大家可以考虑一下：
- 在本文的情景下，假如Proposer2向Acceptor2,3,4发送了prepare请求，而不是向Acceptor1,3,5发送的请求，会怎么样呢？
- 为什么强调prepare阶段时必须接受到一般以上Acceptor的返回，才能进行第二阶段?
&emsp;后续希望能够分析一下`Zookeeper`关于Paxos的具体使用场景和算法，希望大家多多关注。
