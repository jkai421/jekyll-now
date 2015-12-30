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

RPyC为分布式计算和集群提供了很好的基础，它是与体系结构和平台无关的，支持同步和异步触发。
客户端和服务端是对等的关系。在这些特点的基础上，可以很容易构建一个具有如下特点的分布式计算框架：

* 能够处理节点的加入集群和离开集群
* 能处理负载均衡和节点错误
* 能够收集所有节点的计算结果
* 根据运行时分析结果迁移对象和代码


##教程

###一、 经典RPyC介绍

这一则教程是针对经典的PPyC进行的，也就是RpyC 2.60。RPyC 3是重新设计的一个库，有一些重要的修改。
但是如果你熟悉了RpyC 2.60，那么对于RpyC 3也同样不会有任何问题。

**运行一个服务器**

我们先从基本的开始：运行一个服务器。本教程中，我们将服务器和客户端运行在同一台机器上。
在Windows上双击文件就可以完成。

![启动服务器](/images/running-classic-server.png)

第一行显示了服务器运行的参数：SLAVE 表示SlaveService，tid是线程的ID，0.0.0.0:18812是服务器
绑定的PI和端口，0.0.0.0表示全网监听。

**运行一个客户端**

接下来是运行一个客户端来沟通服务器，十分简单。

    import rpyc
    conn = rpyc.classic.connect("localhost")

> *注意*
你需要在实际使用时将localhost改为你的RPyC主机IP。如果服务器不是运行在默认端口18812上，
还需要显示指定端口号port。

**modules命名空间**

连接上服务器后，我们需要控制它。连接对象conn的modules属性将服务器端的模块空间暴露给客户端。

    # dot notation
    mod1 = conn.modules.sys # access the sys module on the server

    # bracket notation
    mod2 = conn.modules["xml.dom.minidom"] # access the xml.dom.minidom module on the server

> *注意*
有两种方式能够访问到远程模块，使用‘.’方式比较直观但是只能对顶层模块使用，比较有局限性。
使用方括号方式可以访问更多层次的模块。


![client](/images/running-classic-client.png)
一个简单的客户端

截图的第一部分打印当前目录，并没有什么特别的。但是第二部分打出的则是服务器的当前目录了，就是这么简单。




###五、异步操作和事件

**异步**
教程的最后一部分来看看关于RPC的更高端的问题，异步操作。到目前为止，你所看到的代码都是同步操作，
同步操作与正常编程的思维方式相似，调用一个函数，然后阻塞等待其执行完毕。而异步触发则不必阻塞等待
运行结果，而是发出请求之后继续运行。所以异步触发得到的不是一次调用返回的结果。而是一个特殊的对象，
成为异步对象，用于存放最终结果。

为了以异步方式触发远程函数，需要以async对函数进行包装，async会创建一个外观方法返回异步结果。
异步结果包含一些属性和方法：

* ready - 指示结果是否到达
* error - 指示结果是值还是异常
* expired - 指示结果是否已经超时。 如果没有set_expiry，对象不会超时。
* value - 异步结果的值。如果结果未就绪，访问该属性会阻塞；如果结果为异常，访问该属性会抛出改异常；
如果结果为超时，访问该属性会抛出异常；否则，返回正常结果。
* wait() - 等待结果到达，或者对象超时。
* add_callback(func) - 增加一个在结果到达时触发的回调。
* set_expiry(seconds) - 为异步对象设置一个超时时间。

下面看一个简单的例子。

    >>> import rpyc
    >>> c=rpyc.classic.connect("localhost")
    >>> c.modules.time.sleep
    <built-in function sleep>
    >>> c.modules.time.sleep(2) # i block for two seconds, until the call returns

     # wrap the remote function with async(), which turns the invocation asynchronous
    >>> asleep = rpyc.async(c.modules.time.sleep)
    >>> asleep
    async(<built-in function sleep>)

    # invoking async functions yields an AsyncResult rather than a value
    >>> res = asleep(15)
    >>> res
    <AsyncResult object (pending) at 0x0842c6bc>
    >>> res.ready
    False
    >>> res.ready
    False

    # ... after 15 seconds...
    >>> res.ready
    True
    >>> print res.value
    None
    >>> res
    <AsyncResult object (ready) at 0x0842c6bc>

一个更有意思的例子:

    >>> aint = rpyc.async(c.modules.__builtin__.int)  # async wrapper for the remote type int

    # a valid call
    >>> x = aint("8")
    >>> x
    <AsyncResult object (pending) at 0x0844992c>
    >>> x.ready
    True
    >>> x.error
    False
    >>> x.value
    8

    # and now with an exception
    >>> x = aint("this is not a valid number")
    >>> x
    <AsyncResult object (pending) at 0x0847cb0c>
    >>> x.ready
    True
    >>> x.error
    True
    >>> x.value #
    Traceback (most recent call last):
    ...
      File "/home/tomer/workspace/rpyc/core/async.py", line 102, in value
        raise self._obj
    ValueError: invalid literal for int() with base 10: 'this is not a valid number'
    >>>

**事件**

异步和回调的组合则产生了一个有趣的结果：异步回调，也叫作事件。事件是由“事件生产者”发出，
将相关变化通知“事件消费者”，这个过程通常是单向的。在RPC中，事件通过异步回调实现，返回值
被忽略。看下面一个文件监控器的例子，这个监控器使用os.stat()监控一个文件的变化，并在变化
发生时通知客户端。

    import rpyc
    import os
    import time
    from threading import Thread

    class FileMonitorService(rpyc.SlaveService):
        class exposed_FileMonitor(object):   # exposing names is not limited to methods :)
            def __init__(self, filename, callback, interval = 1):
                self.filename = filename
                self.interval = interval
                self.last_stat = None
                self.callback = rpyc.async(callback)   # create an async callback
                self.active = True
                self.thread = Thread(target = self.work)
                self.thread.start()
            def exposed_stop(self):   # this method has to be exposed too
                self.active = False
                self.thread.join()
            def work(self):
                while self.active:
                    stat = os.stat(self.filename)
                    if self.last_stat is not None and self.last_stat != stat:
                        self.callback(self.last_stat, stat)   # notify the client of the change
                    self.last_stat = stat
                    time.sleep(self.interval)

    if __name__ == "__main__":
        from rpyc.utils.server import ThreadedServer
        ThreadedServer(FileMonitorService, port = 18871).start()
下面是一个事件的演示:

    >>> import rpyc
    >>>
    >>> f = open("/tmp/floop.bloop", "w")
    >>> conn = rpyc.connect("localhost", 18871)
    >>> bgsrv = rpyc.BgServingThread(conn)  # creates a bg thread to process incoming events
    >>>
    >>> def on_file_changed(oldstat, newstat):
    ...     print "file changed"
    ...     print "    old stat: %s" % (oldstat,)
    ...     print "    new stat: %s" % (newstat,)
    ...
    >>> mon = conn.root.FileMonitor("/tmp/floop.bloop", on_file_changed) # create a filemon

    # wait a little for the filemon to have a look at the original file

    >>> f.write("shmoop") # change size
    >>> f.flush()

    # the other thread then prints
    file changed
        old stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 0L, 1225204483, 1225204483, 1225204483)
        new stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 6L, 1225204483, 1225204556, 1225204556)

    >>>
    >>> f.write("groop") # change size
    >>> f.flush()
    file changed
        old stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 6L, 1225204483, 1225204556, 1225204556)
        new stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 11L, 1225204483, 1225204566, 1225204566)

    >>> f.close()
    >>> f = open(filename, "w")
    file changed
        old stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 11L, 1225204483, 1225204566, 1225204566)
        new stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 0L, 1225204483, 1225204583, 1225204583)

    >>> mon.stop()
    >>> bgsrv.stop()
    >>> conn.close()

在这个实例中采用的是BgServingThread，启动一个后台线程来服务所有请求，而主线程就可以空闲出来完成自己的事情。

    >>> f = open("/tmp/floop.bloop", "w")
    >>> conn = rpyc.connect("localhost", 18871)
    >>> mon = conn.root.FileMonitor("/tmp/floop.bloop", on_file_changed)
    >>>

    # change the size...
    >>> f.write("shmoop")
    >>> f.flush()

    # ... seconds pass but nothing is printed ...
    # until we make some interaction with the connection: printing a remote object invokes
    # the remote __str__ of the object, so that all pending requests are suddenly processed
    >>> print mon
    file changed
        old stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 0L, 1225205197, 1225205197, 1225205197)
        new stat: (33188, 1564681L, 2051L, 1, 1011, 1011, 6L, 1225205197, 1225205218, 1225205218)
    <__main__.exposed_FileMonitor object at 0xb7a7a52c>
    >>>