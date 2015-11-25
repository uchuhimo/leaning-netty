SingleThreadEventLoop的类定义

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {}
```

# register()方法

register()方法最终调用到unsafe做register.

```java
@Override
public ChannelFuture register(Channel channel) {
    return register(channel, new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    if (promise == null) {
        throw new NullPointerException("promise");
    }

    channel.unsafe().register(this, promise);
    return promise;
}
```

# 标志接口NonWakeupRunnable

在SingleThreadEventLoop中,增加了一个标志接口NonWakeupRunnable, 然后wakesUpForTask()被覆盖为检测是否task是否是NonWakeupRunnable.

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {
    protected boolean wakesUpForTask(Runnable task) {
        return !(task instanceof NonWakeupRunnable);
    }

    interface NonWakeupRunnable extends Runnable {}
}
```