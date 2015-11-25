SingleThreadEventExecutor的类定义:

```java
public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor {
}
```

在SingleThreadEventExecutor中,存在两个queue:

1. taskQueue: 这个是可以执行的任务队列, SingleThreadEventExecutor自己实现
2. scheduledTaskQueue: 定时任务队列, 从AbstractScheduledEventExecutor中继承

# taskQueue的实现

## taskQueue的定义和初始化

taskQueue构造函数中通过调用newTaskQueue()方法来初始化, 默认是用LinkedBlockingQueue:

```java
private final Queue<Runnable> taskQueue;

protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
    ......
    taskQueue = newTaskQueue();
}
protected Queue<Runnable> newTaskQueue() {
    return new LinkedBlockingQueue<Runnable>();
}
```

注意newTaskQueue()方法是protected, 依然给子类做覆盖预留了空间.

## pollTask()方法

pollTask()方法用于从taskQueue中获取任务:

```java
protected Runnable pollTask() {
    assert inEventLoop();
    for (;;) {
        Runnable task = taskQueue.poll();
        if (task == WAKEUP_TASK) {
            continue;
        }
        return task;
    }
}
```

WAKEUP_TASK先忽略, 后面再细看.

## takeTask()方法

takeTask()方法方法用来获取需要任务, 注意这个时候任务有两个来源: taskQueue和scheduledTask. 因此取任务时需要考虑很多情况:

```java
protected Runnable takeTask() {
	// 断言当前调用线程是event loop的工作线程,自我保护之一
    assert inEventLoop();
    // 检查当前taskQueue, 如果不是BlockingQueue抛异常退出
    if (!(taskQueue instanceof BlockingQueue)) {
        throw new UnsupportedOperationException();
    }

    BlockingQueue<Runnable> taskQueue = (BlockingQueue<Runnable>) this.taskQueue;
    for (;;) {
    	// 从scheduledTaskQueue取一个scheduledTask, 注意方法是peek, 另外这个task有可能还没有到执行时间
        ScheduledFutureTask< ? > scheduledTask = peekScheduledTask();
        if (scheduledTask == null) {
        	// scheduledTask为null, 说明当前scheduledTaskQueue中没有定时任务
            // 这个时候不用考虑定时任务, 直接从taskQueue中拿任务即可
            Runnable task = null;
            try {
            	// 从taskQueue取任务, 如果没有任务会被阻塞, 直到取到任务,或者被Interrupted
                task = taskQueue.take();
                // 检查如果是WAKEUP_TASK, 则需要跳过
                if (task == WAKEUP_TASK) {
                    task = null;
                }
            } catch (InterruptedException e) {
                // Ignore
            }

            // 如果从taskQueue中拿任务成功,则返回, 如果没有任务或者没有成功,则返回的是null
            return task;
        } else {
        	// scheduledTask不为null, 说明当前scheduledTaskQueue中有定时任务
            // 但是这个定时任务可能还没有到执行时间, 因此需要检查delayNanos
            long delayNanos = scheduledTask.delayNanos();
            Runnable task = null;
            if (delayNanos > 0) {
            	// delayNanos > 0 说明这个定时任务没有到执行时间, 所以这个定时任务是不能用的了
                // 那就需要从taskQueue里面取任务了
                try {
                	// 从taskQueue取任务, 如果没有任务则最多等待delayNanos时间, 注意这里线程会被阻塞
                    // 如果delayNanos时间内有任务加入到taskQueue, 则可以实时取到这个任务
                    // 线程结束阻塞继续执行, 这个task就可以返回了
                    // 如果delayNanos时间内一直没有任务, 则timeout后线程结束阻塞,poll()方法返回null
                    task = taskQueue.poll(delayNanos, TimeUnit.NANOSECONDS);
                } catch (InterruptedException e) {
                	// 还有一种可能就是delayNanos时间内被waken up
                    // 比如有新的scheduledTask加入, delay时间小于前面的delayNanos
                    // 因此不能等待delayNanos timeout,需要提前结束阻塞
                    // Waken up.
                    return null;
                }
            }
            if (task == null) {
                // We need to fetch the scheduled tasks now as otherwise there may be a chance that
                // scheduled tasks are never executed if there is always one task in the taskQueue.
                // This is for example true for the read task of OIO Transport
                // See https://github.com/netty/netty/issues/1614
                // 继续尝试从scheduledTaskQueue取满足条件的task到taskQueue中
                fetchFromScheduledTaskQueue();
                // 然后再从taskQueue获取试试
                task = taskQueue.poll();
            }

            if (task != null) {
            	// 只有task不为空时才返回并退出, 否则前面的for循环就一直
                return task;
            }
        }
    }
}
```

# 任务的执行

execute()方法被覆盖为:

```java
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
    	// 如果当前线程是event loop线程,则将task加入到taskQueue中
        addTask(task);
    } else {
    	// 如果是外部线程
        // 调用startExecution()方法先
        startExecution();
        // 再将task加入到taskQueue中
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

实际上execute()方法只是将任务加入到taskQueue中, 任务真正的执行还不在这里.

# wake up机制

在SingleThreadEventExecutor中, 设计了一套wake up机制, 包含以下内容:

## WAKEUP_TASK任务

定了一个什么都不做的task, 这是一个特殊任务,用来做wake up的标记

```java
private static final Runnable WAKEUP_TASK = new Runnable() {
    @Override
    public void run() {
        // Do nothing.
    }
};
```

比如在pollTask()方法中,遇到WAKEUP_TASK就会自动跳过

```java
    protected Runnable pollTask() {
    assert inEventLoop();
    for (;;) {
        Runnable task = taskQueue.poll();
        if (task == WAKEUP_TASK) {
            continue;
        }
        return task;
    }
}
```

在takeTask()方法中, 也会检查taskQueue取出的task会不会是WAKEUP_TASK:

```java
protected Runnable takeTask() {
......
    for (;;) {
        ScheduledFutureTask< ? > scheduledTask = peekScheduledTask();
        if (scheduledTask == null) {
            Runnable task = null;
            try {
                task = taskQueue.take();
                if (task == WAKEUP_TASK) {
                    // 如果是WAKEUP_TASK, 则设置task为null
                    task = null;
                }
            } catch (InterruptedException e) {
                // Ignore
            }
            // 如果是WAKEUP_TASK, 则返回的是null
            return task;
        } else {
            ......
        }
    }
}
```

## 添加WAKEUP_TASK任务

WAKEUP_TASK这个特殊任务是在wakeup()方法中被添加, wakeup()方法中还需要检查其他条件:

1. inEventLoop==false, 即调用wakeup()的方法应该是外部线程
2. 如果inEventLoop==true, 则调用wakeup()的方法的时event loop线程本身, 则只有当状态为ST_SHUTTING_DOWN即当前Executor正在关闭中时

```java
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop || STATE_UPDATER.get(this) == ST_SHUTTING_DOWN) {
        taskQueue.add(WAKEUP_TASK);
    }
}
```

wakeup()方法在几个shutdown相关的方法中被调用,比如shutdown()/shutdownGracefully()/confirmShutdown()方法中调用. 但是这里我们不是太关心, 我们看最重要的调用的地方, 在execute()方法中:

```java
public void execute(Runnable task) {
	......
    addTask(task);
	......

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}

protected boolean wakesUpForTask(Runnable task) {
    return true;
}
```

在将task添加到taskQueue中之后, 会做一个检查: 如果addTaskWakesUp是false 并且wakesUpForTask(task)也返回true, 则调用wakeup(inEventLoop), 里面还有一个检测是inEventLoop是false, 这样才真正的会向taskQueue中添加一个WAKEUP_TASK.

wakesUpForTask(task)方法默认简单返回true, 这是个protected方法,子类可以按照自己的需要覆盖.

实际在SingleThreadEventLoop中,增加了一个标志接口NonWakeupRunnable, 然后wakesUpForTask()被覆盖为检测是否task是否是NonWakeupRunnable.

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {
    protected boolean wakesUpForTask(Runnable task) {
        return !(task instanceof NonWakeupRunnable);
    }

    interface NonWakeupRunnable extends Runnable {}
}
```

总结说, 在execute(task)方法中, 要最终向taskQueue中添加一个WAKEUP_TASK做wake up, 需要满足以下三个条件:

1. addTaskWakesUp==false: 证实在NioEventLoop的实现中, addTaskWakesUp是设置为false的,因此这个条件在NioEventLoop总是被满足
2. wakesUpForTask(task)返回true: 只要task不要被标记为NonWakeupRunnable就可以
3. inEventLoop==false: 即只能是外部线程调用execute(task)方法

## addTaskWakesUp属性

addTaskWakesUp属性在构造函数中设置, 定义为final设值后不可修改:

```java
private final boolean addTaskWakesUp;
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
    this.addTaskWakesUp = addTaskWakesUp;
}
```

经检查,NioEventLoop的实现中, addTaskWakesUp是设置为false的.

## wake up总结

- 当外部线程调用schedule()方法添加定时任务时, 定时任务会存放在scheduledTaskQueue中
- 用于保存任务的taskQueue是BlockingQueue, 所以当taskQueue中没有任务时, event loop线程会被阻塞

TBD: 看糊涂了......