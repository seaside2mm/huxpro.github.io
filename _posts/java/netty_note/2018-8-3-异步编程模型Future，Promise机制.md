## 异步编程模型:Future，Promise机制

Netty的事件处理采用异步的方式，异步处理则需要表示异步操作的结果。Future正是用来表示异步操作结果的对象



## JDK的Future对象

```java
// 取消异步操作
boolean cancel(boolean mayInterruptIfRunning);
// 异步操作是否取消
boolean isCancelled();
// 异步操作是否完成，正常终止、异常、取消都是完成
boolean isDone();
// 阻塞直到取得异步操作结果
V get() throws InterruptedException, ExecutionException;
// 同上，但最长阻塞时间为timeout
V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```



### 问题点

- 接口中只有`isDone()`方法判断一个异步操作是否完成，但是对于完成的定义过于模糊，JDK文档指出正常终止、抛出异常、用户取消都会使isDone()方法返回真。在我们的使用中，我们极有可能是对这三种情况分别处理，而JDK这样的设计不能满足我们的需求。
- 对于一个异步操作，我们更关心的是这个异步操作触发或者结束后能否再执行一系列动作。



### FutureTask

java的FutureTask通过state记录业务逻辑的执行状态；多线程时使用CAS防止重复进入；业务逻辑未执行完成时，会将线程加入到waiter链表，使用LockSupport.park()阻塞业务线程；业务逻辑执行完毕或发生异常或被取消时，唤醒等待列表的线程。

[Java的Future机制](https://www.kancloud.cn/ssj234/netty-source/433216)



## Netty扩展的Future接口

Netty的Future在concurrent包的Future基础上，增加了更多的功能。在Java的Future中，主要是任务的运行/取消，而Netty的Future增加了更多的功能。

```java
public interface Future<V> extends java.util.concurrent.Future<V> 

boolean isSuccess(); //只有IO操作完成时才返回true
boolean isCancellable(); //只有当cancel(boolean)成功取消时才返回true
Throwable cause();  //IO操作发生异常时，返回导致IO操作以此的原因，如果没有异常，返回null
// 向Future添加事件，future完成时，会执行这些事件，如果add时future已经完成，会立即执行监听事件
Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
// 移除监听事件，future完成时，不会触发
Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
// 异步操作失败的原因
Throwable cause();
// 阻塞直到异步操作完成
Future<V> await() throws InterruptedException;
// 同上，但异步操作失败时抛出异常
Future<V> sync() throws InterruptedException;
// 非阻塞地返回异步结果，如果尚未完成返回null
V getNow();
Future<V> syncUninterruptibly();   // 等待future done，不可打断
Future<V> awaitUninterruptibly();  // 等待future 完成，不可打断
V getNow(); // 立刻获得结果，如果没有完成，返回null
boolean cancel(boolean mayInterruptIfRunning); // 如果成功取消，future会失败，导致CancellationException
```

Netty为Future加入的功能主要是添加/删除监听事件，在Promise中会有实例演示。其他的方法是为get()方法服务的，get()方法可以通过调用await/getNow等方法实现。

```java
/*                                      +---------------------------+
 *                                      | Completed successfully    |
 *                                      +---------------------------+
 *                                 +---->      isDone() = true      |
 * +--------------------------+    |    |   isSuccess() = true      |
 * |        Uncompleted       |    |    +===========================+
 * +--------------------------+    |    | Completed with failure    |
 * |      isDone() = false    |    |    +---------------------------+
 * |   isSuccess() = false    |----+---->      isDone() = true      |
 * | isCancelled() = false    |    |    |       cause() = non-null  |
 * |       cause() = null     |    |    +===========================+
 * +--------------------------+    |    | Completed by cancellation |
 *                                 |    +---------------------------+
 *                                 +---->      isDone() = true      |
 *                                      | isCancelled() = true      |
 * /                                     +---------------------------
```



## Promise接口

在Java的Future中，业务逻辑为一个Callable或Runnable实现类，该类的call()或run()执行完毕意味着业务逻辑的完结；而在Promise机制中，可以在业务逻辑中人工设置业务逻辑的成功与失败。

```java
public interface Promise<V> extends Future<V> {
    // 标记异步操作结果为成功，如果已被设置（不管成功还是失败）则抛出异常IllegalStateException
    Promise<V> setSuccess(V result);
    // 同上，只是结果已被设置时返回False
    boolean trySuccess(V result);

    Promise<V> setFailure(Throwable cause);
    boolean tryFailure(Throwable cause);

    // 设置结果为不可取消，结果已被取消返回False
    boolean setUncancellable();
}
```



## 实现类的关系链

1. AbstractFuture<--CompleteFuture<--CompleteChannelFuture<-Succeeded/FailedChannelFuture

2. DefaultPromise<--DefaultChannelPromise

### 一些Future

- AbstractFuture主要实现Future的get()方法，取得Future关联的异步操作结果

- CompleteFuture表示一个异步操作已完成的结果，由此可推知：该类的实例在异步操作完成时创建，返回给用户，用户则使用addListener()方法定义一个异步操作。
  + GenericFutureListener接口operationComplete方法：异步操作完成后调用
  + ChannelFutureListener接口
  + 由于CompleteFuture表示一个已完成的异步操作，所以可推知sync()和await()方法都将立即返回。
- CompleteChannelFuture抽象类extends CompleteFuture<Void> implements ChannelFuture
  + 将CompleteChannelFuture纯粹的视为一种回调函数机制。
  + 大部分方法实现中，只是将方法返回的Future覆盖为ChannelFuture对象
- Succeeded/FailedChannelFuture为特定的两个异步操作结果

### DefaultPromise 实现类

```java
// 异步操作结果
private volatile Object result;
// 执行listener操作的执行器
private final EventExecutor executor;
// 监听者
private Object listeners;
// 阻塞等待该结果的线程数
private short waiters;
// 通知正在进行标识
private boolean notifyingListeners;
```

也许你已经注意到，listeners是一个Object类型。这似乎不合常理，一般情况下我们会使用一个集合或者一个数组。Netty之所以这样设计，是因为大多数情况下listener只有一个，用集合和数组都会造成浪费。当只有一个listener时，该字段为一个GenericFutureListener对象；当多余一个listener时，该字段为DefaultFutureListeners，可以储存多个listener。明白了这些，我们分析关键方法addListener()：

```java
@Override
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {
    synchronized (this) {
        addListener0(listener); // 保证多线程情况下只有一个线程执行添加操作
    }
    if (isDone()) {
        notifyListeners();  // 异步操作已经完成通知监听者
    }
    return this;
}

private void addListener0(GenericFutureListener<? extends Future<? super V>> listener) {
    if (listeners == null) {
        listeners = listener;   // 只有一个
    } else if (listeners instanceof DefaultFutureListeners) {
        ((DefaultFutureListeners) listeners).add(listener); // 大于两个
    } else {
        // 从一个扩展为两个
        listeners = new DefaultFutureListeners((GenericFutureListener<? extends Future<V>>) listeners, listener);   
    }
}
```

从代码中可以看出，在添加Listener时，如果异步操作已经完成，则会notifyListeners()：

```java
private void notifyListeners() {
    EventExecutor executor = executor();
    if (executor.inEventLoop()) {   //执行线程为指定线程
        final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
        final int stackDepth = threadLocals.futureListenerStackDepth(); // 嵌套层数
        if (stackDepth < MAX_LISTENER_STACK_DEPTH) {
            // 执行前增加嵌套层数
            threadLocals.setFutureListenerStackDepth(stackDepth + 1);   
            try {
                notifyListenersNow();
            } finally {
                // 执行完毕，无论如何都要回滚嵌套层数
                threadLocals.setFutureListenerStackDepth(stackDepth);
            }
            return;
        }
    }
    // 外部线程则提交任务给执行线程
    safeExecute(executor, () -> { notifyListenersNow(); });
}

private static void safeExecute(EventExecutor executor, Runnable task) {
    try {
        executor.execute(task);
    } catch (Throwable t) {
        rejectedExecutionLogger.error("Failed to submit a listener notification task. Event loop shut down?", t);
    }
}
```

所以，外部线程不能执行监听者Listener中定义的操作，只能提交任务到指定Executor，其中的操作最终由指定Executor执行。我们再看notifyListenersNow()方法：

```java
private void notifyListenersNow() {
    Object listeners;
    // 此时外部线程可能会执行添加Listener操作，所以需要同步
    synchronized (this) { 
        if (notifyingListeners || this.listeners == null) {
            // 正在通知或已没有监听者（外部线程删除）直接返回
            return; 
        }
        notifyingListeners = true;  
        listeners = this.listeners;
        this.listeners = null;
    }
    for (;;) {
        if (listeners instanceof DefaultFutureListeners) { // 通知单个
            notifyListeners0((DefaultFutureListeners) listeners);
        } else { // 通知多个（遍历集合调用单个）
            notifyListener0(this, (GenericFutureListener<? extends Future<V>>) listeners);
        }
        synchronized (this) {
            // 执行完毕且外部线程没有再添加监听者
            if (this.listeners == null) {
                notifyingListeners = false; 
                return; 
            }
            // 外部线程添加了监听者继续执行
            listeners = this.listeners; 
            this.listeners = null;
        }
    }
}

private static void notifyListener0(Future future, GenericFutureListener l) {
    try {
        l.operationComplete(future);
    } catch (Throwable t) {
        logger.warn("An exception was thrown by " + l.getClass().getName() + ".operationComplete()", t);
    }
}
```

到此为止，我们分析完了Promise最重要的addListener()和notifyListener()方法。在源码中还有static的notifyListener()方法，这些方法是CompleteFuture使用的，对于CompleteFuture，添加监听者的操作不需要缓存，直接执行Listener中的方法即可，执行线程为调用线程，相关代码可回顾CompleteFuture。addListener()相对的removeListener()方法实现简单，我们不再分析。
 回忆result字段，修饰符有volatile，所以使用RESULT_UPDATER更新，保证更新操作为原子操作。Promise不携带特定的结果（即携带Void）时，成功时设置为静态字段的Signal对象SUCCESS；如果携带泛型参数结果，则设置为泛型一致的结果。对于Promise，设置成功、设置失败、取消操作，**三个操作至多只能调用一个且同一个方法至多生效一次**，再次调用会抛出异常（set）或返回失败（try）。这些设置方法原理相同，我们以setSuccess()为例分析:



[自顶向下深入分析Netty（五）--Future](https://www.jianshu.com/p/a06da3256f0c)

[Netty的Future](https://www.kancloud.cn/ssj234/netty-source/433217)

## 示例

### Future

使用Future机制时，我们调用耗时任务会立刻返回一个Future实例，使用该实例能够以阻塞的方式或者在未来某刻获得耗时任务的执行结果，还可以添加监听事件设置后续程序。

```java
function Future asynchronousFunction(String arg){
  Future future = new Future(new Callable(){
      public Object call(){
        return null;
      }
  });
  return future;
}
 ReturnHandler handler = asynchronousFunction(); //  耗时函数，但会立即返回一个句柄
 handler.getResult(); // 通过句柄可以等待结果
 handler.addListener(); //通过句柄可以添加完成后执行的事件
 handler.cancel(); // 通过句柄取消耗时任务
```

### Promise

在Future机制中，业务逻辑所在任务执行的状态（成功或失败）是在Future中实现的，而在Promise中，可以在业务逻辑控制任务的执行结果，相比Future，更加灵活。

```java
// 异步的耗时任务接收一个promise
function Promise asynchronousFunction(String arg){
	Promise  promise = new PromiseImpl();
	Object result = null;
    result = search()  //业务逻辑,
    if(success){
         promise.setSuccess(result); // 通知promise当前异步任务成功了，并传入结果
    }else if(failed){
        promise.setFailure(reason); //// 通知promise当前异步任务失败了
     }else if(error){
     	promise.setFailure(error); //// 通知promise当前异步任务发生了异常
      }
}

// 调用异步的耗时任务
Promise promise = asynchronousFunction(promise) ；//会立即返回promise
//添加成功处理/失败处理/异常处理等事件
promise.addListener();// 例如，可以添加成功后执行的事件
doOtherThings() ; //　继续做其他事件，不需要理会asynchronousFunction何时结束
```

在Netty中，Promise继承了Future，包含了这两者的功能。