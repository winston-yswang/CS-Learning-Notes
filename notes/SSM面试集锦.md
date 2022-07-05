#### 1、简单介绍一下Spring框架、Spring 等框架的好处

Spring 是一个开源框架，一个基于模块化、轻量级的 Java 开发框架，它是为了解决企业应用开发的复杂性而创建的。

使用 Spring 框架的好处有：

- 方便解耦，简化开发：通过Spring 提供的 IoC 容器，我们可以将对象之间的依赖关系交给Spring进行控制，避免程序过度耦合，可以更专注于上层的应用。
- AOP编程的支持：通过Spring提供的AOP功能，方便进行面向切面的编程，许多不容易用传统OOP实现的功能可以通过AOP轻松应付，如测试监控等。
- 提供良好的事务管理、异常处理机制，并且方便调试。



#### 2、Spring 常用注解

（1）`@SpringBootApplication` 

Spring Boot 项目的核心注解，目的是开启自动配置。`@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@ComponentScan`： 扫描被`@Component` (`@Service`,`@Controller`)注解的 bean，注解默认会扫描该类所在的包下所有的类。
- `@Configuration`：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类

（2）Spring Bean相关

`@Autowired`：自动导入对象到类中，被注入进的类同样要被 Spring 容器管理。

`@Component` ：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。

`@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。

`@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。

`@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 Service 层返回数据给前端页面。

`@RestController`注解是`@Controller`和`@ResponseBody`的合集,表示这是个控制器 bean,并且是将函数的返回值直接填入 HTTP 响应体中,是 REST 风格的控制器。

 `@Scope`：声明 Spring Bean 的作用域，四种常见的 Spring Bean 的作用域：

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- session : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

`@Configuration`：一般用来声明配置类，可以使用 `@Component`注解替代，不过使用`@Configuration`注解声明配置类更加语义化。

（3）前后端传值

`@RequestMapping`： 是一个用来处理请求地址映射的注解，可用于类或方法上。

`@Responsebody` 注解表示该方法的返回的结果直接写入 HTTP 响应正文（ResponseBody）中，一般在异步获取数据时使用，通常是在使用 @RequestMapping 后，返回值通常解析为跳转路径，加上 @Responsebody 后返回结果不会被解析为跳转路径，而是直接写入HTTP 响应正文中。

`@RequestBody`：用于读取 Request 请求（可能是 POST,PUT,DELETE,GET 请求）的 body 部分并且**Content-Type 为 application/json** 格式的数据，接收到数据之后会自动将数据绑定到 Java 对象上去。系统会使用`HttpMessageConverter`或者自定义的`HttpMessageConverter`将请求的 body 中的 json 字符串转换为 java 对象。

`@PathVariable`用于获取路径参数，`@RequestParam`用于获取查询参数。

（4）读取配置信息

 `@Value("${property}")` 读取比较简单的配置信息；

`@ConfigurationProperties`读取配置信息并与 bean 绑定。



#### 3、Spring Bean 的生命周期

实例化一个Bean，默认bean是单例，按照Spring上下文对实例化的Bean进行配置，也就是IoC注入。

接着进行Spring Aware接口注入，将bean 与Spring 框架耦合，让bean能获取Spring容器的服务。然后如果Bean关联了`BeanPostProcessor` 接口和`InitializingBean`接口的话，相应的方法则会生效。

到这bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；

若bean实现了DisposableBean接口，spring将调用它的distroy()接口方法。同样的，如果bean使用了destroy-method属性声明了销毁方法，则该方法被调用；

参考文章：[Spring Aware接口](https://blog.csdn.net/iechenyb/article/details/83788338?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.pc_relevant_default&spm=1001.2101.3001.4242.1&utm_relevant_index=3) 、[Spring Bean生命周期介绍](https://blog.csdn.net/qq_40917230/article/details/80257892)



#### 4、介绍一下 IoC，IoC 是如何实现的？

IoC 控制反转就是把创建和管理 bean 的过程转移给了第三方，而这个第三方，就是 Spring IoC Container。所谓控制，是指控制 bean 的创建、管理的权利，控制 bean 的整个生命周期。而反转则是指把控制的权利交给了 Spring 容器，而不是自己去控制，就是反转。

Spring IOC容器的实现：

1. 根据Bean配置信息在容器内部创建Bean定义注册表
2. 根据注册表加载、实例化bean、建立Bean与Bean之间的依赖关系
3. 将这些准备就绪的Bean放到Map缓存池中，等待应用程序调用



#### 5、如何解决循环依赖，循环依赖中的Bean是怎么去传参的？

类与类之间的依赖关系形成了闭环，就会导致循环依赖问题的产生。

循环依赖问题在Spring中主要有三种情况：

- 通过构造方法进行依赖注入时产生的循环依赖问题。
- 通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
- 通过setter方法进行依赖注入且是在单例模式下产生的循环依赖问题。

在Spring中，只有第3种方式的循环依赖问题被解决了，其他两种方式在遇到循环依赖问题时都会产生异常。其实也很好解释：

- 第1种构造方法注入的情况下，在new对象的时候就会堵塞住了，其实也就是”先有鸡还是先有蛋“的历史难题。
- 第2种setter方法（多例）的情况下，每一次`getBean()`时，都会产生一个新的Bean，如此反复下去就会有无穷无尽的Bean产生了，最终就会导致OOM问题的出现。

Spring解决的单例模式下的setter方法依赖注入引起的循环依赖问题，主要是通过Spring 中的三个缓存，用于存储单例的Bean 实例，这三个缓存是彼此互斥的，不会针对同一个Bean 的实例同时存储。



参考文章：[循环依赖](https://juejin.cn/post/6895753832815394824)



#### 6、介绍一下 AOP，AOP 是如何实现的？

AOP（Aspect-oriented Programming）面向切面编程，为了解决横切逻辑代码存在的代码重复问题、以及横切逻辑代码和业务代码混杂在一起，代码臃肿，不便于维护等问题，而提出的一种编程思想。

它提出了横向抽取机制，将横切逻辑代码和业务逻辑代码分离。使得开发者在不改变源代码的前提下，为系统中不同业务组件添加某些通用功能。

实现AOP功能的框架主要有Spring AOP和AspectJ。

- Spring AOP使用的动态代理，运行时生成 AOP 代理类，缺点是由于 Spring AOP 需要在每次运行时生成 AOP 代理，因此性能略差一些。
- AspectJ是静态代理的增强，采用编译时生成 AOP 代理类，因此也称为编译时增强，具有更好的性能。 缺点是需要使用特定的编译器进行处理。



#### 7、Java拦截器

拦截器的主要是基于Java 的反射机制，属于面向切面编程(AOP)的一种运用。拦截器的实现是继承HandlerInterceptor 接口，并实现接口的preHandle、postHandle和afterCompletion方法。就是在Service或者一个方法前调用一个方法，或者在方法后调用一个方法，甚至在抛出异常的时候做业务逻辑的操作。主要用于权限控制，日志打印，参数校验等。



#### 8、Java过滤器

过滤器Filter基于Servlet实现，过滤器的主要应用场景是对字符编码、跨域等问题进行过滤。Servlet的工作原理是拦截配置好的客户端请求，然后对Request和Response进行处理。Filter过滤器随着web应用的启动而启动，只初始化一次。

Filter的使用比较简单，继承Filter 接口，实现对应的init、doFilter以及destroy方法即可。

#### 9、Spring Boot 启动流程

SpringApplication.run中执行了两步操作，先封装了一个SpringApplication的实例，再执行该实例的重载run方法

SpringApplication封装实例时，读取了classpath下所有的`MTEA-INF/spring.factories` xml配置文件的ApplicationContextInitializer（容器初始化器）还有ApplicaiontListener（侦听器），将这两者封装到SpringApplication实例中

执行SpringApplication实例的run方法

run方法中默认初始化了Annotation配置的容器AnnotationConfigApplicationContext

执行上面ApplicationContextInitializer的initial方法

然后加载Bean到容器中

参考文章：[Spring Boot启动流程](https://juejin.cn/post/6844904002988015629)



#### 10、Spring Boot 自动装配

Spring Boot自动配置的核心注解是@SpringBootApplication，该注解是spring boot的启动类注解，它是一个复合注解。其中的`@EnableAutoConfiguration`  表示开启自动配置，它也是一个复合注解，其中`@AutoConfigurationPackage` 注解会将启动类所在包下的所有子包的组件扫描注入到spring容器中，另一个注解 `@Import(AutoConfigurationImportSelector.class)` 则是会导入了一个类`EnableAutoConfigurationImportSelector` 加载自动配置类，该类方法可以查找位于META-INF/spring.factories文件中的所有自动配置类，并加载这些类。

前面加载的所有自动配置类并不是都生效的，每一个`xxxAutoConfiguration`自动配置类都是在某些特定的条件之下才会生效。这些条件限制是通过`@ConditionOnxxx`注解实现的。

参考文章：[Spring Boot自动装配](https://blog.csdn.net/weixin_42556307/article/details/108405009)





























