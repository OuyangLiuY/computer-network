# Netty核心原理

### 基本说明

1. 只有看过Netty源码,才能说是真的掌握了Netty框架
2. 在io.netty.example.echo包下,有很多netty源码案例,可以用来分析
3. 源码分析章节 需要一定的java基础

## 启动原理

**目的:** 用源码分析的方式走一下netty(服务器)的启动过程,更好的理解Netty的整体设计和运行机制



## BoosEventLoop

 BossEventLoop职责

BossEventLoop主要负责：

- 监听服务器端口

- 接受客户端连接

- 将新连接分配给WorkerEventLoop

```java
// NioEventLoop的run方法 - 事件循环核心
@Override
protected void run() {
    for (;;) {
        try {
            // 1. 选择就绪的I/O事件
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    break;
            }
            
            // 2. 处理I/O事件
            processSelectedKeys();
            
            // 3. 处理任务队列中的任务
            runAllTasks();
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

## WorkerEventLoop

WorkerEventLoop主要负责：

- 处理客户端连接的数据读写

- 执行Pipeline中的Handler

- 处理业务逻辑

### 事件处理流程

```java
// 1. BossEventLoop监听到连接事件
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    int readyOps = k.readyOps();
    
    // 处理连接事件
    if ((readyOps & SelectionKey.OP_ACCEPT) != 0) {
        unsafe.read();  // 触发accept操作
    }
}

// 2. 接受新连接并分配给WorkerEventLoop
// 在NioServerSocketChannel中
@Override
protected void doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());
    if (ch != null) {
        buf.add(new NioSocketChannel(this, ch));
    }
}
```

### 数据读取流程

```java
// 1. WorkerEventLoop监听到读事件
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    int readyOps = k.readyOps();
    
    // 处理读事件
    if ((readyOps & SelectionKey.OP_READ) != 0) {
        unsafe.read();  // 触发读操作
    }
}

// 2. 读取数据到ByteBuf
// 在AbstractNioByteChannel.NioByteUnsafe中
@Override
public final void read() {
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    
    ByteBuf byteBuf = null;
    try {
        do {
            byteBuf = allocHandle.allocate(allocator);
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            
            if (allocHandle.lastBytesRead() <= 0) {
                byteBuf.release();
                byteBuf = null;
                break;
            }
            
            // 3. 触发Pipeline中的channelRead事件
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());
        
        // 4. 触发channelReadComplete事件
        pipeline.fireChannelReadComplete();
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    }
}
```

## PipeLine 事件传播

### Pipeline结构

```java
// DefaultChannelPipeline的结构
final AbstractChannelHandlerContext head;  // 头节点
final AbstractChannelHandlerContext tail;  // 尾节点

// Pipeline中的Handler链
head -> handler1 -> handler2 -> ... -> handlerN -> tail
```

### 事件传播机制

```java
// 事件传播的核心方法
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
    return this;
}

static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(msg);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(msg);
            }
        });
    }
}

private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

## 编解码流程

### 解码器 (ByteToMessageDecoder)

```java
// 解码器处理流程
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) {
                cumulation = data;
            } else {
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            
            // 调用具体的解码方法
            callDecode(ctx, cumulation, out);
            
            // 传播解码后的消息
            fireChannelRead(ctx, out, numElements);
        } finally {
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}

// 具体的解码逻辑由子类实现
protected abstract void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception;
```



### 编码器 (MessageToByteEncoder)

```java
// 编码器处理流程
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        if (acceptOutboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                // 调用具体的编码方法
                encode(ctx, cast, buf);
            } finally {
                ReferenceCountUtil.release(cast);
            }

            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}

// 具体的编码逻辑由子类实现
protected abstract void encode(ChannelHandlerContext ctx, I msg, ByteBuf out) throws Exception;
```



## 完整的事件处理流程图

```text
客户端连接请求
    ↓
BossEventLoop.select() - 监听连接事件
    ↓
BossEventLoop.processSelectedKey() - 处理OP_ACCEPT
    ↓
NioServerSocketChannel.doReadMessages() - 接受连接
    ↓
分配新连接给WorkerEventLoop
    ↓
WorkerEventLoop.select() - 监听数据事件
    ↓
WorkerEventLoop.processSelectedKey() - 处理OP_READ
    ↓
AbstractNioByteChannel.read() - 读取数据
    ↓
Pipeline.fireChannelRead() - 触发读事件
    ↓
ByteToMessageDecoder.channelRead() - 解码
    ↓
业务Handler.channelRead() - 业务处理
    ↓
MessageToByteEncoder.write() - 编码
    ↓
Channel.write() - 写入数据
    ↓
AbstractNioByteChannel.write() - 发送数据
```

## 详细注册流程

一个请求如何注册到WorkEventLoop中的？

### BossEventLoop监听连接事件

```java
// NioEventLoop.run() - BossEventLoop的事件循环
@Override
protected void run() {
    for (;;) {
        try {
            // 1. 选择就绪的I/O事件
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    break;
            }
            
            // 2. 处理I/O事件
            processSelectedKeys();
            
            // 3. 处理任务队列中的任务
            runAllTasks();
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}

// 处理具体的事件
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    int readyOps = k.readyOps();
    
    // 处理连接事件 (OP_ACCEPT)
    if ((readyOps & SelectionKey.OP_ACCEPT) != 0 || readyOps == 0) {
        unsafe.read();  // 触发read操作，对于ServerSocketChannel就是accept
    }
}
```



### ServerSocketChannel接受新连接

```java
// NioServerSocketChannel.doReadMessages() - 接受新连接
@Override
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            // 创建新的NioSocketChannel，这是关键步骤
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```

### AbstractNioMessageChannel处理消息

```java
// AbstractNioMessageChannel.NioMessageUnsafe.read() - 处理消息
@Override
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                // 调用具体的doReadMessages方法
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }

        // 关键：将新创建的Channel传递给Pipeline
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i));  // 触发channelRead事件
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            closed = closeOnReadError(exception);
            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            }
        }
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```



### Pipeline事件传播到ServerBootstrapAcceptor

```java
// DefaultChannelPipeline.fireChannelRead() - 事件传播
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

// AbstractChannelHandlerContext.invokeChannelRead() - 调用Handler
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(msg);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(msg);
            }
        });
    }
}

// 最终调用到ServerBootstrapAcceptor.channelRead()
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

> **channelRead最终会调用的是ServerBootstrapAcceptor的channelRead将channel注册到child中**

###  ServerBootstrapAcceptor处理新连接

真正的注册请求：到这里当前的事件还是select的connect事件。

```java
// ServerBootstrap.ServerBootstrapAcceptor.channelRead() - 核心处理逻辑
@Override
@SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;  // 新创建的NioSocketChannel

    // 1. 为新Channel添加childHandler
    child.pipeline().addLast(childHandler);

    // 2. 设置Channel选项和属性
    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        // 3. 关键：将新Channel注册到WorkerEventLoop
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

### WorkerEventLoop选择策略

到这里会掉用的是MultithreadEventLoopGroup.register() 用于 选择WorkerEventLoop。

默认使用的轮询策略来选择WorkerEventLoop

```java
// MultithreadEventLoopGroup.register() - 选择WorkerEventLoop
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);  // 通过next()选择下一个EventLoop
}

// MultithreadEventExecutorGroup.next() - 使用选择器
@Override
public EventExecutor next() {
    return chooser.next();  // 使用EventExecutorChooser选择
}

// DefaultEventExecutorChooserFactory - 默认的轮询选择策略
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}

// PowerOfTwoEventExecutorChooser - 优化的轮询选择
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    private final AtomicInteger idx = new AtomicInteger();
    private final EventExecutor[] executors;

    @Override
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
}
```

### 新Channel注册到WorkerEventLoop

register事件。

在SingleThreadEventLoop中调用register方法进行注册，并绑定promise回调事件，并调用pipeline的fireChannelRegistered事件。

```java
// SingleThreadEventLoop.register() - 注册Channel到EventLoop
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}

// AbstractChannel.register() - 实际的注册逻辑
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;
    
    // 在EventLoop线程中执行注册
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}

// 实际的注册操作
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();  // 模板方法，子类实现具体的注册逻辑
        neverRegistered = false;
        registered = true;

        // 确保在通知promise之前调用handlerAdded
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        
        // 如果是首次注册且Channel是活跃的，触发channelActive事件
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

## 完整的Boos到Worker的注册流程

```java
客户端连接请求
    ↓
BossEventLoop.select() - 监听OP_ACCEPT事件
    ↓
BossEventLoop.processSelectedKey() - 处理连接事件
    ↓
NioServerSocketChannel.doReadMessages() - 调用accept()
    ↓
创建NioSocketChannel实例
    ↓
AbstractNioMessageChannel.read() - 读取消息
    ↓
pipeline.fireChannelRead(newChannel) - 触发channelRead事件
    ↓
ServerBootstrapAcceptor.channelRead() - 接收新Channel
    ↓
childGroup.register(child) - 选择WorkerEventLoop
    ↓
EventLoopGroup.next() - 使用轮询策略选择
    ↓
WorkerEventLoop.register() - 注册到WorkerEventLoop
    ↓
AbstractChannel.register0() - 执行注册
    ↓
pipeline.fireChannelActive() - 触发channelActive事件
    ↓
WorkerEventLoop开始处理该Channel的后续事件
```

