Channel Pipeline
================

netty中和Servlet Filter机制类似, 实现了的Channel的过滤器, 具体就是ChannelPipeline和ChannelHandler.

- ChannelHandler: 负责对I/O事件进行拦截和处理
- ChannelPipeline: ChannelHandler的容器, 管理ChannelHandler

# Pipeline的构建

通常不需要用户自己构建Pipeline, 在使用ServerBootstrap或者Bootstrap时, ServerBootstrap或者Bootstrap会为每个Channel创建一个独立的Pipeline.

# 代码分析

ChannelPipeline的代码不多,继承关系简单, 就一个ChannelPipeline接口和一个默认实现DefaultChannelPipeline.

```uml
@startuml

ChannelPipeline <|- DefaultChannelPipeline

@enduml
```