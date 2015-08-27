

类分析

# ByteBuf

第一次看到ByteBuf的代码,发现里面其实都是abstract方法定义,没有任何实质代码.所以很奇怪为什么ByteBuf要设计为abstract class而不是直接定义为interface?

后来发现ByteBuf的类定义上特意加了一句@SuppressWarnings("ClassMayBeInterface"),呵呵,看来设计成abstract class 而不是interface是有深意的,只是javadoc里面没有说明到底理由是什么.暂时我还没有找到答案.

	@SuppressWarnings("ClassMayBeInterface")
	public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf>

[接口 ReferenceCounted](../util/interface_ReferenceCounted.html)

## EmptyByteBuf

## SwappedByteBuf

## WrappedByteBuf

## AbstractByteBuf