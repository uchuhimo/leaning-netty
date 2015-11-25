AbstractScheduledEventExecutor在AbstractEventExecutor的基础上提供了对定时任务的支持, 在内部有一个queue用于保存定时任务.

```java
public abstract class AbstractScheduledEventExecutor extends AbstractEventExecutor {

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue;

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue() {
        if (scheduledTaskQueue == null) {
            scheduledTaskQueue = new PriorityQueue<ScheduledFutureTask<?>>();
        }
        return scheduledTaskQueue;
    }
}
```

在queue中存放的是ScheduledFutureTask的实例. 在深入研究AbstractScheduledEventExecutor的代码前, 我们先看一下ScheduledFutureTask.

# ScheduledFutureTask

ScheduledFutureTask继承自PromiseTask, 并实现了ScheduledFuture接口. 而ScheduledFuture接口是从java.util.concurrent.ScheduledFuture继承, 再往上是Delayed接口, 这里有对于定时任务而言最重要的一个方法getDelay().

```java
final class ScheduledFutureTask<V> extends PromiseTask<V> implements ScheduledFuture<V> {
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(delayNanos(), TimeUnit.NANOSECONDS);
    }
}

public interface ScheduledFuture<V> extends Future<V>, java.util.concurrent.ScheduledFuture<V> {
}
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

getDelay()方法用来返回当前这个对象离执行的延迟值, 即还要过多长时间就要执行了(所谓定时任务).

## ScheduledFutureTask.getDelay()方法的实现

ScheduledFutureTask的getDelay()方法中是通过调用delayNanos()方法得到时间结果,然后转换为指定的时间单位. 而delayNanos()方法是通过计算deadlineNanos()和nanoTime()的差值来计算delay时间的.

- deadlineNanos()表示任务计划要执行的时间点
- nanoTime()是取当前时间点

两个时间的差值自然是所谓的delay时间.

```java
public long delayNanos() {
    return Math.max(0, deadlineNanos() - nanoTime());
}
```

1. deadlineNanos()方法

    deadlineNanos()方法返回的deadlineNanos属性在构造函数中被指定, 即在任务构造的时候指定任务计划执行的时间点, 这对于定时任务自然是合理的:

    ```java
    private long deadlineNanos;
    ScheduledFutureTask(EventExecutor executor, Callable<V> callable, long nanoTime, long period) {
        ......
        deadlineNanos = nanoTime;
    }
    public long deadlineNanos() {
        return deadlineNanos;
    }
    ```

2. nanoTime()方法

	nanoTime()方法并没有直接调用System.nanoTime(), 而是先通过常量START_TIME中保存着类装载时的时间值, 然后nanoTime()方法中用每次取当前时间减去这个START_TIME, 返回其差值.

    ```java
    private static final long START_TIME = System.nanoTime();
    static long nanoTime() {
        return System.nanoTime() - START_TIME;
    }
    ```

## ScheduledFutureTask.run()方法的实现

再看ScheduledFutureTask的run()方法, 去掉其他细节处理之后的简化代码如下:

```java
    public void run() {
    	......
        if (setUncancellableInternal()) {
            V result = task.call();
            setSuccessInternal(result);
        }
        ......
    }
```

调用task.call()方法来执行task, 在执行前注明不能cancel, 执行完成后设置执行成功.

看完ScheduledFutureTask之后, 我们回到类AbstractScheduledEventExecutor.

# 添加定时任务

前面看到AbstractScheduledEventExecutor中有一个scheduledTaskQueue用来保存定时任务ScheduledFutureTask, 我们来看这些定时任务时如何添加的.

先看AbstractScheduledEventExecutor的schedule()方法, 在AbstractEventExecutor中这几个schedule()方法都是简单的抛出UnsupportedOperationException, 现在都被实现:

```java
    @Override
    public  ScheduledFuture< ? > schedule(Runnable command, long delay, TimeUnit unit) {
		......
        return schedule(new ScheduledFutureTask<Void>(
                this, toCallable(command), ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
    }
```

这几个schedule()方法都是类似的, 利用输入的参数先生成ScheduledFutureTask对象的实例, 然后调用下面这个schedule(ScheduledFutureTask)方法:

```java
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        scheduledTaskQueue().add(task);
    } else {
        execute(new OneTimeTask() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }

    return task;
}
```

先通过inEventLoop()判断, 如果当前调用的线程是外部线程, 则生成一个OneTimeTask,然后调用execute()方法, 之后event loop的工作线程会执行这个OneTimeTask将task加入到scheduledTaskQueue; 如果调用的线程就是event loop的工作线程, 则直接enqueue.

注: scheduleAtFixedRate()方法我们先掉过.


# 获取定时任务

pollScheduledTask()方法用来获取到期的定时任务:

```java
	protected final Runnable pollScheduledTask() {
        return pollScheduledTask(nanoTime());
    }

    protected final Runnable pollScheduledTask(long nanoTime) {
        assert inEventLoop();

        Queue < ScheduledFutureTask < ? > > scheduledTaskQueue = this.scheduledTaskQueue;
        ScheduledFutureTask< ? > scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
        if (scheduledTask == null) {
        	// 任务队列中没有任务
            return null;
        }

        if (scheduledTask.deadlineNanos() < = nanoTime) {
        	// 取到的任务满足deadline条件, 可以返回
            // 在返回前将任务从queue中删掉, 注意前面取任务用的时peek()方法
            scheduledTaskQueue.remove();
            return scheduledTask;
        }

        // 取到的任务不满足deadline条件,不能返回,只能返回null表示没有满足条件的任务
        return null;
    }
```

hasScheduledTasks()用法用来判断是否有满足条件的定时任务:

```java
    protected final boolean hasScheduledTasks() {
        Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
        ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
        return scheduledTask != null && scheduledTask.deadlineNanos() <= nanoTime();
    }
```

另外当没有定时任务时, 还有一个nextScheduledTaskNano()方法用来查询下一个可以满足条件的任务的定时时间:

```java
    protected final long nextScheduledTaskNano() {
        Queue<ScheduledFutureTask< ? >> scheduledTaskQueue = this.scheduledTaskQueue;
        ScheduledFutureTask< ? > scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
        if (scheduledTask == null) {
        	// 没有任务返回-1
            return -1;
        }
        return Math.max(0, scheduledTask.deadlineNanos() - nanoTime());
    }
```

这几个方法明显是留着给基类调用的了.
