---
title: Docker系列之Jenkins自动化部署
tags:
  - Docker
  - Jenkins
categories:
  - 运维
abbrlink: b4f6f76f
date: 2017-07-02 21:23:35
---

 Devops的概念已经火了很久了，我一直想对这方面进行一定的了解；再加上实验室项目环境依赖比较复杂，希望使用Docker来解决，所以最近就好好研究了一波Docker的相关实践和原理。这里整理一下，希望组成一个系列，从实践到原理详细讲解一下Docker的使用。
 第一篇就讲一下Jenkins+Docker的自动化部署实践。大致的流程如下：目前我有两个服务器，分别是阿里云和bandwagon,代码存储在github上，每次push都会触发阿里云上的jenkins的构建任务，jenkins将github上的代码fetch到本地，编译打包成war文件，生成docker image并上传到docker registry上，然后通过ssh来登录bandwagon服务器pull下来新生成的image并启动。由于篇幅问题，本篇文章不会介绍有关docker image的build和docker registry的搭建，但是我会在后续文章中再做详细讲解。
 学习Docker，我推荐先在网络上找说明指南，一步一步自己尝试的使用，然后如果觉得有必要可以看一下《Docker容器和容器云》这本书。
 本文内容都是docker和jenkins的基础知识，为了节约你的时间，本文的主要内容如下：
- docker 基础命令
- jenkins docker版本的搭建，构建任务的配置
- Pubish Over SSH 安装和配置
- 通过github的webhook来触发jenkins构建任务


####  Docker运行jenkins
 Docker如此火爆的一个原因是因为它形成了一个良好的生态圈，基本上主流的软件应用都有相应的Docker image。如果大家不清楚Docker image的含义，建议大家看一下[Docker中文指南](https://www.gitbook.com/book/richardhc/chinese_docker/details)，我们可以通过`docker pull`命令来下载响应的image,然后运行。比如我们希望在阿里云服务器上部署一个jenkins应用，首先可以执行下列语句来获取一个jenkins的image。
```
docker pull jenkinsci/jenkins:lts
```
 这里我们使用pull从docker registry上拉取image,但是目前业界上有很多共有或在私有的docker registry,比如说docker hub和daoCloud。所以image的全称就由三部分组成:域名或在ip + / + 软件名称 + : + 版本号，所以上边的这条命令就是让docker去jenkinsci这个Jenkins机构自己部署的registry上下载jenkins的lts版本的image.你也可以直接使用`docker pull jenkins`来下载image,但docker会默认的从docker hub上下载jenkins的laster版本。

 下载成功之后，你可以使用`docker images`命令来查看当前下载的image信息

 你可以通过`docker run`命令来运行docker容器，请注意我这里的用词，在Docker中image和container是不同的概念，你可以将他们简单的理解成Java中类和对象的关系。我们使用下面的命令来启动这个jenkins容器。
```
sudo docker run -d --name jenkins -p 9090:8080 -v /var/jenkins_home:/var/jenkins_home jenkinsci/jenkins:lts
```
 我们来依次讲解一下run命令的几个参数把：
- `-d` 后台运行docker容器并打印容器ID。如果不加`-d`参数，那么容器运行会和终端绑定，如果终端关闭，那么容器也会关闭，但是容器不会被删除。但是如果你只是想试一试某个容器，运行后自动进入命令行，那么可以使用-it参数;如果你想容器关闭之后自动删除，那么就使用-rm参数。

- `--name` 给docker container起一个别名，后续可以通过别名来管理容器，否在会系统会默认分配一个随机的别名。

- `-p` docker容器和外侧的端口映射，jenkins服务是运行在docker容器内部的，但是docker容器默认不对外暴露接口，所以通过这个参数将内部的8080端口映射到服务器本身的9090端口上。

- `-v` 数据卷的挂载。这里涉及到docker container的一个特性，container如果停止运行了，那么再次启动时，之前所有运行相关的数据和文件就都不存在了，就类似于设置了自动还原的电脑一般，无论你做了多少的操作，一旦关机重启之后就又恢复到最初的状态。数据卷就是来解决上述问题的，通过Docker container外部的文件夹的挂载，将可持久化的文件存储到外部挂载的文件夹中。

 然后你就可以根据你自己的ip地址来键入下列地址http:ip:9090来访问jenkins的主页了。
&emsp;这里有一点需要注意的是，需要注意你阿里云服务器设置的网络安全协议，是否禁用掉了9090这个端口。

#### Publish over SSH配置
&emsp;Jenkins的初始化配置和SSH Over Publish的安装请大家自行百度，这里我主要讲解一下SSH Over Pushlish配置。
&emsp;首先我们要在jenkins服务器上生成密钥对，使用`ssh-keygen -t rsa`命令来生成秘密对，这样的话，在~/.ssh/下就会有私钥id_rsa和公钥id_rsa.pub。
&emsp;然后你需要上传公钥到目标服务器上，也就是我的bandwagon服务器上，可以使用`ssh-copy-id`来将文件上传到服务器上，类似于`scp`命令的使用方式。
```
ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@<host>
```
&emsp;最后我们需要修改目标服务器的ssh配置文件，配置文件为/etc/ssh/sshd_config。设置ssh-server允许使用私钥和公钥对的方式登录，然后使用`sudo /etc/init.d/ssh restart`命令重启ssh服务。
```
RSAAuthentication yes
PubkeyAuthentication yes
#AuthorizedKeysFile     %h/.ssh/authorized_keys
```
&emsp;上述步骤成功之后，大家在系统管理中配置Publish over SSH。相关的配置信息如下图所示。

![jenkins1.png](http://7xrxif.com1.z0.glb.clouddn.com/201772-docker-jenkins1.png)

&emsp;你还可以点击下方的高级选项，来配置ssh服务器的端口，超时时间等信息，还可以点击Test Configuration来检测是否配置成功。

#### 构建任务配置
&emsp;我们先创建一个构建任务，该任务从github repo上将代码拉取下来，然后执行构建任务，然后通过Publish Over SSH在目标服务器上进行部署。
&emsp;我们首先配置源码管理模块，选择Git选项，然后配置Repository URL 并添加认证信息。可以将自己的github帐号和密码加入其中。

![jenkins2.png](http://7xrxif.com1.z0.glb.clouddn.com/201772-docker-jenkins2.png)


&emsp;不同的项目的构建命令不同，但是我们可以在构建后操作模块设置后续操作，通过ssh登录目标服务器，让目标服务器执行命令行操作来pull最新上传的image并且执行，这样就完成了部署。


![jenkins3.png](http://7xrxif.com1.z0.glb.clouddn.com/201772-docker-jenkins3.png)

#### Push触发构建任务
&emsp;完成上述配置，你就可以手动在jenkins上启动构架任务了，但是要做到自动化部署，还必须设置Push操作自动触发jenkins构建任务的机制。
&emsp;我们先到首页-用户管理界面打开自己的用户界面，然后点击左侧的设置按钮，并点击`show API token`按钮来获取API token.然后在构建任务设置页面的构建触发器模块勾选触发远程构建选项，并将token填到里边去。这是jenkins会提示你如何通过URL来触发构建任务。
![jenkins5.png](http://7xrxif.com1.z0.glb.clouddn.com/201772-docker-jenkins5.png)

&emsp;然后我们打开github上相应库的设置页面。点击左侧的Webhooks选项，然后添加hook.将上述的url填写到Payload URL栏中，点击添加。如果添加成功之后，每次你push一个新版本，那么jenkins就会自动进行部署了。
![jenkins6.png](http://7xrxif.com1.z0.glb.clouddn.com/201772-docker-jenkins6.png)

&emsp;如果你发现webhooks发送请求失败，那可能是因为你jenkins安全设置的问题，禁止掉了发送请求自动化构建。

### 后记
&emsp;本篇讲的都是十分基础性的内容，后一篇文章讲一下dockerfile的原理和注意事项与docker registry。

