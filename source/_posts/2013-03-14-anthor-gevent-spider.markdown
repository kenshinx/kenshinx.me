---
comments: true
date: 2013-03-14 11:12:21
layout: post
slug: python gevent爬虫
title: python gevent爬虫
categories:
- Python
tags:
- gevent
- spider
---
最近新写了一个爬虫，虽然google可以搜到一大堆爬虫，但要不太过简单，无法设置链接深度，最大并发数等常用爬虫规则，要不像scrapy这样的又太重。因此花了几天时间又从新写了一个适合我们需求的爬虫。代码在这里： 
[https://github.com/kenshinx/second-spider](https://github.com/kenshinx/second-spider) 


### 主要使用的库

* 并发框架： [gevent](http://www.gevent.org/)
* http库： [requests](http://docs.python-requests.org/en/latest/)
* html解析库： [pyquery](https://pypi.python.org/pypi/pyquery)

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
3. 对Ajax类url的爬取mar




