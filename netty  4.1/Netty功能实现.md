# Netty功能实现

## Netty概述
Netty是一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端。
Netty的优点：
* 性能（比核心Java API更好的吞吐量，较低的延时；得益于共享池和重用，资源消耗更少；减少内存拷贝）
* 易用性（封装了NIO操作的很多细节，提供易于使用的API）
* 安全性 (完整的SSL/TLS的支持)

协议支持：
HTTP、Protobuf、二进制、文本、WebSocket等一系列常见协议都支持。开发者还可以通过编写编码、解码逻辑来实现自定义协议；

## Bootstrap类
AbstractBoostrap处理服务端和客户端的共同引导步骤，特定的步骤由客户端由Bootstrap、服务端ServerBootstrap分别处理。
AbstractBootstrap被标志为Cloneable，在一个已经配置好的bootstrap上调用clone()方法将返回另一个可立即使用的boostrap实例。  
Bootstrap负责创建客户端或使用无连接协议应用的channels(bootstrap的bind()是用于无连接协议的)。  

通过option()添加ChannelOptions到一个引导程序中，将被自动引用于该引导程序创建的所有Channel上。ChannelOptions的可用性包括了底层连接的细节，比如keep-alive或超时属性和缓存设置。  
```java
// ChannelOption
public static final ChannelOption<Boolean> SO_BROADCAST = valueOf("SO_BROADCAST");
public static final ChannelOption<Boolean> SO_KEEPALIVE = valueOf("SO_KEEPALIVE");
public static final ChannelOption<Integer> SO_SNDBUF = valueOf("SO_SNDBUF");
public static final ChannelOption<Integer> SO_RCVBUF = valueOf("SO_RCVBUF");
public static final ChannelOption<Boolean> SO_REUSEADDR = valueOf("SO_REUSEADDR");
public static final ChannelOption<Integer> SO_LINGER = valueOf("SO_LINGER");
public static final ChannelOption<Integer> SO_BACKLOG = valueOf("SO_BACKLOG");
public static final ChannelOption<Integer> SO_TIMEOUT = valueOf("SO_TIMEOUT");

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
.channel(NioServerSocketChannel.class)
.childOption(ChannelOption.SO_KEEPALIVE, true)
```

Netty提供AttributeMap抽象概念，一个由通道和引导类提供的集合，以及AttributeKey <T>，用于插入和检索属性值的通用类。通过这些工具，你能够安全的关联任何类型的数据项到服务端或客户端的Channels。  
例如在WebSocket协议的代码中用来存储Handshaker对象。  
```java
// WebSocketServerProtocolHandler
private static final AttributeKey<WebSocketServerHandshaker> HANDSHAKER_ATTR_KEY =
            AttributeKey.valueOf(WebSocketServerHandshaker.class, "HANDSHAKER");

static void setHandshaker(Channel channel, WebSocketServerHandshaker handshaker) {
        channel.attr(HANDSHAKER_ATTR_KEY).set(handshaker);
}

static WebSocketServerHandshaker getHandshaker(Channel channel) {
        return channel.attr(HANDSHAKER_ATTR_KEY).get();
}

protected void decode(ChannelHandlerContext ctx, WebSocketFrame frame, List<Object> out) throws Exception {
    if (serverConfig.handleCloseFrames() && frame instanceof CloseWebSocketFrame) {
        WebSocketServerHandshaker handshaker = getHandshaker(ctx.channel());
        if (handshaker != null) {
            frame.retain();
            handshaker.close(ctx.channel(), (CloseWebSocketFrame) frame);
        } else {
            ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
        }
        return;
    }
    super.decode(ctx, frame, out);
}

// WebSocketServerProtocolHandshakeHandler
 public void channelRead(final ChannelHandlerContext ctx, Object msg) throws Exception {
        final FullHttpRequest req = (FullHttpRequest) msg;
        if (!isWebSocketPath(req)) {
            ctx.fireChannelRead(msg);
            return;
        }
        try {
            ...
            final WebSocketServerHandshaker handshaker = wsFactory.newHandshaker(req);
            final ChannelPromise localHandshakePromise = handshakePromise;
            if (handshaker == null) {
                WebSocketServerHandshakerFactory.sendUnsupportedVersionResponse(ctx.channel());
            } else {
                WebSocketServerProtocolHandler.setHandshaker(ctx.channel(), handshaker);
                ctx.pipeline().remove(this);

                final ChannelFuture handshakeFuture = handshaker.handshake(ctx.channel(), req);
                ...
            }
        } finally {
            req.release();
        }
    }
```

## ChannelInitializer
Netty提供了一个ChannelInboundHandlerAdapter的特殊子类：
```java
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter
```
该类定义了如下方法：
```
protected abstract void initChannel(C ch) throws Exception
```
ChannelInitializer实际上为Inbound通道处理器，主要目的是为程序员提供了一个简单的工具，用于在某个Channel注册到EventLoop后，对这个Channel执行一些初始化操作。
```java
childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 ChannelPipeline p = ch.pipeline();
                 if (sslCtx != null) {
                     p.addLast(sslCtx.newHandler(ch.alloc()));
                 }
                 p.addLast(
                         new StringEncoder(CharsetUtil.UTF_8),
                         new LineBasedFrameDecoder(8192),
                         new StringDecoder(CharsetUtil.UTF_8),
                         new ChunkedWriteHandler(),
                         new FileServerHandler());
             }
});
```
ChannelInitializer虽然会在一开始会被注册到Channel相关的pipeline里，但是在初始化完成之后，ChannelInitializer会将自己从pipeline中移除，不会影响后续的操作。
```java
// ChannelInitializer
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.add(ctx)) { // Guard against re-entrance.
            try {
                initChannel((C) ctx.channel()); // 抽象方法，由开发者自己实现
            } catch (Throwable cause) {
                // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
                // We do so to prevent multiple calls to initChannel(...).
                exceptionCaught(ctx, cause);
            } finally {
                ChannelPipeline pipeline = ctx.pipeline();
                if (pipeline.context(this) != null) {
                    pipeline.remove(this);   // 将自己从pipeline中移除
                }
            }
            return true;
        }
        return false;
    }
```

ChannelPipeline中添加的一定是handler，所以ChannelInitializer一定是Handler的子类型。最终会生成handler对应的context实例，并把context实例构建成双向链表数据结构。实际上向pipeline中添加的是Context，每个具体的handler有一个与之对应的唯一的Context。
```java
 public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);

            newCtx = newContext(group, filterName(name, handler), handler);

            addLast0(newCtx);
            ...
        }
        callHandlerAdded0(newCtx);
        return this;
    }

 private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
}
```

## ChannelFuture
JDK提供了interface java.util.concurrent.Future，但JDK对Future的实现需要你手动检查是否一个操作已经完成或者堵塞直到操作完成。这是很笨重的，所以Netty提供了它自己的实现————ChannelFuture。
ChannelFuture提供了addListener方法，这个方法允许你注册一个或多个GenericFutureListener实例。“operationComplete()”在操作完成时会被回调。监听者能够确定操作是否成功或失败。如果失败了，我们能够恢复错误。简而言之，GenericFutureListener的通知机制消除了手动检查操作完成的需要,通过GenericFutureListener将future和callback组合起来。
```java
ublic interface ChannelFuture extends Future<Void> {
    Channel channel();
    @Override
    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture addListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelFuture removeListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture removeListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelFuture sync() throws InterruptedException;
    @Override
    ChannelFuture syncUninterruptibly();
    @Override
    ChannelFuture await() throws InterruptedException;
    @Override
    ChannelFuture awaitUninterruptibly();
    boolean isVoid();
}

public interface GenericFutureListener<F extends Future<?>> extends EventListener {
    void operationComplete(F future) throws Exception;
}

```


## EventLoop
EventLoop用于遍历依次执行所有可运行的事件和任务，通过EventExecutor来为事件/任务提供的执行器。而EventExecutor底层则是依赖于JDK的java.util.concurrent包中的Executor来实现执行器的。
```java
// SingleThreadEventExecutor
 private void doStartThread() {
        assert thread == null;
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

public class DefaultEventLoop extends SingleThreadEventLoop {
    ...
    @Override
    protected void run() {
        for (;;) {
            Runnable task = takeTask();
            if (task != null) {
                task.run();
                updateLastExecutionTime();
            }

            if (confirmShutdown()) {
                break;
            }
        }
    }
}

```
一个EventLoop由一个永远不会改变的线程所驱动，并且任务能被直接提交给EventLoop实现立即或定时的执行。  
Event/Task 执行的顺序：事件和任务根据FIFO(先进先出)的顺序被执行。这消除了数据损坏的可能性，因此保证了以正确的顺序处理字节内容。  
```java
// SingleThreadEventExecutor
protected Runnable takeTask() {
        assert inEventLoop();
        if (!(taskQueue instanceof BlockingQueue)) {
            throw new UnsupportedOperationException();
        }

        BlockingQueue<Runnable> taskQueue = (BlockingQueue<Runnable>) this.taskQueue;
        for (;;) {
            // 定时任务
            ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
            if (scheduledTask == null) {
                Runnable task = null;
                try {
                    task = taskQueue.take();
                    if (task == WAKEUP_TASK) {
                        task = null;
                    }
                } catch (InterruptedException e) {
                    // Ignore
                }
                return task;
            } else {
                long delayNanos = scheduledTask.delayNanos();
                Runnable task = null;
                if (delayNanos > 0) {
                    try {
                        task = taskQueue.poll(delayNanos, TimeUnit.NANOSECONDS);
                    } catch (InterruptedException e) {
                        // Waken up.
                        return null;
                    }
                }
                if (task == null) {
                    fetchFromScheduledTaskQueue();
                    task = taskQueue.poll();
                }

                if (task != null) {
                    return task;
                }
            }
        }
    }
```

如果调用线程就是EventLoop所在线程的话，那么执行该代码块。否则，EventLoop安排一个任务用于随后执行并将该任务放到一个内部队列中。当EventLoop下一次处理它的事件时，EventLoop将执行队列中的任务。（这解释了为什么多个线程能够通过Channel直接交互而不用在ChannelHandler中进行同步操作）
```java
static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }
```

在Channel的生命周期内由EventLoop负责处理一个Channel的所有事件。
ChannelHandlerContext持有ChannelPipeline，而ChannelPipeline持有Channel。
```java
    public Channel channel() {
        return pipeline.channel();
    }

    @Override
    public ChannelPipeline pipeline() {
        return pipeline;
    }

    @Override
    public EventExecutor executor() {
        if (executor == null) {
            return channel().eventLoop();
        } else {
            return executor;
        }
    }
```

## EventLoopGroup
服务I/O和Channels事件的EventLoops包含在一个EventLoopGroup里。EventLoops被创建和分配的方式根据传输实现而有所不同。
在NIO模式下，EventLoop在EventLoopGroup创建的时候就分配好了，EventLoop的个数是固定的。
```java
 protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```
NioServerSocketChannel通过反射被创建，通过EventLoopGroup注册到EventLoop。（ServerBootstrap则通过parentGroup注册）
```java
//AbstractBootstrap
   public B channel(Class<? extends C> channelClass) {
        return channelFactory(new ReflectiveChannelFactory<C>(
                ObjectUtil.checkNotNull(channelClass, "channelClass")
        ));
    }

    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        ObjectUtil.checkNotNull(channelFactory, "channelFactory");
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }

        this.channelFactory = channelFactory;
        return self();
    }
  // 创建Channel并注册到EventLoop（server）
 final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        ...
        return regFuture;
    }
```
EventLoopGroup实现了EventExcutor接口，通过上层父类MultithreadEventExcutorGroup的构造方法创建事件执行器，使用其中的excutor来创建线程并且执行。Excutor线程池本质上就是创建多个NioEventLoop时间执行器。
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) { //  如果执行器为空，则创建一个
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```

```java
// NioEventLoopGroup
 @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }
```

NioEventLoop的实现就是NIO中的selector增强，本质还是selector，而NioEventLoop是不能创建线程的，这里用到了Excutor来创建线程。在线程run方法中会轮训执行selector.select方法和taskqueue里的内容，每个EventLoop对应一个轮询线程。eventLoop是不会自己执行线程的，只有当有任务提交时才触发eventloop，如果eventloop想要自己执行任务，那么就需要使用execute方法来提交一个runnable。  
```java
    @Override
    protected void run() {
        int selectCnt = 0;
        // 执行两件事selector,select的事件 和 taskQueue里面的内容
        for (;;) {
            try {
                int strategy;
                try {
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
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
                            processSelectedKeys();
                        }
                    } finally {
                        // Ensure we always run tasks.
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {
                // Harmless exception - log anyway
                if (logger.isDebugEnabled()) {
                    logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                            selector, e);
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

## 事件
Netty使用不同的事件来通知我们关于状态的变化或者操作的状态。这允许我们基于事件的发生触发适当的操作。
Netty将事件划分为入站数据流和出站数据流  
入站操作或者一个相关的状态变化包括：  
* 连接状态的活跃(active)或不活跃(inactive)
* 数据的读取
* 用户事件触发
* 错误事件触发

```java
public interface ChannelInboundHandler extends ChannelHandler {
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
    void channelActive(ChannelHandlerContext ctx) throws Exception;
    void channelInactive(ChannelHandlerContext ctx) throws Exception;
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
    @Override
    @SuppressWarnings("deprecation")
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
```
一个出站事件是一个Future中动作的结果。出站事件包括：  
* 开启或关闭一个远端连接
* 写或刷新数据到套接字中
```java
public interface ChannelOutboundHandler extends ChannelHandler {
    void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void connect(
            ChannelHandlerContext ctx, SocketAddress remoteAddress,
            SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void read(ChannelHandlerContext ctx) throws Exception;
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
    void flush(ChannelHandlerContext ctx) throws Exception;
}
```

## 分发
ServerBootstrapAcceptor在Channel初始化创建并加入到pipeline中。
```java
// 反射创建的Channel初始化
 @Override
    void init(Channel channel) {
        setChannelOptions(channel, newOptionsArray(), logger);
        setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
        }
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```
ServerBootstrapAcceptor在channelRead事件中添加childHandler并通过childGroup注册到EventLoop。
```java
@Override
@SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
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
NioServerSocketChannel处理连接并创建NioSocketChannel，然后通过pipeline.fireChannelRead将事件传递给ServerBootstrapAcceptor。
```java
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

                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;
                    pipeline.fireChannelRead(readBuf.get(i));
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
    }


protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
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

## ByteBuf
ByteBuf是为了解决Java NIO ByteBuffer的问题和满足网络应用程序开发人员日常需求而设计的。  
JDK ByteBuffer缺点：  
1.无法动态扩容，长度固定，不能动态拓展和收缩，当数据大于ByteBuffer容量时，会引发索引越界。  
2.API使用复杂  
读写切换的时候需要手动调用flip和rewind等方法，需要谨慎使用，否则容易出错。  
ByteBuf对于ByteBuffer的增强:  
1.API操作便捷性  
2.动态扩容  
3.多种ByteBuf实现  
4.高效零拷贝机制  
ByteBuf三个重要特性：capacity（容量）、readerindex（读取位置）、writeindex（写入位置）。readerindex和writeindex这两个指针可以支持顺序读写操作。  
```
    +-------------------+------------------+------------------+
    | discardable bytes |  readable bytes  |  writable bytes  |
    |                   |     (CONTENT)    |                  |
    +-------------------+------------------+------------------+
    |                   |                  |                  |
    0      <=       readerIndex   <=   writerIndex    <=    capacity
```
堆内内存：  
由UnpooledHeapByteBuf实现，本质是数组，对于数组的一个封装。  
堆外内存：  
由UnpooledDirectByteBuf实现，本质是NIO的bytebuffer，堆外内存输出内容时不能使用buf.array()方法，netty和javaNIO都没有实现此方法。其他方法使用都与堆内内存一样。  
netty中默认使用的是pooledUnsafeDirectByteBuf，pool目的是为了复用，Unsafe的目的是为了性能提升，Direct还是为了性能提升，所以netty才能实现高效高性能的特性。而netty建议开发者使用unpooledHeapByteBuf。  

零拷贝机制：  
零拷贝机制是一种应用层的实现。和JVM、操作系统内存机制并无过多关联。
1.CompositeByteBuf，将多个ByteBuf合并为一个逻辑上的ByteBuf，避免了各个ByteBuf之间的拷贝。  
2.wrapedBuffer（）方法，将byte[]数组包装成ByteBuf对象。  
3.slice（）方法，将一个ByteBuf对象切分成多个ByteBuf对象。  
所谓零拷贝就是不改变原buf只是逻辑上做拆分、合并、包装。减少了大量的内存复制，由此提升性能。  

### 关闭
关闭Netty应用，首先需要关闭EventLoopGroup，将处理所有等待执行的事件和任务，随后释放所有的活跃线程。
调用EventLoopGroup.shutdownGracefully()关闭应用，该调用将返回一个Future，该future在关闭完成时会进行通知。
```java
// Shut down all event loops to terminate all threads.
bossGroup.shutdownGracefully();
workerGroup.shutdownGracefully();
```

调用EventLoopGroup.shutdownGracefully()方法前可以对所有活动的channel显示的调用Channel.close()方法来释放channel的资源。但是在所有情况下，都要关闭EventLoopGroup本身。  

```java
MultithreadEventLoopGroup 继承 MultithreadEventExecutorGroup
MultithreadEventExecutorGroup 继承 AbstractEventExecutorGroup

DefaultEventLoop 继承 SingleThreadEventLoop 
SingleThreadEventLoop 继承 SingleThreadEventExecutor

// AbstractEventExecutor
static final long DEFAULT_SHUTDOWN_QUIET_PERIOD = 2;
static final long DEFAULT_SHUTDOWN_TIMEOUT = 15;

// AbstractEventExecutorGroup
@Override
public Future<?> shutdownGracefully() {
    return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD, DEFAULT_SHUTDOWN_TIMEOUT, TimeUnit.SECONDS);
}

// MultithreadEventExecutorGroup
@Override
    public Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit) {
        for (EventExecutor l: children) {
            l.shutdownGracefully(quietPeriod, timeout, unit);
        }
        return terminationFuture();
    }

// SingleThreadEventExecutor
@Override
    public Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit) {
        ObjectUtil.checkPositiveOrZero(quietPeriod, "quietPeriod");
        if (timeout < quietPeriod) {
            throw new IllegalArgumentException(
                    "timeout: " + timeout + " (expected >= quietPeriod (" + quietPeriod + "))");
        }
        ObjectUtil.checkNotNull(unit, "unit");

        if (isShuttingDown()) {
            return terminationFuture();
        }

        boolean inEventLoop = inEventLoop();
        boolean wakeup;
        int oldState;
        for (;;) {
            if (isShuttingDown()) {
                return terminationFuture();
            }
            int newState;
            wakeup = true;
            oldState = state;
            if (inEventLoop) {
                newState = ST_SHUTTING_DOWN;
            } else {
                switch (oldState) {
                    case ST_NOT_STARTED:
                    case ST_STARTED:
                        newState = ST_SHUTTING_DOWN;
                        break;
                    default:
                        newState = oldState;
                        wakeup = false;
                }
            }
            if (STATE_UPDATER.compareAndSet(this, oldState, newState)) {
                break;
            }
        }
        gracefulShutdownQuietPeriod = unit.toNanos(quietPeriod);
        gracefulShutdownTimeout = unit.toNanos(timeout);

        if (ensureThreadStarted(oldState)) {
            return terminationFuture;
        }

        if (wakeup) {
            taskQueue.offer(WAKEUP_TASK);
            if (!addTaskWakesUp) {
                wakeup(inEventLoop);
            }
        }

        return terminationFuture();
    }
```
