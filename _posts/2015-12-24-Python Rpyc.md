---
layout: post
title: Python Rpyc中文文档
---

##简介

早就知道rpc，但一直没有很直观的认识，最近研究promelo引擎，了解到游戏服务器各个功能模块
之间的相互调用是通过rpc进行的，简化了socket接口编程的复杂度，就想了解一下Python的rpc接口，
由于没有找到中文文档，就想着要翻译一下，顺便学习了。

##应用场景
这一节罗列了RPyC适合处理的几种任务的例子。
### 远程服务（Web服务）
RPyC从3.00开始转变为面向服务。这样一来，实现一个安全的远程服务变得很简单了。“服务”是一种类，
定义了一系列对外开放的远程接口和远程对象。这些对外开放的接口可以由服务的客户端调用，并返回
结果。举个例子，一家快递公司开放的追踪包裹服务可能是这样的

TrackYourPackage：

    get_current_location(pkgid)
    get_location_history(pkgid)
    get_delivery_status(pkgid)
    report_package_as_lost(pkgid, info)

RPyC默认不允许对远程对象使用getattr方法，另外，安全方面主要是依靠对模型传递能力的控制。
比如对外开放文件操作时，RPyC没有采用文件名层面的方法--开放open()接口同时禁止其他接口，
而是可以向外传递一个打开文件的对象，客户端可以根据这个对象操作文件（读、写、查找等），
但不能访问文件系统的其他部分。


###管理和中心控制
有效的系统管理是非常困难的，你可能需要管理许多类型的平台，大小端编码，不同位宽的机器，
不同的管理工具，以及不同的shell语言等等。更有甚者，你可能需要应对各种不同的传输协议，
（telnet, ftp, ssh等等）

###硬件资源扩展
很多时候我们需要利用另一台物理机的资源。比如一些测试中心或者设备只能连接Solaris SPARC
的机器，但你更习惯于用Windows系统工作。

###并行运行
在CPython中，由于GIL的存在，多个线程不能同时执行字节码，这简化了python解释器的设计，但是
却使得CPython不能利用多核CPU。要达到可扩展的唯一方法是多进程。进程可以避免线程的同步难题，
但是进程间通信却十分麻烦。

使用RPyC让多进程变得容易，把RPyC连接的所有进程看做一个“大进程”。


###分布式计算平台

RPyC为分布式计算和集群提供了很好的基础，它是与体系结构和平台无关的，支持同步和异步触发。客户端和服务端是对等的关系。在这些特点的基础上，可以很容易构建一个具有如下特点的分布式计算框架：

* 能够处理节点的加入集群和离开集群
* 能处理负载均衡和节点错误
* 能够收集所有节点的计算结果
* 根据运行时分析结果迁移对象和代码


##教程

###一、 经典RPyC介绍

这一则教程是针对经典的PPyC进行的，也就是RpyC 2.60。RPyC 3是重新设计的一个库，有一些重要的修改。但是如果你熟悉了RpyC 2.60，那么对于RpyC 3也同样不会有任何问题。

*运行一个服务器*

我们先从基本的开始：运行一个服务器。本教程中，我们将服务器和客户端运行在同一台机器上。在Windows上双击文件就可以完成。

![启动服务器](/images/running-classic-server.png)

第一行显示了服务器运行的参数：SLAVE 表示SlaveService，tid是线程的ID，0.0.0.0:18812是服务器绑定的PI和端口，0.0.0.0表示全网监听。

*运行一个客户端*

接下来是运行一个客户端来沟通服务器，十分简单。

    import rpyc
    conn = rpyc.classic.connect("localhost")

> *注意*
你需要在实际使用时将localhost改为你的RPyC主机IP。如果服务器不是运行在默认端口18812上，还需要显示指定端口号port。

*modules命名空间*

连接上服务器后，我们需要控制它。连接对象conn的modules属性将服务器端的模块空间暴露给客户端。

    # dot notation
    mod1 = conn.modules.sys # access the sys module on the server

    # bracket notation
    mod2 = conn.modules["xml.dom.minidom"] # access the xml.dom.minidom module on the server

> *注意*
有两种方式能够访问到远程模块，使用‘.’方式比较直观但是只能对顶层模块使用，比较有局限性。使用方括号方式可以访问更多层次的模块。


![client](/images/running-classic-client.png)
一个简单的客户端

截图的第一部分打印当前目录，并没有什么特别的。但是第二部分打出的则是服务器的当前目录了，就是这么简单。

