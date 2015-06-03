---
layout: post
title: 漫谈闭包
description: 谈谈闭包和Lambda
category: blog
tags: 闭包 closure 函数式编程
---


各种书籍和网络文章对闭包的说明何其之多。但大多是流于理论。使得对闭包的概念仍然是不够清晰。

说的玄乎点，**闭包**是指捕获了其环境的代码块。

说的更狭义通俗一些就是一个函数引用了其被创建时候可见的局部变量。

闭包多在动态类型语言中使用。以下面的js代码为例先对其有个整体认识。
	
	function f1() {
		var n=999;
		function f2() {
			alert(n);
		}
		
		return f2;
	}

	var result = f1();
	result();
	
这里函数f2定义时，其内部引用了对同级区域的局部变量n。这样f2就是一个闭包。当f2在f1中被返回时。其对n的引用一直存在。只要f2没有被回收，n就不能被销毁。

由此可见，闭包强依赖于垃圾回收。

当然这在传统的C中是难以实现的。但为了便于理解闭包，还是用C做一下示例。
	
	typedef struct {
	    int (*f)(int, int, int);
	    int *a;
	    int *b;
	} closure_t;

将闭包定义为一个函数指针和该函数对局部变量的引用，也就是指针。很形式...
 
	void map(closure_t *closure, int x[], size_t n) {
	    for (int i = 0; i < n; ++i)
	        x[i] = (*closure->f)(*closure->a, *closure->b, x[i]);
	}
	 
	int f(int a, int b, int x) {
	    return a * x + b;
	}
	 
	void main() {
	    int l[] = {1, 2, 3, 4, 5};
	    int a = 3;
	    int b = 1;
	    closure_t closure = {f, &a, &b};
	    map(&closure, l, 5);
	}
上诉利用map和f函数完成对数组l的每个元素做一次线性计算。(a * item + b) 其中a和b就是该闭包创建时捕获的局部变量。也就是main中的a和b。

但由于a和b是栈变量，这样就要时刻注意变量生命周期问题。由于a和b的生命周期包含了closure的生命周期才使得上诉的map顺利运行。

但如果是下面这样的情形呢？
	
	 void getClosure(closure_t *closure) {
		int a = 3;
		int b = 1;
		closure->f = f;
		closure->a = &a;
		closure->b = &b;
		return;
	}
	
	void main() {
		int l[] = {1, 2, 3, 4, 5};
		closure_t closure;
		getClosure(&closure);
	    map(&closure, l, 5)
	}
	
这时候就会发生访存错误。因为在getClosure返回时就已经释放掉了局部变量a和b。如果这时候还在map中使用a和b就会发生crash.	问题的关键在于c不支持垃圾回收，局部变量在父函数返回时自动释放造成的。

而C++对这一特性的支持如何呢？事实上在C++11中已经引入了Lambda。可以一定程度的支持closure特性了。但由于在垃圾回收上的先天不足。closure还是有限制的。

考虑下面的代码。

	std::function<int(int)> calc()
	{
	    int a = 10;
	    return [＝](int b){ return a + b; };
	}

	int main()
	{
		auto f = calc();
		printf("%d", f(5));
	
	}
[=]中的'='表明返回的lambda以传值的方式捕获了可见变量。表明创建lambda时捕获的是a的一个私有拷贝。这样自然是可行的。但如果想以引用的方式捕获呢？
	
	std::function<int(int)> calc()
	{
	    int a = 10;
	    return [&](int b){ return a + b; };
	}
	
[&]表明以引用方式捕获变量。但这样将会导致和上面c代码一样的后果。访问已经被释放的栈变量。crash!!

但下诉方式却可以奏效。一个const完成的。实际上是利用了编译期常量的优化。
	
	std::function<int(int)> calc()
	{
	    const int a = 10;
	    return [&](int b){ return a + b; };
	}

或者利用嵌套lambda也可以实现:

	int c = []{
        int a = 10;
        return [&](int b){ return a + b; };
    }()(5);

此时c得到正常的15;
为了详细探讨成功的原因，我们将上面代码做拆分：
	 
    auto f = [] { //lambda2
        int a = 10;
        return [&](int b) { return a + b; }; //lambda1
    };
    
    auto f2 = f();
    int c = f2(5);
   
C++11中对lambda的实现实际上是一个重载了operator()的类.这样上诉代码在模板展开阶段会变成下述形式。
	
	class lambda1
	{
	public:
		lambda1(int &a) : _a(a){}
		int operator()(int b)
		{
			return _a + b;
		}
		
	private:
		int& _a
	};

	class lambda2
	{
	public:
		lambda2():_a(10){}
		lambda1 operator()()
		{
			return lambda1(_a);
		}
	
	private:
		int _a;
	};

这样对闭包的调用就变成了:
	
	lambda2 l2;
	lambda1 l1 = l2();
	int c = l1(5);
	
这样其实lambda2的生命周期涵盖了lambda1的生命周期，所以不会发生错误。

实际上在C++11中，closure对局部变量的捕获有下述几种方式,请留意区分!

	[]	Capture nothing (or, a scorched earth strategy?)
	[&]	Capture any referenced variable by reference
	[=]	Capture any referenced variable by making a copy
	[=, &foo]	Capture any referenced variable by making a copy, but capture variable foo by reference
	[bar]	Capture bar by making a copy; don't copy anything else
	[this]	Capture the this pointer of the enclosing class
	
那么lambda的类型是什么？
lambada的类型可以用通用的std::function<>模板表达，其中类型参数为lambda的对应的函数指针类型。如lambda1的类型就是std::function<int(int)>;

另外对于未捕获任何局部变量的lambda，实际上完全等同于函数指针。通常编译器都会做出优化，
这时候可以直接用函数指针表达。

由此可见在不支持自动垃圾回收的语言中实现closure特性是比较麻烦的。其中陷阱用了才有体会。
但借助于lambda,C++的STL已经更加强大。后面如果有机会再做探讨。
欢迎交流！！！

下表罗列了目前主流语言对closure的支持情况:
![closure](/images/blog/closure.png)


[feimengspirit]:    http://feimengspirit.com  "feimengspirit"

<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>

