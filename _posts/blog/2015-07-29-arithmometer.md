---
layout: post
title: 前置算符的四则运算(续逻辑算符)
description: 实现一套新颖而又古老的四则运算表达式形式
category: blog
---

## 背景
日常的编程当中，简单的四则运算可能占据了不少的分量。最简单的形式莫过于

	1 + 2
这样的了。上一篇博文中已经提到，这是高级语言中普遍采用的中缀表达式形式。如果要连环加，我们不得不这样:

	1 + 2 + 3 + ... + n
如果再掺杂其他运算符，就麻烦了。记得小学数学老师一直强调的四则运算法则:
> **先乘除后加减，如果有小括号，先计算小括号里面的**.

真够繁琐的。比如下面这个式子:

	1 + 2 + (3 - 5) * 6 / 7
哈哈，有点糊涂了吧，现在需要一点一点的找清楚优先级次序再去计算，非常痛苦。还记得我在上篇博文中提到的吗?**前缀表达式中不存在优先级问题**.

好进行到这里，我们就利用前缀表达式对上面进行改写。写法将参考Lisp形式。

	(/ 
	 (+ 1 2
	    (* 
	      (- 3 5)
	      6))
	 7)
	
看清楚了吗，如果对编译过程稍有了解的可能知道，高级语言中的顺序表达式在语法分析阶段会被解析成语法树，而我们的改写恰恰就和这个语法树的形式相对应。

<img src="/images/blog/arithmometer.png" width="40%">
唯一不同的是，在我们的表达式里当然无法引入这么多连线，使用嵌套小括号实现。聪明的你可能已经发现这是一种递归形式(树的定义就是递归的，所谓的树就是链表的链表)。

看到这里，你可能会疑问，高级语言费了那么多周折提供的表达形式，为什么我们弃之不用，反而使用难以理解的树表达形式。原因就是**后者的表达能力远远强于前者**。

高级语言在构造四则运算的语法时，参考了人类的自然语言，带来的结果就是存在歧义，需要人为的添加括号来确定优先级。这种中缀表达形式非常贴合人的理解，但当表达式趋于复杂时，就非常容易出错。而包括Lisp等函数式编程语言中采用的前缀形式，只要掌握求值规则，表达式再怎么复杂，也可以一眼看破。当然一开始可能会觉得有些别扭，只要熟悉下来，会喜欢上这种形式。借用一句妄语:

> 人理解迭代，神理解递归

## 实现
下面我们就着手在c++中实现这个形式。我们确定文法形式为:

	PLUS(1, 2, 3, 4, 5, ...)
这样。因为

	(+ 1, 2, 3, 4, 5, ...)
在C的语法下是难以实现的。我们转换一下形式。

首先我们需要实现四则运算的基本算法。以加法为例

	template <typename T>
	struct plus
	{
	    auto operator()(const T& x, const T& y) const { return x + y; }
	};
是一个重载了operator()的类模板函数对象。其他几个也依样画葫芦。

	template <typename T>
	struct minus
	{
	    auto operator()(const T& x, const T& y) const { return x - y; }
	};
	
	template <typename T>
	struct multiplies
	{
	    auto operator()(const T& x, const T& y) const { return x * y; }
	};
	
	template <typename T>
	struct divides
	{
	    auto operator()(const T& x, const T& y) const { return x  / y; }
	};
基本算法实现完以后，我们需要实现一个不定参数的求知器。

	template <template <typename> class O, typename R, typename Car>
	inline void iEval(R& result, const Car& head)
	{
	    result = O<R>()(result, head);
	    return;
	}
	
	template <template <typename> class O, typename R, typename Car, typename... Cdr>
	void iEval(R& result, const Car& head, const Cdr&... tail)
	{
	    result = O<R>()(result, head);
	    return iEval<O, R>(result, tail...);
	}
重载了一对模板函数，递归的方式和上篇博文中的用法如出一辙，只是这里因为考虑了左结合的原则，递归的结果通过result参数完成。这里为了灵活性，预留了一个模板的模板参数。该参数对应之前定义的四个仿函数类。传入不同的仿函数类，会完成不同类型的计算。

为了方便调用，额外提供一个包装函数。

	template<template <typename> class O, typename Car, typename... Cdr>
	auto eval(const Car& head, const Cdr&... tail)
	{
	    auto result = head;
	    iEval<O, Car>(result, tail...);
	    return result;
	}
该模板函数将结果result做一次转发。方便调用方的使用。掉用方只需接收返回值即可，而不需要传出参数的方式。

为了进一步简化使用，我还提供了四个宏。

	#define PLUS(...)       eval<plus>(__VA_ARGS__)
	#define MINUS(...)      eval<minus>(__VA_ARGS__)
	#define MULTIPLIES(...) eval<multiplies>(__VA_ARGS__)
	#define DIVIDES(...)    eval<divides>(__VA_ARGS__)
实现完成啦。接下来我们看一下效果吧。
> 细心的你可能会发现，我为了放便调用，省略了整个表达式返回类型的指定。默认表达式中出现的第一个算数的类型即为最终的结果类型。

## 菜刀小试
我们看一下这些工具嵌套使用的效果吧

	LOG_INFO("+: %f", PLUS(1.1, 2, 3, 4, 5, 6));
	LOG_INFO("-: %f", MINUS(1.1, 2, 3, 4, 5, 6));
	LOG_INFO("*: %f", MULTIPLIES(1.1, 2, 3, 4, 5, 6));
	LOG_INFO("/: %f", DIVIDES(PLUS(1.0, 2, 3, 5, 6),
	                          MINUS(10, 9, 8, 7)));
	 
和原生四则运算表达式不同的是，你可以一眼看出这些表达式的结构。有点意思吧。

Duang一下，如你所料，上面那些表达式的结果为：

	+: 21.100000
	-: -18.900000
	*: 792.000000
	/: -1.214286



<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>