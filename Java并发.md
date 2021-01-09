### :tulip:Java并发

---

[TOC]

#### Java线程池

**1 优点**

1. 降低资源消耗：重复利用线程降低线程创建和销毁的消耗
2. 提高响应速度：减少等待线程创建的时间
3. 提高线程可管理性

**2 实现Runnable和Callable的区别**

* Runnable：接口不会返回结果或抛出异常检查
* Callable：支持返回结果或抛出异常检查

**3 执行execute() 和submit() 方法的区别**

- execute() ：提交**不需要返回值**的任务，无法判断任务执行成功与否
- submit() ：提交**需要返回值**的任务。通过线程池返回的**Future类型对象**，判断是否执行成功。Future.get() 获取返回值，get()方法阻塞到当前线程完成，或 get（long timeout，TimeUnit unit）阻塞指定时间

**4 线程池的创建**

**4.1 Executor框架的工具类Excutors**

- `SingleThreadExecutor`：返回一个只有一个线程的线程池

- `FixedThreadPool`：返回一个固定线程数量的线程池
- `CachedThreadPool`：返回一个可根据实际情况调整线程数量的线程池
- `ScheduledThreadPool`：线程池能在给定的延迟后执行任务，或定期执行任务

**4.2 ThreadPoolExcutor**

```java
/**
  * 用给定的初始参数创建一个新的ThreadPoolExecutor。
*/
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
  ...
}
```

- **corePoolSize:** 最小可以同时运行的线程数量
- **maximumPoolSize:** 当任务队列满时，可运行的最大线程数 
- keepAliveTime：线程池中的线程数大于 `corePoolSize` 时，空闲的线程等待 `keepAliveTime`，然后被回收销毁；
- unit：`keepAliveTime` 参数的时间单位
- **workQueue:** 当前运行的线程数量到 `corePoolSize` 时，新任务被放在队列中
- threadFactory：用于创建新线程
- handler：饱和策略

**4.3 饱和策略**

当前运行线程数达到最大线程数，任务队列已满时，执行饱和策略

- **ThreadPoolExecutor.AbortPolicy（默认）**：抛出 `RejectedExecutionException`来拒绝新任务的处理
- **ThreadPoolExecutor.CallerRunsPolicy：**调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。`因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略`
- **ThreadPoolExecutor.DiscardPolicy：** 不处理新任务，直接丢弃掉
- **ThreadPoolExecutor.DiscardOldestPolicy：** 丢弃最早的未处理的任务请求

---

#### 线程池流程（原理）

**为了搞懂线程池的原理，我们需要首先分析一下 `execute`方法。**我们使用 `executor.execute(worker)`来提交一个任务到线程池中去，这个方法非常重要，下面我们来看看它的源码：

```java
   // 存放线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)
   private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    private static int workerCountOf(int c) {
        return c & CAPACITY;
    }

    private final BlockingQueue<Runnable> workQueue;

    public void execute(Runnable command) {
        // 如果任务为null，则抛出异常。
        if (command == null)
            throw new NullPointerException();
        // ctl 中保存的线程池当前的一些状态信息
        int c = ctl.get();

        //  下面会涉及到 3 步 操作
        // 1.首先判断当前线程池中之行的任务数量是否小于 corePoolSize
        // 如果小于的话，通过addWorker(command, true)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2.如果当前之行的任务数量大于等于 corePoolSize 的时候就会走到这里
        // 通过 isRunning 方法判断线程池状态，线程池处于 RUNNING 状态才会被并且队列可以加入任务，该任务才会被加入进去
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次获取线程池状态，如果线程池状态不是 RUNNING 状态就需要从任务队列中移除任务，并尝试判断线程是否全部执行完毕。同时执行拒绝策略。
            if (!isRunning(recheck) && remove(command))
                reject(command);
                // 如果当前线程池为空就新创建一个线程并执行。
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //3. 通过addWorker(command, false)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
        //如果addWorker(command, false)执行失败，则通过reject()执行相应的拒绝策略的内容。
        else if (!addWorker(command, false))
            reject(command);
    }
```

通过下图可以更好的对上面这 3 步做一个展示：

![线程池原理](Java并发.assets/线程池原理.png)

---

#### CAS原理

即Compare and swap，是乐观锁的实现方式之一。CAS是一种无锁算法，即不用锁实现多线程之间的变量同步，线程不会被阻塞。CAS算法涉及3个操作数：

- 需要读写的内存值V
- 进行比较的值A
- 需要写入的新值B

当且仅当V=A时，CAS通过原子方式用新值来更新V的值。如果失败了会进行不断的重试。

---

#### Synchronized原理

https://www.cnblogs.com/paddix/p/5367116.html

Synchronized通常用于修饰普通方法、静态方法和代码块，但是对于同步代码块和同步方法的实现原理，稍微有些区别。

**1 同步代码块原理**

每个对象有一个监视器锁（monitor），当monitor被占用时对象处于锁定状态。

线程执行**`moniterenter`**指令尝试获取monitor，过程为：

1. 若monitor进入数为0，则线程进入，monitor进入数+1
2. 若该线程已经占有该monitor，则重新进入，且monitor+1
3. 若其他线程占用了monitor，则该线程阻塞，直到monitor进入数为0，再尝试进入

线程要释放monitor时，执行**`monitorexit`**，执行后monitor进入数-1，若-1后进入数为0，则线程退出monitor。其他线程可以尝试获取monitor

> 在JVM中`monitorenter`和`monitorexit`字节码依赖于底层的操作系统的`Mutex Lock`来实现的，使用Mutex Lock需要将当前线程挂起，并从用户态切换到内核态来执行

**2 同步方法原理**

方法的同步并未通过上述两个指令来完成，而是通过**`ACC_SYNCHRONIZED`**标志符。当方法调用时，调用指令会检查方法的**`ACC_SYNCHRONIZED`**标志符是否被设置，如果被设置了，执行线程将先获取monitor，获取成功后才能执行方法体，方法执行完成后再释放monitor。方法的同步是一种隐式的方式来实现，无需通过字节码来完成

---

#### Synchronized的锁升级策略 

https://zhuanlan.zhihu.com/p/92808298

Synchronized的优化：https://www.cnblogs.com/wuqinglong/p/9945618.html

- 由于`synchronized`的实现，依赖于操作系统的**`Mutex Lock`**来实现，用户态和内核态的切换代价很昂贵，所以 JDK1.6 开始对`synchronized`做了优化。通过对象头的**`Mark World`**来区分不同场景下，同步锁的不同类型，来减少线程切换的次数。`Mark World`具体结构如下所示

  ![img](Java并发.assets/v2-e47232518a4e042f31a9e0eb6a48f88c_720w.jpg)

- 锁主要存在四种状态，依次是**无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态**，他们会随着竞争的激烈而逐渐升级（但是无法降级），以提高获得和释放锁的效率

1. **偏向锁**
- 由于大多数时候是不存在锁竞争的，所以引入偏向锁。偏向锁在无竞争的情况下会**把整个同步都消除掉**
  
- 偏向锁的作用是，当有线程访问同步代码或方法时，线程只需要判断对象头的**Mark Word**中，**是否有偏向锁，以及指向线程ID是否为自己**，而不需要进入Monitor去竞争对象。
  
- 出现其他线程竞争资源时，由于该锁已经是偏向锁，当发现对象头 Mark Word 中的线程 ID 不是自己的线程 ID，就会进行 **CAS 操作**获取锁，**如果获取成功**，直接**替换 Mark Word 中的线程 ID 为自己的 ID**，该锁会**保持偏向锁**状态；**如果获取锁失败**，偏向锁将**升级为轻量级锁**。
  
2. **轻量级锁**
- 轻量级锁考虑的是**竞争锁对象的线程不多**，而且**线程持有锁的时间也不长**的情景
   - 如果**没有竞争**，轻量级锁使用 **CAS 操作避免了获得锁的开销**。但如果存在锁竞争，**除了锁的开销外，还会额外发生CAS操作**，所以竞争激烈时，轻量级锁比传统的重量级锁更慢
   - 当有其他线程想访问加了轻量级锁的资源时，会使用**自旋锁优化**。
   
3. **自旋锁与适应性自旋锁**
- 基于大多数情况下，**线程持有锁的时间都不会太长，线程被挂起阻塞可能会得不偿失**。JVM 提供了一种自旋锁，**可以通过自旋方式不断尝试获取锁，从而避免线程被挂起阻塞。**
   - 自旋次数由 JVM 设置决定，默认值是10次。**不建议设置的重试次数过多**，因为 CAS 重试操作会占用 CPU。自旋锁重试之后如果抢锁依然失败，同步锁就会升级至重量级锁
   - **适应性自旋锁**：在 JDK1.6 中引入了自适应的自旋锁。其自旋的时间不再固定，而是**和前一次同一个锁上的自旋时间，以及锁的拥有者的状态来决定**
   
4. **重量级锁**

   - 自旋失败直接升级成重量级锁，进行线程阻塞，减少cpu消耗。
   - 当锁升级为重量级锁后，**未抢到锁的线程都会被阻塞**，进入阻塞队列。

---

#### synchronized和Lock的区别

https://blog.csdn.net/Maxiao1204/article/details/85065510

1. `Lock`是一个接口，而`synchronized`是Java中的关键词
2. `synchronized`在发生异常时，会自动释放线程占有的锁，不会产生死锁，而`Lock`在发生异常时需要手动在finally块中释放锁
3. 通过`Lock`可以知道是否获得锁，`synchronized`不行
4. `Lock`可以让等待锁的线程响应中断，`synchronized`不行
5. `Lock`可以提高多个线程读的效率
6. `synchronized`本质是悲观锁，`Lock`是乐观锁（底层基于`volatile`和`CAS`实现）

---

#### 并发编程的重要特性

1. **原子性**：`synchronized`可以保证
2. **可见性**：当一个线程对共享变量进行了修改，其他线程都可以立即看到修改后的值。`volatile`关键词可以保证
3. **有序性**：由于Java在编译和运行期的优化，代码的执行顺序未必就是编写代码时候的顺序。`volatile`可以禁止指令进行重排序优化

---

#### volatile原理，和synchronized区别

**1 volatile作用**

- 当前的Java内存模型下，线程可以把变量保存在本地内存，可能造成数据不一致。将变量声明成**volatile**，则告诉JVM每次读取这个变量，都要到**主存中进行读取**。Java内存模型如下：

![数据不一致](Java并发.assets/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539352542302545362538442541452545342542382538442545342542382538302545382538372542342e706e67)

- Volatile还能**防止指令重排**

**2 volatile原理** https://zhuanlan.zhihu.com/p/29868853

1. **内存可见性的实现**
   - 对声明了volatile的变量进行写操作，JVM会向处理器发送一条**Lock前缀的指令**，将这个变量所在缓存的数据，**立即写回到内存**。
   - 多处理器下，为了保证多个处理器缓存一致，就会实现**缓存一致协议**，每个处理器通过嗅探在总线上传播的数据，以保证自己的数据和内存中保持一致

2. **禁止指令重排的实现** https://www.jianshu.com/p/ef8de88b1343

   - Java编译器通过**内存屏障**禁止指令重排
   - 内存屏障分为两种：**Load Barrier（读屏障）** 和 **Store Barrier（写屏障）**
   - **Load Barrier**：在指令前插入，可以让高速缓存中的数据失效，强制重新从主存中加载数据
   - **Store Barrier**：在指令后插入，让写入缓存中的数据更新到主存中，对其他线程可见
   - 四种常见的内存屏障**LoadLoad**，**StoreStore**，**LoadStore**，**StoreLoad**，实际上也是上述两两组合，完成一系列的屏障和数据同步功能。

   `volatile`**的内存屏障策略非常严格保守，**非常悲观且毫无安全感的心态：

> ​	在每个**volatile写操作**前插入**StoreStore**屏障，在写操作后插入**StoreLoad**屏障；
> ​	在每个**volatile读操作**前插入**LoadLoad**屏障，在读操作后插入**LoadStore**屏障；

**3 和synchronized的区别**

1. `volatile`关键字是`synchronized`的**轻量级实现**，所以性能更好，但是`synchronized`在jdk1.6之后进行了优化，性能有了显著提高。`volatile`只能修饰**变量**，`synchronized`还可以修饰**方法和代码块**
2. 多线程访问`volatile`关键字**不会阻塞**，`synchronized`会
3. `volatile`能保证数据**可见性**，**不能保证原子性**。`synchronized`两者都能保证
4. `volatile`主要解决**变量在多个线程间的可见性**，`synchronized`解决多**个线程访问资源的同步性**。

#### 公平锁和非公平锁（效率比较）

https://www.cnblogs.com/fxtx/p/11657021.html

- **公平锁**：按照线程在队列中的排队顺序，先到者先拿到锁
- **非公平锁**：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的。如果没抢到，就加入队列中进行排队。

**比较**：公平锁会有更多线程切换的开销，而非公平锁减少了线程挂起的几率。所以**非公平锁性能更高**，能更充分的利用CPU的时间片。但是**可能会导致饥饿现象**，甚至有线程永远获取不到锁。

---

#### AQS原理

AQS的全称为（AbstractQueuedSynchronizer），在java.util.concurrent.locks包下面，是一个用来构建锁和同步器的框架。

**1 AQS原理概览**

**AQS的核心思想是，如果被请求的资源空闲，则将当前请求资源的线程，设置为有效的工作线程，并将共享资源设置为锁定状态。如果共享资源被占用，则需要一套线程阻塞等待，以及被唤醒时锁分配的机制。这个机制AQS使用CLH队列锁实现的，即将暂时获取不到锁的线程加入队列中。**

> CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点（Node）来实现锁的分配。

AQS(AbstractQueuedSynchronizer)原理图：

![AQS原理图](Java并发.assets/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4151532545352538452539462545372539302538362545352539422542452e706e67)

AQS使用一个 **volatile** 修饰的 **int 成员变量**表示同步状态；使用**CAS** 对该同步状态进行原子操作，实现对其值的修改；通过内置的FIFO队列完成获取资源的排队工作。

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

同步状态信息通过protected类型的`getState，setState，compareAndSetState`进行操作

**2 AQS对资源的共享方式**

AQS定义两种资源共享方式：

- **Exclusive**（独占）：只有一个线程能执行，如`ReentrantLock`。可分为**公平锁**和**非公平锁**
- **Share**（共享）：多个线程可同时执行，如`CountDownLatch`、 `CyclicBarrier`

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时，**只需要实现共享资源 state 的获取与释放方式即可**，具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了

**3 AQS底层使用了模板方法模式**

AQS使用了模板方法模式，自定义同步器时需要**重写AQS提供的模板方法**。默认情况下，每个方法都抛出 `UnsupportedOperationException`

```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

**ReentrantLock与AQS同步框架：**

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程`lock()`时，会调用`tryAcquire()`独占该锁并将`state+1`。此后，其他线程再`tryAcquire()`时就会失败，直到A线程`unlock()`到`state=0`（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。

---

#### ReentrantLock及其底层原理

ReentrantLock的实现基础是AQS。AQS分为独占和共享，ReentrantLock实现了独占功能。

ReentrantLock的内部类Sync继承了AQS，分为公平锁FairSync和非公平锁NonfairSync。

参考链接：https://www.jianshu.com/p/fe027772e156

https://www.cnblogs.com/weiqihome/p/9665718.html（写的有点乱）



---

#### synchronized和ReentrantLock 区别

1. **两者都是可重入锁**，既可以再次获取自己的内部锁
2. **synchronized依赖于JVM 而 ReentrantLock依赖于API**。synchronized是依赖于JVM实现的，而ReentrantLock是JDK实现的
3. JDK1.6开始对synchronized进行了很多优化**，两者性能差距不大**
4. ReentrantLock比synchronized增加了一些高级功能
   - ReentrantLock 提供了**等待可中断机制**：通过`lock.lockInterruptibly()` 来实现，也就是说正在等待的线程可以选择放弃等待，改为处理其他事亲
   - ReentrantLock **可以指定是公平锁还是非公平锁**，默认非公平锁。而synchronized 只能是非公平锁
   - ReentrantLock **支持选择性通知**。一个Lock对象中可以创建多个`condition`实例，线程对象可以注册在指定的`condition`中，从而有选择的通知某几个`condition`

---

#### 乐观锁和悲观锁区别

**1 悲观锁**

总是假设最坏的情况，每次去拿数据时都认为别人会修改，所以在拿数据时都会上锁。即共享资源每次只给一个线程使用，其他请求该资源的线程阻塞，用完后再把资源转让给其他线程。Java中`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

**2 乐观锁**

总是假设最好的情况，每次去拿数据时，都认为别人不会修改，所以不会上锁。但是在更新的时候，会判断一下别的线程有没有更新过该数据。**可以使用版本号机制+CAS实现**。

**3 适用场景**

乐观锁适用于多读的应用类型。这样省去了锁的开销，提高吞吐量。但如果是多写的情况，一般会经常产生冲突，这就会导致上层应用会不断的进行retry，反倒降低了性能，所以**一般多写的场景下用悲观锁就比较合适。**

---

#### JAVA线程的状态和怎么转化的

1. **新建（NEW）**

   初始状态，线程创建后还未执行start（）

2. **可运行（RUNNABLE）**

   Java线程将操作系统中的**就绪**（等待资源调度）和**运行**两种状态，统称为可运行。

   **新建**状态调用**`Thread.start()`**后，进入该状态中的**就绪态**，**就绪态**获得**CPU时间片**后进入**运行态**，**运行态**通过调用**`yeild()`**进入**就绪态**。

3. **阻塞（BLOCKED）**

   阻塞等待锁。其他线程释放锁后，该线程才有机会进入`RUNNABLE`中的就绪态状态

4. **无限期等待（WAITING）**

   线程主动进入无限期等待状态，等待其他线程显式唤醒。

   | 进入方法                                   | 退出方法                             |
   | ------------------------------------------ | ------------------------------------ |
   | 没有设置 Timeout 参数的 Object.wait() 方法 | Object.notify() / Object.notifyAll() |
   | 没有设置 Timeout 参数的 Thread.join() 方法 | 被调用的线程执行完毕                 |
   | LockSupport.park() 方法                    | LockSupport.unpark(Thread)           |

5. **限期等待（TIMED_WAITING）**

   线程主动进入限期等待状态，等待其他线程显式唤醒，或者一段时间后被系统自动唤醒。

   | 进入方法                                 | 退出方法                                        |
   | ---------------------------------------- | ----------------------------------------------- |
   | Thread.sleep() 方法                      | 时间结束                                        |
   | 设置了 Timeout 参数的 Object.wait() 方法 | 时间结束 / Object.notify() / Object.notifyAll() |
   | 设置了 Timeout 参数的 Thread.join() 方法 | 时间结束 / 被调用的线程执行完毕                 |
   | LockSupport.parkNanos() 方法             | LockSupport.unpark(Thread)                      |
   | LockSupport.parkUntil() 方法             | LockSupport.unpark(Thread)                      |

6. **终止（TERMINATED)**

   线程结束任务后自行终止，或者产生了异常而终止

![img](Java并发.assets/4bed2e738bd4b31c558a9715a70ecd799c2ff8cd.png)

---

#### 单例模式实现

https://www.runoob.com/design-pattern/singleton-pattern.html

**1 懒汉式**

在第一次调用的时候进行实例化

1. **懒汉式**：不支持多线程

   ```java
   public class Singleton{
   		private static Singleton instance;
   		private Singleton(){}
   		
   		public static Singleton getInstance(){
   				if(instance==null){
   						instance=new Singleton();
   				}
   				return instance;
   		}
   }
   ```

2. **懒汉式**，线程安全

   优点：第一次调用才初始化，避免内存浪费。
   缺点：必须加锁 synchronized 才能保证单例，但加锁会影响效率。

   ```java
   public class Singleton{
   		private static Singleton instance;
   		private Singleton(){}
   		
   		public static synchronized Singleton getInstance(){
   				if(instance==null){
   						instance=new Singleton();
   				}
   				return instance;
   		}
   }
   ```

3. **双重校验**

   优点：采用双重锁机制，安全且在多线程情况下能保持高性能

   ```java
   public class Singleton{
     	private volatile static Singleton instance;
     	private Singleton(){}
     	public static Singleton getInstance(){
         	if(instance==null){
             	synchronized(Singleton.class){
                 	if(instance==null){
                     	instance=new Singleton();
                   }
               }
           }
         	return instance;
       }
   }
   ```

4. **静态内部类**

   一个类被加载时，其静态内部类singletonHolder不会被加载，直到显示调用getInstance()方法，才会显式装载 SingletonHolder 类，从而实例化 instance

   - 优点：效果类似双重校验，实现更简单。

   ```java
   public class Singleton{
     	private static class SingletonHolder{
         	private static final Singleton instance=new Singleton();
       }
     	private Singleton(){}
     	private static final Singleton getInstance(){
         	return SingletonHolder.instance;
       }
   }
   ```

**2 饿汉式**

在类初始化时，自己进行实例化（基于 classloader 机制避免了多线程的同步问题）

1. **饿汉式**

   优点：没有加锁，执行效率会提高。
   缺点：类加载时就初始化，浪费内存。

   ```java
   public class Singleton{
     	private static Singleton instance=new Singleton();
     	private Singleton(){}
     	public static Singleton getInstance(){
         	return instance;
       }
   }
   ```

2. **枚举**

   优点：它不仅能避免多线程同步问题，而且还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化

   ```java
   public enum Singleton{
     	INSTANCE;
     	public void dosomething(){
         	//do something
       }
   }
   ```

   