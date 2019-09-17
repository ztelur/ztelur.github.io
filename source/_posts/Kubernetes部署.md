---
title: Kubernetes部署
tags:
  - docker
  - kubernetes
categories:
  - 容器
abbrlink: 51083ebf
date: 2017-07-23 15:33:52
---

&emsp;学习完Docker之后，发现了kubernetes这个容器云框架，于是就自己部署来玩玩。大家也可以按照这个[和我一步步部署 kubernetes 集群](和我一步步部署 kubernetes 集群)文章来部署。最近在这里花费了大量的时间，之后希望整理一下相关的原理介绍。


![kuber1.png](http://7xrxif.com1.z0.glb.clouddn.com/2017723-kube-kuber1.png)


![kube3.png](http://7xrxif.com1.z0.glb.clouddn.com/2017723-kube-kube3.png)



### 问题列表和解决方案
- google源找不到解决方案：
http://www.jianshu.com/p/4f5066dad9b4
公钥未安装导致无法安装
- Created API client, waiting for the control plane to become ready
卡死在这里，阿里云需要使用内网ip地址 你也可以使用journalctl -u kubelet 查看日志
ApiServer的debug https://github.com/kubernetes/kubernetes/wiki/Debugging-FAQ
- SSL/TLS协议
http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html
- etcd cant bind the addr
https://github.com/coreos/etcd/issues/4789
nc -l 10.5.0.9 2380
iptables查看端口问题
- flanneld
https://stackoverflow.com/questions/34439659/flannel-and-docker-dont-start
flanneld 启动/kubernetes 没有找到
//fail to retrieve network config: invalid charactar
- linux低版本不支持flanneld的vxlan功能，需要换成udp 
cant register network : oeperation not supported
https://github.com/coreos/etcd/issues/3710

- linux低版本不支持docker
http://dockone.io/question/1060
pod-infra-container-image
- dashboard
https://github.com/kubernetes/kubernetes/issues/39722
- 127.0.3.1:9090 cant connection
http://blog.csdn.net/xinghun_4/article/details/50492041
add route
https://github.com/kubernetes/dashboard/issues/672
