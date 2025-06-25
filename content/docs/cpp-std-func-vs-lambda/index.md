---
title: C++ std::function 与 lambda
description: 浅浅比较下cpp的std::function与lambda
slug: cpp-std-function-lambda
date: 2019-04-08 00:00:00+0800
image:
categories:
    - learn
tags:
    - c++
    - lambda
    - std::function
toc: true
---

c++中的lambda可以理解为匿名的闭合类。labmda实例可以被当作函数调用(仿函数functor type)。  
std::function<R(T1, T2, ..., TN)>是一个仿函数类，可以将不同的lambda类(参数和返回值相同)进行统一的调用处理。实际上，闭合类型会被隐式转换为std::function。  

<!--more-->

对于std::function，MSLIB、GCCLIB、Boost都有自己的实现。

std::function在创建时的隐藏开销主要有两点：  
* 构造函数在以functor 类型为参数进行创建时，会进行多次拷贝。MSLIB和GCCLIB会进行4次拷贝，Boost会进行7次。(不是最大的开销)  
* std::function本身的大小是固定的，当functor的大小较小时，可以直接将数据放入std::function的数据字段里。当functor的大小较大时，std::function会在堆上分配内存用于存放functor的数据。堆上实际分配的内存大小取决于平台和对齐等问题。通常，MSLIB 12字节，GCCLIB 16字节，Boost 24字节是能够将数据直接包含在std::function的字节数。  

