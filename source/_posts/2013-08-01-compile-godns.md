
---
comments: true
date: 2013-08-01 10:35:50
layout: post
slug: godns
title: 编译godns
categories:
- Go
tags:
- go,dns,godns,dns cache
---

*nix系统编译godns的详细步骤

1.安装golang 

```

$ sudo  wget https://go.googlecode.com/files/go1.1.1.src.tar.gz

$ sudo tar zxvf go1.1.1.src.tar.gz

$ cd go/src

$ sudo ./all.bash

$ sudo mv go /usr/local/go

$ sudo cp /usr/local/go/bin/*  /usr/local/bin 

$ sudo ln -s  /usr/local/bin/go /usr/bin/goma

```

2.设置环境变量


```
$ vim ~/.bashrc

export PATH=$PATH:/usr/local/go/pkg/tool/linux_amd64/

GOBIN=/usr/local/go/bin

GOROOT=/usr/local/go/

GOPATH=$HOME/godns

export GOBIN GOROOT GOPATH

```

设置完成后，运行go env， 看设置是否都生效



3.部署代码

``` 
$ go get github.com/kenshinx/godns
```


4.编译

```
$ cd $GOPATH/src/github.com/kenshinx/godns
$ go build -o godns *.go
```

5.安装

```
sudo cp godns /usr/local/bin/ && sudo chown root:root /usr/local/bin/godns
sudo godns.conf /etc/godns.conf  && sudo chown root:root /etc/godns.conf
```

因为要绑定53端口，因此要以root权限运行。


6.运行

```
sudo nohup /usr/local/bin/godns -c /etc/godns &

```
由于Go语言不能像C，C++一样实现Daemon,因此只能通过nohup的方式在后台运行。
但是nohup的方式运行，管理起来不是很方便，要结束，重启进程还得自己写shell管理脚本。
最好的部署还是通过supervisord这样的东西。


