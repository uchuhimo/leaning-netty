Netty学习笔记
============

Netty学习笔记, 包括代码学习, 源码分析.基于最新的netty5, 用于记录Netty学习过程的各种信息和心得.

# 章节内容

* [Buffer](buffer/buffer.md)
	* [ByteBuf继承结构](buffer/inheritance.md)
	* [ByteBuf重要特性](buffer/important_features.md)
        * [Pooled buffer](buffer/pooled_buffer.md)
        * [Reference Count](buffer/reference_count.md)
        * [Zero Copy](buffer/zero_copy.md)
* [Channel](channel/channel.md)
	* [接口Channel](channel/interface_Channel.md)
	* [类AbstractChannel](channel/class_AbstractChannel.md)
	* [Channel Unsafe](channel/interface_Unsafe.md)
* [Eventloop](eventloop/eventloop.md)
	* [接口EventExecutor](eventloop/interface_EventExecutor.md)
	* [接口EventLoop](eventloop/interface_EventLoop.md)
	* [NIO EventLoop](eventloop/nio_eventloop.md)

内容陆续添加中......

注: 如果你看到的是github的源代码,请点击[这里](http://skyao.github.io/leaning-netty/) 查看html内容.

