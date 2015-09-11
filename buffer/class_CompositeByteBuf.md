在netty的zero-copy的实现中,类CompositeByteBuf非常的重要.

CompositeByteBuf的javadoc这样描述:

	A virtual buffer which shows multiple buffers as a single merged buffer.
    虚拟的buffer,将多个buffer展现为一个简单合并的buffer.

使用上,推荐使用以下两类帮助方法来创建CompositeByteBuf对象,尽量不要直接调用构造函数:

- ByteBufAllocator.compositeBuffer()
- Unpooled.wrappedBuffer(ByteBuf...)

## 继承关系

```java
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {}
```

## 属性

以下几个属性是CompositeByteBuf特有的属性:

```java
    private final ByteBufAllocator alloc;
    private final boolean direct;
    private final List<Component> components = new ArrayList<Component>();
    private final int maxNumComponents;
```

Component是私有内嵌类,用于保存ByteBuf的引用,长度等信息.

```java
    private static final class Component {
        final ByteBuf buf;
        final int length;
        int offset;
        int endOffset;
		......
    }
```

另外为了支持ReferenceCounted和resourceleakdetect,还有两个特殊属性:

```java
    private final ResourceLeak leak;
    private boolean freed;
```

## 构造函数

第一个构造函数比较简单,没有传入ByteBuf,需要在后面通过调用addComponents()方法来增加内容:

```java
public CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents) {
    super(Integer.MAX_VALUE);	//设置maxCapacity为Integer.MAX_VALUE
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    this.alloc = alloc;
    this.direct = direct;
    this.maxNumComponents = maxNumComponents;
    leak = leakDetector.open(this);
}
```

第二个构造函数传入了ByteBuf:

```java
public CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents, ByteBuf... buffers) {
    super(Integer.MAX_VALUE);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (maxNumComponents < 2) {
    	// 这里做法比较狠,如果传入的maxNumComponents<2就直接抛异常退出
        throw new IllegalArgumentException(
                "maxNumComponents: " + maxNumComponents + " (expected: >= 2)");
    }

    this.alloc = alloc;
    this.direct = direct;
    this.maxNumComponents = maxNumComponents;

    addComponents0(0, buffers);		//添加传入的buffers,index自然是从0开始
    consolidateIfNeeded();
    setIndex(0, capacity());
    leak = leakDetector.open(this);
}
```

addComponent0()方法:

```java
private int addComponent0(int cIndex, ByteBuf buffer) {
    checkComponentIndex(cIndex);

    if (buffer == null) {
        throw new NullPointerException("buffer");
    }

    int readableBytes = buffer.readableBytes();

    // No need to consolidate - just add a component to the list.
    Component c = new Component(buffer.order(ByteOrder.BIG_ENDIAN).slice());
    if (cIndex == components.size()) {
    	// 加在最后
        components.add(c);
        if (cIndex == 0) {
            c.endOffset = readableBytes;
        } else {
            Component prev = components.get(cIndex - 1);
            c.offset = prev.endOffset;
            c.endOffset = c.offset + readableBytes;
        }
    } else {
    	// 加在中间指定index上
        components.add(cIndex, c);
        if (readableBytes != 0) {
            updateComponentOffsets(cIndex);
        }
    }
    return cIndex;
}
```

再看consolidateIfNeeded()方法:

```java
private void consolidateIfNeeded() {
    // Consolidate if the number of components will exceed the allowed maximum by the current
    // operation.
    final int numComponents = components.size();
    if (numComponents > maxNumComponents) {	//这个if判断至关重要!!
        final int capacity = components.get(numComponents - 1).endOffset;

        ByteBuf consolidated = allocBuffer(capacity);

        // We're not using foreach to avoid creating an iterator.
        for (int i = 0; i < numComponents; i ++) {
            Component c = components.get(i);
            ByteBuf b = c.buf;
            consolidated.writeBytes(b);
            c.freeIfNecessary();
        }
        Component c = new Component(consolidated);
        c.endOffset = c.length;
        components.clear();
        components.add(c);
    }
}
```

可以看到: **当实际components的数量大于maxNumComponents时**, 所有的components就会被合并为一个ByteBuf, "consolidated.writeBytes(b)" 这段代码就是在将原来ByteBuf的数据写入到新的consolidated这个合并之后的ByteBuf. 这意味着数据被复制, zeor-copy也就不存在了.

反回来看numComponents的设值, 这是一个final,通过构造函数一次性赋值.

```java
private final int maxNumComponents;
public CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents) {
    ......
    this.maxNumComponents = maxNumComponents;
    ......
}
```

而在前面Unpooled.wrappedBuffer()方法中, 如果没有明确设置maxNumComponents,则默认为16:

```java
public static ByteBuf wrappedBuffer(byte[]... arrays) {
    return wrappedBuffer(16, arrays);
}
```

这里是否存在风险: __如果默认maxNumComponents为16,那么当component添加时,一旦超过16, 就将触发consolidate的操作, 违背zero-copy__.

## 其他方法

貌似没有什么特殊