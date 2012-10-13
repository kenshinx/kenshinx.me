---
comments: true
date: 2012-05-24 12:24:28
layout: post
slug: django-qurycache
title: Django 缓存
wordpress_id: 288
categories:
- Python
tags:
- django
- memcache
- 缓存
---

Django提供了多个粒度的服务器端缓存实现，包括对整个站点进行缓存，对单个view进行缓存，以及对页面局部片段进行缓存。通过在settings.py中简单配置，django还可以把缓存内容放到不同的位置，包括memcache，数据库，文件，以及WEB服务器内存等。缓存的配置不详述，memcache是官方推荐的方式，相信也是多数Django应用采用的方式。接下来介绍一下Django对不同粒度缓存的实现细节。

**全站缓存**

全站缓存即默认对站点所有页面进行缓存，不分静态页面或动态页面。但一般对整个动态页面进行缓存是不合适的，可以简单通过setting.CACHE_MIDDLEWARE_ANONYMOUS_ONLY 来设置不缓存需要登录的页面。 开启全站缓存的方法很简单，只需要增加如下两个中间件即可。

    
    django.middleware.cache.UpdateCacheMiddleware
    django.middleware.cache.FetchFromCacheMiddleware


UpdateCacheMiddleware负责更新cache，当满足下面几个条件时，它就会把当前Response内容存到cache中。

	
1. 当前Response不在cache中
	
2. 请求类型为get或head
	
3. 响应状态码等于200
	
4. CACHE_MIDDLEWARE_ANONYMOUS_ONLY 设置为false，或者当前页面不需要登录
	
5. Response设置了响应头 Cache-Control，并且max-age 的值不等于 0

在判断当前response是否在cache中时，需要先计算该response对应的key值，然后根据这个key去查询memcache。这里key的生成方法比较复杂，主要由当前请求的url，language,method，以及响应头中的Vary字段计算得来。具体的生成代码如下（伪代码）：

    
    headerlist = response.get('Vary', None)
    # 一般headerlist=['HTTP_ACCEPT_LANGUAGE', 'HTTP_COOKIE','USER-AGENT']
    header_hash =md5([request.META.get(header) for header in headerlist)
    path = md5(request.path)
    key = "views.decorators.cache.cache_page"+settings.CACHE_MIDDLEWARE_KEY_PREFIX\
            +request.method+path+header_hash+request.language

这里比较麻烦的是在计算key值时，还要考虑response header中的Vary字段，例如response['Vary']?=?'user-agent'，那么缓存单位不仅仅再是url，language，还需要为不同的浏览器类型进行缓存，这个大大增加了Django cache的灵活性。

FetchFromMiddleware负责检查当前request对应的key是否在cache中，如果已在cache中，就直接从cache取出内容进行响应，key的生成方法与上面一样。

**per-view缓存**

对单个view进行缓存，这是一种更为细粒度的缓存控制。

    
    from django.views.decorators.cache import cache_page
    
    @cache_page(60 * 15)
    def my_view(request):
        ...


cache_page这个装饰器实际上是调用CacheMiddleware来实现缓存功能，它也是以请求的URL为缓存单位。例如上面view对应的urls如下

    
    urlpatterns = ('',
        (r'^foo/(\d{1,2})/$', my_view),
    )



那么Django会对foo/1/, foo/2/，分别进行缓存。上面全站缓存提到的缓存命中条件，缓存更新以及key生成算法，对于view级别缓存也都适用。view级别缓存是对全站缓存的一个补充，因为一般开启全站缓存后，为避免动态页面被错误缓存都会把setting.CACHE_MIDDLEWARE_ANONYMOUS_ONLY设为True，即不缓存所有需要登录的页面。这个时候如果想缓存某个登录后的页面，就可以通过cache_page装饰器来辅助。

**模板片段缓存**

这个是Django提供的最为细粒度的缓存，即为动态页面中，局部静态的HTML片段进行缓存。基本语法如下：

    
    {% load cache %}
    {% cache 500 sidebar request.user.username %}
        welcome {{request.user.username}} !
    {% endcache %}


上面的例子中，将用户名放到了{% cache %} 语法块内，django在计算这个缓存的key时就会带上用户名这个参数，这样就保证了每个用户的这块内容会被分别缓存。根据不同的需要，还可以加上不同的参数，例如为多语言页面加上language code等。{% cache %}标签计算key的方法要比前面两个简单很多

    
    cache_key = 'template.cache.%s.%s' % (fragment_name, md5(args))


这里的fragment_name就等于上面的sidebar，args等于request.user.username

**数据库缓存**

Django没有直接提供数据库缓存的函数，但是通过缓存api，并配合signals实现数据库缓存也很容易，下面是一个对profile进行缓存的例子：

    
    #查询数据库前先查cache，如果未查到，就把profile存到cache中
    
    def make_get_profile(func):
        def get_profile(user):
            key = 'profile@user.%s' % user.pk
            profile = cache.get(key)
            if profile is None:
                try:
                    profile = func(user)
                except Profile.DoesNotExist:
                    profile = Profile.objects.create(user=user)
                cache.set(key, profile, settings.CACHE_PROFILE_PERIOD)
            return profile
        return get_profile
    
    User.get_profile = make_get_profile(User.get_profile.im_func)


如果profile记录修改，或被删除，就需要删除缓存中的记录，可以通过signals实现：

    
    from django.db.models import signals
    
    def update_profile_cache(sender, instance, **kwargs):
        key = 'profile@user.%s' % instance.user.pk
        cache.delete(key)
    
    signals.post_save.connect(update_profile_cache, sender=Profile)
    signals.post_delete.connect(update_profile_cache, sender=Profile)



