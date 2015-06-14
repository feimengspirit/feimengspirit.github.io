---
layout: post
title: 趣谈图灵停机问题
description: 对图灵停机问题的一点想法
category: opinion
tags: turing machine
---

图灵停机问题的描述: 不存在一个程序(算法)，它能够计算任何程序在给定输入上是否会结束(停机)。
图灵对这个假设给出了一个很诡异的证明：

	bool God_algo(char * program, char * input)
	{
	    if(<program> halts on <input>)
	        return true;
	    else
	        return false;
	}
这里假设if的判断语句是人类天才思考的结晶，它能像上帝一样洞察所有程序的宿命，那么我们可以再写一个程序：

	bool Satan_algo(char * program)
	{
	    if(God_algo(program, program))
	    {
	        while(true);  // loop forever!
	        return false; // can never get here!
	    }
	    else
	        return true;
	}
和这个程序的名字一样，它太邪恶了。

当这个算法运用到自身时： Satan\_algo(Satan\_algo);
它肯定和所有的程序一样，要么停止，要么永不结束。
那我们先假设这个程序能停机，那上图代码块中的if条件判断肯定为真（因为God\_algo(Satan\_algo, Satan\_algo)这个函数返回true），从而程序进入那个包含while(true);语句的分支，那我们就可以得出这个程序不能停机。

我们再假设这个程序不能停机，类似的，我们可以得出这个程序能停机。

总之，我们有：

* Satan\_algo(Satan\_algo)能停机 =\> 它不能停机
* Satan\_algo(Satan\_algo)不能停机 =\> 它能停机

那么，我们得出了一个悖论。从而我们可以推翻我们最初的假设：不存在这样的算法！

我们不妨对这个问题做一个思考，Satan\_algo本身是构造在God\_algo的基础上的。可以这么想，如果God\_algo通过一系列判断标准来判断任何程序能否停机，那么Satan\_algo就总能够构造出一个程序，而这个程序通过God\_algo的标准是无法判断是否停机的。

想象一下，如果有个同学想要设计出一个程序，用来检测其他的程序是否存在死循环，通过不断的发现新的死循环类型，该同学不断完善自己的检测程序。终于有一天，他自信满满的声称自己的检测程序可以检测任何其他程序是否存在死循环。很可悲，这是无法实现的。只要知道他的判断标准，我们就可以构造出一个超出他的标准的死循环程序，虽然可能很复杂，但其必然性是已经被证明的了。呵呵！

最近正好看了图灵的传记电影《模拟游戏》，卷福的演出很赞。一个伟大得无法赞誉的天才，一个传奇的生命。致敬我们现在所从事工作的先驱，人工智能之父，上个世纪最伟大的数学家、计算机学家-Alan Mathison Turing
![turing][image-1]

## 扩展阅读
* [康托尔、哥德尔、图灵—永恒的金色对角线][1]



<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>

[1]:	http://mindhacks.cn/2006/10/15/cantor-godel-turing-an-eternal-golden-diagonal/

[image-1]:	/images/blog/turing.jpg