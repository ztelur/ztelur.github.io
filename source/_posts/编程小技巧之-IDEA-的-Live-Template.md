---
title: 编程小技巧之 IDEA 的 Live Template
tags: 工具
abbrlink: f35ff8b9
date: 2019-06-23 23:26:14
---

合格的程序员都善于使用工具，正所谓君子性非异也，善假于物也。

使用自动化工具可以减少自己的工作量，提高工作效率。日常编程过程中，我们经常需要编写重复的代码片段，比如说

```
private static final Logger LOGGER = LoggerFactory.getLogger(HashServiceImpl.class);
```
每次编写时都要键入很多键，有什么方法可以快速生成这段代码呢？类似的，如何保存格式固定的常用代码片段，然后在需要时快速生成呢。IDEA 的 Live Template 是一个可行的途径。

我也是最近才逐渐使用 IDEA 的 Live Template 功能，之前虽然知道这个功能，但是没有养成使用的习惯。最近一段时间在不断审视并反思自己的编程、工作和生活习惯，才发现其中有很多可以优化精进的地方。

这也是《程序员修炼之道》中所说的 Think ! About Your Work 。


IDEA 是一个很强大的编程工具，学会使用它能够极大的提高工作效率，将精力投入到更关键的事情上，而不是将时间浪费在编写重复代码上面。

而作为 Java 程序员，令人苦恼的地方是 Java 开发过程中经常需要编写有固定格式的代码，例如说声明一个私有变量，Logger 或者 Bean 等等。对于这种小范围的代码生成，我们可以利用 IDEA 提供的 Live Templates 功能。

Live Template 并不是简单的 Code Snippet，它甚至支持 Groovy函数配置，可以编写一些复杂的逻辑，支持很复杂的代码生成。

### 基本使用
IDEA 自带很多常用的动态模板，都是大家平常编码时的常用语句格式。比如说下面四张动图中的语句。

四张图分别是 声明静态 String 类型成员变量，判断字符串为空，for 循环和打印函数参数。

![psfs](https://upload-images.jianshu.io/upload_images/623378-93937a8adfb634a3.gif?imageMogr2/auto-orient/strip)

![ifn](https://upload-images.jianshu.io/upload_images/623378-72b01c6425b7214f.gif?imageMogr2/auto-orient/strip)

![fori](https://upload-images.jianshu.io/upload_images/623378-ecfa28e13dc3834e.gif?imageMogr2/auto-orient/strip)

![soutp](https://upload-images.jianshu.io/upload_images/623378-6b35fdb8a3cabff1.gif?imageMogr2/auto-orient/strip)

### 自定义 Template

打开配置页面，进入 Live Template 选项卡，我们可以看到 IDEA 预先设置的模板配置。这些模板都是最常用的一些语句，我们先来看一下它们都是如何定义的。

![](https://upload-images.jianshu.io/upload_images/623378-1e1a20edb9359b3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

缩写就是 IDEA 识别的模板的别名，就像文章开头展示的当你键入 `soutm` 时，IDEA 就会自动识别为该模板。

而应用上下文则表示该模板在什么上下文中生效。比如说上文中时一个 `System.out` 的语句，它只应该在 Java 的函数体中有效，所以它的应用上下文设置为 `Java: statement`，在其他类型文件或者 Java 文件的成员变量声明位置都无法使用该模板。

![](https://upload-images.jianshu.io/upload_images/623378-779f92162c27402b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


模板内容就是你按下 Tab 键之后，IDEA 自动生成的内容，它一般包含两个部分，纯文本和参数。参数可以进行值绑定，还支持光标的自动跳转。如同上文所示，`$CLASS_NAME$` 和 `$METHOD_NAME$` 就是参数，而`$END$`是一个特殊的参数，它表示光标最后一个跳转的位置。

而参数设置就是设置这些参数的值，可以使用 IDEA 提供的一些内置函数，还可以使用强大的 Groovy 脚本。去 IDEA 的官网可以查看这些函数的具体作用。

![variables](https://upload-images.jianshu.io/upload_images/623378-0f95752ee3371bd9.gif?imageMogr2/auto-orient/strip)

我们这里讲解一下 `groovyScript("groovy code", arg1)` 的使用。它能提供一切你想要的能力，它支持执行 Groovy 脚本处理输入，然后输出处理后的字符串

```
groovyScript("code", ...)

|  code   |   一段Groovy代码或者Groovy脚本代码绝对路径    |
|  ...    |   可选入参，这些参数会绑定到`_1, _2, _3, ..._n`, 在 Groovy 代码中使用。|
```

比如之前打印函数参数的模板是这样定义的。

![](https://upload-images.jianshu.io/upload_images/623378-8bc425c2ed384c2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
groovyScript("'\"' + _1.collect { it + ' = [\" + ' + it + ' + \"]'}.join(', ') + '\"'", methodParameters())
```

`methodParameters` 是 IDEA 内置的函数，它返回的结果作为参数输入到 Groovy 的脚本中，生成打印参数函数的字符串。


### 后记

感谢大家的阅读，希望大家继续关注，也可以留言分享你最喜欢使用的编程工具和编程小技巧。


![](https://upload-images.jianshu.io/upload_images/623378-289642734e6eb81b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



