# spring-source-study
spring 源码学习



# Spring 源码



## 一、Spring IOC源码

​		`IoC`（`Inversion of Control`，控制倒转）。这是`spring`的核心，贯穿始终。所谓`IoC`，对于`spring`框架来说，就是由`spring`来负责控制对象的生命周期和对象间的关系。所谓控制反转，就是把原先我们代码里面需要实现的对象创 建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让 容器知道需要创建的对象与对象的关系。这个描述最具体表现就是我们所看到的配置文件。	

​		spring boot项目启动启动，IOC初始化流程。

```java
SpringApplication.run(UserApplication.class, args);
进入SpringApplication类，跟踪run方法，直到ConfigurableApplicationContext run(String... args)方法。

调用 createApplicationContext()方法，方法内部反射 new AnnotationConfigApplicationContext(),进入AnnotationConfigApplicationContext的无参构造函数。
//默认构造函数，初始化一个空容器，容器不包含任何 Bean 信息，需要在稍后通过调用其 register() 
//方法注册配置类，并调用 refresh()方法刷新容器，触发容器对注解 Bean 的载入、解析和注册过程
	public AnnotationConfigApplicationContext() {
  	//Bean 定义读取器
		this.reader = new AnnotatedBeanDefinitionReader(this);
  	// Bean 定义的扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

进入AnnotatedBeanDefinitionReader类的无参构造函数
  public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

再次进入AnnotatedBeanDefinitionReader类的构造函数
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

进入AnnotationConfigUtils类void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry)方法
registerAnnotationConfigProcessors(registry, null);
调用Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(BeanDefinitionRegistry registry, @Nullable Object source)方法
在调用BeanDefinitionHolder registerPostProcessor(BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName)方法

进入GenericApplicationContext类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法
  
进入DefaultListableBeanFactory类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法，该方法会把BeanDefinition缓存到beanDefinitionMap中，以便后面Bean的依赖注入做准备
  //向 IOC 容器注册解析的 BeanDefiniton
	/** Map of bean definition objects, keyed by bean name. */
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);


回到AnnotationConfigApplicationContext类无参构造函数，然后进入ClassPathBeanDefinitionScanner类的无参构造函数
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
		this(registry, true);
	}

调用ClassPathBeanDefinitionScanner构造函数，直到调用构造函数
ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters, Environment environment, @Nullable ResourceLoader resourceLoader)
  
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

		if (useDefaultFilters) {
      // ClassPathScanningCandidateComponentProvider类void registerDefaultFilters()方法
      // 使用Spring默认的过滤规则，向容器注册过滤规则
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(resourceLoader);
	}
  
ClassPathScanningCandidateComponentProvider类void registerDefaultFilters()方法
包含 Component 注解(@Controller、@Service、@Re@Repository)、JSR-250 @ManagedBean注解、JSR-330 @Named注解
  
回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续调用void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) 方法
调用load()方法,再次调用int load(Object source)方法，然后进入BeanDefinitionLoader类int load(Class<?> source)方法
this.annotatedReader.register(source);

进入AnnotatedBeanDefinitionReader类void register(Class<?>... componentClasses)方法
registerBean(componentClass);

继续调用类中void registerBean(Class<?> beanClass)方法，然后继续调用
  <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers)方法
  
  //处理注解 Bean 定义中的通用注解
  AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

进入AnnotationConfigUtils类void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd)方法，在进入
  void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata)方法
  
然后回到 <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers)方法继续执行以下代码
//创建对于作用域的代理对象
definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);

再次进入AnnotationConfigUtils类BeanDefinitionHolder applyScopedProxyMode(
			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry)方法
  
进入BeanDefinitionReaderUtils类void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException方法
  BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);

进入GenericApplicationContext类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法
  registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

进入DefaultListableBeanFactory类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法 

回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续调用void refreshContext(ConfigurableApplicationContext context)
  refreshContext(context);
继续调用SpringApplication类void refreshContext(ConfigurableApplicationContext context)方法
  refresh((ApplicationContext) context);
调用void refresh(ApplicationContext applicationContext)方法
  refresh((ConfigurableApplicationContext) applicationContext);
调用void refresh(ConfigurableApplicationContext applicationContext)方法
	applicationContext.refresh();
进入AbstractApplicationContext类void refresh() throws BeansException, IllegalStateException方法
  invokeBeanFactoryPostProcessors(beanFactory);
	继续调用void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)方法
 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

进入PostProcessorRegistrationDelegate类void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)方法
  registryProcessor.postProcessBeanDefinitionRegistry(registry);

进入SharedMetadataReaderFactoryContextInitializer类void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException方法
  register(registry);
调用void register(BeanDefinitionRegistry registry)方法
  registry.registerBeanDefinition(BEAN_NAME, definition);
  
```



