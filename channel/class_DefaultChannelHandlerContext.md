类DefaultChannelHandlerContext
=============================

ChannelHandlerContext的实现类有三个:

1. DefaultChannelHandlerContext
2. HeadContext (内嵌在DefaultChannelPipeline中)
3. TailContext (内嵌在DefaultChannelPipeline中)

# 类DefaultChannelHandlerContext

DefaultChannelHandlerContext的代码特别的少, 基本上除了保存一个handler属性和getter方法, 就没有其他内容了:

```java
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {
    private final ChannelHandler handler;

    DefaultChannelHandlerContext(ChannelPipeline pipeline, ChannelHandlerInvoker invoker, String name, ChannelHandler handler) {
        super(pipeline, invoker, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler;
    }

    @Override
    public ChannelHandler handler() {
        return handler;
    }
}
```

比较奇怪的是, 为什么handler要放在DefaultChannelHandlerContext中, 而不是AbstractChannelHandlerContext?

# TailContext

TailContext和HeadContext有些奇怪, 他们既是ChannelHandlerContext, 又实现了ChannelHandler接口.

TailContext是ChannelInboundHandler:

```java
// A special catch-all handler that handles both bytes and messages.
static final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {
    private static final String TAIL_NAME = generateName0(TailContext.class);

    TailContext(DefaultChannelPipeline pipeline) {
        // invoker特意设置为null
        // inbound=true, outbound=false
        super(pipeline, null, TAIL_NAME, true, false);
    }

    @Override
    public ChannelHandler handler() {
        return this;
    }
}
```

# HeadContext

HeadContext是ChannelOutboundHandler:

```java
static final class HeadContext extends AbstractChannelHandlerContext implements ChannelOutboundHandler {
    private static final String HEAD_NAME = generateName0(HeadContext.class);

    private final Unsafe unsafe;

    HeadContext(DefaultChannelPipeline pipeline) {
        // invoker特意设置为null
        // inbound=false, outbound=true
        super(pipeline, null, HEAD_NAME, false, true);
        unsafe = pipeline.channel().unsafe();
    }
}
```

注: 从代码实现上看, TailContext和HeadContext在DefaultChannelPipeline中扮演特殊角色, 还是和DefaultChannelPipeline的代码一起看比较好.


































