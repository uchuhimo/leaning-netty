在类Unpooled中提供给了多个名为wrappedBuffer()的帮助方法,用来创建包装给定数据的ByteBuf对象.

## 包装单个byte数组

代码实现:

```java
public static ByteBuf wrappedBuffer(byte[] array) {
    if (array.length == 0) {
        return EMPTY_BUFFER;
    }
    return new UnpooledHeapByteBuf(ALLOC, array, array.length);
}
```

结论: byte[]被包装成一个UnpooledHeapByteBuf对象.

## 包装多个byte数组

代码实现:

```java
public static ByteBuf wrappedBuffer(int maxNumComponents, byte[]... arrays) {
    switch (arrays.length) {
    case 0:
    	//如果长度为0,返回EMPTY_BUFFER
        break;
    case 1:
        if (arrays[0].length != 0) {
        	//长度为1, 调用回wrappedBuffer(byte[])
            return wrappedBuffer(arrays[0]);
        }
        break;
    default:
        // Get the list of the component, while guessing the byte order.
        final List<ByteBuf> components = new ArrayList<ByteBuf>(arrays.length);
        for (byte[] a: arrays) {		// 游历数组,如果为null或空就跳过
            if (a == null) {
                break;
            }
            if (a.length > 0) {
            	// 有效的数据就通过调用wrappedBuffer(byte[]包装为一个ByteBuf
                components.add(wrappedBuffer(a));
            }
        }

        if (!components.isEmpty()) {
        	//将多个ByteBuf包装为1个CompositeByteBuf
            return new CompositeByteBuf(ALLOC, false, maxNumComponents, components);
        }
    }

    return EMPTY_BUFFER;
}
```

结论: 多个byte[]被包装成多个UnpooledHeapByteBuf对象,最后再包装进一个CompositeByteBuf.

## 包装单个ByteBuf

代码实现:

```java
public static ByteBuf wrappedBuffer(ByteBuf buffer) {
    if (buffer.isReadable()) {
    	// 有可读数据, 调用buffer.slice()将可读数据包装为一个ByteBuf
        return buffer.slice();
    } else {
    	// 如果没有可读数据,返回EMPTY_BUFFER
        return EMPTY_BUFFER;
    }
}
```

结论: 多个ByteBuf对象(的可读内容)被包装进一个CompositeByteBuf.

## 包装多个ByteBuf

代码实现:

```java
public static ByteBuf wrappedBuffer(int maxNumComponents, ByteBuf... buffers) {
    switch (buffers.length) {
    case 0:
    	//如果长度为0,返回EMPTY_BUFFER
        break;
    case 1:
        if (buffers[0].isReadable()) {
        	//长度为1并且有可读数据, 调用回wrappedBuffer(ByteBuf)
            // TBD: 不清楚为什么这里需要调用 order(BIG_ENDIAN)?
            return wrappedBuffer(buffers[0].order(BIG_ENDIAN));
        }
        break;
    default:
    	// 注意这里的判断: 只要有一个ByteBuf有可读数据,就包装为CompositeByteBuf
        for (ByteBuf b: buffers) {
            if (b.isReadable()) {
            	//将多个ByteBuf包装为1个CompositeByteBuf
                return new CompositeByteBuf(ALLOC, false, maxNumComponents, buffers);
            }
        }
    }
    return EMPTY_BUFFER;
}
```

结论: 多个ByteBuf对象(的可读内容)被包装进一个CompositeByteBuf.

