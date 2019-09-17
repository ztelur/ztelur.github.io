---
title: Spring AOP(一) AOP基本概念
tags: AOP
abbrlink: 69e55fe0
date: 2019-02-08 19:15:32
---
&emsp;Spring框架自诞生之日就拯救我等程序员于水火之中，它有两大法宝，一个是IoC控制反转，另一个便是AOP面向切面编程。今日我们就来破一下它的AOP法宝，以便以后也能自由使出一手AOP大法。

&emsp;AOP全名Aspect-oriented programming面向切面编程大法，它有很多兄弟，分别是经常见的面向对象编程，朴素的面向过程编程和神秘的函数式编程等。所谓AOP的具体解释，以及和OOP的区别不清楚的同学可以自行去了解。

&emsp;AOP实现的关键在于AOP框架自动创建的AOP代理，AOP代理主要分为静态代理和动态代理。本文就主要讲解AOP的基本术语，然后用一个例子让大家彻底搞懂这些名词，最后介绍一下AOP的两种代理方式：
- 以AspectJ为代表的静态代理。
- 以Spring AOP为代表的动态代理。


### 基本术语
#### (1)切面(Aspect)
&emsp;切面是一个横切关注点的模块化，一个切面能够包含同一个类型的不同增强方法，比如说事务处理和日志处理可以理解为两个切面。切面由切入点和通知组成，它既包含了横切逻辑的定义，也包括了切入点的定义。 Spring AOP就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。

```
@Component
@Aspect
public class LogAspect {
}
```
&emsp;***可以简单地认为, 使用 @Aspect 注解的类就是切面***

#### (2) 目标对象(Target)
&emsp;目标对象指将要被增强的对象，即包含主业务逻辑的类对象。或者说是被一个或者多个切面所通知的对象。


#### (3) 连接点(JoinPoint)
&emsp;程序执行过程中明确的点，如方法的调用或特定的异常被抛出。连接点由两个信息确定：

- 方法(表示程序执行点，即在哪个目标方法)
- 相对点(表示方位，即目标方法的什么位置，比如调用前，后等)

&emsp;简单来说，连接点就是被拦截到的程序执行点，因为Spring只支持方法类型的连接点，所以在Spring中连接点就是被拦截到的方法。
```
@Before("pointcut()")
public void log(JoinPoint joinPoint) { //这个JoinPoint参数就是连接点
}
```
#### (4) 切入点(PointCut)

&emsp;切入点是对连接点进行拦截的条件定义。切入点表达式如何和连接点匹配是AOP的核心，Spring缺省使用AspectJ切入点语法。 
&emsp;一般认为，所有的方法都可以认为是连接点，但是我们并不希望在所有的方法上都添加通知，而切入点的作用就是提供一组规则(使用 AspectJ pointcut expression language 来描述) 来匹配连接点，给满足规则的连接点添加通知。

```
@Pointcut("execution(* com.remcarpediem.test.aop.service..*(..))")
public void pointcut() {
}
```
&emsp;上边切入点的匹配规则是`com.remcarpediem.test.aop.service`包下的所有类的所有函数。
#### (5) 通知(Advice)
&emsp;通知是指拦截到连接点之后要执行的代码，包括了“around”、“before”和“after”等不同类型的通知。Spring AOP框架以拦截器来实现通知模型，并维护一个以连接点为中心的拦截器链。 

```
// @Before说明这是一个前置通知，log函数中是要前置执行的代码，JoinPoint是连接点，
@Before("pointcut()")
public void log(JoinPoint joinPoint) { 
}
```

#### (6) 织入(Weaving)
&emsp;织入是将切面和业务逻辑对象连接起来, 并创建通知代理的过程。织入可以在编译时，类加载时和运行时完成。在编译时进行织入就是静态代理，而在运行时进行织入则是动态代理。


#### (7) 增强器(Adviser)
&emsp;Advisor是切面的另外一种实现，能够将通知以更为复杂的方式织入到目标对象中，是将通知包装为更复杂切面的装配器。Advisor由切入点和Advice组成。
&emsp;Advisor这个概念来自于Spring对AOP的支撑，在AspectJ中是没有等价的概念的。Advisor就像是一个小的自包含的切面，这个切面只有一个通知。切面自身通过一个Bean表示，并且必须实现一个默认接口。

```
// AbstractPointcutAdvisor是默认接口
public class LogAdvisor extends AbstractPointcutAdvisor {
    private Advice advice; // Advice
    private Pointcut pointcut; // 切入点

    @PostConstruct
    public void init() {
        // AnnotationMatchingPointcut是依据修饰类和方法的注解进行拦截的切入点。
        this.pointcut = new AnnotationMatchingPointcut((Class) null, Log.class);
        // 通知
        this.advice = new LogMethodInterceptor();
    }
}

```
#### 深入理解
&emsp;看完了上面的理论部分知识, 我相信还是会有不少朋友感觉AOP 的概念还是很模糊, 对 AOP 的术语理解的还不是很透彻。现在我们就找一个具体的案例来说明一下。
&emsp;简单来讲，整个 aspect 可以描述为: ***满足 pointcut 规则的 joinpoint 会被添加相应的 advice 操作***。我们来看下边这个例子。
```
@Component
@Aspect // 切面
public class LogAspect {
    private final static Logger LOGGER = LoggerFactory.getLogger(LogAspect.class.getName());
     // 切入点，表达式是指com.remcarpediem.test.aop.service
     // 包下的所有类的所有方法
    @Pointcut("execution(* com.remcarpediem.test.aop.service..*(..))")
    public void aspect() {}
    // 通知，在符合aspect切入点的方法前插入如下代码，并且将连接点作为参数传递
    @Before("aspect()")
    public void log(JoinPoint joinPoint) { //连接点作为参数传入
        if (LOGGER.isInfoEnabled()) {
            // 获得类名，方法名，参数和参数名称。
            Signature signature = joinPoint.getSignature();
            String className = joinPoint.getTarget().getClass().getName();
            String methodName = joinPoint.getSignature().getName();
            Object[] arguments = joinPoint.getArgs();
            MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();

            String[] argumentNames = methodSignature.getParameterNames();

            StringBuilder sb = new StringBuilder(className + "." + methodName + "(");

            for (int i = 0; i< arguments.length; i++) {
                Object argument = arguments[i];
                sb.append(argumentNames[i] + "->");
                sb.append(argument != null ? argument.toString() : "null ");
            }
            sb.append(")");

            LOGGER.info(sb.toString());
        }
    }
}
```
&emsp;上边这段代码是一个简单的日志相关的切面，依次定义了切入点和通知，而连接点作为log的参数传入进来，进行一定的操作，比如说获取连接点函数的名称，参数等。

### 静态代理模式
&emsp;所谓静态代理就是AOP框架会在编译阶段生成AOP代理类，因此也称为编译时增强。ApsectJ是静态代理的实现之一，也是最为流行的。静态代理由于在编译时就生成了代理类，效率相比动态代理要高一些。AspectJ可以单独使用，也可以和Spring结合使用。

### 动态代理模式
&emsp;与静态代理不同，动态代理就是说AOP框架不会去修改编译时生成的字节码，而是在运行时在内存中生成一个AOP代理对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

&emsp;Spring AOP中的动态代理主要有两种方式：JDK动态代理和CGLIB动态代理。

&emsp;JDK代理通过反射来处理被代理的类，并且要求被代理类必须实现一个接口。核心类是 InvocationHandler接口 和 Proxy类。
&emsp;而当目标类没有实现接口时，Spring AOP框架会使用CGLIB来动态代理目标类。
&emsp;CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类。CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。核心类是 MethodInterceptor 接口和Enhancer 类

![两种动态代理的区别](https://upload-images.jianshu.io/upload_images/623378-c854d59f8fb95c4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 后记
&emsp;AOP的基础知识都比较枯燥，本人也不擅长概念性的文章，不过下一篇文章就是AOP源码分析了，希望大家可以继续关注。

![](https://upload-images.jianshu.io/upload_images/623378-7d960275042f309d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
