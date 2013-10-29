---
comments: true
date: 2013-10-29 11:58:50
layout: post
slug: Lua
title: Lua模块化开发
categories:
- Lua/Nginx
tags:
- Lua,Lua module
---


## 定义模块的方式

定义module有两种方式，旧的方式，适用于Lua 5.0以及早期的5.1版本，新的方式支持新发布的Lua5.1和5.2版本。

#### 旧的方式

通过`module("...", package.seeall)`来显示声明一个包。看很多github上面早期的开源项目使用的都是这种方式，但官方不推荐再使用这种方式。

定义：


```
-- oldmodule.lua

module("oldmodule", package.seeall)

function foo()
	print("oldmodule.foo called")
end
```

使用：

```
require "oldmodule"

oldmodule.foo()
```

1. `module()` 第一个参数就是模块名，如果不设置，缺省使用文件名。
2. 第二个参数package.seeall,默认在定义了一个`module()`之后，前面定义的全局变量就都不可用了，包括print函数等，如果要让之前的全局变量可见，必须在定义module的时候加上参数package.seeall。
   具体参考云风这篇[文章](http://blog.codingnow.com/2006/02/lua_51_module.html)

之所以不再推荐`module("...", package.seeall)`这种方式，官方给出了两个原因。 

1. package.seeall这种方式破坏了模块的高内聚，原本引入oldmodule只想调用它的`foo()`函数，但是它却可以读写全局属性，例如oldmodule.os.
2. 第二个缺陷是module函数的side-effect引起的，它会污染全局环境变量。  
   `module("hello.world")`会创建一个hello的table，并将这个table注入全局环境变量中，这样使得不想引用它的模块也能调用hello模块的方法。

#### 新的方式
通过return table来实现一个模块

```
--newmodule.lua

local newmodule = {}
function newmodule.foo()
	print("newmodule.foo called")
end
return newmodule
```


使用
```
local new = require "newmodule"

new.foo()
```

因为没有了全局变量和module关键字，引用的时候必须把模块指定给一个变量。


## 引用模块的方式

Lua使用require语句来引入第三方模块，

```
require "foo"
```
require后面不需要写.lua后缀，解释器会自动加上。

require搜索包的路径是`package.path`:

```
> print(package.path)
/usr/local/share/lua/5.2/?.lua;/usr/local/share/lua/5.2/?/init.lua;/usr/local/lib/lua/5.2/?.lua;/usr/local/lib/lua/5.2/?/init.lua;./?.lua
```

上面的?代表的就是包名。

