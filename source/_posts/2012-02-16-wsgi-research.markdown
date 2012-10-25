---
comments: true
date: 2012-02-16 15:31:50
layout: post
slug: wsgi研究
title: wsgi研究
wordpress_id: 108
categories:
- Python
tags:
- paste
- pylons
- python
- wsgi
- wsgiref
---

wsgi是一个搞web开发的pythoner必须了解的内容，之前也零散的看过一些文章，但总感觉好多概念很模糊。这几天抽空又把相关内容好好整理了一下，把笔记贴出来，一些只言片语也许对某些正在研究这个的人有所帮助。  
wsgi 是一个 web 组件的接口规范.，wsgi将 web 组件分为三类： web服务器,web中间件,web应用程序，下图来自[ibm developerworks](http://www.ibm.com/developerworks/cn/web/wa-wsgi/)，很好的说明了三者之间的关系。

[![](http://photo.yupoo.com/xiha211/CmMaSs5G/medish.jpg)](http://photo.yupoo.com/xiha211/CmMaSs5G/medish.jpg)


从上图可以看出来，wsgi基本处理模式为 ： WSGI Server -> (WSGI Middleware)*-> WSGI Application 。

下面分别来看这三个组件


#### WSGI Server/gateway

wsgi server可以理解为一个符合wsgi规范的webserver，接收request请求，封装一系列环境变量，按照wsgi规范调用注册的wsgi app，最后将response返回给客户端。  
文字很难解释清楚wsgi server到底是什么东西，以及做些什么事情，最直观的方式还是看wsgiserver的实现代码。以python自带的wsgiref为例，wsgiref是按照wsgi规范实现的一个简单wsgiserver。它的代码也不复杂，下图是我读wsgiref代码后整理的。

<!-- more -->

[![](http://photo.yupoo.com/xiha211/CmMc7d89/medish.jpg)](http://photo.yupoo.com/xiha211/CmMc7d89/medish.jpg)

通过这个图可以看出来wsgi server 基本工作流程

	
1. 服务器创建socket，监听端口，等待客户端连接。
	
2. 当有请求来时，服务器解析客户端信息放到环境变量environ中，并调用绑定的handler来处理请求。
	
3. handler解析这个http请求，将请求信息例如method，path等放到environ中。

4. wsgi handler再将一些服务器端信息也放到environ中，最后服务器信息，客户端信息，本次请求信息全部都保存到了环境变量environ中。
	
5. wsgi handler 调用注册的wsgi app，并将environ和回调函数传给wsgi app

6. wsgi app 将reponse header/status/body 回传给wsgi handler

7. 最终handler还是通过socket将response信息塞回给客户端。

#### WSGI Application


wsgi application就是一个普通的callable对象，当有请求到来时，wsgi server会调用这个wsgi app。这个对象接收两个参数，通常为environ,start_response。environ就像前面介绍的，可以理解为环境变量，跟一次请求相关的所有信息都保存在了这个环境变量中，包括服务器信息，客户端信息，请求信息。start_response是一个callback函数，wsgi application通过调用start_response，将response headers/status 返回给wsgi server。此外这个wsgi app会return 一个iterator对象 ，这个iterator就是response body。这么空讲感觉很虚，对着下面这个简单的例子看就明白很多了。


下面这个例子是一个最简单的wsgi app，引自[http://www.python.org/dev/peps/pep-3333/](http://www.python.org/dev/peps/pep-3333/)

    
    def simple_app(environ, start_response):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        start_response(status, response_headers)
        return [u"This is hello wsgi app".encode('utf8')]


我们再用wsgiref 作为wsgi server ，然后调用这个wsgi app，就能直观看到一次request,response的效果，简单修改代码如下：

    
    from wsgiref.simple_server import make_server
    
    def simple_app(environ, start_response):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        start_response(status, response_headers)
        return [u"This is hello wsgi app".encode('utf8')]
    
    httpd = make_server('', 8000, simple_app)
    print "Serving on port 8000..."
    httpd.serve_forever()


访问http://127.0.0.1:8000 就能看到效果了。

此外，上面讲到了wsgi app只要是一个callable对象就可以了，因此不一定要是函数，一个实现了__call__方法的实例也可以，示例代码如下：

    
    from wsgiref.simple_server import make_server
    
    class AppClass:
    
    	def __call__(self,environ, start_response):
    	    status = '200 OK'
    	    response_headers = [('Content-type', 'text/plain')]
    	    start_response(status, response_headers)
   	    return ["hello world!"]
    
    app = AppClass()
    httpd = make_server('', 8000, app)
    print "Serving on port 8000..."
    httpd.serve_forever()


#### WSGI MiddleWare


上面的application看起来没什么意思，感觉没有太大用，但加上一层层的middleware包装之后就不一样了。一堆文字解释可能还没有一个demo更容易说明白，我写了一个简单Dispatcher Middleware，用来实现URL 路由：

    
    from wsgiref.simple_server import make_server
    
    URL_PATTERNS= (
        ('hi/','say_hi'),
        ('hello/','say_hello'),
        )
    
    class Dispatcher(object):
    
        def _match(self,path):
            path = path.split('/')[1]
            for url,app in URL_PATTERNS:
                if path in url:
                    return app
    
        def __call__(self,environ, start_response):
            path = environ.get('PATH_INFO','/')
            app = self._match(path)
            if app :
                app = globals()[app]
                return app(environ, start_response)
            else:
                start_response("404 NOT FOUND",[('Content-type', 'text/plain')])
                return ["Page dose not exists!"]
    
    def say_hi(environ, start_response):
        start_response("200 OK",[('Content-type', 'text/html')])
        return ["kenshin say hi to you!"]
    
    def say_hello(environ, start_response):
        start_response("200 OK",[('Content-type', 'text/html')])
        return ["kenshin say hello to you!"]
    
    app = Dispatcher()
    
    httpd = make_server('', 8000, app)
    print "Serving on port 8000..."
    httpd.serve_forever()


上面的例子可以看出来，middleware 包装之后，一个简单wsgi app就有了URL dispatch功能。然后我还可以在这个app外面再加上其它的middleware来包装它，例如加一个权限认证的middleware：

    
    class Auth(object):
        def __init__(self,app):
            self.app = app
    
        def __call__(self,environ, start_response):
            #TODO
            return self.app(environ, start_response)
    
    app = Dispatcher()
    auth_app = Auth(app)
    
    httpd = make_server('', 8000, auth_app)
    print "Serving on port 8000..."
    httpd.serve_forever()


经过这些middleware的包装，已经有点框架的感觉了。其实基于wsgi的框架，例如paste,pylons就是这样通过一层层middleware组合起来的。只是一个成熟的框架，这样的middleware会有很多，例如：
    
    def configure(app):
       return ErrorHandlerMiddleware(
               SessionMiddleware(
                IdentificationMiddleware(
                 AuthenticationMiddleware(
                  UrlParserMiddleware(app))))))


只要这些Middleware符合wsgi规范，甚至还可以在各个框架之间组合重用。例如pylons的认证Middleware可以直接被TurboGears拿去使用。
