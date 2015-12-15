类ServerBootstrap
=================

类ServerBootstrap用于帮助服务器端引导ServerChannel.

ServerBootstrap除了处理ServerChannel外, 还需要处理从ServerChannel下创建的Channel.Netty中称这两个关系为parent和child.

# 类定义

```java
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {}
```

# 类属性

## childGroup

childGroup属性用于指定处理客户端连接的EventLoopGroup, 设置的方式有两种:

1. group(parentGroup, childGroup)方法用于单独设置parentGroup, childGroup, 分别用于处理ServerChannel和Channel.
2. group(group)方法设置parentGroup, childGroup为使用同一个EventLoopGroup. 注意这个方法覆盖了基类的方法.

```java
private volatile EventLoopGroup childGroup;

@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
public EventLoopGroup childGroup() {
    return childGroup;
}
```

## childOptions/childAttrs/childHandler

这三个属性和parent的基本对应, 设值方法和检验都是一模一样的:

```java
private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();

private final Map<AttributeKey<?>, Object> childAttrs = new LinkedHashMap<AttributeKey<?>, Object>();

private volatile ChannelHandler childHandler;
```

# 类方法

## init(channel)方法

ServerBootstrap的init(channel)方法相比Bootstrap的要复杂一些, 除了设置options/attrs/handler到channel外, 还需要为child设置childGroup, childHandler, childOptions, childAttrs:

```java
@Override
void init(Channel channel) throws Exception {
    final Map<ChannelOption< ? >, Object> options = options();
    synchronized (options) {
        channel.config().setOptions(options);
    }

    final Map<AttributeKey< ? >, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey< ? >, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption< ? >, Object>[] currentChildOptions;
    final Entry<AttributeKey< ? >, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            pipeline.addLast(new ServerBootstrapAcceptor(
                    currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
        }
    });
}
```

ServerBootstrapAcceptor的实现, 主要看channelRead()方法:

```java
private static class ServerBootstrapAcceptor extends ChannelHandlerAdapter {
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
    	// 获取child channel
        final Channel child = (Channel) msg;
		// 设置childHandler到child channel
        child.pipeline().addLast(childHandler);
		// 设置childOptions到child channel
        for (Entry<ChannelOption< ? >, Object> e: childOptions) {
            try {
                if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                    logger.warn("Unknown channel option: " + e);
                }
            } catch (Throwable t) {
                logger.warn("Failed to set a channel option: " + child, t);
            }
        }

		// 设置childAttrs到child channel
        for (Entry<AttributeKey< ? >, Object> e: childAttrs) {
            child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }

		// 将child channel注册到childGroup
        try {
            childGroup.register(child).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (!future.isSuccess()) {
                        forceClose(child, future.cause());
                    }
                }
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
}
```

