---
comments: true
date: 2012-11-21 11:32:32
layout: post
slug: javascript面向对象
title: javascript面向对象
categories:
- javascript
tags:
- javascript
----

# javascript 面向对象


最近在看[《javascript语言精粹》>](http://book.douban.com/subject/3590768/),书很薄，但是内容很精炼，一些内容仅仅看书还无法很好理解，需要查很多资料来补充。这里记录一下对javascript面向对象编程思想的一些整理，主要就是想弄清楚以下几个问题：
* javascript inheritance
* javascript constructor
* javascript prototype

	<script type="text/javascript">

	var Foo = function(){
		this.a = "xx";
		console.debug(this)
	}
	Foo.prototype.b ="yy";
	foo = new Foo()
	</script>

仅仅上面几行代码，实际包含非常多的信息。

### javascript  constructor

javascript不是通过类继承，而是通过原型继承。叫继承好像不准确，貌似叫实例化更准确。js中不通过类创建实例化对象，而是通过原型对象(构造函数)创建实例化对象

* 所谓构造函数，其实就是一个普通函数，但是内部使用了this变量，例如最上面的Foo函数。
* 对构造函数使用new运算符，就能生成实例，并且this变量会绑定在这个实例对象上
* 通过new关键字创建的实例对象会自动包含一个constructor属性，constructor的值等于构造函数

		foo = new Foo()
		foo.constructor === Foo //true
		Foo.prototype.Constructor == Foo  //true
 		
* javascript中，new后面跟不是类，而是构造函数，这里Foo就是一个构造函数
* 任何一个prototype对象都有一个constructor属性，指向它的构造函数 

		Foo.prototype.Constructor == Foo  //true
  
* 每一个实例有一个constructor属性，默认调用prototype对象的constructor属性 

		foo.constructor == Foo.prototype.constructor //true
		foo.constructor == Foo.prototype.constructor == Foo //true
		

		

### javasctipt prototype

* 在javascript的继承体系中，不需要共享的属性和方法放在构造函数中,实例之间需要共享的属性和方法放在prototype对象中，

		var Foo = function(){
			this.a = "xx";
		}

		foo = new Foo();
		Foo.b = "yy";
		console.log(foo.a) // xx
		console.log(foo.b) // undifined
		Foo.prototype.c = "zz";
		console.log(foo.c) // zz
		

### javascript inheritance

在基于类的语言中，类可以从其他类继承，在javascript中是对象可以从其他对象继承。
例如下面这个例子：

	function Animal(){
		this.species = "Animal"
	}

	function Dog(name,color){
		this.name = name
		this.color = color
	}
 
 如何让Dog对象继承Animal对象？一共有五种方法(参考[阮一峰的文章](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html)):  

1. 构造函数绑定,使用call或apply方法
2. prototype模式,将子类的prototype对象设置成父类的一个实例化对象
3. 直接继承prototype，子类.prototype = 父类.prototype，这种方式下父类和子类的prototype对象会相互污染
4. 利用空对象作为中介,extend函数
5. 拷贝继承



笔记： 

* 这是很重要的一点，编程时务必要遵守,即如果替换了prototype对象，那么，下一步必然是为新的prototype对象加上constructor属性，并将这个属性指回原来的构造函数


参考文档： 


1. [http://stackoverflow.com/questions/1646698/what-is-the-new-keyword-in-javascript/](http://stackoverflow.com/questions/1646698/what-is-the-new-keyword-in-javascript/)
2. [http://www.ruanyifeng.com/blog/2011/06/designing_ideas_of_inheritance_mechanism_in_javascript.html](http://www.ruanyifeng.com/blog/2011/06/designing_ideas_of_inheritance_mechanism_in_javascript.html)
3. [http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html)


