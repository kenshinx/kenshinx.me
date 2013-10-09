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
---

讨论一门语言的面向对象特性，无非就是封装、继承、和多态的实现机制。
封装可以理解成创建对象，并为对象添加属性和方法的过程，javascript的封装、继承、多态都离不开两个重要概念：

* constructor 构造函数
* prototype 原型对象

接下来讨论如何通过这两个东西实现封装、继承和多态。

<!-- more -->

## 封装：

首先来看创建对象的两种常见方式：

1.使用对象字面量

```
var person = {
	name:  "ken",
	age:  29,
	sayHi:  function(){
		console.log("Hi "+this.name); 
	}, 
};

p1 = new person();
p1.sayHi(); // Hi ken
```
这个方式很简单，但是对象有很多属性和方法时，这么写就不是很好看。
补充：ECMA5新增了一个`Object.create()`，如果是通过字面量创建对象实例，推荐这种方式：

```
p2 = Object.create(person);
p2.sayHi(); // Hi ken
```

2.使用构造函数

```
function Person(name,age) {
	this.name = name;
	this.age = age;
	this.sayHi = function(){
		console.log("Hi "+this.name); 
	}
};

var person = new Person("ken",29);
person.sayHi(); // Hi ken
```
这里的`Person()`就是构造函数，构造函数定义了两个属性name,age和一个方法sayHi()。  
除了在可以在构造函数中定义属性和方法，还可以在原型对象中定义属性和方法。上面的例子可以改为：

```
function Person(name,age) {
	this.name = name;
	this.age = age;	
}

Person.prototype.sayHi = function(){
	console.log("Hi "+this.name); 
}

person = new Person("ken",29);
person.sayHi();  //Hi ken
```
实际上这种将属性定义在构造函数中，将方法定义在原型对象中是最为推荐的创建对象方式。  
到了这里constructor和prototype都出场了，接下来就需要解释一下这两个概念了。

### 构造函数constructor

到底什么才是构造函数？

1. 所谓构造函数，其实就是一个普通函数，但是内部使用了this变量?  
错，构造函数内不一定非得要this

2. `new` 后面的函数就是构造函数，它本身也是一个函数，只不过它的目的是用来创建一个新的对象。  
对，如果一个函数通过`new Person()`来调用，它就是构造函数，普通方式调用`Person()`,它就是一个普通函数

3. 构造函数默认以大写字母开头  
对，这是规范，不是语法规则

每一个对象都有一个`constructor`属性，指向它的构造函数。  
再回到上面的例子

```
function Person(name,age) {
	this.name = name;
	this.age = age;	
}

person = new Person("ken",29);

person.constructor == Person; // true
```

### javasctipt prototype
所有的函数都有prototype属性，它是一个指针，指向一个原型对象。这个对象保存了所有实例之间可以共享的数据。  

```
function Person(name,age) {
	this.name = name;
	this.age = age;	
}

p1 = new Person("ken",29);
p2 = new Person("xiao",26);

Person.prototype.sayHi() = function() {
	console.log("Hi "+this.name)
}

p1.sayHi(); // Hi ken
p2.sayHi(); // Hi xiao

```

* 任何一个prototype对象都有一个constructor属性，默认指向它的构造函数 

		Person.prototype.Constructor == Person  //true
  
* 每一个实例有一个constructor属性，默认调用prototype对象的constructor属性 

		person.constructor == Person.prototype.constructor //true
		person.constructor == Person.prototype.constructor == Person //true

下面是构造函数，实例对象，原型对象之间的一系列转换关系：

```
p.constructor == Person

Person.prototype.constructor == Person

p.__proto__ == Person.prototype

p.__proto__ == Object.getPrototypeOf(p)

p.__proto__  == p.constructor.prototype 
```

![](http://pic.yupoo.com/xiha211/D8KdzWye/ib3tu.jpg)


## 对象继承
继承是OO语言的重要特性，通常有两种方式实现继承，一种是接口继承，一种是实现继承。  
python是实现继承，Object-c是接口继承，Go是接口继承 + 组合继承， 而javascript也是实现继承。
接下来介绍javascript实现继承的几种方式：


1.最典型的通过原型链继承

```
function Person(name,age){
	this.name = name;
	this.age = age;
	this.sayHi = function(){
		return "Hi "+this.name; 
	}
};

function Chinese(){
	this.country = "China";
}

Chinese.prototype = new Person();
Chinese.prototype.constructor = Chinese;

```


```
c = new Chinese();
c.name = "ken";
c.sayHi(); // Hi ken

c instanceof Chinese; //true
c instanceof Person;  //true
c instanceof Object;  //true
```

这就比较容易理解为什么叫原型链继承了。

但这种继承方式有明显的问题

```
function Person(name,age){
	this.name = name;
	this.age = age;
	this.foods = [];
	this.sayHi = function(){
		return "Hi "+this.name; 
	}
};

function Chinese(){
	this.country = "China";	
}


Chinese.prototype = new Person();
Chinese.prototype.constructor =Chinese;

var a = new Chinese();
var b = new Chinese();

a.foods.push("dami");

console.log(b.foods);  //['dami']

```

* 上面例子可以看出来，对实例a的操作污染到了b实例，正常a的值和b应该相互隔离的。  
* 在创建Chinese的实例时，无法向Person构造函数传递参数。


2.借用构造函数的方式

```
function Person(name,age){
	this.name = name;
	this.age = age;
	this.sayHi = function(){
		return "Hi "+this.name; 
	}
};

function Chinese(){
	Person.apply(this,arguments);
	this.country = "China";
}

c = new Chinese();
console.log(c);

//Chinese {name: undefined, age: undefined, sayHi: function, country: "China"}

```
这种方式可以实际继承，但是也会有问题：

```
function Person(name,age){
	this.name = name;
	this.age = age;
	this.sayHi = function(){
		return "Hi "+this.name; 
	}
};

Person.prototype.phone = 18898;
Person.prototype.call = function(){
	return this.phone;
}

function Chinese(){
	Person.apply(this,arguments);
	this.country = "China";
}

c = new Chinese();
console.log(c.phone); //undefined
console.log(c.call); //undefined

```
Person在原型对象中定义的属性和方法，Chinese都没有继承到。这是一个严重的缺陷，因此很少会单独使用这种方式实现继承。


3.组合继承

所谓组合继承，也就是将前面两种，原型链继承和借用构造函数继承结合起来用。

```
function Person(name,age){
	this.name = name;
	this.age = age;
	this.foods = [];
	this.sayHi = function(){
		return "Hi "+this.name; 
	}
};

Person.prototype.phone = 18898;
Person.prototype.call = function(){
	return this.phone;
}

function Chinese(){
	Person.apply(this,arguments);
	this.country = "China";
}

Chinese.prototype = new Person();
Chinese.prototype.constructor = Chinese;

var c = new Chinese();
console.log(c.phone); //18898， 没有第二种方式存在的问题了


var a = new Chinese();
var b = new Chinese();

a.foods.push("dami");
console.log(b.foods);  //[]，看来也没有了第一种方式的bug了。

```

这种方式是最常见的继承模式，虽然也有缺陷，就是会调用Person()构造函数两次。

4.寄生组合式继承

相当拗口的名字，名字来自 [《Professional JavaScript for Web Developers》](http://book.douban.com/subject/7157249/)这本书,
阮一峰的[这篇文章](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html) 中称为'利用空对象作为中介' 。
YUI的YAHOO.lang.extend()使用的就是这种方式，也是被认为最好的方式。这种继承方式的细节看阮一峰的文章吧，写的很清楚。

吐槽：一个最基本的继承，javascript硬是搞得这么复杂，还得靠一堆奇技淫巧才能完整实现，实在是太弱了。

## 多态：
实现多态的机制通常有两种：覆盖和重载  
javascript没有函数签名，又是弱类型语言，无法实现重载，只能通过覆盖的方式来实现多态。下面是一个覆盖实现多态的例子：

```
function Person(){
	this.sayHi = function(){
		return "Hi man"; 
	}
};

function Chinese(){
	this.country = "China";	
}

Chinese.prototype = new Person();
Chinese.prototype.constructor =Chinese;


Chinese.prototype.sayHi = function(){
    console.log("Ni hao zhongguo");
}

function American(){
    this.country = "USA";
}

American.prototype = new Person();
American.prototype.constructor =American;


American.prototype.sayHi = function(){
    console.log("Hello USA");
}

P = new Person();
c = new Chinese();
a = new American();

p.sayHi();  //Hi man
c.sayHi();  //Ni hao zhongguo
a.sayHi();  //Hello USA

```



