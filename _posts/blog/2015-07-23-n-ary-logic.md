---
layout: post
title: 前置算符的多元逻辑表达式
description: 探讨实现一套多元逻辑表达
category: blog
---

## 前言
在任何一门编程语言中，Bool表达式都是极其重要的一部分。根本上说，逻辑表达式是计算的本质。
以c/c++语言为例，与(&&)、或(||)、非(!)是任何一个初学者都必须掌握的。而且一般语言都支持逻辑表达式的短路计算(short circuit evaluation)。例如下面表达式：

	exp1 && exp2;
	exp1 || exp2;

在第1个表达式中，如果exp1为假，则整个表达式的结果已经可以得到了(为假)。也就无需再去计算exp2的结果。同理第2个表达式中，如果exp1为真，也就无需再去计算exp2的结果了。

这种表达式非常简洁易用，但也有一定的弊端。由于这种表达式属于中缀表达式(即算符处于被计算数的中间)，优先级的问题需要通过括号来解决。考虑下面表达式

	exp1 && exp2 || exp3

算符默认是左结合的，如果要改变结合性，必须通过加括号的方式解决，比如上面的表达式，如果我们的本意是求exp1与(exp2||exp3)的结果的与结果，就必须加括号了。

	exp1 && (exp2 || exp3)

在复杂程度不高的时候，这种括号的方式尚可以接受，如果逻辑过于复杂，这种方式无异于天书，比如：

	exp1 || (exp2 && ((exp3 || exp4) && exp5)
试问程序中如果出现这种表达式，将如何维护？

## 方案
传统的逻辑表达式问题的根源在于中缀表达式的存在结合性问题。那么什么样的表达式可以规避掉算符的结合性问题呢？答案是前缀表达式。前缀表达式的所有操作数都是平等的。不存在结合性问题。

如果你接触过函数式编程语言，比如Lisp中，就是全部基于前缀表达的。

	(+ 1 2 3 4 5)
上面是一个加法表达式，被加数是被平等的参数计算。那么如果出现嵌套的表达式呢？如

	(+ 1 2 3 4 5
	  (- 5 2))
可以看到这是一种递归的表达，在计算’+’之前, ‘(- 5 2)’表达式的结果先被计算出来。自然就化解掉了结合性难题。

实现二元的前缀表达式非常简单。

	bool and(bool e1, bool e2)
	{
	   return e1 && e2;
	}
然而上述实现却用处不大，试想，如果要将 e1 && e2 && e3 实现成前缀的形式，我们不得不嵌套使用and函数

	and(e1, and(e2, e3))
这样显然难以接受。我们需要的显然是:

	and(e1, e2, e3)
这样的表达能力。我们可以用与Lisp相似的办法构造出一个复杂的逻辑表达式：

	(or false false
	  (and true true
	     (not false)))

可以看到，这种前缀算符的逻辑表达式更为清晰，即使结构异常复杂的逻辑，也可以轻松读懂。

## 实现
好了，接下来我们就着手在c++中实现出这样一套逻辑。首先,算符在c/c++等高级语言中对应的其实就是函数。因此我们将要需要一系列支持不定长参数的与或非函数。可以基于递归来实现。如

	bool logic_and(bool e1, bool e2, bool e3, ...)
	{
	   return e1 && logic_and(e2, e3, ...);
	}
大体上是这个思路，但上面代码是无法通过编译的,因此需要一点技巧。c++11支持了不定长度模板参数。可以利用这一特性。

	inline bool logic_and(bool v)
	{
	    return v;
	}
	
	template <typename... Cdr>
	inline bool logic_and(bool car, Cdr... tail)
	{
	    return car && logic_and(tail...);
	}
 
利用c++编译器对模板和普通函数的精确匹配原则，重载实现。当表达式只有一个元素时直接返回之，将第一个元素与对其余元素递归调用后的结果进行逻辑&&;

同样的手法实现or和not算符。

	inline bool logic_or(bool v)
	{
	    return v;
	}
	
	template <typename... Cdr>
	inline bool logic_or(bool car, Cdr... tail)
	{
	    return car || logic_or(tail...);
	}
	
	inline bool logic_not(bool v)
	{
	    return !v;
	}
 

哈哈，实现完毕。让我们观察一下，由于是通过递归传参，所以我们损失掉了短路计算特性。不过相对于其强大的表达能力，完全可以接受的。有了上面的基本函数，你可以轻松构造出下面的表达式：

	logic_and(e1, e2, e3
	         logic_or(e4, e5, e6,
	                  logic_and(e7, e8, e9
	                            not(e10))))
怎么样，虽然结构很复杂，但你可以一眼看出结构，不会被那么多的括号弄花眼了吧。

## 应用案例
接下来，我们可以结合一个例子看一下用法。假如你有很多样本，每个样本包含有若干参数(如param1、param2…)。现在你需要写一个简单的分类器，根据这些参数去分类。传统的写法可能是这样的。

	class classify(const Param& params)
	{
	   if (params.param1 == "param1" && params.param2 = "param2") {
	        return class1;
	   }
	   if (....) {
	        return ...;
	   }
	}

上面代码是很烂的，if的泛滥使用带来的后果是及其难以维护。而且没有使用语言本身的任何高级特性。下面我们就着手使用之前的逻辑表达式来重构一番。

	class classify(const PageParam& params)
	{
	   map<class, bool> desc = 
	   {
	       { class1, logic_and(params.param1 == "param1", params.param2 == "param2")},
	       { class2, logic_or(params.param1=="param1", params.param2=="param2", 
	                           logic_and(params.param1=="param1", params.desc="i am page1"))}
	   }
	
	   for_each(begin(desc), end(desc), [&](const map<class, bool>::value_type& item) {
	        if (item.second) {
	            return item.first;
	        }
	   });
	   return class_other;
	}

结果上面的改造后，不仅分类逻辑更加清晰，也更加便于维护了。如果需要修改，则只需要修改该map即可。

## 扩展
本文到此已经算是结束了。但我在构造多元逻辑表达式时，发现还有另外一种实现的技巧，可以贴出来分享一下。

	/*==多元与==*/
	template <typename T>
	auto LOGIC_AND(const T& functor)->decltype(functor)
	{
	    return functor;
	}
	
	template <typename Car, typename... Cdr>
	auto LOGIC_AND(Car head, Cdr... tail)
	{
	    return std::bind(std::logical_and<bool>(), head, LOGIC_AND(tail...));
	}
	
	/*==多元或==*/
	template <typename T>
	auto LOGIC_OR(const T& functor)->decltype(functor)
	{
	    return functor;
	}
	
	template <typename Car, typename... Cdr>
	auto LOGIC_OR(const Car& head, const Cdr&... tail)
	{
	    return std::bind(std::logical_or<bool>(), head, LOGIC_OR(tail...));
	}
	
	/*==取反==*/
	template <typename T>
	auto LOGIC_NOT(T functor)->decltype(std::bind(std::logical_not<bool>(), functor))
	{
	    return std::bind(std::logical_not<bool>(), functor);
	}

上述实现基于std::bind来carry谓词，得到一个最终的仿函数。可以实现惰性求值。但由于采用的是递归模板，编译下来会使得目标文件急剧膨胀。慎用，哈哈。

<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>