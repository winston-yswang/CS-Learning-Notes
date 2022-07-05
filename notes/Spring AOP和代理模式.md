## 1、AOP基本概念

AOP（Aspect-oriented Programming）面向切面编程，为了解决横切逻辑代码存在的**代码重复问题、以及横切逻辑代码和业务代码混杂在一起，代码臃肿，不便于维护等问题**，而提出的一种编程思想。

它提出了横向抽取机制，将横切逻辑代码和业务逻辑代码分离。使得开发者在不改变源代码的前提下，为系统中不同业务组件添加某些通用功能。

![aop01](pics\aop01.webp)

在学习如何实现AOP前，需要了解的AOP常用的定义：

![aop02](pics\aop02.webp)

- JoinPoint（连接点）：**能够被拦截的地方**，由于Spring AOP是基于动态代理的，所以是类中的成员方法都可以被增强，可以称为连接点。
- Pointcut（切点）：**具体定位的JoinPoints**，JoinPoints 只是切面代码可以被织入的地方，但我并不想对所有的 JoinPoint 进行织入，这就需要某些条件来筛选出那些需要被织入的 JoinPoint，Pointcut 就是通过一组规则(使用 AspectJ pointcut expression language 来描述) 来定位到匹配的 joinpoint。
- **Advice**:  代码织入（也叫增强），**表示添加到切点的一段逻辑代码，并包含增强的位置信息**。Pointcut 通过其规则指定了哪些 joinpoint 可以被织入，而 Advice 则指定了这些 joinpoint 被织入（或者增强）的具体时机与逻辑，是**切面代码真正被执行的地方**，主要有五个织入时机
  1. Before Advice: 在 JoinPoints 执行前织入
  2. After Advice: 在 JoinPoints 执行后织入（不管是否抛出异常都会织入）
  3. After returning advice: 在 JoinPoints 执行正常退出后织入（抛出异常则不会被织入）
  4. After throwing advice: 方法执行过程中抛出异常后织入
  5. Around Advice: 这是所有 Advice 中最强大的，它在 JoinPoints 前后都可织入切面代码，也可以选择是否执行原有正常的逻辑，如果不执行原有流程，它甚至可以用自己的返回值代替原有的返回值，甚至抛出异常。在这些 advice 里我们就可以写入切面代码了 综上所述，切面（Aspect）我们可以认为就是 pointcut 和 advice，pointcut 指定了哪些 joinpoint 可以被织入，而 advice 则指定了在这些 joinpoint 上的代码织入时机与逻辑

## 2、AOP的实现

AOP 是 Spring 的重要组成部分，虽然 Spring IoC 容器不依赖于AOP，但 AOP 补充了 Spring IoC，提供了一个非常强大的中间件解决方案。

实现AOP功能的框架主要有Spring AOP和AspectJ。

- Spring AOP使用的动态代理，运行时生成 AOP 代理类，缺点是由于 Spring AOP 需要在每次运行时生成 AOP 代理，因此性能略差一些。
- AspectJ是静态代理的增强，采用编译时生成 AOP 代理类，因此也称为编译时增强，具有更好的性能。 缺点是需要使用特定的编译器进行处理。

在 spring 中支持基于注解和基于XML两种方式配置AOP，两者差别不大，下面主要来举个例子说明基于注解如何实现AOP。

首先是导入相关 jar 包，主要是spring aop 和 aspect 。

```xml
<!--spring aop支持-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>5.2.9.RELEASE</version>
</dependency>

<!--aspectj支持-->
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.9.8.RC1</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.8.RC1</version>
</dependency>
```

然后需要在xml中导入aop的命名空间，并且开启AOP 注解，此外别忘了包扫描！！

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--scan package-->
    <context:component-scan base-package="tech.wys.springTest"/>

    <!--start AOP annotation-->
    <aop:aspectj-autoproxy/>
    
</beans>

```

紧接着我们就需要指定实验类，其方法比如这里swim、run 和 jump都可以作为 Joinpoint。

```java
public interface TestService {
    // swim
    void swim();

    // run
    void run();

    // jump
    void jump();
}

@Component
public class TestServiceImpl implements TestService{

    @Override
    public void swim() {
        System.out.println("I'm swimming");
    }

    @Override
    public void run() {
        System.out.println("I'm running");
    }

    @Override
    public void jump() {
        System.out.println("I'm jumping");
    }
}
```

之后，筛选需要被织入的 JoinPoint 来定位到匹配的 joinpoint。

```java
@Aspect
@Component
public class TestAdvice {

    // 定义 pointcut
    // 匹配规则表达式含义 拦截 com.wys.springTest 包下 以及子包下 所有类的所有方法绑定到pointcut方法上
    @Pointcut("execution(* tech.wys.springTest..*.*(..))")
    public void pointcut() {}

    // 2.定义应用于 JoinPoint 中所有满足 PointCut 条件的 advice, 这里我们使用 around advice，在其中织入增强逻辑
    @Around("pointcut()")
    public void handlerRpcRequest(ProceedingJoinPoint point) throws Throwable {

        System.out.println("before pointcut...");

        point.proceed();    // 可视情况是否执行

        System.out.println("before pointcut...");
    }
}
```

最后测试一下，切面是否生效。

```java
public class MyTest {
    @Test
    public void myTest() {
        // create and configure beans
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-aop.xml");

        // 类名要小写！！！
        TestService testService = (TestService) context.getBean("testServiceImpl");

        testService.run();
    }
}
```



## 3、AOP实现原理与动态代理

AOP实现原理是基于动态代理，AOP框架会自动创建的AOP代理，在代理类中增强目标对象。

讲到动态代理，那我们不妨深入了解代理模式。

代理模式的定义也简单，是指为其他对象提供一种代理，以控制对这个对象的访问，属于一种结构性设计模式。

使用代理模式往往出于两个目的：一是保护目标对象，二是增强目标对象。

![proxy](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\proxy.webp)

代理模式包含如下角色：

- **subject**：抽象主题角色，是一个接口。该接口是对象和它的代理共用的接口; 
- **RealSubject**：真实主题角色，是实现抽象主题接口的类; 
- **Proxy**：代理角色，内部含有对真实对象RealSubject的引用，从而可以操作真实对象。代理对象提供与真实对象相同的接口，以便代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。
- Client：客户是直接和 Proxy 打交道的

代理主要分为两种类型：静态代理和动态代理，动态代理又有 JDK 代理和 CGLib 代理两种。

### 3.1 静态代理和动态代理的区别

要理解静态和动态这两个含义，我们首先需要理解一下 Java 程序的运行机制

![proxy](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\proxy2.webp)

首先 Java 源码经过编译生成字节码，然后再由 JVM 经过类加载，连接，初始化成 Java 类型，可以看到字节码是关键，静态和动态的区别就在于字节码生成的时机：

- **静态代理**：**由程序员创建代理类或特定工具自动生成源码再对其编译**。在编译时已经将接口，被代理类（委托类），代理类等确定下来，在**程序运行前**代理类的.class文件就已经存在了。
- **动态代理**：在**程序运行后**通过反射创建生成字节码再由 JVM 加载而成。

### 3.2 静态代理

```java
// 出租
public interface Rent {
    public void rent();
}

// 房东
public class Landlord implements Rent {
    @Override
    public void rent() {
        System.out.println("房东要出租房子！");
    }
}

public class Proxy implements Rent{
    private Landlord landlord;

    public Proxy() {};

    public Proxy(Landlord landlord) {
        this.landlord = landlord;
    }

    @Override
    public void rent() {
        landlord.rent();
        System.out.println("中介带看房与签合同");
    }

    public void otherTask() {
        System.out.println("看房与签合同");
    }
}

// 租客
public class Client {
    public static void main(String[] args) {
        // 房东
        Landlord landlord = new Landlord();
        // 代理，一般会有附加操作
        Proxy proxy = new Proxy(landlord);
        // 代理请求
        proxy.rent();
    }
}
```

静态代理虽然简单，但是有两大劣势：

1. 代理类只代理一个委托类（其实可以代理多个，但不符合单一职责原则），也就意味着如果要代理多个委托类，就要写多个代理；
2. 静态代理无法解决横切逻辑代码存在的代码重复问题，应用场景有限。

### 3.3 动态代理

动态代理的代理类是在运行期间动态生成的，恰好弥补静态代理的不足。动态代理主要分为 JDK 提供的动态代理和 Spring AOP 用到的 CGLib 生成的代理两种。

#### JDK 的动态代理

JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口。JDK动态代理的核心是InvocationHandler接口和Proxy类。

```java
// 出租
public interface Rent {
    public void rent();
}

// 房东
public class Landlord implements Rent {
    @Override
    public void rent() {
        System.out.println("房东要出租房子！");
    }
}

// 生成代理类
public class ProxyInvocationHandler {
    Object target;  // 被代理的接口对象，实际的方法执行者

    public void setTarget(Object target) {
        this.target = target;
    }


    public Object getProxy() {
        // 使用Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
        // 该方法用于为指定类装载器、一组接口及调用处理器生成动态代理类实例
        /**
         * ClassLoader loader:Java类加载器; 可以通过这个类型的加载器，在程序运行时，将生成的代理类加载到JVM即Java虚拟机中，以便运行时需要！
         * Class<?>[] interfaces:被代理类的所有接口信息; 便于生成的代理类可以具有代理类接口中的所有方法
         * InvocationHandler h:调用处理器; 调用实现了InvocationHandler 类的一个回调方法
         * */
        return Proxy.newProxyInstance(
                this.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new InvocationHandler() {
                    /**
                     * InvocationHandler接口只定义了一个invoke方法，直接使用一个匿名内部类来实现
                     * 在invoke方法编码指定返回的代理对象干的工作
                     * proxy : 把代理对象自己传递进来
                     * method：把代理对象当前调用的方法传递进来
                     * args:把方法参数传递进来
                     * */
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        Object result = method.invoke(target, args);
                        return result;
                    }
                });
    }
}

// 租客
public class Client {
    public static void main(String[] args) {
        Landlord landlord = new Landlord(); // 被代理对象
        ProxyInvocationHandler pih = new ProxyInvocationHandler();
        pih.setTarget(landlord);
        Rent proxy = (Rent) pih.getProxy(); // 生成代理对象，即com.sun.proxy.$Proxy0 类对象
        proxy.rent();
    }
}
```

同样的租房代理，可以注意到代理对象的 class 是  `com.sun.proxy.$Proxy0` , 它是由 `Proxy.newProxyInstance` 返回得到的，**该函数会从你传入的接口Class中，拷贝一份类结构信息到一个新的Class对象中，并且新的Class对象带有构造器，是可以创建对象的。根据代理Class的构造器创建对象时，需要传入InvocationHandler。每次调用代理对象的方法，最终都会转发到InvocationHandler的invoke()方法。**

```java
public static Object newProxyInstance(ClassLoader loader,
                                         Class<?>[] interfaces,
                                         InvocationHandler h);
// loader: 代理类的ClassLoader，最终读取动态生成的字节码，并转成 java.lang.Class 类的一个实例（即类），通过此实例的 newInstance() 方法就可以创建出代理的对象
// interfaces: 委托类实现的接口，JDK 动态代理要实现所有的委托类的接口
// InvocationHandler: 委托对象所有接口方法调用都会转发到 InvocationHandler.invoke()，在 invoke() 方法里我们可以加入任何需要增强的逻辑 主要是根据委托类的接口等通过反射生成的
```

这样做的好处在于，由于动态代理是程序运行后才生成的，哪个委托类需要被代理到，只要生成动态代理即可，避免了静态代理那样的硬编码，另外所有委托类实现接口的方法都会在 Proxy 的 InvocationHandler.invoke() 中执行，相同的逻辑可以统一在 InvocationHandler 里写， 也就避免了静态代理那样需要在所有的方法中插入同样代码的问题，代码的可维护性极大的提高了。

这样做的缺点也是明显，那就是参数 Interfaces 是委托类的接口，是必传的，JDK 动态代理是通过与委托类实现同样的接口，然后在实现的接口方法里进行增强来实现的，这就意味着**如果要用 JDK 代理，委托类必须实现接口**。



#### CGLib 的动态代理

如果目标类没有实现接口，就得选择使用CGLIB来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类（通过修改字节码来实现代理）。 

CGlib 动态代理也提供了类似的  Enhance 类，增强逻辑写在 MethodInterceptor.intercept() 中，也就是说所有委托类的**非 final 方法**都会被方法拦截器拦截，从而在拦截器里实现逻辑增强。

```java
public class MyMethodInterceptor implements MethodInterceptor {
   @Override
   public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
       System.out.println("目标类增强前！！！");
       //注意这里的方法调用，不是用反射
       Object object = proxy.invokeSuper(obj, args);
       System.out.println("目标类增强后！！！");
       return object;
   }
}

public class CGlibProxy {
   public static void main(String[] args) {
       //创建Enhancer对象，类似于JDK动态代理的Proxy类，下一步就是设置几个参数
       Enhancer enhancer = new Enhancer();
       //设置目标类的字节码文件
       enhancer.setSuperclass(RealSubject.class);
       //设置回调函数
       enhancer.setCallback(new MyMethodInterceptor());

       //这里的creat方法就是正式创建代理类
       RealSubject proxyDog = (RealSubject) enhancer.create();
       //调用代理类的eat方法
       proxyDog.request();
   }
}
/*
* 代理类:class com.example.demo.proxy.staticproxy.RealSubject$$EnhancerByCGLIB$$889898c5
* 目标类增强前！！！
* 卖房
* 目标类增强后！！！
*/
```

可以看到它并不要求委托类实现任何接口，而且 CGLIB 是高效的代码生成包，底层依靠 ASM（开源的 java 字节码编辑类库）操作字节码实现的，性能比 JDK 强，所以 Spring AOP 最终使用了 CGlib 来生成动态代理。

当然 CGLib 动态代理也有自身限制：只能代理委托类中任意的非 final 的方法，另外它是通过继承自委托类来生成代理的，所以如果委托类是 final 的，就无法被代理了。

最后，JDK 动态代理的拦截对象是通过反射的机制来调用被拦截方法的，而CGlib 则是采用了FastClass 的机制来实现对被拦截方法的调用。FastClass 机制就是对一个类的方法建立索引，通过索引来直接调用相应的方法。



参考资料：[漫画:AOP 面试造火箭事件始末](https://mp.weixin.qq.com/s/NXZp8a3n-ssnC6Y1Hy9lzw)