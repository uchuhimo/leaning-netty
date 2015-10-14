AbstractChannel顾名思义, 从Channel的继承结构上也可以看到, 在排除ServerChannel/ServerSocketChannel/SocketChannel三个接口定义之后, Channen的类层次就是从Channel -> AbstractChannel -> AbstractNioChannel 一路继承下去.

![nio channels](./image/nio_channels.png)

下面详细看代码实现.

# 类定义和外围实现

## 类定义

```java
package io.netty.channel

public abstract class AbstractChannel extends DefaultAttributeMap implements Channel {}
```

## 外围接口实现

在Channel接口定义中申明的需要继承的两个接口AttributeMap/Comparable<Channel>

```java
public interface Channel extends AttributeMap, Comparable<Channel> {
```

通过继承DefaultAttributeMap实现了对接口AttributeMap的实现. Comparable接口的实现则是通过实现compareTo()方法来实现, compareTo()方法的代码:

```java
    @Override
    public final int compareTo(Channel o) {
        if (this == o) {
            return 0;
        }

        return id().compareTo(o.id());
    }
```

# Channel接口实现

AbstractChannel实现了Channel接口定义的部分方法,作为所有Channel子类的基本实现.

下面是AbstractChannel中定义的几个基本属性和构造函数, 注意这几个基本属性都是final:

```java
    private final Channel parent;
    private final ChannelId id;
    private final Unsafe unsafe;
    private final DefaultChannelPipeline pipeline;

    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = DefaultChannelId.newInstance();
        unsafe = newUnsafe();
        pipeline = new DefaultChannelPipeline(this);
    }

    protected AbstractChannel(Channel parent, ChannelId id) {
        this.parent = parent;
        this.id = id;
        unsafe = newUnsafe();
        pipeline = new DefaultChannelPipeline(this);
    }
```

## id

每个Channel的实例都应该有一个唯一的id,这个id一旦设置就不能被修改. 默认通过DefaultChannelId.newInstance()方法来获取:

```java
private final ChannelId id;

protected AbstractChannel(Channel parent) {
    id = DefaultChannelId.newInstance();
    ......
}

protected AbstractChannel(Channel parent, ChannelId id) {
    this.id = id;
    ......
}

@Override
public final ChannelId id() {
    return id;
}
```

## parent

每个Channel的parent在构造函数中指定:

```java
private final Channel parent;

protected AbstractChannel(Channel parent) {
    this.parent = parent;
    ......
}

@Override
public Channel parent() {
    return parent;
}
```

###　pipeline

pipeline的初始化在构造函数中，目前看时直接固定为使用DefaultChannelPipeline的实现：

```java
private final DefaultChannelPipeline pipeline;

protected AbstractChannel(Channel parent, ChannelId id) {
    ......
    pipeline = new DefaultChannelPipeline(this);
}

public ChannelPipeline pipeline() {
    return pipeline;
}
```

> 比较奇怪的是这里pipeline的类型定义是"DefaultChannelPipeline"，　而不是"ChannelPipeline"．这个方式按说不正确，呵呵，验证过将类型修改为ChannelPipeline代码编译和测试都OK．

> **已经提交了一个pull request 给netty，看看他们能否接受**
> 更新: netty接受了这个commit, 现在pipeline的类型已经修改为ChannelPipeline了.

在AbstractChannel的实现中, 有很多和IO相关的方法是直接delagate给pipeline的, 例如bind()方法:

```java
public ChannelFuture bind(SocketAddress localAddress) {
    return pipeline.bind(localAddress);
}
```

类似delagate给pipeline的方法包括:

- bind()
- connect()
- disconnect()
- close()
- deregister()
- flush()
- read()
- write()
- writeAndFlush()

### unsafe

unsafe的初始化也是在构造函数中，通过定义模板方法newUnsafe()让子类来给出具体的Unsafe的实现：

```java
private final Unsafe unsafe;

protected AbstractChannel(Channel parent, ChannelId id) {
    ......
    unsafe = newUnsafe();
}

protected abstract AbstractUnsafe newUnsafe();

public Unsafe unsafe() {
    return unsafe;
}
```

在AbstractChannel的实现中, 有部分方法是直接delagate给unsafe的, 例如isWritable()方法:

```java
public boolean isWritable() {
    ChannelOutboundBuffer buf = unsafe.outboundBuffer();
    return buf != null && buf.isWritable();
}
```

类似delagate给unsafe的方法包括:

- isWritable()
- bytesBeforeUnwritable()
- bytesBeforeWritable()
- localAddress()
- remoteAddress()