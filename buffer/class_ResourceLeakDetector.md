netty在package util中实现了类ResourceLeakDetector：

```java
package io.netty.util;
public final class ResourceLeakDetector<T> {
```

# 基本概念

## Detect Level

ResourceLeakDetector.Level定义了资源泄露检测的等级：

- DISABLED： 禁用，不做检测
- SIMPLE： 开启资源泄露检测的简化采样，仅仅报告是否存在泄露，开销小. 这个是默认级别。
- ADVANCED: 开始资源泄露检测的高级采样，报告被泄露的对象最后一次访问的地址，消耗高
- PARANOID： 开始资源泄露检测的偏执采样，报告被泄露的对象最后一次访问的地址，消耗最高(仅适用于测试)

类ResourceLeakDetector被classloader装载时，会执行下面的静态初始化代码：

```java
private static final String PROP_LEVEL = "io.netty.leakDetectionLevel";
private static final Level DEFAULT_LEVEL = Level.SIMPLE;

private static Level level;

static {
    String levelStr = SystemPropertyUtil.get(PROP_LEVEL, DEFAULT_LEVEL.name()).trim().toUpperCase();
    Level level = DEFAULT_LEVEL;
    for (Level l: EnumSet.allOf(Level.class)) {
        if (levelStr.equals(l.name()) || levelStr.equals(String.valueOf(l.ordinal()))) {
            level = l;
        }
    }

    ResourceLeakDetector.level = level;
    if (logger.isDebugEnabled()) {
        logger.debug("-D{}: {}", PROP_LEVEL, level.name().toLowerCase());
    }
}
```

读取系统配置io.netty.leakDetectionLevel，如果没有设置或者设置不正确则取默认值 Level.SIMPLE。

## Sample Interval

采样间隔在初始化时指定，注意samplingInterval是final，设定后不可修改。

```java
private final int samplingInterval;

public ResourceLeakDetector(String resourceType, int samplingInterval, long maxActive) {
    ......
    this.samplingInterval = samplingInterval;
}
```

如果不指定，则取默认值113：

```java
private static final int DEFAULT_SAMPLING_INTERVAL = 113;

public ResourceLeakDetector(String resourceType) {
    this(resourceType, DEFAULT_SAMPLING_INTERVAL, Long.MAX_VALUE);
}
```

samplingInterval的使用和leakDetectionLevel直接相关：

- 如果leakDetectionLevel被设置为PARANOID，则每次都采样，samplingInterval设置失效
- 如果leakDetectionLevel被设置为DISABLED，则禁用采样，samplingInterval设置失效
- 当leakDetectionLevel被设置为SIMPLE或者ADVANCED时， samplingInterval设置才生效

看open()方法中的代码实现，先忽略其他细节，只看level和samplingInterval的部分：

```java
public ResourceLeak open(T obj) {
    Level level = ResourceLeakDetector.level;
    if (level == Level.DISABLED) {
        return null;
    }

    if (level.ordinal() < Level.PARANOID.ordinal()) {
        if (leakCheckCnt ++ % samplingInterval == 0) {
            reportLeak(level);
            return new DefaultResourceLeak(obj);
        } else {
            return null;
        }
    } else {
        reportLeak(level);
        return new DefaultResourceLeak(obj);
    }
}
```

可以看到这里的处理逻辑是：

1. 当"level == Level.DISABLED" 时，return null表示不采样
2. 当"level.ordinal() < Level.PARANOID"不成立时，实际就是level等于PARANOID时，每次都取样
3. 当"level.ordinal() < Level.PARANOID" 时, 则根据当前计数器leakCheckCnt的值对samplingInterval取余数判断是否为0来决定是否取样

考虑默认情况samplingInterval=113，采样概率为1/113 大概等于9/1000，或者近似于1%.

## leakCheckCnt

leakCheckCnt用于计数，配合samplingInterval来判断是否满足取样的条件。

```java
private long leakCheckCnt;

public ResourceLeak open(T obj) {
	if (leakCheckCnt ++ % samplingInterval == 0) {}
}
```

## resourceType

resourceType在构造函数中指定，依然是final设置后不可修改：

```java
private final String resourceType;

public ResourceLeakDetector(String resourceType, int samplingInterval, long maxActive) {
    if (resourceType == null) {
        throw new NullPointerException("resourceType");
    }
    this.resourceType = resourceType;
}
```

resourceType可以通过String类型来指定，也可以通过resource类的Class来设置：

```java
public ResourceLeakDetector(Class<?> resourceType) {
    this(simpleClassName(resourceType));
}
public ResourceLeakDetector(String resourceType){}
```

通过Class指定时会通过调用simpleClassName()方法去掉package名而获取简单类名。

这里貌似有个隐患：如果有两个同名而package不同的resource，会被视为同一个。

resourceType没有涉及到任何具体的实现逻辑，仅仅是在打印日志的时候作为一个标识符输出。

## active

和active概念相关的有三个变量：

```java
private final long maxActive;
private long active;
private final AtomicBoolean loggedTooManyActive = new AtomicBoolean();
```

- maxActive： 容许的active最大数量，在初始化时指定，如果不指定则默认为Long.MAX_VALUE
- active：当前active的数量
- loggedTooManyActive： 是否已经打印太多active实例的标识符

active默认为0，在DefaultResourceLeak构造函数中+1，在close()中-1（细节先忽略）：

```java
DefaultResourceLeak(Object referent) {
	......
    active ++;
}

public boolean close() {
	......
    active --;
}
```

然后在reportLeak()方法中检测当前active：

```java
private void reportLeak(Level level) {
	......
    // Report too many instances.
    int samplingInterval = level == Level.PARANOID? 1 : this.samplingInterval;
    if (active * samplingInterval > maxActive && loggedTooManyActive.compareAndSet(false, true)) {
        logger.error("LEAK: You are creating too many " + resourceType + " instances.  " +
                resourceType + " is a shared resource that must be reused across the JVM," +
                "so that only a few instances are created.");
    }
```

如果"active * samplingInterval > maxActive"成立，则以CAS的方式设置loggedTooManyActive为true，如果成功，则打印错误信息，提示当前resourceType类型的资源已经创建了太多的实例。

注意**这个日志打印的操作只会触发一次**，后续即使满足"active * samplingInterval > maxActive"， loggedTooManyActive因为已经被设置为true了，所以"loggedTooManyActive.compareAndSet(false, true)"必然失败返回false从而if条件不满足。

# 内部辅助类

## interface ResourceLeak

接口ResourceLeak定于在package io.netty.util中。

```java
package io.netty.util;
public interface ResourceLeak {}
```

方法如下：

- void record(Object hint)： 记录调用者当前的stack trace和指定的额外的任意的(arbitrary)信息以便ResourceLeakDetector能告诉说被泄露的资源最后一次访问是在哪里。
- void record()： 等同于record(null)
- boolean close(): 关闭当前leak，这样ResourceLeakDetector就不再担心资源泄露

## class DefaultResourceLeak

DefaultResourceLeak类实现ResourceLeak接口，继承自PhantomReference：

```java
private final class DefaultResourceLeak extends PhantomReference<Object> implements ResourceLeak {}
```

### creationRecord

creationRecord属性用来记录DefaultResourceLeak对象初始化时的记录，只有当**referent不等于null并且level大于或等于Level.ADVANCED时**才做这个记录。

```java
private final String creationRecord;

DefaultResourceLeak(Object referent) {
	......
    if (referent != null) {
        Level level = getLevel();
        if (level.ordinal() >= Level.ADVANCED.ordinal()) {
            creationRecord = newRecord(null, 3);
        } else {
            creationRecord = null;
        }
		......
    } else {
    	......
        creationRecord = null;
    }
}
```

看newRecord()函数的实现，会发现为了得到stack trace，采用了一个非常规的动作"new Throwable()":

```java
static String newRecord(Object hint, int recordsToSkip) {
	......
    // Append the stack trace.
    StackTraceElement[] array = new Throwable().getStackTrace();
    for (StackTraceElement e: array) {
        ......
    }
}
```

在运行时创建Throwable对象是一个开销非常巨大的操作，因此这个操作对性能影响非常巨大。

### lastRecords

lastRecords记录最后几次访问记录。初始化为空，之后在record方法被调用时开始记录信息：

```java
private final Deque<String> lastRecords = new ArrayDeque<String>();

private void record0(Object hint, int recordsToSkip) {
    if (creationRecord != null) {
        String value = newRecord(hint, recordsToSkip);

        synchronized (lastRecords) {
            int size = lastRecords.size();
            if (size == 0 || !lastRecords.getLast().equals(value)) {
                lastRecords.add(value);
            }
            if (size > MAX_RECORDS) {
                lastRecords.removeFirst();
            }
        }
    }
}
```

注意"creationRecord != null"的判断， 考虑creationRecord是final，前面看到creationRecord属性只有当**referent不等于null并且level大于或等于Level.ADVANCED时**才做记录。因此这里lastRecords的记录实际上也是遵循上面的判断条件。

记录时同样调用newRecord()，里面依然是一次"new Throwable()"的开销。

MAX_RECORDS限制了最大记录数量，默认时4，看代码中没有能设置的地方，而且也是final，因此次数就被固定为4了：

```java
private static final int MAX_RECORDS = 4;
```
### freed

freed在初始化时赋值,如果referent为空,则初始值时true.

```java
private final AtomicBoolean freed;

DefaultResourceLeak(Object referent) {
	......
    if (referent != null) {
    	......
        freed = new AtomicBoolean();
    } else {
    	......
        freed = new AtomicBoolean(true);
    }
}
```

唯一使用freed属性的地方是close()函数:

```java
public boolean close() {
    if (freed.compareAndSet(false, true)) {
		......
        return true;
    }
    return false;
}
```

只有当构造函数中referent不为null时, freed的初始值为false,才能让这里的"freed.compareAndSet(false, true)"返回true.

## linked list of active resources

DefaultResourceLeak内部有两个引用prev/next, 分别指向上一个和下一个.这样就将所有DefaultResourceLeak都链接起来.

```java
public final class ResourceLeakDetector<T> {
    /** the linked list of active resources */
    private final DefaultResourceLeak head = new DefaultResourceLeak(null);
    private final DefaultResourceLeak tail = new DefaultResourceLeak(null);

    private final class DefaultResourceLeak {
        private DefaultResourceLeak prev;
        private DefaultResourceLeak next;
    }
}
```

而在类ResourceLeakDetector中, 还保存有两个指向active resources 链接列表的头和尾的引用head/tail.在ResourceLeakDetector的构造函数中,将head.next指向tail,将tail.prev指向head:

```java
public ResourceLeakDetector(String resourceType, int samplingInterval, long maxActive) {
    ......

    head.next = tail;
    tail.prev = head;
}
```

在DefaultResourceLeak的构造函数中,如果"referent != null", 则需要在原来的head和head.next之间插入当前实例:

	"head -- head.next"   >>> "head   -- this -- head.next"

然后需要分别调整head/this/head.next的引用

```java
DefaultResourceLeak(Object referent) {
	......

    if (referent != null) {
        ......
        synchronized (head) {
            prev = head;				//将this.prev指向原来的head
            next = head.next;			//将this.next指向原来的head.next
            head.next.prev = this;		//将head.next对象的prev从指向原来的head变成指向this
            head.next = this;			//将head的next指向this
        }
        ......
    } else {
        ......
    }
}
```

考虑到head和tail只是两个虚拟的引用,其实上面的操作相当于在linked list的最前面增加了一个元素.

在close()方法中,需要将当前对象从链中摘出来:

	"head   -- this -- head.next"  >>>  "head -- head.next"

代码类似:

```java
public boolean close() {
    if (freed.compareAndSet(false, true)) {
        synchronized (head) {
            prev.next = next;			//将perv的next从指向this修改为指向this.next
            next.prev = prev;			//将next的prev从指向this修改为指向this.prev
            prev = null;				//清理this.prev
            next = null;				//清理this.next
        }
    }
}
```

# 核心代码

完成上面的代码分析之后,我们来看最核心的代码逻辑:

```java
private final ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();
```

这里定义了refQueue作为PhantomReference的ReferenceQueue,还记得DefaultResourceLeak类继承PhantomReference吗?

## ResourceLeakDetector.open()方法

open()方法用户创建一个新的ResourceLeak对象, 当相关的资源被回收时期望这个ResourceLeak对象会通过调用close()方法来关闭.

```java
public ResourceLeak open(T obj) {
    Level level = ResourceLeakDetector.level;
    if (level == Level.DISABLED) {
        return null;
    }

    if (level.ordinal() < Level.PARANOID.ordinal()) {
        if (leakCheckCnt ++ % samplingInterval == 0) {
            reportLeak(level);
            return new DefaultResourceLeak(obj);
        } else {
            return null;
        }
    } else {
        reportLeak(level);
        return new DefaultResourceLeak(obj);
    }
}
```

如果满足level和samplingInterval的要求,则开始采样.open()方法就会返回一个DefaultResourceLeak对象,包含传入的需要被监测的资源对象.

注意"new DefaultResourceLeak(obj)"的调用中, DefaultResourceLeak的构造函数会调用super(也就是PhantomReference)的构造函数,传入要引用的对象和ReferenceQueue:

```java
DefaultResourceLeak(Object referent) {
	super(referent, referent != null? refQueue : null);
    ......
}
```

这样就完成了对需要检测的对象的监控,之后如果被检测的引用对象被JVM回收,则PhantomReference会被放入ReferenceQueue(也就是refQueue).

## DefaultResourceLeak.close()方法

如果close()被正常调用,那么freed.compareAndSet(false, true)就会成立,最后返回true.如果是第二次调用,则会返回false. 因此可以通过调用close()方法然后判断返回值来看是否有资源泄露.

```java
public boolean close() {
    if (freed.compareAndSet(false, true)) {
		......
        return true;
    }
    return false;
}
```

## ResourceLeakDetector.reportLeak()方法

在ResourceLeakDetector.open()方法当中, 如果需要采样,则会先调用一次reportLeak()方法来判断之前检测的对象是否有出现资源泄露的情况.

```java
private void reportLeak(Level level) {
    if (!logger.isErrorEnabled()) {
        for (;;) {
            @SuppressWarnings("unchecked")
            DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
            if (ref == null) {
                break;
            }
            ref.close();
        }
        return;
    }
    ......
}
```

这段代码先判断是否容许输出错误日志,如果不被容许,那么也就没有机会输出内存泄露的信息了,所有没有必要做后面的真实检测,直接清理refQueue中的数据.

如果容许输出错误日志,则继续检测流程. 先判断一下是否当前有太多的实例了, 注意只是实例太多,实例太多不代表一定是内存泄露.但太多超过了maxActive的限制,就应该输出对应的错误信息: 

```java
// Report too many instances.
int samplingInterval = level == Level.PARANOID? 1 : this.samplingInterval;
if (active * samplingInterval > maxActive && loggedTooManyActive.compareAndSet(false, true)) {
    logger.error("LEAK: You are creating too many " + resourceType + " instances.  " +
            resourceType + " is a shared resource that must be reused across the JVM," +
            "so that only a few instances are created.");
}
```

然后才是最核心的检测代码: 从refQueue中循环poll数据,然后调用ref.close(). 如果返回false,说明之前已经被正常调用一次close(),资源没有泄露.

```java
// Detect and report previous leaks.
for (;;) {
    @SuppressWarnings("unchecked")
    DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
    if (ref == null) {
        break;
    }

    ref.clear();

    if (!ref.close()) {
        continue;
    }

    ......
```

如果ref.close()返回true,则说明之前close()方法没有被调用.这里就需要汇报资源泄露的信息了:

```java
String records = ref.toString();
if (reportedLeaks.putIfAbsent(records, Boolean.TRUE) == null) {
    if (records.isEmpty()) {
        logger.error("LEAK: {}.release() was not called before it's garbage-collected. " +
                "Enable advanced leak reporting to find out where the leak occurred. " +
                "To enable advanced leak reporting, " +
                "specify the JVM option '-D{}={}' or call {}.setLevel() " +
                "See http://netty.io/wiki/reference-counted-objects.html for more information.",
                resourceType, PROP_LEVEL, Level.ADVANCED.name().toLowerCase(), simpleClassName(this));
    } else {
        logger.error(
                "LEAK: {}.release() was not called before it's garbage-collected. " +
                "See http://netty.io/wiki/reference-counted-objects.html for more information.{}",
                resourceType, records);
    }
}
```