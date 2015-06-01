---
layout: post
title: 简单的C++智能指针
description: 实现了一个简单的C++智能指针工具
category: blog
---

##背景
在iOS上进行C++和objc混编开发时，颇为讨厌的是C++的内存问题。一不小心就会泄漏。堆变量的使用一直困扰着你。一个行之有效的方式是使用智能指针。智能指针的原理比较简单，如下图示：
![smart_ptr](/images/blog/smart_ptr.png)

对编译原理略有了解的同学肯定知道引用计数这种最原始的垃圾回收方式。智能指针的原理就是基于引用技术(objc的指针也是基于这种方式，但objc的指针可以脱离用户的管理，直接由编译器跟踪变量的生命周期，自动计数)。

相比拷贝收集、隔代收集等技术，引用计数的效率是比较低的。因为平摊下来，变量的每次赋值，更改都被强制的增加了一次级数操作。但C++采用这种方式进行垃圾收集也是有苦衷的，根本原因在雨C/C++中的指针是弱类型，指针可以强制转化为数值类型。编译器无法判断一个变量的值是直接值还是间接值(地址, 地址的地址，地址的地址的地址)，从而也就无法判定是否要回收。

C++11中可以通过std:::shared_ptr<typename>模板使用共享智能指针。在clang++的编译器中只需要
	
	#include <memory>

就可以轻松使用智能指针.


但是..该模板类很庞大，对于体积比较在意的App，引入STL真可谓是噩梦。
为此编写了一个轻量的共享指针工具。虽然功能有限，但是基本可以满足日常需求了。

闲话少说，这就开始吧!

##实现

我将自己的这个工具取名为XP(不是Microsoft的擦屁，而是X-pointer)
显而易见，XP将是一个模板类:

	template <typename T>
	class XP
	{
	};

首先需要定义XP中所包含的实际指针的类型和引用技术。
	
	template <typename T>
	class XP
	{
	public:
	    typedef T ElementType;
	    typedef long CountType;
	    typedef std::function<void(T*)> Deletor;
    };
    
其中

* ElementType是XP包含的真实指针对应的类型。
* CountType是引用计数的数值类型。
* Deletor是当ElementType超出生命周期用于销毁其的函数对象(默认为delete操作符)


接下来定义XP的构造函数、拷贝构造函数、operator=等。
首先是最基本的构造函数：

    XP(ElementType *other, Deletor d = [](T *p) { if (p) delete p; });
    
接受ElementType*型的堆指针和一个可选的Deletor函数对象(默认为delete).为了能兼容数组和其他资源的释放，特意增加了该参数。如：
	
	XP<ArrayItemType> xp(new ArrayItemType[10], [](ArrayItemType *p){
		delete []p;
	});
就不必担心数组的泄漏了。
另外其他抽象资源也可以放到XP中管理，只需提供Deletor就可以了。如：

	XP<SomeResource> xp(SoumeResource::Create(), [](SomeResource *p){
		release(p);
	});
通过Deletor函数对象,可以使设计更为通用和抽象。
普通构造函数的实现非常简单:

	template <typename T>
	XP<T>::XP(ElementType *other, Deletor d)
	: _count(new CountType(0)), _pointer(nullptr), _deletor(d)
	{
	}
	
由于模板是精确匹配的，上诉设计后将导致XP<Type1>中只能存放Type1的指针，而不能存放Type1的派生类或其他能通过隐式类型转换到Type1的指针，这肯定无法接受，所以将提供另外一个构造函数。

	template <typename Y>
    XP(Y *other, Deletor d = [](T *p) { if (p) delete p; });
    
该构造函数本身是个函数模板其接受的指针为可以隐式类型转换到T*的的指针。同时附带了一个可选的函数对象参数，用于超出生命周期释放指针。

	template <typename T>
	template <typename Y>
	XP<T>::XP(Y *other, Deletor d)
	: _count(new CountType(1)), _pointer(other), _deletor(d)
	{
	}

定义好了原生指针到XP的构造函数之后，还需要定义拷贝构造函数和operator=。与构造函数一样，拷贝构造函数和operator＝也都有两个版本，用于匹配两种情况。

	XP(const XP<T>& other); //普通拷贝构造函数
    
    template <typename Y>
    XP(const XP<Y>& other); //支持隐式类型转换的拷贝构造函数
    
    XP<T>& operator=(const XP<T>& other); //普通的operator=
    
    template <typename Y>
    XP<T>& operator=(const XP<Y>& other); //支持隐式类型转换的operator=
    
实现为：

	template <typename T>
	XP<T>::XP(const XP<T>& other) //普通拷贝构造函数
	:_count(other._count), _pointer(other._pointer), _deletor(other._deletor)
	{
	    ++*_count;
	}
	
	template <typename T>
	template <typename Y>
	XP<T>::XP(const XP<Y>& other) //支持隐式类型转换的拷贝构造函数
	: _count(other._count), _pointer(static_cast<ElementType*>(other._pointer)),
	    _deletor(other._deletor)
	{
	}
	    
	template <typename T>
	XP<T>& XP<T>::operator=(const XP<T>& other) //普通的operator=
	{
	    this->release();
	    this->_pointer = other._pointer;
	    this->_count = other._count;
	    this->_deletor = other._deletor;
	    ++*_count;
	    return *this;
	}
	    
	template <typename T>
	template <typename Y>
	XP<T>& XP<T>::operator=(const XP<Y>& other) //支持隐式类型转换的operator=
	{
	    this->release();
	    this->_pointer = static_cast<ElementType*>(other._pointer);
	    this->_count = other._count;
	    this->_deletor = other._deletor;
	    ++*_count;
	    return *this;
	}

接着是XP的析构函数了

	template <typename T>
	void XP<T>::release()
	{
	    if (0 == --*_count) {
	        _deletor(_pointer);
	        SAFE_DELETE(_count);
	    }
	}

	template <typename T>
	XP<T>::~XP()
	{
	    this->release();
	}

显然析构函数调用release.release自减其引用技术。当计数值为0时表明已经没有指针对象指向该内存区域，这时候就可以调用_deleator函数对象释放指针。并且释放掉之前为共享计数分配的计数值地址。

其中SAFE_DELETE是一个宏：
	
	#define SAFE_DELETE(p)  \
	do {                    \
	    if (p) delete p;    \
	} while (0)


接下来是一个使用的范例:

	class Base
	{
	public:
	    virtual void test() = 0;
	    virtual ~Base()
	    {
	        printf("析构!\n");
	    }
	
	    int _i;
	};
	
	class Derive : public Base
	{
	public:
	    void test()override
	    {
	        printf("hello\n");
	    }
	};

	XP<Base> aXP(new Derive);
	XP<Base> bXP = aXP;
	XP<Base> cXP(bXP);
	
	aXP->test();
	bXP->test();
	cXP->test();
	
	XP<int> iXP(new int[10], [](int *p){ delete []p; });
	
好了，实现到此结束。

附上完整[源码]，如有问题不吝指出！

[feimengspirit]:    http://feimengspirit.com  "feimengspirit"
[源码]: https://github.com/feimengspirit/XP

<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>