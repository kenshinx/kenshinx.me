---
comments: true
date: 2012-03-28 11:48:51
layout: post
slug: wsgi-sleepy
title: MongoDB REST接口sleepy
wordpress_id: 232
categories:
- opensource
tags:
- github
- MongoDB
- rest
---

[sleepy.mongoose](https://github.com/kchodorow/sleepy.mongoose )是一个MongoDB的rest接口，代码被托管到了github上面。?试用了一下发现还是挺不错的，也很简单。但是作者只提供了一个单线程的http server，无法配合nginx，apache等web服务器在线上部署。这两天自己动手，在原来的基础上加上了wsgi的支持。可以直接通过python自带的wsgiref实验一下，执行
    
     python wsgi.py 

另外，我配合nginx+uwsgi部署着测试了一下也没问题，quick start
    
     uwsgi --http :9090 --wsgi-file  mongo_uwsgi.py 

代码可以从github checkout ?:?[https://github.com/kenshinx/sleepy.mongoose](https://github.com/kenshinx/sleepy.mongoose), 具体的使用文档可以看作者的[blog](http://www.snailinaturtleneck.com/blog/2010/02/22/sleepy-mongoose-a-mongodb-rest-interface/)?。
