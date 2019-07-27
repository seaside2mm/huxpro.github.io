# 第四章 NioEventLoop事件循环



## 两条线索

两条线索（这里使用自下而上）：NioEventLoop以及NioEventLoopGroup即线程和线程池。

```java
//类关系
NioEventLoop --> SingleThreadEventLoop --> SingleThreadEventExecutor -->
AbstractScheduledEventExecutor --> AbstractScheduledEventExecutor --> 
AbstractEventExecutor --> AbstractExecutorService

NioEventLoopGroup --> MultithreadEventLoopGroup --> 
MultithreadEventExecutorGroup --> AbstractEventExecutorGroup
```

使用自顶向下的方法，从类图顶部向下、从线程池到线程分析。

### NioEventLoopGroup线

#### EventExecutorGroup

在类图中处于承上启下的位置，其上是Java原生的接口和类，其下是Netty新建的接口和类

```java
interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor> 
boolean isShuttingDown();
Future<?> shutdownGracefully();
Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
Future<?> terminationFuture();  //返回线程池终止时的异步结果
EventExecutor next();  //从线程池中选择一个线程

/* 1 */
//Executor接口 提供了异步执行任务
// 一个方法：Executes the given command at some time in the future
void execute(Runnable command);

/* 2 */
interface ExecutorService extends Executor 
void shutdown();
List<Runnable> shutdownNow();
boolean isShutdown();
boolean isTerminated();
boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
//execute()方法的扩展，相较于execute不关心执行结果，submit返回一个异步执行结果Future。
//但这里的Future不提供回调操作，显得很鸡肋，所以Netty将Java原生的java.util.concurrent.Future扩展为io.netty.util.concurrent.Future
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);

/* 3 */
interface ScheduledExecutorService extends ExecutorService
//调度任务使任务在延迟一段时间后执行
ScheduledFuture<?> schedule(Runnable command,long delay, TimeUnit unit);
<V> ScheduledFuture<V> schedule(Callable<V> callable,long delay, TimeUnit unit);
//延迟一段时间后以固定频率执行任务
ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,
                                       long period,TimeUnit unit);
//延迟一段时间后以固定延时执行任务
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay,
                                          long delay,TimeUnit unit);

```

`abstractEventExecutorGroup`实现了`EventExecutorGroup`接口的大部分方法，实现都长的和下面的差不多：

```java
//当线程池执行一个任务或命令时，步骤是这样的：(1).找一个线程。(2).交给线程执行。
@Override
public void execute(Runnable command) {
    next().execute(command);
}
```

#### MultithreadEventExecutorGroup

实现了线程的创建和线程的选择

```java
// 线程池，数组形式可知为固定线程池
private final EventExecutor[] children;
// 线程索引，用于线程选择
private final AtomicInteger childIndex = new AtomicInteger();
// 终止的线程个数
private final AtomicInteger terminatedChildren = new AtomicInteger();
// 线程池终止时的异步结果
private final Promise<?> terminationFuture = 
    new DefaultPromise(GlobalEventExecutor.INSTANCE);
// 线程选择器
private final EventExecutorChooser chooser;
```



```java
//构造方法详细见下->创建方法
protected MultithreadEventExecutorGroup(int nThreads, 
                                        ThreadFactory threadFactory, Object... args)
```



#### MultithreadEventLoopGroup

实现了EventLoopGroup接口的方法，EventLoopGroup接口作为Netty并发的关键接口，我们看其中扩展的方法

```java
interface EventLoopGroup extends EventExecutorGroup
// 将通道channel注册到EventLoopGroup中的一个线程上
ChannelFuture register(Channel channel);
// 返回的ChannelFuture为传入的ChannelPromise
ChannelFuture register(Channel channel, ChannelPromise promise);
// 覆盖父类接口的方法，返回EventLoop，
@Override EventLoop next();

//MultithreadEventLoopGroup具体实现
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
// 使用上述的选择器选择一个线程
@Override
public EventLoop next() {
    return (EventLoop) super.next();
}

//线程数的默认设置
//默认情况，线程数最小为1，如果配置了系统参数io.netty.eventLoopThreads，设置为该系统参数值，否则设置为核心数的2倍。
DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
 "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));
```



#### NioEventLoopGroup

```java
//创建线程池中的单个线程
class NioEventLoopGroup extends MultithreadEventLoopGroup
@Override
protected EventExecutor newChild(ThreadFactory threadFactory, Object... args) {
    return new NioEventLoop(this, threadFactory, (SelectorProvider) args[0],
((SelectStrategyFactory) args[1]).newSelectStrategy(),  (RejectedExecutionHandler) args[2]);
}
//NioEventLoopGroup还提供了setIoRatio()和rebuildSelectors()两个方法，一个用来设置I/O任务和非I/O任务的执行时间比，一个用来重建线程中的selector来规避JDK的epoll 100% CPU Bug。其实现也是依次设置各线程的状态，故不再列出。
```

### NioEventLoop线

#### AbstractExecutorService

`AbstractExecutorService`是JDK并发包中的类，实现了ExecutorService中的`submit()`和`invoke***()`方法，关键实现是其中的`newTaskFor()`方法，使用FutureTask包装一个Ruannble对象和结果或者一个Callable对象。注意，这个方法是一个protected方法，子类中可以覆盖这个实现。

```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

#### AbstractEventExecutor

AbstractEventExecutor继承自AbstractExecutorService并实现了EventExecutor接口，该类中只实现了一些简单的方法

```java
abstract class AbstractEventExecutor extends AbstractExecutorService implements EventExecutor 
//next()方法在线程池的讲解中已经接触过，功能是选择线程池中的一个线程，将AbstractEventExecutor看为只有一个线程的线程池，所以next()返回它本身
public EventExecutor next() {
    return this;
}    

public boolean inEventLoop() {
    return inEventLoop(Thread.currentThread());
}

public Future<?> shutdownGracefully() {
    return shutdownGracefully(2, 15, TimeUnit.SECONDS);
}
//AbstractEventExecutor类中有四个创建异步结果的方法,实现类似如下
public <V> Promise<V> newPromise() {
    return new DefaultPromise<V>(this);
}
//覆盖了父类的newTaskFor()方法,使用Netty的PromiseTask代替JDK的FutureTask
//此外，还用Netty的Future对象覆盖了subimt()方法的返回值(原本为JDK的Future).
@Override
protected final <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new PromiseTask<T>(this, callable);
}
```

#### AbstractScheduledEventExecutor

#### SingleThreadEventExecutor

一个单线程的Executor实现

```java
private static final int ST_NOT_STARTED = 1;    // 没有启动
private static final int ST_STARTED = 2;    // 启动
private static final int ST_SHUTTING_DOWN = 3;  // 正在关闭
private static final int ST_SHUTDOWN = 4;   // 关闭
private static final int ST_TERMINATED = 5; // 终止

//构造方法核心
thread = threadFactory.newThread(() -> {
    updateLastExecutionTime();
    try {
        SingleThreadEventExecutor.this.run();   // 这是一个模板方法
    } catch (Throwable t) {
        logger.warn("Unexpected exception from an event executor: ", t);
    } finally {
        // shutdown
    }
});
taskQueue = newTaskQueue(); // 这里使用该方法是为了子类可以优化
```





#### SingleThreadEventLoop

SingleThreadEventLoop终于与Channel取得联系，其中最重要的便是register()方法，功能是将一个Channel对象注册到EventLoop上，其最终实现委托Channel对象的Unsafe对象完成

```java
//Abstract base class for {@link EventLoop}s that execute all its submitted tasks in a single thread.
abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop

@Override
public ChannelFuture register(Channel channel) {
    return register(channel, new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
    // 代码中省略了NullPointer检查
    channel.unsafe().register(this, promise);
    return promise;
}

//覆盖了父类的wakesUpForTask(Runnable task)方法
@Override
protected boolean wakesUpForTask(Runnable task) {
    return !(task instanceof NonWakeupRunnable);
}

// 标记接口，用于标记不唤醒原生线程的任务
interface NonWakeupRunnable extends Runnable { }
```



#### NioEventLoop

NioEventLoop的功能是对注册到其中的Channnel的就绪事件以及对用户提交的任务进行处理



```java
Selector selector;  // NIO中的多路复用器Selector
private SelectedSelectionKeySet selectedKeys;   // 就绪事件的键值对，优化时使用
private final SelectorProvider provider;    // selector的工厂
// 唤醒标记，由于select()方法会阻塞
private final AtomicBoolean wakenUp = new AtomicBoolean(); 
private final SelectStrategy selectStrategy; // 选择策略
private volatile int ioRatio = 50;  // IO任务占总任务(IO+普通任务)比例
private int cancelledKeys;  // 取消的键数目
private boolean needsToSelectAgain;
```



###  继承体系总结

- JDK的AbstractExecutorService类定义了任务的提交和执行，留下了newTaskFor()方法用于子类定义执行的任务；

- Netty的AbstractEventExecutor类覆盖了newTaskFor()方法，使用PromiseTask表示待执行的任务；

- AbstractScheduledEventExecutor类将待执行的调度任务封装为ScheduledFutureTask提交给调度任务队列；

- SingleThreadEventExecutor类实现了任务执行器即线程，其覆盖了execute()方法，当使用execute()执行一个任务时，实质是向任务队列提交一个任务；该类中还有一个重要的**模板方法run()**，在这个方法中执行任务队列中的任务（调度任务队列中的待执行任务移入普通任务队列），留给子类实现；

- SingleThreadEventLoop类实现对Channel对象的注册。

- 从NioEventLoop继承体系的分析可以看出，NioEventLoop要实现的最关键方法就是基类的模板方法run()。


[自顶向下深入分析Netty（四）--EventLoop-1](https://www.jianshu.com/p/da4398743b5a)



## 主要功能

- 处理网络I/O读写事件

- 执行系统任务和定时任务



  在主循环中我们可以看到netty对I/O任务和提交到事件循环中的系统任务的调度。

![](http://web.uxiaowo.com/netty/Future/EventLoopTask.png)

### I/O事件

由于NIO的I/O读写需要使用选择符，因此，netty在NioEventLoop初始化时，会使用SelectorProvider打开selector。

### 任务处理

runAllTasks执行提交到EventLoop的任务，首先从scheduledTaskQueue获取需要执行的任务，加入到taskQueue，然后依次执行taskQueue的任务。

参见下`runAllTasks`



## 代码流程

### NioEventLoopGroup创建

标准的netty程序会调用到`NioEventLoopGroup`的父类`MultithreadEventExecutorGroup`的如下代码.

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
EventExecutorChooserFactory chooserFactory, Object... args) {
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
}

//然后通过newChild的方式传递给NioEventLoop
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```

netty中的默认Nio线程都是由 `DefaultThreadFactory`的`newThread()`方法创建出来的

??代码位置

`new NioEventLoopGroup() `—>调用MultithreadEventExecutorGroup初始化

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
    if (executor == null) {
    	//创建线程创建器 （1）
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    //构造NioEventLoop （2）
    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            //保存ThreadPerTaskExecutor，创建Mpscqueue，selector
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {  // 如果不成功，所有已经实例化的线程优雅关闭
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) { // 确保已经实例化的线程终止
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

    //创建线程选择器 （3）
    chooser = chooserFactory.newChooser(children);

    //设置线程终止异步结果
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

#### 线程执行器

每次执行都会创建一个线程实体。netty的实现DefaultThreadFactory，newThread利用FastThreadLocalThread方法创建线程。

```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;
    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

#### 创建NioEventLoop

1.保存线程执行器。

2.创建MpscQueue

3.创建一个selector

```java
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
    ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);}
```

#### 创建线程选择器

```java
//创建线程选择器
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    //(val & -val) == val;
    if (isPowerOfTwo(executors.length)) {
        //核心方法executors[idx.getAndIncrement() & executors.length - 1]
    	return new PowerOfTowEventExecutorChooser(executors);
    } else {
    	return new GenericEventExecutorChooser(executors);
    }
}
```

### 启动

![](https://raw.githubusercontent.com/seaside2mm/github-photos/master/Screen%20Shot%202018-12-24%20at%204.40.40%20PM.png)



对应代码`ChannelFuture f = b.bind(8888).sync(); `

```java
private static void doBind0(
    final ChannelFuture regFuture, final Channel channel,
    final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } 
        }
    });
}
```

NioEventLoop 父类 SingleThreadEventExecutor 的execute方法

```java
@Override
public void execute(Runnable task) {
    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();  //启动线程
        addTask(task);
    }
}

private void startThread() {
    if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            doStartThread();
        }
    }
}
```

外部线程在往任务队列里面添加任务的时候执行 `startThread()` ，netty会判断reactor线程有没有被启动，如果没有被启动，那就启动线程再往任务队列里面添加任务

```java
private void doStartThread() {
    assert thread == null;
    //创建并启动线程
    executor.execute(new Runnable() {
        @Override
        public void run() {
            //保存当前线程
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;

            try {
                //启动NioEventLoop
                SingleThreadEventExecutor.this.run();
                success = true;
            } 
        }
    });}
```

SingleThreadEventExecutor 在执行`doStartThread`的时候，会调用内部执行器`executor`的execute方法，将调用NioEventLoop的run方法的过程封装成一个runnable塞到一个线程中去执行.

该线程就是`executor`创建，对应netty的reactor线程实体。`executor` 默认是`ThreadPerTaskExecutor`

默认情况下，`ThreadPerTaskExecutor` 在每次执行`execute` 方法的时候都会通过`DefaultThreadFactory`创建一个`FastThreadLocalThread`线程，而这个线程就是netty中的reactor线程实体

### 执行

**事件循环**:首先根据默认的选择策略DefaultSelectStrategy判断本次循环是否select，具体逻辑为：如果当前有任务则使用selectNow立刻查询是否有准备就绪的I/O；如果当前没有任务则返回SelectStrategy.SELECT，并将wakenUp设置为false，并调用select()进行查询。

#### run方法

NioEventLoop的run方法是reactor线程的主体，在第一次添加任务的时候被启动。

```java
@Override
protected void run() {
    for (;;) {
        try {
            //选择策略
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    
                    //1 轮询io事件，调研select()方法
                    select(wakenUp.getAndSet(false));
                    // 唤醒select()的线程
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                    // fallthrough
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            
            if (ioRatio == 100) {
                try {
                    //2 处理这些IO事件（process selected keys）
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } 
}
```

reactor线程大概做的事情分为对三个步骤不断循环

1.首先轮询注册到reactor线程对用的selector上的所有的channel的IO事件

2.处理产生网络IO事件的channel

3.处理任务队列

##### 检测io事件

`wakenUp` 表示是否应该唤醒正在阻塞的select操作，可以看到netty在进行一次新的loop之前，都会将`wakeUp` 被设置成false，标志新的一轮loop的开始，具体的select操作我们也拆分开来看

```java
//检测io事件
private void select(boolean oldWakenUp) 
```

```java
int selectCnt = 0;
long currentTimeNanos = System.nanoTime();
long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

for (;;) {
    long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
    if (timeoutMillis <= 0) {
        if (selectCnt == 0) {
            selector.selectNow();
            selectCnt = 1;
        }
        break;
    }
    ....
}
```

我们可以看到，NioEventLoop中reactor线程的select操作也是一个for循环，在for循环第一步中，如果发现当前的定时任务队列中有任务的截止事件快到了(<=0.5ms)，就跳出循环。此外，跳出之前如果发现目前为止还没有进行过select操作（`if (selectCnt == 0)`），那么就调用一次`selectNow()`，该方法会立即返回，不会阻塞

这里说明一点，netty里面定时任务队列是按照延迟时间从小到大进行排序， `delayNanos(currentTimeNanos)`方法即取出第一个定时任务的延迟时间

```java
protected long delayNanos(long currentTimeNanos) {
    ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
    if (scheduledTask == null) {
        return SCHEDULE_PURGE_INTERVAL;
    }
    return scheduledTask.delayNanos(currentTimeNanos);
 }
```

关于netty的任务队列(包括普通任务，定时任务，tail task)相关的细节不表

> 2.轮询过程中发现有任务加入，中断本次轮询

```java
for (;;) {
    // 1.定时任务截至事时间快到了，中断本次轮询
    ...
    // 2.轮询过程中发现有任务加入，中断本次轮询
    if (hasTasks() && wakenUp.compareAndSet(false, true)) {
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    ....
}
```

netty为了保证任务队列能够及时执行，在进行阻塞select操作的时候会判断任务队列是否为空，如果不为空，就执行一次非阻塞select操作，跳出循环

> 3.阻塞式select操作

```java
for (;;) {
    // 1.定时任务截至事时间快到了，中断本次轮询
    ...
    // 2.轮询过程中发现有任务加入，中断本次轮询
    ...
    // 3.阻塞式select操作
    int selectedKeys = selector.select(timeoutMillis);
    selectCnt ++;
    if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
        break;
    }
    ....
}
```

执行到这一步，说明netty任务队列里面队列为空，并且所有定时任务延迟时间还未到(大于0.5ms)，于是，在这里进行一次阻塞select操作，截止到第一个定时任务的截止时间

这里，我们可以问自己一个问题，如果第一个定时任务的延迟非常长，比如一个小时，那么有没有可能线程一直阻塞在select操作，当然有可能！But，只要在这段时间内，有新任务加入，该阻塞就会被释放

> 外部线程调用execute方法添加任务

```java
@Override
public void execute(Runnable task) { 
    ...
    wakeup(inEventLoop); // inEventLoop为false
    ...
}
```

> 调用wakeup方法唤醒selector阻塞

```java
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
        selector.wakeup();
    }
}
```

可以看到，在外部线程添加任务的时候，会调用wakeup方法来唤醒 `selector.select(timeoutMillis)`

阻塞select操作结束之后，netty又做了一系列的状态判断来决定是否中断本次轮询，中断本次轮询的条件有

- 轮询到IO事件 （`selectedKeys != 0`）
- oldWakenUp 参数为true
- 任务队列里面有任务（`hasTasks`）
- 第一个定时任务即将要被执行 （`hasScheduledTasks（）`）
- 用户主动唤醒（`wakenUp.get()`）

> 4.解决jdk的nio bug

关于该bug的描述见 [http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6595055)](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6595055)

该bug会导致Selector一直空轮询，最终导致cpu 100%，nio server不可用，严格意义上来说，netty没有解决jdk的bug，而是通过一种方式来巧妙地避开了这个bug，具体做法如下

```java
long currentTimeNanos = System.nanoTime();
for (;;) {
    // 1.定时任务截止事时间快到了，中断本次轮询
    ...
    // 2.轮询过程中发现有任务加入，中断本次轮询
    ...
    // 3.阻塞式select操作
    selector.select(timeoutMillis);
    // 4.解决jdk的nio bug
    long time = System.nanoTime();
    if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
        selectCnt = 1;
    } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {

        rebuildSelector();
        selector = this.selector;
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    currentTimeNanos = time; 
    ...
 }
```

netty 会在每次进行 `selector.select(timeoutMillis)` 之前记录一下开始时间`currentTimeNanos`，在select之后记录一下结束时间，判断select操作是否至少持续了`timeoutMillis`秒（这里将`time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos`改成`time - currentTimeNanos >= TimeUnit.MILLISECONDS.toNanos(timeoutMillis)`或许更好理解一些）,
 如果持续的时间大于等于timeoutMillis，说明就是一次有效的轮询，重置`selectCnt`标志，否则，表明该阻塞方法并没有阻塞这么长时间，可能触发了jdk的空轮询bug，当空轮询的次数超过一个阀值的时候，默认是512，就开始重建selector

空轮询阀值相关的设置代码如下

```java
int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
    selectorAutoRebuildThreshold = 0;
}
SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;
```

下面我们简单描述一下netty 通过`rebuildSelector`来fix空轮询bug的过程，`rebuildSelector`的操作其实很简单：new一个新的selector，将之前注册到老的selector上的的channel重新转移到新的selector上。我们抽取完主要代码之后的骨架如下

```java
public void rebuildSelector() {
    final Selector oldSelector = selector;
    final Selector newSelector;
    newSelector = openSelector();

    int nChannels = 0;
     try {
        for (;;) {
                for (SelectionKey key: oldSelector.keys()) {
                    Object a = key.attachment();
                     if (!key.isValid() || key.channel().keyFor(newSelector) != null) {
                         continue;
                     }
                     int interestOps = key.interestOps();
                     key.cancel();
                     SelectionKey newKey = key.channel().register(newSelector, interestOps, a);
                     if (a instanceof AbstractNioChannel) {
                         ((AbstractNioChannel) a).selectionKey = newKey;
                      }
                     nChannels ++;
                }
                break;
        }
    } catch (ConcurrentModificationException e) {
        // Probably due to concurrent modification of the key set.
        continue;
    }
    selector = newSelector;
    oldSelector.close();
}
```

首先，通过`openSelector()`方法创建一个新的selector，然后执行一个死循环，只要执行过程中出现过一次并发修改selectionKeys异常，就重新开始转移

具体的转移步骤为

1. 拿到有效的key
2. 取消该key在旧的selector上的事件注册
3. 将该key对应的channel注册到新的selector上
4. 重新绑定channel和新的key的关系

转移完成之后，就可以将原有的selector废弃，后面所有的轮询都是在新的selector进行

最后，我们总结reactor线程select步骤做的事情：不断地轮询是否有IO事件发生，并且在轮询的过程中不断检查是否有定时任务和普通任务，保证了netty的任务队列中的任务得到有效执行，轮询过程顺带用一个计数器避开了了jdk空轮询的bug，过程清晰明了

##### 处理轮询到的事件

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip());
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

一种是处理优化过的selectedKeys，一种是正常的处理

首先，selectedKeys是一个 `SelectedSelectionKeySet` 类对象，在`NioEventLoop` 的 `openSelector` 方法中创建，之后就通过反射将selectedKeys与 `sun.nio.ch.SelectorImpl` 中的两个field绑定

```java
private Selector openSelector() {
        final Selector selector;
        try {
            selector = provider.openSelector();
        } catch (IOException e) {
            throw new ChannelException("failed to open a new selector", e);
        }

        if (DISABLE_KEYSET_OPTIMIZATION) {
            return selector;
        }

        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    return Class.forName(
                            "sun.nio.ch.SelectorImpl",
                            false,
                            PlatformDependent.getSystemClassLoader());
                } catch (ClassNotFoundException e) {
                    return e;
                } catch (SecurityException e) {
                    return e;
                }
            }
        });

        if (!(maybeSelectorImplClass instanceof Class) ||
                // ensure the current selector implementation is what we can instrument.
                !((Class<?>) maybeSelectorImplClass).isAssignableFrom(selector.getClass())) {
            if (maybeSelectorImplClass instanceof Exception) {
                Exception e = (Exception) maybeSelectorImplClass;
                logger.trace("failed to instrument a special java.util.Set into: {}", selector, e);
            }
            return selector;
        }

        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    selectedKeysField.setAccessible(true);
                    publicSelectedKeysField.setAccessible(true);

                    selectedKeysField.set(selector, selectedKeySet);
                    publicSelectedKeysField.set(selector, selectedKeySet);
                    return null;
                } 
            }
        });

        return selector;
    }
```

netty的reactor线程第二步做的事情为处理IO事件，netty使用数组替换掉jdk原生的HashSet来保证IO事件的高效处理，每个SelectionKey上绑定了netty类`AbstractChannel`对象作为attachment，在处理每个SelectionKey的时候，就可以找到`AbstractChannel`，然后通过pipeline的方式将处理串行到ChannelHandler，回调到用户方法

[netty源码分析之揭开reactor线程的面纱（二）](https://www.jianshu.com/p/467a9b41833e)



##### 执行任务队列中的任务

runAllTasks执行逻辑

```java
//Poll all tasks from the task queue and run them via {@link Runnable#run()} method.
protected boolean runAllTasks() {
    assert inEventLoop();
    boolean fetchedAll;
    boolean ranAtLeastOne = false;
    do {
        //定时任务排队
        fetchedAll = fetchFromScheduledTaskQueue();
        if (runAllTasksFrom(taskQueue)) {
            ranAtLeastOne = true;
        }
    } while (!fetchedAll); // keep on processing until we fetched all scheduled tasks.

    if (ranAtLeastOne) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
    }
    afterRunningAllTasks();
    return ranAtLeastOne;
}
//ioRatio不为100时，会调用runAllTasks(ioTime * (100 - ioRatio) / ioRatio)，首先计算出I/O处理的事件，然后按照比例为执行task分配事件，内部主要逻辑与runAllTasks()主要逻辑相同。

//定时任务
protected boolean runAllTasks(long timeoutNanos) {
    //从scheduledTaskQueue转移定时任务到taskQueue(mpsc queue)
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);

        runTasks ++;
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

表示了尽量在一定的时间内，将所有的任务都取出来run一遍。`timeoutNanos` 表示该方法最多执行这么长时间，netty为什么要这么做？我们可以想一想，reactor线程如果在此停留的时间过长，那么将积攒许多的IO事件无法处理(见reactor线程的前面两个步骤)，最终导致大量客户端请求阻塞，因此，默认情况下，netty将控制内部队列的执行时间.

reactor执行task的所有逻辑，可以拆解成下面几个步骤

1. 从scheduledTaskQueue转移定时任务到taskQueue(mpsc queue)
2. 计算本次任务循环的截止时间
3. 执行任务
4. 收尾

```java
//任务的聚合,调用  fetchFromScheduledTaskQueue()方法，将到期的定时任务转移到mpsc queue里面
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    while (scheduledTask != null) {
        if (!taskQueue.offer(scheduledTask)) {
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
        scheduledTask  = pollScheduledTask(nanoTime);
    }
    return true;
}
```

可以看到，netty在把任务从scheduledTaskQueue转移到taskQueue的时候还是非常小心的，当taskQueue无法offer的时候，需要把从scheduledTaskQueue里面取出来的任务重新添加回去

从scheduledTaskQueue从拉取一个定时任务的逻辑如下，传入的参数`nanoTime`为当前时间(其实是当前纳秒减去`ScheduledFutureTask`类被加载的纳秒个数)

```java
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    if (scheduledTask == null) {
        return null;
    }

    if (scheduledTask.deadlineNanos() <= nanoTime) {
        scheduledTaskQueue.remove();
        return scheduledTask;
    }
    return null;
}
```

可以看到，每次 `pollScheduledTask` 的时候，只有在当前任务的截止时间已经到了，才会取出来

- 计算本次任务循环的截止时间

```java
Runnable task = pollTask();
//...
final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
long runTasks = 0;
long lastExecutionTime;
```

这一步将取出第一个任务，用reactor线程传入的超时时间 `timeoutNanos` 来计算出当前任务循环的deadline，并且使用了`runTasks`，`lastExecutionTime`来时刻记录任务的状态

- 循环执行任务

```java
for (;;) {
    safeExecute(task);
    runTasks ++;
    if ((runTasks & 0x3F) == 0) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
        if (lastExecutionTime >= deadline) {
            break;
        }
    }

    task = pollTask();
    if (task == null) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
        break;
    }
}
```

这一步便是netty里面执行所有任务的核心代码了。
 首先调用`safeExecute`来确保任务安全执行，忽略任何异常

```java
protected static void safeExecute(Runnable task) {
    try {
        task.run();
    } catch (Throwable t) {
        logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```

然后将已运行任务 `runTasks` 加一，每隔`0x3F`任务，即每执行完64个任务之后，判断当前时间是否超过本次reactor任务循环的截止时间了，如果超过，那就break掉，如果没有超过，那就继续执行。可以看到，netty对性能的优化考虑地相当的周到，假设netty任务队列里面如果有海量小任务，如果每次都要执行完任务都要判断一下是否到截止时间，那么效率是比较低下的

- 收尾

```java
afterRunningAllTasks();
this.lastExecutionTime = lastExecutionTime;
```

收尾工作很简单，调用一下 `afterRunningAllTasks` 方法

```java
@Override
protected void afterRunningAllTasks() {
        runAllTasksFrom(tailTasks);
}
```

`NioEventLoop`可以通过父类`SingleTheadEventLoop`的`executeAfterEventLoopIteration`方法向`tailTasks`中添加收尾任务，比如，你想统计一下一次执行一次任务循环花了多长时间就可以调用此方法

```java
public final void executeAfterEventLoopIteration(Runnable task) {
        // ...
        if (!tailTasks.offer(task)) {
            reject(task);
        }
        //...
}
```

`this.lastExecutionTime = lastExecutionTime;`简单记录一下任务执行的时间，搜了一下该field的引用，发现这个field并没有使用过，只是每次不停地赋值，赋值，赋值...，改天再去向netty官方提个issue...

reactor线程第三曲到了这里基本上就给你讲完了，如果你读到这觉得很轻松，那么恭喜你，你对netty的task机制已经非常比较熟悉了，也恭喜一下我，把这些机制给你将清楚了。我们最后再来一次总结，以tips的方式

- 当前reactor线程调用当前eventLoop执行任务，直接执行，否则，添加到任务队列稍后执行
- netty内部的任务分为普通任务和定时任务，分别落地到MpscQueue和PriorityQueue
- netty每次执行任务循环之前，会将已经到期的定时任务从PriorityQueue转移到MpscQueue
- netty每隔64个任务检查一下是否该退出任务循环

[netty源码分析之揭开reactor线程的面纱（三）](https://www.jianshu.com/p/58fad8e42379)

[Nio事件循环](https://www.kancloud.cn/ssj234/netty-source/434741)

