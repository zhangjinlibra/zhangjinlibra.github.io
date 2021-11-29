---
title: NioEventLoopGroup和NioEventLoop的工作流程
categories: Netty
tags: NioEventLoop
---



### NioEventLoopGroup 

![NioEventLoopGroup类继承关系](https://inus-markdown.oss-cn-beijing.aliyuncs.com/img/20201224112405121.png)

> `EventExecutorGroup` 主要职责是管理和治理所属的 `EventExecutor` , 通过 `next()` 获取下一个 `EventExecutor` 执行器执行提交的事件, 本质就是线程池
>
> `AbstractEventExecutorGroup` 是 `EventExecutorGroup` 抽象实现类,  主题功能基本实现, `next()` 和 `shutdown相关` 交给具体的实现类来实现
>
> `MultithreadEventExecutorGroup` 是 `AbstractEventExecutorGroup` 继承类, 职责和 `EventExecutorGroup`, 内部定义了 `newChild` 创建被治理的 `EventExecutor` 的方法, 该事件执行组各个执行主要是 `EventExecutor`
>
> `MultithreadEventLoopGroup` 是 `MultithreadEventExecutorGroup` 继承类, 执行真正执行者是 `EventLoop` 
>
> `NioEventLoopGroup` 主要职责是 `EventLoop` 是治理, 定义创建被治理的 `EventLoop` 方法

> `EventExecutorGroup` 本质就是线程池, 治理事件执行器, 包括事件执行器的创建, 关闭和分配事件执行



#### NioEventLoopGroup是如何初始化NioEventLoop线程组的

> `NioEventLoop` 的父类 `SingleThreadEventExecutor` 中方法 ` execute(Runnable task) ` 就是线程执行任务的方法

```java
@Override
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    // 执行指定任务
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}

private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();// 检查当前线程是否是NioEventLoop的线程
    addTask(task);// 添加任务
    if (!inEventLoop) {
        startThread();// 开始线程, 看下面该方法的具体实现
        if (isShutdown()) {
            ...
        }
    }

    // 唤醒线程立即执行
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}

private void startThread() {
    if (state == ST_NOT_STARTED) {// 线程未启动, 设置标志位, 启动线程
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();// 下面是该方法的具体实现
                success = true;
            } finally {
                // 启动失败, 重置标志位
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}

private void doStartThread() {
    assert thread == null;
    // 该处是如何创建线程并执行的, 看下面的代码
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 单例线程池执行run()方法, 该方法就是NioEventLoop中定义的run方法, 线程启动完毕
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                ... 
            }
        }
    });
}

// 初始化NioEventLoopGroup时, 构建MultithreadEventExecutorGroup对象时有下面的一段代码
if (executor == null) {
    // 构造线程单任务执行器, 该线程自始至终值运行一个方法的执行器
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
}
// 看一下线程工厂DefaultThreadFactory创建新线程的代码
public Thread newThread(Runnable r) {
    // 创建新的线程
    Thread t = newThread(FastThreadLocalRunnable.wrap(r), prefix + nextId.incrementAndGet());
    try {
        if (t.isDaemon() != daemon) {
            t.setDaemon(daemon);
        }

        if (t.getPriority() != priority) {
            t.setPriority(priority);
        }
    } catch (Exception ignored) {
        // Doesn't matter even if failed to set.
    }
    return t;
}
```





### NioEventLoop

![NioEventLoop继承关系图](https://inus-markdown.oss-cn-beijing.aliyuncs.com/img/2020070416155161.png)

> `NioEventLoop` 是事件真正的 `NioEvent` 执行者



> `Netty`中每创建一个`Channel`都会连带创建一个`ChannelPipeline` 和 `Netty的Unsafe`, 主要是拦截`Nio`事件和处理`Channel`的`IO`数据

`NioServerSocketChannel` 内部都包含一个单独的多路复用器 `SelectorProvider`, `SelectorProvider` 是nio的java原生类, 主要职责是创建多路复用器和Channel, 具体代码如下

![image-20210603113004202](http://inus-markdown.oss-cn-beijing.aliyuncs.com/img/image-20210603113004202.png)

`NioEventLoop` 中定义的`run()` 是当前线程事件轮询器的主要执行的方法, 具体执行的代码如如下

```java
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                // 计算执行策略, 如果队列中还有任务, 则执行任务, 无任务则SELECT等待任务
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        // 阻塞等待IO事件
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                        // 优先处理io任务
                        processSelectedKeys();
                    }
                } finally {
                    // 执行非io任务
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                // io开始时间
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // io结束时间
                    final long ioTime = System.nanoTime() - ioStartTime;
                    // 按照比例执行非io任务
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                // 没有io任务, 全力执行非io任务
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.", selectCnt - 1, selector);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?", selector, e);
            }
        } catch (Error e) {
            throw (Error) e;
        } catch (Throwable t) {
            handleLoopException(t);
        } finally {
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Error e) {
                throw (Error) e;
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}
```

#### NioEventLoop中是如何处理io事件的

重点看该类的中的下面一段代码

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    // 每个Channel中都存在一个Netty的Unsafe, 该处返回的是NioMessageUnsafe, 主要值处理io事件
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // 连接事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // 写Channel
            ch.unsafe().forceFlush();
        }

        // 读Channel
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();// 该方法中会触发pipeline.fireChannelRead(readBuf.get(i));事件, 随后事件进入ChannelPipeline中进行链式传播
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

#### DefaultChannelPipeline是如何传播io事件的

着重看用下面的一段代码

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    // 头指针
    final AbstractChannelHandlerContext head;
    // 尾指针
    final AbstractChannelHandlerContext tail;
    
    @Override
    public final ChannelPipeline fireChannelRead(Object msg) {
        // 将消息传播到head context中进行处理, 后面的Context中的handler需要主动调用ctx.fireChannelRead(msg);向下传播消息
        AbstractChannelHandlerContext.invokeChannelRead(head, msg);
        return this;
    }
    
    @Override
    public final ChannelFuture write(Object msg) {
        // 出栈消息是从尾部向前传播
        return tail.write(msg);
    }
}
```

