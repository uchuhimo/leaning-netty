接口ChannelHandlerContext
========================

# 接口属性

## name属性

```java
String name();
```

ChannelHandlerContext的名字, unique,不能重复.

这个名字在ChannelHandler被添加到ChannelPipeline时使用. 例如: p1.addLast("f1", handler);

也可以用来通过名字在ChannelPipeline访问已经注册的ChannelHandler.

```java
ChannelHandler get(String name);
```

实际上就是在ChannelPipeline的各个方法中出现的参数 name 或者 baseName.

# 接口方法

## 获取属性

### removed

removed属性如果是true, 表明属于当前context的ChannelHandler已经从ChannelPipeline中移除.

注意这个方法通常只在EventLoop中调用.

```java
boolean isRemoved();
```

## 获取context保存相关对象

以下方法用于获取context相关的几个重要对象, context中保存有他们的引用:

- channel: 当前的channel, 代表当前的连接
- handler: 当前的ChannelHandler, 记住context是一个handler实例添加到一个pipeline时产生的一个"契约"(subscription).
- pipeline: 当前所在的ChannelPipeline, 解释同上
- executor: EventExecutor, 用于执行任意任务
- alloc: 当前被分配的ByteBufAllocator, 用于ByteBuf的内存分配
- invoker: netty5.0中多暴露了一个invoker, netty4.1中没有

```java
Channel channel();
ChannelHandler handler();
ChannelPipeline pipeline();
EventExecutor executor();
ByteBufAllocator alloc();
```

## Inbound的I/O事件方法

这些方法会导致当前Channel的ChannelPipeline中包含的下一个ChannelInboundHandler的相应的方法被调用:

```java
ChannelHandlerContext fireChannelRegistered();
ChannelHandlerContext fireChannelUnregistered();
ChannelHandlerContext fireChannelActive();
ChannelHandlerContext fireChannelInactive();
ChannelHandlerContext fireExceptionCaught(Throwable cause);
ChannelHandlerContext fireUserEventTriggered(Object event);
ChannelHandlerContext fireChannelRead(Object msg);
ChannelHandlerContext fireChannelReadComplete();
ChannelHandlerContext fireChannelWritabilityChanged();
```

## outbound的I/O方法

这些方法会导致当前Channel的ChannelPipeline中包含的下一个ChannelOutboundHandler的相应的方法被调用:

```java
ChannelFuture bind(SocketAddress localAddress);
ChannelFuture connect(SocketAddress remoteAddress);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
ChannelFuture disconnect();
ChannelFuture close();
ChannelFuture deregister();
```

以及以上方法的带promise参数的重载方法:

```java
ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
ChannelFuture disconnect(ChannelPromise promise);
ChannelFuture close(ChannelPromise promise);
ChannelFuture deregister(ChannelPromise promise);
```

## I/O操作方法

```java
// 请求从Channel中读取数据到第一个inbound buffer, 如果数据被读取则触发ChannelInboundHandler.channelRead(ChannelHandlerContext, Object) 事件, 并触发channelReadComplete事件以便handler可以决定继续读取.
// 将会导致当前Channel的ChannelPipeline中包含的下一个ChannelOutboundHandler的ChannelOutboundHandler.read(ChannelHandlerContext)方法被调用
ChannelHandlerContext read();

// 请求通过当前ChannelHandlerContext将消息写到ChannelPipeline.
// 这个方法不会要求做实际的flush, 因此一旦你想将所有待处理的数据flush到实际的transport请务必调用flush()方法.    
ChannelFuture write(Object msg);
ChannelFuture write(Object msg, ChannelPromise promise);

// 请求通过ChannelOutboundInvokerflush所有待处理的消息
ChannelHandlerContext flush();

// write()和flush()方法的快捷调用版本
ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
ChannelFuture writeAndFlush(Object msg);
```

TODO: 不理解为什么read()会导致ChannelOutboundHandler.read(ChannelHandlerContext)方法被调用, 为什么是Outbound? 后面注意看实现代码.


## 创建Future/Promise的方法

```java
ChannelProgressivePromise newProgressivePromise();
ChannelFuture newSucceededFuture();
ChannelFuture newFailedFuture(Throwable cause);
ChannelPromise voidPromise();
```


