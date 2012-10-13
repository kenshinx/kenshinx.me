---
comments: true
date: 2012-02-09 12:13:43
layout: post
slug: diy-python-web-framework
title: DIY Python Web Framework
wordpress_id: 70
categories:
- Python
tags:
- django
- Framework
- jinja2
- python
- SQLAlchemy
- wsgi
---

突发奇想，现在的python web framework组件这么多，是不是通过一些胶合代码就可以组装一个自己的web框架？ok, 如果让我自己diy一个web框架的话，我应该会这么组合。

**Template** ?：** jinja2**

其它的template引擎还有很多，像mako，kid，genghi。选择jinja2是因为对django比较熟悉，这个语法跟django比较类似，而且看过一些jinja2,mako,kid,django的benchmark报告，除了django很不给力之外，其它几个差不了太多。

**ORM：SQLAlchemy**

其它的ORM还有SQLObject，这两个都没用过，很多框架像pylons,turbogears都是用的这两个ORM组件。

**URL Dispatcher: ?selector**

selector的风格也有点类django,可以根据URL选择一个wsgi application运行，其它Dispatcher还有Routes 有点类Rails。

**WSGI: webob**

webob可以理解成是实现了wsgi相关应用的模块，核心包括对request，response,http header,cookie等的处理，并且这些实现完全遵守wsgi规范 。??[Pyramid](https://www.pylonsproject.org/)?,?[Pylons](https://www.pylonsproject.org/projects/pylons-framework/about),?[TurboGears](http://turbogears.org/)?等都是基于此。类似的有werkzeug，flask和uliweb就使用了werkzeug。

上面几个核心组件放在一起，就可以搭出一个简单的web framework。具体这几个东西怎么搭，大概如下：

[![](http://www.kenshinx.me/wp-content/uploads/2012/02/framework.png)](http://www.kenshinx.me/wp-content/uploads/2012/02/framework.png)

整个框架还是以wsgi为基础，通过wsgi将各个组件粘合起来，有点类似pylons的感觉。

除了上面的核心组件，还可以根据自己的需要加上**Form ?, ?****I18N ?**，?**Log**? ，?**Session?**?， ?**Mail , ?****Authentication ,?**?**Admin，Ajax支持，Rss Fed**?这些小菜，这样一个功能全面的Web framework就出来了，哈哈。

说明:这篇文章只是自己对一些相关东西的整理，可能还有很多理解不对的地方，欢迎读过的人在下面留言。真正要开发一个功能全面的web framework 肯定不会这么简单，而且现在的python web framework 这么多了，也没必要去做这么个东西。真做的话，估计也不会有人用。:)
