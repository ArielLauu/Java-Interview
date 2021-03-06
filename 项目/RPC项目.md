## 项目

[TOC]

### TME实习

#### 实习项目

参与“TME差旅统购”项目开发，实现TME差旅机票、酒店预订及公司结算的业务流程。

#### Aurora框架

相当于一个配置解器，通过配置解析来实现各种功能，能够快速实现数据库相关的操作。

和java最大的不同，就是不会将数据对象映射到Java对象中，而是以纯粹的数据容器作为数据载体。

**优点**：更加灵活，传统的OR mapping所产生的对象，其结构式编译时期决定的。而容器承载的数据，可以在运行时期改变其结构、内容。整个系统实现了数据级别的耦合

框架整体架构上分为**客户端组件**，和服务端组件：**Business model**，**Service model**

1. 客户端组件负责前端页面的构建，分为：表单组件、布局组件、数据组件

2. **business model**：一个bm相当于数据库的一个表，里面包含了bm的增删查改等操作
3. **Service model**：svc中注入bm，进行逻辑的处理。

#### 差旅统购项目

完成原有差旅系统与“腾讯驿行”项目的对接，实现了机票/酒店预订，以及公司审核、结算的业务流程。

“腾讯驿行”那边提供接口支持，差旅项目中通过调用对方的接口实现酒店、机票订购等流程。订购以后，由腾讯这边执行订单的审核，审核后向腾讯驿行提交审核结果，返回最终的确定结果

---

### RPC项目

#### 介绍自己的RPC项目

项目基于Zookeeper实现注册中心功能，支持服务的订阅和发布功能，实现了两种负载均衡策略。在网络传输上使用Netty，序列化使用了Kryo和ProtoStuff两种方式。

除了基础RPC功能以外，还对调用过程进行了动态代理。生成一个代理类RpcClientProxy，通过其getProxy()方法获取服务的代理，进而直接调用远程的服务。

为了进一步简化RPC调用，还提供了通过注解来注册服务、消费服务的功能，这个功能是通过与spring容器整合实现的，通过BeanPostProcessor的postProcessBeforeInitialization()方法对注解了RpcService的类进行服务发布，postProcessAfterInitialization()方法对注解了RpcReference的类进行代理赋值。

最后，还通过SPI机制的核心ExtensionLoader实现了动态加载实现类的功能，有利于程序的扩展。

---

#### 实现高性能的RPC关键在于哪些方面

选择合适的**传输协议**和**序列化协议**

合适的负载均衡策略

---

#### 整体服务调用链路是怎样的

1. 服务端发布服务（通过注解or手动发布）
2. 客户端生成服务动态代理对象，代理对象的invoke方法中封装了远程调用的细节
3. 远程调用的具体过程，就是先从注册中心获取服务（使用负载均衡），然后与服务端建立连接，发送消息，获取调用结果
4. 服务端收到客户端请求后，反射调用对应方法，返回结果

https://blog.csdn.net/sinat_34814635/article/details/79215308?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control&dist_request_id=c1723898-6eb7-4a92-9daf-9adc82f8c6b4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control

---

#### JDK动态代理机制是怎么实现的

1、编写需要被代理的类和接口

2、编写代理类，需要实现 `InvocationHandler` 接口，重写 `invoke()` 方法；

3、使用`Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)`动态创建服务代理对象，通过代理类对象调用业务方法。

---

#### 如何实现跨语言的序列化或者rpc框架

一个rpc框架要想跨语言，本质是在解决**序列化/反序列化**的跨语言问题

**只要 RPC 框架保证在不同的编程语言中，使用相同的序列化协议，就可以实现跨语言的通信。**另外，为了在不同的语言中能描述相同的服务定义，跨语言的 RPC 框架还需要提供一套描述服务的语言，称为 IDL（Interface description language）。所有的服务都需要用 IDL 定义，再由 RPC 框架转换为特定编程语言的接口或者抽象类。这样，就可以实现跨语言调用了。

---

#### 项目最困难的地方



---

#### 项目改进点

1. **增加可配置比如序列化方式、注册中心的实现方式,避免硬编码** 
2. **服务监控中心（类似dubbo admin）**，增加dubbo类似的容错机制

---

#### 比较RPC和HTTP

本质上来说，这两个并不是并行概念。RPC 是一种**设计**，就是为了解决**不同服务之间的调用问题**，完整的 RPC 实现一般会包含有 **传输协议** 和 **序列化协议** 这两个。而HTTP本质上是一种**传输协议**。

**相同点**：两种方式都能实现远程服务调用

**不同点**：

- RPC要求服务提供方和服务调用方都需要使用相同的技术，比如说都是dubbo
- 当使用http进行服务间调用的时候，无需关注服务提供方使用的编程语言，也无需关注服务消费方使用的编程语言，服务提供方只需要提供restful风格的接口，服务消费方，按照restful的原则，请求服务即可。

**如何选择：**

- 速度：RPC要比http更快，虽然底层都是TCP，但是http协议的信息往往比较臃肿
- 难度：RPC实现较为复杂，http相对比较简单
- 灵活性：http更胜一筹，因为它不关心实现细节，跨平台、跨语言。

因此，两者都有不同的使用场景：

- RPC：对效率要求更高，并且开发过程使用统一的技术栈
- HTTP：需要更加灵活，跨语言、跨平台

