---
title: Spring Cloud Stream(一)：Spring基础知识
date: 2017-10-10 21:51:45
tags: 
	- Spring Boot
categories:
	- 后端
    - Spring
---

&emsp;我研究和阅读`Spring Cloud Stream`源码已经有一个多月了，但是由于自己的Spring基础知识不是很充足，所以导致很多地方都没有融会贯通，并且相关的文章一直无从下手。于是我先整理了当时阅读代码时的知识点记录，算是源码分析之前的基础知识储备吧，整理的有些杂乱，希望大家理解。
&emsp;本文涉及的Spring知识如下：
- Spring Boot的`@Import`用法和原理,与`Configuration`和`ImportBeanDefinitionRegistrar`相关
- Bean初始化各个周期的回调，比如`InitializingBean`,`BeanPostProcessor`,`SmartInitializingSingleton`
- `FactoryBean`和`MethodInterceptor`
- `Aware`系列回调
- `Lifecycle`和`SmartLifecycle`和`DefaultLifecycleProcessor`


#### BeanDefinitionRegistryPostProcessor
 `BeanDefinitionRegistryPostProcessor`实现了`BeanFactoryPostProcessor`接口，是Spring框架的`BeanDefinitionRegistry`的后处理器，用来注册额外的`BeanDefinition`。`postProcessBeanDefinitionRegistry`方法会在所有的`BeanDefinition`已经被加载了，但是所有的`Bean`还没有被创建前调用。`BeanDefinitionRegistryPostProcessor`经常被用来注册`BeanFactoryPostProcessor`的`BeanDefinition`。

#### ImportBeanDefinitionRegistrar
 `@Import`注解用来支持在`Configuration`类中引入其他的配置类，包括`Configuration`类，`ImportSelector`和`ImportBeanDefinitionRegistrar`的实现类。`ImportBeanDefinitionRegistrar`在`ConfigurationClassPostProcessor`处理`Configuration`类期间被调用，用来生成该`Configuration`类所需要的`BeanDefinition`。而`ConfigurationClassPostProcessor`正实现了`BeanDefinitionRegistryPostProcessor`接口。下面我们就来看一下其`processConfigBeanDefinitions`方法到底是如何处理`Configuration`类的。

``` java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();
        //第一步：先把所有Configuration的beanDefinition找到。
		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			//利用AnnotationMetadata是否有@Configuration这个注解。需要注意的是
            //Configuration是一个元注解，它是可以使用在其他注解上的，被这些注解注释的类也被认为是Configuration
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}
		//第二步：通过Order注解的值来排序，定义了Configuration的先后顺序
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
		});
        //..... 此处有省略
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
            //第三步：通过BeanDefinition来读取ConfigurationClass
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
            //第四步：重点，通过ConfigurationClass来获得BeanDefinition
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
            //第五步：由于在loadBeanDefinitions过程中会向registry中添加BeanDefinition,所以这里需要把新的Definition
            //在重新检测一遍，先看是否是Configuration类，如果是的那么还要再进行一次处理。
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
                //.....此处有省略，大致逻辑就是通过registry多出的BeanDefinition获得新的candidateNames
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());
        //.....此处有省略
	}
```
 接着我们直接到`ConfigurationClassBeanDefinitionReader`类中查看`loadBeanDefinition`函数的实现。它会调用`loadBeanDefinitionsForConfigurationClass`函数。在该函数中会处理所有和`Configuration`相关的`BeanDefinition`,其中就会调用`loadBeanDefinitionsFromRegistrars`来通过`ImportBeanDefinitionRegistrar`加载`BeanDefinition`。
 看到这里，大家可能会有个疑问，多个`Configuration`和多个`ImportBeanDefinitionRegistrar`存在的情况下，它们之间的对应关系是如何确定的呢？
 `ConfigurationClassParser`的parse方法会将Configuration类相关的配置信息全部解析出来。我们可以看其`doProcessConfigurationClass`方法的源码。通过`@Import`注解将`Configuration`类和相应的`ImportBeanDefinitionRegistrar`联系在一起。
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {
        //首先处理内部成员类的情况
		processMemberClasses(configClass, sourceClass);

		// 处理 @PropertySource 注解
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
            }
		}

		// 处理 @ComponentScan 注解
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(
							holder.getBeanDefinition(), this.metadataReaderFactory)) {
						parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// 处理 @Import 注解
		processImports(configClass, sourceClass, getImports(sourceClass), true);

		// 处理 @ImportResource 注解
		if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
			AnnotationAttributes importResource =
					AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// 处理configuration中的 @Bean 函数
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}
        //......有省略
		return null;
	}
```


#### InitializingBean，FactoryBean，MethodInterceptor
 Spring Cloud Stream的`BindableProxyFactory`类实现了上述接口。
``` java
BindableProxyFactory implements MethodInterceptor, FactoryBean<Object>, Bindable, InitializingBean
```
 其中，`InitializingBean`接口有一个`afterPropertiesSet`方法，该方法在`bean`所有的属性都被赋值后调用。bean的属性被初始化是在初始化的时候做的，与`BeanPostProcessor`结合来看，`afterPropertiesSet`方法在`postProcessBeforeInitialization`和`postProcessAfterInitialization`之间被调用。
 Spring中有两个类型的Bean,普通Bean和工厂Bean。FactoryBean有三个接口，分别是:
- Object getObject():返回FactoryBean创建的对象实例。
- boolean isSingleton():表示FactoryBean返回的对象实例是否为单例。
- Class getObjectType():返回FactoryBean返回的对象类型。
 我们可以看一下`BindableProxyFactory`的相关实现，这里会和`MethodInterceptor`配合。`MethodInterceptor`是AOP相关的接口，用于在调用对象接口时进行切片注入或在直接实现接口。
``` java

@Override
	public synchronized Object getObject() throws Exception {
        //使用AOP的ProxyFactory类，由于该类本身也是先了MethodInterceptor接口
        //所以这样配合使用，直接返回ProxyFactory类。
		if (this.proxy == null) {
			ProxyFactory factory = new ProxyFactory(this.type, this);
			this.proxy = factory.getProxy();
		}
		return this.proxy;
	}

	@Override
	public Class<?> getObjectType() {
		return this.type;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
```

#### BeanPostProcessor，ApplicationContextAware，BeanFactoryAware，SmartInitializingSingleton

 Spring Cloud Stream的`StreamListenerAnnotationBeanPostProcessor`实现了如下接口
``` java
public class StreamListenerAnnotationBeanPostProcessor
		implements BeanPostProcessor, ApplicationContextAware, BeanFactoryAware, SmartInitializingSingleton,
		InitializingBean
```
 `BeanPostProcessor`是`bean`的后处理器，通过它我们可以在`Bean`初始化前后进行处理。它的`postProcessBeforeInitialization`方法在`Bean`初始化之前被调用，而`postProcessAfterInitialization`在`Bean`初始化后被调用。相关原理涉及到Spring创建Bean的流程，这个之后有时间再研究吧。
#### Aware系列接口
 Spring中提供了一些`Aware`相关的接口，像是`BeanFactoryAware`,`ApplicationContextAware`等。当一个类实现了这些接口之后，`Aware`接口的Bean在初始化之后，可以取得相应的资源的实例。比如`StreamListenerAnnotationBeanPostProcessor`对象就实现了`ApplicationContextAware`和`BeanFactoryAware`接口来获取`ConfigurableApplicationContext`与`BeanFactory`实例。
#### SmartInitializingSingleton
 当所有的singleton的bean都初始化完成之后才会调用这个接口
的`afterSingletonsInstantiated`函数
#### SmartLifecycle
 之前介绍的接口都是在Bean的生命周期内的某个阶段中被调用，如果我们希望在容器本身的生命周期事件上做一些事情该怎麽办呢？Spring容器提供了`Lifecycle`接口。当`ApplicationContext`接口启动或在关闭时，它会调用本容器内所有的Lifecycle接口。
``` java
public interface Lifecycle {
    //启动该组件
	void start();
    //停止组件
	void stop();
    //查看组件是否正在运行
	boolean isRunning();

}
```
 如果两个对象有依赖关系，希望某一个bean先初始化完成，完成一些工作之后，再初始化另一个bean。在这个场景下，可以使用`SmartLifecycle`接口，该接口的`getPhase`方法返回一个整型数字，表明执行顺序。如果其`getPhase()`方法返回`Integer.MIN_VALUE`,那么该对象最先启动，最后停止；如果返回`Integer.MAX_VALUE`,那么该对象最后启动，最先停止。在`Spring`容器里，有`DefaultLifecycleProcessor`这个类来处理所有的`Lifecycle`的bean。在`AbstractApplicationContext`的`finishRefresh`函数中会调用到该processer的`onRefresh函数`，从其调用其本身的`startBeans`函数。
``` java
private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
        //遍历所有的Lifecycle,按照phase分成不同的LifecycleGroup
		for (Map.Entry<String, ? extends Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(entry.getKey(), bean);
			}
		}
		if (!phases.isEmpty()) {
            //按照phase排序，然后启动
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```