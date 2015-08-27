Jetty 4.x 增加了Pooled Buffer，实现了高性能的buffer池，分配策略则是结合了buddy allocation和slab allocation的jemalloc变种(其实我也不懂...)，代码在io.netty.buffer.PoolArena。

