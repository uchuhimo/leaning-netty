类Bootstrap
=================

类Bootstrap用于帮助客户端引导Channel.

# 类定义

```java
public class Bootstrap extends AbstractBootstrap<Bootstrap, Channel> {}
```

# 类属性

## resolver

resolver默认设置为DefaultAddressResolverGroup.INSTANCE, 可以通过resolver()方法来赋值:

```java
private static final AddressResolverGroup< ? > DEFAULT_RESOLVER = DefaultAddressResolverGroup.INSTANCE;
private volatile AddressResolverGroup<SocketAddress> resolver = (AddressResolverGroup<SocketAddress>) DEFAULT_RESOLVER;

public Bootstrap resolver(AddressResolverGroup< ? > resolver) {
    if (resolver == null) {
        throw new NullPointerException("resolver");
    }
    this.resolver = (AddressResolverGroup<SocketAddress>) resolver;
    return this;
}
```

## remoteAddress

remoteAddress可以通过remoteAddress()方法赋值, 有多个装载:

```java
private volatile SocketAddress remoteAddress;

public Bootstrap remoteAddress(SocketAddress remoteAddress) {
    this.remoteAddress = remoteAddress;
    return this;
}
public Bootstrap remoteAddress(String inetHost, int inetPort) {
    remoteAddress = InetSocketAddress.createUnresolved(inetHost, inetPort);
    return this;
}
public Bootstrap remoteAddress(InetAddress inetHost, int inetPort) {
    remoteAddress = new InetSocketAddress(inetHost, inetPort);
    return this;
}
```

# 类方法

## validate()方法

重写了validate()方法, 在调用AbstractBootstrap的validate()方法(检查group和channelFactory)外, 增加了对handler的检查:

```java
@Override
public Bootstrap validate() {
    super.validate();
    if (handler() == null) {
        throw new IllegalStateException("handler not set");
    }
    return this;
}
```

## connect()方法

有多个connect()方法重载, 功能都是一样, 拿到输入的remoteAddress然后调用doResolveAndConnect()方法:

```java
public ChannelFuture connect() {
	// 先做前置检查
    validate();
    // 检查remoteAddress, 确认已经赋值
    SocketAddress remoteAddress = this.remoteAddress;
    if (remoteAddress == null) {
        throw new IllegalStateException("remoteAddress not set");
    }

	// 调用doResolveAndConnect()方法
    return doResolveAndConnect(remoteAddress, localAddress());
}
```

看doResolveAndConnect()方法:

```java
private ChannelFuture doResolveAndConnect(SocketAddress remoteAddress, final SocketAddress localAddress) {
	// 先初始化channel并注册到event loop
    final ChannelFuture regFuture = initAndRegister();
    if (regFuture.cause() != null) {
    	// 如果注册失败则退出
        return regFuture;
    }

    final Channel channel = regFuture.channel();
    final EventLoop eventLoop = channel.eventLoop();
    final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);

    if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
        // Resolver 不知道该怎么处理给定的远程地址, 或者已经解析
        return doConnect(remoteAddress, localAddress, regFuture, channel.newPromise());
    }

	// 开始解析远程地址
    final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);
    final Throwable resolveFailureCause = resolveFuture.cause();

    if (resolveFailureCause != null) {
        // 如果地址解析失败, 则立即失败
        channel.close();
        return channel.newFailedFuture(resolveFailureCause);
    }

    if (resolveFuture.isDone()) {
    	// 理解成功的解析了远程地址, 开始做连接
        return doConnect(resolveFuture.getNow(), localAddress, regFuture, channel.newPromise());
    }

    // 地址解析还没有完成, 只能等待完成后在做connectio, 增加一个promise来操作
    final ChannelPromise connectPromise = channel.newPromise();
    resolveFuture.addListener(new FutureListener<SocketAddress>() {
        @Override
        public void operationComplete(Future<SocketAddress> future) throws Exception {
            if (future.cause() != null) {
                channel.close();
                connectPromise.setFailure(future.cause());
            } else {
                doConnect(future.getNow(), localAddress, regFuture, connectPromise);
            }
        }
    });

    return connectPromise;
}
```

doConnect()方法中才是真正的开始处理连接操作, 但是还是需要检查注册操作是否完成:

```java
private static ChannelFuture doConnect(
        final SocketAddress remoteAddress, final SocketAddress localAddress,
        final ChannelFuture regFuture, final ChannelPromise connectPromise) {
    // 判断一下前面的注册操作是否已经完成
    // 因为注册操作是异步操作, 前面只是返回一个feature, 代码执行到这里时, 可能完成, 也可能还在进行中
    if (regFuture.isDone()) {
    	// 如果注册已经完成, 可以执行连接了
        doConnect0(remoteAddress, localAddress, regFuture, connectPromise);
    } else {
    	// 如果注册还在进行中, 增加一个ChannelFutureListener, 等操作完成之后再在回调方法中执行连接操作
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                doConnect0(remoteAddress, localAddress, regFuture, connectPromise);
            }
        });
    }

    return connectPromise;
}
```

异步操作就是这点比较麻烦, 总是需要一个一个future的做判断/处理, 如果没有完成还的加promise/future来依靠回调函数继续工作处理流程.

终于到了最后的doConnect0()方法, 总算可以真的连接了:

```java
private static void doConnect0(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelFuture regFuture,
        final ChannelPromise connectPromise) {
    // 这个方法在channelRegistered()方法被触发前调用.
    // 给我们的handler一个在它的channelRegistered()实现中构建pipeline的机会
    final Channel channel = connectPromise.channel();
    // 取当前channel的eventlopp, 执行一个一次性任务
    channel.eventLoop().execute(new OneTimeTask() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
            	// 如果注册成功
                if (localAddress == null) {
                    channel.connect(remoteAddress, connectPromise);
                } else {
                    channel.connect(remoteAddress, localAddress, connectPromise);
                }
                connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                connectPromise.setFailure(regFuture.cause());
            }
        }
    });
}
```

## init(channel)方法

前面看基类AbstractBootstrap时看到过, 这个init()方法是一个模板方法, 需要子类做具体实现.

看看Bootstrap是怎么做channel初始化的:

```java
@Override
@SuppressWarnings("unchecked")
void init(Channel channel) throws Exception {
	// 取channel的ChannelPipeline
    ChannelPipeline p = channel.pipeline();
    // 增加当前Bootstrap的handle到ChannelPipeline中
    p.addLast(handler());

	// 取当前Bootstrap设置的options, 逐个设置到channel中
    final Map<ChannelOption< ? >, Object> options = options();
    synchronized (options) {
        for (Entry<ChannelOption< ? >, Object> e: options.entrySet()) {
            try {
                if (!channel.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                    logger.warn("Unknown channel option: " + e);
                }
            } catch (Throwable t) {
                logger.warn("Failed to set a channel option: " + channel, t);
            }
        }
    }

	// 同样取当前Bootstrap的attrs, 逐个设置到channel中
    final Map<AttributeKey< ? >, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey< ? >, Object> e: attrs.entrySet()) {
            channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }
    }
}
```

总结上在init()方法中, Bootstrap只做了一个事情: **将Bootstrap中保存的信息(handle/options/attrs)设置到新创建的channel**.




