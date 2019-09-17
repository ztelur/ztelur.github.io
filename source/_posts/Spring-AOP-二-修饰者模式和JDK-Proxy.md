---
title: Spring AOP(二) 修饰者模式和JDK Proxy
tags: AOP
abbrlink: 90b14a95
date: 2019-02-17 13:20:37
---

&emsp;在上边一篇[文章](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483735&idx=1&sn=2b526949cc4e69c75d1a96268920da68&chksm=fc04c537cb734c21e84076865259243b4165fa00911920a30fe09d3a75b0cfc002fe66e43d05&token=2009122217&lang=zh_CN#rd)中我们介绍了Spring AOP的基本概念，今天我们就来学习一下与AOP实现相关的修饰者模式和Java Proxy相关的原理，为之后源码分析打下基础。

### 修饰者模式
&emsp;Java设计模式中的修饰者模式能动态地给目标对象增加额外的职责(Responsibility)。它使用组合(object composition)，即将目标对象作为修饰者对象(代理)的成员变量，由修饰者对象决定调用目标对象的时机和调用前后所要增强的行为。

&emsp;装饰模式包含如下组成部分：
- Component: 抽象构件，也就是目标对象所实现的接口，有operation函数
- ConcreteComponent: 具体构件，也就是目标对象的类
- Decorator: 抽象装饰类，也实现了抽象构件接口，也就是目标类和装饰类都实现了相同的接口
- ConcreteDecorator: 具体装饰类，其中addBeavior函数就是增强的行为，装饰类可以自己决定addBeavior函数和目标对象函数operation函数的调用时机。

![修饰者模式的类图](https://upload-images.jianshu.io/upload_images/623378-8e64b6435042b3b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;修饰者模式调用的时序图如下图所示。程序首先创建目标对象，然后创建修饰者对象，并将目标对象传入作为其成员变量。当程序调用修饰者对象的operation函数时，修饰者对象会先调用目标对象的operation函数，然后再调用自己的addBehavior函数。这就是类似于AOP的后置增强器，在目标对象的行为之后添加新的行为。

![修饰者模式的时序图](https://upload-images.jianshu.io/upload_images/623378-e250659fe563087c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;Spring AOP的实现原理和修饰者模式类似。在上一篇文章中说到AOP的动态代理有两种实现方式，分别是JDK Proxy和cglib。

&emsp;如下图所示，JDK Proxy的类结构和上文中修饰者的类图结构类似，都是代理对象和目标对象都实现相同的接口，代理对象持有目标对象和切面对象，并且决定目标函数和切面增强函数的调用时机。
&emsp;而cglib的实现略有不同，它没有实现实现相同接口，而是代理对象继承目标对象类。
![两种动态代理的对标](https://upload-images.jianshu.io/upload_images/623378-e7fb928a23ea31a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;本文后续就讲解一下JDK Proxy的相关源码分析。

### JDK Proxy
&emsp;JDK提供了Proxy类来实现动态代理的，可通过它的newProxyInstance函数来获得代理对象。JDK还提供了InvocationHandler类，代理对象的函数被调用时，会调用它的invoke函数，程序员可以在其中实现所需的逻辑。

&emsp;JDK Proxy的基本语法如下所示。先构造一个`InvocationHandler `的实现类，然后调用`Proxy`的`newProxyInstance `函数生成代理对象，传入类加载器，目标对象的接口和自定义的`InvocationHandler`实例。

```
public class CustomInvocationHandler implements InvocationHandler {
    private Object target;

    public CustomInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before invocation");
        Object retVal = method.invoke(target, args);
        System.out.println("After invocation");
        return retVal;
    }
}

CustomInvocationHandler customInvocationHandler = new CustomInvocationHandler(
        helloWord);
//通过Proxy.newProxyInstance生成代理对象
ProxyTest proxy = (ProxyTest) Proxy.newProxyInstance(
        ProxyTest.class.getClassLoader(),
       proxyObj.getClass().getInterfaces(), customInvocationHandler);
```


### 生成代理对象
&emsp;我们首先来看一下`Proxy`的`newProxyInstance`函数。`newProxyInstance`函数的逻辑大致如下：
- 首先根据传入的目标对象接口动态生成代理类
- 然后获取代理类的构造函数实例
- 最后将`InvocationHandler`作为参数通过反射调用构造函数实例，生成代理类对象。
&emsp;具体源码如下所示。
```
public static Object newProxyInstance(ClassLoader loader,
                                        Class<?>[] interfaces,
                                        InvocationHandler h)
    throws IllegalArgumentException
{

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    // 1 动态生成代理对象的类
    Class<?> cl = getProxyClass0(loader, intfs);

    // ... 代码省略，下边代码其实是在try catch中的
    if (sm != null) {
        checkNewProxyPermission(Reflection.getCallerClass(), cl);
    }
    // 2 获取代理类的构造函数
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    final InvocationHandler ih = h;
    if (!Modifier.isPublic(cl.getModifiers())) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                cons.setAccessible(true);
                return null;
            }
        });
    }
    // 3调用构造函数，传入InvocationHandler对象
    return cons.newInstance(new Object[]{h});
}
```
&emsp;`getProxyClass0`函数的源码如下所示，通过代理类缓存获取代理类信息，如果不存在则会生成代理类。

```
// 生成代理类
private static Class<?> getProxyClass0(ClassLoader loader,
                                        Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // 如果已经有Proxy类的缓存则直接返回，否则要进行创建
    return proxyClassCache.get(loader, interfaces);
}
```
### 生成代理类
&emsp;JDK Proxy通过`ProxyClassFactory`生成代理类。其`apply`函数大致逻辑如下：

- 校验接口是否符合规范
- 生成代理类的名称和包名
- 生成代理类字节码
- 根据字节码生成代理类Class

```
// 生成代理类的工厂类
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // 所有代理类名的前缀
    private static final String proxyClassNamePrefix = "$Proxy";

    // 生成唯一类名的原子Long对象
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            // 通过loader找到接口对应的类信息。
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            // 判断找出来的类确实是一个接口
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            // 判断接口是否重复
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }
        // 代理类的包路径
        String proxyPkg = null;
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        // 记录非公开的代理接口，以便于生成的代理类和原来的类在同一个路径下。 
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }
        // 如果没有非公开的Proxy接口，使用com.sun.proxy报名
        if (proxyPkg == null) {
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        long num = nextUniqueNumber.getAndIncrement();
        // 默认情况下，代理类的完全限定名为：com.sun.proxy.$Proxy0，$Proxy1……依次递增  
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        // 生成代理类字节码
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            // 根据字节码返回相应的Class实例
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```
&emsp;其中关于字节码生成的部分逻辑我们就暂时不深入介绍了，感兴趣的同学可以自行研究。

### $Proxy反编译
&emsp;我们来看一下生成的代理类的反编译代码。代理类实现了`Object`的基础函数，比如`toString`、`hasCode`和`equals `，也实现了目标接口中定义的函数，比如说`ProxyTest`接口的`test `函数。

&emsp; `$Proxy`中函数的实现都是直接调用了`InvocationHandler `的`invoke`函数。

```
public final class $Proxy0 extends Proxy
  implements ProxyTest 
// 会实现目标接口，但是由于集成了Proxy，所以无法再集成其他类
{
  private static Method m1;
  private static Method m0;
  private static Method m3;
  private static Method m2;
  // 构造函数要传入一个InvocationHandler对象
  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }
  // equal函数
  public final boolean equals(Object paramObject)
    throws 
  {
      try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final int hashCode()
    throws 
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }
  // test函数，也就是ProxyTest接口中定义的函数
  public final void test(String paramString)
    throws 
  {
    try
    {
      // 调用InvocationHandler的invoke函数
      this.h.invoke(this, m3, new Object[] { paramString });
      return;
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final String toString()
    throws 
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }
  // 获取各个函数的Method对象
  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m3 = Class.forName("com.proxy.test2.HelloTest").getMethod("say", new Class[] { Class.forName("java.lang.String") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
    }
    throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
  }
}
```

### 后记
&emsp;下一篇文章就是AOP的源码分析了，希望大家继续关注。

![](https://upload-images.jianshu.io/upload_images/623378-7d960275042f309d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

