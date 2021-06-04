---
title: NioEventLoopGroup
---
![NioEventLoopGroup类继承关系](https://inus-markdown.oss-cn-beijing.aliyuncs.com/img/20201224112405121.png)

> `EventExecutorGroup` 主要职责是管理和治理所属的 `EventExecutor` , 通过 `next()` 获取下一个 `EventExecutor` 执行器执行提交的事件, 本质就是线程池
>
> `AbstractEventExecutorGroup` 是 `EventExecutorGroup` 抽象实现类,  `next()` 和 `shutdown功能` 的方法没有实现
>
> `MultithreadEventExecutorGroup` 是 `AbstractEventExecutorGroup` 继承类, 职责和 `EventExecutorGroup`, 内部定义了 `newChild` 创建被治理的 `EventExecutor` 的方法, 该事件执行组各个执行主要是 `EventExecutor`
>
> `MultithreadEventLoopGroup` 是 `MultithreadEventExecutorGroup` 继承类, 执行真正执行者是 `EventLoop` 
>
> `NioEventLoopGroup` 主要职责是 `EventLoop` 是治理, 定义创建被治理的 `EventLoop` 方法

> `EventExecutorGroup` 本质就是线程池, 治理事件执行器, 包括事件执行器的创建, 关闭和分配事件执行



![NioEventLoop继承关系图](https://inus-markdown.oss-cn-beijing.aliyuncs.com/img/2020070416155161.png)

> `NioEventLoop` 是事件真正的 `NioEvent` 执行者



> `Netty`中每创建一个`Channel`都会连带创建一个`ChannelPipeline` 和 `Netty的Unsafe`, 主要是拦截`Nio`事件和处理`Channel`的`IO`数据

`NioServerSocketChannel` 内部都包含一个单独的多路复用器, 主要代码如下, 

![image-20210603113004202](http://inus-markdown.oss-cn-beijing.aliyuncs.com/img/image-20210603113004202.png)
