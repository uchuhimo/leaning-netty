NioEventLoop是一个SingleThreadEventLoop的实现, 将每个Channel注册到NIO Selector并执行multiplexing.

# NioEventLoop的类继承结构

![](./image/nio_eventloop.png)

NioEventLoop的类继承结构比较深,继承结构中比较关键的是:

- AbstractScheduledEventExecutor: netty的EventExecutor,提供对定时任务的支持
- SingleThreadEventExecutor: 在单个线程中执行所有提交的任务
- SingleThreadEventLoop: 单线程版本的EventLoop

这里可以得到一个重要的信息: **NioEventLoop时用单线程来处理NIO任务的! **

在这个多层的继承结构中:

1. ExecutorService/AbstractExecutorService 是属于JDK的代码, 著名的JDK Executor, package是"java.util.concurrent"
2. AbstractEventExecutor/AbstractScheduledEventExecutor/SingleThreadEventExecutor 是属于netty util的通用类, package 是 "io.netty.util.concurrent"
3. SingleThreadEventLoop 是属于netty的通用的event loop实现基类, package是"io.netty.channel"
4. NioEventLoop是属于netty的基于NIO的event loop实现类, package是"io.netty.channel.nio"

后面我们从上到下来研究具体的代码实现.