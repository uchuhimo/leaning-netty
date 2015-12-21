ChannelHandler相关接口
=================

# 接口ChannelHandler

ChannelHandler接口只定义了两个方法:

1. handlerAdded: 在ChannelHandler被添加到实际的context并准备处理事件后被调用
2. handlerRemoved: 在ChannelHandler被从实际的context中删除并不再处理事件后被调用

```java
void handlerAdded(ChannelHandlerContext ctx) throws Exception;
void handlerRemoved(ChannelHandlerContext ctx) throws Exception;
```

补充: 还有一个exceptionCaught()方法, 但是这个方法已经被标注为@Deprecated, javaoc上说"part of ChannelInboundHandler", 应该是被转移到ChannelInboundHandler中了.

此外, 还有个@Sharable 标签的定义:

```java
@Inherited
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface Sharable {
    // no value
}
```

前面说过, 这个@Sharable 标签只是用于文档目的.


# 接口ChannelInboundHandler

接口ChannelInboundHandler的定义, 继承自接口ChannelHandler:

```java
public interface ChannelInboundHandler extends ChannelHandler {}
```

然后添加了一系列的方法定义, 这些方法和inbound操作相关:

- channelRegistered(): ChannelHandlerContext的Channel被注册到EventLoop
- channelUnregistered(): ChannelHandlerContext的Channel被从EventLoop中注销
- channelActive(): ChannelHandlerContext的Channel被激活
- channelInactive(): 被注册的ChannelHandlerContext的Channel现在被取消并到达了它生命周期的终点
- channelRead(): 当Channel读取到一个消息时被调用
- channelReadComplete(): 当前读操作读取的最后一个消息被channelRead()方法消费时调用. 如果ChannelOption.AUTO_READ 属性被设置为off, 不会再尝试从当前channel中读取inbound数据, 直到ChannelHandlerContext.read()方法被调用.
- userEventTriggered(): 当用户事件被触发时调用
- channelWritabilityChanged(): 当channel的可写状态变化时被调用, 可以通过Channel.isWritable()来检查状态.
- exceptionCaught(): 当Throwable被抛出时被调用

```java
void channelRegistered(ChannelHandlerContext ctx) throws Exception;
void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
void channelActive(ChannelHandlerContext ctx) throws Exception;
void channelInactive(ChannelHandlerContext ctx) throws Exception;
void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
```

# 接口ChannelOutboundHandler

接口ChannelOutboundHandler的定义, 继承自接口ChannelHandler:

```java
public interface ChannelOutboundHandler extends ChannelHandler {}
```

然后添加了一系列的方法定义, 这些方法和outbound操作相关:

- bind(): 当bind操作发生时被调用
- connect(): 当connect操作发生时被调用
- disconnect(): 当disconnect操作发生时被调用
- close(): 当close操作发生时被调用
- deregister(): 当在当前已经注册的EventLoop中发生deregister操作时被调用
- read(): 拦截 ChannelHandlerContext.read()方法
- write(): 当write操作发生时被调用. 写操作将通过ChannelPipeline写信息. 这些信息做好了被flush到实际channel中的准备, 一旦Channel.flush()被调用.
- flush(): 当flush操作发生时被调用. flush操作将试图将所有之前写入的被暂缓的信息flush出去.

```java
void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception;
void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void read(ChannelHandlerContext ctx) throws Exception;
void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
void flush(ChannelHandlerContext ctx) throws Exception;
```


