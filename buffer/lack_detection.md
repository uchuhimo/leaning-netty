考虑到buffer释放的复杂性,理论上总是有可能因为某些失误导致有buffer没有正确释放从而发生内存泄露的可能.

为此netty提供了资源泄露检测机制,在AbstractByteBuf中提供了一个ResourceLeakDetector的实例leakDetector来做资源泄露检测.

```java
public abstract class AbstractByteBuf extends ByteBuf {
    static final ResourceLeakDetector<ByteBuf> leakDetector = new ResourceLeakDetector<ByteBuf>(ByteBuf.class);
}
```

# 工作原理

为了检测资源是否泄露,netty中使用了PhantomReference(虚引用)和ReferenceQueue(引用队列).

工作原理如下:

1. 根据检测级别和采样率的设置, 在需要时为需要检测的ByteBuf创建PhantomReference
2. 当JVM回收掉ByteBuf对象时,JVM会将PhantomReference放入ReferenceQueue
3. 通过对ReferenceQueue中PhantomReference的检查,判断在GC前是否有释放ByteBuf的资源,就可以知道是否有资源释放

# 实现机制

理论上, ByteBuf应该做两个事情:

1. 告之leakDetector需要监控的所有对象
	在创建时应该调用leakDetector.open()方法,传入自身(也就是this)的引用.这样leakDetector就可以通过PhantomReference + ReferenceQueue 的机制来监控这些对象.
    为了后面能调用close方法,这里需要保存返回的ResourceLeak对象.
2. 当被监控的对象正常释放时告之leakDetector
	在ByteBuf回收时,应该调用ResourceLeak.close()方法来告之.

以类CompositeByteBuf为例:

```java
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {
	private final ResourceLeak leak;

    public CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents) {
		......
        leak = leakDetector.open(this);
    }
}

protected void deallocate() {
	......

    if (leak != null) {
        leak.close();
    }
}
```

# 代码实现

## open()

ResourceLeakDetector的open()方法生成leakDetector对象:

```java
public final class ResourceLeakDetector<T> {
    public ResourceLeak open(T obj) {
    	......
        return new DefaultResourceLeak(obj);
    }
}
```

这个leakDetector对象实际是一个PhantomReference, 注意构造函数的实现,refQueue是前面所说的ReferenceQueue:

```java
private final class DefaultResourceLeak extends PhantomReference<Object> implements ResourceLeak {
    DefaultResourceLeak(Object referent) {
        super(referent, referent != null? refQueue : null);
    }
}
```

## close()

ResourceLeakDetector的close()方法,根据内部的freed属性来判断是否是第一次调用,然后返回true/false:

```java
public boolean close() {
    if (freed.compareAndSet(false, true)) {
        ......
        return true;
    }
    return false;
}
```

判断的基本的原则:

1. 如果ByteBuf在GC前被正常释放,那么就会正常调用close()方法,这样下一次再调用close时会返回false
2. 如果ByteBuf被GC前未能正常释放, close()方法没能及时调用,那么再调用close()时会返回true

## reportLeak()

检测的方式就是从refQueue中获取已经被GC了的DefaultResourceLeak对象,然后按照上面的逻辑调用close()方法做检测:

```java
private void reportLeak(Level level) {
    for (;;) {
        DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
        if (ref == null) {
            break;
        }

        ref.clear();

        if (!ref.close()) {
        	// 返回false, 说明之前已经调用郭,资源被正常释放
            continue;
        }

        // 发现泄露了,在这里报错!
    }
}
```

详尽的代码分析,请见下节.
