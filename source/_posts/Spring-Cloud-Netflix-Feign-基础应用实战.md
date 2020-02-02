---
title: Spring Cloud Netflix Feign 基础应用实战
tags: spring-cloud
category: Spring
abbrlink: 6fc98a53
date: 2019-10-09 21:40:45
---

&emsp;微服务是软件系统架构上的一种设计风格，它倡导将一个原本独立的服务系统分成多个小型服务，这些小型服务都在独立的进程中运行，通过各个小型服务之间的协作来实现原本独立系统的所有业务功能。小型服务基于多种跨进程的方式进行通信协作，而在`Spring Cloud`架构中比较常见的跨进程的方式是RESTful HTTP请求和RPC调用。

&emsp;RPC就是远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。比如说，计算机 A 上的进程调用另外一台计算机 B 上的进程，其中 A 上的调用进程被挂起，而 B 上的被调用进程开始执行，当值返回给 A 时，A 进程继续执行。调用方可以通过使用参数将信息传送给被调用方，而后可以通过传回的结果得到信息。而这一过程，对于开发人员来说是透明的。

![RPC示意图](/images/19_109/image1.png)

&emsp;REST是Representational State Transfer的缩写,是表现层状态转移的含义。

&emsp;Resource是资源，所谓“资源”就是网络上的一个实体，或者说网上的一个具体信息。它可以是一段文本，一首歌曲，一种服务，总之就是一个具体的存在。你可以使用一个URI指向它，每种”资源“对应一个URI。

&emsp;Representational是”表现层“的意思，”资源“是一种消息实体，它可以有多种外在的表现形式，我们把”资源“的具体呈现出来的形式叫做它的”表现层“。比如说，文本可以用txt格式进行表现，也可以使用xml格式，JSON格式和二进制格式；视频可以以MP4格式表现，也可以以AVI格式表现。URI只代表资源的实体，不代表它的形式。它的具体表现形式，应该在HTTP请求的头信息Accept和Content-Type字段指定，这两个字段才是对”表现层“的描述。

&emsp;State Transfer是指状态转化。客户端访问服务的过程中必然涉及到数据和状态的转化。如果客户端想要操作服务器，必须通过某种手段，让服务器端发生”状态转化“。而这种转化是建立在表现层之上的，所以就是”表现层状态转化“。客户端通过使用HTTP协议中的四个动词来实现上述操作，它们分别：用来获取资源的GET，用来新建或更新资源的POST，用来更新资源的PUT，用来删除资源的DELETE。

&emsp;REST是Web Service的一种实现方式，另外一种实现方式为SOAP。REST致力于通过HTTP协议中的POST/GET/PUT/DELETE等方法和一个可读性较强的URL来提供一个HTTP请求；而SOAP致力于通过wsdl数据格式来实现通信。二者的使用场景和设计目标不同。SOAP一般作为应用层协议来进行服务间的消息调用。

&emsp;RPC和REST之间的最大差别在于RPC调用可以不依赖HTTP协议，底层直接使用TPC/IP协议进行传输，传输效率相比于REST会有一定的提升。

### Feign简介

&emsp;`Feign`是一个声明式RESTful HTTP请求客户端，它使得编写Web服务客户端更加方便和快捷。使用Feign创建一个接口并使用Feign提供的注解修饰该接口，然后就可以使用该接口进行RESTful HTTP请求的发送。`Feign`还可以集成Ribbon和Eureka来为自己提供负载均衡和断路器的机制。

&emsp;`Feign`会将带有注解的函数接口信息转化为网络请求模板，在发送网络请求之前，函数的参数值会以一定的方式设置到这些请求模板中。虽然这样的模式使得`Feign`只能支持基于文本的网络请求，但是它可以简化网络请求的实现，方便编程人员快速构建自己的网络请求架构。
![Feign架构示意图](/images/19_109/image2.png)


&emsp;如上图所示，使用`Feign`的程序的架构一般分为三个部分，分别为服务注册中心，服务提供者和服务消费者。服务提供者向服务注册中心注册自己，然后服务消费者通过`Feign`发送请求时，`Feign`会向去服务注册中心获取关于服务提供者的信息，然后再向服务提供者发送网络请求。

### 代码示例

#### 服务注册中心
&emsp;`Feign`可以配合`eureka`等服务注册中心同时使用。`eureka`来作为服务注册中心，为`Feign`提供关于服务端信息的获取，比如说IP地址。关于`eureka`的具体使用可以参考第四章中关于`eureka`的快速入门介绍。

#### 服务提供者
&emsp;`Spring Cloud Feign`是声明式RESTful请求客户端，所以它不会侵入服务提供者程序的实现。也就是说，服务提供者只需要提供Web Service的API接口，至于具体实现既可以是`Spring Controler`也可以是`Jersey`。我们只需要确保该服务提供者被注册到服务注册中心上。

``` java
@RestController
@RequestMapping("/feign-service")
public class FeignServiceController {

    private static final Logger logger = LoggerFactory.getLogger(FeignServiceController.class);

    private static String DEFAULT_SERVICE_ID = "application";
    private static String DEFAULT_HOST = "localhost";
    private static int DEFAULT_PORT = 8080;

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceId(@PathVariable("serviceId") String serviceId){
        logger.info("Get Instance by serviceId {}", serviceId);
        return new Instance(serviceId, DEFAULT_HOST, DEFAULT_PORT);
    }

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.DELETE)
    public String deleteInstanceByServiceId(@PathVariable("serviceId") String serviceId){

        logger.info("Delete Instance by serviceId {}", serviceId);
        return "Instance whose serviceId is " + serviceId + " is deleted";

    }

    @RequestMapping(value = "/instance", method = RequestMethod.POST)
    public String createInstance(@RequestBody Instance instance){

        logger.info("Create Instance whose serviceId is {}", instance.getServiceId());
        return "Instance whose serviceId is" + instance.getServiceId() + " is created";
    }

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.PUT)
    public String updateInstanceByServiceId(@RequestBody Instance instance, @PathVariable("serviceId") String serviceId){
        logger.info("Update Instance whose serviceId is {}", serviceId);
        return "Instance whose serviceId is " + serviceId + " is updated";
    }
}
```

&emsp;上述代码中通过`@RestController`和`@RequestMapping`声明了四个网络API接口，分别是对`Instance`资源的增删改查操作。

&emsp;除了实现网络API接口之外，还需要将该service注册到`eureka`上。如下列代码所示，需要在`application.yml`文件中设置服务注册中心的相关信息和代表该应用的名称。

``` yml
eureka:
  instance:
    instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
  client:
    service-url:
      default-zone: http://localhost:8761/eureka/
spring:
  application:
    name: feign-service
server:
  port: 0
```

####  服务消费者

&emsp;`Feign`是声明式RESTful客户端，所以构建`Feign`项目的关键在于构建服务消费者。通过下面六步可以创建一个`Spring Cloud Feign`的服务消费者。

&emsp;首先创建一个普通的`Spring Boot`工程，取名为`chapter-feign-client`。
&emsp;然后在pom文件中添加`eureka`和`feign`相关的依赖。其中`spring-cloud-starter-eureka`是`eureka`的starter依赖包，`spring-cloud-starter-feign`是`feign`的starter依赖包。

``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
    </dependency>
</dependencies>
```

&emsp;接着在工程的入口类上添加`@EnableFeignClients`注解表示开启`Spring Cloud Feign`的支持功能，代码如下所示。

``` java
@SpringBootApplication
@EnableFeignClients()
public class ChapterFeignClientApplication {
	public static void main(String[] args) {
		SpringApplication.run(ChapterFeignClientApplication.class, args);
	}
}
```

&emsp;`@EnableFeignClients`就像是一个开关，如果你使用了该注解，那么`Feign`相关的组件和处理机制才会生效，否则不会生效。`@EnableFeignClients`还可以对`Feign`相关组件进行自定义配置，它的方法和原理会在本章的源码分析章节在做具体的讲解。

&emsp;接下来我们定义一个`FeignServiceClient`接口，通过`@FeignClient`注解来指定服务名进而绑定服务。这一类被`@FeignClient`修饰的接口类一般被称为FeignClient。我们可以通过`@RequestMapping`来修饰相应的方法来定义调用函数。

``` java
@FeignClient("feign-service")
@RequestMapping("/feign-service")
public interface FeignServiceClient {

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceId(@PathVariable("serviceId") String serviceId);

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.DELETE)
    public String deleteInstanceByServiceId(@PathVariable("serviceId") String serviceId);

    @RequestMapping(value = "/instance", method = RequestMethod.POST)
    public String createInstance(@RequestBody Instance instance);

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.PUT)
    public String updateInstanceByServiceId(@RequestBody Instance instance, @PathVariable("serviceId") String serviceId);
}
```

&emsp;如上面代码片段所显示的，如果你调用`FeignServiceClient`对象的`getInstanceByServiceId`函数，那么`Feign`就会向`feign-service`服务的`/feign-service/instance/{serviceId}`接口发送网络请求。

&emsp;创建一个`Controller`来调用上边的服务，通过`@Autowired`来自动装载`FeignServiceClient`示例。代码如下：

``` java
@RestController
@RequestMapping("/feign-client")
public class FeignClientController {

    @Autowired
    FeignServiceClient feignServiceClient;

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceId(@PathVariable("serviceId") String serviceId){
        return feignServiceClient.getInstanceByServiceId(serviceId);
    }

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.DELETE)
    public String deleteInstanceByServiceId(@PathVariable("serviceId") String serviceId){
        return feignServiceClient.deleteInstanceByServiceId(serviceId);
    }

    @RequestMapping(value = "/instance", method = RequestMethod.POST)
    public String createInstance(@RequestBody Instance instance){
        return feignServiceClient.createInstance(instance);
    }

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.PUT)
    public String updateInstanceByServiceId(@RequestBody Instance instance, @PathVariable("serviceId") String serviceId){
        return feignServiceClient.updateInstanceByServiceId(instance, serviceId);
    }
}
```
&emsp;最后，`application.yml`中需要配置`eureka`服务注册中心的相关配置，具体配置如下所示：
``` yml
eureka:
  instance:
    instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
  client:
    service-url:
      default-zone: http://localhost:8761/eureka/

spring:
  application:
    name: feign-client
server:
  port: 8770
```

&emsp;相信读者通过搭建`Feign`的项目，已经对`Feign`的相关使用原理有了一定的了解，相信这个过程将对于理解`Feign`相关的工作原理大有裨益。

![](/images/logo.png)