---
comments: true
date: 2012-09-07 10:53:26
layout: post
slug: python并发编程
title: python并发编程
wordpress_id: 379
categories:
- Python
tags:
- concurrency
- python
---

总结了一下python的并发编程模型，并通过一个网络I/O操作的不同版本实现，来帮助理解各种并发模型间的差异。例子很简单，就是下载多个网页。
下面是一个单线程的版本，主要为了说明实验的内容，以及为后面并行版本提供对比。

    
    #!/usr/bin/env python
    import time
    import urllib2
    import HTMLParser
    from BeautifulSoup import BeautifulSoup
    
    hosts = ["http://www.baidu.com", "http://www.amazon.com","http://www.ibm.com",
             "http://www.python.org","http://www.microsoft.com"]
    
    def read(host):
        try:
            context = urllib2.urlopen(host,timeout=5)
        except urllib2.URLError:
            print "load %s failure." %host
            return
        try:
            title = BeautifulSoup(context).title.string
        except HTMLParser.HTMLParseError:
            print "paser %s tile failure" %host
            return
        print "%s  : %s" %(host,title)
    
    def singleRead():
        start = time.time()
        for i in range(30):
            for host in hosts:
                read(host)
        end = time.time()
        print "Elapsed Time : %d" %(end-start)
    
    if __name__ == '__main__':
        singleRead()


接下来通过不同的线程库和协程库实现了不同的并发下载版本，包括：多线程、多进程、stackless、greenlet、asyncore、gevent。


实验代码：[https://github.com/kenshinx/concurrency](https://github.com/kenshinx/concurrency)



代码都比较简单，不过在实现stackless版本时犯了一个错误，要利用协程的优点，在并发的地方必须是非阻塞式，而我直接用urllib2来下载网页，urllib2是阻塞式IO，因此没有实现想要的并发效果，最后下载完成的时间跟单线程一样。下面是反面代码：

    
    #!/usr/bin/env python
    
    import time
    import stackless
    import urllib2
    import HTMLParser
    from BeautifulSoup import BeautifulSoup
    
    """
    This is a negative concurrence example.
    run need stackless python.
    """
    
    hosts = ["http://www.baidu.com", "http://www.amazon.com","http://www.ibm.com",
             "http://www.python.org","http://www.microsoft.com"]
    
    def read(host):
        try:
            context = urllib2.urlopen(host,timeout=5)
        except urllib2.URLError:
            print "load %s failure." %host
            return
        try:
            title = BeautifulSoup(context).title.string
        except HTMLParser.HTMLParseError:
            print "paser %s tile failure" %host
            return
        print "%s  : %s" %(host,title)
    
    class Reader(object):
        def __init__(self,channel):
            self.channel = channel
            stackless.tasklet(self.run)()
    
        def run(self):
            host = self.channel.receive()
            read(host)
    
    def blockRead():
        start = time.time()
        channel = stackless.channel()
        [Reader(channel) for i in range(len(hosts))]
        [channel.send(host)  for host in hosts]
        stackless.run()
        end = time.time()
    
        print "Elapsed Time : %d" %(end-start)
    
    if __name__ == '__main__':
        blockRead()



在[python-cn邮件列表](http://groups.google.com/group/python-cn/browse_thread/thread/37b14963b9af02c7/af2fbffe58e7ea31?lnk=gst&q=stackless#af2fbffe58e7ea31)发现有人也提出了同样的疑问，最后终于搞明白只有非阻塞才能真正利用协程的优点。因为多个微线程（即协程）并发，微线程本身都是跑在一个线程里，当某个微线程阻塞住无法switch到其它微线程时，整个线程也就block住了。于是基于greenlet的协程模型，并将网络IO换成异步非阻塞式，我重新实现了一个版本，性能就非常不错。

    
    #!/usr/bin/env python
    
    import sys
    import time
    import socket
    import urlparse
    import StringIO
    import HTMLParser
    from BeautifulSoup import BeautifulSoup
    from greenlet import greenlet
    from greenlet import getcurrent
    
    is_windows = sys.platform == 'win32'
    
    if is_windows:
        from errno import WSAEINVAL as EINVAL
        from errno import WSAEWOULDBLOCK as EWOULDBLOCK
        from errno import WSAEINPROGRESS as EINPROGRESS
        from errno import WSAEALREADY as EALREADY
        from errno import WSAEISCONN as EISCONN
        EAGAIN = EWOULDBLOCK
    else:
        from errno import EINVAL
        from errno import EWOULDBLOCK
        from errno import EINPROGRESS
        from errno import EALREADY
        from errno import EAGAIN
        from errno import EISCONN
    
    hosts = ["http://www.baidu.com", "http://www.amazon.com","http://www.ibm.com",
             "http://www.python.org","http://www.microsoft.com"]
    
    class Listener(object):
    
        _greenreaders = []
    
        @classmethod
        def register(cls,reader):
            cls._greenreaders.append(reader)
    
        @classmethod
        def unregister(cls,reader):
            cls._greenreaders.remove(reader)
    
    class GreenReader(object):
    
        def __init__(self,url):
            self.socket = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
            self.socket.setblocking(0)
            self.url = url
            host = urlparse.urlparse(url).netloc
            self.address = (host,80)
            self.connected = False
            self.terminate = False
            self.send_buffer = 'GET %s HTTP/1.0\r\n\r\n' % url
            self.out = StringIO.StringIO()
            Listener.register(self)
            self._greenloop = greenlet(self._loop)
    
        def connect(self):
            try:
                self.socket.connect(self.address)
                self.connected=True
            except socket.error,why:
                if why[0] in (EWOULDBLOCK, EINPROGRESS, EALREADY,EINVAL):
                    pass
                elif why[0] == EISCONN:
                    self.connected = True
                else:
                    sys.stderr.write("CONNECT ERROR:%s\n" %why[0])
    
        @property
        def recvable(self):
            return True and self.connected
    
        def recv(self):
            try:
                data = self.socket.recv(128)
            except socket.error , why:
                data = why
            if isinstance(data,Exception):
                if why[0] not in [EWOULDBLOCK,EAGAIN]:
                    self.stop()
                    sys.stderr.write("RECEIVE ERROR:%s\n" %why[0])
            elif data:
                self.out.write(data)
            else:
                self.stop()
                self._onfinish()
    
        @property
        def sendable(self):
            return len(self.send_buffer) and self.connected
    
        def send(self):
            try:
                sent = self.socket.send(self.send_buffer)
                self.send_buffer = self.send_buffer[sent:]
            except socket.error,why:
                if why[0] == EWOULDBLOCK:
                    pass
                else:
                    sys.stderr.write("SEND ERROR:%s\n" %why[0])
                    self.stop()
    
        def _loop(self):
            pg = getcurrent().parent
            while not self.terminate:
                if not self.connected:self.connect()
                if self.sendable:self.send()
                if self.recvable:self.recv()
                pg.switch()
    
        def stop(self):
            self.terminate = True
            Listener.unregister(self)
            self._close()
    
        def _close(self):
            try:
                self.socket.close()
            except socket.error,why:
                sys.stderr.write("CLOSE ERROR:%s\n" %why[0])
    
        def __repr__(self):
            return "Reading %s" %self.url
    
        def _onfinish(self):
            context = self.out.getvalue()
            if context is not None:
                try:
                    title = BeautifulSoup(context).title.string
                except HTMLParser.HTMLParseError:
                    print "paser %s tile failure" %self.url
                    return
                print "%s : %s" %(self.url,title)
    
    def mainloop():
        while Listener._greenreaders:
            for gr in Listener._greenreaders:
                gr._greenloop.switch()
    
    def concuryRead():
        start = time.time()
        grs = []
        for i in range(30):
            for host in hosts:
                grs.append(GreenReader(host))
        mainloop()
        end = time.time()
        print "Elapsed Time : %d" %(end-start)
    
    if __name__ == '__main__':
        concuryRead()



**不同版本完成150个页面下载的时间对比：**


	
  * 单线程版本 ： 252s

	
  * 多线程版本： 17s

	
  * 多进程版本： 40s

	
  * 阻塞式stackless： 240s

	
  * acyncore版本 ： 13s

	
  * greenlet版本：6s

	
  * gevent版本：6s



上面的结果只能供参考没有严格的对比意义，但是基本能说明不同并发模型的性能差异。其中greenlet和gevent两个版本是性能最好的，而且两者完成时间一样。这个是因为两者实现基本一样，虽然gevent版本看起来简单很多，但协程和网络IO部分其实都是一样，只是gevent已经封装好了，用起来跟省事。gevent的线程也是用的greenlet，网络操作部分gevent版本用的是urllib2，但实际上通过monkey，也将socket换成了unblocking，跟上面greent版本socket部分基本完全一样。

最后补充说明一点，协程在这里还并没有完全发挥优势，因为这里实验只有150个并发请求，协程真正的优势是CPU计算资源上的开销，当并发数高到10K时，对系统资源的低消耗才是他真正的优势。


