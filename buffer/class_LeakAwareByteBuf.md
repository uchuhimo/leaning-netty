netty中为了方便监控ByteBuf的泄露, 实现了两个LeakAwareByteBuf:

- AdvancedLeakAwareByteBuf
- SimpleLeakAwareByteBuf

# 调用方式

AdvancedLeakAwareByteBuf和SimpleLeakAwareByteBuf和resource leak detect相关,在方法 AbstractByteBufAllocator.toLeakAwareBuffer()中被调用,分别对应resource leak detect的不同级别: 

```java
protected static ByteBuf toLeakAwareBuffer(ByteBuf buf) {
    ResourceLeak leak;
    switch (ResourceLeakDetector.getLevel()) {
        case SIMPLE:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new SimpleLeakAwareByteBuf(buf, leak);
            }
            break;
        case ADVANCED:
        case PARANOID:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new AdvancedLeakAwareByteBuf(buf, leak);
            }
            break;
        default:
            break;
    }
    return buf;
}
```

toLeakAwareBuffer()方法在以下几个地方被调用,都是用来将普通ByteBuf包装为LeakAwareBuffer.

- PooledByteBufAllocator.newDirectBuffer()
- PooledByteBufAllocator.newHeapBuffer()
- UnpooledByteBufAllocator.newDirectBuffer()

代码类似如下:

```java
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
	// 创建目标ByteBuf
    ByteBuf buf = ......;
	// 包装为LeakAwareBuffer
    return toLeakAwareBuffer(buf);
}
```

总结: **在使用ByteBufAllocator分配ByteBuf时,netty会根据当前的resource detect level的设置使用对应的AdvancedLeakAwareByteBuf或SimpleLeakAwareByteBuf包装真实的ByteBuf**

# 代码实现

## AdvancedLeakAwareByteBuf

构造函数中传入需要包装的ByteBuf和ResourceLeak对象:

```java
final class AdvancedLeakAwareByteBuf extends WrappedByteBuf {
    private final ResourceLeak leak;
    AdvancedLeakAwareByteBuf(ByteBuf buf, ResourceLeak leak) {
        super(buf);
        this.leak = leak;
    }
    ......
}
```

然后几乎所有的方法,都在delegate之前多加一句对leak.record()的调用:

```java
public double getDouble(int index) {
    leak.record();
    return super.getDouble(index);
}
```

然后所有的slice()/duplicate()/readSlice()/order()方法都在返回新的ByteBuf前再次包装成AdvancedLeakAwareByteBuf:

```java
public ByteBuf slice() {
    leak.record();
    return new AdvancedLeakAwareByteBuf(super.slice(), leak);
}
```

## SimpleLeakAwareByteBuf

实现方式基本和AdvancedLeakAwareByteBuf相似,但是只有一个方法添加了对leak.record()的调用:

- order(ByteOrder endianness)