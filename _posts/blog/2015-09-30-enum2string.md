---
layout: post
title: C/C++中处理enum转string的技巧
description: 一点小技巧但很实用哦！
category: blog
---

经常写C/C++的朋友可能知道，在C中，enum只是编译时的类型，如何将enum在运行时转成string很麻烦。

比如如果定义了下面的enum

	typedef enum CountryType {
	    COUNTRY_SOCIALISM = 0,
	    COUNTRY_CAPTIALISM
	} CountryType;
 
为了可以将该枚举形转到string，一般的做法都是额外定义一个array，该array里按照enum相同的顺序定义一系列字符串。如

	#define CountryTypeDesc \
	(const char*[]) { \
	    "socialism", \
	    "captialism" \
	}

这种方式明显不够灵活：

1. 一旦enum中定义的item顺序变过，还需要认为的将描述数组里的顺序也相应调整，维护工作量太大；
2. 根本原因则是相同的数据人工的定义了两次，这两个数据在语言层面脱离了联系，必须人工维护其一致性。

一个更好的方式是利用预编译的机制实现。废话不多说，上代码。

	#define COUNTRY_ITEMS  \
	    ITEM(COUNTRY_SOCIALISM), \
	    ITEM(COUNTRY_CAPTIALISM)
	
	#define ITEM(a) a
	typedef enum CountryType {
	    COUNTRY_ITEMS
	} CountryType;
	
	#undef ITEM
	#define ITEM(a) #a
	#define CountryTypeDesc \
	(const char*[]) { \
	    COUNTRY_ITEMS \
	}
 

利用预编译，一次定义了两个数据类型，enum和string数组，并且自然的保持一致。后面如果要增加、删除或者变动枚举顺序，也仅需要改动一处。好处多多吧。


<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>