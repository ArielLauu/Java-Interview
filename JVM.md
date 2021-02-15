### JVM

---

[TOC]

#### JVM如何判断对象死亡

**1 引用计数器**

- 每个对象添加一个引用计数器，对象被引用+1，引用失效计数器-1。计数器为0的对象就是可以被回收的

- 无法解决**循环引用**

**2 可达性分析**

“`GC Roots`"作为起点向下搜索，走过的路径称为引用连。一个对象到`GC Roots`没有任何引用链相连的话，则该对象不可用

可作为`GC Roots`的对象：

- 虚拟机栈（栈桢中的本地变量表）中引用的对象
- 本地方法栈中引用的对象
- 方法区中静态类属性引用的对象
- 方法区中常量引用的对象

>  注：不可达的对象并非非死不可

>宣告一个对象死亡，经历**两次标记过程**
>
>不可达的对象被第一次标记并经历一次筛选，筛选的条件是此对象**是否有必要执行finalize方法**。如果对象没有覆盖finalize方法，或者finalize方法已经被虚拟机调用，则没有必要执行。
>
>被判定为需要执行的对象，会被放入队列中进行**二次标记**，除非该对象与引用链上的对象建立关系，否则就会被真的回收。

以下为**方法区的回收**，主要包含对常量池的回收，和对类的卸载。方法区主要存放永久代对象，而永久代对象的回收率比新生代低很多

**3 判断常量是否为废弃常量**

常量池中存在字符串 "abc"，若没有任何 String 对象引用该字符串常量，则常量 "abc" 就是废弃常量

**4 判断类是无用的类**

- 该类所有实例都被回收
- 加载该类的`ClassLoader`已经被回收
- 该类对应的`java.lang.Class`对象没有在任何地方被引用，无法通过反射访问该类的方法

---

#### JVM内存模型

JVM在执行Java程序的过程中，会把它管理的内存划分为多个数据区域。区域划分在 JDK1.8版本和之前的略有不同，后面会详细介绍

JDK1.8之前：

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/JVM%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA%E5%9F%9F.png" alt="img" style="zoom: 67%;" align="center"/>

JDK 1.8之后：

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/2019-3Java%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA%E5%9F%9FJDK1.8.png" alt="img" style="zoom:67%;"  align="center"/>

**线程私有：**

- **程序计数器**
- **虚拟机栈**
- **本地方发栈**

**线程共享：**

- **堆**
- **方法区**
- 直接内存（非运行时数据区的一部分）

**1 程序计数器**

- 可以看做当前线程，所执行的字节码的行号指示器。**字节码解释器通过改变程序计数器的值，来选取下一条执行的字节码指令**，循环、跳转、线程恢复等功能都依靠这个计数器完成。
- 为了**线程切换后能够恢复到正确的执行位置**，**每个线程都要有一个独立的程序计数器**，线程之间互不影响。
- 是唯一一个不会出现 **`OutOfMemoryError`**的区域

**2 Java虚拟机栈**

- 线程私有，生命周期同线程，**描述的是Java执行的内存模型**。**每次方法调用的数据都是通过栈传递**
- **Java内存可以粗糙的分为堆内存、栈内存，栈主要就是虚拟机栈。虚拟机栈由一个个栈桢组成**，而每个栈帧都包含：局部变量表、操作数栈、动态链接和方法出口信息
- 局部变量表主要存放了 编译器可知的各种数据类型，和对象引用
- 可能出现**`StackOverFlowError` 和 `OutOfMemoryError`**错误

**3 本地方法栈**

- 作用类似虚拟机栈，但是**Java虚拟机栈为虚拟机执行Java方法服务，而本地方法栈为虚拟机使用到的Native方法服务**
- 在Hotspot中和Java虚拟机栈合二为一

**4 堆**

- 是Java虚拟机管理的内存中最大的一块，**所有线程共享**，**用来存放对象实例，几乎所有的对象实例和数组都在这里分配内存**
- 从JDK1.7开始默认开启逃逸分析，如果某些方法中的对象引用没有被返回，或者没有被外面使用，那么对象可以直接在栈上分配内存。
- 是垃圾回收的主要区域，可以细分为Eden、From Survivor、To Survivor 空间等。

**5 方法区**

- 各线程共享，**用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**
- 方法区也被称为永久代。方法区是Java虚拟机规范所定义的，永久代是HotSpot虚拟机对方法区的实现。
- 方法区被替换为元空间后，使用直接内存，内存不再受JVM限制，减少了溢出的几率

**6 运行时常量池**

- 是方法区的一部分，用于存放编译器生成的各种字面量和符号引用

> JDK1.7 之前运行时常量池在方法区，HotSpot虚拟机对方法区的实现为永久代
>
> JDK1.7 字符串常量池被拿到了堆中
>
> HDK1.8 hotspot 用元空间（Metaspace)实现方法区。字符常量池仍然在堆中

**7 直接内存**

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 错误出现。

---

#### JVM类加载机制

类是在运行期间第一次使用时动态加载的，而不是一次性加载所有的类。因为如果一次性加载，那么会占用很多的内存。

**1 类的生命周期**

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33333566653139632d346137362d343561622d393332302d3838633930643661306437652e706e67.jpg" alt="img" style="zoom:200%;" />

类加载过程包括以下几个阶段：

- **加载**
- **验证**
- **准备**
- **解析**
- **初始化**
- 使用
- 卸载

**2 类加载过程**

1. **加载**

   加载过程主要完成以下三件事：

   - 通过全类名获得定义此类的二进制字节流
   - 将二进制字节流表示的静态存储结构，转换为方法区的运行时存储结构
   - 在内存中生成一个代表该类的Class对象，作为方法区该类各数据的访问入口

2. **验证**

   确保Class文件的字节流中包含的信息，符合当前虚拟机的要求，并且不会危害虚拟机自身的安全

3. **准备**

   **准备阶段正式为类变量分配内存，并设置初始值，这些内存都在方法区分配**。实例变量会在类实例化时，随着对象一块分配在堆中。这里设置的初始值，为各数据类型的默认零值，初始化阶段才会赋值。如果类变量被final修饰，则直接赋值。

4. **解析**

   **将常量池的符号引用替换为直接引用的过程，也就是得到类、字段或方法在内存中的指针或偏移量**。其中解析在某些情况下，可以在初始化阶段之后再开始，以支持Java的动态绑定

5. **初始化**

   初始化阶段才开始真正执行类中定义的Java程序代码。初始化阶段是执行类构造器 `<clinit> ()`方法的过程。 `<clinit> ()`方法是由编译器自动收集类中所有类变量的赋值操作，和静态语句块中的语句合并产生的。

**3 类初始化时机（哪些情况下不用初始化这一步）**

虚拟机严格规范了只有以下6种情况，必须对类进行初始化，**即只有主动去引用类才会初始化**：

1. 当遇到**new，getstatic，putstatic或者invokestatic**这四条字节码指令时。这四条指令最常见的场景是：用new关键字实例化对象；读取/设置一个类的静态字段；调用一个类的静态方法时
2. 使用**java.lang.reflect**对类进行反射调用时，要初始化未初始化的类
3. 初始化一个类时，如果其父类没有初始化，则先初始化其父类
4. 当虚拟机启动时，用户指定一个执行的**主类（main方法）**，虚拟机会先初始化这个主类
5. **MethodHandle和VarHandle**可以看作是轻量级的反射调用机制，而要想使用这2个调用， 就必须先使用**findStaticVarHandle**来初始化要调用的类。
6. 当一个接口中定义了**JDK1.8新加入的默认方法（被default关键字修饰的接口方法）时**，如果有这个接口的实现类发生了初始化，则先初始化接口

除了以上给出的主动引用，所有的引用方式都不会触发初始化，称为**被动引用**，常见例子为：

1. 过子类引用父类的静态字段，不会导致子类初始化。

   ```java
   System.out.println(SuperClass.value);  // value 字段在 SuperClass 中定义
   ```

2. 通过数组定义来引用类，不会触发此类的初始化。

   ```java
   SuperClass[] sca = new SuperClass[10];
   ```

3. 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

   ```java
   System.out.println(ConstClass.HELLOWORLD);
   ```

---

#### 类加载器

JVM内置了三个重要的ClassLoader，除了BootstrapClassLoader，其他类加载器均由Java实现，且全部继承自java.lang.ClassLoader：

1. **BootstrapClassLoader（启动类加载器）**：最顶层的加载类，由C++实现，负责加载 `%JAVA_HOME/lib` 目录下的jar包和类，或者被 `-XBootclasspath` 参数指定的路径中的所有类
2. **ExtensionClassLoader（扩展类加载器）**：主要负责加载目录 `%JRE_HOME/lib/ext` 目录下的jar包和类，或者被 `java.ext.dirs` 系统变量所指定的路径下的包
3. **AppClassLoader（应用程序类加载器）**：面向用户的加载器，负责加载当前应用`classpath`下的所有jar包和类

##### 双亲委派模型

双亲委派模型即类在加载的时候，系统会首先**判断当前类是否被加载过**。已经被加载的类会直接返回，否则才会尝试加载。加载的时候，首先**把该请求委派给该父类加载器的 `LoadClass()` 处理**，因此所有的请求最终都会汇总到顶层的**启动类加载器 `BootstrapClassLoader`** 中。当父类加载器无法处理时，才由自己来处理。当父类加载器为Null时，会使用启动类加载器 `BootstrapClassLoader` 作为父类加载器

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f636c6173736c6f616465725f5750532545352539422542452545372538392538372e706e67.jpg" alt="ClassLoader" style="zoom:67%;" />

**双亲委派机制的优点**

双亲委派模型保证了**Java程序的稳定运行**，可以**避免类的重复加载**（因为JVM区分不同类的方式不仅仅是类名，相同的类文件被不同的类加载器加载，产生的是不同的类），同时**保证了Java的核心API不被篡改**。如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 `java.lang.Object` 类的话，那么程序运行的时候，系统就会出现多个不同的 `Object` 类

**自定义类加载器**

自定义类加载器需要继承 `ClassLoader`。如果不想打破双亲委派模型，就重写 `ClassLoader` 类中的 `findClass()` 方法即可，否则重写 `loadClass()` 方法。（`loadClass()` 实现了双亲委派模型的逻辑，自定义类加载器一般不去重写它。）

双亲委派模型源码：

```java
private final ClassLoader parent; 
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //抛出异常说明父类加载器无法完成加载请求
                }
                
                if (c == null) {
                    long t1 = System.nanoTime();
                    //自己尝试加载
                    // 如果使用内置的类加载器不能加载类，那么就会调用 findClass 方法来加载类，findClass 方法要求重写，否则会直接抛出 ClassNotFoundException
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

==打破双亲委派机制原理，以及常用场景== （待整理）

常用场景：Tomcat https://www.jianshu.com/p/7706a42ba200

有哪些类加载器 ， 能否自定义 Java.Object.String 的类加载器 

.启动一个java程序的过程哪个类加载器会参与

是否自己写过类加载器

类加载讲类加载器（三种，如何定义用户加载器）

---

#### 对象创建的过程

![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/73f30855e4cdfd6f2e944398e97981a2.png)

1. **类加载检查**

   在常量池中检查，能否定位到这个对象对应的类的符号引用，以及这个符号引用对应的类是否被加载。如果未加载，先执行类加载过程。

2. **分配内存**

   为对象在堆分配内存，所需的内存大小在类加载完后即可确定。分配方式有 **”指针碰撞“** 和 **“空闲列表”** 两种方式。这两种算法的选用，取决于Java堆是否规整。

   ![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/3af6db384fba7d42e2f9a07fb57b72d8..png)

   **内存分配的并发问题：**

   - **TLAB**：为每一个线程在eden区分配一块内存。JVM中在给线程中的对象分配内存时，首先在TLAB分配，当TLAB内存不够时，采用CAS+失败重试进行内存分配。

   - **CAS+失败重试**：CAS是乐观锁的一种实现方式。不加锁，假设没有冲突。如果因为冲突失败就重试。保证了操作的原子性。

3. **初始化零值**

   将分配到的内存空间都初始化为0值（除了对象头），保证对象的字段不初始化就可以直接使用。程序可以访问到各字段对应的默认零值。

4. **设置对象头**

   对象的很多信息都放在对象头中，要进行设置。如对象是哪个类的实例，对象的GC信息，对象的hash码等。

5. **执行init方法**

   按照程序员编写的代码，进行真正的初始化。

---

#### 垃圾回收算法

1. **标记-清除**

   标记存活对象，对未标记对象进行回收。另外，还会判断回收后的分块与前一个空闲块是否连续，若连续则合并。

2. **标记-整理**

   标记存活对象， 让所有存活对象往一端移动，直接清理掉端边界以外的内存。

3. **复制算法**

   将内存分为两块，每次只用其中一块，当这块内存用完，就将存活的对象复制到另一块上，清空这一块。

   `HopSpot JVM`对于复制算法的实现：

   - 一个`Eden`区，两个`Survivor`区（from和to）
   - 对象一开始只在`Eden`区，经过一次GC存活后，并且能被`Survivor`区容纳，进入`To Survivor`区，年龄+1
   - 对`From Survivor`区进行GC时，HotSpot遍历所有`From Survivor`区的对象，按年龄从小到大对其所占用的大小进行累计，累积到某个年龄占用空间大小超过 `Survivor`一半时，取这个年龄和 `MaxTenuringThreshold`中的更小值，作为晋升到老年代的新年龄阈值
   - From和Eden被清空，From与To交换角色
   
4. **分代收集**

   新生代和老年代采用不同的算法，如新生代复制算法，老年代标记-整理算法。

---

#### 垃圾收集器

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f63363235626161302d646465362d343439652d393364662d6333613637663266343330662e6a7067.jpg" alt="img"  />

1. **Serial收集器**

   - 单线程收集器，在进行GC的时候会暂停其他所有工作线程
   - 新生代：复制算法，老年代：标记-整理

   ![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f32326664613461652d346464352d343839642d616231302d3965626664616432326165302e6a7067.jpg)

2. **ParNew收集器**

   - Serial收集器的多线程版本
   - 新生代：复制算法，老年代：标记-整理

   ![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38313533386364352d316263662d346533312d383665352d6531393864663165303133622e6a7067.jpg)

3. **Parallel Scavenge收集器**

   - 不同于其他收集器关注用户线程的停顿时间，`Parallel Scavenge`收集器**关注吞吐量**（高效利用CPU时间）。吞吐量即CPU中用于运行用户代码的时间与CPU总消耗时间的比值。
   - 新生代：复制算法，老年代：标记-整理

4. **Serial Old收集器**

   - Serial收集器的老年代版本
   - JDK1.5以前，在`Parallel Old`诞生之前配合`Parallel Scavenge`使用
   - 作为CMS收集器的后备方案，在并发收集发生`Concurrent Mode Failure`时使用

5. **Parallel Old收集器**

   - Parallel Scavenge收集器的老年代版本
   - 适合重吞吐量以及CPU资源敏感的场合使用

6. **CMS（Concurrent Mark Sweep）收集器**

   **CMS收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用**，是一种 **“标记-清除”算法**收集器

   - **初始标记**：暂停其他线程，标记GC Roots直接相连的对象
   - **并发标记**：同时开启GC和用户线程，记录所有GC Roots的可达对象。此时用户线程会不断更新引用域，算法会跟踪引用更新的地方。
   - **重新标记**：修正并发标记阶段更新的引用
   - **并发清除**：开启用户线程，同时GC线程清理未标记区域

   ![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/30adcbef76094b365d8fa82f257404df8c109d16.jpeg)

   - **CMS优点：**并发收集、低停顿

   - **CMS缺点**：
     - **无法处理浮动垃圾**。如果预留的内存不够存放浮动垃圾，就会出现`Concurrent Mode Failure`，虚拟机此时用`Serial Old`替代`CMS`
     - **吞吐量低**：低停顿时间牺牲吞吐量为代价
     - **标记-清除法**产生的**空间碎片**，导致需要提前`Full GC`

7. **G1收集器**

   - 适用于**多CPU和大内存**场景，面向**服务器**
   - 可以回收整个新生代+老年代

   - G1收集器把堆划分成多个大小相等的`Region`，新生代和老年代不再物理隔离。收集器针对各个小空间单独进行垃圾回收，使可预测的停顿时间成为可能。

   - 通过记录每个`Region`的回收时间，以及所获得的空间（根据过去经验获得），维护一个`Region`优先列表，每次根据允许的停顿时间，优先回收价值最大的`Region`。
   - 每个`Region`都有一个`Remembered Set`，记录该`Region`的对象所引用的对象所在的Region。这样在可达性分析时，可以避免全堆扫描。

   ![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39626264646565622d653933392d343166302d386538652d3262316130616137653061372e706e67.jpg)

   **可分为以下几个步骤：**

   - **初始标记**
   - **并发标记**
   - **最终标记**：修正并发标记阶段，因用户导致的标记产生变动。虚拟机将这段时间的变化记录在线程的`Remembered Set Logs`。最终标记将`Remembered Set Logs`的数据合并到 `Remembered Set`
   - **筛选回收**：对各个`Region`中的回收价值和成本进行排序，根据用户期望的停顿时间，制定回收计划。

   ![G1收集器](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/G1%E6%94%B6%E9%9B%86%E5%99%A8.jpg)

---

#### 如何分析堆转储快照

JVM 能够记录下问题发生时系统的部分运行状态，并将其存储在堆转储 (Heap Dump) 文件中。

**1 生成堆转储**

1. `jmap`：打印堆转储到指定的文件位置，该工具打包在JDK中，在bin文件夹中可以找到。调用方法为 `jmap -dump:live,format=b,file=<file-path> <pid>`。其中`live`指定只转储堆中的活动对象
2. `HeapDumpOnOutOfMemoryError`：设置`“ -XX：+ HeapDumpOnOutOfMemoryError”`系统属性后，JVM将在JVM遇到OutOfMemoryError时捕获堆转储。`-XX:HeapDumpPath=<file-path>`设置存储位置

**2 堆转储文件分析**

通过eclipse提供的堆转储工具 `MAT（Eclipse Memory Analyzer）`

1. 查看问题发生时刻，内存消耗的整体状态
2. 找到有可能导致内存泄漏的对象（通常为消耗内存最多的对象）
3. 分析该对象到`GC Roots`的引用链，查看该对象的详细分析，判断该对象为什么占用这么多内存，且没被gc回收
4. 在代码中定位问题代码，进行解决

---

#### 如何分析gc日志



---

#### 6



