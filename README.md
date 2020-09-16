# spring-source-study
spring 源码学习



# Spring 源码

​		主要分析Spring IOC、DI、AOP及MVC模块源码，相关模块时序图见时序图文件夹下html。

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



##### Spring Boot启动总结 

1、创建一个StopWatch并执行start方法，这个类主要记录任务的执行时间  
2、配置Headless属性，Headless模式是在缺少显示屏、键盘或者鼠标时候的系统配置  
3、在文件META-INF\spring.factories中获取SpringApplicationRunListener接口的实现类EventPublishingRunListener，主要发布   	    SpringApplicationEvent  
4、把输入参数转成DefaultApplicationArguments类  
5、创建Environment并设置比如环境信息，系统熟悉，输入参数和profile信息  
6、打印Banner信息  
7、创建Application的上下文，根据WebApplicationTyp来创建Context类，如果非web项目则创建  AnnotationConfigApplicationContext，在构造方法中初始化AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner  
8、在文件META-INF\spring.factories中获取SpringBootExceptionReporter接口的实现类FailureAnalyzers  
9、准备application的上下文  
	a.初始化ApplicationContextInitializer  
	b.执行Initializer的contextPrepared方法，发布ApplicationContextInitializedEvent事件  
	c.如果延迟加载，在上下文添加处理器LazyInitializationBeanFactoryPostProcessor  
	d.执行加载方法，BeanDefinitionLoader.load方法，主要初始化了AnnotatedGenericBeanDefinition  
	e.执行Initializer的contextLoaded方法，发布ApplicationContextInitializedEvent事件  
10、刷新上下文（后文会单独分析refresh方法），在这里真正加载bean到容器中。如果是web容器，会在onRefresh方法中创建一个Server并启动。  

#####  IOC容器初始化总结

  1、扫描资源  
  2、加载资源，获取Bean定义  
  3、注册Bean  



## 二、依赖注入

​		**依赖注入**（Dependency Injection，简称**DI**），还有一种方式叫“依赖查找”（Dependency Lookup）。通过控制反转，对象在被创建的时候，由一个调控系统内所有对象的外界实体将其所依赖的对象的引用传递给它。也可以说，依赖被注入到对象中。

​		spring依赖注入发生在getBean()方法，spring获取到Bean的定义后，并未实例化，先缓存Bean的定义，需要使用时，在进行实例化。这样能够解决依赖注入时，循环依赖问题。

    		Bean依赖注入入口AbstractApplicationContext类public void refresh() throws BeansException, IllegalStateException方法，该方法继续调用了protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)方法。
     // Instantiate all remaining (non-lazy-init) singletons.
      finishBeanFactoryInitialization(beanFactory);
    
    进入DefaultListableBeanFactory类public void preInstantiateSingletons() throws BeansException方法
      // Instantiate all remaining (non-lazy-init) singletons.
      beanFactory.preInstantiateSingletons();
    	该方法从Bean的注册表中获取Bean定义，获取class信息,诺bean是FactoryBean，及工厂Bean，需要拼接前缀&。
    
    进入AbstractBeanFactory类public Object getBean(String name) throws BeansException方法
        getBean(beanName);
     继续调用AbstractBeanFactory类方法
     1、protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
    			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException方法
       return doGetBean(name, null, null, false);
     2、protected String transformedBeanName(String name)方法
       //解析别名，及把别名解析为规范名称
       final String beanName = transformedBeanName(name);
     经过一系列校验及判断是单例、原型模式等后，调用创建bean方法createBean。
    
    进入AbstractAutowireCapableBeanFactory类protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    			throws BeanCreationException方法
      return createBean(beanName, mbd, args);
    	继续调用AbstractAutowireCapableBeanFactory类方法
      1、protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException方法
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        先重IOC容器中获取，不存在则创建
        //IOC容器
        private final ConcurrentMap<String, BeanWrapper> factoryBeanInstanceCache = new ConcurrentHashMap<>();
    	2、protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)方法	
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    	3、protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd)方法
        //初始化bean
        return instantiateBean(beanName, mbd);
    
    进入SimpleInstantiationStrategy类public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)方法
      //根据策略就行初始化
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    	//初始化实例包装为一个BeanWrapper
    	BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    
    回到AbstractAutowireCapableBeanFactory类protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    			throws BeanCreationException方法
      继续调用AbstractAutowireCapableBeanFactory类方法
      1、protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)方法
     	  //将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
      	populateBean(beanName, mbd, instanceWrapper);
    		该方法中，如果实例化Bean需要依赖注入其他Bean，那么会先走以下逻辑，源码如下：
        //获取所有的后置处理器getBeanPostProcessors()
        //BeanPostProcessor就是在Bean实例创建之后，在进行populateBean赋值之后，init初始化方法之前进行一次调用，init方法之后进行一		//次调用，这样一来，整个Bean的生命周期，全部掌控在了Spring之下，包括Bean实例创建new Instance()，赋值前后populateBean()，		//初始化前后init()，销毁前后destroy()。
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
    				if (bp instanceof InstantiationAwareBeanPostProcessor) {
    					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
              //当实例化Bean需要依赖注入其他Bean，遍历到bp如果是AutowiredAnnotationBeanPostProcessor，进入AutowiredAnnotationBeanPostProcessor类public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)方法
    					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    					if (pvsToUse == null) {
    						if (filteredPds == null) {
    							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    						}
    						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
    						if (pvsToUse == null) {
    							return;
    						}
    					}
    					pvs = pvsToUse;
    				}
    			}
    
    		实例化Bean需要依赖注入其他Bean时，获取到getBeanPostProcessors()进行遍历，当遍历到bp是AutowiredAnnotationBeanPostProcessor时，进入AutowiredAnnotationBeanPostProcessor类public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)方法进行相关依赖的Bean实例化。
    	  PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    
    进入InjectionMetadata类public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable方法
      metadata.inject(bean, beanName, pvs);
    
    进入InjectedElement类protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable方法
      element.inject(target, beanName, pvs);
    
    回到DefaultListableBeanFactory类，调用public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
    			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException方法
      value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
    	该方法中会注入相关的依赖，源码如下:
    	//把相关的依赖bean注入
      ReflectionUtils.makeAccessible(field);
      field.set(bean, value);
    	继续调用DefaultListableBeanFactory类方法
    	1、public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
    			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException方法
        
    进入DependencyDescriptor类public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory) throws BeansException方法
      instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
    
    然后再次回到AbstractBeanFactory类public Object getBean(String name) throws BeansException方法，该方法后的执行流程已经分析。
      return beanFactory.getBean(beanName);
    
    当相关的依赖都注入，再次回到AbstractAutowireCapableBeanFactory类protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)方法，继续调用protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs)方法
      if (pvs != null) {
        //对属性进行注入，注入参数的方法（注解的Bean的依赖注入除外）
        applyPropertyValues(beanName, mbd, bw, pvs);
      }
    
    回到AbstractAutowireCapableBeanFactory类protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    			throws BeanCreationException方法
      继续调用AbstractAutowireCapableBeanFactory类方法
      1、protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) 方法
      exposedObject = initializeBean(beanName, exposedObject, mbd);
    	方法中继续调用
        //Aware实现处理,为 Bean 实例对象包装相关属性，如名称，类加载器，所属容器等信息
        invokeAwareMethods(beanName, bean);
    		//对BeanPostProcessor后置处理器的postProcessBeforeInitialization回调方法的调用，为Bean实例初始化前做一些处理
    		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    		//调用Bean实例对象初始化的方法，这个初始化方法是在 Spring Bean定义配置文件中通过init-Method属性指定的
    		invokeInitMethods(beanName, wrappedBean, mbd);
    		//对BeanPostProcessor后置处理器的postProcessAfterInitialization回调方法的调用，为Bean实例初始化之后做一些处理
    		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    


## 三、Spring AOP

​		AOP 是 OOP 的延续，是 Aspect Oriented Programming 的缩写，意思是面向切面编程。可以通过预 编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。AOP 设计模式孜孜不倦追求的是调用者和被调用者之间的解耦，AOP 可以说也是这种目标的一种实现。 我们现在做的一些非业务，如:日志、事务、安全等都会写在业务代码中(也即是说，这些非业务类横切 于业务类)，但这些代码往往是重复，复制——粘贴式的代码会给程序的维护带来不便，AOP 就实现了 把这些业务需求与系统需求分开来做。这种解决的方式也称代理机制。

​		Spring 的 AOP 是通过接入 BeanPostProcessor 后置处理器开始的，它是 Spring IOC 容器经常使用到 的一个特性，这个 Bean 后置处理器是一个监听器，可以监听容器触发的 Bean 声明周期事件。后置处 理器向容器注册以后，容器中管理的 Bean 就具备了接收 IOC 容器事件回调的能力。

​		initializeBean()方法为容器产生的 Bean 实例对象添加 BeanPostProcessor 后置处理器，AbstractAutowireCapableBeanFactory 类中，initializeBean()方法实现为容器创建的 Bean 实例对象添加 BeanPostProcessor 后置处理器。该方法就是spring AOP的入口。

```java
进入AbstractAutowireCapableBeanFactory类protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd)方法
  exposedObject = initializeBean(beanName, exposedObject, mbd);
	继续调用AbstractAutowireCapableBeanFactory类方法
	1、public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException 方法
  //调用 BeanPostProcessor 后置处理器实例对象初始化之前的处理方法
  wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	//遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
  public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
  throws BeansException {
    Object result = existingBean;
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
      //调用Bean实例所有的后置处理中的初始化前处理方法，为Bean实例对象在初始化之前做一些自定义的处理操作
      Object current = beanProcessor.postProcessBeforeInitialization(result, beanName); 
      if (current == null) {
        return result;
      }
      result = current; 
    }
    return result;
  }
	2、public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException方法
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    //调用BeanPostProcessor后置处理器实例对象初始化之后的处理方法
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
         throws BeansException {
        Object result = existingBean;
        //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
          //调用Bean实例所有的后置处理中的初始化后处理方法，为Bean实例对象在初始化之后做一些自定义的处理操作
          Object current = beanProcessor.postProcessAfterInitialization(result, beanName); 
          if (current == null) {
              return result;
           }
          result = current; 
        }
    		return result;
    }

BeanPostProcessor是一个接口，其初始化前的操作方法和初始化后的操作方法均委托其实现子类来实现，在Spring中，BeanPostProcessor的实现子类非常的多，分别完成不同的操作，如:AOP面向切面编程的注册通知适配器、Bean对象的数据校验、Bean继承属性、方法的合并等，我们以最简单的AOP切面织入来简单了解其主要的功能。下面我们来分析其中一个创建AOP代理对象的子类AbstractAutoProxyCreator类。该类重写了 postProcessAfterInitialization()方法。
  进入AbstractAutoProxyCreator类public Object postProcessAfterInitialization(@Nullable Object bean, String beanName)方法
  Object current = processor.postProcessAfterInitialization(result, beanName);
	继续调用AbstractAutoProxyCreator类方法
  1、protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey)方法
    return wrapIfNecessary(bean, beanName, cacheKey);

进入AspectJAwareAdvisorAutoProxyCreator类protected boolean shouldSkip(Class<?> beanClass, String beanName)方法
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName))

进入AnnotationAwareAspectJAutoProxyCreator类protected List<Advisor> findCandidateAdvisors()方法
  List<Advisor> candidateAdvisors = findCandidateAdvisors();

进入AbstractAdvisorAutoProxyCreator类protected List<Advisor> findCandidateAdvisors()方法
  List<Advisor> advisors = super.findCandidateAdvisors();

进入BeanFactoryAspectJAdvisorsBuilder类public List<Advisor> buildAspectJAdvisors()方法
  advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());

回到AbstractAutoProxyCreator类public Object postProcessAfterInitialization(@Nullable Object bean, String beanName)方法，继续调用
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

再次进入AbstractAdvisorAutoProxyCreator类，调用protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource)方法
  继续调用AbstractAdvisorAutoProxyCreator类方法
  1、protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName)方法

进入AopUtils类public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) 方法
  return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);

再次回到AbstractAutoProxyCreator类protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey)方法，继续调用protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource)方法方法
  Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
	继续调用AbstractAutoProxyCreator类方法
  1、protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors)方法
    //封装成 Advisor 
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);

进入DefaultAdvisorAdapterRegistry类public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException 方法
  advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));

再次回到AbstractAutoProxyCreator类protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource)方法，继续调用public Object getProxy(@Nullable ClassLoader classLoader)方法
  return proxyFactory.getProxy(getProxyClassLoader());

进入ProxyFactory类public Object getProxy(@Nullable ClassLoader classLoader)方法
  return proxyFactory.getProxy(getProxyClassLoader());

进入ProxyCreatorSupport类protected final synchronized AopProxy createAopProxy()方法
  return createAopProxy().getProxy(classLoader);

进入DefaultAopProxyFactory类public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException方法
  return getAopProxyFactory().createAopProxy(this);
	代理2种实现方式：
    1.jdk动态代理
    2.cglib代理
  spring5及spring boot2.x AOP默认是cglib。此处使用cglib代理。
    
回到ProxyFactory类public Object getProxy(@Nullable ClassLoader classLoader)方法，进入CglibAopProxy类public Object getProxy(@Nullable ClassLoader classLoader)方法
    return createAopProxy().getProxy(classLoader);
	继续调用CglibAopProxy类方法
	1、private Callback[] getCallbacks(Class<?> rootClass) throws Exception方法
    Callback[] callbacks = getCallbacks(rootClass);

进入ObjenesisCglibAopProxy类protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks)方法
  return createProxyClassAndInstance(enhancer, callbacks);

进入Enhancer类public Class createClass()方法
  Class<?> proxyClass = enhancer.createClass();
	继续调用Enhancer类方法
  1、private Object createHelper()方法
    return (Class) createHelper();

进入AbstractClassGenerator类protected Object create(Object key)方法
  Object result = super.create(key);
	继续调用AbstractClassGenerator类方法
  1、public Object get(AbstractClassGenerator gen, boolean useCache)方法			
    Object obj = data.get(this, getUseCache());

进入LoadingCache类public V get(K key)方法
  Object cachedValue = generatedClasses.get(gen);
	继续调用LoadingCache类方法
  1、protected V createEntry(final K key, KK cacheKey, Object v)方法
    return v != null && !(v instanceof FutureTask) ? v : this.createEntry(key, cacheKey, v);
		源码：
    protected V createEntry(final K key, KK cacheKey, Object v) {
        boolean creator = false;
        FutureTask task;
        Object result;
        if (v != null) {
            task = (FutureTask)v;
        } else {
            task = new FutureTask(new Callable<V>() {
                public V call() throws Exception {
                  	//执行线程方法
                    return LoadingCache.this.loader.apply(key);
                }
            });
            result = this.map.putIfAbsent(cacheKey, task);
            if (result == null) {
                creator = true;
              	//启动一个线程，调用run方法，在run方法内部在回调call()方法，即上述源码return LoadingCache.this.loader.apply(key)处
                task.run();
            } else {
                if (!(result instanceof FutureTask)) {
                    return result;
                }

                task = (FutureTask)result;
            }
        }

        try {
            result = task.get();
        } catch (InterruptedException var9) {
            throw new IllegalStateException("Interrupted while loading cache item", var9);
        } catch (ExecutionException var10) {
            Throwable cause = var10.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException)cause;
            }

            throw new IllegalStateException("Unable to load cache item", cause);
        }

        if (creator) {
            this.map.put(cacheKey, result);
        }

        return result;
    }
		上述源码启动一个线程，调用run方法，在run方法内部在回调call()方法，即上述源码return LoadingCache.this.loader.apply(key)处，然后进入AbstractClassGenerator类内部类ClassLoaderData的构造函数public ClassLoaderData(ClassLoader classLoader)。
		
进入AbstractClassGenerator类内部类ClassLoaderData的public ClassLoaderData(ClassLoader classLoader)构造函数
		return LoadingCache.this.loader.apply(key);
		此处调用是初始化时的一个function接口，源码如下：        
    public ClassLoaderData(ClassLoader classLoader) {
        if (classLoader == null) {
          	throw new IllegalArgumentException("classLoader == null is not yet supported");
        } else {
            this.classLoader = new WeakReference(classLoader);
          	//function接口
            Function<AbstractClassGenerator, Object> load = new Function<AbstractClassGenerator, Object>() {
              public Object apply(AbstractClassGenerator gen) {
                Class klass = gen.generate(ClassLoaderData.this);
                return gen.wrapCachedClass(klass);
              }
            };
          	this.generatedClasses = new LoadingCache(GET_KEY, load);
        }
    }
		当在类LoadingCache的protected V createEntry(final K key, KK cacheKey, Object v)方法调用task.run()方法时，会启动一个线程，最终回调到线程的public V call() throws Exception方法，执行LoadingCache.this.loader.apply(key)，因此执行到初始化的function接口。

再次进入Enhancer类，调用protected Class generate(ClassLoaderData data)方法
  	Class klass = gen.generate(ClassLoaderData.this);

再次进入AbstractClassGenerator类，调用protected Class generate(ClassLoaderData data)方法
  	return super.generate(data);

进入ClassLoaderAwareGeneratorStrategy类public byte[] generate(ClassGenerator cg) throws Exception方法
  public byte[] generate(ClassGenerator cg) throws Exception
  
进入DefaultGeneratorStrategy类public byte[] generate(ClassGenerator cg) throws Exception方法
  return super.generate(cg);

再次进入Enhancer类，调用public void generateClass(ClassVisitor v) throws Exception方法
  this.transform(cg).generateClass(cw);
	继续调用Enhancer类方法
	1、private void emitMethods(final ClassEmitter ce, List methods, List actualMethods)方法
    emitMethods(e, methods, actualMethods);

再次CglibAopProxy类，调用public int accept(Method method)方法
  int index = filter.accept(actualMethod);

进入AdvisedSupport类public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass)方法
  List<?> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

进入DefaultAdvisorChainFactory类public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass)方法
  cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
  this, method, targetClass);

进入DefaultAdvisorAdapterRegistry类public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException方法
  MethodInterceptor[] interceptors = registry.getInterceptors(advisor);

cglib执行过程，执行入口在CglibAopProxy内部类DynamicAdvisedInterceptor类public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable方法
  
再次进入AdvisedSupport类	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass)方法
  // 获取当前方法的拦截器链
  List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

进入CglibAopProxy类public Object proceed() throws Throwable方法
  retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();

进入ReflectiveMethodInvocation类public Object proceed() throws Throwable方法
  return super.proceed();
	源码如下：
  public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
    //如果是最后一个拦截器执行改方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

    //获取拦截器，执行方法
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}

进入ExposeInvocationInterceptor类public Object invoke(MethodInvocation mi) throws Throwable方法
  return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	ExposeInvocationInterceptor暴露调用器的拦截器，其就是起了暴露一个调用器作用的拦截器。
    
进入CglibAopProxy内部类CglibMethodInvocation的public Object proceed() throws Throwable方法
  return mi.proceed();

再次回到ReflectiveMethodInvocation类public Object proceed() throws Throwable方法
  //CglibMethodInvocation#proceed
  return super.proceed();

进入AspectJAroundAdvice类public Object invoke(MethodInvocation mi) throws Throwable方法
  //根据不同的拦截器，进入不同的类
  return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);

进入AbstractAspectJAdvice类方法protected Object invokeAdviceMethod(JoinPoint jp, @Nullable JoinPointMatch jpMatch,@Nullable Object returnValue, @Nullable Throwable t) throws Throwable方法
  return invokeAdviceMethod(pjp, jpm, null, null);
	继续调用AbstractAspectJAdvice类方法
  1、return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t))方法
    return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));

进入Method类public Object invoke(Object obj, Object... args)方法
  return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);

进入DelegatingMethodAccessorImpl类public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException方法
  return ma.invoke(obj, args);

进入NativeMethodAccessorImpl类public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException方法
  return this.delegate.invoke(var1, var2);

进入切面方法，Demo是以LogAspect为列，故进入LogAspect类public Object doAround(ProceedingJoinPoint point) throws Throwable方法
  return invoke0(this.method, var1, var2);
  LogAspect类源码如下：
  /**
   * 项目名称：spring-cloud-service
   * 类 名 称：LogAspect
   * 类 描 述：TODO
   * 创建时间：2020/9/12 11:06 下午
   * 创 建 人：chenyouhong
   */
  @Aspect
  @Component
  public class LogAspect {

      @Pointcut("execution(* com.cloud.user.controller.*.*(..))")
      public void aspect() {
      }

      @Before("aspect()")
      public void before(JoinPoint jp) throws Throwable {
          //...
          System.out.println("Before");
      }


      @Around("aspect()")
      public Object doAround(ProceedingJoinPoint point) throws Throwable {
          //...
          System.out.println("==========");
          Object returnValue =  point.proceed(point.getArgs());
          System.out.println("**********");
          //...
          return returnValue;
      }

      @After("aspect()")
      public void after(JoinPoint jp) throws Throwable {
          //...
          System.out.println("after");
      }
  }

	此处会进入切面LogAspect类public Object doAround(ProceedingJoinPoint point) throws Throwable方法
  
进入MethodInvocationProceedingJoinPoint类public Object proceed(Object[] arguments) throws Throwable方法
  Object returnValue =  point.proceed(point.getArgs());

再次回到ReflectiveMethodInvocation类public Object proceed() throws Throwable方法
  //CglibMethodInvocation#proceed
  return super.proceed();

进入MethodBeforeAdviceInterceptor类public Object invoke(MethodInvocation mi) throws Throwable方法
  return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	因为此时的拦截器interceptorOrInterceptionAdvice对应的类型是MethodBeforeAdviceInterceptor。

进入AspectJMethodBeforeAdvice类public void before(Method method, Object[] args, @Nullable Object target) throws Throwable方法
  this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());

再次进入AbstractAspectJAdvice类方法protected Object invokeAdviceMethod(JoinPoint jp, @Nullable JoinPointMatch jpMatch,@Nullable Object returnValue, @Nullable Throwable t) throws Throwable方法
  return invokeAdviceMethod(pjp, jpm, null, null);
	继续调用AbstractAspectJAdvice类方法
  1、return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t))方法
    return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));
	  此处逻辑同上述invokeAdviceMethod方法执行一直，只是执行切面时，进入方法是LogAspect类public void before(JoinPoint jp) throws Throwable方法

再次回到CglibAopProxy内部类CglibMethodInvocation的public Object proceed() throws Throwable方法
  return mi.proceed();
	继续执行，当执行到拦截器，因为此时的拦截器interceptorOrInterceptionAdvice对应的类型是AspectJAfterAdvice。
    
进入AspectJAfterAdvice类public Object invoke(MethodInvocation mi) throws Throwable方法
  //interceptorOrInterceptionAdvice类型是AspectJAfterAdvice
  return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    
当执行到最后一个拦截器时，进入CglibAopProxy内部类CglibMethodInvocation的protected Object invokeJoinpoint() throws Throwable方法
  return invokeJoinpoint();
	
  if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
  }

进入MethodProxy类public Object invoke(Object obj, Object[] args) throws Throwable方法
  return this.methodProxy.invoke(this.target, this.arguments);
	继续调用MethodProxy类方法
  1、init()方法
  
进入被代理类需要执行的方法，demo是UserController类public Mono<String> login()方法
  return fci.f1.invoke(fci.i1, obj, args);
	被代理类源码如下：
  @RestController
  @RequestMapping("/user")
  public class UserController {

      @Autowired
      private TestService testService;

      private List<String> list;

      /**
       * 密码模式  认证.
       *
       * @return
       */
      @RequestMapping("/test")
      public Mono<String> login() {
          //登录 之后生成令牌的数据返回

          testService.test();
          return Mono.just("test");
      }

      /**
       * 密码模式  认证.
       *
       * @return
       */
      @RequestMapping("/test2")
      @PreAuthorize(value = "hasAnyAuthority('test')")
  //    @PreAuthorize("hasPermission('test8988989')")
      public Mono<String> login2() {
          //登录 之后生成令牌的数据返回
          System.out.println("rest2");
          return Mono.just("test2");
      }
  }

再次回到AspectJAfterAdvice类public Object invoke(MethodInvocation mi) throws Throwable方法
  继续之前并未执行完逻辑，继续往后执行，源码如下：
  @Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
      //执行被代理类想要执行的方法
			return mi.proceed();
		}
		finally {
      //此处逻辑同上述invokeAdviceMethod方法执行一直，只是执行切面时，进入方法是LogAspect类public void after(JoinPoint jp) throws Throwable方法
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```

