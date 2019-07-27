# 第六章 Pipeline



Pipleline中可以讲ChannelHandler维护成一个双向链表，实现上通过将ChannelHandler包装为AbstractChannelHandlerContext，然后将各个Context连接起来。



## 初始化



在Netty启动时，创建Channel的构造方法中会初始化一个默认的DefaultChannelPipeline

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    //一个处理输入的处理器
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```



在创建channel时被创建

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```





### Pipeline数据结构ChannelHandlerContext

```java
interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker
```



### Head Tail哨兵



```java
//一个处理输入的处理器
class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler 
//调用Channel的unsafe处理所有的I/O操作。
class HeadContext extends AbstractChannelHandlerContext implements ChannelOutboundHandler, ChannelInboundHandler
```



## 添加channelHandler

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```



```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        ctx.handler().handlerAdded(ctx);
        ctx.setAddComplete();
    } catch (Throwable t) {
        boolean removed = false;
        try {
            remove0(ctx);
            try {
                ctx.handler().handlerRemoved(ctx);
            } finally {
                ctx.setRemoved();
            }
            removed = true;
        } catch (Throwable t2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to remove a handler: " + ctx.name(), t2);
            }
        }

        if (removed) {
            fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() +
                ".handlerAdded() has thrown an exception; removed.", t));
        } else {
            fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() +
                ".handlerAdded() has thrown an exception; also failed to remove.", t));
        }
    }
}
```



## 删除channelHandler



```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail;

    synchronized (this) {
        remove0(ctx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we remove the context from the pipeline and add a task that will call
        // ChannelHandler.handlerRemoved(...) once the channel is registered.
        if (!registered) {
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }

        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx);
    return ctx;
}
```





**通知机制**

在Netty的启动过程以及后续的I/O操作中，很多阶段都会通知Pipeline上面的ChannelHandler。
例如，在启动过程在，注册完成后，调用pipeline.fireChannelRegistered();绑定完成后调用pipeline.fireChannelActive();
我们以fireChannelRead为例，看看如何实现的按照链表通知。

```java
//DefaultChannelPipeline.java
 public final ChannelPipeline fireChannelRead(Object msg) {
     AbstractChannelHandlerContext.invokeChannelRead(head, msg);
     return this;
 }

// AbstractChannelHandlerContext.java
void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    // ReferenceCountUtil 引用计数
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    // 调用invokeChannelRead
    next.invokeChannelRead(m); 
}
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) { // 判断是否已经添加完成
        try {
            // 调用ChannelHandler的channelRead
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg); // 未完成继续下一个
    }
}
```

上面的程序中，从head出发查找ChannelInboundHandler，调用其channelRead，如果需要继续调用链表后面的channelRead，需要调用ctx.fireChannelRead(msg);继续通知，在今后自己定义channelRead时需要注意，需要手动继续传递消息。





## inBound事件传播



## outBound事件传播



## 异常的传播

