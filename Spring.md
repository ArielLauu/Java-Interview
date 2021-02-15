## Spring

[TOC]

#### 什么是Spring框架，作用

Spring 是一种**轻量级开发框架**，旨在**提高开发人员的开发效率以及系统的可维护性**。 Spring 框架是很多模块的集合，为开发Java应用程序提供了全面的基础架构支持，使用这些模块可以很方便地协助我们进行开发，如核心容器（主要提供 IoC 依赖注入）、AOP等。

---

#### IOC依赖注入

**1 IOC介绍**

- IoC（Inverse of Control:控制反转）是一种**设计思想**，就是 **将原本在程序中手动创建对象的控制权，交由Spring框架来管理。** **IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value），Map 中存放的是各种对象。**

- **IOC作用**：让编程更简便，有利于代码的松耦合，最终便于程序的单元测试，和后期升级维护

**2 Spring如何实现IOC**

- Bean的创建是典型的工厂模式
- **BeanFactory** 是IOC容器最顶层的接口，负责管理Bean。BeanFactory主要提供了getBean()方法，用于获取以及创建Bean。
- 而 **ApplicationContext** 是对BeanFactory的扩展，其内部有一个实例化的DefaultListableBeanFactory（所有的 Bean 注册后会放入其中的 beanDefinitionMap，本质为ConcurrentHashMap）。
- **容器初始化**：首先进入refresh()进行IOC容器的初始化。将Bean配置加载为BeanDefinition，注册到容器中，此时的依赖关系用占位符代替。
- **bean的初始化**：进入getBean()方法获取或创建Bean。如果bean的作用域是singleton，先去缓存查找，未找到才新建实例并保存；如果不是singleton，直接新建实例。如果不存在方法覆写，那就使用 java 反射进行实例化，否则使用 CGLIB。在调用getBean的时候，如果碰到了属性是bean依赖的，则先初始化依赖的 bean，进行依赖注入。

参考链接：https://javadoop.com/post/spring-ioc IOC源码分析

https://www.cnblogs.com/shoshana-kong/p/9042220.html 

https://zhuanlan.zhihu.com/p/55777290

---

#### BeanFactory、FactoryBean、ApplicationContext的区别

- **BeanFactory**：是一个Factory顶层接口，是用来管理bean的IOC容器或对象工厂，较为古老，不支持spring的一些插件。
- **FactoryBean**：是一个Bean接口，是一个可以生产或者装饰对象的工厂Bean，可以通过实现该接口，自定义实例化Bean的逻辑。
- **ApplicationConext**：是BeanFactory的子接口，扩展了其功能。一般推荐使用ApplicationContext。

---

#### spring怎么解决循环依赖问题

循环依赖发生在依赖注入阶段。可以分为以下三种：

- **构造器** 方式无法解决，只能抛出异常。因为加入singletonFactories三级缓存的前提，是执行了构造器先构建出对象。所以构造器的循环依赖无法解决。
- **Prototype模式** 无法解决，只能抛出异常。因为Spring容器不缓存“prototype”作用域的bean，所以无法提前暴露一个创建中的bean。
- **单例模式** 可以通过**三级缓存**解决。

**三级缓存主要指：**

```java
//一级缓存，存放完全初始化的单例bean
//bean name --> bean instance
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

//二级缓存，存放半成品bean，即已创建对象但是未注入属性和初始化的bean。用来解决循环依赖
//bean name --> bean instance
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

//三级缓存，存放bean工厂对象，用来生成半成品的bean，并放入到二级缓存。用来解决循环依赖
//如果bean存在AOP,返回AOP的代理对象
//bean name --> ObjectFactory
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```

**三级缓存的流程：**

- A和B产生了循环依赖

- A在调用`createBeanInstance()` 后，如果spring配置了支持提前暴露半成品的bean`（allowEarlyReference=true）`，就会将当前生成的半成品的bean放到`singletonFactories`中，从而使其他bean能够引用到该bean
- 判断A依赖了B，此时就尝试获取B，则进行B的初始化流程。
- B发现自己依赖了A，于是尝试获取A。尝试一级缓存`singletonObjects`找不到（A未完全初始化），尝试二级缓存`earlySingletonObjects`也没有。此时如果spring配置了支持提前暴露半成品的bean，再尝试三级缓存`singletonFactories`，并通过三级缓存中`ObjectFactory.getObject()` 方法，将提取出半成品bean到二级缓存（保证A只经过一次AOP代理）。
- B走完初始化流程，将自己放入一级缓存，A获取到B顺利完成初始化，放入一级缓存

**为什么一定要三级缓存：**

- **三级缓存singletonFactory** ：暴露ObjectFactory目的是为了完成AOP代理。对象工厂清楚如何创建对象的AOP代理，但是不会立马创建，而是到合适的时机进行AOP代理对象的创建。（合适时机：利用后置处理器AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法，中对初始化后的Bean完成AOP代理）
- **二级缓存earlySingletonObject** ：保证对象只有一次AOP代理，避免再次通过ObjectFactory获取bean这一复杂流程，提升加载效率。接下来的依赖者调用的时候， 会先判断二级缓存是否有目标对象，如果存在直接返回。因为从三级缓存获取对像时，如果每次都通过工厂ObjectFactory去拿，需要遍历所有的后置处理器、判断是否创建代理对象，而判断是否创建代理对象本身也是一个复杂耗时的过程。

参考链接：https://blog.csdn.net/qwe123147369/article/details/108159155

https://zhuanlan.zhihu.com/p/84267654

https://www.jianshu.com/p/6c359768b1dc

---

#### AOP的作用和实现原理

##### 1 AOP作用

AOP 面向切面编程，就是将那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来（如权限认证、日志、事务处理），便于**减少系统的重复代码**，降低模块间的耦合度，并有利于未来的可操作性和可维护性

##### 2 AOP实现原理

1. 注册`AnnotationAwareAspectJAutoProxyCreator`，它是一个 BeanPostProcessor 实现类，会在bean初始化后执行 `postProcessAfterInitialization()`，通过`wrapIfNecessary`方法判断后，对需要进行代理的bean进行增强封装

2. **创建bean代理类的流程**

   - 为bean获取和匹配合适的增强器（advisor，是advice和pointcut的结合）。

     - 获取增强器：遍历容器中注册的所有bean，找到xml配置和@AspectJ注解的advisor
     - 匹配适合的增强器：遍历所有增强器，采用增强器的 Poincut 对一个对象的所有方法进行匹配

   - **创建代理类**：接着通过 ProxyFactory 的 getProxy方法创建代理类。这时候需要判断是JDK代理 or CGLib代理。判断准则：

     - CGLib：采取了优化策略 / ProxyTargetClas=true / 用户没有实现接口
     - JDK动态代理：需要代理的类本身就是一个接口 / 需要被代理的类本身就是一个通过jdk动态代理生成的类/ 其他所有情况

     tips：ProxyTargetClas=true意味着基于类代理（cglib），反之意味着基于接口代理（JDK）

3. **增强方法的执行**

   JDK代理类实现了InvocationHandler接口，执行过程主要看它的`invoke`方法。cglib动态代理类似，执行逻辑是在`intercept`方法中。以JDK动态代理为例

   - 创建拦截器链，拦截器链中将Advisor封装成 MethodInterceptor，并通过`proceed()`方法实现增强的递归调用。直到拦截器执行结束，执行目标方法
   - 关于协调前置，后置各种通知的代码，都是由MethodInterceptor 的invoke()方法维护。如前置MethodInterceptor是先激活前置增强方法，再递归调用下一个拦截器。后置MethodInterceptor是先递归调用下一个拦截器，在通过finally语句激活后置增强方法

   ```java
   //前置MethodInterceptor  
   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
       //激活 前置增强方法
       this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
       //继续调用下一个 拦截器。
       return mi.proceed();
   }
   
   //后置MethodInterceptor 
   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            //继续调用一下拦截器。
            return mi.proceed();
        }
        finally {
            //在finally 里面激活 后置增强方法
            invokeAdviceMethod(getJoinPointMatch(), null, null);
        }
    }
   ```

   

参考链接：https://www.jianshu.com/p/8af3cdf3b499 （最完善版）

https://www.jianshu.com/p/ae207ff6db41 (上面的细节版)

https://mp.weixin.qq.com/s/CrppMvAxoTtIyPsMBYtF5w （另一完善版）

https://www.zhihu.com/question/23641679?sort=created （里面涉及cglib动态代理）

---

#### 循环依赖与AOP的关系（Spring AOP怎么实现，围绕bean生命周期去讲）

1. **三级缓存的添加**：在循环依赖中，通过三级缓存 `singletonFactory.getObject()` 获取的实例，经过了SmartInstantiationAwareBeanPostProcessor的`getEarlyBeanReference`方法处理。这步操作是`BeanPostProcessor`的一种,目的是用于`被提前引用`时进行拓展。(曝光的时候并不调用该后置处理器，只有曝光，且被提前引用的时候才调用，确保了`被提前引用`这个时机触发)
2. **提前曝光代理earlyProxyReferences**：`getEarlyBeanReference`方法里有一个 `wrapIfNecessary`方法，用对进行AOP代理。所以循环依赖中(先创建A)，B的提前引用，将引用到A的代理。Spring IOC最后返回的bean，也是经过代理后的bean

**Spring AOP代理时机**：

- 正常情况下，都是在bean初始化后，调用 `postProcessAfterInitialisation()` 进行Spring AOP代理。
- 如果需要提前曝光代理，则在从三级缓存 `getObject()` 的时候进行 AOP代理

![img](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/8190955-982ed2cabfa2ee76.jpeg)

参考链接：https://www.jianshu.com/p/3bc6c6713b08

---

####  cglib，JDK动态代理区别

| 类型          | 机制                                                         | 回调方式                  | 适用场景                         | 效率                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------- | -------------------------------- | ------------------------------------------------------------ |
| JDK动态代理   | 委托机制，代理类和目标类都实现了同样的接口，InvocationHandler持有目标类，代理类委托InvocationHandler去调用目标类的原始方法 | 反射                      | 目标类是接口类                   | 效率瓶颈在反射调用稍慢                                       |
| CGLIB动态代理 | 继承机制，代理类继承了目标类并重写了目标方法，通过回调函数MethodInterceptor调用父类方法执行原始逻辑 | 通过FastClass方法索引调用 | 非接口类，非final类，非final方法 | 第一次调用因为要生成多个Class对象较JDK方式慢，多次调用因为有方法索引较反射方式快，如果方法过多switch case过多其效率还需测试 |

https://blog.csdn.net/john_lw/article/details/79539070

---

#### SpringMVC工作流程和作用

**工作流程**

1. 用户发送请求至前端控制器 `DispatcherServlet`
2. `DispatcherServlet`根据请求调用`handlerMapping`处理器映射器，解析请求对应的`Handler`
3. 解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，返回一个`HandlerExcutionChain`，包含一个`Handler`处理器对象和多个`HandlerInterceptor`处理器拦截器，返回给`DispatcherServlet`。
4. `DispatcherServlet`通过`HandlerAdapter` ，根据 `Handler `来调用真正的处理器来处理请求，并处理相应的业务逻辑。
5. 处理器处理完业务后，会返回一个 `ModelAndView` 对象，`Model` 是返回的数据对象，`View` 是个逻辑上的 `View`。`HandlerAdapter` 将结果返回给`DispatcherServlet`
6. `DispatcherServlet`将 `ModelAndView` 传给`ViewResolver` ，`ViewResolver`会根据逻辑 `View` 查找实际的 `View`，并返回给`DispatcherServlet`。
7. `DispaterServlet` 对  `View`进行渲染，即将模型数据填充到视图中。

![SpringMVC运行原理](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31302d31312f34393739303238382e6a7067.jpg)

**作用**：将包含业务数据的模块与显示模块的视图解耦，代码松耦合；与spring其他框架无缝集成

---

#### spring bean生命周期（初始化过程）

- Bean容器找到配置文件中Spring Bean的定义。
- Bean容器利用Java Reflection API创建一个Bean的实例。
- 设置Bean的属性值
- 如果实现了*Aware接口，就调用相应的方法。如Bean实现了BeanNameAware接口，调用setBeanName()方法，传入Bean的名字。
- 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessBeforeInitialization()方法
- 如果Bean实现了InitializingBean接口，执行afterPropertiesSet()方法。
- 如果Bean在配置文件中的定义包含`init-method`属性，执行指定的方法。
- 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessAfterInitialization()方法
- 当要销毁Bean的时候，如果Bean实现了DisposableBean接口，执行destroy()方法。
- 当要销毁Bean的时候，如果Bean在配置文件中的定义包含`destroy-method`属性，执行指定的方法。

---

#### bean的作用域

- **singleton** : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- **prototype** : 每次请求都会创建一个新的 bean 实例。
- **request** : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
- **session** : 在一次Http Session中，容器会返回该Bean的同一实例，该实例仅在当前Session有效
- **global-session**： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

---

#### Spring事务

**1 Spring怎样开启事务**

声明式事务基于 AOP，一种是在xml中做相关的事务规则声明，另一种是通过 **@Transactional** 注解。当`@Transactional`注解作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，在方法级别使用该标注，可以覆盖类级别的定义。

在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事物只会在遇到`RuntimeException`的时候才会回滚

**注**：注解一般加在Service层。因为一个Service完成一个服务，但是service层的方法通常都是由很多dao层的方法组成的，出错需要整体回滚。如果注解加在dao层，回滚的时候只能回滚到当前方法。

https://developer.ibm.com/zh/articles/j-master-spring-transactional-use/

**2 Spring事务分类**

1. **编程式事务**：用户通过代码的方式手动控制事务，粒度较细，比较灵活，但是开发起来比较繁琐。每次都要开启、提交、回滚
2. **声明式事务**：分为XML方式和注解方式，由Spring提供对事务的控制管理

**3 事务传播机制**

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于`TransactionDefinition.PROPAGATION_REQUIRED`。

**4 事务失效的情况**

1. @Transactional 应用在非 public 修饰的方法上
2. @Transactional 注解属性 propagation 设置错误
   - `PROPAGATION_SUPPORTS`、`PROPAGATION_NOT_SUPPORTED`、`PROPAGATION_NEVER` 这三种事务传播方式，事务不会发生回滚
3. @Transactional  注解属性 rollbackFor 设置错误。Spring默认抛出了未检查`unchecked`异常（继承自 `RuntimeException` 的异常）或者 `Error`才回滚事务；其他异常不会触发回滚事务
4. 方法必须外部调用，事务才会起作用。如未声明注解事务的方法A，调用注解了事务的B，则B的事务失效。
5. 事务方法抛出的异常，被其他方法catch了
6. 数据库引擎不支持事务

https://mp.weixin.qq.com/s/IcDEEft7bLhnqyo5knwUdw

---

#### Spring相关的设计模式

![image-20200902183508555](https://typora-image-ariellauu.oss-cn-beijing.aliyuncs.com/uPic/image-20200902183508555.png)

https://github.com/Snailclimb/JavaGuide/blob/master/docs/system-design/framework/spring/Spring-Design-Patterns.md

---

#### Springboot、Spring、SpringMVC区别

- **Spring** 框架为开发Java应用程序提供了全面的基础架构支持，它包含一些很好的功能，如依赖注入和开箱即用的模块，这些模块可以大大缩短应用程序的开发时间。
- **Spring MVC** 是Spring的一个模块，是基于 Servlet 的一个 MVC 框架，提供了一种轻度耦合的方式来开发web应用，主要解决 WEB 开发的问题。
- **Spring Boot** 基本上是Spring框架的扩展，它实现了自动配置，降低了项目搭建的复杂度。同时，Spring Boot为通用 Spring项目提供了很多非功能性特性，例如嵌入式 Server

https://www.jianshu.com/p/42620a0a2c33

**SpringBoot 优点**

1. Spring Boot理念是约定优于配置，尽可能地进行自动配置，减少了用户需要动手写的各种冗余配置项，简化开发。
2. 提供了多个可选择的”starter”以简化Maven的依赖管理（也支持Gradle），让您可以按需加载需要的功能模块。
3. Spring Boot 应用程序提供嵌入式HTTP服务器，如Tomcat和Jetty，可以轻松地开发和测试web应用程序。
4. 提供了一整套的对应用状态的监控与管理的功能模块（通过引入spring-boot-starter-actuator），包括应用的线程信息、内存信息、应用是否处于健康状态等

---

#### SpringBoot自动配置原理



---

#### springboot的启动过程

https://www.jianshu.com/p/dc12081b3598



---



SpringBoot的启动流程；

springboot为什么能简化配置，如何实现的

Springboot没有配置包路径，它是怎么实现包路径扫描的

SpringBoot自动装配的原理

说说springboot各个层怎么交互的

springboot如果想要加载特定的bean，怎么实现

SpringBoot常用注解