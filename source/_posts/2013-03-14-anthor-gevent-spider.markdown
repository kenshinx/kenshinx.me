---
comments: true
date: 2013-03-14 11:12:21
layout: post
slug: python gevent爬虫
title: 又一个python gevent爬虫
categories:
- python
tags:
- gevent
- spider
---
最近新写了一个爬虫，虽然google可以搜到一大堆爬虫，但大都太过简单，无法设置链接深度，最大并发数等常用爬虫规则。偶尔有一些能支持这些规则，但是感觉代码写的又不怎么样，与其维护这样的还不如自己写一个。因此花了几天时间又从新造了一个轮子。代码在这里： 
[https://github.com/kenshinx/second-spider](https://github.com/kenshinx/second-spider) 


### 主要使用的库

* 并发框架使用的[gevent](http://www.gevent.org/)
* http库使用[requests](http://docs.python-requests.org/en/latest/)
* html解析用的[pyquery](https://pypi.python.org/pypi/pyquery)

### 支持的爬虫规则

* 最大并发请求数
* 最大爬行URL数
* 最大链接深度
* http请求的headers,cookies
* 是否只爬相同域名下的url
* 是否只爬相同主机名下的url

### 还需要做的事

1. 继续添加一些规则，例如解析robots文件，可设置一些不爬的url等
2. 对相似url进行识别，排重
3. 对Ajax类的url的爬取




