# jpa 实现原理？

我们定义JPA的接口，却不用去做具体的实现，为什么它能正常执行呢？
```
// 例子:定义JPA 接口
public interface UserRepository extends JpaRepository<Operator, Long> {
     Operator findByUsername(String username);
}
// 使用
Operator user = userRepository.findByUsername(userName);
```

首先从JPA的自动配置开始，在springboot自动装配原理那一节我们知道springboot启动后会找到spring.factories 配置文件中所有的要自动装配的类，过滤调那些不满足条件的剩下的就是需要自动装配的类，而我们要使用jpa，需要引入spring-data-jpa 这个包，而jpa对应的自动装配配置类为：org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(DataSource.class)
@ConditionalOnClass(JpaRepository.class)
@ConditionalOnMissingBean({ JpaRepositoryFactoryBean.class, JpaRepositoryConfigExtension.class })
@ConditionalOnProperty(prefix = "spring.data.jpa.repositories", name = "enabled", havingValue = "true",
		matchIfMissing = true)
@Import(JpaRepositoriesRegistrar.class)
@AutoConfigureAfter({ HibernateJpaAutoConfiguration.class, TaskExecutionAutoConfiguration.class })
public class JpaRepositoriesAutoConfiguration {
}
```
该类主要就是一些条件的设置，不过它会@Import(JpaRepositoriesRegistrar.class) 这个类：
```
// 继承了AbstractRepositoryConfigurationSourceSupport  抽象类
class JpaRepositoriesRegistrar extends AbstractRepositoryConfigurationSourceSupport {
	@Override
	protected Class<? extends Annotation> getAnnotation() {
		return EnableJpaRepositories.class;
	}
	@Override
	protected Class<?> getConfiguration() {
		return EnableJpaRepositoriesConfiguration.class;
	}
	@Override
	protected RepositoryConfigurationExtension getRepositoryConfigurationExtension() {
		return new JpaRepositoryConfigExtension();
	}
        ...
}

// 在该抽象类中的registerBeanDefinitions 方法中，会使用RepositoryConfigurationDelegate  创建代理类，并调用registerRepositoriesIn方法
public abstract class AbstractRepositoryConfigurationSourceSupport
		implements ImportBeanDefinitionRegistrar, BeanFactoryAware, ResourceLoaderAware, EnvironmentAware {
        ...
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
			BeanNameGenerator importBeanNameGenerator) {
		RepositoryConfigurationDelegate delegate = new RepositoryConfigurationDelegate(
				getConfigurationSource(registry, importBeanNameGenerator), this.resourceLoader, this.environment);
		delegate.registerRepositoriesIn(registry, getRepositoryConfigurationExtension());
	}

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		registerBeanDefinitions(importingClassMetadata, registry, null);
	}
        ...
}
// 在 RepositoryConfigurationDelegate 代理类的registerRepositoriesIn方法中
public class RepositoryConfigurationDelegate {
    public List<BeanComponentDefinition> registerRepositoriesIn(BeanDefinitionRegistry registry,
			RepositoryConfigurationExtension extension) {

		extension.registerBeansForRoot(registry, configurationSource);

		RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension,
				configurationSource, resourceLoader, environment);
		List<BeanComponentDefinition> definitions = new ArrayList<>();

		StopWatch watch = new StopWatch();

		watch.start();
                // 扫描所有的自定义的JpaRepository
		Collection<RepositoryConfiguration<RepositoryConfigurationSource>> configurations = extension
				.getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode);

		Map<String, RepositoryConfiguration<?>> configurationsByRepositoryName = new HashMap<>(configurations.size());

		for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : configurations) {

			configurationsByRepositoryName.put(configuration.getRepositoryInterface(), configuration);

			BeanDefinitionBuilder definitionBuilder = builder.build(configuration);

			extension.postProcess(definitionBuilder, configurationSource);

			if (isXml) {
				extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
			} else {
				extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
			}

			AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
			beanDefinition.setResourceDescription(configuration.getResourceDescription());

			String beanName = configurationSource.generateBeanName(beanDefinition);

			beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());

			registry.registerBeanDefinition(beanName, beanDefinition);
			definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
		}

		potentiallyLazifyRepositories(configurationsByRepositoryName, registry, configurationSource.getBootstrapMode());

		watch.stop();
		return definitions;
	}
    
}
```
在 RepositoryConfigurationDelegate 代理类的registerRepositoriesIn方法中 省略了一些日志代码，这里主要的地方是，先创建了RepositoryBeanDefinitionBuilder 这个builder，然后通过 extension.getRepositoryConfigurations() 扫描所有我们自定义的JpaRepository，并遍历Repository

![输入图片说明](https://images.gitee.com/uploads/images/2021/0429/110425_f573305c_8076629.png "屏幕截图.png")

在遍历的过程中通过BeanDefinitionBuilder definitionBuilder = builder.build(configuration); 该方法对我们的接口进行封装：
```
class RepositoryBeanDefinitionBuilder {
    public BeanDefinitionBuilder build(RepositoryConfiguration<?> configuration) {

        // 创建builder 跟踪代码发现configuration.getRepositoryFactoryBeanClassName() 最后实际返回的是 JpaRepositoryFactoryBean.class.getName();
	BeanDefinitionBuilder builder = BeanDefinitionBuilder
			.rootBeanDefinition(configuration.getRepositoryFactoryBeanClassName());

	builder.getRawBeanDefinition().setSource(configuration.getSource());
	builder.addConstructorArgValue(configuration.getRepositoryInterface());
	builder.addPropertyValue("queryLookupStrategyKey", configuration.getQueryLookupStrategyKey());
	builder.addPropertyValue("lazyInit", configuration.isLazyInit());
	builder.setLazyInit(configuration.isLazyInit());
	builder.setPrimary(configuration.isPrimary());

	configuration.getRepositoryBaseClassName()//
			.ifPresent(it -> builder.addPropertyValue("repositoryBaseClass", it));

	NamedQueriesBeanDefinitionBuilder definitionBuilder = new NamedQueriesBeanDefinitionBuilder(
			extension.getDefaultNamedQueryLocation());
	configuration.getNamedQueriesLocation().ifPresent(definitionBuilder::setLocations);

	builder.addPropertyValue("namedQueries", definitionBuilder.build(configuration.getSource()));

	registerCustomImplementation(configuration).ifPresent(it -> {
		builder.addPropertyReference("customImplementation", it);
		builder.addDependsOn(it);
	});

	BeanDefinitionBuilder fragmentsBuilder = BeanDefinitionBuilder
			.rootBeanDefinition(RepositoryFragmentsFactoryBean.class);

	List<String> fragmentBeanNames = registerRepositoryFragmentsImplementation(configuration) //
			.map(RepositoryFragmentConfiguration::getFragmentBeanName) //
			.collect(Collectors.toList());

	fragmentsBuilder.addConstructorArgValue(fragmentBeanNames);

	builder.addPropertyValue("repositoryFragments",
			ParsingUtils.getSourceBeanDefinition(fragmentsBuilder, configuration.getSource()));

	return builder;
   }
}
```
我们所有的自定义的JpaRepository最后都会通过JpaRepositoryFactoryBean 这个工厂对象创建：

```
public class JpaRepositoryFactoryBean<T extends Repository<S, ID>, S, ID>
		extends TransactionalRepositoryFactoryBeanSupport<T, S, ID> {
    @Override
	public void afterPropertiesSet() {
		super.afterPropertiesSet();
	}
}
// 调用的父类 afterPropertiesSet方法
public abstract class RepositoryFactoryBeanSupport<T extends Repository<S, ID>, S, ID>
		implements InitializingBean, RepositoryFactoryInformation<S, ID>, FactoryBean<T>, BeanClassLoaderAware,
		BeanFactoryAware, ApplicationEventPublisherAware {
    public void afterPropertiesSet() {

		this.factory = createRepositoryFactory();
		this.factory.setQueryLookupStrategyKey(queryLookupStrategyKey);
		this.factory.setNamedQueries(namedQueries);
		this.factory.setEvaluationContextProvider(
				evaluationContextProvider.orElseGet(() -> QueryMethodEvaluationContextProvider.DEFAULT));
		this.factory.setBeanClassLoader(classLoader);
		this.factory.setBeanFactory(beanFactory);

		if (publisher != null) {
			this.factory.addRepositoryProxyPostProcessor(new EventPublishingRepositoryProxyPostProcessor(publisher));
		}

		repositoryBaseClass.ifPresent(this.factory::setRepositoryBaseClass);

		RepositoryFragments customImplementationFragment = customImplementation //
				.map(RepositoryFragments::just) //
				.orElseGet(RepositoryFragments::empty);

		RepositoryFragments repositoryFragmentsToUse = this.repositoryFragments //
				.orElseGet(RepositoryFragments::empty) //
				.append(customImplementationFragment);

		this.repositoryMetadata = this.factory.getRepositoryMetadata(repositoryInterface);

		// Make sure the aggregate root type is present in the MappingContext (e.g. for auditing)
		this.mappingContext.ifPresent(it -> it.getPersistentEntity(repositoryMetadata.getDomainType()));

		this.repository = Lazy.of(() -> this.factory.getRepository(repositoryInterface, repositoryFragmentsToUse));

		if (!lazyInit) {
			this.repository.get();
		}
	}
}
```
在afterPropertiesSet 方法中创建了实际的repository，最后那段代码，如果不是lazy 则获取该repository：
![输入图片说明](https://images.gitee.com/uploads/images/2021/0429/113716_01ca7cbb_8076629.png "屏幕截图.png")

调试时跟踪的内容，可以看到最后实际创建的是org.springframework.data.jpa.repository.support.SimpleJpaRepository 这个类的代理类。
```
// 部分方法
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
    @Override
    public Optional<T> findById(ID id) {
	Assert.notNull(id, ID_MUST_NOT_BE_NULL);
	Class<T> domainType = getDomainClass();
	if (metadata == null) {
		return Optional.ofNullable(em.find(domainType, id));
	}

	LockModeType type = metadata.getLockModeType();

	Map<String, Object> hints = getQueryHints().withFetchGraphs(em).asMap();

	return Optional.ofNullable(type == null ? em.find(domainType, id, hints) : em.find(domainType, id, type, hints));
    }
    @Transactional
    @Override
    public <S extends T> S save(S entity) {

	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
    }	
}
```
可以看到SimpleJpaRepository 其实已经实现了 基础的CRUD方法，不过其核心还是主要通过EntityManager em 来实现具体的逻辑

而我们在接口中自定义的方法，如一开始中的例子，findByUsername 这种方法，在SimpleJpaRepository 代码里面是没有的，只是在生成SimpleJpaRepository 动态代理时添加上去了这个方法

![输入图片说明](https://images.gitee.com/uploads/images/2021/0429/144118_6ab17ca4_8076629.png "屏幕截图.png")
 
那么这个方法具体怎么使用的呢？即使在SimpleJpaRepository 有了该代理方法，但是也没有它的实现啊，那它又是如何调用的呢？


# jpa 如何将调用的方法翻译为sql语句的？