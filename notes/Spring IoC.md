### 1、IoC 控制反转

**IoC (Inversion of Control)**，控制反转就是把创建和管理 bean 的过程转移给了第三方，而这个第三方，就是 Spring IoC Container。所谓控制，是指控制 bean 的创建、管理的权利，控制 bean 的整个生命周期。而反转则是指把控制的权利交给了 Spring 容器，而不是自己去控制，就是反转。

容器负责创建、配置和管理 bean，也就是它管理着 bean 的生命，控制着 bean 的依赖注入。而**Bean** 则是由第三方包装好了的 Object，无论是控制反转还是依赖注入，它们的主语都是 object。

那么Spring IOC容器是怎么实现对象的创建和依赖的：

![ioc](pics\ioc.jpg)

1. 根据Bean配置信息在容器内部创建Bean定义注册表
2. 根据注册表加载、实例化bean、建立Bean与Bean之间的依赖关系
3. 将这些准备就绪的Bean放到Map缓存池中，等待应用程序调用

### 2、IoC Container

容器是IoC的重要部分，而 Spring Framework 的 IoC 容器实现依赖于 `ApplicationContext`，它是 `BeanFactory` 的子类，更好的补充并实现了 `BeanFactory` 的。

![image-20211231153015313](pics\image-20211231153015313.png)

`BeanFactory` 接口很简单，类似于HashMap，只有get，put 两个功能，所以称之为“低级容器”。

而 `ApplicationContext` 则多了很多功能，因为它继承了多个接口，可称之为“高级容器”。`ApplicationContext` 的里面有两个具体的实现子类，用来读取配置配件的：

- `ClassPathXmlApplicationContext` - 从 class path 中加载配置文件；
- `FileSystemXmlApplicationContext` - 从本地文件中加载配置文件。



### 3、DI 依赖注入

依赖注入，是IOC的一个方面，是个通常的概念，它有多种解释。这概念是说你不用创建对象，而只需要描述它如何被创建。你**不在代码里直接组装你的组件和服务，但是要在配置文件里描述哪些组件需要哪些服务**，之后一个容器（IOC容器）负责把他们组装起来。

我们知道控制反转是把对象之间的依赖关系提到外部去管理，可依赖是提到对象外面了，对象本身还是要用到依赖对象的，这时候就要用到依赖注入了。

![IOC](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\IOC.webp)

**控制反转的关注点是控制权的转移，而依赖注入则内含了控制反转的意义，明确的描述了依赖对象在外部被管理然后注入到对象中。**实现了依赖注入，控制也就反转了。

依赖注入方式：

- 通过对象的构造函数传参注入，这种叫做`构造器注入（Constructor Injection）`，适合实现强制依赖。

- 通过`JavaBean`的属性方法传参注入，就叫做`设值方法注入(Setter Injection)`，适合实现可选依赖。



### 4、Spring Framework

![springFramework](pics\springFramework.webp)

Spring 全家桶中最重要的几个项目都是基于 Spring Framework，Spring Framework由八大模块组成

，每个模块既可以单独使用，又可以与其他模块联合使用。

`Core Container` 则是 Spring 的核心模块，主要实现 IoC 容器，它依赖于4个 jar 包：

- Beans
- Core
- Context
- SpEL，spring express language

如果想要用 IoC 这个功能，需要把这 4个 jar 包导进去。其中，Core 模块是 Spring 的核心，Spring 的所有功能都依赖于这个 jar 包，Core 主要是实现 IoC 功能，那么说白了 Spring 的所有功能都是借助于 IoC 实现的。



### 5、构建Spring IoC项目

在构建 IoC 时推荐是使用 Maven来管理项目中的 jar 包，免除手动下载导入的繁琐。

首先是添加对应的 pom 依赖，导入4个 jar 包。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.9.RELEASE</version>
</dependency>
```

![image-20220102131113695](pics\image-20220102131113695.png)

然后是新建一个配置文件，配置项一些命名空间及其规范的介绍，还有就是bean里面的配置啦，详细参考官网。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="user" class="tech.wys.springTest.User">
        <!-- collaborators and configuration for this bean go here -->
        <property name="id" value="1"/>
        <property name="name" value="wys"/>
    </bean>


    <!-- more bean definitions go here -->

</beans>
```

最后一步，就是怎么使用这个容器。在Spring 中 `ApplicationContext` 是 IoC 容器的入口，其实也就是 Spring 容器的入口，它通过实现子类中读取数据，创建具体的 bean并使用。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class User {
    private int id;
    private String name;
}


public class MyTest {
    @Test
    public void myTest() {
        // create and configure beans
        ApplicationContext context = new ClassPathXmlApplicationContext("user.xml");

        // retrieve configured instance
        User user = context.getBean("user", User.class);

        System.out.println("id: " + user.getId() + " username: " + user.getName());
    }
}
```

从上面的配置和代码中，我们可以看出其实Spring主要就帮我们省略了 `new 的过程` ，把 new 的过程交给第三方来创建、管理，这就是 `解耦`。

此外，**bean对象是通过反射在每次启动容器时创建，并且在容器中默认是单例的。**

之所以默认是单例，为了适配Web应用环境相关的Bean作用域--->每个request都需要一个对象，此时我们**返回一个代理对象**出去就可以完成我们的需求了！

### 6、Spring IoC 相关面试题

> 使用Spring框架的好处是什么？

- **轻量**：Spring 是轻量的，基本的版本大约2MB。
- **控制反转**：Spring通过控制反转实现了松散耦合，对象们给出它们的依赖，而不是创建或查找依赖的对象们。
- **面向切面的编程**(AOP)：Spring支持面向切面的编程，并且把应用业务逻辑和系统服务分开。
- **容器**：Spring 包含并管理应用中对象的生命周期和配置。
- **MVC框架**：Spring的WEB框架是个精心设计的框架，是Web框架的一个很好的替代品。
- **事务管理**：Spring 提供一个持续的事务管理接口，可以扩展到上至本地事务下至全局事务（JTA）。
- **异常处理**：Spring 提供方便的API把具体技术相关的异常（比如由JDBC，Hibernate or JDO抛出的）转化为一致的unchecked 异常。

> 解释Spring框架中bean的生命周期

- Spring容器 从XML 文件中读取bean的定义，并实例化bean。
- Spring根据bean的定义填充所有的属性。
- 如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法。
- 如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法。
- 如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们。
- 如果bean实现IntializingBean了，调用它的afterPropertySet方法，如果bean声明了初始化方法，调用此初始化方法。
- 如果有BeanPostProcessors 和bean 关联，这些bean的postProcessAfterInitialization() 方法将被调用。
- 如果bean实现了 DisposableBean，它将调用destroy()方法。

> 解释不同方式的自动装配

- no：默认的方式是不进行自动装配，通过显式设置ref 属性来进行装配。
- byName：通过参数名 自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byname，之后容器试图匹配、装配和该bean的属性具有相同名字的bean。
- byType:：通过参数类型自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byType，之后容器试图匹配、装配和该bean的属性具有相同类型的bean。如果有多个bean符合条件，则抛出错误。
- constructor：这个方式类似于byType， 但是要提供给构造器参数，如果没有确定的带参数的构造器参数类型，将会抛出异常。
- autodetect：首先尝试使用constructor来自动装配，如果无法工作，则使用byType方式。

只用注解的方式时，**注解默认是使用byType的**！