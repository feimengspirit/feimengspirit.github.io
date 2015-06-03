---
layout: post
title: 一个简单的C++智能指针
description: 实现了一个简单的C++智能指针工具
category: blog
tags: C++ 智能指针
---


##背景
在iOS上进行C++和objc混编开发时，颇为讨厌的是C++的内存问题。一不小心就会泄漏。堆变量的使用一直困扰着你。一个行之有效的方式是使用智能指针。智能指针的原理比较简单，如下图示：
![smart_ptr](/images/blog/smart_ptr.png)

* 如果两个智能指针对象指向同一个动态内存，则他们共享相同的计数器地址。所以动态内存和计数器地址是一一对应
* 如果两个智能指针引用同一个计数器地址，则为了防止他们并发更改计数造成混乱，必须引用相同的锁对象来控制；

这样就得出动态内存、计数器地址、锁对象是一一对应，共生共亡的关系。

对编译原理略有了解的同学肯定知道引用计数这种最原始的垃圾回收方式。智能指针的原理就是基于引用技术(objc的指针也是基于这种方式，但objc的指针可以脱离用户的管理，直接由编译器跟踪变量的生命周期，自动计数)。

相比拷贝收集、隔代收集等技术，引用计数的效率是比较低的。因为平摊下来，变量的每次赋值，更改都被强制的增加了一次级数操作。但C++采用这种方式进行垃圾收集也是有苦衷的，根本原因在雨C/C++中的指针是弱类型，指针可以强制转化为数值类型。编译器无法判断一个变量的值是直接值还是间接值(地址, 地址的地址，地址的地址的地址)，从而也就无法判定是否要回收。

C++11中可以通过std:::shared_ptr<typename>模板使用共享智能指针。在clang++的编译器中只需要
	
	#include <memory>

就可以轻松使用智能指针.


但是..该模板类很庞大，对于体积比较在意的App，引入STL真可谓是噩梦。
为此编写了一个轻量的共享指针工具。虽然功能有限，但是基本可以满足日常需求了。

闲话少说，这就开始吧!

##实现
我将自己的这个工具取名为XP(不是Microsoft的擦屁，而是X-pointer)。显而易见，XP将是一个模板类:

	template <typename T>
	class XP
	{
	};

首先需要定义XP中所包含的实际指针的类型和引用计数。
	
	template <typename T>
	class XP
	{
	public:
	    typedef T ElementType;
	    typedef long CountType;
	    typedef std::function<void(T*)> Deletor;
	
	public:
	    CountType *_count;
    	ElementType *_pointer;
    	Deletor _deletor;
    	std::mutex *_mutex;
    };
    
其中

* ElementType是XP包含的原生动态内存的指针对应的类型。
* CountType是引用计数的数值类型。
* Deletor是当ElementType超出生命周期用于销毁其的函数对象(默认为delete操作符)
* _mutex是为了控制对_count的并发操作的锁变量。


接下来定义XP的构造函数、拷贝构造函数、operator=等。

###构造函数
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
	: _count(new CountType(1)),
	  _pointer(other),
	  _deletor(d),
	  _mutex(new std::mutex)
	{
	}

这里分别为_count和_mutex动态分配了空间，这样是为了多个XP对象可以同时引用到。

另外为了实现引用技术，还需要两个辅助工具:

	template <typename T>
	void XP<T>::release()
	{
	    {
	    std::lock_guard<std::mutex> lck(*_mutex);
	    --*_count;
	    }
	    if (0 ==*_count) {
	        _deletor(_pointer);
	        _pointer = nullptr;
	        SAFE_DELETE(_count);
	        SAFE_DELETE(_mutex);
	    }
	}
	    
	template <typename T>
	void XP<T>::increment()
	{
	    std::lock_guard<std::mutex> lck(*_mutex);
	    ++*_count;
	}

其中：

* release是递减计数，并在计数达到0时负责释放掉_counter和_mutex的动态空间，并调用_deletor释放_pointer的空间；
* increment是递增计数；
* release和increment都有锁控制，使用的锁就是_mutex。该_mutex与_pointer共生共亡，保证对同一_pointer的增减都使用同一个_mutex保护；
* 这里SAFE_DELETE是一个宏:

		#define SAFE_DELETE(p)  \
		do {                    \
		    if (p) delete p;    \
		    p = nullptr;		\
		} while (0)

显然构造函数接受的参数既可以是类型实参T指定的原生指针类型，也可以是其他能通过隐式类型转换到T* 的指针。

###拷贝构造函数
接下来是拷贝构造函数

	XP(const XP<T>& other); //普通拷贝构造函数

其实现为:

	template <typename T>
	XP<T>::XP(const XP<T>& other)
	: _count(other._count), _pointer(static_cast<ElementType*>(other._pointer)),
	  _deletor(other._deletor),
	  _mutex(other._mutex)
	{
	    increment();
	}

由于是构造自一个已有XP对象，这里除了将_pointer、_count、_deletor、_mutex只想已有对象的成员外，还有记得增加一次计数。


由于模板是精确匹配的，上面的构造函数可以保证具有相同类型实参的XP对象之间可以互相拷贝构造。但是考虑以下情况:

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

因为Derive类继承自Base类，所以Derive指针天然也是一个Base指针。但以下的用法将是错误的:

	XP<Derive> dXP(new Derive);
	XP<Base> bXP(dXP);	// ERROR!
	
因为模板的精确匹配原则，不能满足上诉用法，使得XP类模板的设计不尽人意。为此，需要添加一个拷贝构造模板函数:
	
	template <typename Y>
    XP(const XP<Y>& other); //支持隐式类型转换的拷贝构造函数
该拷贝构造函数可以从另一个类型实参的XP对象进行构造。其实现如下:

	template <typename T>
	template <typename Y>
	XP<T>::XP(const XP<Y>& other)
	: _count(other._count), _pointer(static_cast<ElementType*>(other._pointer)),
	  _deletor([&](T *p){other._deletor(static_cast<Y*>(p));}),
	  _mutex(other._mutex)
	{
	    increment();
	}

这样就保证了:
	
	如果Y*指针能通过隐式类型转换到T*指针，
	那么XP<Y>也能隐式类型转换到XP<T>;
从而保证了与原生指针类型用法上的一致性。

与普通拷贝构造函数类似，该构造函数主要完成：

* 完成_count和_pointer指针的初始化(利用原生指针类型的隐式类型转换)
* 完成_mutex初始化
* 重新构造_deletor函数对象，其实现中调用other的_deletor函数对象，函数形参的配型匹配通过static_cast转换完成。
* 自增计数器；

这样两个版本的拷贝构造函数就完成了。

###运算符
个人认为运算符是C++中的一大亮点，这种灵活性带来了C++中的复杂数据类型与原生数据类型的契合。所以没有赋值运算符是万万不能的。

与拷贝构造函数一样，我们需要提供两个版本的operator=。如下：
	
	XP<T>& operator=(const XP<T>& other); //普通的
    
    template <typename Y>
    XP<T>& operator=(const XP<Y>& other); //泛化的
    
不同的是这两个operator＝的实现更为轻松:
	
	template <typename T>
	XP<T>& XP<T>::operator=(const XP<T>& other)
	{
	    return this->operator=<T>(other);
	}
	
	template <typename T>
	template <typename Y>
	XP<T>& XP<T>::operator=(const XP<Y>& other)
	{
	    this->release();
	    this->_pointer = static_cast<ElementType*>(other._pointer);
	    this->_count = other._count;
	    this->_deletor = [&](T *p){ other._deletor(static_cast<Y*>(p)); };
	    this->_mutex = other._mutex;
	    increment();
	    return *this;
	}

让普通的operator=的实现借用泛化版本的operator=。

最后的最后，需要重载一下operator->，这样才能让XP对象具备原生指针的表达力。

	template <typename T>
	typename XP<T>::ElementType* XP<T>::operator->()
	{
	    return _pointer;
	}

###析构函数
析构函数非常简单，就是调用了一下release实现。

	template <typename T>
	XP<T>::~XP()
	{
	    this->release();
	}

实现完毕。

##使用范例
接下来我们用一个小范例展示下XP的威力吧！

	XP<Base> aXP(new Derive);
	XP<Base> bXP = aXP;
	XP<Base> cXP(bXP);
	XP<Derive> dXP = new Derive;
	XP<Base> eXP(dXP);
	XP<Base> fXP = eXP;
	
	aXP->test();
	bXP->test();
	cXP->test();
	dXP->test();
	eXP->test();
	fXP->test();
	
	XP<int> iXP(new int[10], [](int *p){ delete []p; });
	XP<Base> jXP(new Derive[20], [Base *p]{ delete []p; });
	
	
好了，实现到此结束。有机会会继续完善下去...

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