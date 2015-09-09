Jetty 4.x 增加了Pooled Buffer，实现了高性能的buffer池，分配策略则是结合了buddy allocation和slab allocation的jemalloc变种(其实我也不懂...)，代码在io.netty.buffer.PoolArena。

官方说提供了以下优势：

- 频繁分配、释放buffer时减少了GC压力；
- 在初始化新buffer时减少内存带宽消耗（初始化时不可避免的要给buffer数组赋初始值）；
- 及时的释放direct buffer

这块内容太细,代码量太大,暂时还没有深入研读.


