
# 类定义

AbstractReferenceCountedByteBuf的类定义:

````
public abstract class AbstractReferenceCountedByteBuf extends AbstractByteBuf

public abstract class AbstractByteBuf extends ByteBuf

public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf>
````

ByteBuf申明了对接口ReferenceCounted的实现,但是实际的代码支持在AbstractReferenceCountedByteBuf中.

# 代码分析

类AbstractReferenceCountedByteBuf中主要是实现了ByteBuf对接口ReferenceCounted的支持,没有其他代码.

## refCntUpdater

静态变量refCntUpdater和静态初始化代码:

````
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater;

static {
    AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> updater =
            PlatformDependent.newAtomicIntegerFieldUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");
    if (updater == null) {
        updater = AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");
    }
    refCntUpdater = updater;
}
````
在类装载时初始化refCntUpdater以备用. 使用到PlatformDependent类(这块细节不展开).

## refCnt

类属性refCnt的定义, 注意初始化值为1:

````
private volatile int refCnt = 1;
````

关键字volatile使得对refCnt 的访问都会基于主内存.refCnt()方法就只是简单返回这个refCnt属性.

```
public final int refCnt() {
    return refCnt;
}
```

## retain()方法

ReferenceCounted的工作原理: 当引用计数对象被初始化时, 引用计数器从1开始计数. retain()方法增加引用计数,而release()方法减少引用计数.如果引用计数被减到0,则这个对象就将被显式回收,而访问被回收的对象将通常会导致访问违规.

前面看到refCnt初始化值为1,和上面的原理一致. 继续看retain()方法的实现:

```
@Override
public ByteBuf retain() {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt == 0) {
            throw new IllegalReferenceCountException(0, 1);
        }
        if (refCnt == Integer.MAX_VALUE) {
            throw new IllegalReferenceCountException(Integer.MAX_VALUE, 1);
        }
        if (refCntUpdater.compareAndSet(this, refCnt, refCnt + 1)) {
            break;
        }
    }
    return this;
}
```

核心代码是for循环加compareAndSet()方法, 通过CAS机制在不加锁的情况下实现了对refCnt变量的并发更新, "refCnt + 1"表明计数器加1:

```
    for (;;) {
        int refCnt = this.refCnt;
        if (refCntUpdater.compareAndSet(this, refCnt, refCnt + 1)) {
            break;
        }
    }
```
"refCnt == 0"的检测则是遵循原理中的要求: 访问被回收的对象将通常会导致访问违规.

retain(int increment) 方法的实现和retain()几乎完全相同,只是increment不是固定为1而已. 有点不太理解的是netty为什么不直接通过在retain()中调用retain(1)的方式来实现, 我能想到的理由就是retain()的调用非常频繁(远远超过retain(int increment)), 因此现在的做法可以带来少许的性能提升?

## release()方法

release()方法的代码实现:

```
@Override
public final boolean release() {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt == 0) {
            throw new IllegalReferenceCountException(0, -1);
        }

        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - 1)) {
            if (refCnt == 1) {
                deallocate();
                return true;
            }
            return false;
        }
    }
}
```

代码核心有两个功能:

1. 通过CAS实现refCnt减一,方式和retain()里面是一样的
2. 判断refCnt,如果==1,意味着这次release()调用之后计数器会被置零.按照原理要求,"如果引用计数被减到0,则这个对象就将被显式回收".因此调用deallocate()方法显式回收对象

返回值只有在deallocate()时才return true, 其他情况都是return false.

"refCnt == 0"的检测同样时遵循原理的要求: 访问被回收的对象将通常会导致访问违规.

然后看deallocate(),只是一个protected abstract的模板方法, 子类自行实现.

```
    /**
     * Called once {@link #refCnt()} is equals 0.
     */
    protected abstract void deallocate();
```

release(int decrement)方法实现基本相同.

## touch()方法

AbstractReferenceCountedByteBuf中提供了touch()方法的两个空实现, 基本没有任何实际意义:

```
@Override
public ByteBuf touch() {
    return this;
}

@Override
public ByteBuf touch(Object hint) {
    return this;
}
```

找了一下子类,也没有发现有其他的具体实现,这块有点糊涂(难道netty还没有完成这个touch机制的实现?).
