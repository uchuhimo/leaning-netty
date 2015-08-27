ReferenceCounted的设计和ByteBuf的实现(AbstractReferenceCountedByteBuf)都不复杂。

但是对于如何使用ByteBuf, 在使用时候应该调用remain()/release()依然是个非常重要的话题。

＃　原则

究竟应该谁，在申明时候来销毁ByteBuf？

[netty的wiki](http://netty.io/wiki/reference-counted-objects.html)中给出建议：

** 通常的经验是谁最后访问引用计数对象，谁就负责销毁它**

具体来说是以下两点：

- 如果组件A把一个引用计数对象传给另一个组件B，那么组件A通常不需要销毁对象，而是把决定权交给组件B
- 如果一个组件不再访问一个引用计数对象，那么这个组件负责销毁它