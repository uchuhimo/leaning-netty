类ChannelInitializer
===================

# 介绍

类ChannelInitializer是一个特殊的ChannelInboundHandler, 当一个Channel被注册到它的EventLoop时, 提供一个简单的方式用于初始化.

它的实现类通常在Bootstrap.handler(ChannelHandler)/ServerBootstrap.handler(ChannelHandler)和ServerBootstrap.childHandler(ChannelHandler)的上下文中使用, 用于构建channel的ChannelPipeline.

```java
public class MyChannelInitializer extends ChannelInitializer {
    public void initChannel(Channel channel) {
     channel.pipeline().addLast("myHandler", new MyHandler());
    }
}

ServerBootstrap bootstrap = ...;
...
bootstrap.childHandler(new MyChannelInitializer());
...
```

注意这个类被标记为ChannelHandler.Sharable, 因此它的**实现类必须可以安全的被重用**. 

注: 平时实现自己的ChannelInitializer时需要注意这点.

# 实现

## 类定义

ChannelInitializer类定义如下, 继承自ChannelInboundHandlerAdapter:

```java
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {}
```

## 初始化逻辑实现

ChannelInitializer覆盖了方法channelRegistered(), 在这里实现了自己的初始化逻辑:

```java
@Override
@SuppressWarnings("unchecked")
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception { // step2
    initChannel((C) ctx.channel()); //step3
    ctx.pipeline().remove(this);  //step5
    ctx.fireChannelRegistered();  // step6
}
```

### 步骤细节

我们结合前面的实现例子看:

```java
public class MyChannelInitializer extends ChannelInitializer {
    public void initChannel(Channel channel) {  // step3
     channel.pipeline().addLast("myHandler", new MyHandler()); // step4
    }
}

bootstrap.childHandler(new MyChannelInitializer());  //step1
```

整个初始化的步骤和顺序如下:

- step1: "bootstrap.childHandler(new MyChannelInitializer());"

    将MyChannelInitializer作为childHandler加入到bootstrap, 而且只加了这一个Handler.

- step2: bootstrap启动时回调channelRegistered()

	bootstrap在启动后注册所有的handler, 此时会调用它所有的handler的channelRegistered()方法.

	在前面我们只给bootstrap添加了一个handler(即MyChannelInitializer), 因此此时只有这个handler的channelRegistered()方法被调用.

- step3: 调用"initChannel((C) ctx.channel())"

	ChannelInitializer的channelRegistered()方法实现中, 第一步就是调用抽象方法(模板方法)initChannel(), 实际的实现是MyChannelInitializer的initChannel()方法.

- step4: add MyHandler()到pipiline

	这里是MyChannelInitializer真正做事情的地方, 添加它需要的handler到pipline.

    例子中只添加了一个, 实际这里通常是添加多个.

- step5: 将MyChannelInitializer删除

	"ctx.pipeline().remove(this)" 将当前MyChannelInitializer从pipeline中删除,以后的handler就只剩下MyChannelInitializer在step4中添加的handler了.

- step6:

	"ctx.fireChannelRegistered()" 这里重新激活ChannelRegistered事件, 以便可以通知到step4中添加的各个handler.

下面是整个步骤的流程图:

```uml
@startuml

bootstrap -> bootstrap: add MyChannelInitializer to handler
note left: step1

-> bootstrap: start

bootstrap -> bootstrap : create Channel
bootstrap -> bootstrap : register MyChannelInitializer to pipeline
bootstrap ->  bootstrap: fire channelRegistered
note left: step2
ChannelInitializer -> ChannelInitializer : channelRegistered() triggered

group channelRegistered()
ChannelInitializer -> MyChannelInitializer : initChannel()
note left: step3
group initChannel()
MyChannelInitializer -> MyChannelInitializer : add MyHandler to pipeline
note left: step4
end
MyChannelInitializer -->  ChannelInitializer

ChannelInitializer -> ChannelInitializer : remove MyChannelInitializer from pipeline
note left: step5
ChannelInitializer -> ChannelInitializer : fire channelRegistered
note left: step6
end

MyHandler -> MyHandler: channelRegistered() triggered
@enduml
```

## 异常处理

类ChannelInitializer对异常处理的方式就是做两个事情:

1. 打印日志
2. 关闭channel

```java
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
	//1. 打印日志
    logger.warn("Failed to initialize a channel. Closing: " + ctx.channel(), cause);
    try {
        ChannelPipeline pipeline = ctx.pipeline();
        // 在关闭channel之前, 检查channel的pipeline中是否还包含当前ChannelInitializer的handler
        if (pipeline.context(this) != null) {
        	// 如果有, 去除它
            pipeline.remove(this);
        }
    } finally {
    	// 2. 关闭channel
        ctx.close();
    }
}
```

当然子类可以继续覆盖这个实现. 不过一般都够用了.
