类AbstractChannelHandlerContext
==============================

# 类定义

```java
abstract class AbstractChannelHandlerContext implements ChannelHandlerContext, ResourceLeakHint {}
```

# 类属性

只有一个构造函数, name/pipeline/invoker/inbound/outbound都在这里初始化:

```java
private final boolean inbound;
private final boolean outbound;
private final ChannelPipeline pipeline;
private final String name;
private boolean removed;

final ChannelHandlerInvoker invoker;

AbstractChannelHandlerContext(
        DefaultChannelPipeline pipeline, ChannelHandlerInvoker invoker,
        String name, boolean inbound, boolean outbound) {

    if (name == null) {
        throw new NullPointerException("name");
    }

    this.pipeline = pipeline;		// 为什么pipeline不做null检测? 从代码实现上看肯定不能为null的, 既然检查了name就应该检查pipeline
    this.name = name;
    this.invoker = invoker;			// 这个invoker是容许设置为null的!! 看invoker()函数

    this.inbound = inbound;
    this.outbound = outbound;
}
```

TODO: 准备增加pipeline的null check, 然后pull给netty. 等上一个request被接受再试.

有几个属性的getter方法比较特殊:

```java
@Override
public ByteBufAllocator alloc() {
	// alloc从channel的config中获取
    return channel().config().getAllocator();
}

@Override
public EventExecutor executor() {
	// EventExecutor来自invoker
    return invoker().executor();
}

public ChannelHandlerInvoker invoker() {
	// 检查invoker是否有设置为null, 见构造函数
    if (invoker == null) {
    	// 如果构造函数中设置为null了,则通过unsafe来获取
        return channel().unsafe().invoker();
    } else {
        return invoker;
    }
}
```

## invoker属性

invoker容许为null比较奇怪, 特意查了一下, 有两个地方是可以设置invoker为null的:

```java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
}
TailContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, TAIL_NAME, true, false);
}
```

TODO: 后面看HeadContext/TailContext的代码实现时再仔细看.

## removed属性

这是一个非常特别的属性, 只用于非常极端的情况, getter/setter方法如下:

```java
void setRemoved() {
    removed = true;
}

@Override
public boolean isRemoved() {
    return removed;
}
```

其中etRemoved()还不是public的, 调用的地方只有一处, 在类DefaultChannelPipeline:

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        ctx.handler().handlerRemoved(ctx);
        ctx.setRemoved();
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```

只有当一个handler被从pipiline上remove或者replace时, 才会代用setRemoved()方法.

而这个属性使用的地方也非常少, 只找到三处调用, 从注释上看都和issues 1664相关:

```java
// Check if this handler was removed before continuing the loop.
// If it was removed, it is not safe to continue to operate on the buffer.
//
// See https://github.com/netty/netty/issues/1664
if (ctx.isRemoved()) {
    break;
}
```

## next 和 prev

每个AbstractChannelHandlerContext都保存有两个属性, next和prev, 明显是做链表.

```java
volatile AbstractChannelHandlerContext next;
volatile AbstractChannelHandlerContext prev;
```

在DefaultChannelPipeline中, 则保存了名为head和tail的两个引用:

```java
final class DefaultChannelPipeline implements ChannelPipeline {
    final AbstractChannelHandlerContext head;
    final AbstractChannelHandlerContext tail;
}
```

这样在DefaultChannelPipeline中就实现了一个从head一路next/next到tail, 再从tail一路prev/prev到head的一个双向链表.

## inbound 和 outbound

这两个属性用来表明当前handler是inbound还是outbound, 通常和ChannelInboundHandler/ChannelOutboundHandler联系起来,.

比如HeadContext实现了ChannelOutboundHandler, 而TailContext实现了ChannelInboundHandler, 他们在调用super构造函数时就写死了inbound和outbound属性:

```java
static final class HeadContext extends AbstractChannelHandlerContext implements ChannelOutboundHandler {
    HeadContext(DefaultChannelPipeline pipeline) {
        super(pipeline, null, HEAD_NAME, false, true);	// inbound=false, outbound=true
    }
}
static final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {
    TailContext(DefaultChannelPipeline pipeline) {
        super(pipeline, null, TAIL_NAME, true, false);	// inbound=true, outbound=false
    }
}
```

注: netty设计上是使用两个boolean来记录inbound/outbound, 不知道是否会存在即是inbound又是outbound
的情况?

在DefaultChannelHandlerContext的实现中, 通过检查传入的handler来判断inbound/outbound, 判断的方法非常直接, instanceof ChannelInboundHandler/ChannelOutboundHandler:

```java
DefaultChannelHandlerContext(
        ChannelPipeline pipeline, ChannelHandlerInvoker invoker, String name, ChannelHandler handler) {
    super(pipeline, invoker, name, isInbound(handler), isOutbound(handler));
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
}
private static boolean isInbound(ChannelHandler handler) {
    return handler instanceof ChannelInboundHandler;
}
private static boolean isOutbound(ChannelHandler handler) {
    return handler instanceof ChannelOutboundHandler;
}
```

注: 如果一个handler同时实现ChannelInboundHandler/ChannelOutboundHandler两个接口, 是有可能的.但具体是否有这种情况还不清楚, 继续看代码.

# 类方法

## 来自AttributeMap的方法

ChannelHandlerContext申明继承AttributeMap, AbstractChannelHandlerContext中有实现AttributeMap要求的两个方法:

```java
@Override
public <T> Attribute<T> attr(AttributeKey<T> key) {
    return channel().attr(key);
}

@Override
public <T> boolean hasAttr(AttributeKey<T> key) {
    return channel().hasAttr(key);
}
```

最终还是delegate给channel的对应方法了.

## Inbound的I/O事件方法

这些方法最终都是delegate给invoker的对应方法, 以fireChannelRegistered()为例:

```java
@Override
public ChannelHandlerContext fireChannelRegistered() {
	// 找到下一个inbound的context
    AbstractChannelHandlerContext next = findContextInbound();
    // 调用invoker的invokeChannelRegistered()方法
    next.invoker().invokeChannelRegistered(next);
    return this;
}

private AbstractChannelHandlerContext findContextInbound() {
	// ctx初始化为指向当前context, 也就是this
    AbstractChannelHandlerContext ctx = this;
    do {
    	// 然后指向ctx的next指针
        ctx = ctx.next;
    } while (!ctx.inbound);  // 检查是否是inbound, 如果不是则继续next,直到找到下一个
    return ctx;
}
```

这和javadoc中对这些方法的说明一致: "这些方法会导致当前Channel的ChannelPipeline中包含的下一个ChannelInboundHandler的相应的方法被调用"

类似的方法有一下:

```java
ChannelHandlerContext fireChannelRegistered();
ChannelHandlerContext fireChannelUnregistered();
ChannelHandlerContext fireChannelActive();
ChannelHandlerContext fireChannelInactive();
ChannelHandlerContext fireExceptionCaught(Throwable cause);		// 这个例外!
ChannelHandlerContext fireUserEventTriggered(Object event);
ChannelHandlerContext fireChannelRead(Object msg);
ChannelHandlerContext fireChannelReadComplete();
ChannelHandlerContext fireChannelWritabilityChanged();
```

但是fireExceptionCaught方法非常的特殊, 与众不同的是, 这个方法中的next是**不区分inbound和outbound**的, 直接取this.next:

```java
@Override
public ChannelHandlerContext fireExceptionCaught(Throwable cause) {
    AbstractChannelHandlerContext next = this.next;
    next.invoker().invokeExceptionCaught(next, cause);
    return this;
}
```

TODO: 暂时还没有理解为什么fireExceptionCaught()方法会有这个特例?

## outbound的I/O方法

首先, 下面这些方法在实现时都会转变为他们对应的带promise的版本:

```java
ChannelFuture bind(SocketAddress localAddress);
ChannelFuture connect(SocketAddress remoteAddress);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
ChannelFuture disconnect();
ChannelFuture close();
ChannelFuture deregister();
```

例如bind()方法, 通过newPromise()方法创建一个promise之后, 就调用带promise的版本了:

```java
@Override
public ChannelFuture bind(SocketAddress localAddress) {
    return bind(localAddress, newPromise());
}

@Override
public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}
```

继续看bind()方法的实现, 发现机制和前面的雷同:

```java
@Override
public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
	// 找到下一个outbound的context
    AbstractChannelHandlerContext next = findContextOutbound();
    // 调用invokeBind()方法
    next.invoker().invokeBind(next, localAddress, promise);
    return promise;
}
```

## I/O操作方法

实现方式类似, 不细看了.




