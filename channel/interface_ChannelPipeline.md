接口ChannelPipeline
==================

# 接口定义

ChannelPipeline继承了Iterable, Entry<String, ChannelHandler>表明ChannelPipeline内部是保存有一个以String为key, ChannelHandler为vulue的类似map的结构:

```java
public interface ChannelPipeline extends Iterable<Entry<String, ChannelHandler>> {
}
```

# 接口方法

## 管理ChannelHandler

### 增加ChannelHandler

1. addFirst()方法: 将handler添加到pipeline的第一个位置

    ```java
    ChannelPipeline addFirst(String name, ChannelHandler handler);
    ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler);
    ChannelPipeline addFirst(ChannelHandlerInvoker invoker, String name, ChannelHandler handler);
    ```

    参数中, name是当前要插入的handler的名字, 如果设置为null则自动生成.

    注意: **name不容许重复**, 如果添加的时候发现已经存在同样name的handler, 则会抛出IllegalArgumentException.

2. addLast()方法

	和addFirst()方法完全对应


3. addBefore()方法

    ```java
    ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
    ChannelPipeline addBefore(EventExecutorGroup group, String baseName, String name, ChannelHandler handler);
    ChannelPipeline addBefore(ChannelHandlerInvoker invoker, String baseName, String name, ChannelHandler handler);
    ```

	参数中, name和addFirst()中一致, 但是多了一个baseName, 这个是插入的基准handler的名字, 即要插在这个名字的handle前面.

4. addAfter()方法

	和addBefore()方法完全对应.

另外以上方法还有可以参数的方法重载, 无非就是将参数中的一个handler变成ChannelHandler... handlers.

### 删除ChannelHandler

remove()方法查找给定参数的handler, 并从ChannelPipeline中删除它, 然后将被删除的handle返回:

```java
ChannelPipeline remove(ChannelHandler handler);
ChannelHandler remove(String name);
<T extends ChannelHandler> T remove(Class<T> handlerType);
ChannelHandler removeLast();
ChannelHandler removeFirst();
```

注意: 如果删除时发现找不到要删除的目标, 这些remove()方法会抛出NoSuchElementException, 这个要小心处理.

### 替换ChannelHandler

replace()方法用于替换原来的handler为新的handler, 

```java
ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler);
ChannelHandler replace(String oldName, String newName, ChannelHandler newHandler);
<T extends ChannelHandler> T replace(Class<T> oldHandlerType, String newName, ChannelHandler newHandler);
```

注意: 如果替换时发现找不到要替换的目标, replace()方法会抛出NoSuchElementException.

### 获取ChannelHandler

```java
ChannelHandler first();
ChannelHandler last();
ChannelHandler get(String name);
<T extends ChannelHandler> T get(Class<T> handlerType);
```

注意: 如果pipeline为空, 则上面的方法返回null值.

### 获取ChannelHandlerContext

```java
ChannelHandlerContext firstContext();
ChannelHandlerContext lastContext();
ChannelHandlerContext context(ChannelHandler handler);
ChannelHandlerContext context(String name);
ChannelHandlerContext context(Class< ? extends ChannelHandler> handlerType);
```

注意: 如果pipeline为空或者pipeline中找不到要求的hanlder, 则上面的方法返回null值.

### 其他管理handler的方法

```java
List<String> names();
Map<String, ChannelHandler> toMap();
```

## 获取Channel

```java
Channel channel();
```

注意: 如果当前pipeline 还没有绑定到channel, 则你这里返回null.

## fire方法

fire开头的这些方法是给事件通知用的:

```java
// channel被注册到eventloop
ChannelPipeline fireChannelRegistered();

// channel被从eventloop注销
ChannelPipeline fireChannelUnregistered();

// channel被激活, 通常是连接上了
ChannelPipeline fireChannelActive();

// channle被闲置, 通常是被关闭
ChannelPipeline fireChannelInactive();

// channel收到一个Throwable, 比较有意思的是javadoc中明确指出发生的地点是"in one of its inbound operations"
ChannelPipeline fireExceptionCaught(Throwable cause);

// channel收到一个用户自定义事件
ChannelPipeline fireUserEventTriggered(Object event);
```

还有几个方法是给channel读写的:

```java
// channel收到信息
ChannelPipeline fireChannelRead(Object msg);

ChannelPipeline fireChannelReadComplete();
ChannelPipeline fireChannelWritabilityChanged();
```

## I/O操作方法

```java
ChannelFuture bind(SocketAddress localAddress);
ChannelFuture connect(SocketAddress remoteAddress);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
ChannelFuture disconnect();
ChannelFuture close();
ChannelFuture deregister();
```

以及带ChannelPromise的变体版本:

```java
ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
ChannelFuture disconnect(ChannelPromise promise);
ChannelFuture close(ChannelPromise promise);
ChannelFuture deregister(ChannelPromise promise);
```

## I/O读写方法

```java
ChannelPipeline read();
ChannelFuture write(Object msg);
ChannelFuture write(Object msg, ChannelPromise promise);
ChannelPipeline flush();
ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
ChannelFuture writeAndFlush(Object msg);
```














