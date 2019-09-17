---
title: 'Spring Boot (一): Spring Boot starter自定义'
tags:
  - Spring Boot
categories:
  - 后端
abbrlink: 81d689c6
date: 2017-09-10 20:25:35
---

 前些日子在公司接触了`spring boot`和`spring cloud`,有感于其大大简化了spring的配置过程，十分方便使用者快速构建项目，而且拥有丰富的starter供开发者使用。但是由于其自动化配置的原因，往往导致出现问题，新手无法快速定位问题。这里我就来总结一下spring boot 自定义starter的过程,相信大家看完这篇文章之后，能够对`spring boot starter`的运行原理有了基本的认识。
 为了节约你的时间，本篇文章的主要内容有：
- spring boot starter的自定义
- spring boot auto-configuration的两种方式,spring.factories和注解
- Conditional注解的使用

### 引入pom依赖
 相信接触过spring boot的开发者都会被其丰富的starter所吸引，如果你想给项目添加redis支持，你就可以直接引用`spring-boot-starter-redis`，如果你想使项目微服务化，你可以直接使用`spring-cloud-starter-eureka`。这些都是spring boot所提供的便利开发者的组件，大家也可以自定义自己的starter并开源出去供开发者使用。
 创建自己的starter项目需要maven依赖是如下所示:
``` xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>1.4.4.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <version>1.4.4.RELEASE</version>
        </dependency>
```
#### 核心配置类StorageAutoConfigure
 构建starter的关键是编写一个装配类，这个类可以提供该starter核心bean。这里我们的starter提供一个类似`redis`的键值存储功能的bean，我们叫它为`StorageService`。负责对这个bean进行自动化装配的类叫做`StorageAutoConfigure`。保存application.properties配置信息的类叫做`StorageServiceProperties`。这三种类像是铁三角一样，你可以在很多的`spring-boot-starter`中看到他们的身影。
 我们首先来看`StorageAutoConfigure`的定义。
``` java
@Configuration
@ConditionalOnClass(StorageService.class)
@EnableConfigurationProperties(StorageServiceProperties.class)
public class StorageAutoConfigure {
    @Autowired
    private StorageServiceProperties properties;

    @Bean
    @ConditionalOnMissingBean(StorageService.class)
    @ConditionalOnProperty(prefix = "storage.service", value = "enabled", havingValue = "true")
    StorageService exampleService() {
        return new StorageService(properties);
    }
}
```
 我们首先讲一下源码中注解的作用。
- `@Configuration`,被该注解注释的类会提供一个或则多个`@bean`修饰的方法并且会被spring容器处理来生成`bean definitions`。
- `@bean`注解是必须修饰函数的，该函数可以提供一个`bean`。而且该函数的函数名必须和bean的名称一致，除了首字母不需要大写。
- `@ConditionalOnClass`注解是条件判断的注解，表示对应的类在classpath目录下存在时，才会去解析对应的配置文件。
- `@EnableConfigurationProperties`注解给出了该配置类所需要的配置信息类，也就是`StorageServiceProperties`类，这样spring容器才会去读取配置信息到`StorageServiceProperties`对象中。
- `@ConditionalOnMissingBean`注解也是条件判断的注解，表示如果不存在对应的bean条件才成立，这里就表示如果已经有`StorageService`的bean了，那么就不再进行该bean的生成。这个注解十分重要，涉及到默认配置和用户自定义配置的原理。也就是说用户可以自定义一个`StorageService`的bean,这样的话，spring容器就不需要再初始化这个默认的bean了。
- `ConditionalOnProperty`注解是条件判断的注解，表示如果配置文件中的响应配置项数值为true,才会对该bean进行初始化。

 看到这里，大家大概都明白了`StorageAutoConfigure`的作用了吧，spring容器会读取相应的配置信息到`StorageServiceProperties`中，然后依据调节判断初始化StorageService这个bean。集成了该`starter`的项目就可以直接使用`StorageService`来存储键值信息了。

#### 配置信息类StorageServiceProperties
 存储配置信息的类`StorageServiceProperties`很简单，源码如下所示:
``` java
@ConfigurationProperties("storage.service")
public class StorageServiceProperties {
    private String username;
    private String password;
    private String url;
    
    ......
    //一系列的getter和setter函数
}
```
 `@ConfigurationProperties`注解就是让spring容器知道该配置类的配置项前缀是什么，上述的源码给出的配置信息项有`storage.service.username`,`storage.service.password`和`storage.service.url`，类似于数据库的host和用户名密码。这些配置信息都会由spring容器从`application.properties`文件中读取出来设置到该类中。

#### starter提供功能的StorageService
 `StorageService`类是提供整个starter的核心功能的类，也就是提供键值存储的功能。
``` java
public class StorageService {
    private Logger logger = LoggerFactory.getLogger(StorageService.class);
    private String url;
    private String username;
    private String password;
    private HashMap<String, Object> storage = new HashMap<String, Object>();
    public StorageService(StorageServiceProperties properties) {
        super();
        this.url = properties.getUrl();
        this.username = properties.getUsername();
        this.password = properties.getPassword();
        logger.debug("init storage with url " + url + " name: " + username + " password: " + password);
    }


    public void put(String key, Object val) {
        storage.put(key, val);
    }

    public Object  get(String key) {
        return storage.get(key);
    }
}
```

#### 注解配置和spring.factories
&emsp;自定义的`starter`有两种方式来通知spring容器导入自己的auto-configuration类，也就是本文当中的`StorageAutoConfigure`类。
&emsp;一般都是在`starter`项目的`resources/META-INF`文件夹下的spring.factories文件中加入需要自动化配置类的全限定名称。
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=starter.StorageAutoConfigure
```
&emsp;`spring boot`项目中的`EnableAutoConfigurationImportSelector`会自动去每个jar的相应文件下查看spring.factories文件内容，并将其中的类加载出来在auto-configuration过程中进行配置。而`EnableAutoConfigurationImportSelector`在`@EnableAutoConfiguration`注解中被`import`。
&emsp;第一种方法只要是引入该starter，那么spring.factories中的auto-configuration类就会被装载，但是如果你希望有更加灵活的方式，那么就使用自定义注解来引入装配类。
``` java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(StorageAutoConfigure.class)
@Documented
public @interface EnableStorage {
}
```
&emsp;有了这个注解，你可以在你引入该starter的项目中使用该注解，通过`@import`注解，spring容器会自动加载`StorageAutoConfigure`并自动化进行配置。

#### 后记
&emsp;上述只是关于spring boot starter最为简单的定制和原理分析，后续我准备研究一下`spring cloud stream`的源码，主要是因为工作上一直在使用这个框架。请大家继续关注。
