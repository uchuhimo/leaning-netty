# 继承关系

类SlicedByteBuf的继承关系,继承自AbstractDerivedByteBuf:

```java
public class SlicedByteBuf extends AbstractDerivedByteBuf {
    private ByteBuf buffer;
    private int adjustment;
    private int length;
}
```

# 内部属性

SlicedByteBuf有三个最重要的内部属性:

1. buffer: 引用(或者说映射/指向)的底层ByteBuf对象,真实的数据时存放在这里
2. adjustment: 调整量,类似偏移量的概念,指当前SlicedByteBuf映射到底层ByteBuf对象时的偏移量
3. length: 当前SlicedByteBuf的长度

可以理解为SlicedByteBuf是对原有ByteBuf对象的部分数据的抽象,比如原来有一个ByteBuf有100个字节,我们将其中第20到49这30个字节的数据映射到一个新的SlicedByteBuf,这个SlicedByteBuf

1. buffer: 指向原来的这个100字节的ByteBuf
2. adjustment: 设置为20,表示从下标20开始的数据
3. length: 设置为30,表示从adjustment开始的30字节的数据范围

# 代码实现

## 构造函数

我们来看SlicedByteBuf的构造函数:

```java
public SlicedByteBuf(ByteBuf buffer, int index, int length) {
    super(length);
    init(buffer, index, length);
}

final void init(ByteBuf buffer, int index, int length) {
    if (buffer instanceof SlicedByteBuf) {
        this.buffer = ((SlicedByteBuf) buffer).buffer;
        adjustment = ((SlicedByteBuf) buffer).adjustment + index;
    } else if (buffer instanceof DuplicatedByteBuf) {
        this.buffer = buffer.unwrap();
        adjustment = index;
    } else {
        this.buffer = buffer;
        adjustment = index;
    }
    this.length = length;
    maxCapacity(length);
    setIndex(0, length);
    discardMarks();
}
```

可以看到buffer/adjustment/length的设置和前面说的一致.

注意SlicedByteBuf是支持对另外一个SlicedByteBuf做slice的.

## 方法

部分方法是直接delegate给buffer:

```java
public boolean hasArray() {
    return buffer.hasArray();
}
```

部分方法是delegate给buffer,但是需要考虑adjustment因素:

```java
protected byte _getByte(int index) {
    return buffer.getByte(index + adjustment);
}
```
