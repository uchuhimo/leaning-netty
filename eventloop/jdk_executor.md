在NioEventLoop的类继承机构中, 位于最上端的是ExecutorService/AbstractExecutorService,这是属于JDK的代码, 著名的JDK Executor框架.

在研究netty的NioEventLoop之前, 先重温一下JDK Executor的基础代码.

# Executor

大名鼎鼎的JDK Executor, 它的接口定义其实非常简单,就一个execute()方法:

```java
package java.util.concurrent;

public interface Executor {
    void execute(Runnable command);
}
```

传入的Runnable对象代表需要执行的命令, 但是注意Executor接口并没有定义这个命令的执行方式, 因此这个命令有可能被多种方式执行:

- 最简单的同步方式, 直接被调用这个execute()方法的线程执行
- 异步方式, 启动一个新的线程来执行这个命令
- 带线程池的异步方式, 从线程池中取出一个空闲线程来执行这个命令,执行完毕之后归还线程到线程池
- 线程池的实现可能有多种, 比如只有单个工作线程和有多个工作线程
- 支持定时任务的实现, 可以在内部保存要求执行的命令, 等到任务执行条件满足后再执行命令

而这些具体执行方式是交给Executor的实现类来实现, 对于调用者只需要选择调用不同的实现类即可轻松实现在多种方式中间选择和切换, 甚至可以不关心具体到底是用什么实现类, 直接针对Executor接口编程.

这样就轻松的将"任务的执行内容"(比如删除一条记录)和"任务的执行方式"(同步/异步/用线程池/5分钟后再执行)在代码上实现隔离和解耦.

# ExecutorService

Executor接口只定义了一个简单的execute()方法, 而ExecutorService在Executor的基础上做了扩展:

```java
public interface ExecutorService extends Executor {
	void shutdown();
	List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

ExecutorService定义的方法包括:

1. 和关闭相关的方法
	包括shutdown()/shutdownNow()/isShutdown()/isTerminated()/awaitTermination().
2. 提交任务的方法
	包括3个submit()方法, 提供对Future的支持 (注意execute()方法返回void)
3. 执行任务的方法
	包括invokeAll()和invokeAny()

在这里我们跳过shutdown和invoke的方法, 重点看任务执行的方法submit()/execute().

注意这里出现了两个接口用来表示任务或者命令, 除了之前Executor中用到的Runnable之外,ExecutorService还支持Callable接口:

- Runnable接口

    Runnable接口只定义了一个run()方法, 注意它的返回值是void.

    ```java
    public interface Runnable {
        public abstract void run();
    }
    ```

- Callable接口

    Callable接口和Runnable接口最大的不同在于Callable的call()可以返回一个结果, 另外容许定义抛出受查异常:

    ```java
    public interface Callable<V> {
        V call() throws Exception;
    }
	```

ExecutorService中定义的三个submit()方法, 都支持返回结果, 而且是通过Future可以实现异步不阻塞.

1. <T> Future<T> submit(Callable<T> task)
	用Callable接口来提交任务, 返回的类型是Future<T>, 支持泛型和Future
2. <T> Future<T> submit(Runnable task, T result)
	用Runnable接口来提交任务, 由于Runnable接口无法表示返回类型, 因此多增加一个result参数.
3. Future< ? > submit(Runnable task)
	用Runnable接口来提交任务, 无返回值.

继续看AbstractExecutorService的代码,看这三个submit()方法具体是如何实现的.

# AbstractExecutorService

AbstractExecutorService的类定义,申明实现ExecutorService接口:

```java
public abstract class AbstractExecutorService implements ExecutorService {}
```

先看submit(Callable)方法的实现:

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

这里去掉检查task为null的fail fast代码之外, 剩下的三行代码做了三件事情:

1. newTaskFor()方法将Callable类型的task包装为RunnableFuture
2. 调用execute(ftask)方法执行任务
3. 返回任务执行的结果

