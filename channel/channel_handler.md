Channel Handler
==============

# 继承结构

ChannelHandler的类继承结构如下:

```uml
@startuml

ChannelHandler <|-left- ChannelInboundHandler
ChannelHandler <|-right- ChannelOutboundHandler

ChannelHandler <|-- ChannelHandlerAdapter

ChannelInboundHandler <|-- ChannelInboundHandlerAdapter
ChannelHandlerAdapter <|-- ChannelInboundHandlerAdapter

ChannelOutboundHandler <|-- ChannelOutboundHandlerAdapter
ChannelHandlerAdapter <|-- ChannelOutboundHandlerAdapter

ChannelInboundHandlerAdapter <|-- ChannelDuplexHandler
ChannelOutboundHandler <|-- ChannelDuplexHandler

ChannelInboundHandlerAdapter <|-- ChannelInitializer

interface ChannelHandler {
+ handlerAdded(ctx)
+ handlerRemoved(ctx)
+ @interface Sharable
}

interface ChannelInboundHandler {
+ void channelRegistered(ctx)
+ void channelUnregistered(ctx)
+ void channelActive(ctx)
+ void channelInactive(ctx)
+ void channelRead(ctx, Object msg)
+ void channelReadComplete(ctx)
+ void userEventTriggered(ctx, evt)
+ void channelWritabilityChanged(ctx)
+ void exceptionCaught(ctx, cause)
}

interface ChannelOutboundHandler {
+ void bind(ctx, localAddress, promise)
+ void connect(ctx, remoteAddress, localAddress, promise)
+ void disconnect(ctx, promise)
+ void close(ctx, promise)
+ void deregister(ctx, promise)
+ void read(ctx)
+ void write(ctx, msg, promise)
+ void flush(ctx)
}

abstract class ChannelHandlerAdapter {
~ boolean added
}

class ChannelInboundHandlerAdapter {
}

class ChannelOutboundHandlerAdapter {
}

class ChannelDuplexHandler {
}

abstract ChannelInitializer {
# {abstract} void initChannel(ch)
}

@enduml
```

# 功能

Channel Handler的功能, 如在javadoc中所说:

> 处理I/O事件或者拦截I/O操作, 并转发给它所在ChannelPipeline中的下一个handler.


## 子类型

ChannelHandler本身并不提供很多方法, 但是通常需要实现它的子类型之一:

- ChannelInboundHandler 用于处理inbound I/O事件
- ChannelOutboundHandler 用于处理outbound I/O事件

或者, 为了方便使用, 提供了下面的adapter 类:

- ChannelInboundHandlerAdapter 用于处理inbound I/O事件
- ChannelOutboundHandlerAdapter 用于处理outbound I/O事件
- ChannelDuplexHandler 用于同时处理 inbound 和 outbound事件

## 上下文对象

ChannelHandler伴随有一个ChannelHandlerContext对象. ChannelHandler被设定为通过一个context对象来和它所属的ChannelPipeline交互.

通过使用这个context对象, ChannelHandler可以在上游和下游中传递事件, 动态修改pipeline, 或者存储handler特定的信息(使用AttributeKeys).

## 状态管理

ChannelHandler通常需要存储一些状态化的信息. 最简单的而且推荐的方式是使用成员变量:

### 使用成员变量

```java
public interface Message {
 // your methods here
}

public class DataServerHandler extends SimpleChannelInboundHandler<Message> {

    private boolean loggedIn;

    @Override
    public void channelRead0(ChannelHandlerContext ctx, Message message) {
     Channel ch = e.getChannel();
     if (message instanceof LoginMessage) {
         authenticate((LoginMessage) message);
         loggedIn = true;
     } else (message instanceof GetDataMessage) {
         if (loggedIn) {
             ch.write(fetchSecret((GetDataMessage) message));
         } else {
             fail();
         }
     }
    }
    ...
}
```

因为这个handler实例有一个归属于某一个特定连接的状态变量, 我们不得不为每个新channel创建一个新的handler实例来.

```java
// Create a new handler instance per channel.
// See ChannelInitializer.initChannel(Channel).
public class DataServerInitializer extends ChannelInitializer<Channel> {
 @Override
 public void initChannel(Channel channel) {
 	// 注意这里的DataServerHandler的实例是每次initChannel时都全新new出来一个
    // 这保证了DataServerHandler实例和channel的严格一对一关系
	channel.pipeline().addLast("handler", new DataServerHandler());
 }
}
```

### 使用AttributeKeys

Although it's recommended to use member variables to store the state of a handler, for some reason you might not want to create many handler instances. In such a case, you can use AttributeKeys which is provided by ChannelHandlerContext:

虽然推荐使用成员变量来存储handler的状态, 但是因为某些原因,可能不想创建太多的handler实例. 在这种情况下, 可以使用ChannelHandlerContext提供的AttributeKeys:

```java
public interface Message {
 // your methods here
}

@Sharable
public class DataServerHandler extends SimpleChannelInboundHandler<Message> {
 // 申明一个AttributeKey
 private final AttributeKey<Boolean> auth = AttributeKey.valueOf("auth");

 @Override
 public void channelRead(ChannelHandlerContext ctx, Message message) {
 	 // 从context对象中获取AttributeKey对应的Attribute
     Attribute<Boolean> attr = ctx.attr(auth);
     Channel ch = ctx.channel();
     if (message instanceof LoginMessage) {
         authenticate((LoginMessage) o);
         // 保存结果到Attribute
         attr.set(true);
     } else (message instanceof GetDataMessage) {
     	 // 检查Attribute保存的值
         if (Boolean.TRUE.equals(attr.get())) {
             ch.write(fetchSecret((GetDataMessage) o));
         } else {
             fail();
         }
     }
 }
 ...
}
```

现在handler的状态被附加在ChannelHandlerContext, 你可以将同一个handler实例添加到不同的pipeline(对应不同的channel):

```java
public class DataServerInitializer extends ChannelInitializer<Channel> {

 // 注意这里的DataServerHandler的实例是共享的, 只有一个
 private static final DataServerHandler SHARED = new DataServerHandler();

 @Override
 public void initChannel(Channel channel) {
 	 // 所有的channel和pipeline, 都使用这同一个handler的实例
     channel.pipeline().addLast("handler", SHARED);
 }
}
```

## @Sharable 标签

在上面的使用AttributeKey的例子中, 你可能已经发现@Sharable标签.

如果一个ChannelHandler被标注为@Sharable标签, 这意味着你可以仅仅创建这个handler的一个实例然后将它多次添加到一个或者多个ChannelPipelines中, 不会导致冲突或竞争.

如果没有特别标明这个标签, 你将不得不在每次你将它添加到pipeline时创建一个新的handler实例, 因为它有不共享的状态例如成员变量.

这个标签仅用于文档目的.



