---
layout: post
title: Swift语法特性初探(一)
description: 整理Swift中的一些语法特性
category: blog
---

##[写作中...]

##一、简介
从去年开始接触iOS开发，由于工作中涉及的大部分仍是C++部分，混编而言还是objc更为和谐。所以虽然一直摩拳擦掌，还是没有怎么深入学习swift。近期有了一些时间进行学习，记录下学习的过程，权当做笔记吧。

##二、特性
swift作为一门更为现代的语言，非常灵活。语法也很优美。而且已经非常脚本话了。事实上完全可以将swift作为一门脚本来运行，只需在文件开头注释

	#!/usr/bin/env xcrun swift
	import Foundation
	… // do the things
第一回是为了告知shell使用swift解释器运行即可。这方面有机会我后面会单独写博文。
总之，现在发明的语言已经逐渐吸收Lisp的特性。新语言的语法特性逃脱了古老语言Lisp划定的圈圈。有兴趣的同学可以参考下《计算机程序的构造和解释》这本书。精妙非凡!
闲话少说，进入正题。


###多返回值
swift中你可以让函数返回一个tuple.tuple中包含若干个返回值。如下面例子:

	func calculateStatistics(scores: [Int]) -> (min: Int, max: Int, sum: Int) {
	    var min = scores[0]
	    var max = scores[0]
	    var sum = 0
	    
	    for score in scores {
	        if score > max {
	            max = score
	        } else if score < min {
	            min = score
	        }
	        
	        sum += score
	    }
	    
	    return (min, max, sum)
	}
	
	let statistics = calculateStatistics([5, 3, 100, 3, 9])
	
返回值
	
	(.0 3, .1 100, .2 120)
	
该特性的一个用处是可以返回值检查中。

###不定参数

	func sumOf(numbers: Int...) -> Int {
    var sum = 0
    for number in numbers {
        sum += number
    }
    return sum
	}

	sumOf() //返回0
	sumOf(42, 579, 12)	//返回633
	
###函数型返回值
在swift中，函数是一等公民。虽然这句话在各类说明中重复出现。但很多人并不了解其意义。一等公民意味着函数可以像其他基本类型一样作为参数和作为返回值。有人会反驳说：c/c++中函数也可以作为参数和返回值。请记住，c/c++中作为参数和返回值的不是函数本身，而是函数地址，即指针。指针作为基本类型，当然是一等公民的,但c/c++中的函数并不是。

	func makeIncrement() -> (Int -> Int) {
	    func addOne(number: Int) -> Int {
	        return 1 + number
	    }
    
    	return addOne;
	}

	var increment = makeIncrement()
	increment(7) //返回8
	
	
	
	func hasAnyMatches(list: [Int], condition: Int -> Bool) -> Bool {
	    for item in list {
	        if condition(item) {
	            return true
	        }
	    }
    
    	return false
	}
	
	func lessThenTen(number: Int) ->Bool {
    	return number < 10
	}
	
	var numbers = [20, 19, 7, 12]
	hasAnyMatches(numbers, lessThenTen)
	
	let mappedNumbers = numbers.map({
	    (number: Int) -> Int in
	    return 3 * number
	})
	
	mappedNumbers