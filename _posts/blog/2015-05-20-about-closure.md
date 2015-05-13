---
layout: post
title: 漫谈闭包
description: 谈谈闭包
category: blog
---

##[写作中..请关注!]



闭包是指捕获了其环境的代码块。说的更通俗一些就是一个函数引用了其被定义时候的局部变量。

闭包多在动态类型语言中常用。以js为例先留个概念。
	
	function f1() {
		var n=999;
		function f2() {
			alert(n);
		}
		
		return f2;
	}

	var result = f1();
	result();
	
这里函数f2定义时，其内部引用了对同级区域的局部变量n。f2就是一个闭包。当f2在f1中被返回时。其对n的引用一直存在。只要f2没有被回收，n就不能被销毁。

由此可见，闭包强依赖于垃圾回收。

这在传统的C中就难以实现。为了便于理解闭包，还是用c做一下示例。
	
	typedef struct {
	    int (*f)(int, int, int);
	    int *a;
	    int *b;
	} closure_t;

将闭包定义为一个函数指针和该函数对局部变量的引用，也就是指针。
 
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

这样就能顺利完成。但如果是下面这样的情形呢？
	
	closure_t getClosure() {
		int a = 3;
		int b = 1;
		closure_t closure = {f, &a, &b};
		return closure;
	}
	
	void main() {
		int l[] = {1, 2, 3, 4, 5};
		closure_t closure = getClosure();
	    map(&closure, l, 5)
	}
	
这时候就会发生访存错误。因为在getClosure返回时就已经释放掉了局部变量a和b。如果这时候还在map中使用a和b就会发生crash.	问题的关键在于c不支持垃圾回收。局部变量在返回时自动释放造成的。

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
[=]中的'='表明返回的lambda以传值的方式捕获了可见变量。这样自然是可行的。但如果想以引用的方式捕获呢？
	
	std::function<int(int)> calc()
	{
	    int a = 10;
	    return [&](int b){ return a + b; };
	}
	
[&]表明以引用方式捕获变量。但这样将会导致和上面c代码一样的后果。访问已经被释放的栈变量。crash!!
	
	std::function<int(int)> calc()
	{
	    const int a = 10;
	    return [&](int b){ return a + b; };
	}

当上诉方式却可以奏效。一个const完成的。实际上是利用了编译期的优化。

由此可见在不支持自动垃圾回收的语言中实现closure特性是比较困难的。

下表罗列了目前主流语言对closure的支持情况:
![closure](/images/blog/closure.png)

	
	


[feimengspirit]:    http://feimengspirit.com  "feimengspirit"
