类ChannelHandlerAppender
=======================

# 类的功能

类ChannelHandlerAppender是一个特殊的ChannelHandler, 如名字中的Appender所示, 这是一个包含多个特定  ChannelHandler 的Appender. 而且这些ChannelHandler会在ChannelHandlerAppender的右边(在pipiline中的位置).

默认情况下, 一旦这些特定的ChannelHandler被添加, 它会从ChannelPipeline中将自身删除. 当然, 也可以通过在构造函数中设置参数selfRemoval为false来保留它.

# 类的定义

类ChannelHandlerAppender的定义, 简单继承自ChannelInboundHandlerAdapter:

```java
public class ChannelHandlerAppender extends ChannelInboundHandlerAdapter {}
```

ChannelHandlerAppender的子类实现有一下, 基本都是给codec用的, 具体继承结构如图:

```uml
@startuml

ChannelInboundHandlerAdapter <|-- ChannelHandlerAppender

ChannelHandlerAppender <|-- HttpServerCodec
ChannelHandlerAppender <|-- HttpClientCodec
ChannelHandlerAppender <|-- SpdyHttpCodec
ChannelHandlerAppender <|-- BinaryMemcacheServerCodec
ChannelHandlerAppender <|-- BinaryMemcacheClientCodec

class ChannelHandlerAppender {
- boolean selfRemoval
- List<Entry> handlers
- boolean added

+ add(handler)
}

@enduml
```

# 类的实现

## 属性

ChannelHandlerAppender有三个属性:

```java
private final boolean selfRemoval;
private final List<Entry> handlers = new ArrayList<Entry>();
private boolean added;
```

1. selfRemoval: 标记在将append的各个handler添加到pipeline之后, 是否要将自身删除, 默认行为是true/删除, 可以在构造函数中设置
2. handlers: 这个就是内部保存的要append的各个handler了, 可以通过add()方法添加
3. added: 标记append的各个handler是否已经被添加到pipeline中了

	上面的add()方法中, 会检查added属性, 如果已经添加了则抛异常.

相关的代码示意:

```java
public ChannelHandlerAppender(boolean selfRemoval, ChannelHandler... handlers) {
    this.selfRemoval = selfRemoval;	// 构造函数中指定
    add(handlers);
}
protected final ChannelHandlerAppender add(String name, ChannelHandler handler) {
    if (added) {	// 检查added属性
        throw new IllegalStateException("added to the pipeline already");
    }

    handlers.add(new Entry(name, handler));
    return this;
}

@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    added = true;	//设置added为true

    DefaultChannelPipeline pipeline = (DefaultChannelPipeline) dctx.pipeline();
    try {

    } finally {
        if (selfRemoval) {	// 检查selfRemoval属性看是否要删除自身
            pipeline.remove(this);
        }
    }
}
```

## addpend的代码实现

handlerAdded()方法中是这个类真正的逻辑实现:

```java
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    added = true;

    AbstractChannelHandlerContext dctx = (AbstractChannelHandlerContext) ctx;
    DefaultChannelPipeline pipeline = (DefaultChannelPipeline) dctx.pipeline();
    String name = dctx.name();
    try {
        for (Entry e: handlers) { // 循环处理每个append的handler
            String oldName = name; // 第一个oldname是dctx.name(), 也就是ChannelHandlerAppender的name, 以后oldname是上一次的name
            if (e.name == null) {
                name = pipeline.generateName(e.handler);
            } else {
                name = e.name;
            }

			// 添加这个handler到上一个hanler(由oldName指定)之后
            pipeline.addAfter(dctx.invoker, oldName, name, e.handler);
        }
    } finally {
        if (selfRemoval) { // 检查是否要删除自身
            pipeline.remove(this);
        }
    }
}
```

备注: 这个类的具体功能, 在后面细读codec实现时再结合使用场景一起看.
