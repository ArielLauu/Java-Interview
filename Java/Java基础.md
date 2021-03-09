## Java

[TOC]

### :peach:Java基础

#### Java什么时候会死锁，如何避免

线程死锁：多个线程由于等待资源释放，被无限期阻塞，程序不能正常终止。

##### 1 死锁的四个条件

1. **互斥条件**：资源任意时刻只能由一个线程占用
2. **请求与保持**：线程因请求资源而阻塞时，不释放已获得的资源
3. **不剥夺条件**：线程已获得的资源在未使用完时，不能被其他线程剥夺
4. **循环等待条件**：线程形成头尾相接的循环等待资源关系

##### 2 如何避免死锁

1. **破坏**互斥条件：:negative_squared_cross_mark:
2. **破坏**请求与保持：一次性申请所有资源
3. **破坏**不剥夺条件：占用资源的线程申请其他资源时，如果申请不到，可以主动释放它占有的资源
4. **破坏**循环等待条件：靠按序申请资源。规定所有的线程申请资源必须以一定的顺序，释放顺序与之相反，来避免循环等待

---

#### 面向对象的三大特征

- 面向对象三大特性：**封装、继承、多态**
- **封装**：把一个对象的信息隐藏在对象内部，不允许外部对象直接访问对象的内部信息。但可以提供一些可以被外界访问的方法，访问内部属性
- **继承**：继承是使用已存在的类的定义作为基础，建立新类的技术。新类的定义可以增加新的数据或新的功能，也可以用父类的功能，但不能选择性地继承父类
- **多态**：即一个引用变量所**指向的具体类型**，和通过该引用变量发出的**方法调用**，在编程时不确定，在**程序运行期间**才能确定；增加了程序的**灵活性**

---

#### 基本数据类型及其包装类，区别

**1 基本数据类型及其包装类**

![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/20190226164706730.png)

**2 区别**

1. 包装类型是对象，有用方法和字段，对象的调用都是通过引用对象的地址。基本类型不是
2. 包装类型是引用的传递，基本类型是值得传递
3. 声明方式不同，包装类型要用New
4. 存储位置不同，基本数据类型保存在值栈中，包装类型把对象放在堆中
5. 初始值不同，包装类型初始值为null
6. 使用方式不同，基本数据类型直接赋值使用，包装类型通常在集合中使用

---

#### 接口和抽象类的区别

1. 所有方法在接口里不能有默认实现（JDK8开始，可以有默认实现）。而抽象类可以有非抽象的方法
2. 接口中除了 final和static变量，不能有其他变量。抽象类不一定
3. 类可以实现多个接口，但只能继承一个抽象类
4. 接口方法默认修饰符是public，抽象类方法修饰符可以是public/protected/default
5. 从设计层面来说，抽象类是对类的抽象，是一种模板设计；接口是对行为的抽象，是一种行为规范

---

#### private default protected public 权限

- **private：**作用于属性和方法时，就只有在同一个类中能访问它们
- **default：**作用于属性和方法时，除了在同一个类中能访问它们，同一个包中的其它类（包括该类的子类和任意其它类）中也能访问它们。当属性或者方法没有权限修饰符时，其实就是default修饰的
- **protected：**作用于属性和方法时，除了在同一个类中和同一个包中的类（包括子类和其它任意类）中能访问它们外，其它包中该类的子类中也能访问它们
- **public：**所有类都能访问

---

#### 重载和重写

**1 重载**：方法名字相同，而参数个数/类型/顺序不同，或返回类型不同。即同样的一个方法能够根据输入数据的不同，做出不同的处理

**2 重写**：就是当子类继承自父类的相同方法，输入数据一样，但要做出有别于父类的响应时，你就要覆盖父类方法

1. 返回值类型、方法名、参数列表必须相同，抛出的异常范围小于等于父类，访问修饰符范围大于等于父类。
2. 如果父类方法访问修饰符为 `private/final/static` 则子类就不能重写该方法，但是被 static 修饰的方法能够被再次声明。
3. 构造方法无法被重写

---

#### Java范型

范型提供了**编译时类型安全的检查机制**，允许程序员在编译时检测到非法的类型。范型的本质是**参数化类型**，也就是说操作的类型数据被指定为一个参数。常见的就是范型方法/范型类/类型通配符

https://www.runoob.com/java/java-generics.html

---

#### Java异常

- Java中的所有异常都有一个共同祖先，java.lang.**Throwable类**。Throwable 有两个重要的子类：**Exception（异常）** 和 **Error（错误）**

- **Error **：表示程序无法处理的错误。大多数都与代码编写者执行的操作无关，而表示代码运行时JVM出现的问题。
- **Expection**是程序可以处理的异常，分为**运行时异常（RuntimeException，不受检异常：还包括Error)**和**编译异常（IOException，受检异常）**
  - **RuntimeException **表示JVM在运行期间可能出现的异常，如数组下标越界`（ArrayIndexOutBoundException）`。这类异常一般由程序逻辑错误引起，可以选择捕获处理，也可以不处理。
  - **IOException **是编译器要求必须处理的异常，否则编译不通过。

<img src="https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/exception-architechture-java.png" alt="exception-architechture-java" style="zoom: 50%;" />

##### Exception的最佳实践

1. **首先捕获最具体的异常**或者使用try-with-resource语句
2. **指定具体的异常：**尽可能的使用最具体的异常来声明方法，这样才能使得代码更容易理解
3. **对异常进行文档说明**：当在方法上声明抛出异常时，也需要进行文档说明
4. **抛出异常的时候包含描述信息**：在抛出异常时，需要尽可能精确地描述问题和相关信息
5. **首先捕获最具体的异常**
6. **不要捕获Throwable**
7. **不要忽略异常**：如catch了异常不处理
8. **不要记录并抛出异常**：不要捕获异常、记录日志并再次抛出
9. **包装异常时不要抛弃原始的异常**

参考链接：https://yq.aliyun.com/articles/762814

---

#### Java如何实现跨平台

- JVM是实现跨平台的桥梁。
- JVM不认识 `.java` 文件，需要将其编译为JVM认识的 `.class` 字节码（二进制）文件，然后由不同平台中的JVM，将字节码文件翻译成特定平台下的机器码，然后运行。
- 所以虽然Java是平台无关的，但是JVM是平台有关的，通过JVM的不同实现方式，实现了跨平台的功能。

---

#### 如何通过反射访问私有方法和属性

- `clazz.getDeclaredFields()`获取所有属性，然后通过`fs[i].setAccessible(true)`让某个（私有）属性可以被访问
- `classType.getDeclaredField("属性名")`获得属性，`field.setAccessible(true)`，并通过 `field.set(对象, "属性值")`修改属性值
- `clazz.getDeclaredMethods()`获取某个类的所有方法，`ms[i].setAccessible(true)`将方法设置为可以访问
- `classType.getDeclaredMethod("方法名", 参数类型)`获取指定方法,`method.setAccessible(true)`，并通过`method.invoke(对象, 具体参数)`进行调用

https://blog.csdn.net/vipwalkingdog/article/details/7684349

---

#### Java8的新特性

**1. 接口可以有默认方法（default关键字修饰）、静态方法**

**2. lambda表达式**

```java
Collections.sort(names, (String a, String b) -> b.compareTo(a));
names.sort((a, b) -> b.compareTo(a));
```

**lambda表达式作用域**

- 我们可以直接在 lambda 表达式中访问外部的局部变量（变量后面不可被修改）
- 对lambda表达式中的实例字段和静态变量都有读写访问权限
- 无法从 lambda 表达式中访问默认方法

**3. 函数式接口**

“函数式接口”是指**仅包含一个抽象方法**，但是可以有多个非抽象方法(也就是上面提到的默认方法)的接口，可以使现有的函数友好地支持Lambda

**4. 方法和构造函数引用**

通过`::`关键字传递方法或构造函数的引用，进而调用构造函数和方法

```java
Integer::valueOf //方法
PersonFactory<Person> personFactory = Person::new; //构造函数
```

**5. 提供更多内置函数式接口**

Predicate、Function、Supplier、Consumer

**6. Optional**

用于防止 NullPointerException

**7. Streams**

- `java.util.Stream` 表示能应用在一组元素上一次执行的操作序列。

- Stream 操作分为**中间操作**或者**最终操作**两种，最终操作返回一特定类型的计算结果，而中间操作返回Stream本身，这样你就可以将多个操作依次串起来。

- Stream 的创建需要指定一个数据源，比如` java.util.Collection` 的子类，List 或者 Set， Map 不支持。

- Stream 的操作可以**串行执行或者并行执行**。

**8. Map** 

支持各种新的和有用的方法来执行常见任务

**9. Date API(日期相关API)**

Java 8在 `java.time` 包下包含一个全新的日期和时间API。如：Clock 类提供了访问当前日期和时间的方法

**10. Annotations(注解)**

在Java 8中支持多重注解了，允许我们把同一个类型的注解使用多次，只需要给该注解标注一下`@Repeatable`即可。

https://blog.csdn.net/yczz/article/details/50896975

---

#### Java中的IO（BIO、NIO和AIO）

**1. BIO (Blocking I/O)**：同步阻塞I/O模式，数据的读取写入必须阻塞在一个线程内等待其完成。是典型的 **一请求一应答通信模型**。

在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。但是高并发场景下，传统的 BIO 模型是无能为力的。

![传统BIO通信模型图](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f322e706e67.png)

**2. NIO（New I/O）**：同步非阻塞的I/O模型，在Java 1.4 中引入，提供了 Channel , Selector，Buffer等抽象。

通常来说NIO中的所有IO都是从 Channel（通道） 开始的。

- 从通道进行数据读取 ：创建一个缓冲区，然后请求通道读取数据。同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据

- 从通道进行数据写入 ：创建一个缓冲区，填充数据，并要求通道写入数据。但不需要等待它完全写入，这个线程同时可以去做别的事情。

  **选择器**：用于使用单个线程处理多个通道。因此，它需要较少的线程来处理这些通道

**3. AIO（Asynchronous I/O）**：在 Java 7 中引入，它是异步非阻塞的IO模型。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

#### 同步和异步、阻塞和非阻塞

当一个read操作发生时，它会经历两个阶段：

1. 等待数据准备好
2. 将数据从内核拷贝到进程中

**同步**：导致请求进程阻塞，直到I/O操作完成

**异步**：不导致请求进程阻塞，异步只用处理I/O操作完成后的通知，并不主动读写数据，由系统内核完成数据的读写

**阻塞**： 发起recvfrom系统调用，等待数据报准备好的时候会阻塞

**非阻塞**： 发起recvfrom系统调用后，用户进程需要**不断的主动询问**kernel数据好了没有。如果没准备好，会返回一个EWOULDBLOCK error，并不会马上阻塞用户进程

---

#### String和StringBuilder、StringBuffer区别

1. 底层实现

   - String底层是final修饰的字符数组（JDK1.9字符数组变为byte数组），所以String对象不可变

   - StringBuilder和StringBuffer都继承自AbstractStringBuilder，底层是char[]，可变

2. 线程安全性

   - Sring不可变，线程安全
   - StringBuilder线程不安全
   - StringBuffer在AbstractStringBuilder基础上加了同步锁，线程安全

3. 性能

   - 每次对String对象的修改，都会创建一个新的String对象
   - StringBuilder和StringBuffer都是对本身进行修改，StringBuilder不需加锁，性能略高

总结：

1. 操作少量的不变数据：String
2. 单线程下操作大量数据：StringBuilder
3. 多线程下操作大量数据：StringBuffer



##### String不可变的原理是什么？为什么要设计成不可变的？ 

 用final修饰的数组；如果可变，字符串常量池引用会混乱；String缓存了自己的hash，如果可变，但是hash不会变，在HashMap、HashSet中会出现问题；String经常用作参数，如果可变则不安全。

1. **字符串常量池的需要：**可能存在多个String对象，引用了同一个常量池中的对象。如果修改其中一个，可能会影响另一个，造成逻辑混乱
2. **String对象缓存了HashCode**：String是不可变的，保证了hashcode的唯一性，于是在创建对象时其hashcode就可以放心的缓存了，不需要重新计算。这也就是Map喜欢将String作为Key的原因，处理速度要快过其它的键对象。
3. **安全性**：String被许多的Java类(库)用来当做参数，例如网络连接地址URL，文件路径path等, 假若String不是固定不变的,将会引起各种安全隐患

https://blog.csdn.net/renfufei/article/details/16808775



**String str1 = "abc" 创建了几个对象？String str2 = "ab"+"c"创建了几个对象？**

参考JVM部分

---

#### 怎么实现深拷贝

**1. 构造函数**

我们可以通过在调用构造函数进行深拷贝，形参如果是基本类型和字符串则直接赋值，如果是对象则重新new一个。

**优点**：底层实现简单；不需要引入第三方包；系统开销小

缺点：可用性较差，每次新增成员变量，都要增加新的拷贝构造函数

```java
Address address = new Address("杭州", "中国");
User user = new User("大山", address);

// 调用构造函数时进行深拷贝
User copyUser = new User(user.getName(), new Address(address.getCity(), address.getCountry()));
```

**2. 重写clone方法，实现Cloneable**

**优点**：底层实现简单；不需要引入第三方包；系统开销小

**缺点**：可用性较差，每次新增成员变量可能要修改clone方法；拷贝类需要实现cloneable接口

```java
 public class Address implements Cloneable {
 
    private String city;
    private String country;
 
    // constructors, getters and setters
 
    @Override
    public Address clone() throws CloneNotSupportedException {
        return (Address) super.clone();
    }
 
}

public class User implements Cloneable {
 
    private String name;
    private Address address;
 
    // constructors, getters and setters
 
    @Override
    public User clone() throws CloneNotSupportedException {
        User user = (User) super.clone();
        user.setAddress(this.address.clone());
        return user;
    }
 
}
```

**3. 序列化和反序列化**

我们可以先将源对象进行序列化，再反序列化生成拷贝对象，前提是拷贝的类（包括其成员变量）需要实现Serializable接口

**缺点**：序列化与反序列化存在一定的系统开销

**4. 反射**

[反射实现深拷贝实例](https://blog.csdn.net/qq_38598257/article/details/85123817)

[多种深拷贝实现方式整理](https://www.cnblogs.com/xinruyi/p/11537963.html)

---

### :peach:其他

#### 知道怎么撑爆方法区内存吗？或者说怎么动态生成很多方法？

CGlib 动态生成类、方法

https://blog.csdn.net/u013555976/article/details/87182366?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control