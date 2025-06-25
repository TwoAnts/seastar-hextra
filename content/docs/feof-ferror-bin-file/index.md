---
title: feof()和ferror() 与 二进制文件
description: feof()与ferror()的区别
slug: feof-ferror-bin-file
date: 2017-09-25 00:00:00+0800
image:
categories:
    - learn
tags:
    - linux
    - feof()
    - ferror()
toc: true
---

当以二进制方式(binary mode)打开文件时，使用fgetc()很可能读到0xff字节。

由于EOF恰恰就是-1，也即0xff, 同时fgetc()出错时，也返回-1，所以当fgetc()返回-1时，究竟是文件尾？还是字节0xff？还是操作出错呢？

这时候就需要feof()和ferror()出场了。

<!--more-->

* feof(): 当读到EOF时，判断文件是否到尾部。（到尾部之前，可能会多给一个EOF字符。）
* ferror(): 当读到EOF时，判断文件是否出错。


> 不要简单的用`while(!feof())`来判断文件是否结束。因为当文件出错时，feof()是不会返回真的，于是就可能陷入死循环。
