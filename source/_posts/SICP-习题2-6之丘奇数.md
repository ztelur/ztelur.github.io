---
title: SICP 习题2.6之丘奇数
tags:
  - SICP
categories:
  - SICP
abbrlink: b367660b
date: 2016-05-01 19:34:38
---
&emsp;最近一直在阅读《SICP》，然后下午做其中的习题2.6，对其题意很不理解，于是搜索了相关资料，不禁如题设所说感到如雷灌顶，特此记录下来，以供大家阅读和交流

####题目
&emsp;如果觉得将序对表示最为过程还不足以令人如雷灌顶，那么请考虑，在一个可以对程序做各类操作的语言中，我们完全可以没有数（至少在只考虑非整数的情况下），可以将0和加一操作实现为：
```lisp
(define zero (lambda (f) (lambda (x) x)))
(define (add-1 n)
	(lambda (f) (lambda (x) (f ((n f) x)))))
```
这一表现形式称为Church计数，名字来源于其发明人数理学家Church，请直接定义one和two（不使用zero和add-1）（提示：利用代换法去求值）。请给出加法过程+的一个直接定义（不要反复应用add-1）
#### 丘奇数
&emsp;说实话，我一直没有看懂题目中关于zero和add-1的定义，于是我搜索了相关资料。下边就结合资料谈一下它的概念。
&emsp;首先要明确的是丘奇数中的zero，one等并不等同于数值上的0,1,2。你可以理解为它是零这个概念的一种表现形式。换句话说，它就是零的函数式表现形式。我们先来看一下丘奇数中zero，one和two的表现形式
``` lisp
(define zero
  (lambda (f) (lambda (x) x)))
(define one
  (lambda (f) (lambda (x) (f x))))
(define two
  (lambda (f) (lambda (x) (f (f x)))))
```

&emsp;我们会发现zero，one和two都是一个函数，它接收(f)作为参数，其结果是一个接收(x)作为参数的函数。大家可能需要注意的是，(f)在这里显然也是一个函数，在LISP中函数是可以作为输入参数的。然后我们会发现zero，one和two的区别好像就是(f)函数被使用的次数。zero中(f)没有被使用，one中(f)使用了一次，two中(f)使用了两次，对就是这个次数来表示0,1,2的概念。
&emsp;我们可以实验一下检验一下
``` lisp
(define (inc n)
   (+ n 1))
> ((zero inc) 0)
0
> ((zero inc) 1)
1
> ((one inc) 1)
2
> (((add-1 one) inc) 1)
3
```
&emsp;我们定义一个过程inc就是数值意义上的加一，然后使用zero,one和add-1发现确实如同我们所想的，zero表示inc过程不会被使用，返回原数值；one表示inc被使用一次，返回加一的数值。

#### 替换法求two
&emsp;我们来使用替换法通过add-1和zero来求解one吧。
``` lisp
(add-1 zero)
;展开add-1定义
(lambda (f) (lambda (x) (f ((zero f) x))))

; 替换zero
(lambda (f) (lambda (x) (f ((lambda (x) x) x))))

; 简化因为((lambda (x) x) x)就等于x
(lambda (f) (lambda (x) (f x)))
```

#### 加法定义
&emsp;首先我们要明白丘奇数加法的含义，我们先记其加法为add-church,那么如下代码所示，最后一个过程得出的值应该是多少呢？
``` lisp
> ((one inc) 1)
2
> ((two inc) 1)
3
> (((add-church one two) inc) 1)
?
```
&emsp;根据猜测，应该是4吧。因为one表示inc对1这个数值使用一次，two标示使用两次，那么将二者加起来，那么就应该是three的含义啦，就是表示inc对1这个数值使用3次，那么就是4啦。
&emsp;add-church的实现如下
``` lisp
(define (add-church m n)
   (lambda (f) (lambda (x) ((m f) ((n f) x)))))
```
&emsp;如果你将其与add-1进行对比，你会发现add-1中的f变成了(m f),如果吧add-1中的f记住(one f)，你可能就会更加理解。那么你就会想add-m是如何标示的呢？

>参考资料
>https://pfmiles.wordpress.com/2009/11/12/%E9%80%90%E6%AD%A5%E7%90%86%E8%A7%A3%E4%B8%98%E5%A5%87%E6%95%B0%E4%B8%80/
>http://www.billthelizard.com/2010/10/sicp-26-church-numerals.html
