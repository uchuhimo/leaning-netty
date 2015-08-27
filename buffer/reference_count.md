自从Netty 4开始，对象的生命周期由它们的引用计数（reference counts）管理，而不是由垃圾收集器（garbage collector）管理。

netty的官方wiki给出了Reference counted的一份介绍资料: [Reference counted objects](http://netty.io/wiki/reference-counted-objects.html)(也可以看它的[中文翻译版本](http://damacheng009.iteye.com/blog/2013657)).

Netty 中通过接口　ReferenceCounted　定义了引用计数的基本功能，然后　ByteBuf　申明实现了ReferenceCounted接口：

```
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf>
```

先看一下ReferenceCounted的定义，和ByteBuf是如果实现ReferenceCounted的，后面再来看如何使用Reference　Counte特性。