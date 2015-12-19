类DefaultChannelPipeline
========================

# 类定义

DefaultChannelPipeline实现了ChannelPipeline方法, 注意这个类是包级私有, 而且这个是final类不能继承.

```java
final class DefaultChannelPipeline implements ChannelPipeline {
}
```

继续看具体的代码实现, 代码量有点大.

# 类属性和初始化

## 类属性

类DefaultChannelPipeline有以下属性:

```java
final AbstractChannel channel;

final AbstractChannelHandlerContext head;
final AbstractChannelHandlerContext tail;

private final boolean touch = ResourceLeakDetector.getLevel() != ResourceLeakDetector.Level.DISABLED;

private Map<EventExecutorGroup, ChannelHandlerInvoker> childInvokers;
```

除了childInvokers外其他属性都是final.

## 构造函数

构造函数只有一个:

```java
DefaultChannelPipeline(AbstractChannel channel) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    this.channel = channel;

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

这里channel/tail/head都被初始化:

- channel 由参数指定, 不容许为空, final属性表明设置之后不能更改. 这个和ChannelPipeline的设计是一致的: ChannelPipeline 和 channel之间是一一对应的
- tail/head 分别new出了具体的对象, 后面具体看实现代码
- 初始化时head的next指向了tail, tail的prev指向了head, 完成链表初始化

# HandlerContext

先看看HandlerContext的代码, 



