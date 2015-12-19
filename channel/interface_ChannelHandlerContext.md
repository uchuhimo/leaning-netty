接口ChannelHandlerContext
========================

# 接口功能

参考[接口ChannelHandlerContext的javadoc](http://netty.io/4.1/api/io/netty/channel/ChannelHandlerContext.html)说明, 接口ChannelHandlerContext的作用:

> 使得ChannelHandler可以和它的ChannelPipeline和其他handlers交互.
>   除此之外, handler可以通知ChannelPipeline中的下一个ChannelHandler, 此外还可以动态的修改它所属的ChannelPipeline

具体功能如下(下面翻译自javadoc):

1. 通知

	可以通过调用这里提供的多个方法中的某一个方法来通知在同一个ChannelPipeline的最近的handler. 参考ChannelPipeline来理解事件是如何流动的.

2. 修改pipeline

	可以通过调用pipeline()方法来得到handler所属的ChannelPipeline. 有一定分量的(non-trivial)应用能在pipeline中动态的插入, 删除, 或者替换handler.

3. 保存以便后续可以取回使用

	可以保存ChannelHandlerContext以便后续使用, 例如在handler方法外触发一个事件, 甚至可以从不同的线程发起.

	```java
    public class MyHandler extends ChannelHandlerAdapter {

        private ChannelHandlerContext ctx;

        public void beforeAdd(ChannelHandlerContext ctx) {
         this.ctx = ctx;
        }

        public void login(String username, password) {
         ctx.write(new LoginMessage(username, password));
        }
        ...
    }
	```

4. 存储状态化信息

	AttributeMap.attr(AttributeKey) 容许你保存和访问和handler以及它的上下文相关的状态化信息. 参考ChannelHandler来学习多种推荐用于管理状态化信息的方法.

有一个非常重要的信息, 有些出乎意外, 需要特别注意和留神:

> **一个hanlder可以多个context**

请注意, 一个ChannelHandler实例可以被添加到多个ChannelPipeline. 这意味着一个单独的ChannelHandler实例可以有多个ChannelHandlerContext, 因此一个单独的ChannelHandler实例可以在被调用时传入不同的ChannelHandlerContext, 如果它被添加到多个ChannelPipeline.

例如, 下面的handler被加入到pipelines多少次, 就会有多少个相互独立的AttributeKey. 不管它是多次添加到同一个pipeline还是不同的pipeline.

```java
public class FactorialHandler extends ChannelHandlerAdapter {

    private final AttributeKey<Integer> counter = AttributeKey.valueOf("counter");

    // This handler will receive a sequence of increasing integers starting
    // from 1.
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        Attribute<Integer> attr = ctx.getAttr(counter);
        Integer a = ctx.getAttr(counter).get();

        if (a == null) {
        	a = 1;
        }

        attr.set(a * (Integer) msg);
    }
}

// 不同的context对象被赋予"f1", "f2", "f3", 和 "f4", 虽然他们引用到同一个handler实例.
// 由于FactorialHandler在context对象中保存它的状态(使用AttributeKey),
// 一旦这两个pipelines (p1 和 p2)被激活, factorial会计算4次.
FactorialHandler fh = new FactorialHandler();

ChannelPipeline p1 = Channels.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);

ChannelPipeline p2 = Channels.pipeline();
p2.addLast("f3", fh);
p2.addLast("f4", fh);
```

备注: 这个的代码和例子来自javadoc,但是有点问题, ctx.getAttr()应该是ctx.attr(). 已经[pull了一个request给netty](https://github.com/netty/netty/pull/4590).
更新: 修改已经被netty接受, netty4.1版本会包含这个修订.

总结: 也就是说, 在handler被添加到pipiline时, 会创建一个新的context, 无视handler是否是一个实例添加多次.

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


