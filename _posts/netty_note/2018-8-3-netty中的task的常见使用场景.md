## netty中的task的常见使用场景

我们取三种典型的task使用场景来分析

### 一. 用户自定义普通任务

```java
ctx.channel().eventLoop().execute(new Runnable() {
    @Override
    public void run() {
        //...
    }
});
```

我们跟进`execute`方法，看重点

```java
@Override
public void execute(Runnable task) {
    //...
    addTask(task);
    //...
}
```

`execute`方法调用 `addTask`方法

```java
protected void addTask(Runnable task) {
    // ...
    if (!offerTask(task)) {
        reject(task);
    }
}
```

然后调用`offerTask`方法，如果offer失败，那就调用`reject`方法，通过默认的 `RejectedExecutionHandler` 直接抛出异常

```java
final boolean offerTask(Runnable task) {
    // ...
    return taskQueue.offer(task);
}
```

跟到`offerTask`方法，基本上task就落地了，netty内部使用一个`taskQueue`将task保存起来，那么这个`taskQueue`又是何方神圣？

我们查看 `taskQueue` 定义的地方和被初始化的地方

```java
private final Queue<Runnable> taskQueue;
taskQueue = newTaskQueue(this.maxPendingTasks);

@Override
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    // This event loop never calls takeTask()
    return PlatformDependent.newMpscQueue(maxPendingTasks);
}
```

我们发现 `taskQueue`在NioEventLoop中默认是mpsc队列，mpsc队列，即多生产者单消费者队列，netty使用mpsc，方便的将外部线程的task聚集，在reactor线程内部用单线程来串行执行，我们可以借鉴netty的任务执行模式来处理类似多线程数据上报，定时聚合的应用

在本节讨论的任务场景中，所有代码的执行都是在reactor线程中的，所以，所有调用 `inEventLoop()` 的地方都返回true，既然都是在reactor线程中执行，那么其实这里的mpsc队列其实没有发挥真正的作用，mpsc大显身手的地方其实在第二种场景

### 二. 非当前reactor线程调用channel的各种方法

```
// non reactor thread
channel.write(...)
```

上面一种情况在push系统中比较常见，一般在业务线程里面，根据用户的标识，找到对应的channel引用，然后调用write类方法向该用户推送消息，就会进入到这种场景

关于channel.write()类方法的调用链，后面会单独拉出一篇文章来深入剖析，这里，我们只需要知道，最终write方法串至以下方法`AbstractChannelHandlerContext.java`

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // ...
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```

外部线程在调用`write`的时候，`executor.inEventLoop()`会返回false，直接进入到else分支，将write封装成一个`WriteTask`（这里仅仅是write而没有flush，因此`flush`参数为false）, 然后调用 `safeExecute`方法

```java
private static void safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg) {
    // ...
    executor.execute(runnable);
    // ...
}
```

接下来的调用链就进入到第一种场景了，但是和第一种场景有个明显的区别就是，第一种场景的调用链的发起线程是reactor线程，第二种场景的调用链的发起线程是用户线程，用户线程可能会有很多个，显然多个线程并发写`taskQueue`可能出现线程同步问题，于是，这种场景下，netty的mpsc queue就有了用武之地

### 三. 用户自定义定时任务

```java
ctx.channel().eventLoop().schedule(new Runnable() {
    @Override
    public void run() {
    }
}, 60, TimeUnit.SECONDS);
```

第三种场景就是定时任务逻辑了，用的最多的便是如上方法：在一定时间之后执行任务

我们跟进`schedule`方法

```java
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
//...
    return schedule(new ScheduledFutureTask<Void>(
            this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
} 
```

通过 `ScheduledFutureTask`, 将用户自定义任务再次包装成一个netty内部的任务

```java
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    // ...
    scheduledTaskQueue().add(task);
    // ...
    return task;
}
```

到了这里，我们有点似曾相识，在非定时任务的处理中，netty通过一个mpsc队列将任务落地，这里，是否也有一个类似的队列来承载这类定时任务呢？带着这个疑问，我们继续向前

```java
Queue<ScheduledFutureTask<?>> scheduledTaskQueue() {
    if (scheduledTaskQueue == null) {
        scheduledTaskQueue = new PriorityQueue<ScheduledFutureTask<?>>();
    }
    return scheduledTaskQueue;
}
```

果不其然，`scheduledTaskQueue()` 方法，会返回一个优先级队列，然后调用 `add` 方法将定时任务加入到队列中去，但是，这里为什么要使用优先级队列，而不需要考虑多线程的并发？

因为我们现在讨论的场景，调用链的发起方是reactor线程，不会存在多线程并发这些问题

但是，万一有的用户在reactor之外执行定时任务呢？虽然这类场景很少见，但是netty作为一个无比健壮的高性能io框架，必须要考虑到这种情况。

对此，netty的处理是，如果是在外部线程调用schedule，netty将添加定时任务的逻辑封装成一个普通的task，这个task的任务是添加[添加定时任务]的任务，而不是添加定时任务，其实也就是第二种场景，这样，对 `PriorityQueue`的访问就变成单线程，即只有reactor线程

> 完整的schedule方法

```java
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        scheduledTaskQueue().add(task);
    } else {
        // 进入到场景二，进一步封装任务
        execute(new Runnable() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }
    return task;
}
```

在阅读源码细节的过程中，我们应该多问几个为什么？这样会有利于看源码的时候不至于犯困！比如这里，为什么定时任务要保存在优先级队列中，我们可以先不看源码，来思考一下优先级对列的特性

优先级队列按一定的顺序来排列内部元素，内部元素必须是可以比较的，联系到这里每个元素都是定时任务，那就说明定时任务是可以比较的，那么到底有哪些地方可以比较？

每个任务都有一个下一次执行的截止时间，截止时间是可以比较的，截止时间相同的情况下，任务添加的顺序也是可以比较的，就像这样，阅读源码的过程中，一定要多和自己对话，多问几个为什么

带着猜想，我们研究与一下`ScheduledFutureTask`，抽取出关键部分

```java
final class ScheduledFutureTask<V> extends PromiseTask<V> implements ScheduledFuture<V> {
    private static final AtomicLong nextTaskId = new AtomicLong();
    private static final long START_TIME = System.nanoTime();

    static long nanoTime() {
        return System.nanoTime() - START_TIME;
    }
    private final long id = nextTaskId.getAndIncrement();
    /* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
    private final long periodNanos;

    @Override
    public int compareTo(Delayed o) {
        //...
    }
    // 精简过的代码
    @Override
    public void run() {
    }
```

这里，我们一眼就找到了`compareTo` 方法，`cmd+u`跳转到实现的接口，发现就是`Comparable`接口

```java
public int compareTo(Delayed o) {
    if (this == o) {
        return 0;
    }
    ScheduledFutureTask<?> that = (ScheduledFutureTask<?>) o;
    long d = deadlineNanos() - that.deadlineNanos();
    if (d < 0) {
        return -1;
    } else if (d > 0) {
        return 1;
    } else if (id < that.id) {
        return -1;
    } else if (id == that.id) {
        throw new Error();
    } else {
        return 1;
    }
}
```

进入到方法体内部，我们发现，两个定时任务的比较，确实是先比较任务的截止时间，截止时间相同的情况下，再比较id，即任务添加的顺序，如果id再相同的话，就抛Error

这样，在执行定时任务的时候，就能保证最近截止时间的任务先执行

下面，我们再来看下netty是如何来保证各种定时任务的执行的，netty里面的定时任务分以下三种

1.若干时间后执行一次
 2.每隔一段时间执行一次
 3.每次执行结束，隔一定时间再执行一次

netty使用一个 `periodNanos` 来区分这三种情况，正如netty的注释那样

```java
/* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
private final long periodNanos;
```

了解这些背景之后，我们来看下netty是如何来处理这三种不同类型的定时任务的

```java
public void run() {
    if (periodNanos == 0) {
        V result = task.call();
        setSuccessInternal(result);
    } else { 
        task.call();
        long p = periodNanos;
        if (p > 0) {
            deadlineNanos += p;
        } else {
            deadlineNanos = nanoTime() - p;
        }
            scheduledTaskQueue.add(this);
        }
    }
}
```

`if (periodNanos == 0)` 对应 `若干时间后执行一次` 的定时任务类型，执行完了该任务就结束了。

否则，进入到else代码块，先执行任务，然后再区分是哪种类型的任务，`periodNanos`大于0，表示是以固定频率执行某个任务，和任务的持续时间无关，然后，设置该任务的下一次截止时间为本次的截止时间加上间隔时间`periodNanos`，否则，就是每次任务执行完毕之后，间隔多长时间之后再次执行，截止时间为当前时间加上间隔时间，`-p`就表示加上一个正的间隔时间，最后，将当前任务对象再次加入到队列，实现任务的定时执行.

