UnreleasableByteBuf 用于阻止他人对目标ByteBuf的销毁.

# 实现方式

在构造函数中传入需要包裹的ByteBuf:

```java
final class UnreleasableByteBuf extends WrappedByteBuf {

    UnreleasableByteBuf(ByteBuf buf) {
        super(buf);
    }
}
```

然后覆盖retain()/release()方法,不做任何操作,只是简单的返回false:

```java
@Override
public ByteBuf retain(int increment) {
    return this;
}

@Override
public ByteBuf retain() {
    return this;
}

@Override
public boolean release() {
    return false;
}

@Override
public boolean release(int decrement) {
    return false;
}
```

再覆盖slice()/readSlice()/duplicate()方法,将需要返回的ByteBuf再次包装为UnreleasableByteBuf:

```java
@Override
public ByteBuf readSlice(int length) {
    return new UnreleasableByteBuf(buf.readSlice(length));
}

@Override
public ByteBuf slice() {
    return new UnreleasableByteBuf(buf.slice());
}

@Override
public ByteBuf slice(int index, int length) {
    return new UnreleasableByteBuf(buf.slice(index, length));
}

@Override
public ByteBuf duplicate() {
    return new UnreleasableByteBuf(buf.duplicate());
}
```

# 使用方式

Unpooled中提供unreleasableBuffer()工具方法,代码够简单的:

```java
public static ByteBuf unreleasableBuffer(ByteBuf buf) {
    return new UnreleasableByteBuf(buf);
}
```

一般的使用场景就是定义特殊的常量ByteBuf,然后包装成unreleasableBuffer()后就不怕被其他人错误的销毁掉:

```java
public abstract class HttpObjectEncoder<H extends HttpMessage> extends MessageToMessageEncoder<Object> {
    private static final ByteBuf CRLF_BUF = unreleasableBuffer(directBuffer(CRLF.length).writeBytes(CRLF));
}
```
