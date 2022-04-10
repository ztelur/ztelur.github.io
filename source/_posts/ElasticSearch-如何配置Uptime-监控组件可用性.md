---
title: ElasticSearch 如何配置Uptime 监控组件可用性
tags: ElasticSearch
abbrlink: c1e308a7
date: 2021-05-31 22:26:20
---

本文收录至《Elastic Stack 实战手册》，欢迎和我一起解锁开发者共创书籍，系统学习 Elasticsearch，地址[点我](https://developer.aliyun.com/topic/elasticstack/playbook)

![](http://cdn.remcarpediem.net/2021-05-31-142843.png)

现在互联网架构随着用户的增加，而越来越复杂，可能要有成千上万个不同的组件和不同的实例，对这些组件可用性的监控是提供高可用服务的关键之一，Elastic 为此推出了 Uptime App。

Elasticsearch 使用 Heartbeat 进行组件的监控。

Heartbeat 也就是我们通常所说的心跳，通过 Hearteat 我们可以判断一个网络组件，当前是否存活，是否可以对外正常提供服务。

Heartbeat 是一个轻量级的数据收集器。它用来帮我们进行 Uptime 的健康监控。它可以定期通过 HTTP、TCP 或 ICMP 等方式验证组件是否处于运行状态，然后将收集到的状态和信息上报给 Elasticsearch。

而 Kibana 中的 Uptime app 则为我们提供了查看可用性数据的仪表板，以监控服务器或服务的正常运行，并提供了报警功能支持。

Elasticsearch 使用 Heartbeat 来进行 Uptime 的监控的架构可以表述如下：

![架构示意图](http://cdn.remcarpediem.net/2021-05-31-142200.png)

下面，我们将依次讲解 Uptime App 的安装，Heartbeat 的配置和各类监控组件的配置。

#### 安装 Uptime App



如果我们打开我们的 Kibana 并点击 Uptime 应用，那么第一次打开的时候，我们可以看到，如下的界面。

![](http://cdn.remcarpediem.net/2021-05-31-142309.png)

点击 Install Heartbeat，就会跳转到配置 Uptime Monitors 的文档界面，你可以按照这个界面上的步骤进行 Heartbeat 的安装，配置，启动和测试 Kibana 是否接收到 Heartbeat 上传的数据。

![](http://cdn.remcarpediem.net/2021-05-31-142323.png)



Heartbeat 在不同平台有多种安装方式，比如说 macOS、DEB、RPM 和 Windows 等，我们这里介绍最为常用的 Docker 安装方式，其后续部署和启动步骤则大同小异，读者可以自行根据需要进行实践。

需要注意的是，安装的 Heartbeat 必须和 Elasticsearch 或 Kibana 版本相同，所以我们这里选取 heartbeat:7.10.0 版本的镜像。

```
docker pull docker.elastic.co/beats/heartbeat:7.10.0
```

接着，我们可以使用如下命令启动 Heartbeat 容器。

```bash
docker run -d   --name=heartbeat   --user=heartbeat   
--volume="/tmp/heartbeat.docker.yml:/usr/share/heartbeat/heartbeat.yml:ro"   
docker.elastic.co/beats/heartbeat:7.10.0   --strict.perms=false
```

这里使用了 docker 的 --volume 参数，挂载了宿主机文件系统路径下的 heartbeat.docker.yml 文件到容器的对应路径下，这是在为 Heartbeat 提供配置文件。具体配置文件内容后续继续讲解，我们这里先演示完整个 Uptime 安装流程。

启动 Heartbeat 容器后，通过 docker ps 和 docker exec 命令可以进入到相应的容器内部。

```bash
docker ps
docker exec -it 5b3785357c26(要替换为自己ps命令输出的CONTAINER ID) bash
```

然后，通过 ls 命令，我们可以看到 Heartbeat 的整体文件结构。

```bash
bash-4.2$ ls
LICENSE.txt  NOTICE.txt  README.md  data  fields.yml  heartbeat  
heartbeat.reference.yml  heartbeat.yml  kibana  logs  monitors.d
```

在目录中，有一个叫做 heartbeat.yml 的配置文件，这个文件就是上边通过 --volume 参数挂载进来的。同时在 monitor.d 目录中，有一些不同监控器配置的配置文件案例可供大家参考，heartbeat.reference.yaml 中则是最全的配置案例。

接着，我们要使用如下命令来启动 Heartbeat，让它开始收集数据并向配置文件中指定的 Elasticsearch 中上报数据。

```bash
./heartbeat setup
./heartbeat -e
```

查看上述命令的输出日志没有什么异常后，可以再次来到 Uptime Monitors 界面，点击其 Check data 按钮检查是否接收到了数据，如果接受到了数据，则可以点击 Uptime App 按钮，前往 Uptime App 界面查看详细数据。

![](http://cdn.remcarpediem.net/2021-05-31-142340.png)

运行过一段时间的 Uptime App 界面如下图所示。

![](http://cdn.remcarpediem.net/2021-05-31-142350.png)



我们可以看到界面分为两大部分，上半部分是统计区，通过饼图和柱状图展示了当前监控器 Monitor 的状态和过去一段时间中 Monitor 的状态。而下半部分是具体的 Monitor 列表，一共有两个 Monitors，分别是监听 taobao 网和 aliyun 网站，目前两个都是 Up 状态。

#### 配置 Heartbeat

上边讲解了安装 Heartbeat 和 Uptime 的整体流程，本小节详细解决一下 Heartbeat 的配置，也就是 heartbeat.yml 文件的配置。

heartbeat.yml 文件一般有两部分组成：

- 监控器配置 heartbeat.monitors，配置要监控的目标和监控的方式；
- 输出配置 output.elasticsearch，配置数据上报的 Elasticsearch 的地址，用户名和密码。



比如说，上一小节我们启动 docker 时指定的 heartbeat.yaml 文件如下所示：

```yaml
heartbeat.monitors:
- type: http # 使用http方式监控，还可以使用 TCP 和 ICMP
  schedule: '@every 5s' # 每 5s 抓取一次
  urls: # 需要监控的 url 地址
    - https://cn.aliyun.com/
    - https://www.taobao.com/

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:http://es-cn-n6w24fib900797tgz.public.elasticsearch.aliyuncs.com:9200}'
  username: '${ELASTICSEARCH_USERNAME:111}'
  password: '${ELASTICSEARCH_PASSWORD:111}'
```

为了使 Heartbeat 知道要检查的服务，它需要一个 URL 列表。

heartbeat.yaml 中的 heartbeat.monitors 中指定了此配置。 如上的 heartbeat.yaml 配置文件，对 cn.aliyun.com 和 www.taobao.com 两个网址每隔 5s 进行一次 HTTP 检查。

除了 HTTP 监视器，Heartbeat 还可以进行 TCP 和 ICMP 类型的检查。 

```yaml
heartbeat.monitors:
- type: icmp
  schedule: '@every 5s'
  hosts:
    - http://cn.aliyun.com/
    - http://www.taobao.com/
- type: tcp
  schedule: '@every 5s'
  hosts:
    - 127.0.0.1:8080
```

此外，它还支持定义不同的检查语句，例如，使用 HTTP 监视器，可以检查响应代码(code)、正文（body）和标头（header）。 使用 TCP 监视器，能定义端口检查和字符串检查。

```yaml
heartbeat.monitors:
- type: http
  schedule: '@every 5s'
  urls:
    - https://cn.aliyun.com/
  # request details:
  check.request:
       method: GET
  check.response:
       body: "aliyun"  
```

如上的配置， Heartbeat 会每 5s 使用 GET 调用一次 https://cn.aliyun.com/ ，并在其 Response 的 Body 中寻找字符串 aliyun。如果没有找到这个字符串，则本次检查未通过。

其他更加详细的配置，你可以参考 heartbeat.reference.yml 文件。


[个人博客，欢迎来玩](http://remcarpediem.net/)


![](http://cdn.remcarpediem.net/2020-05-26-144752.png)
