## Zookeeper

[TOC]



#### 服务注册中心怎么做的（服务注册原理）

通过zookeeper实现的服务注册和发现

**服务注册**：服务注册的时候，将完整的服务名称rpcServiceName(className+group+version)作为根节点，子节点是对应的服务地址(ip+端口号)。如果相同服务被部署多份，一个根节点会对应多个子节点。

**服务发现**：当我们要获取某个服务的对应地址，直接根据完整服务名获得其下所有子节点，然后根据负载均衡策略取出一个就行了。同时对服务节点设置watcher。

**服务通知**：Zookeeper对应的服务节点产生变化时，会触发对应的Watcher，Zookeeper注册中心会异步向服务所关联的所有服务消费者发出节点删除的通知，服务消费者根据收到的通知更新缓存的服务列表。

> group：用于处理一个接口有多个实现类
>
> version：为后续服务版本不兼容升级提供可能

---

#### CAP理论

在理论计算机科学中，CAP 定理（CAP theorem）指出对于一个分布式系统来说，当设计读写操作时，只能能同时满足以下三点中的两个：

- **一致性（Consistence）** : 所有节点访问同一份最新的数据副本
- **可用性（Availability）**: 非故障的节点在合理的时间内返回合理的响应（不是错误或者超时的响应）。
- **分区容错性（Partition tolerance）** : 分布式系统出现网络分区的时候，仍然能够对外提供服务。

---

#### 为什么用Zookeeper做注册中心(优点，与其他选型对比下) (待整理)

**常见注册中心**

常见的可以作为注册中心的组件有：ZooKeeper、Eureka、Nacos...。

1. **ZooKeeper 保证的是 CP。** 任何时刻对 ZooKeeper 的读请求都能得到一致性的结果，但是， ZooKeeper 不保证每次请求的可用性比如在 Leader 选举过程中或者半数以上的机器不可用的时候服务就是不可用的。
2. **Eureka 保证的则是 AP。** Eureka 在设计的时候就是优先保证 A （可用性）。在 Eureka 中不存在什么 Leader 节点，每个节点都是一样的、平等的。因此 Eureka 不会像 ZooKeeper 那样出现选举过程中或者半数以上的机器不可用的时候服务就是不可用的情况。 Eureka 保证即使大部分节点挂掉也不会影响正常提供服务，只要有一个节点是可用的就行了。只不过这个节点上的数据可能并不是最新的。
3. **Nacos 不仅支持 CP 也支持 AP。**

**为什么选择Zookeeper**

1. Zookeeper 是 开源Dubbo 推荐的注册中心实现， 且原生 Java 并且可以用Curator操作
2. Zookeeper 是满足 CP 原则的，相比于 Eureka 的AP 和Nacos的AP/CP 对于服务发现来说不是很占优势

[Zookeeper](http://zookeeper.apache.org/) 是 Apache Hadoop 的子项目，是一个**树型的目录服务**，支持变更推送，适合作为 Dubbo 服务的注册中心，工业强度较高，可用于生产环境

Zookeeper 紧遵CP原则，任何时候对 Zookeeper 的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性，但是 Zookeeper 不能保证每次服务请求都是可达的。

**不能保证可用性**：主要是会出现Leader宕机选举或者半数的节点不可用，虽然在分布式环境中，数据一致性应该是首先被保证的，但是但是对于服务发现来说，情况就不太一样了，针对同一个服务，即使注册中心的不同节点保存的服务提供者信息不尽相同，也并不会造成灾难性的后果（消费者虽然拿到可能不正确的服务实例信息后尝试消费一下，也要胜过因为无法获取实例信息而不去消费）

<img src="https://typora-image-elias.oss-cn-hangzhou.aliyuncs.com/uPic/image-20210308212428588.png" alt="image-20210308212428588" style="zoom:50%;" />

[各个注册中心对比](https://juejin.cn/post/6844904205870694413)

---

#### zookeeper节点宕掉的完整流程(待整理)



---

#### Zookeeper角色

三种角色，分为Leader、Follower和Observer

- ZooKeeper 集群中的所有机器通过一个 **Leader 选举过程** 来选定一台称为 “**Leader**” 的机器，Leader 可以提供读写服务。

- 除了 Leader 外，**Follower** 和 **Observer** 都只能提供读服务。
- 此外，**Observer** 机器不参与 Leader 选举和“过半写成功”策略，因此 Observer 机器可以在不影响写性能的情况下提升集群的读性能。

![img](https://snailclimb.gitee.io/javaguide/docs/system-design/distributed-system/zookeeper/images/zookeeper集群中的角色.png)

| 角色     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| Leader   | 提供读和写，负责投票的发起和决议，更新系统状态。             |
| Follower | 提供读服务，如果是写服务则转发给 Leader。在选举过程中参与投票。 |
| Observer | 提供读服务器，如果是写服务则转发给 Leader。不参与选举投票和与“过半写成功”策略。在不影响写性能的情况下提升集群的读性能。 |

##### Leader选举过程

1. **Leader election（选举阶段）**：节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader。
2. **Discovery（发现阶段）** ：在这个阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。
3. **Synchronization（同步阶段）** :同步阶段主要是利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后 准 leader 才会成为真正的 leader。
4. **Broadcast（广播阶段）** :到了这个阶段，ZooKeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。

---

#### Leader选举算法(Fast Leader Election)

**Zookeeper节点状态：**

- LOOKING：不确定Leader的“寻找”状态，即当前节点认为集群中没有Leader，进而发起选举；
- LEADING：“领导”状态，即当前节点就是Leader，并维护与Follower和Observer的通信；
- FOLLOWING：“跟随”状态，即当前节点是Follower，且正在保持与Leader的通信；
- OBSERVING：“观察”状态，即当前节点是Observer，且正在保持与Leader的通信，但是不参与Leader选举。

**选举相关的信息：**

- electionEpoch：“选民”的**选举轮次**，在每个节点中以逻辑时钟logicalclock的形式存储。每发起一轮新的选举，该值会加1。若节点重启，此值会归零。
- sid：“选民”自己的**服务器ID**，是一个正整数，由各个ZK实例中的$dataDir/myid指定。
- state：“选民”的**状态**。
- votedLeaderSid：这一票推选的**“候选人”的服务器ID**。在代码中直接命名为leader，为了防止混淆，这里稍作更改。
- votedLeaderZxid：这一票推选的**“候选人”的事务ID**。所谓事务ID即写操作的proposal ID，其高32位是Leader纪元值，低32位是当前Leader纪元下的操作序号，亦即zxid肯定是单调递增的。在代码中直接命名为zxid，为了防止混淆，这里稍作更改。
- recvset：“选民”的**票箱**，其中存储有自己的和其他节点的选票。注意，**每张选票都包含上述的electionEpoch、sid、state、leader和zxid信息**，并且票箱中都**只会记录每个“选民”的最近一次投票信息**。

**选举流程：**

首先，**自增本地的逻辑时钟**logicalclock。

接下来**给自己投一票**（即该票中包含自己的sid、zxid等信息），并将该选票**广播**给其他节点。

投出这一票后，只要当前服务器处于LOOKING状态，就会循环执行**收取其他选票、更新并广播自己的选票、计算投票结果**的操作，直到可以确定Leader为止。具体叙述如下。

- 如果对方的状态也是LOOKING，说明**两方都处于选举流程中**（可能是集群刚刚启动，或者Leader掉线）——
  - 若对方选票中的electionEpoch等于当前的logicalclock，说明当前节点与对方**处于同一选举轮次**，需要将对方的选票与自己刚才投出的票进行对比，看哪个候选Leader更优。规则是**先比较事务ID**（votedLeaderZxid），**再比较服务器ID**（votedLeaderSid），**值较大的候选Leader更优**。然后投更优的候选Leader一票，并广播出去。
  - 若对方选票中的electionEpoch大于当前的logicalclock，说明**当前节点的选举轮次已经滞后**。此时将logicalclock更新为该选票的electionEpoch，并清空recvset（因为上一轮的选票已经没用了）。然后按照上一条的规则对比选票，并投出自己的新选票。
  - 若对方选票中的electionEpoch小于当前的logicalclock，说明是**对方滞后**，忽略这一票。
  - 将自己的选票和其他人的选票放入recvset中，并进行计票。如果收到了**所有节点**的选票，则直接认为选举结束，根据票数修改自己的状态为LEADING或FOLLOWING；如果没有收到所有选票，但已经**过半（即满足Quorum原则）**，那么就等待一个较短的时期（默认200ms），如果选举的结果没有改变，则仍然认为选举已经结束，修改状态。这里我们只考虑数量，不考虑节点权重的问题。
- 如果对方的状态是LEADING/FOLLOWING，说明**对方不处于选举流程中**（可能是当前节点因故重启）——
  - 若对方选票中的electionEpoch等于当前的logicalclock，说明选举结果已经出来了，将它们放入recvset。特别地，如果收到了对方称自己处于LEADING状态的票，则进行计票并检查该节点是否得到了过半数的支持。如是，说明Leader有效，修改自身状态为FOLLOWING并退出选举。
  - 若对方选票中的electionEpoch不等于当前的logicalclock，说明在另一场选举中已经有了结果，只需听从该结果即可。这时需要将logicalclock直接设为对方的electionEpoch值，并将其他节点的选票放入一个旁路的outofelection集合并进行计票（顾名思义，这个集合只用来表征选举结果，不用于实际的选举流程），根据得出的结果修改自身状态，再结束选举。

可见，Fast Leader Election流程的本质就是每个节点通过不断做出最优的选择并进行广播，最终使所有节点对Leader和Follower角色的认知收敛到一致。

https://www.pianshen.com/article/60192001302/

---

### 分布式数据一致性协议

#### 2PC两阶段提交协议

**一、准备阶段（prepare）**

1. 协调者节点向所有参与者节点询问是否可以执行提交操作（prepare请求），并开始等待各参与者节点的响应。
2. 参与者节点执行询问发起为止的所有事务操作，并将Undo信息和Redo信息写入日志
3. 各参与者节点响应协调者节点发起的询问。如果参与者节点的事务操作实际执行成功，则它返回一个"同意"消息；如果参与者节点的事务操作实际执行失败，则它返回一个"中止"消息

**二、提交阶段（commit）**

**成功**

当协调者节点从所有参与者节点获得的响应消息都为"同意"时：

1. 协调者节点向所有参与者节点发出"**正式提交**"的请求。
2. 参与者节点正式完成操作，并释放在整个事务期间内占用的资源。
3. 参与者节点向协调者节点发送"完成"消息。
4. 协调者节点收到所有参与者节点反馈的"完成"消息后，完成事务。

**失败**

如果任一参与者节点在第一阶段返回的响应消息为"终止"，或者协调者节点在第一阶段的询问超时之前无法获取所有参与者节点的响应消息时：

1. 协调者节点向所有参与者节点发出"**回滚操作**"的请求。
2. 参与者节点利用之前写入的Undo信息执行回滚，并释放在整个事务期间内占用的资源。
3. 参与者节点向协调者节点发送"回滚完成"消息。
4. 协调者节点收到所有参与者节点反馈的"回滚完成"消息后，取消事务。

**2PC缺点：**

1. 执行过程中，**所有参与节点都是事务阻塞型的**。当参与者占有公共资源时，其他第三方节点访问公共资源不得不处于阻塞状态
2. **协调者发生故障**：参与者会一直阻塞下去。因为参与者没有超时机制
3. **二阶段无法解决的问题**：协调者在发出 “正式提交” 消息之后宕机，而唯一接收到这条消息的参与者同时也宕机了。那么即使协调者通过选举协议产生了新的协调者，这条事务的状态也是不确定的，没人知道事务是否被已经提交

https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4

---

#### 3PC三阶段提交协议

**3PC和2PC的区别：**

- 对于协调者[Coordinator]和参与者[Cohort]都设置了超时机制（在2PC中，只有协调者拥有超时机制，即如果在一定时间内没有收到cohort的消息则默认失败）。
- 在2PC的准备阶段和提交阶段之间，插入预提交阶段，使3PC拥有CanCommit、PreCommit、DoCommit三个阶段。说白了，PreCommit是一个缓冲，保证了在最后提交阶段之前各参与节点的状态是一致的。

**一、CanCommit阶段**

1. **事务询问**：Coordinator向Cohort发送CanCommit请求。询问是否可以执行事务提交操作。然后开始等待参与者的响应。
2. **响应反馈**：Cohort接到CanCommit请求之后，正常情况下，如果其自身可以顺利执行事务，则返回Yes响应，并进入预备状态。否则反馈No

**二、PreCommit阶段**

Coordinator根据Cohort的反应情况来决定是否可以继续事务的PreCommit操作。有以下两种可能：

**事务的预执行**：如果Coordinator从所有的Cohort获得的反馈都是Yes响应

1. 发送预提交请求。Coordinator向Cohort发送PreCommit请求，并进入Prepared阶段。
2. 事务预提交。Cohort接收到PreCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中。
3. 响应反馈。如果Cohort成功的执行了事务操作，则返回ACK响应，同时开始等待最终指令。

**中断事务**：如果有任何一个Cohort向Coordinator发送了No响应，或者等待超时之后，Coordinator都没有接到Cohort的响应

1. 发送中断请求。Coordinator向所有Cohort发送abort请求。
2. 中断事务。Cohort收到来自Coordinator的abort请求之后，执行事务的中断。

**三、DoCommit阶段**

该阶段进行真正的事务提交，也可以分为以下两种情况。

**执行提交**

1. 发送提交请求。Coordinator接收到Cohort发送的ACK响应，那么他将从预提交状态进入到提交状态。并向所有Cohort发送doCommit请求。
2. 事务提交。Cohort接收到doCommit请求之后，执行正式的事务提交。并在完成事务提交之后释放所有事务资源。
3. 响应反馈。事务提交完之后，向Coordinator发送ACK响应。
4. 完成事务。Coordinator接收到所有Cohort的ACK响应之后，完成事务。

**中断事务**

Coordinator没有接收到Cohort发送的ACK响应（可能是接受者发送的不是ACK响应，也可能响应超时），那么就会执行中断事务。

1. 发送中断请求。Coordinator向所有Cohort发送abort请求
2. 事务回滚。Cohort接收到abort请求之后，利用其在阶段二记录的undo信息来执行事务的回滚操作，并在完成回滚之后释放所有的事务资源。
3. 反馈结果。Cohort完成事务回滚之后，向Coordinator发送ACK消息
4. 中断事务。Coordinator接收到参与者反馈的ACK消息之后，执行事务的中断。

**如何解决2PC的问题**

**在doCommit阶段，如果Cohort无法及时接收到来自Coordinator的doCommit或者abort请求时，会在等待超时之后，会继续进行事务的提交**。（其实这个应该是基于概率来决定的，当进入第三阶段时，说明参与者在第二阶段已经收到了PreCommit请求，那么Coordinator产生PreCommit请求的前提条件是他在第二阶段开始之前，收到所有参与者的CanCommit响应都是Yes。一旦参与者收到了PreCommit，意味他知道大家其实都同意修改了。所以，一句话概括就是，当进入第三阶段时，由于网络超时等原因，虽然参与者没有收到commit或者abort响应，但是他有理由相信：成功提交的几率很大。）

**缺点：**

如果进入PreCommit后，Coordinator发出的是abort请求，如果只有一个Cohort收到并进行了abort操作，而其他对于系统状态未知的Cohort会根据3PC选择继续Commit，那么系统的不一致性就存在了。所以无论是2PC还是3PC都存在问题，后面会继续了解那个传说中唯一的一致性算法Paxos

https://csruiliu.github.io/blog/20160530-intro-3pc/

---

#### Paxos

Paxos算法解决的问题是一个分布式系统如何就某个值达成一致。

Paxos中一共有三种角色，proposers，acceptors，和 learners。

- proposers 提出提案，提案信息包括提案编号和提议的 value；
- acceptor 收到提案后可以接受（accept）提案，若提案获得多数派（majority）的 acceptors 的接受，则称该提案被批准（chosen）；
- learners 只能“学习”被批准的提案

**算法内容：**

- 为了满足P2c的约束，proposer提出一个提案前，首先要和足以形成多数派的acceptors进行通信，获得他们进行的最近一次接受（accept）的提案（prepare过程），之后根据回收的信息决定这次提案的value，形成提案开始投票。

- 当获得多数acceptors接受（accept）后，提案获得批准（chosen），由acceptor将这个消息告知learner。这个简略的过程经过进一步细化后就形成了Paxos算法。

> **P2c**：如果一个编号为 n 的提案具有 value v，该提案被提出（issued），那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于 n 的任何提案，要么他们已经接受（accept）的所有编号小于 n 的提案中编号最大的那个提案具有 value v。

通过一个决议分为两个阶段：

**一、prepare阶段**

1. proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；
2. acceptor收到prepare消息后：
   - 如果提案的编号大于它已经回复的所有prepare消息(回复消息表示接受accept)，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案。
   - 如果一个acceptor发现存在一个更高编号的提案，则需要通知proposer，提醒其中断这次提案。

**二、批准阶段**

1. 当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和根据P2c决定的value（如果根据P2c没有已经接受的value，那么它可以自由决定value）。
2. 在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即批准这个请求。

https://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95#.E5.AE.9E.E4.BE.8B

---

#### ZAB

ZooKeeper 并没有完全采用 Paxos算法 ，而是使用 ZAB 协议作为其保证数据一致性的核心算法。另外，在ZooKeeper的官方文档中也指出，ZAB协议并不像 Paxos 算法那样，是一种通用的分布式一致性算法，它是一种特别为Zookeeper设计的崩溃可恢复的原子消息广播算法。

**ZAB 协议介绍**

ZAB（ZooKeeper Atomic Broadcast 原子广播） 协议是为分布式协调服务 ZooKeeper 专门设计的一种支持崩溃恢复的原子广播协议。 在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各个副本之间的数据一致性。

**协议两种基本的模式：崩溃恢复和消息广播**

- **崩溃恢复** ：当整个服务框架在启动过程中，或是当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB 协议就会进入恢复模式并选举产生新的Leader服务器。当选举产生了新的 Leader 服务器，同时集群中已经有过半的机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。其中，**所谓的状态同步是指数据同步，用来保证集群中存在过半的机器能够和Leader服务器的数据状态保持一致**。
- **消息广播** ：**当集群中已经有过半的Follower服务器完成了和Leader服务器的状态同步，那么整个服务框架就可以进入消息广播模式了。** 当一台同样遵守ZAB协议的服务器启动后加入到集群中时，如果此时集群中已经存在一个Leader服务器在负责进行消息广播，那么新加入的服务器就会自觉地进入数据恢复模式：找到Leader所在的服务器，并与其进行数据同步，然后一起参与到消息广播流程中去。

https://www.jianshu.com/p/fb527a64deee

https://www.cnblogs.com/makelu/p/11123103.html

---

#### Raft协议

