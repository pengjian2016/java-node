
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

在 @SpringBootApplication 这个注解里面，有3个重要的注解

```
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })

```

1. @SpringBootConfiguration 注解里面，没做太多的事情，主要是对@Configuration注解的封装，主要作用是标注这个类是配置类，功能与@Configuration一样

2. @ComponentScan 是扫描包的注解，自动扫描符合条件的bean（如@Service、@Controller、@Repository等等注解的类）加载到容器中

3. @EnableAutoConfiguration 这个注解便是自动装配的关键，在该注解中会去 引入 @Import(AutoConfigurationImportSelector.class) 这个类，它实现了DeferredImportSelector.Group中的process方法其内部会调用getAutoConfigurationEntry()方法，springboot启动得时候会找到ImportSelector类并调用process方法，下面主要分析getAutoConfigurationEntry源码（当前基于springboot 2.3.4 如果你发现你看的源码不一样，那可能是版本不一样的原因）：


```

protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // getCandidateConfigurations 方法会调用 SpringFactoriesLoader.loadFactoryNames()
        // 在 loadFactoryNames() 中通过loadSpringFactories()方法加载 META-INF/spring.factories 配置文件
        // 读取配置文件中的内容后过滤出 org.springframework.boot.autoconfigure.EnableAutoConfiguration 这个key对应的所有value 
        // 返回的内容即是要加载的所有类，但是并不是所有的模块都能用到，所以下面会进行过滤，过滤掉不满足条件的配置
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
        // 对configurations进行过滤，剔除掉@Conditional条件不成立的配置类（比如某个配置可能需要某个特定的类，该类属于某个包，如果未引入包，则会过滤掉）
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}

META-INF/spring.factories 文件 org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的部分内容（太长了）:
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
....

```
所以如果面试官问你springboot如果实现自动装配的，可以这样大致的回答：

在@SpringBootApplication  注解下有三个重要的注解，其中有一个@EnableAutoConfiguration注解，它会使用@Import注解引入AutoConfigurationImportSelector.class 这个类，自动装配主要在这个类中完成，它会去加载META-INF/spring.factories 配置文件，找到org.springframework.boot.autoconfigure.EnableAutoConfiguration 这个key对应的所有配置，剔除那些不满足条件的配置，剩下的就是要自动装配的类。


参考：

[AutoConfigurationImportSelector到底怎么初始化](https://zhuanlan.zhihu.com/p/143912268)

[SpringBoot自动装配原理与源码分析](http://autumn200.com/2020/06/27/spring-boot-autoconfig/)

### springboot 启动过程

在我们自定义的Application类的main方法中，会执行下面这样的代码：

```
SpringApplication.run(MyApplication.class, args);
```
启动过程从这里开始，源码如下（当前基于 springboot 2.3.4  ）：

```
    /**
     * SpringApplication.run() 方法点击进去之后调用到这里
     * */
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class<?>[] { primarySource }, args);
    }
    /**
     * 最后实际调用的构造函数
     * */
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        /**
         * 确定应用类型，如果有org.springframework.web.reactive.DispatcherHandler 这个类，而没有
         * org.springframework.web.servlet.DispatcherServlet，也没有org.glassfish.jersey.servlet.ServletContainer
         * 则认为是 reactive （响应式应用）应用，
         * 如果有javax.servlet.Servlet 和 org.springframework.web.context.ConfigurableWebApplicationContext
         * 则认为是 servlet 即web应用
         * **/
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        /**
         * 从spring.factories配置文件中获得ApplicationContextInitializer接口的实现类
         * 设置到 initializers 属性中
         * */
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        /**
         * 从spring.factories配置文件中获得ApplicationListener接口的实现类
         * 设置到 listeners 属性中
         * */
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 获取哪个类调用了main函数，主要用来打印日志
        this.mainApplicationClass = deduceMainApplicationClass();
    }
    
    /**
     * 初始化好SpringApplication对象之后，会直接调用run方法
     * */
    public ConfigurableApplicationContext run(String... args) {
        // 主要用于记录run方法执行的时长，调用start记录开始时间，stop计算两个差值
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        // 存放启动过程的异常信息
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        // 设置 java.awt.headless 模式，默认为true，即无需键盘鼠标这些设备
        configureHeadlessProperty();
        // 从spring.factories配置中获取SpringApplicationRunListeners的实现类，并启动监听
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            // 封装 main方法传递的args 参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 准备环境，将各个配置环境整合起来，如命令行配置的参数或配置文件配置的参数等所有的配置项
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            // 设置spring.beaninfo.ignore 为true ，即跳过搜索beaninfo的classes
            configureIgnoreBeanInfo(environment);
            // 打印bannner
            Banner printedBanner = printBanner(environment);
            // 根据不同的应用类型（webApplicationType）创建ApplicationContext 容器
            // 在初始化的时候webApplicationType默认的是SERVLET，所以这里创建的是AnnotationConfigServletWebServerApplicationContext
            context = createApplicationContext();
            // 从spring.factories配置文件中获取异常报告实例
            exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            // 为刚ApplicationContext容器做一些准备工作，如设置上一步初始化好的环境变量到容器中
            // 对ApplicationContext做后置处理，如设置resourceLoader属性
            // 调用初始化时生成好的ApplicationContextInitializer实例的initialize方法
            // 调用SpringApplicationRunListeners 的contextPrepared方法，发送ApplicationContextInitializedEvent事件，表示容器已经准备化好
            // 从context容器中获取beanFactory，并向beanFactory中注册一些单例bean(applicationArguments，printedBanner)
            // 加载bean到application context，只是加载了部分bean比如mainApplication这个bean
            // 调用SpringApplicationRunListeners的contextLoaded 事件，表示容器已加载
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //
            refreshContext(context);
            //
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
            }
            //
            listeners.started(context);
            //
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }

    

```


### springboot 内置web容器有哪些
tomcat，默认最大线程数200

jetty，undertow

### springboot中重要的注解

### 如何实现自定义注解

注解中的方法支持哪些返回类型，8种基本类型，String，Class和他们对应的数组类型，以及枚举类型。


### spring bean 的生命周期

### spring aop 和拦截器的区别
