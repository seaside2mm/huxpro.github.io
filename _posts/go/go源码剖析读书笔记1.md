# 一切的起点

### 汇编引导

### 初始化

命令⾏参数整理，环境变量设置，以及内存分配器、垃圾回收器和并发调度器的⼯作现场准备

#### 调度器初始化

schedinit：要关注的所有运⾏时环境初始化构造都在这⾥被调⽤, `proc.go` ;

```go
// call osinit
// call schedinit
// make & queue new G
// call runtime·mstart

```

包初始化函数 init 的执⾏`proc.go`

# 内存分配

## 2.1 概述

#### 2.1.1 内存块种类

span

> 分配器以页为单位向操作系统申请大块内存。这些大块内存由 n 个地址连续的页组成，并用名为 span 的对象进行管理。
>
> 按照特定size class进行切分。

object

> 当需要时，span 所管理内存被切分成多个大小相等的小块，每个小块可存储一个对象，故称作 object。

分配器以 32KB 为界，将对象分为大小两种。

```go
MaxSmallSize = 32<<10,
```

大对象直接找一个大小合适的 span，这个无需多言。小对象则以 8 的倍数分为不同大小等级 (size class)。比如 class1 为 8 字节，可存储 1 ~ 8 字节大小的对象。

```go
NumSizeClasses = 67,
```

当然，实际的对应规则并不是连续和固定的，会根据一些经验和测试结果进行调整，以获得最佳的性能和内存利用率。



#### 2.2.2 分配器三级组件

heap

> 全局根对象。负责向操作系统申请内存，> 1M
>
> 管理由未使用的或者垃圾回收器收回的空闲 span 内存块。

central 

> 从 heap 获取空闲 span，并按需要将其切分成 object 块。不同size class请求被分散到不同central组件，减小锁粒度
>
> heap 管理着多个central 对象，每个 central 负责处理一种等级的内存分配需求。

cache

> 运行期，每个 cache 都与某个具体线程相绑定，实现无锁内存分配操作。
>
> 其内部有个以等级为序号的数组，持有多个切分好的 span 对象。缺少空间时，向等级对应的 central 获取新的 span 即可。

## 2.2 初始化

#### 2.2.1 问题分析

Q1: 如何在对象没有额外附加字段，可内存分配和垃圾回收却要设置管理状态，并且如何存储和高效检索？

思路：根据对象地址，计算索引号，并利用专门数组存储以访问。

Q2: 分配对象如何同时确保其对应有管理状态的元数据数组能同时分配？并且地址不被占用？

思路：指定二者所需虚拟地址空间，采用线性分配策略

#### 2.2.2 内存布局

分配器管理算法依赖连续内存地址。因此，在初始化时，分配器会预留一块巨大的虚拟地址空间。该空间被成三个部分：

- arena: 用户内存实际分配范围。
- bitmap: 为每个地址提供 4bit 标记位，用于垃圾回收操作。
- spans: 记录每个页所对应 span 地址，用于反查和合并操作。

![img](https://7n.w3cschool.cn/attachments/image/wk/thegostudynotesfourthedition/21.png)

在 64 位系统下，arena 最大容量是 128GB，bitmap 8GB，spans 128MB。这些内存并非一次性分配，而是随着 arena 线性增加，每个区域都有指针标记当前分配位置。

```go
type mheap struct {
    spans []*mspan
    
    bitmap uintptr // Points to one byte past the end of the bitmap
    bitmap_mapped uintptr
    
    arena_start uintptr
    arena_used uintptr // Set with setArenaUsed.
    arena_alloc uintptr
    arena_end uintptr
}
```



`mallocinit`函数进行初始化



## 分配

流程：

- 通过 size class 反查表计算待分配对象等级。
- 从 cache.alloc[sizeclass] 找到等级相同的 span。
- 从 span 切分好的链表中提取可用 object。
- 如 span 没剩余空间，则从 heap.central[sizeclass] 找到对应 central，获取 span。
- 如 central 没可用 span，则向 heap 申请，并切割成所需等级的 object 链表。
- 如 heap 也没有多余 span，那么就向操作系统申请新的内存。

### 零长度对象

### 大对象

#### 管理方式

- 数组free
- 树freelarge

#### 获取

#### 分割

#### 合并

#### 申请

### 小对象





## 回收

流程：

- 垃圾回收器或其他行为引发内存回收操作。
- 将可回收 object 交还给所属 span。
- 将 span 交给对应 central 管理，以便某个 cache 重新获取。
- 如 span 内存全部收回，那么将其返还给 heap，以便被重新切分复用。
- 垃圾回收器定期扫描 heap 所管理的空闲 spans，释放超期不用的物理内存。

从 heap 申请和回收 span 的过程中，分配器会尝试合并地址相邻的 span 块，以形成更大内存块，减少碎片。

## 释放



# 垃圾回收

