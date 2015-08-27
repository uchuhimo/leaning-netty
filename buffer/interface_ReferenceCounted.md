接口ReferenceCounted用于实现引用计数,以便显式回收.

定义在package io.netty.util中:

```
package io.netty.util;

public interface ReferenceCounted {}
```

# 原理

ReferenceCounted的工作原理: 当引用计数对象被初始化时, 引用计数器从1开始计数. retain()方法增加引用计数,而release()方法减少引用计数.如果引用计数被减到0,则这个对象就将被显式回收,而访问被回收的对象将通常会导致访问违规.

如果一个实现引用计数的对象是一个包含其他实现了引用计数的容器, 那么当容器的引用计数变成0时, 被包含的对象也应该通过release()方法释放一个引用.

# 方法定义

接口ReferenceCounted中定义了以下方法:

- int refCnt()
    返回对象的引用计数.**如果返回0,意味着对象已经被回收.**
- ReferenceCounted retain()
	将引用计数增加1
- ReferenceCounted retain(int increment)
	将引用计数增加指定数量
- boolean release()
	将引用计数减一, 如果引用计数达到0则回收这个对象.
    **注意: 返回的boolean值, 当且仅当引用计数变成0并且这个对象被回收才返回true.**
- boolean release(int decrement)
	同上,将引用计数减少指定数量
- ReferenceCounted touch(Object hint)
	出于调试目的,用一个额外的任意的(arbitrary)信息记录这个对象的当前访问地址. 如果这个对象被检测到泄露了, 这个操作记录的信息将通过ResourceLeakDetector提供.
- ReferenceCounted touch()
	这个方法等价于touch(null).

注意除了refCnt()方法之外,其他的几个方法都是返回ReferenceCounted对象. 实现中一般时返回当前对象本身,以便实现链式(train)调用.