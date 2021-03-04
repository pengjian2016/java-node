
### spring IOC和AOP

#####  什么是IOC？

Inversion of Control 控制反转，它是一种设计思想，在开发过程中对象的创建及管理过程交由容器来完成，不再是我们自己创建，控制权由程序代码转移到了Spring容器，控制权发生了反转，即控制反转。又叫依赖注入。

Spring IOC提供了两种IOC容器，分别是BeanFactory和ApplicationContext。



##### 什么是AOP？

AOP的全程是"Aspect Oriented Programming"，即面向切面编程，它将业务逻辑的各部分进行隔离，使开发人员在编写业务逻辑时可以专心核心业务，从而提高开发基础。

AOP如何实现呢？答案就是代理对象。

Spring AOP使用动态代理技术在运行期间织入增强的代码，主要有两种代理机制：基于JDK的动态代理；基于cglib的动态代理。当被代理的类实现了接口则使用JDK动态代理，否则使用cglib。

jdk动态代理和cglib动态代理的区别?

JDK动态代理只能代理实现了接口的类，而cglib不仅可以代理实现了接口的类同时还能代理普通类，主要是因为它是继承被代理的类，在其上实现增强效果，当然因为是继承关系，所以被代理的类要要注意不能是final修饰的，不能是private方法，否则没有效果。


### springMVC和springboot区别？

说道springMVC和springboot，不得不说一下spring框架，spring是什么？它是一个轻量级的java框架，是所有spring开源项目的总称，springmvc就是其中的一个开源项目。

springmvc是什么呢？

它是一个用于开发Web工程的MVC框架，MVC是什么？既是所谓的Model，View，Controller，实际上就是业务上的分层，比如Model层主要专注于数据方面的处理（主要与数据库等打交道），View专注于UI，Controller专注于逻辑处理等。springmvc就是集成这些东西，快速构建项目等。

springboot是什么呢？

SpringBoot是由Pivotal团队在2013年开始研发、2014年4月发布第一个版本的全新开源的轻量级框架。它基于Spring4.0设计，不仅继承了Spring框架原有的优秀特性，而且还通过简化配置来进一步简化了Spring应用的整个搭建和开发过程。另外SpringBoot通过集成大量的框架使得依赖包的版本冲突，以及引用的不稳定性等问题得到了很好的解决。简单来讲SpringBoot是一套整合了框架的框架。实际上也没有什么新东西，就是把之前搞得一堆东西融合在一块，更加优雅，更加容易上手等。

区别：springmvc更加专注于web项目，springboot更倾向于微服务接口等（前后端分离，移动端接口等）

### springboot 自动装配原理

### springboot 启动过程

### springboot 内置web容器有哪些
tomcat，默认最大线程数200

jetty，undertow

### springboot中重要的注解

### 如何实现自定义注解

注解中的方法支持哪些返回类型，8种基本类型，String，Class和他们对应的数组类型，以及枚举类型。


### spring bean 的生命周期

### spring aop 和拦截器的区别
