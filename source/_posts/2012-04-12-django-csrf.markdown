---
comments: true
date: 2012-04-12 15:38:45
layout: post
slug: django-csrf%e9%98%b2%e6%8a%a4%e5%8e%9f%e7%90%86
title: django csrf防护原理
wordpress_id: 262
categories:
- secure
tags:
- csrf
- django
- web安全
---

通过Django的CSRF中间件，开发人员很轻松就能够为自己的系统提供CSRF防护的能力。关于CSRF的介绍可以看几篇不错的文章，[owasp](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))?，[80sec](http://www.80sec.com/csrf-securit.html)?, [lake](http://blog.csdn.net/lake2/article/details/2245754)?。关于django如何使用CSRF中间件可以看[django手册](https://docs.djangoproject.com/en/dev/ref/contrib/csrf/)?，写的也很清楚，这篇文章会介绍一下django实现CSRF防护的细节。

CSRF的防护通常有两种方式，一个是通过Challenge-Response的方式，例如通过Captcha和重新输入密码等方式来验证请求是否伪造，但这会影响用户体验，类似银行付款会采用这样的方式。另一种是通过随机Token的方式，多数Web系统都会采用这种方式，Django也是用的这种。下面具体描述Django的实现步骤，大多数基于Token的CSRF防护系统都是类似的流程。






	
  1. 如果有form在提交时需要验证token，那么django在打开这个页面时就会在用户的cookie中插入csrftoken记录，csrftoken的生成方式：



    
    hashlib.md5("%s%s"% (randrange(0, 18446744073709551616L),\
                                    settings.SECRET_KEY)).hexdigest()


原本我以为Django的Token值是通过sessionid 加一个salt值计算得来的，但是看了django的源码发现并非如此，而是通过一个随机数加salt的方式。好像django 1.1之前也是用的sessionid进行hash，不清楚为什么在django 1.2之后改用随机数的方式。另外这个csrftoken写入cookie的有效期会非常的长

    
    response.set_cookie(settings.CSRF_COOKIE_NAME,\
        request.META["CSRF_COOKIE"], max_age = 60 * 60 * 24 * 7 * 52,\
        domain=settings.CSRF_COOKIE_DOMAIN)




?通过代码看来，这个cookie的有效期有52周，并不会随着session的销毁而注销或变化。











? ? 2. ?form提交时，会通过隐藏表单的方式提交cookie中的csrftoken记录



    
    <input?type="hidden"? value="4dc3a02f858e4bd54b57" name="csrfoken">







? 3. ?服务器端接到POST请求时，会验证提交的Token和用户cookie中的Token值是否一致，如不一致就返回403错误。








有一点需要注意的，如果使用的是django.middleware.csrf.CsrfViewMiddleware这个中间件，django会默认验证每一个post请求，因此所有的<form>表单内都需要加上csrf_token的tag，否则站内提交也会被阻止，除非通过@csrf_exempt装饰器来显示声明不验证token值。







