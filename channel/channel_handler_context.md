Channel Handler Context
=======================

# 功能

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

# 类继承结构

ChannelHandlerContext继承关系简单, 就一个ChannelHandlerContext接口, 一个AbstractChannelHandlerContext抽象类, 和一个默认实现DefaultChannelHandlerContext.

另外在DefaultChannelPipeline中内嵌有两个特殊的实现类TailContext和HeadContext.

```uml
@startuml

AttributeMap <|-- ChannelHandlerContext
ChannelHandlerContext <|-- AbstractChannelHandlerContext
AbstractChannelHandlerContext <|-- DefaultChannelHandlerContext
AbstractChannelHandlerContext <|-left- TailContext
AbstractChannelHandlerContext <|-right- HeadContext

interface AttributeMap {
}

interface ChannelHandlerContext {
}

abstract class AbstractChannelHandlerContext {
- boolean inbound;
- boolean outbound;
- ChannelPipeline pipeline;
- String name;
- boolean removed;

- AbstractChannelHandlerContext next;
- AbstractChannelHandlerContext prev;
}


class DefaultChannelHandlerContext {
- ChannelHandler handler
}

class TailContext {
- ChannelHandler handler
}

class HeadContext {
- ChannelHandler handler
}

@enduml
```
