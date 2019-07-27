# Unix IO 模型

[Unix 网络 IO 模型及 Linux 的 IO 多路复用模型](http://matt33.com/2017/08/06/unix-io/)
本文主要探讨的问题有以下两个：

1. Unix 中的五种网络 IO 模型；
2. Linux 中 IO 多路复用的实现。

文章目录

1.  基本概念
   1. 1.1. 文件描述符 fd
   2. 1.2. 用户空间与内核空间、内核态与用户态
   3. 1.3. 上下文切换
2.  UNIX 的网络 IO 模型
   1. 2.1. 阻塞 IO 模型
   2. 2.2. 非阻塞 IO 模型
   3. 2.3. IO 多路复用模型
   4. 2.4. 信号驱动 IO 模型
   5. 2.5. 异步 IO 模型
   6. 2.6. 几种 IO 模型比较
3.  Linux 的 IO 多路复用模型
   1. 3.1. select
   2. 3.2. poll
   3. 3.3. epoll
   4. 3.4. 三种模型的区别



在Linux平台上，Java NIO就是基于epoll来实现的。所有基于epoll的框架，都有3个阶段： 
注册事件(connect,accept,read,write)， 轮询IO是否就绪，执行实际IO操作。



# Java IO模型

[谈一谈 Java IO 模型](http://matt33.com/2017/08/12/java-nio/)

# NIO

Java   NIO有几个重要的概念**Channel**,**Buffer**,**Selector**。NIO是基于`Channel`和`Buffer`操作的，数据只能通过`Buffer`写入到`Channel`或者从`Channel`读出数据到`Buffer`中。`Selector`可以监听多个通道的事件（连接打开，数据到达），这样便可以用一个线程监听多个`Channel`的事件，从而可以用一个线程处理多个网络连接。

## Part 1:Buffer核心类讲解

Buffer，本质上是一块内存区，可以用来读写数据，它包含一些要写入或者要读出的数据。在 NIO 中，所有数据都是通过 Buffer 处理的，读取数据时，它是直接读到缓冲区中，写入数据时，写入到缓冲区。

最常用的缓冲区是 ByteBuffer，一个 ByteBuffer 提供了一组功能用于操作 byte 数组，除了 ByteBuffer，还有其他的一些 Buffer，如：CharBuffer、IntBuffer 等，它们之间的关系如下图所示。7种原生数据类型各自对应buffer类型，IntBuffer, LongBuffer...

Buffer 实质上就是一块内存，用于读写数据，这块内存被 NIO Buffer 管理，一个 Buffer 有三个属性是必须掌握的，分别是：

- capacity：容量；
- position：位置；
- limit：限制；

其中，position 和 limit 的具体含义取决于当前 buffer 的模式，capacity 在两种模式下都表示容量，Buffer 读模式和写模式如下图所示。

![](http://matt33.com/images/java/buffer-position.png)

### 状态属性

1. 容量（capacity）
   - Buffer 有一块固定的内存，其大小就是 capacity，一旦 Buffer 写满，就需要清空已读数据以便下次继续写入新的数据；
2. 位置（Position）
   - 写模式时，当写入数据到 Buffer 的时候从一个确定的位置开始，初始化时这个位置 position 为0，写入数据后，position 的值就会指向数据之后的单元，position 最大的值可以达到 `capacity-1`；
   - 读模式时，也需要从一个确定的位置开始，Buffer 从写模式变为读模式时，position 会归零，每次读取后，position 向后移动；
3. 上限（limit）
   - 写模式时，limit 就是能写入的最大数据量，等同于 Buffer 的容量；
   - 读模式时，limit 代表我们能读取的最大容量，它的值等同于写模式下 position 位置。

#### Buffer 常用方法

- `flip()`：把 buffer 从模式调整为读模式，在读模式下，可以读取所有已经写入的数据；
- `clear()`：清空整个 buffer；
- `compact()`：只清空已读取的数据，未被读取的数据会被移动到 buffer 的开始位置，写入位置则紧跟着未读数据之后；
- `rewind()`：将 position 置为0，这样我们可以重复读取 Buffer 中的数据，limit 保持不变；
- `mark()`和`reset()`：通过mark方法可以标记当前的position，通过reset来恢复mark的位置
- `equals()`：判断两个 Buffer 是否相等，需满足：类型相同、Buffer 中剩余字节数相同、所有剩余字节相等；
- `compareTo()`：compareTo 比较 Buffer 中的剩余元素，只不过这个方法适用于比较排序的。



### Transferring data

对于其子类，定义两类操作get, put

- Relative operations

  > 从当前position开始，根据number of elements transferred增加position

- Absolute operations

  > take an explicit element index and do not affect the position

### mark

缓冲区的*标记* 是一个索引，在调用 [`reset`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/Buffer.html#reset()) 方法时会将缓冲区的位置重置为该索引。并非总是需要定义标记，但在定义标记时，不能将其定义为负数，并且不能让它大于位置。如果定义了标记，则在将位置或限制调整为小于该标记的值时，该标记将被丢弃。

标记、位置、限制和容量值遵守以下不变式：

0 <= mark <= position <= limit <= capacity

### 基本操作

> 除了访问位置、限制、容量值的方法以及做标记和重置的方法外，此类还定义了以下可对缓冲区进行的操作：
>
> - [`clear()`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/Buffer.html#clear()) 使缓冲区为一系列新的通道读取或相对*放置* 操作做好准备：它将限制设置为容量大小，将位置设置为 0。
> - [`flip()`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/Buffer.html#flip()) 使缓冲区为一系列新的通道写入或相对*获取* 操作做好准备：它将限制设置为当前位置，然后将位置设置为 0。
> - [`rewind()`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/Buffer.html#rewind()) 使缓冲区为重新读取已包含的数据做好准备：它使限制保持不变，将位置设置为 0。

### ByteBuffer

字节缓冲区。

此类针对字节缓冲区定义了以下六类操作：

- 读写单个字节的绝对和相对 [``*get*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#get()) 和 [``*put*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#put(byte)) 方法；
- 将此缓冲区的连续字节序列传输到数组中的相对[``*批量 get*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#get(byte[])) 方法；
- 将 byte 数组或其他字节缓冲区中的连续字节序列传输到此缓冲区的相对[``*批量 put*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#put(byte[])) 方法；
- 读写其他基本类型值，并按照特定的字节顺序在字节序列之间转换这些值的绝对和相对 [``*get*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#getChar()) 和 [``*put*``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#putChar(char)) 方法；
- 创建*视图缓冲区* 的方法，这些方法允许将字节缓冲区视为包含其他基本类型值的缓冲区；
- 对字节缓冲区进行 [``compacting``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#compact())、[``duplicating``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#duplicate()) 和 [``slicing``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#slice()) 的方法。

#### 直接*与* 非直接缓冲区

字节缓冲区要么是*直接的*，要么是*非直接的*。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在此缓冲区上执行本机 I/O 操作。也就是说，在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。

直接字节缓冲区可以通过调用此类的 [`allocateDirect`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#allocateDirect(int)) 工厂方法来创建。此方法返回的缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的本机 I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。

直接字节缓冲区还可以通过 [``mapping``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/FileChannel.html#map(java.nio.channels.FileChannel.MapMode,%20long,%20long)) 将文件区域直接映射到内存中来创建。Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在访问期间或稍后的某个时间导致抛出不确定的异常。

#### 访问二进制数据

此类定义了除 `boolean` 之外，读写所有其他基本类型值的方法。这些基本值可以根据缓冲区的当前字节顺序与字节序列互相进行转换，并可以通过 [`order`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#order()) 方法获取和修改。特定的字节顺序由 [`ByteOrder`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteOrder.html) 类的实例表示。字节缓冲区的初始顺序始终是 [`BIG_ENDIAN`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteOrder.html#BIG_ENDIAN)。

为了访问异类二进制数据（即其他类型的值的序列），此类还针对每种类型定义了一系列绝对和相对的 *get* 和 *put* 方法。例如，对于 32 位浮点值，此类定义了以下方法：

> ```
>  float  getFloat()
>  float  getFloat(int index)
>   void  putFloat(float f)
>   void  putFloat(int index, float f)
> ```

并针对 `char`、`short`、`int`、`long` 和 `double` 等类型定义了相应的方法。绝对 *get* 和 *put* 方法的 index 参数是根据字节定义的，而不是根据所读写的类型定义的。

为了访问同类二进制数据（即相同类型的值的序列），此类还定义了可以为指定类型的缓冲区创建*视图* 的方法。*视图缓冲区* 只是其内容受该字节缓冲区支持的另一种缓冲区。字节缓冲区内容的更改在视图缓冲区中是可见的，反之亦然；这两种缓冲区的位置、限制和标记值都是独立的。例如，[`asFloatBuffer`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html#asFloatBuffer()) 方法将创建 [`FloatBuffer`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/FloatBuffer.html) 类的一个实例，该类受在其上调用该方法的字节缓冲区的支持。该类将为 `char`、`short`、`int`、`long` 和 `double` 等类型定义相应的视图创建方法。

与上述特定于类型的 *get* 和 *put* 方法系列相比，视图缓冲区有以下三大主要优势：

- 视图缓冲区不是根据字节进行索引，而是根据其特定于类型的值的大小进行索引；
- 视图缓冲区提供了相对批量 *get* 和 *put* 方法，这些方法可在缓冲区和数组或相同类型的其他缓冲区之间传输值的连续序列；
- 视图缓冲区可能更高效，这是因为，当且仅当其支持的字节缓冲区为直接缓冲区时它才是直接缓冲区。

视图缓冲区的字节顺序固定为创建该视图时其字节缓冲区的字节顺序。

### DirectByteBuffer

存储在java堆上的标志java对象，但是持有一个在jvm堆外的操作系统原生创建的一块内存空间的引用address，即直接内存模型。若java堆上数据想外界发生数据交换，则需通过操作系统native方法进行一次额外数据拷贝，然后再进行数据交互。通过这样方式，实现io交换零拷贝。

![](https://raw.githubusercontent.com/seaside2mm/github-photos/master/Screen%20Shot%202019-01-05%20at%202.35.14%20PM.png)

传统：

1. 操作系统：磁盘 —》 内核空间的页缓存
2. 应用程序：内核空间 —〉用户空间缓存
3. 应用程序：用户空间缓存  —》内核空间socket缓存
4. 操作系统：内核空间socket缓存 —》 网卡缓冲区，网络发出

零拷贝：

1. 操作系统：磁盘 —》 内核空间的页缓存
2. 操作系统：将数据位置，长度信息的描述符增加到内核空间socket缓存
3. 操作系统：内核拷贝到网卡缓冲区



#### MappedByteBuffer

直接字节缓冲区，其内容是文件的内存映射区域。

映射的字节缓冲区是通过 [`FileChannel.map`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/FileChannel.html#map(java.nio.channels.FileChannel.MapMode,%20long,%20long)) 方法创建的。此类用特定于内存映射文件区域的操作扩展 [`ByteBuffer`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/ByteBuffer.html) 类。

映射的字节缓冲区和它所表示的文件映射关系在该缓冲区本身成为垃圾回收缓冲区之前一直保持有效。

映射的字节缓冲区的内容可以随时更改，例如，在此程序或另一个程序更改了对应的映射文件区域的内容的情况下。这些更改是否发生（以及何时发生）与操作系统无关，因此是未指定的。

全部或部分映射的字节缓冲区可能随时成为不可访问的，例如，如果我们截取映射的文件。试图访问映射的字节缓冲区的不可访问区域将不会更改缓冲区的内容，并导致在访问时或访问后的某个时刻抛出未指定的异常。因此强烈推荐采取适当的预防措施，以避免此程序或另一个同时运行的程序对映射的文件执行操作（读写文件内容除外）。



## Part 2:Channel通道

### Channel继承结构

![](https://raw.githubusercontent.com/seaside2mm/github-photos/master/20160923131810163.jpg)



```java
public interface Channel extends Closeable {
    public boolean isOpen();
    public void close() throws IOException;
}

public interface ReadableByteChannel extends Channel {
    public int read(ByteBuffer dst) throws IOException;
}

public interface WritableByteChannel extends Channel {
    public int write(ByteBuffer src) throws IOException;
}
```

一个Channel最基本的操作就是read/write，并且其传进去的必须是ByteBuffer类型，而不是普通的内存buffer。

### FileChannel文件通道

> Channel用于 I/O 操作的连接。通道表示到实体，如硬件设备、文件、网络套接字或可以执行一个或多个不同 I/O 操作（如读取或写入）的程序组件的开放的连接。通道可处于打开或关闭状态。创建通道时它处于打开状态，一旦将其关闭，则保持关闭状态。

[api](http://tool.oschina.net/apidocs/apidoc?api=jdk-zh)

用于读取、写入、映射和操作文件的通道。

*特点*

- connected to a file

File

- file has a current position that can be queried and modified
- The file itself contains a variable-length sequence of bytes that can be read and written and size can be queried 
- the file may also have some associated <i>metadata</i> such as access permissions, content type, and last-modification time;

操作

- 基本：familiar read, write, and close operations of byte channels

- 从文件的absolute position，向ByteBuffer读取/写入字节，并且不会影响channel的当前position

- A region of a file may be  <i>mapped</i>  directly into memory;对于大文件的读写操作更有效率

  > MappedByteBuffer的map方法

- Updates made to a file may be  <i>forced out</i> to the underlying storage device, ensuring that data are not lost in the event of a system crash. 

- Bytes can be transferred from a file {@link #transferTo <i>to some other channel</i>}, and {@link #transferFrom <i>vice versa</i>}, in a way that can be optimized by many operating systems into a very fast transfer directly to or from the filesystem cache.

- A region of a file may be {@link FileLock <i>locked</i>} against access by other programs. 

主要关注三个类:

`SocketChannel`, `ServerSocketChannel` 以及`DataGramChannel`

其中`SocketChannel`是一个连接到TCP网络套接字的通道，可由两种方式创建：

1. 打开一个`SocketChannel`并连接到互联网上的某台服务器。
2. 一个新连接到达`ServerSocketChannel`时，会创建一个`SocketChannel`(即由`serverSocketChannel.accept()`方法返回)。

#### ServerSocketChannel

针对面向流的侦听套接字的可选择通道。

服务器套接字通道不是侦听网络套接字的完整抽象。必须通过调用 [`socket`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/ServerSocketChannel.html#socket()) 方法所获得的关联 [`ServerSocket`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/net/ServerSocket.html) 对象来完成对套接字选项的绑定和操作。不可能为任意的已有服务器套接字创建通道，也不可能指定与服务器套接字通道关联的服务器套接字所使用的 [`SocketImpl`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/net/SocketImpl.html)对象。

通过调用此类的 [`open`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/ServerSocketChannel.html#open()) 方法创建服务器套接字通道。新创建的服务器套接字通道已打开，但尚未绑定。试图调用未绑定的服务器套接字通道的 [`accept`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/ServerSocketChannel.html#accept()) 方法会导致抛出 [`NotYetBoundException`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/NotYetBoundException.html)。可通过调用相关服务器套接字的某个 [`bind`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/net/ServerSocket.html#bind(java.net.SocketAddress,%20int)) 方法来绑定服务器套接字通道。

## Part 3 :Selector选择器

Selector 是 Java NIO 核心部分，Selector的主要目的是网络事件的 loop 循环，通过调用selector.poll，不断轮询每个Channel上读写事件。简单来说，它的作用就是：Selector 不断轮询注册在其上的 Channel，如果某个 Channel 上面有新的 TCP 连接、读和写事件，这个 Channel 就处于就绪状态，会被 Selector 轮询出来，然后通过 `SelectorKey()` 可以获取就绪 Channel 的集合，进行后续的 IO 操作。

一个 Selector 可以轮询多个 Channel，由于 JDK 底层使用了 `epoll()` 实现，它并没有最大连接句柄 1024/2048 的限制，这就意味着只需要一个线程负责 Selector 的轮询，就可以连接上千上万的 Client。





### api翻译

[`SelectableChannel`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectableChannel.html) 对象的多路复用器。

可通过调用此类的 [`open`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html#open()) 方法创建选择器，该方法将使用系统的默认[``选择器提供者``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/spi/SelectorProvider.html)创建新的选择器。也可通过调用自定义选择器提供者的 [`openSelector`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/spi/SelectorProvider.html#openSelector()) 方法来创建选择器。通过选择器的 [`close`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html#close()) 方法关闭选择器之前，它一直保持打开状态。

通过 [`SelectionKey`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectionKey.html) 对象来表示可选择通道到选择器的注册。选择器维护了三种选择键集：

- *键集*key sets 包含的键表示当前通道到此选择器的注册。此集合由 [`keys`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html#keys()) 方法返回。
- *已选择键集selected-key set* 是这样一种键的集合，即在前一次选择操作期间，检测每个键的通道是否已经至少为该键的相关操作集所标识的一个操作准备就绪。此集合由 [`selectedKeys`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html#selectedKeys()) 方法返回。已选择键集始终是键集的一个子集。
- *已取消键集*cancelled-key set 是已被取消但其通道尚未注销的键的集合。不可直接访问此集合。已取消键集始终是键集的一个子集。

在新创建的选择器中，这三个集合都是空集合。

通过某个通道的 [`register`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectableChannel.html#register(java.nio.channels.Selector,%20int)) 方法注册该通道时，所带来的副作用是向选择器的键集中添加了一个键。在选择操作期间从键集中移除已取消的键。键集本身是不可直接修改的。

不管是通过关闭某个键的通道还是调用该键的 [`cancel`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectionKey.html#cancel()) 方法来取消键，该键都被添加到其选择器的已取消键集中。取消某个键会导致在下一次选择操作期间注销该键的通道，而在注销时将从所有选择器的键集中移除该键。

通过选择操作将键添加到已选择键集中。可通过调用已选择键集的 [`remove`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/Set.html#remove(java.lang.Object)) 方法，或者通过调用从该键集获得的 [``iterator``](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/Iterator.html) 的 [`remove`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/Iterator.html#remove())方法直接移除某个键。通过任何其他方式从不会将键从已选择键集中移除；特别是，它们不会因为影响选择操作而被移除。不能将键直接添加到已选择键集中。

通过register方法，



- select()阻塞直到注册的某个事件就绪并会更新SelectionKey的状态
- selectedKeys()得到就绪的key集合，key中保存有就绪的事件以及对应的Channel通道

#### 注册 Channel

举一个栗子，简单介绍 `Selector` 的使用。

```java
// 创建一个 Selector
Selector selector = Selector.open();
// 将一个 Channel 注册到 Selector 上
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

`register()` 的第二个参数代表的是 selector 监听的事件类型，Selector 可以监听事件类型总共有以下四种：

1. SelectionKey.OP_CONNECT：只会注册一次，成功之后（TCP 连接建立之后），这个监听事件就取消了；
2. SelectionKey.OP_ACCEPT：主要用于服务端，就是监听是否有新的连接请求；
3. SelectionKey.OP_READ：注册之后不会取消，监听是否数据到来；
4. SelectionKey.OP_WRITE：最好的使用方法是每当发送数据时，就注册一次，然后再取消，否则每次 select 轮询时，注册 OP_WRITE 事件的 Channel 都是 ready 的，除非 socket send buffer 满了

#### SelectionKey

SelectionKey用来记录一个Channel上的事件集合，每个Channel对应一个SelectionKey。 
SelectionKey也是Selector和Channel之间的关联，通过SelectionKey可以取到对应的Selector和Channel。

#### Api翻译

表示 [`SelectableChannel`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectableChannel.html) 在 [`Selector`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html) 中的注册的标记。

每次向选择器注册通道时就会创建一个选择键。通过调用某个键的 [`cancel`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectionKey.html#cancel()) 方法、关闭其通道，或者通过关闭其选择器来*取消* 该键之前，它一直保持有效。取消某个键不会立即从其选择器中移除它；相反，会将该键添加到选择器的[*已取消键集*](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/Selector.html#ks)，以便在下一次进行选择操作时移除它。可通过调用某个键的 [`isValid`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectionKey.html#isValid()) 方法来测试其有效性。

选择键包含两个表示为整数值的*操作集*。操作集的每一位都表示该键的通道所支持的一类可选择操作。

- *interest 集合* 确定了下一次调用某个选择器的选择方法时，将测试哪类操作的准备就绪信息。创建该键时使用给定的值初始化 interest 集合；之后可通过 [`interestOps(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/SelectionKey.html#interestOps(int)) 方法对其进行更改。
- *ready 集合* 标识了这样一类操作，即某个键的选择器检测到该键的通道已为此类操作准备就绪。创建该键时 ready 集合被初始化为零；可以在之后的选择操作中通过选择器对其进行更新，但不能直接更新它。



#### NIO Scatter/Gather

Java NIO 发布时内置了对 scatter/gather的支持：

- Scattering read 指的是从通道读取的操作能把数据写入多个 Buffer，也就是 sctters 代表了数据从一个 Channel 到多个 Buffer的过程。
- Gathering write 则正好相反，表示的是从多个 Buffer 把数据写入到一个 Channel中



下面的表格展示了connect, accept, read, write 这4种事件，分别在这3个阶段对应的函数：

![](https://raw.githubusercontent.com/seaside2mm/github-photos/master/20160923141539570.jpg)

## NIO 原理

Java NIO 实现的关键是 IO 多路复用（具体可以参考上篇文章：[Linux 的 IO 多路复用模型](http://matt33.com/2017/08/06/unix-io/#Linux-%E7%9A%84-IO-%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E6%A8%A1%E5%9E%8B)），在 Linux 平台，Java NIO 是基于 epoll（2.6以上，之前是 Select） 来实现的。

Linux 的 select/epoll 使用的是 Reactor 网络 IO 模式。网络编程中，有两种常用的设计模式，它们都是基于事件驱动：

- Reactor 模式：主动模式，应用程序不断去轮询，问操作系统 IO 是否就绪，实际的 IO 操作还是由应用实现（IO 多路复用采用的模式）；
- Proactor 模式：被动模式，操作系统把 IO 完成后通知应用程序，此时数据已经就绪。

这两种模式详细内容可以参考[两种高性能 I/O 设计模式 Reactor 和 Proactor](http://daoluan.net/linux/%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/2013/08/20/two-high-performance-io-design-patterns.html)一文。





[自顶向下深入分析Netty（一）--预备知识](https://www.jianshu.com/p/80a2db190f42)