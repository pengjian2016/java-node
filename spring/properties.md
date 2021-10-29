# springboot 内置容器Tomcat、Jetty、undertow、Reactor Netty

上一章节中，简单的提了一下 springboot 内置web容器有哪些。

Tomcat、Jetty、undertow 它们都是web容器，能提供RestFulApi的接口能力，同时还能提供响应视图页面的web能力。

在微服务的大环境里，或者前后端分离的设计方案中，有时候我们只希望提供RestFulApi能力，并不需要返回web页面的能力。
因而引入了 Reactor Netty。当然这里不仅仅是介绍Reactor Netty，下面将对这几个容器分别做一下介绍。

上一节中我们同样讲到了springboot的自动装配原理，知道启动后会去找spring.factories配置文件中对应的自动装配配置类

其中org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration 就是springboot容器相关的配置类，这个类的部分代码如下：

```
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
}

```

可以看到其中会引入EmbeddedTomcat、EmbeddedJetty、EmbeddedUndertow 三个类，因为这里都是web容器，所以并没有netty

查看ServletWebServerFactoryConfiguration中的源码：

```
@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedTomcat {

		@Bean
		TomcatServletWebServerFactory tomcatServletWebServerFactory(
				ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
				ObjectProvider<TomcatContextCustomizer> contextCustomizers,
				ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
			TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
			factory.getTomcatConnectorCustomizers()
					.addAll(connectorCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatContextCustomizers()
					.addAll(contextCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatProtocolHandlerCustomizers()
					.addAll(protocolHandlerCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

	/**
	 * Nested configuration if Jetty is being used.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedJetty {

		@Bean
		JettyServletWebServerFactory JettyServletWebServerFactory(
				ObjectProvider<JettyServerCustomizer> serverCustomizers) {
			JettyServletWebServerFactory factory = new JettyServletWebServerFactory();
			factory.getServerCustomizers().addAll(serverCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

	/**
	 * Nested configuration if Undertow is being used.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedUndertow {

		@Bean
		UndertowServletWebServerFactory undertowServletWebServerFactory(
				ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers,
				ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
			UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
			factory.getDeploymentInfoCustomizers()
					.addAll(deploymentInfoCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getBuilderCustomizers().addAll(builderCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

}
```

这里会根据条件创建容器，比如Tomcat，如果满足@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
这些条件就会创建Tomcat容器，而如果你想要使用Jetty，则需要排除Tomcat依赖。

@EnableConfigurationProperties(ServerProperties.class) 这段代码中可以看到关于这几个容器的默认配置：

```
代码过长，只能贴几段核心的代码块
public class ServerProperties {
	private Integer port;
	private InetAddress address;
    ...
	private final Servlet servlet = new Servlet();

	private final Tomcat tomcat = new Tomcat();

	private final Jetty jetty = new Jetty();

	private final Netty netty = new Netty();

	private final Undertow undertow = new Undertow();

	public static class Tomcat {
		private final Threads threads = new Threads();
		private int maxConnections = 8192;

		private int acceptCount = 100;

		private int processorCache = 200;

		public static class Threads {

			private int max = 200;

			private int minSpare = 10;
			...
		}
	}

	public static class Jetty {
		private final Threads threads = new Threads();

		private DataSize maxHttpFormPostSize = DataSize.ofBytes(200000);

		private Duration connectionIdleTimeout;

		public static class Threads {
			private Integer acceptors = -1;
			private Integer selectors = -1;
			private Integer max = 200;
			private Integer min = 8;
			private Duration idleTimeout = Duration.ofMillis(60000);
			...
		}
	}

	public static class Netty {
		
		private Duration connectionTimeout;

		public Duration getConnectionTimeout() {
			return this.connectionTimeout;
		}

		public void setConnectionTimeout(Duration connectionTimeout) {
			this.connectionTimeout = connectionTimeout;
		}

	}

	public static class Undertow {
		private final Threads threads = new Threads();
		public static class Threads {
			private Integer io;
			private Integer worker;
			...
		}
	}

}

```

限于篇幅，只摘抄了部分代码。

可以看到Tomcat中，默认的最大线程数是200，最小是10，最大连接数8192

连接数表示能接受最大多少个请求建立连接，而每个请求又必须要有对应的线程处理，最大线程数表示的是最大能同时处理多少个请求。

Jetty ，默认的最大线程数是200，最小是8

Undertow 需要自己指定io线程数（一般为 2 或者 CPU 的个数），和工作线程数（一般为io 线程个数的 8 倍）

如何使用Netty替换其他web容器

参考这边文章[SpringBoot项目中使用Netty替换Tomcat，Undertow等Web容器](https://www.jianshu.com/p/89403b1b314d)

Netty 的优点

- 1 netty使用多路复用技术大幅提升性能
- 2 减少web容器依赖，减少jar包体积
- 3 灵活配置简单，适合所有需要提供restful接口的微服务应用
- 4 完全按照springmvc的模式开发配置

缺点：目前springboot配置中没有直接支持netty的相关配置，需要自己写代码实现

还没能做到和spirng DispatcherServlet那么强大到支持各种规则的path配置


### 那么Tomcat、Jetty、undertow、Reactor Netty这些容器的各自的优缺点以及适用场景是什么样的呢？

| 容器                                  | 优点 | 缺点 | 适用场景 |
|-------------------------------------|----|----|------|
| Tomcat | 成熟稳定（毕竟这么多年的积累），使用更广泛，很多漏洞也已经暴露出来并被解决  | 高并发情况下，性能表现略差（实际跟undertow差距并发太大） | 大多数web应用  |
| Jetty  |  与tomcat类似，比tomcat稍轻量，可扩展及易嵌入，性能方便二者差不多，tomcat处理短链接时性能较高，而jetty在处理长连接方面表现较好  |    |  云平台、分布式环境，Web聊天应用   |
| undertow | 高性能 在多款同类产品的压测中，在高并发情况下表现出色，相比tomcat更加轻量级   | 学习资料相对较少 |   适合追求高并发的场景  |
|Reactor Netty   | | | |


