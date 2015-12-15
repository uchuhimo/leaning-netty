类AbstractBootstrap
=================

# 类定义

AbstractBootstrap是Bootstrap的基类, 类定义如下:

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {}
```

类定义中的泛型B要求是AbstractBootstrap的子类, 而泛型C要求是Channel的子类.

# 成员变量

## group

```java
volatile EventLoopGroup group;

public B group(EventLoopGroup group) {
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return (B) this;
}

public EventLoopGroup group() {
    return group;
}
```

注意this.group只能设置一次, 这意味着group()方法只能被调用一次.

## localAddress

localAddress用于绑定本地终端/end, 有多个设值的方法:

```java
private volatile SocketAddress localAddress;

public B localAddress(SocketAddress localAddress) {
    this.localAddress = localAddress;
    return (B) this;
}

public B localAddress(int inetPort) {
    return localAddress(new InetSocketAddress(inetPort));
}

public B localAddress(String inetHost, int inetPort) {
    return localAddress(new InetSocketAddress(inetHost, inetPort));
}

public B localAddress(InetAddress inetHost, int inetPort) {
    return localAddress(new InetSocketAddress(inetHost, inetPort));
}

final SocketAddress localAddress() {
    return localAddress;
}
```

这些重载的localAddress(), 最终都指向了InetSocketAddress的几个构造函数.

## options

options属性是一个LinkedHashMap, option()方法用于设置单个的key/value, 如果value为null则删除该key.

```java
private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();

public <T> B option(ChannelOption<T> option, T value) {
    if (option == null) {
        throw new NullPointerException("option");
    }
    if (value == null) {
        synchronized (options) {
            options.remove(option);
        }
    } else {
        synchronized (options) {
            options.put(option, value);
        }
    }
    return (B) this;
}

final Map<ChannelOption<?>, Object> options() {
    return options;
}
```

## attrs

attrs和options属性类似.

```java
private final Map<AttributeKey<?>, Object> attrs = new LinkedHashMap<AttributeKey<?>, Object>();

public <T> B attr(AttributeKey<T> key, T value) {
    if (key == null) {
        throw new NullPointerException("key");
    }
    if (value == null) {
        synchronized (attrs) {
            attrs.remove(key);
        }
    } else {
        synchronized (attrs) {
            attrs.put(key, value);
        }
    }
    return (B) this;
}

final Map<AttributeKey<?>, Object> attrs() {
    return attrs;
}
```

## handler


```java
private volatile ChannelHandler handler;

public B handler(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
    return (B) this;
}

final ChannelHandler handler() {
    return handler;
}
```

## channelFactory

channelFactory这个属性有点麻烦, 根源在于ChannelFactory这个类:

- 原来在package io.netty.bootstrap中, 明显这个packge名不适合, 所以这个io.netty.bootstrap.ChannelFactory已经被标记为"@Deprecated".

    ```java
    package io.netty.bootstrap;

    @Deprecated
    public interface ChannelFactory<T extends Channel> {
    /**
     * Creates a new channel.
     */
    T newChannel();
    }
    ```

- 现在要求使用packge io.netty.channel下的ChannelFactory类, 这个新类的定义有点奇怪, 是从旧的ChannelFactory下继承:

    ```java
    package io.netty.channel;

    public interface ChannelFactory<T extends Channel> extends io.netty.bootstrap.ChannelFactory<T> {
        /**
         * Creates a new channel.
         */
        @Override
        T newChannel();
    }
    ```

但是现在的情况是内部已经转为使用新类, 对外的接口还是继续保持使用原来的旧类, 因此代码有些混乱:

```java
// 这里的channelFactory的类型定义用的是旧类,因此需要加SuppressWarnings
@SuppressWarnings("deprecation")
private volatile ChannelFactory< ? extends C> channelFactory;

// 返回的channelFactory也是用的旧类, 没的说, 继续SuppressWarnings
@SuppressWarnings("deprecation")
final ChannelFactory< ? extends C> channelFactory() {
    return channelFactory;
}

// 这个方法已经被标志为"unchecked"
@Deprecated
@SuppressWarnings("unchecked")
public B channelFactory(ChannelFactory< ? extends C> channelFactory) {
    if (channelFactory == null) {
        throw new NullPointerException("channelFactory");
    }
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return (B) this;
}


// 这个方法是现在推荐使用的设置channelFactory的方法, 使用新类
// 但是底层的实现还是调用回上面被废弃的channelFactory()方法
@SuppressWarnings({ "unchecked", "deprecation" })
public B channelFactory(io.netty.channel.ChannelFactory< ? extends C> channelFactory) {
    return channelFactory((ChannelFactory<C>) channelFactory);
}
```

# 方法

## validate()

validate()用于检验所有的参数, 实际代码中检查的是group和channelFactory两个参数, 这两个参数必须设置不能为空:

```java
public B validate() {
    if (group == null) {
        throw new IllegalStateException("group not set");
    }
    if (channelFactory == null) {
        throw new IllegalStateException("channel or channelFactory not set");
    }
    return (B) this;
}
```

## register()

register()方法创建一个新的Channel并将它注册到EventLoop, 在执行前会调用validate()方法做前置检查:

```java
public ChannelFuture register() {
    validate();
    return initAndRegister();
}
```

initAndRegister()是关键代码, 细细读一下:

```java
final ChannelFuture initAndRegister() {
	// 创建一个新的Channel
    final Channel channel = channelFactory().newChannel();
    try {
    	// 调用抽象方法, 子类来做初始化
        init(channel);
    } catch (Throwable t) {
    	// 如果出错, 强行关闭这个channel
        channel.unsafe().closeForcibly();
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
    }

	// 创建成功则将这个channel注册到eventloop中
    ChannelFuture regFuture = group().register(channel);
    // 如果出错了
    if (regFuture.cause() != null) {
    	// 判断是否已经注册
        if (channel.isRegistered()) {
        	// channel已经注册的就关闭
            channel.close();
        } else {
        	// 还没有注册的就强行关闭
            channel.unsafe().closeForcibly();
        }
    }

    // 如果代码走到这里而且promise没有失败, 那么是下面两种情况之一:
    // 1) 如果我们尝试了从event loop中注册, 那么现在注册已经完成
    //    现在可以安全的尝试 bind()或者connect(), 因为channel已经注册成功
    // 2) 如果我们尝试了从另外一个线程中注册, 注册请求已经成功添加到event loop的任务队列中等待后续执行
    //    现在可以安全的尝试 bind()或者connect():
    //         因为 bind() 或 connect() 会在安排的注册任务之后执行
    //         而register(), bind(), 和 connect() 都被确认是同一个线程

    return regFuture;
}
```

中途调用的init()方法定义如下, 后面看具体子类代码时再展开.

```java
abstract void init(Channel channel) throws Exception;
```

## bind()

bind()方法有多个重载, 差异只是bind操作所需的InetSocketAddress参数从何而来而已:

1. 从属性this.localAddress来

	这个时候bind()方法无需参数, 直接使用属性this.localAddress, 当前调用之前this.localAddress必须有赋值(通过函数localAddress()):

    ```java
    public ChannelFuture bind() {
        validate();
        SocketAddress localAddress = this.localAddress;
        if (localAddress == null) {
            throw new IllegalStateException("localAddress not set");
        }
        return doBind(localAddress);
    }
    ```

2. 从bind()方法的输入参数中来

	在输入参数中来直接指定localAddress:

    ```java
    public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }
    ```

	另外为了方便, 重载了下面三个方法, 用不同的方式来创建InetSocketAddress而已:

    ```java
    public ChannelFuture bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
    }

    public ChannelFuture bind(String inetHost, int inetPort) {
        return bind(new InetSocketAddress(inetHost, inetPort));
    }

    public ChannelFuture bind(InetAddress inetHost, int inetPort) {
        return bind(new InetSocketAddress(inetHost, inetPort));
    }
    ```

注: 使用带参数的bind()方法, 忽略了localAddress()设定的参数. 而且也没有设置localAddress属性. 这里的差异, 后面留意.

继续看doBind()方法的细节, 这个依然是这个类的核心内容:

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
	// 调用initAndRegister()方法, 先初始化channel,并注册到event loop, 细节前面看过了
    final ChannelFuture regFuture = initAndRegister();
    // 获取注册的channel
    final Channel channel = regFuture.channel();
    // 检查注册是否出错
    if (regFuture.cause() != null) {
    	// 如果出错了直接返回
        return regFuture;
    }

	// 检查注册操作是否完成
    if (regFuture.isDone()) {
    	// 如果完成
        // 在这个点上我们知道注册已经完成并且成功
        // 继续bind操作, 创建一个ChannelPromise
        ChannelPromise promise = channel.newPromise();
        // 调用doBind0()方法来继续真正的bind操作
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // 通常这个时候注册的future应该都已经完成,但是万一没有, 我们也需要处理
        // 为这个channel创建一个PendingRegistrationPromise
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        // 然后给注册的future添加一个listener, 在operationComplete()回调时
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                // 检查是否出错
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an 在event loop上注册失败, 因此直接让ChannelPromise失败, 避免一旦我们试图访问这个channel的eventloop导致IllegalStateException
                    promise.setFailure(cause);
                } else {
                	// 注册已经成功, 因此设置正确的executor以便使用
                    // 注: 这里已经以前有过一个bug, 有issue记录
                    // See https://github.com/netty/netty/issues/2586
                    promise.executor = channel.eventLoop();
                }
                // 调用doBind0()方法来继续真正的bind操作
                doBind0(regFuture, channel, localAddress, promise);
            }
        });
        return promise;
    }
}
```

关键的doBind0()方法:

```java
private static void doBind0(final ChannelFuture regFuture, final Channel channel, final SocketAddress localAddress, final ChannelPromise promise) {

    // 这个方法在channelRegistered()方法触发前被调用.
    // 让handleer有机会在它的channelRegistered()实现中构建pipeline
    // 给channel的event loop增加一个一次性任务
    channel.eventLoop().execute(new OneTimeTask() {
        @Override
        public void run() {
        	// 检查注册是否成功
            if (regFuture.isSuccess()) {
            	// 如果成功则绑定localAddress到channel
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
            	// 如果不成功则设置错误到promise
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```











