# spring-source-study
spring 源码学习



# Spring 源码



## 一、Spring Boot启动流程及IOC源码

​		Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化Spring应用初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。Spring Boot其实就是一个整合很多可插拔的组件（框架），内嵌了使用工具（比如内嵌了Tomcat、Jetty等），方便开发人员快速搭建和开发的一个框架。

​		`IoC`（`Inversion of Control`，控制倒转）。这是`spring`的核心，贯穿始终。所谓`IoC`，对于`spring`框架来说，就是由`spring`来负责控制对象的生命周期和对象间的关系。所谓控制反转，就是把原先我们代码里面需要实现的对象创 建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让 容器知道需要创建的对象与对象的关系。这个描述最具体表现就是我们所看到的配置文件。	

​		spring boot项目启动启动，IOC初始化流程。

```java
SpringApplication.run(UserApplication.class, args);
进入SpringApplication类，跟踪run方法，直到ConfigurableApplicationContext run(String... args)方法。
  	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      //进入
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
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

进入方法ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments)方法
  private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
  	//进入
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}

进入SpringApplicationRunListeners类environmentPrepared(ConfigurableEnvironment environment)方法
  listeners.environmentPrepared(environment);

进入EventPublishingRunListener类void environmentPrepared(ConfigurableEnvironment environment)方法
  //SpringApplicationRunListeners类调用
  listener.environmentPrepared(environment);

进入SimpleApplicationEventMulticaster类void multicastEvent(ApplicationEvent event)方法
   //EventPublishingRunListener类调用
		this.initialMulticaster
				.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
继续调用SimpleApplicationEventMulticaster类方法
  1、void multicastEvent(ApplicationEvent event)方法
  	multicastEvent(event, resolveDefaultEventType(event));
  3、void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType)方法
    invokeListener(listener, event);
  2、void invokeListener(ApplicationListener<?> listener, ApplicationEvent event)方法
		doInvokeListener(listener, event);
  4、void doInvokeListener(ApplicationListener listener, ApplicationEvent event) 方法
 		listener.onApplicationEvent(event);
  
进入BootstrapApplicationListener类void onApplicationEvent(ApplicationEnvironmentPreparedEvent event)方法
	继续调用BootstrapApplicationListener类方法
  该方法会先判断是否监听事件，源码如下:
	// don't listen to events in a bootstrap context
  if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
    return;
  }
  1、ConfigurableApplicationContext bootstrapServiceContext(ConfigurableEnvironment environment, final SpringApplication application, String configName)方法
  context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);

进入SpringApplicationBuilder类ConfigurableApplicationContext run(String... args)类
  final ConfigurableApplicationContext context = builder.run();

继续跟进，再次调用了SpringApplication类ConfigurableApplicationContext run(String... args)方法
  this.context = build().run(args);
	继续调用SpringApplication类方法
	1、调用 createApplicationContext()方法，方法内部反射 new AnnotationConfigApplicationContext(),进入AnnotationConfigApplicationContext的无参构造函数。
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
继续调用AnnotatedBeanDefinitionReader类构造函数
	1、AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment)
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
      Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
      Assert.notNull(environment, "Environment must not be null");
      this.registry = registry;
      this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
      AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

进入AnnotationConfigUtils类void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry)方法
registerAnnotationConfigProcessors(registry, null);
继续调用AnnotationConfigUtils类方法
	1、Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(BeanDefinitionRegistry registry, @Nullable 			Object source)方法
  	registerAnnotationConfigProcessors(registry, null);
	2、BeanDefinitionHolder registerPostProcessor(BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName)方法
    beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));

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
  继续调用ClassPathBeanDefinitionScanner类构造函数
  1.ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters, Environment environment, @Nullable ResourceLoader resourceLoader)

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
  
进入ClassPathScanningCandidateComponentProvider类void registerDefaultFilters()方法
包含 Component 注解(@Controller、@Service、@Re@Repository)、JSR-250 @ManagedBean注解、JSR-330 @Named注解
  
回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续调用void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) 方法
  load(context, sources.toArray(new Object[0]));
  继续调用SpringApplication类方法
	1、load()方法,再次调用int load(Object source)方法
		loader.load();

进入BeanDefinitionLoader类int load(Class<?> source)方法
  count += load(source);
	继续调用BeanDefinitionLoader类方法
  1、int load(Object source)方法
    count += load(source);
	2、int load(Class<?> source)方法
		this.annotatedReader.register(source);

进入AnnotatedBeanDefinitionReader类void register(Class<?>... componentClasses)方法
registerBean(componentClass);
	继续调用AnnotatedBeanDefinitionReader类方法
	1、void registerBean(Class<?> beanClass)方法
    registerBean(componentClass);
	2、<T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers)方法
    doRegisterBean(beanClass, null, null, null, null);	
    //doRegisterBean方法中处理注解 Bean 定义中的通用注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

进入AnnotationConfigUtils类void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd)方法
  继续调用AnnotationConfigUtils类方法
  void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata)方法
  processCommonDefinitionAnnotations(abd, abd.getMetadata());
  
然后回到AnnotatedBeanDefinitionReader类 <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers)方法继续执行以下代码
//创建对于作用域的代理对象
definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);

再次进入AnnotationConfigUtils类BeanDefinitionHolder applyScopedProxyMode(
			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry)方法
  definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);

进入BeanDefinitionReaderUtils类void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException方法
  BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);

进入GenericApplicationContext类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法
  registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

进入DefaultListableBeanFactory类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法 
	this.beanFactory.registerBeanDefinition(beanName, beanDefinition);

回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续调用void refreshContext(ConfigurableApplicationContext context)
  refreshContext(context);
  继续调用SpringApplication类方法
  1、SpringApplication类void refreshContext(ConfigurableApplicationContext context)方法
  	refresh((ApplicationContext) context);
	2、void refresh(ApplicationContext applicationContext)方法
  	refresh((ConfigurableApplicationContext) applicationContext);
	3、void refresh(ConfigurableApplicationContext applicationContext)方法
		applicationContext.refresh();

进入AbstractApplicationContext类void refresh() throws BeansException, IllegalStateException方法
  invokeBeanFactoryPostProcessors(beanFactory);
	applicationContext.refresh();
	继续调用AbstractApplicationContex类方法
	1、void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)方法
 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
	invokeBeanFactoryPostProcessors(beanFactory);

进入PostProcessorRegistrationDelegate类void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)方法
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

进入SharedMetadataReaderFactoryContextInitializer类void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException方法
  registryProcessor.postProcessBeanDefinitionRegistry(registry);
  继续调用SharedMetadataReaderFactoryContextInitializer类方法
	1、void register(BeanDefinitionRegistry registry)方法
	register(registry);

进入DefaultListableBeanFactory类void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException方法
	registry.registerBeanDefinition(BEAN_NAME, definition);

回到PostProcessorRegistrationDelegate类void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)方法，继续调用void invokeBeanDefinitionRegistryPostProcessors(Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry)方法
  invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
	继续调用PostProcessorRegistrationDelegate类方法
  1、void invokeBeanDefinitionRegistryPostProcessors(
			Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry)
    postProcessor.postProcessBeanDefinitionRegistry(registry);

进入ConfigurationClassPostProcessor类void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)方法
  postProcessor.postProcessBeanDefinitionRegistry(registry);
	继续调用ConfigurationClassPostProcessor类方法
  1、void processConfigBeanDefinitions(BeanDefinitionRegistry registry)方法
    processConfigBeanDefinitions(registry);

进入ConfigurationClassParser类void parse(Set<BeanDefinitionHolder> configCandidates)方法
  parser.parse(candidates);
	继续调用ConfigurationClassParser类方法
  1、void parse(AnnotationMetadata metadata, String beanName) throws IOException方法
		parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
	2、void parse(AnnotationMetadata metadata, String beanName) throws IOException方法
    processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
	3、void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter)方法
    sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
  4、SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass, 
    Predicate<String> filter) throws IOException方法
    processMemberClasses(configClass, sourceClass, filter);
	5、void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass,
			Predicate<String> filter) throws IOException方法
  6、void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports)
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
	7、void processInterfaces(ConfigurationClass configClass, SourceClass sourceClass) throws IOException方法
    processInterfaces(configClass, sourceClass);
	8、回到ConfigurationClassParser类void parse(Set<BeanDefinitionHolder> configCandidates)方法
    继续调用void process()方法
  9、void processGroupImports()方法
    handler.processGroupImports();
	10、再次进入void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports)方法
    processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
		Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)), exclusionFilter, false);
	11、再次进入void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws  
    IOException方法
    processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);

回到ConfigurationClassPostProcessor类void processConfigBeanDefinitions(BeanDefinitionRegistry registry)方法
  继续调用void loadBeanDefinitions(Set<ConfigurationClass> configurationModel)方法
  this.reader.loadBeanDefinitions(configClasses);
	
进入ConfigurationClassBeanDefinitionReader类void loadBeanDefinitions(Set<ConfigurationClass> configurationModel)方法
  this.reader.loadBeanDefinitions(configClasses);
	1、void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass, TrackedConditionEvaluator 	
    trackedConditionEvaluator)方法
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
	2、void loadBeanDefinitionsFromImportedResources(
			Map<String, Class<? extends BeanDefinitionReader>> importedResources)方法
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
	3、void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars)方 
    法
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());

进入ImportBeanDefinitionRegistrar接口default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
			BeanNameGenerator importBeanNameGenerator)方法
  registrars.forEach((registrar, metadata) ->
                     registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));

进入EnableConfigurationPropertiesRegistrar类void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry)方法
  registerBeanDefinitions(importingClassMetadata, registry);
	继续调用EnableConfigurationPropertiesRegistrar类方法
  1、void registerInfrastructureBeans(BeanDefinitionRegistry registry)方法
    registerInfrastructureBeans(registry);

回到PostProcessorRegistrationDelegate类void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)方法，继续调用void invokeBeanDefinitionRegistryPostProcessors(Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry)方法
	继续调用PostProcessorRegistrationDelegate类方法
  继续往下执行，又调用了2次void invokeBeanDefinitionRegistryPostProcessors(
			Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry)
    postProcessor.postProcessBeanDefinitionRegistry(registry)方法
	继续往下执行调用了5次void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory)方法
  invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
	注：invokeBeanFactoryPostProcessors
  Bean工厂的后置处理器：BeanFactoryPostProcessor（触发时机：bean定义注册之后bean实例化之前）和BeanDefinitionRegistryPostProcessor（触发时机：bean定义注册之前），所以可以在Bean工厂的后置处理器中修改Bean的定义信息，比如是否延迟加载、加入一些新的Bean的定义信息等

回到AbstractApplicationContext类void refresh() throws BeansException, IllegalStateException方法
    继续调用void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)方法
    registerBeanPostProcessors(beanFactory);

再次进入PostProcessorRegistrationDelegate类void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext)方法
  PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
		继续调用PostProcessorRegistrationDelegate类方法
    1、void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors)方法，该方法被执行了5次，	 
      具体见源码，此处只做简单说明。
      registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
			
再次回到AbstractApplicationContext类void refresh() throws BeansException, IllegalStateException方法，继续执行后续逻辑，此处refresh方法使用到了模板模式，该方法中的后续方法不做说明。
  
最终回到ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments)方法，继续执行后续逻辑
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
  	//回到此处继续执行后续逻辑，执行完prepareEnvironment再次回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，实际回到第一次调用run方法地方，具体流程可以查看时序图，项目的启动执行了2次run方法
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
  
最终回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续执行后续逻辑，此处run方法使用到了模板模式。
  第一次执行run方法时，执行到prepareEnvironment方法时，在方法内部 bindToSpringApplication(environment)时又执行了一次builder.run(),及第一次执行run在bindToSpringApplication()方法被挂起，等待第一次执行流程结束之后，继续执行。因此第一次run会回到ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments)继续执行
  public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      //第一次执行，执行到此处时，在方法内部又做了一次builder.run()，及第二次进入ConfigurableApplicationContext run(String... args)方法，待第二次执行结束时，回到此处继续执行后续方法
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
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

进入AnnotationConfigServletWebServerApplicationContext类public AnnotationConfigServletWebServerApplicationContext()无参构造函数
  public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
构造函数中源码在AnnotationConfigApplicationContext类的无参构造函数已经分析，此处流程和AnnotationConfigApplicationContext类中执行流程一致
  
回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续执行private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner)方法
  prepareContext(context, environment, listeners, applicationArguments, printedBanner);
  继续调用SpringApplication类方法
  1、protected void applyInitializers(ConfigurableApplicationContext context)方法
		applyInitializers(context);
    最终执行ParentContextApplicationContextInitializer类void initialize(ConfigurableApplicationContext applicationContext)方法
  @Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		if (applicationContext != this.parent) {
      //设置父容器，this.parent是AnnotationConfigApplicationContext，applicationContext是AnnotationConfigServletWebServerApplicationContext
			applicationContext.setParent(this.parent);
      //创建监听器，主要用来发布项目中存在父子容器事件
			applicationContext.addApplicationListener(EventPublisher.INSTANCE);
		}
	}
	
回到SpringApplication类ConfigurableApplicationContext run(String... args)方法，继续执行private void refreshContext(ConfigurableApplicationContext context)方法  
	refreshContext(context);
	继续调用SpringApplication类方法
	1、private void refreshContext(ConfigurableApplicationContext context)方法
    refresh((ApplicationContext) context);
	2、protected void refresh(ApplicationContext applicationContext)方法
    refresh((ConfigurableApplicationContext) applicationContext);
	3、protected void refresh(ConfigurableApplicationContext applicationContext)方法
    applicationContext.refresh();

进入ServletWebServerApplicationContext类public final void refresh() throws BeansException, IllegalStateException方法
  super.refresh();

再次进入AbstractApplicationContextpublic void refresh() throws BeansException, IllegalStateException方法,进入该方法后，后续执行流程同上分析。
  
执行到ConfigurationClassParser类	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException 方法，第一次执行该方法时，由于componentScans为空，继续往下执行，不进入循环执行。
  源码如下
  @Nullable
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass, filter);
		}

		// Process any @PropertySource annotations
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
  	//第一次执行到该方法时，因为componentScans为空，不进入循环
  	//第二次执行到该方法时，因为componentScans非空，进入循环执行，因为在spring boot的启动类上加入了注解@SpringBootApplication，而@SpringBootApplication注解是一个复合注解，包含了@ComponentScan注解，因此componentScans非空
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// Process any @ImportResource annotations
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}

		// No superclass -> processing is complete
		return null;
	}

	由于在第一次进入方法时，并未分析如下源码部分，因此此处会进行详细分析。
    // Process any @ComponentScan annotations
  	//第一次执行到该方法时，因为componentScans为空，不进入循环
  	//第二次执行到该方法时，因为componentScans非空，进入循环执行，因为在spring boot的启动类上加入了注解@SpringBootApplication，而@SpringBootApplication注解是一个复合注解，包含了@ComponentScan注解，因此componentScans非空
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

进入ComponentScanAnnotationParser类public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass)方法
  return scanner.doScan(StringUtils.toStringArray(basePackages));
	
  方法中获取所需要扫描的包
  basePackages.add(ClassUtils.getPackageName(declaringClass));

进入ClassPathBeanDefinitionScanner类protected Set<BeanDefinitionHolder> doScan(String... basePackages)方法
  Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
	
进入ClassPathScanningCandidateComponentProvider类public Set<BeanDefinition> findCandidateComponents(String basePackage)方法
  //扫描basePackage包下的资源文件,得到BeanDefinitionHolder的集合
  return scanCandidateComponents(basePackage);

回到ConfigurationClassParser类protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException方法，继续执行后续逻辑
  拿到BeanDefinitionHolder的集合后，继续往下执行
  继续调用ConfigurationClassParser类方法
  1、protected final void parse(@Nullable String className, String beanName) throws IOException方法
  	parse(bdCand.getBeanClassName(), holder.getBeanName());

```



## 二、Spring Boot启动总结 

1、创建一个StopWatch并执行start方法，这个类主要记录任务的执行时间
2、配置Headless属性，Headless模式是在缺少显示屏、键盘或者鼠标时候的系统配置
3、在文件META-INF\spring.factories中获取SpringApplicationRunListener接口的实现类EventPublishingRunListener，主要发布   	    SpringApplicationEvent
4、把输入参数转成DefaultApplicationArguments类
5、创建Environment并设置比如环境信息，系统熟悉，输入参数和profile信息
6、打印Banner信息
7、创建Application的上下文，根据WebApplicationTyp来创建Context类，如果非web项目则创建AnnotationConfigApplicationContext，在构造方法中初始化AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
8、在文件META-INF\spring.factories中获取SpringBootExceptionReporter接口的实现类FailureAnalyzers
9、准备application的上下文
	a.初始化ApplicationContextInitializer
	b.执行Initializer的contextPrepared方法，发布ApplicationContextInitializedEvent事件
	c.如果延迟加载，在上下文添加处理器LazyInitializationBeanFactoryPostProcessor
	d.执行加载方法，BeanDefinitionLoader.load方法，主要初始化了AnnotatedGenericBeanDefinition
	e.执行Initializer的contextLoaded方法，发布ApplicationContextInitializedEvent事件
10、刷新上下文（后文会单独分析refresh方法），在这里真正加载bean到容器中。如果是web容器，会在onRefresh方法中创建一个Server并启动。



##  三、IOC容器初始化总结
  1、扫描资源
  2、加载资源，获取Bean定义
  3、注册Bean
    