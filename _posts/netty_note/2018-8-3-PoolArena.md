# PoolArena



[上一节](https://www.jianshu.com/p/15304cd63175)讲述了jemalloc的思想，本节将分析Netty的实现细节。在Netty实现中，相关的类都加上了前缀`Pool`，比如`PoolArena`、`PoolChunk`等，本节分析`PoolArena`的源码实现细节。

首先看类签名：

```
    abstract class PoolArena<T> implements PoolArenaMetric

```

该类是一个抽象类，这是因为`ByteBuf`分为Heap和Direct，所以`PoolArena`同样分为两类：Heap和Direct。该类实现的接口`PoolArenaMetric`是一些信息的测度统计，忽略这些信息不再分析。
其中的关键成员变量如下：

```java
private final int maxOrder; // chunk相关满二叉树的高度
final int pageSize; // 单个page的大小
final int pageShifts; // 用于辅助计算
final int chunkSize; // chunk的大小
final int subpageOverflowMask; // 用于判断请求是否为Small/Tiny
final int numSmallSubpagePools; // small请求的双向链表头个数
final int directMemoryCacheAlignment; // 对齐基准
final int directMemoryCacheAlignmentMask; // 用于对齐内存
private final PoolSubpage<T>[] tinySubpagePools; // Subpage双向链表
private final PoolSubpage<T>[] smallSubpagePools; // Subpage双向链表

final PooledByteBufAllocator parent;

```

对于前述分析的如QINIT、Q0等chunk状态，Netty使用`PoolChunkList`作为容器存放相同状态的Chunk块，相关变量如下：

```java
//PoolChunkList(PoolChunkList<T> nextList, int minUsage, int maxUsage, int chunkSize)结构
private final PoolChunkList<T> q050;
private final PoolChunkList<T> q025;
private final PoolChunkList<T> q000;
private final PoolChunkList<T> qInit;
private final PoolChunkList<T> q075;
private final PoolChunkList<T> q100;
```

构造方法如下：

```java
protected PoolArena(PooledByteBufAllocator parent, int pageSize, int maxOrder, int pageShifts, int chunkSize, int cacheAlignment) {
    this.parent = parent;
    this.pageSize = pageSize;
    this.maxOrder = maxOrder;
    this.pageShifts = pageShifts;
    this.chunkSize = chunkSize;
    
    directMemoryCacheAlignment = cacheAlignment;
    directMemoryCacheAlignmentMask = cacheAlignment - 1;
    subpageOverflowMask = ~(pageSize - 1);
    tinySubpagePools = new PoolSubpage[numTinySubpagePools];
    
    for (int i = 0; i < tinySubpagePools.length; i ++) {
        tinySubpagePools[i] = newSubpagePoolHead(pageSize);
    }

    numSmallSubpagePools = pageShifts - 9;
    smallSubpagePools = new PoolSubpage[numSmallSubpagePools];
    for (int i = 0; i < smallSubpagePools.length; i ++) {
        smallSubpagePools[i] = newSubpagePoolHead(pageSize);
    }

    initPoolChunkList();
}

private PoolSubpage<T> newSubpagePoolHead(int pageSize) {
    PoolSubpage<T> head = new PoolSubpage<T>(pageSize);
    head.prev = head;
    head.next = head;
    return head;
}
```

其中`initPoolChunkList()`如下：

```java
q100 = new PoolChunkList<T>(this, null, 100, Integer.MAX_VALUE, chunkSize);
q075 = new PoolChunkList<T>(this, q100, 75, 100, chunkSize);
q050 = new PoolChunkList<T>(this, q075, 50, 100, chunkSize);
q025 = new PoolChunkList<T>(this, q050, 25, 75, chunkSize);
q000 = new PoolChunkList<T>(this, q025, 1, 50, chunkSize);
qInit = new PoolChunkList<T>(this, q000, Integer.MIN_VALUE, 25, chunkSize);

q100.prevList(q075);
q075.prevList(q050);
q050.prevList(q025);
q025.prevList(q000);
q000.prevList(null);
qInit.prevList(qInit);

```

这段代码实现如的双向链表

Netty使用一个枚举来表示每次请求大小的类别：

```java
enum SizeClass {
    Tiny,
    Small,
    Normal
    // 除此之外的请求为Huge
}
```

根据请求分配大小判断所属分类的代码如下，体会其中的位运算：

```java
// capacity < pageSize
boolean isTinyOrSmall(int normCapacity) {
    // subpageOverflowMask = ~(pageSize - 1)
    return (normCapacity & subpageOverflowMask) == 0;
}

// normCapacity < 512
static boolean isTiny(int normCapacity) {
    return (normCapacity & 0xFFFFFE00) == 0;
}

// capacity <= chunkSize
boolean isNormal(int normCapacity){
    return normCapacity <= chunkSize;
}
```

对容量进行规范化的代码如下：

```java
int normalizeCapacity(int reqCapacity) {
    // Huge 直接返回（直接内存需要对齐）
    if (reqCapacity >= chunkSize) {
        return directMemoryCacheAlignment == 0 ? reqCapacity : 
        alignCapacity(reqCapacity);
    }

    // Small和Normal 规范化到大于2的n次方的最小值
    if (!isTiny(reqCapacity)) { // >= 512
        int normalizedCapacity = reqCapacity;
        normalizedCapacity --;
        normalizedCapacity |= normalizedCapacity >>>  1;
        normalizedCapacity |= normalizedCapacity >>>  2;
        normalizedCapacity |= normalizedCapacity >>>  4;
        normalizedCapacity |= normalizedCapacity >>>  8;
        normalizedCapacity |= normalizedCapacity >>> 16;
        normalizedCapacity ++;

        if (normalizedCapacity < 0) {
            normalizedCapacity >>>= 1;
        }
        return normalizedCapacity;
    }

    // Tiny且直接内存需要对齐
    if (directMemoryCacheAlignment > 0) {
        return alignCapacity(reqCapacity);
    }

    // Tiny且已经是16B的倍数
    if ((reqCapacity & 15) == 0) {
        return reqCapacity;
    }

    // Tiny不是16B的倍数则规范化到16B的倍数
    return (reqCapacity & ~15) + 16;
}

```

规范化的结果可查看请求分类图，实现中使用了大量位运算，请仔细体会。另外，直接内存对齐后的请求容量为基准的倍数，比如基准为64B，则分配的内存都需要为64B的整数倍，也就是常说的按64字节对齐，实现代码如下（依然使用位运算）：

```
    int alignCapacity(int reqCapacity) {
        // directMemoryCacheAlignmentMask = cacheAlignment - 1;
        int delta = reqCapacity & directMemoryCacheAlignmentMask;
        return delta == 0 ? reqCapacity : reqCapacity + directMemoryCacheAlignment - delta;
    }

```

对于Small和Tiny的请求，随着请求的分配，`PoolArena`可能会形成如下的双向循环链表：

[图片上传失败...(image-4ee8ff-1545104511757)]

其中的每个节点都是`PoolSubpage`，在jemalloc的介绍中，说明Subpage会以第一次请求分配的大小为基准划分，之后也只能进行这个基准大小的内存分配。在`PoolArena`中继续对`PoolSubpage`进行分组，将相同基准的`PoolSubpage`连接成为双向循环链表，便于管理和内存分配。需要注意的是链表头结点head是一个特殊的`PoolSubpage`，不进行实际的内存分配任务。得到链表head节点的代码如下：

```
    PoolSubpage<T> findSubpagePoolHead(int elemSize) {
        int tableIdx;
        PoolSubpage<T>[] table;
        if (isTiny(elemSize)) { // < 512 Tiny
            tableIdx = elemSize >>> 4;
            table = tinySubpagePools;
        } else {    // Small
            tableIdx = 0;
            elemSize >>>= 10;   // 512=0, 1KB=1, 2KB=2, 4KB=3
            while (elemSize != 0) {
                elemSize >>>= 1;
                tableIdx ++;
            }
            table = smallSubpagePools;
        }

        return table[tableIdx];
    }

```

明白了这些，继续分析重要的内存分配方法`allocate()`:

```
    private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, 
                                              final int reqCapacity) {
        // 规范化请求容量
        final int normCapacity = normalizeCapacity(reqCapacity);
        // capacity < pageSize, Tiny/Small请求
        if (isTinyOrSmall(normCapacity)) { 
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512 Tiny请求
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    return; // 尝试从ThreadCache进行分配
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else { // Small请求
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    return; // 尝试从ThreadCache进行分配
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            // 分组的Subpage双向链表的头结点
            final PoolSubpage<T> head = table[tableIdx];

            synchronized (head) {   // 锁定防止其他操作修改head结点
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate(); // 进行分配
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                    return;
                }
            }

            synchronized (this) {
                // 双向循环链表还没初始化，使用normal分配
                allocateNormal(buf, reqCapacity, normCapacity);
            }
            return;
        }

        // Normal请求
        if (normCapacity <= chunkSize) {
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                return; // 尝试从ThreadCache进行分配
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
            }
        } else {
            // Huge请求直接分配
            allocateHuge(buf, reqCapacity);
        }
    }

```

对于Normal和Huge的分配细节如下：

```
    private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || 
            q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || 
            qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            return;
        }

        // 无Chunk或已存Chunk不能满足分配，新增一个Chunk
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);   // Chunk初始状态为QINIT
    }

    private void allocateHuge(PooledByteBuf<T> buf, int reqCapacity) {
        PoolChunk<T> chunk = newUnpooledChunk(reqCapacity);
        buf.initUnpooled(chunk, reqCapacity);
    }

```

总结一下内存分配过程：

1.  对于Tiny/Small、Normal大小的请求，优先从线程缓存中分配。
2.  没有从缓存中得到分配的Tiny/Small请求，会从以第一次请求大小为基准进行分组的Subpage双向链表中进行分配；如果双向链表还没初始化，则会使用Normal请求分配Chunk块中的一个Page，Page以请求大小为基准进行切分并分配第一块内存，然后加入到双向链表中。
3.  没有从缓存中得到分配的Normal请求，则会使用伙伴算法分配满足要求的连续Page块。
4.  对于Huge请求，则直接使用Unpooled直接分配。

内存分配过程分析完毕，接着分析内存释放：

```
    void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
        if (chunk.unpooled) {   // Huge
            int size = chunk.chunkSize();
            destroyChunk(chunk);    // 模板方法，子类实现具体释放过程
        } else {    // Normal, Small/Tiny
            SizeClass sizeClass = sizeClass(normCapacity);
            if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
                return;  // 可以缓存则不释放
            }
            // 否则释放
            freeChunk(chunk, handle, sizeClass);
        }
    }

    void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
        final boolean destroyChunk;
        synchronized (this) {
            // parent为所属的chunkList，destroyChunk为true表示Chunk内存使用装填
            // 从QINIT->Q0->...->Q0，最后释放
            destroyChunk = !chunk.parent.free(chunk, handle);
        }
        if (destroyChunk) {
            destroyChunk(chunk); // 模板方法，子类实现具体释放过程
        }
    }

```

需要注意的是`finalize()`，该方法是`Object`中的方法，在对象被GC回收时调用，可知在该方法中需要清理资源，本类中主要清理内存，代码如下：

```
    protected final void finalize() throws Throwable {
        try {
            super.finalize();
        } finally {
            destroyPoolSubPages(smallSubpagePools); 
            destroyPoolSubPages(tinySubpagePools);
            destroyPoolChunkLists(qInit, q000, q025, q050, q075, q100);
        }
    }

    private static void destroyPoolSubPages(PoolSubpage<?>[] pages) {
        for (PoolSubpage<?> page : pages) {
            page.destroy();
        }
    }

    private void destroyPoolChunkLists(PoolChunkList<T>... chunkLists) {
        for (PoolChunkList<T> chunkList: chunkLists) {
            chunkList.destroy(this);
        }
    }

```

此外，当`PooledByteBuf`容量扩增时，内存需要重新分配，代码如下：

```
    void reallocate(PooledByteBuf<T> buf, int newCapacity, boolean freeOldMemory) {
        int oldCapacity = buf.length;
        if (oldCapacity == newCapacity) {
            return;
        }

        PoolChunk<T> oldChunk = buf.chunk;
        long oldHandle = buf.handle;
        T oldMemory = buf.memory;
        int oldOffset = buf.offset;
        int oldMaxLength = buf.maxLength;
        int readerIndex = buf.readerIndex();
        int writerIndex = buf.writerIndex();

        // 分配新内存
        allocate(parent.threadCache(), buf, newCapacity);
        // 将老数据copy到新内存
        if (newCapacity > oldCapacity) {
            memoryCopy(oldMemory, oldOffset,
                        buf.memory, buf.offset, oldCapacity);
        } else if (newCapacity < oldCapacity) {
            if (readerIndex < newCapacity) {
                if (writerIndex > newCapacity) {
                    writerIndex = newCapacity;
                }
                memoryCopy(oldMemory, oldOffset + readerIndex,
                            buf.memory, buf.offset + readerIndex, writerIndex - readerIndex);
            } else {
                readerIndex = writerIndex = newCapacity;
            }
        }

        // 重新设置读写索引
        buf.setIndex(readerIndex, writerIndex);

        // 如有必要，释放老的内存
        if (freeOldMemory) {
            free(oldChunk, oldHandle, oldMaxLength, buf.cache);
        }
    }

```

最后，由于该类是一个抽象类，其中的抽象方法如下：

```
    // 新建一个Chunk，Tiny/Small，Normal请求请求分配时调用
    protected abstract PoolChunk<T> newChunk(int pageSize, int maxOrder, int pageShifts, int chunkSize);
    // 新建一个Chunk，Huge请求分配时调用
    protected abstract PoolChunk<T> newUnpooledChunk(int capacity);
    // 
    protected abstract PooledByteBuf<T> newByteBuf(int maxCapacity);
    // 复制内存，当ByteBuf扩充容量时调用
    protected abstract void memoryCopy(T src, int srcOffset, T dst, int dstOffset, int length);
    // 销毁Chunk，释放内存时调用
    protected abstract void destroyChunk(PoolChunk<T> chunk);
    // 判断子类实现Heap还是Direct
    protectted abstract boolean isDirect();

```

该类的两个子类分别是`HeapArena`和`DirectArena`，根据底层不同而实现不同的抽象方法。方法简单易懂，不再列出代码。

相关链接：

1.  [JEMalloc分配算法](https://www.jianshu.com/p/15304cd63175)
2.  [PoolChunk](https://www.jianshu.com/p/70181af2972a)
3.  [PoolChunkList](https://www.jianshu.com/p/2b8375df2d1a)
4.  [PoolSubpage](https://www.jianshu.com/p/7afd3a801b15)
5.  [PooThreadCache](https://www.jianshu.com/p/9177b7dabd37)

