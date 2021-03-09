## Netty

[TOC]

### 为什么选用Netty来做通信框架

Netty 是一个**基于 NIO** 的 client-server网络应用程序框架，封装了 JDK 的 NIO，让我们使用起来更加灵活方便。

使用Netty主要因为其以下特点和优势：

- **使用简单**：封装了 NIO 的很多细节，使用更简单。 
- **功能强大**：预置了多种编解码功能，支持多种主流协议。 
- **定制能力强**：可以通过 ChannelHandler 对通信框架进行灵活地扩展。 
- **性能高**：通过与其他业界主流的 NIO 框架对比，Netty的综合性能最优。 

**为什么不用NIO？**

- 不用NIO主要是因为NIO的编程模型复杂而且存在一些BUG，并且对编程功底要求比较高。而且，NIO在面对断连重连、包丢失、粘包等问题时处理过程非常复杂。

---

### Reactor和Proactor模型

I/O多路复用机制都依赖于一个事件**多路分离器(Event Demultiplexer)**。分离器对象可将来自事件源的I/O事件分离出来，并分发到对应的**read/write事件处理器(Event Handler)**

**两个与事件分离器有关的模式是Reactor和Proactor。Reactor模式采用同步IO，而Proactor采用异步IO。**

#### 1. Reactor

Reactor的中心思想是将所有要处理的I/O事件注册到一个中心I/O多路复用器上，同时主线程/进程阻塞在多路复用器上；一旦有I/O事件到来或是准备就绪，多路复用器返回，并将事先注册的相应I/O事件分发到对应的处理器中

**相关概念：**

- **事件**：就是状态；比如：**读就绪事件**指的是我们可以从内核读取数据的状态

- **事件分离器**：一般会把事件的等待发生交给epoll、select；而事件的到来是随机，异步的，所以需要循环调用epoll，在框架里对应封装起来的模块就是事件分离器（简单理解为对epoll封装）

- **事件处理器**：事件发生后需要进程或线程去处理，这个处理者就是事件处理器，一般和事件分离器是不同的线程

**Reactor的一般流程**

1）应用程序在**事件分离器**注册**读写就绪事件**和**读写就绪事件处理器**

2）事件分离器等待读写就绪事件发生

3）读写就绪事件发生，激活事件分离器，分离器调用读写就绪事件处理器

4）事件处理器先从内核把数据读取到用户空间，然后再处理数据


![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/2052f411dc7041e792c4c8b1a4700c8d~tplv-k3u1fbpfcp-watermark.image)

取决于Reactor的数量和Hanndler线程数量的不同，Reactor模型有3个变种：[Reactor三种模型](https://blog.csdn.net/Woo_home/article/details/106119218)

- 单Reactor单线程
- 单Reactor多线程
- 主从Reactor多线程

可以这样理解，Reactor就是一个执行while (true) { selector.select(); ...}循环的线程，会源源不断的产生新的事件，称作反应堆很贴切。

#### 2. Proactor

**Proactor的一般流程**

1）应用程序在事件分离器注册**读完成事件**和**读完成事件处理器**，并向系统发出异步读请求

2）事件分离器等待读事件的完成

3）在分离器等待过程中，系统利用并行的内核线程执行实际的读操作，并将数据复制进程缓冲区，最后通知事件分离器读完成到来

4）事件分离器监听到**读完成事件**，激活**读完成事件的处理器**

5）读完成事件处理器直接处理用户进程缓冲区中的数据

#### 3. Proactor和Reactor的区别

- Proactor是基于异步I/O的概念，而Reactor一般则是基于同步I/O的概念
- Proactor中，当回调handler时，表示IO操作已经完成；Reactor中，回调handler时，表示IO设备可以进行某个操作(can read or can write)。
- Proactor不需要把数据从内核复制到用户空间，这步由系统完成

---

### Netty IO模型

Netty主要**基于主从Reactors多线程模型**做了一定的修改，其中主从Reactor多线程模型有多个Reactor：MainReactor和SubReactor：

 有多个Reactor：MainReactor和SubReactor：

- MainReactor负责客户端的连接请求，并将请求转交给SubReactor
- SubReactor负责相应通道的IO读写请求
- 非IO请求（具体逻辑处理）的任务则会直接写入队列，等待worker threads进行处理

Netty的主Reactor体现为bossGroup，从Reactor体现为workerGroup。bossGroup不断监听是否有客户端的连接，当发现一个新的客户端连接到来时，bossGroup就会为此链接初始化各种资源，然后从workerGroup中选一个EventLoop绑定到此客户端的链接中。接下来服务器与客户端的链接，就全在此分配的eventLoop中了Selector

**Selector**

Netty基于Selector对象实现I/O多路复用，通过 Selector, 一个线程可以监听多个连接的Channel事件, 当向一个Selector中注册Channel 后，Selector 内部的机制就可以自动不断地查询(select) 这些注册的Channel是否有已就绪的I/O事件(例如可读, 可写, 网络连接完成等)，这样程序就可以很简单地使用一个线程高效地管理多个 Channel 

#### IO多路复用

**1. Select**

文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述符就绪（有数据可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过遍历fdset，来找到就绪的描述符

**缺陷:** 需要轮询所有fd，时间复杂度 O(n) n有上限1024(32位),2048(64位) fd集合会从用户态拷贝到内核态

**2. Poll**

实现和select非常相似，只是描述fd集合的方式不同，没有最大连接数的限制（原因是它是基于链表来存储的）

**缺陷:** 需要轮询所有fd，时间复杂度 O(n) fd集合会从用户态拷贝到内核态 水平触发（LT），如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd

**3. Epoll**

**epoll可以理解为event poll**，**一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知**。(此处去掉了遍历文件描述符，而是通过监听回调的机制)

**优点:** 通过监听回调机制，时间复杂度是 O(1) 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销 没有最大并发连接的限制，能打开的FD的上限远大于1024 支持边缘触发(ET)

**5. 两种触发模式**

LT是默认的模式，ET是“高速”模式。LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作，而在ET（边缘触发）模式中，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读。

**ET 模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符**

如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。而采用EPOLLET这种边沿触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你

---

### netty高性能主要依赖了哪些特性

- **IO 线程模型**：同步非阻塞，用最少的资源做更多的事。 

- **内存零拷贝**：尽量减少不必要的内存拷贝，实现了更高效率的传输。 

- **内存池设计**：申请的内存可以重用，主要指直接内存。内部实现是用一颗二叉查找树管理内存分配情况。 

- **串行化处理读写**：避免使用锁带来的性能开销。 

- **高性能序列化协议**：支持 protobuf ，Kryo等高性能序列化协议。

---

### Netty Bytebuf工作原理，和NIO ByteBuffer区别

**1. ByteBuffer**

- ByteBuffer长度固定，一旦分配完成，它的容量不能动态扩展和收缩，当需要编码的对象大于ByteBuffer的容量时，会发生索引越界异常；
- ByteBuffer只有一个标识位置的指针position，读写的时候需要手工调用flip()和rewind()等
- ByteBuffer的API功能有限

**2. ByteBuf**

- 长度可以实现动态扩展
- 通过两个位置指针来协助缓冲区的读写操作，读操作使用readerIndex，写操作使用writerIndex
- **内存分配：**可以分**为堆内存（HeapByteBuf）字节缓冲区**和**直接内存（DirectByteBuf）字节缓冲区**。直接内存缓冲区中分配和回收速度相对较慢，但是读写时少了一次内存复制，速度更快。
- **内存回收角度：**分为**基于对象池的ByteBuf**和**普通ByteBuf**，基于对象池的ByteBuf可以重用ByteBuf对象，它自己维护了一个内存池，可以循环利用创建的ByteBuf，提升内存的使用效率。

---

### Netty怎么处理粘包/拆包

TCP 粘包/拆包 就是你基于 TCP 发送数据的时候，出现了多个字符串“粘”在了一起或者一个字符串被“拆”开的问题

**1. 使用 Netty 自带的解码器**

- **`LineBasedFrameDecoder`** : 发送端发送数据包的时候，每个数据包之间以换行符作为分隔，`LineBasedFrameDecoder` 的工作原理是它依次遍历 `ByteBuf` 中的可读字节，判断是否有换行符，然后进行相应的截取。
- **`DelimiterBasedFrameDecoder`** : 可以自定义分隔符解码器，**`LineBasedFrameDecoder`** 实际上是一种特殊的 `DelimiterBasedFrameDecoder` 解码器。
- **`FixedLengthFrameDecoder`**: 固定长度解码器，它能够按照指定的长度对消息进行相应的拆包。

**2.自定义序列化编解码器**

在 Java 中自带的有实现 `Serializable` 接口来实现序列化，但由于它性能、安全性等原因一般情况下是不会被使用到的。通常情况下，我们使用 Protostuff、Kryo等序列方式比较多。

自定义编解码器的重点，在于自定义序列化协议。项目中的序列化协议如下：

![image-20210221185918122](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/image-20210221185918122.png)

- **魔数** ： 通常是 4 个字节。这个魔数主要是为了筛选来到服务端的数据包，有了这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接以节省资源。
- **序列化器编号** ：标识序列化的方式，比如是使用 Java 自带的序列化，还是 json，kyro 等序列化方式。
- **消息体长度** ： 运行时计算出来。
- **RequestId**：每一个请求的唯一识别id（由于采用异步通讯的方式，用来把请求request和返回的response对应上）

---

### Netty长连接，心跳机制

**一、长连接**

- TCP 在进行读写之前，server 与 client 之间必须提前建立一个连接。建立连接的过程，需要我们常说的三次握手，释放/关闭连接的话需要四次挥手。这个过程是比较消耗网络资源并且有时间延迟的。

- **短连接**：说的就是 server 端 与 client 端建立连接之后，读写完成之后就关闭掉连接，如果下一次再要互相发送消息，就要重新连接。
  - 优点：就是管理和实现都比较简单
  - 缺点：每一次的读写都要建立连接，会带来大量网络资源的消耗，并且连接的建立也需要耗费时间。

- **长连接**：说的就是 client 向 server 双方建立连接， 完成一次读写后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。
  - 优点：可以省去较多的 TCP 建立和关闭的操作，降低对网络资源的依赖，节约时间。对于频繁请求资源的客户来说，非常适用长连接。

**二、心跳机制**

在 TCP 保持长连接的过程中，可能会出现断网等网络异常出现，异常发生的时候， client 与 server 之间如果没有交互的话，它们是无法发现对方已经掉线的。为了解决这个问题，我们就需要引入 **心跳机制** 。

**心跳机制的工作原理**： 在 client 与 server 之间在一定时间内没有数据交互时， 即处于 idle 状态时，客户端或服务器就会发送一个特殊的数据包给对方，当接收方收到这个数据报文后，也立即发送一个特殊的数据报文，回应发送方，此即一个 PING-PONG 交互。所以，当某一端收到心跳消息后，就知道了对方仍然在线，这就确保 TCP 连接的有效性。

TCP 实际上自带的就有长连接选项，本身是也有心跳包机制，但是该机制默认的心跳时间是2小时，依赖操作系统实现不够灵活；所以通常在应用层实现自定义心跳机制，比如Netty实现心跳机制；

> Netty通过 IdleStateHandler 实现最常见的心跳机制不是一种双向心跳的PING-PONG模式，而是客户端发送心跳数据包，服务端接收心跳但不回复，因为如果服务端同时有上千个连接，心跳的回复需要消耗大量网络资源；如果服务端一段时间内没有收到客户端的心跳数据包则认为客户端已经下线，将通道关闭避免资源的浪费；在这种心跳模式下服务端可以感知客户端的存活情况，无论是宕机的正常下线还是网络问题的非正常下线，服务端都能感知到

https://blog.csdn.net/u013967175/article/details/78591810

---

### 还知道其他网络通信框架

Mina，和Netty同一个人设计的

与Mina相比有什么优势：

1. 都是Trustin Lee的作品，Netty更晚；
2. Mina将内核和一些特性的联系过于紧密，使得用户在不需要这些特性的时候无法脱离，相比下性能会有所下降，Netty解决了这个设计问题；
3. Netty的文档更清晰，很多Mina的特性在Netty里都有；
4. Netty更新周期更短，新版本的发布比较快；
5. 它们的架构差别不大，Mina靠apache生存，而Netty靠jboss，和jboss的结合度非常高，Netty有对google protocal buf的支持，有更完整的ioc容器支持(spring,guice,jbossmc和osgi)；
6. Netty比Mina使用起来更简单，Netty里你可以自定义的处理upstream events或/和downstream events，可以使用decoder和encoder来解码和编码发送内容；
7. Netty和Mina在处理UDP时有一些不同，Netty将UDP无连接的特性暴露出来；而Mina对UDP进行了高级层次的抽象，可以把UDP当成&quot;面向连接&quot;的协议，而要Netty做到这一点比较困难。
8. 从任务调度粒度，mina会将有IO任务的session写入队列中，当循环执行任务时，则会轮询所有的session，并依次把session中的所有任务取出来运行。这样粗粒度的调度是不公平调度，会导致某些请求的延迟很高。

---

### 项目如果要实现内存零拷贝怎么做

Netty的内存零拷贝体现在：

1. **DIRECT BUFFERS（堆外内存）**：Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
2. **CompositeByteBuf**：Netty 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那样方便的对组合 Buffer 进行操作，这样就免去了重新分配空间再复制数据的开销。
3. **FileRegion类的transferTo()**：Netty 的文件传输采用了 transferTo 方法，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。

参考链接：https://zhuanlan.zhihu.com/p/88599349

---

### 怎么实现序列化，Kryo原理

实现了两种序列化方式，一种是kryo，一种是Protobuf

**1. Kryo**

Kryo是专门专对Java语言的，性能非常好

- Kryo在做序列化时，也**没有记录属性的名称，而是给每个属性分配了一个id**，但是他却并没有GPB那样通过一个schema文件去做id和属性的一个映射描述，所以一旦我们修改了对象的属性信息，比如说新增了一个字段，那么Kryo进行反序列化时就可能发生属性值错乱甚至是反序列化失败的情况
- 由于Kryo没有序列化属性名称的描述信息，所以序列化/反序列化之前，需要先将要处理的类**在Kryo中进行注册**，这一操作在首次序列化时也会消耗一定的性能

**2. Protobuf**

出自Google，性能比较优秀，支持多种语言，同时跨平台。使用比较繁琐，需要自定义proto文件，用来完成Java对象中的基本数据类型和GPB自己定义的类型之间的一个映射。Java中通过与ProtoStuff的结合，使用更加简便了，无需手动编写proto文件

---

### Java Serializable和Externalizable

**1. Serializable**

**性能**：Java序列化会把要序列化的对象类的**元数据和业务数据全部序列化**成字节流，而且是**把整个继承关系上的东西全部序列化**了。它序列化出来的字节流是对那个对象结构到内容的完全描述，包含所有的信息，因此效率较低而且字节流比较大。[序列化过程](https://www.jb51.net/article/36408.htm)

**安全性**：比如一个对象拥有private，public等field，对于一个要传输的对象，比如写到文件，或者进行rmi传输等等，在序列化进行传输的过程中，这个对象的**private域是不受保护的**

**2. Externalizable**

使用该接口之后，之前基于Serializable接口的序列化机制就将失效，序列化的细节需要由程序员去完成。

若使用Externalizable进行序列化，当读取对象时，会调用被序列化类的**无参构造器去创建一个新的对象**，然后再将被保存对象的字段的值分别填充到新对象中。由于这个原因，实现Externalizable接口的类必须要提供一个无参的构造器，且它的访问权限为public

自定制序列化的方法：writeExternal()与readExternal()方法

[Java Serializable（序列化）的理解和总结](https://www.cnblogs.com/yangjian-java/p/7813623.html)

---

### RPC为什么不用json

json体积太大，并且缺少类型信息，实际上只用在RESTful接口上，并没有看到RPC框架会默认选json做序列化的

