bootstrap
=========

# Bootstrap的作用

Bootstrap的作用可以参考AbstractBootstrap的javadoc:

> AbstractBootstrap is a helper class that makes it easy to **bootstrap a Channel**.

也就是说, Bootstrap存在的意义就是为了方便的"引导"Channel.

在netty中, 存在两种Channel, 因此也对应有两种Bootstrap:

1. ServerChannel, 使用类ServerBootstrap来引导
2. Channel, 使用类Bootstrap来引导

# Bootstrap的继承结构

在netty的代码中, 类ServerBootstrap和类Bootstrap都继承自基类AbstractBootstrap:

```uml
@startuml

class AbstractBootstrap{
- EventLoopGroup group
- ChannelFactory channelFactory
- SocketAddress localAddress
- Map options
- Map attrs
- ChannelHandler handler
+ ChannelFuture register()
+ ChannelFuture bind()
{abstract} void init(Channel channel)
}
class ServerBootstrap{
- Map childOptions
- Map childAttrs
- EventLoopGroup childGroup
- ChannelHandler childHandler
}
class Bootstrap{
- AddressResolverGroup<SocketAddress> resolver
- SocketAddress remoteAddress
+ ChannelFuture connect()
}

AbstractBootstrap  		<|-- 	Bootstrap
AbstractBootstrap		<|--	ServerBootstrap

@enduml
```

