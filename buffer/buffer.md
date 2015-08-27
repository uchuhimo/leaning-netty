
netty javadoc对bytebuffer的定义:

> Abstraction of a byte buffer - the fundamental data structure to represent a low-level binary and text message.

byte buffer的抽象 - 用于表述底层二进制和文本信息的基本数据结构.

> Netty uses its own buffer API instead of NIO ByteBuffer to represent a sequence of bytes. This approach has significant advantage over using ByteBuffer. Netty's new buffer type, ByteBuf, has been designed from ground up to address the problems of ByteBuffer and to meet the daily needs of network application developers.

Netty 使用自己的 buffer API 替代java NIO的ByteBuffer来表示字节系列.理由自然是觉得JAVA NIO那套ByteBuffer设计的不好, 所以自己整一套更好用的出来. 而实际上netty设计的这套ByteBuf的确要强大和实用.

netty列出了认为酷的特性:

- You can define your buffer type if necessary.
- Transparent zero copy is achieved by built-in composite buffer type.
- A dynamic buffer type is provided out-of-the-box, whose capacity is expanded on - demand, just like StringBuffer.
- There's no need to call the flip() method anymore.
- It is often faster than ByteBuffer.

TBD:

- [Netty 4.x学习笔记 – ByteBuf](http://hongweiyi.com/2014/01/netty-4-x-bytebuf/)

以下是 ByteBuf 的几个重要特性,后面详细展开:

- Pooled
- Reference Count
- Zero Copy
