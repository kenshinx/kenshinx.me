---
comments: true
date: 2012-06-13 16:20:10
layout: post
slug: mysql-querycache%e8%b0%83%e4%bc%98
title: mysql query cache调优
wordpress_id: 345
categories:
- DB
tags:
- mysql
---

最近系统整理了一下mysql的query cache，这里不讲缓存失效，内存分片等基础知识，如果想系统了解可以看文章最后的几篇参考文章，这里主要记录一下我们对query cache tuning的实践。

查看query cache相关设置的命令

mysql> show variables like '%query_cache%';


> +------------------------------+-----------+
| Variable_name | Value |
+------------------------------+-----------+
| have_query_cache | YES |
| query_cache_limit | 1048576 |
| query_cache_min_res_unit | 4096 |
| query_cache_size | 268435456 |
| query_cache_type | ON |
| query_cache_wlock_invalidate | OFF |
+------------------------------+-----------+





	
  * have_query_cache：该MySQL 是否支持Query Cache；

	
  * query_cache_limit：Query Cache 存放的单条Query 最大Result Set ，默认1M；

	
  * query_cache_min_res_unit：Query Cache 每个Result Set 存放的最小内存大小，默认4k；

	
  * query_cache_size：系统中用于Query Cache 内存的大小；

	
  * query_cache_type：系统是否打开了Query Cache 功能；

	
  * query_cache_wlock_invalidate：针对于MyISAM 存储引擎，设置当有WRITE LOCK 在某个Table 上面的时候，读请求是要等待WRITE LOCK 释放资源之后再查询还是允许直接从QueryCache 中读取结果，默认为FALSE（可以直接从Query Cache 中取得结果）。






下面的命令可以查看query cache工作状态，缓存命中率是否理想，有没有必要调大query_cache_size或缩小query_cache_min_res_unit等指标，通过下面一组数字都可以分析出来。

mysql> show status like 'Qcache%';






> 

>
>> +-------------------------+-----------+
| Variable_name?????????? | Value???? |
+-------------------------+-----------+
| Qcache_free_blocks????? | 8199????? |
| Qcache_free_memory????? | 29597504? |
| Qcache_hits???????????? | 282116098 |
| Qcache_inserts????????? | 497278707 |
| Qcache_lowmem_prunes??? | 35715784? |
| Qcache_not_cached?????? | 10293109? |
| Qcache_queries_in_cache | 20237???? |
| Qcache_total_blocks???? | 48837???? |
+-------------------------+-----------+
> 
> 





> 

> 
> 
	
>   * ?Qcache_free_blocks：Query Cache 中目前还有多少剩余的blocks。如果该值显示较大，则说明 Query Cache 中的内存碎片较多了，可能需要寻找合适的机会进行整理。 可以用那个FLUSH QUERY CACHE来清空free blocks
> 
	
>   * Qcache_free_memory：Query Cache 中目前剩余的内存大小。通过这个参数我们可以较为准确的观察出当前系统中的Query Cache 内存大小是否足够，是需要增加还是过多了；
> 
	
>   * ?Qcache_hits：多少次命中。通过这个参数我们可以查看到Query Cache 的基本效果；
> 
	
>   * ?Qcache_inserts：多少次未命中然后插入。
> 
	
>   * ?Qcache_lowmem_prunes：多少条Query 因为内存不足而被清除出Query Cache。通过 “Qcache_lowmem_prunes” 和 “Qcache_free_memory” 相互结合，能够更清楚的了解到我们系统中Query Cache 的内存大小是否真的足够，是否非常频繁的出现因为内存不足而有Query 被换出。如果Qcache_lowmem_prunes持续增长，就要尝试加到query_cache_size大小了。
> 
	
>   * ?Qcache_not_cached：?自mysql进程启动起，没有被cache的只读查询数量（包括select,show,use,desc等）
> 
	
>   * ?Qcache_queries_in_cache：当前Query Cache 中cache 的Query 数量；
> 
	
>   * ?Qcache_total_blocks：当前Query Cache 中的block 数量；
> 




通过上面的结果，可以对数据库的query cache进行一些简单诊断：

1.当Qcache_lowmem_prunes持续增长，而free memory还比较大，说明query cache有很多碎片。

2.缓存命中率= Qcache_hits/Qcache_hits+com_select，这个公式不一定对，[网上有很多计算的公式](http://bbs.chinaunix.net/thread-1700706-1-1.html)，我也搞不清楚到底哪个是对的，但是可以通过其它一些工具来查看这个指标。

3.缓存失效的数量 = Qcache_inserts-Qcache_quries_in_cache

4.未避免过多碎片，可以通过下面方法计算 缓存结果集的平均大小 = (query_cache_size-Qcache_free_memory)/Qcache_queries_in_cache，query_cache_min_res_unit可以根据这个值来设置。

5. 如果Qcache_total_blocks比Qcache_queries_in_cache多很多，说明大量的queryset使用了多个block,所以需要增加query_cache_min_res_unit的大小。



下面是一个更有指导意义的，setp by step 调优过程：





[![](http://www.kenshinx.me/wp-content/uploads/2012/06/MySQL-Query-Cach.png)](http://www.kenshinx.me/wp-content/uploads/2012/06/MySQL-Query-Cach.png)








	
  1. is hit rate acceptable?







? ? ? ? ? 不知道到底怎么计算缓存命中率的，但是通过[tuning-primer.sh](http://www.day32.com/MySQL/tuning-primer.sh)可以看到我们的缓存命中率是75%，看起来还不错







? ? ?2. ?Are most queries uncacheable?




? ? ? ? ? 通过查看Qcache_not_cached就可以判断出来，是否有大量查询无法cache，无法cache的原因有好多，可能是where中有date等可变函数，或use ,show等无法cache的语句，也可能是结果集大于query_cache_limit,我们这里的数量跟query_hit相比不算高。







? ? ?3. Is query_cache_limit large enough?




? ? ? ? ? 可以大概计算出当前数据库平均结果集的大小??= (query_cache_size-Qcache_free_memory)/Qcache_queries_in_cache，算出来的结果是2.5k ，小于query_cache_min_res_unit 4K，因此我们这里也没有问题。







? ? ?4. Are there many invalidations?




? ? ? ? ? 通过看Qcache_inserts可以大概看出来，是否有很多缓存失效。造成缓存失效的原因可能是有大量内存碎片，也可能是刚启动，也可能是大量update操作导致。




??????????我们这里的数据显示，我们确实有很多的缓存失效值得注意。







? ? ?5. Is the cache fragmented?




? ? ? ? ?当Qcache_lowmem_prunes持续增长，而free memory还比较大，说明query cache有很多碎片。通过这个分析，我们的数据库就有较多碎片，我们通过flush query cache来清除，清除之后效果很不错，Qcache_lowmem_prunes不再增长。







整体来说我们的数据库就是碎片较多，基本没别的什么问题。







参考文章：







[http://www.surfchen.org/wiki/MySQL%E4%BC%98%E5%8C%96#Query_Cache](http://www.surfchen.org/wiki/MySQL%E4%BC%98%E5%8C%96#Query_Cache)







[http://hi.baidu.com/jabber/blog/item/dc140b4f67e99531afc3abf5.html](http://hi.baidu.com/jabber/blog/item/dc140b4f67e99531afc3abf5.html)




[http://wenku.baidu.com/view/14fc9ec3d5bbfd0a7956733f.html](http://wenku.baidu.com/view/14fc9ec3d5bbfd0a7956733f.html)






