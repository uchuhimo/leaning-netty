# EventExecutorGroup

## 接口定义

先看EventExecutorGroup接口的定义,注意这个类在packag "io.netty.util.concurrent"下:

```java
package io.netty.util.concurrent

public interface EventExecutorGroup extends ScheduledExecutorService, AutoCloseable {}
```

EventExecutorGroup继承自jdk java.util.concurrent包下的ScheduledExecutorService, 这意味着EventExecutorGroup本身就是一个标准的jdk executor, 提供定时任务的支持.

## 增强shutdown()方法

EventExecutorGroup中针对ExecutorService的shutdown()提供了增强，提供了优雅关闭的方法shutdownGracefully().

从代码上看,具体做法是EventExecutorGroup覆盖了executor的标准方法shutdown()和shutdownNow(), 加上了@Deprecated标记. 在javadoc中要求使用者不要调用这两个方法, 改为调用EventExecutorGroup接口中定义的shutdownGracefully()方法. 另外增加了一个isShuttingDown()方法来检查是否正在关闭:

```java
    /**
     * 当且仅当这个EventExecutorGroup管理的所有EventExecutor正在被shutdownGracefully()方法优雅关闭或被已经被关闭.
     */
	boolean isShuttingDown();

    /**
     * 等同于以合理的默认参数调用shutdownGracefully(quietPeriod, timeout, unit)方法
     */
    Future<?> shutdownGracefully();

    /**
     * 发信号给这个executor,告之调用者希望这个executor关闭. 一旦这个方法被调用, isShuttingDown()方法就将开始返回true, 然后这个executor准备关闭自己. 和shutdown()不同, 优雅关闭保证在关闭之前,在静默时间(the quiet period, 通常是几秒钟)内没有任务提交. 如果在静默时间内有任务提交, 这个任务将被接受, 而静默时间将重头开始.
     *
     * @param quietPeriod 上面文档中描述的静默时间
     * @param timeout     等待的最大超时时间, 直到executor被关闭, 无论在静默时间内是否有任务被提交
     * @param unit        静默时间和超时时间的单位
     *
     * @return the {@link #terminationFuture()}
     */
    Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);

    Future<?> terminationFuture();

    @Deprecated			//禁用,用shutdownGracefully()代替
    void shutdown();

    @Deprecated			//禁用,用shutdownGracefully()代替
    List<Runnable> shutdownNow();
```

这里还有增加了一个terminationFuture()方法, 获取一个Future, 当这个EventExecutorGroup管理的所有的EventExecutors都被终止时可以得到通知.

## 管理EventExecutor

然后是最重要的两个方法next()和children():

```java
    /**
     * 返回这个EventExecutorGroup管理的一个EventExecutor
     */
    EventExecutor next();

    /**
     * 返回这个EventExecutorGroup管理的所有EventExecutor的不可变集合
     */
    <E extends EventExecutor> Set<E> children();
```

这里可以看到EventExecutorGroup和EventExecutor的关系是:

- **EventExecutorGroup管理了(从对象关系上说是"聚合"了)多个EventExecutor**
- next()方法返回其中的一个EventExecutor
- children()返回所有的EventExecutor

## 覆盖submit()和schedule()方法

EventExecutorGroup中的submit()方法, 对照了一下, 和ExecutorService接口中完全相同的.

```java
    @Override
    Future<?> submit(Runnable task);

    @Override
    <T> Future<T> submit(Runnable task, T result);

    @Override
    <T> Future<T> submit(Callable<T> task);
```

但是仔细看,会有个非常不起眼的小地方,submit()方法方法的返回对象不同,虽然都是Future:

- EventExecutorGroup中submit()方法返回的 Future 是 "io.netty.util.concurrent.Future"
- ExecutorService中submit()方法返回的 Future 是 "java.util.concurrent.Future"

下面是io.netty.util.concurrent.Future的代码, 继承自java.util.concurrent.Future,然后增加了一些特有方法:

```java
public interface Future<V> extends java.util.concurrent.Future<V> {
	......
}
```

类似的, schedule()方法也是同样覆盖了接口ScheduledExecutorService中的对应方法, 依然是修改了返回的
对象类型, 用io.netty.util.concurrent.ScheduledFuture替代了java.util.concurrent.ScheduledFuture:

```java
public interface ScheduledFuture<V> extends Future<V>, java.util.concurrent.ScheduledFuture<V> {
	......
}
```

# EventExecutor

## 接口定义

EventExecutor的设计比较费解, 居然是从EventExecutorGroup下继承:

```java
public interface EventExecutor extends EventExecutorGroup {
}
```

考虑到EventExecutorGroup的设计中, EventExecutorGroup内部是聚合/管理了多个EventExecutor. 然后现在EventExecutor再继承EventExecutorGroup一把, 我有些凌乱了......

## 和EventExecutorGroup的关系

在EventExecutor中覆盖了EventExecutorGroup的next()/children(), 在我看来这是netty在努力的收拾凌乱的局面:

```java
@Override
EventExecutor next();

@Override
<E extends EventExecutor> Set<E> children();

EventExecutorGroup parent();
```

- next()方法: EventExecutorGroup中next()方法用来返回管理的EventExecutor中的其中一个, 到了EventExecutor中, 这里应该不会继续再管理其他EventExecutor了, 所以next()方法被覆盖为仅仅返回当前EventExecutor本身的引用
- children()方法: EventExecutorGroup中children()方法用来返回管理的所有的EventExecutor, 到了EventExecutor中,返回的集合就只能包含自身一个引用.
- parent()方法: 新增加的方法,返回当前EventExecutor所属的EventExecutorGroup

## inEventLoop()方法

inEventLoop()方法用于查询某个线程是否在EventExecutor所管理的线程中.

```java
    /**
     * 等同于调用inEventLoop(Thread.currentThread())
     */
    boolean inEventLoop();

    /**
     * 当给定的线程在event loop中返回true, 否则返回false
     */
    boolean inEventLoop(Thread thread);
```

## unwrap()方法

unwrap()方法用于返回一个非WrappedEventExecutor的EventExecutor.

- 对于WrappedEventExecutor的实现类,需要返回底层包裹的EventExecutor, 而且要保证返回的EventExecutor也不是WrappedEventExecutor (这意味着可能需要调用多次unwrap())
- 对于非WrappedEventExecutor的实现类, 只要简单返回自身实例就好了
- 强调, 一定不能返回null

```java
    EventExecutor unwrap();
```

## 创建Promise和Future

```java
    <V> Promise<V> newPromise();

    <V> ProgressivePromise<V> newProgressivePromise();

    <V> Future<V> newSucceededFuture(V result);

    <V> Future<V> newFailedFuture(Throwable cause);
```

这个细节后面研究Promise和Future时再先看, 先跳过.
