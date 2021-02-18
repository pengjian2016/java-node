#spring IOC和AOP

什么是IOC？

Inversion of Control 控制反转，在开发过程中对象的创建及管理过程交由容器来完成，不再是我们自己创建，控制权由程序代码转移到了Spring容器，控制权发生了反转，即控制反转。又叫依赖注入。

Spring IOC提供了两种IOC容器，分别是BeanFactory和ApplicationContext。




什么是AOP？

AOP的全程是"Aspect Oriented Programming"，即面向切面编程，它将业务逻辑的各部分进行隔离，使开发人员在编写业务逻辑时可以专心核心业务，从而提高开发基础。

AOP如何实现呢？答案就是代理对象。

Spring AOP使用动态代理技术在运行期间织入增强的代码，主要有两种代理机制：基于JDK的动态代理；基于cglib的动态代理。当被代理的类实现了接口则使用JDK动态代理，否则使用cglib。

jdk动态代理和cglib动态代理的区别?

JDK动态代理只能代理实现了接口的类，而cglib不仅可以代理实现了接口的类同时还能代理普通类，主要是因为它是继承被代理的类，在其上实现增强效果，当然因为是继承关系，所以被代理的类要要注意不能是final修饰的，不能是private方法，否则没有效果。