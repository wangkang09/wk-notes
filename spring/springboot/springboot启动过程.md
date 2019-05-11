[TOC]

# 0 SpringApplication.run()

```java
SpringApplication.run(SpringbootLearnApplication.class, args);//先初始化SpringApplication类，再调用它的run方法
//第一步：初始化SpringApplication类，primarySources对应SpringbootLearnApplication.class
public SpringApplication(ResourceLoader resourceLoader, Class... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;//banner
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    //判断应用的类型，枚举类型：NONE,SERVLET,REACTIVE;
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //这里就是读取所有的jar包中的META-INF/下的spring.factories文件中
    //key为ApplicationContextInitializer的值，生成一个List<String>
    //如下图initializers所示
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //同理查找key为ApplicationListener
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    //找到main函数对应的类
    this.mainApplicationClass = this.deduceMainApplicationClass();
    //# Initializers
	//org.springframework.context.ApplicationContextInitializer=\
	//org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
	//org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
    //# Application Listeners
    //org.springframework.context.ApplicationListener=\
	//org.springframework.boot.autoconfigure.BackgroundPreinitializer
}

//第二步：调用run方法，这个args就是main函数输入的参数
public ConfigurableApplicationContext run(String... args) {
    //记录应用启动时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
    //1 设置使用Headless，对于只有远程登录使用的服务器来说这样性能要好一些
    this.configureHeadlessProperty();
    //2 查找spring.factory中的SpringApplicationRunListeners，并创建所有的监听器
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    //3 通知所有监听器启动
    listeners.starting();

    Collection exceptionReporters;
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    //4 查找配置文件(包括：通过main入口参数args，解析commandLines！)
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    //获取System中spring.beaninfo.ignore属性，不知道干啥的
    this.configureIgnoreBeanInfo(environment);
    //打印Spring Banner，可在resource下自定义了一个banner.txt文件
    //5 打印Banner
    Banner printedBanner = this.printBanner(environment);
    //6 根据应用的类型，创建Spring容器
    context = this.createApplicationContext();
    //7 从Spring.factories中查找这个类，用于异常路径的报错（应该就是如果异常是，可以带人context信息）
    exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
    
    //主要是调用所有初始化类的initialize方法，准备上下文资源
    //1. 将准备好的Environment设置给ApplicationContext
    //2. 遍历调用所有的ApplicationContextInitializer的initialize()方法，对已经创建的ApplicatinContext做进一步处理
    //3. 调用SpringApplicationRunListener的ContextPrepared()方法，通知所有监听者：ApplicationContext已经准备完毕
    //4. 调用SpringApplicationRunListener的contextLoaded()方法，通知所有监听者：ApplicationContext装载完毕
    //prepareContext()方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联
    //8 准备上下文环境
    this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    //9 加载业务bean，启动tomcat，发布对应事件
    this.refreshContext(context);
    //10 执行 Spring 容器的初始化的后置逻辑，默认实现为空
    this.afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
        (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
    }
    //11 通知监听者 Spring 容器启动完成
    listeners.started(context);
    //12 调用 ApplicationRunner 或者 CommandLineRunner 的运行方法
    this.callRunners(context, applicationArguments);
    //13 通知监听者，应用在运行
    listeners.running(context);
    return context;
}
```
# 1 configureHeadlessProperty()：远程登录使用的服务器配置

```java
//SpringApplication：设置使用Headless，对于只有远程登录使用的服务器来说这样性能要好一些
private void configureHeadlessProperty() {
    System.setProperty("java.awt.headless", System.getProperty("java.awt.headless", Boolean.toString(this.headless)));
}
```

# 2 getRunListeners()：获取spring.factory中的SpringApplicationRunListeners，并创建所有的监听器

```java
//SpringApplication：查找spring.factory中的SpringApplicationRunListeners，并创建所有的监听器
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class[]{SpringApplication.class, String[].class};
    return new SpringApplicationRunListeners(logger, this.getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

# 3 listeners.starting()：通知所有Run监听器开始运行

```java
//SpringApplicationRunListeners：
public void starting() {
    Iterator var1 = this.listeners.iterator();
    while(var1.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var1.next();
        listener.starting();
    }
}
```

# 4 prepareEnvironment()：关键是获取配置文件

```java
//SpringApplication：
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
    //根据应用类型，创建应用环境：如得到系统的参数和JVM等参数，等
    ConfigurableEnvironment environment = this.getOrCreateEnvironment();
    //4.1 创建环境并配置
    this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
    //4.2 Run监听器准备配置环境（重要-获取所有配置文件）
    listeners.environmentPrepared((ConfigurableEnvironment)environment);
    //4.3 不知道干啥
    this.bindToSpringApplication((ConfigurableEnvironment)environment);
    if (!this.isCustomEnvironment) {
        environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
    }
	//加一个ConfigurationPropertySources
    ConfigurationPropertySources.attach((Environment)environment);
    return (ConfigurableEnvironment)environment;
}
```
## 4.1 configureEnvironment()：配置defaultProperties和commandLineArgs属性

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService)conversionService);
    }
	//4.1.1 如果有defaultProperties和commandLineArgs属性则配置
    //defaultProperties还真不知道在哪设值，因为new完SpringApplicaiton类之后，里面就开始这一步了，
    //不知道在哪一步调用springApplicaiton.setDefaultProperties(...)；
    this.configurePropertySources(environment, args);
    //4.1.2 获取spring.profiles.active
    this.configureProfiles(environment, args);
}
```

### 4.1.1 如果有defaultProperties和commandLineArgs属性则配置

```java
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    if (this.addCommandLineProperties && args.length > 0) {
        String name = "commandLineArgs";
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        } else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
```

### 4.1.2 获取spring.profiles.active—只从4个路径中查询

```java
//这个profiles参数目前仅仅在servletConfigInitParams、servletContextInitParams、systemProperties、systemEnvironment中查询，可以设置用户变量或环境变量即可
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    environment.getActiveProfiles();
    Set<String> profiles = new LinkedHashSet(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```

## 4.2 listeners.environmentPrepared(）：Run监听器准备配置环境（重要-获取所有配置文件）

- 运行监听器中的环境配置类，重要的是ConfigFileApplicationListener，来读取配置文件
- 查找spring.factories中EnvironmentPostProcessor的类
- 关键，查找、读取配置文件默认在classpath:/,classpath:/config/,file:./,file:./config/
- 如果设置了spring.config.location，则指定扫描的配置文件路径，commaDelimitedListToStringArray（用逗号隔开可以表示多个路径）
- 如果配置了spring.config.additional-location默认路径加上它
- 如果设置了spring.config.location，则指定扫描的配置文件路径，commaDelimitedListToStringArray（用逗号隔开可以表示多个路径）

```java
//SpringApplicationRunListeners：
//这里之后，environment中的PropertySources中已经包含了所有的配置文件了
//这里的listeners就一个：EventPublishingRunListener
public void environmentPrepared(ConfigurableEnvironment environment) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.environmentPrepared(environment);
    }
}
//EventPublishingRunListener：
public void environmentPrepared(ConfigurableEnvironment environment) {
    //4.2.1
    this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
}
```

### 4.2.1 通过`SimpleApplicationEventMulticaster`类来多播调用7个监听事件

- 遍历调用7个监听器

```java
//SimpleApplicationEventMulticaster：
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    //一般共有7个listners：ConfigFileAppLst、AnsiOutputAppLst、LoggingAppLst、ClasspathLoggingAppLst、BackgroundPreinitializer、DelegatingAppLst、FileEncodingAppLst
    Iterator var4 = this.getApplicationListeners(event, type).iterator();
    while(var4.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var4.next();
        Executor executor = this.getTaskExecutor();
        if (executor != null) {
            executor.execute(() -> {
                this.invokeListener(listener, event);
            });
        } else {
            //执行方法
            this.invokeListener(listener, event);
        }
    }
}
//ConfigFileApplicationListener：这个类很重要，操作读取各种配置文件的类
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
        //4.2.1.1 因为是Environment，所以走这
        this.onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent)event);
    }
    if (event instanceof ApplicationPreparedEvent) {
        this.onApplicationPreparedEvent(event);
    }
}
```

#### 4.2.1.1  调用7个监听中的ConfigFileApplicationListener类来读取配置文件

- 查找spring.factories中EnvironmentPostProcessor的类，这里有3个
- 加上本身就是4个，在按Order排序
- 遍历处理每个processor

```java
//ConfigFileApplicationListener：
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
    //查找spring.factories中EnvironmentPostProcessor的类，这里有3个
    //SystemEnvironmentPropertySourceEnvironmentPostProcessor：读取设置系统参数，Java_HOME等，应该有bootStrapProperties里的参数
    //SpringApplicationJsonEnvironmentPostProcessor：propertySources没有Json
    //CloudFoundryVvcapEnvironmentPostProcessor：没有
    List<EnvironmentPostProcessor> postProcessors = this.loadPostProcessors();
    //加上ConfigFileApplicationListener：关键是这个！！！
    postProcessors.add(this);
    //按照AnnotationAwareOrderComparator类排序
    AnnotationAwareOrderComparator.sort(postProcessors);
    Iterator var3 = postProcessors.iterator();
    while(var3.hasNext()) {
        EnvironmentPostProcessor postProcessor = (EnvironmentPostProcessor)var3.next();
        //按照每个processor做处理
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }
}
```

##### 4.2.1.1.1 调用SystemEnvironmentPropertySourceEnvironmentPostProcessor处理器，处理systemEnvironment

```java
//SystemEnvironmentPropertySourceEnvironmentPostProcessor：
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    String sourceName = "systemEnvironment";
    //本机的系统参数
    PropertySource<?> propertySource = environment.getPropertySources().get(sourceName);
    if (propertySource != null) {
        //没变，包装了下吧
        this.replacePropertySource(environment, sourceName, propertySource);
    }
}
```

##### 4.2.1.1.2 调用SpringApplicationJsonEnvironmentPostProcessor处理器，处理JsonProperty

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    MutablePropertySources propertySources = environment.getPropertySources();
    //在这4个配置文件中找spring.application.json和SPRING_APPLICATION_JSON的属性，用JsonParser来解析属性对应的值，最后添加到this.JsonPropertySource中
  propertySources.stream().map(SpringApplicationJsonEnvironmentPostProcessor.JsonPropertyValue::get).filter(Objects::nonNull).findFirst().ifPresent((v) -> {
        this.processJson(environment, v);
    });
}
private void processJson(ConfigurableEnvironment environment, SpringApplicationJsonEnvironmentPostProcessor.JsonPropertyValue propertyValue) {
    JsonParser parser = JsonParserFactory.getJsonParser();
    Map<String, Object> map = parser.parseMap(propertyValue.getJson());
    if (!map.isEmpty()) {
        this.addJsonPropertySource(environment, new SpringApplicationJsonEnvironmentPostProcessor.JsonPropertySource(propertyValue, this.flatten(map)));
    }

}
```

##### 4.2.1.1.3 调用CloudFoundryVcapEnvironmentPostProcessor处理器，设置Vcap配置

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    //环境是否包含：environment.containsProperty("VCAP_APPLICATION") || environment.containsProperty("VCAP_SERVICES");这两个key
    if (CloudPlatform.CLOUD_FOUNDRY.isActive(environment)) {
        //如果包含
        Properties properties = new Properties();
        JsonParser jsonParser = JsonParserFactory.getJsonParser();
        this.addWithPrefix(properties, this.getPropertiesFromApplication(environment, jsonParser), "vcap.application.");
        this.addWithPrefix(properties, this.getPropertiesFromServices(environment, jsonParser), "vcap.services.");
        MutablePropertySources propertySources = environment.getPropertySources();
        if (propertySources.contains("commandLineArgs")) {
            propertySources.addAfter("commandLineArgs", new PropertiesPropertySource("vcap", properties));
        } else {
            propertySources.addFirst(new PropertiesPropertySource("vcap", properties));
        }
    }
}
```

##### 4.2.1.1.4 调用ConfigFileApplicationListener加载配置文件——重要

- 读取配置文件默认的路径：classpath:/,classpath:/config/,file:./,file:./config/
- 如果设置了spring.config.location，则指定扫描的配置文件路径，commaDelimitedListToStringArray（用逗号隔开可以表示多个路径）
- 如果配置了spring.config.additional-location默认路径加上它

```java
//ConfigFileApplicationListener，
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    this.addPropertySources(environment, application.getResourceLoader());
}
//ConfigFileApplicationListener：
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    RandomValuePropertySource.addToEnvironment(environment);
    (new ConfigFileApplicationListener.Loader(environment, resourceLoader)).load();
}
```

###### 4.2.1.1.4.1 初始化文件加载器——包括placeholder、resourceLoader、propertyLoader

- placeholder、resourceLoader是默认的
- propertyLoader在spring.factories中查找

```java
//新建文件加载器
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    this.logger = ConfigFileApplicationListener.this.logger;
    this.loadDocumentsCache = new HashMap();
    this.environment = environment;
    //新建各种解析器和加载器，并将环境设置进去
    this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
    this.resourceLoader = (ResourceLoader)(resourceLoader != null ? resourceLoader : new DefaultResourceLoader());
    //配置文件加载器，各个包都有自己的加载器来加载自己的文件，接口实现分离！
    //基本有两种，PropertiesPropertySourceLoader和YamlPropertySourceLoader！
    this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class, this.getClass().getClassLoader());
}
```

###### [4.2.1.1.4.2 开始加载配置文件](./4.2.1.1.4.2 加载配置文件.md)

#### 4.2.1.2  调用7个监听中的AnsiOutputApplicationListener类

```java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment environment = event.getEnvironment();
    Binder.get(environment).bind("spring.output.ansi.enabled", Enabled.class).ifBound(AnsiOutput::setEnabled);
    AnsiOutput.setConsoleAvailable((Boolean)environment.getProperty("spring.output.ansi.console-available", Boolean.class));
}
```

#### 4.2.1.3  调用7个监听中的LoggingApplicationListener类—处理日志基本

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationStartingEvent) {
        this.onApplicationStartingEvent((ApplicationStartingEvent)event);
    } else if (event instanceof ApplicationEnvironmentPreparedEvent) {
        this.onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent)event);
    } else if (event instanceof ApplicationPreparedEvent) {
        this.onApplicationPreparedEvent((ApplicationPreparedEvent)event);
    } else if (event instanceof ContextClosedEvent && ((ContextClosedEvent)event).getApplicationContext().getParent() == null) {
        this.onContextClosedEvent();
    } else if (event instanceof ApplicationFailedEvent) {
        this.onApplicationFailedEvent();
    }

}
```

#### 4.2.1.4  调用7个监听中的ClasspathLoggingApplicationListener类

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (logger.isDebugEnabled()) {
        if (event instanceof ApplicationEnvironmentPreparedEvent) {
            logger.debug("Application started with classpath: " + this.getClasspath());
        } else if (event instanceof ApplicationFailedEvent) {
            logger.debug("Application failed to start with classpath: " + this.getClasspath());
        }
    }
}
```

#### 4.2.1.5  调用7个监听中的BackgroundPreinitializer类

```java
public void onApplicationEvent(SpringApplicationEvent event) {
    if (!Boolean.getBoolean("spring.backgroundpreinitializer.ignore") && event instanceof ApplicationStartingEvent && preinitializationStarted.compareAndSet(false, true)) {
        this.performPreinitialization();
    }
    if ((event instanceof ApplicationReadyEvent || event instanceof ApplicationFailedEvent) && preinitializationStarted.get()) {
        try {
            preinitializationComplete.await();
        } catch (InterruptedException var3) {
            Thread.currentThread().interrupt();
        }
    }
}
```

#### 4.2.1.6  调用7个监听中的DelegatingApplicationListener类

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
        List<ApplicationListener<ApplicationEvent>> delegates = this.getListeners(((ApplicationEnvironmentPreparedEvent)event).getEnvironment());
        if (delegates.isEmpty()) {
            return;//走这里了
        }
        this.multicaster = new SimpleApplicationEventMulticaster();
        Iterator var3 = delegates.iterator();

        while(var3.hasNext()) {
            ApplicationListener<ApplicationEvent> listener = (ApplicationListener)var3.next();
            this.multicaster.addApplicationListener(listener);
        }
    }
    if (this.multicaster != null) {
        this.multicaster.multicastEvent(event);
    }
}
```

#### 4.2.1.7  调用7个监听中的FileEncodingApplicationListener类

- 判断编解码是否强制

```java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment environment = event.getEnvironment();
    if (environment.containsProperty("spring.mandatory-file-encoding")) {
        String encoding = System.getProperty("file.encoding");
        String desired = environment.getProperty("spring.mandatory-file-encoding");
        if (encoding != null && !desired.equalsIgnoreCase(encoding)) {
            logger.error("System property 'file.encoding' is currently '" + encoding + "'. It should be '" + desired + "' (as defined in 'spring.mandatoryFileEncoding').");
            logger.error("Environment variable LANG is '" + System.getenv("LANG") + "'. You could use a locale setting that matches encoding='" + desired + "'.");
            logger.error("Environment variable LC_ALL is '" + System.getenv("LC_ALL") + "'. You could use a locale setting that matches encoding='" + desired + "'.");
            throw new IllegalStateException("The Java Virtual Machine has not been configured to use the desired default character encoding (" + desired + ").");
        }
    }
}
```

## 4.3 bindToSpringApplication：不知道干啥

# 5 打印Banner

```java
//SpringApplication：
private Banner printBanner(ConfigurableEnvironment environment) {
    if (this.bannerMode == Mode.OFF) {
        return null;
    } else {
        ResourceLoader resourceLoader = this.resourceLoader != null ? this.resourceLoader : new DefaultResourceLoader(this.getClassLoader());
        SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter((ResourceLoader)resourceLoader, this.banner);
        return this.bannerMode == Mode.LOG ? bannerPrinter.print(environment, this.mainApplicationClass, logger) : bannerPrinter.print(environment, this.mainApplicationClass, System.out);
    }
}
```

# 6 createApplicationContext()：根据应用类型创建应用上下文

```java
//SpringApplication：
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch(this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName("org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext");
                break;
		   ...
    }
    return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
}
```

# 7 从Spring.factories中查找创建异常报错类

```java
////SpringApplication：将上下文传入各个异常处理类，使得异常能在任何时候捕获上下文
exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
```

# 8 prepareContext()：准备上下文

- 调用所有ApplicationContextInitializer的initializer方法
- 通知所有的SpringApplicationRunListener监听者，Spring容器准备好了
- 注册几个单例类，不知道干啥
- 设置BeanDefinition是否能重复定义（后面覆盖前面），默认false
- 通知监听者context装载完成

```java
//SpringApplication：
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    //将环境设置到上下文中
    context.setEnvironment(environment);
    //如果beanNameGenerator不为null，注册beanName生成类
    //如果resourceLoader不为null，将它设置到context中
    this.postProcessApplicationContext(context);
    //调用所有ApplicatinContextInitializer的initialize方法
    //例如加载环境变量信息、注册EmbeddedServletContainerInitializedEvent的监听、注册CachingMetadataReaderFactoryPostProcessor等
    //8.1 调用所有ApplicationContextInitializer的initializer方法
    this.applyInitializers(context);
    //8.2 通知所有的SpringApplicationRunListener监听者，Spring容器准备好了
    listeners.contextPrepared(context);
    
    //根据配置情况打印了启动或运行以及profile是否配置的日志
    //线程不安全
    if (this.logStartupInfo) {//logStartupInfo在SpringApplication类实例化的时候在构造函数中置true了
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
    }

    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    //注册几个单例类，不知道干啥
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    //设置BeanDefinition是否能重复定义（后面覆盖前面），默认false
    if (beanFactory instanceof DefaultListableBeanFactory) {
    ((DefaultListableBeanFactory)beanFactory).setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
	//这个即是main对应的类
    Set<Object> sources = this.getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    //和sources有关
    this.load(context, sources.toArray(new Object[0]));
    //8.3 通知监听者context装载完成
    listeners.contextLoaded(context);
}
```
## 8.1 applyInitializers()：调用所有ApplicationContextInitializer的initializer方法

```java
//SpringApplication：
protected void applyInitializers(ConfigurableApplicationContext context) {
    Iterator var2 = this.getInitializers().iterator();
    while(var2.hasNext()) {
        ApplicationContextInitializer initializer = (ApplicationContextInitializer)var2.next();
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```
### 8.1.1 调用DelegatingApplicationContextInitializer

```java
public void initialize(ConfigurableApplicationContext context) {
    ConfigurableEnvironment environment = context.getEnvironment();
    List<Class<?>> initializerClasses = this.getInitializerClasses(environment);
    //如果环境中有context.initializer.classes属性，应用
    if (!initializerClasses.isEmpty()) {
        this.applyInitializerClasses(context, initializerClasses);
    }

}
```

### 8.1.2 调用SharedMetadataReaderFactoryContextInitializer

```java
//SharedMetadataReaderFactoryContextInitializer:加入一个beanProcessor
public void initialize(ConfigurableApplicationContext applicationContext) {
    applicationContext.addBeanFactoryPostProcessor(new SharedMetadataReaderFactoryContextInitializer.CachingMetadataReaderFactoryPostProcessor());
}
```

### 8.1.3 调用ContextIdApplicationContextInitializer

```java
//注册ContextIdApplicationContextInitializer
public void initialize(ConfigurableApplicationContext applicationContext) {
    ContextIdApplicationContextInitializer.ContextId contextId = this.getContextId(applicationContext);
    applicationContext.setId(contextId.getId());
    applicationContext.getBeanFactory().registerSingleton(ContextIdApplicationContextInitializer.ContextId.class.getName(), contextId);
}
```

### 8.1.4 调用ConfigurationWarningsApplicationContextInitializer

```java
//注册ConfigurationWarningsPostProcessor
public void initialize(ConfigurableApplicationContext context) {
    context.addBeanFactoryPostProcessor(new ConfigurationWarningsApplicationContextInitializer.ConfigurationWarningsPostProcessor(this.getChecks()));
}
```

### 8.1.5 调用ServerPortInfoApplicationContextInitializer

```java
//向应用上下文添加本监听器
public void initialize(ConfigurableApplicationContext applicationContext) {
    applicationContext.addApplicationListener(this);
}
```

### 8.1.6 调用ConditionEvaluationReportLoggingListener

```java
//注册另一个监听器
public void initialize(ConfigurableApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    applicationContext.addApplicationListener(new ConditionEvaluationReportLoggingListener.ConditionEvaluationReportListener());
    if (applicationContext instanceof GenericApplicationContext) {
        this.report = ConditionEvaluationReport.get(this.applicationContext.getBeanFactory());
    }

}
```

## 8.2 listeners.contextPrepared()：通知所有的Run监听者，Spring容器准备好了

```java
//SpringApplication：
public void contextPrepared(ConfigurableApplicationContext context) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.contextPrepared(context);
    }

}
```

## 8.3 listeners.contextLoaded()：通知所有的Run监听者，Spring容器加载好了

```java
//SpringApplicationRunListeners：
public void contextLoaded(ConfigurableApplicationContext context) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.contextLoaded(context);
    }

}
```

# 9 refreshContext(context)—准备业务上下文

```java
//SpringApplication：
private void refreshContext(ConfigurableApplicationContext context) {
	this.refresh(context);
    //9.14 注册shutdown后的逻辑
    context.registerShutdownHook();
}
//ServletWebServerApplicationContext中的方法
public final void refresh() throws BeansException, IllegalStateException {
    super.refresh();
}
//AbstractApplicationContext类中的方法
public void refresh() throws BeansException, IllegalStateException {
    Object var1 = this.startupShutdownMonitor;
    synchronized(this.startupShutdownMonitor) {
        //9.1 初始化一些配置属性，验证配置文件
        this.prepareRefresh();
        //9.2 简单的获取beanFactory
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        //9.3 将context中的一些属性设置到beanFactory中
        this.prepareBeanFactory(beanFactory);
	    //9.4 注册Scope相关的类
        this.postProcessBeanFactory(beanFactory);
        //9.5 和业务类、自动配置类有关
        this.invokeBeanFactoryPostProcessors(beanFactory);
        //9.6 分类、排序、注册（注入）所有的BeanPostProcessors
        this.registerBeanPostProcessors(beanFactory);
        //9.7 国际化
        this.initMessageSource();
        //9.8 初始化多播事件
        this.initApplicationEventMulticaster();
        //9.9 主要创建并初始化容器
        this.onRefresh();
        //9.10 向多播事件注册监听者
        this.registerListeners();
        //9.11 主要是初始化单例
        this.finishBeanFactoryInitialization(beanFactory);
        //9.12 主要是开启Web容器
        this.finishRefresh();
	    //9.13 清除一些缓存
        this.resetCommonCaches();
    }
}
```

## 9.1 prepareRefresh()：初始化一些配置属性，验证配置文件

- 初始化servletContextInitParams，servletConfigInitParams如果有的话
- 验证所有的requiredProperties是否都存在，否则报错

```java
//AnnotationConfigServletWebServerApplicationContext
protected void prepareRefresh() {
    this.scanner.clearCache();
    super.prepareRefresh();
}
//AbstractApplicationContext
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);
    if (this.logger.isDebugEnabled()) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Refreshing " + this);
        } else {
            this.logger.debug("Refreshing " + this.getDisplayName());
        }
    }
	//9.1.1 初始化servletContextInitParams，servletConfigInitParams如果有的话
    this.initPropertySources();
	//9.1.2 验证所有的requiredProperties是否都存在，否则报错
    this.getEnvironment().validateRequiredProperties();
    this.earlyApplicationEvents = new LinkedHashSet();

}
```

### 9.1.1 initPropertySources()：初始化servletContextInitParams，servletConfigInitParams如果有的话

- 初始化servletContextInitParams，servletConfigInitParams如果有的话

```java
//初始化servletContextInitParams，servletConfigInitParams如果有的话
//GenericWebApplicationContext
protected void initPropertySources() {
    ConfigurableEnvironment env = this.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        //转下面的initPropertySources
        ((ConfigurableWebEnvironment)env).initPropertySources(this.servletContext, (ServletConfig)null);
    }
}
//StandardServletEnvironment
public void initPropertySources(@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
    WebApplicationContextUtils.initServletPropertySources(this.getPropertySources(), servletContext, servletConfig);
}
//WebApplicationContextUtils
//初始化servletContextInitParams，servletConfigInitParams如果有的话
public static void initServletPropertySources(MutablePropertySources sources, @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
    Assert.notNull(sources, "'propertySources' must not be null");
    String name = "servletContextInitParams";
    if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        sources.replace(name, new ServletContextPropertySource(name, servletContext));
    }

    name = "servletConfigInitParams";
    if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
    }
}
```

### 9.1.2 validateRequiredProperties()：验证所有的requiredProperties是否都存在，否则报错

```java
//AbstractPropertyResolver
public void validateRequiredProperties() {
    MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
    Iterator var2 = this.requiredProperties.iterator();//一般都是0
    while(var2.hasNext()) {
        String key = (String)var2.next();
        if (this.getProperty(key) == null) {
            ex.addMissingRequiredProperty(key);
        }
    }
    if (!ex.getMissingRequiredProperties().isEmpty()) {
        throw ex;
    }
}
```

## 9.2 obtainFreshBeanFactory()：简单获取BeanFactory

```java
//AbstractApplicationContext：简单获取BeanFactory
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    this.refreshBeanFactory();
    //这里获取的BeanFactory里的属性没有业务相关的类，全是引擎需要的类
    return this.getBeanFactory();
}
```

## 9.3 prepareBeanFactory()：将context中的一些属性注册到beanFactory中

```java
//AbstractApplicationContext
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.setBeanClassLoader(this.getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    if (beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
    if (!beanFactory.containsLocalBean("environment")) {
        beanFactory.registerSingleton("environment", this.getEnvironment());
    }
    if (!beanFactory.containsLocalBean("systemProperties")) {
        beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean("systemEnvironment")) {
        beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
    }
}
```

## 9.4 postProcessBeanFactory()：并向beanFactory注册几个Scope类

```java
//AnnotationConfigServletWebServerApplicationContext
//这里的basePackages和annotatedClasses是在哪了设置的？
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    super.postProcessBeanFactory(beanFactory);
    //一般都没有吧，到这里
    //如果有basePackages，则扫描包，将包中指定的bean注入
    if (this.basePackages != null && this.basePackages.length > 0) {
        this.scanner.scan(this.basePackages);
    }
    //如果有注解，则注册有注解的bean
    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}
//ServletWebServerApplicationContext：super到这，添加Web相关Processor
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    //添加beanFactory的resolvableDependencies属性
    this.registerWebApplicationScopes();
}
//ServletWebServerApplicationContext：
private void registerWebApplicationScopes() {
    //这里设置的scopes的map属性-----00
    ServletWebServerApplicationContext.ExistingWebApplicationScopes existingScopes = new ServletWebServerApplicationContext.ExistingWebApplicationScopes(this.getBeanFactory());
    //向beanFactory中注册几个作用域：ServletRequest、ServletResponse、HttpSession、WebRequest、RequestScope、SessionScope
    WebApplicationContextUtils.registerWebApplicationScopes(this.getBeanFactory());
    //注册scopes这个map中的scope到beanfactory中，这个map-----00
    //Scope scope = beanFactory.getRegisteredScope(scopeName);-----00
    existingScopes.restore();
}
```

## [**9.5 invokeBeanFactoryPostProcessors()：解析配置文件重要—点击链接到目的地，将所有的bean注册成beanDefinition以便后面的初始化**！](./9.5 invokeBeanFactoryPostProcessors()解析.md)

## 9.6 registerBeanPostProcessors()：分类、排序、注册（注入beanFactory中）所有的BeanPostProcessors

- 就是将所有的后置处理器排序，并注入到beanFactory中

```java
//组成一些后置处理器
public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    //获取后置处理器的名字
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    //加入一个新的后置处理器
    beanFactory.addBeanPostProcessor(new PostProcessorRegistrationDelegate.BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
    //用于分类后置处理器
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList();
    List<String> orderedPostProcessorNames = new ArrayList();
    List<String> nonOrderedPostProcessorNames = new ArrayList();
    
    //遍历后置处理器并开始分类
    String[] var8 = postProcessorNames;
    int var9 = postProcessorNames.length;
    String ppName;
    BeanPostProcessor pp;
    for(int var10 = 0; var10 < var9; ++var10) {
        ppName = var8[var10];
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            //初始化后置处理器，并加入
            pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                //第一次加入
                internalPostProcessors.add(pp);
            }
        } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        } else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }
	//a 排序并调用priorityOrderedPostProcessors:3个
    //AutowiredAnnotation~/CommonAnnotation~/ConfigurationPropertiesBinding~
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)priorityOrderedPostProcessors);
    
    //初始化得到orderedPostProcessors：4个
    //methodValidation~/dataSourceInitializer~/persistenceException~/AutoProxy~
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList();
    Iterator var14 = orderedPostProcessorNames.iterator();
    while(var14.hasNext()) {
        String ppName = (String)var14.next();
        //初始化
        BeanPostProcessor pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            //第二次加入
            internalPostProcessors.add(pp);
        }
    }
	//b 排序并调用orderedPostProcessors	
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)orderedPostProcessors);
    
    //初始化得到nonOrderedPostProcessors
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList();
    Iterator var17 = nonOrderedPostProcessorNames.iterator();
    while(var17.hasNext()) {
        ppName = (String)var17.next();
        pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            //第三次加入
            internalPostProcessors.add(pp);
        }
    }
    //c 调用nonOrderedPostProcessors
    registerBeanPostProcessors(beanFactory, (List)nonOrderedPostProcessors);
    
	//d 排序并调用internalPostProcessors	
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)internalPostProcessors);
    
    //加入另一个后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## 9.7 initMessageSource()：国际化

```java
//主要注册了下面这个类，说是用来国际化的
//AbstractApplicationContext
beanFactory.registerSingleton("messageSource", this.messageSource);
```

## 9.8 initApplicationEventMulticaster()：注册多播事件

```java
//主要就是注册下面这个类，事件多播？
//AbstractApplicationContext
this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
beanFactory.registerSingleton("applicationEventMulticaster", this.applicationEventMulticaster);
```

## 9.9 onRefresh()：主要是创建并初始化Web容器

```java
ServletWebServerApplicationContext
protected void onRefresh() {
    super.onRefresh();
	this.createWebServer();
}
//GenericWebApplicationContext:super.
protected void onRefresh() {
	//始化主题
    this.themeSource = UiApplicationContextUtils.initThemeSource(this);
}

private void createWebServer() {
	WebServer webServer = this.webServer;
	ServletContext servletContext = this.getServletContext();
	if (webServer == null && servletContext == null) {
		//获取ServletWebServerFactory实例
		//内部：String[] beanNames = this.getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);获取容器，根据引入的包的不同，可以创建tomcat或jetty等web容器
		//初始化一个ServletWebServerFactory
		ServletWebServerFactory factory = this.getWebServerFactory();
		//如果是tomcat就到tomcatSevletWebServerFactory中，同理jetty等
		this.webServer = factory.getWebServer(new ServletContextInitializer[]						{this.getSelfInitializer()});
	} else if (servletContext != null) {//如果不为null，从这里创建容器，一般不走这
		this.getSelfInitializer().onStartup(servletContext);
	}
	//1初始化ServletPropertySources
	//servletContextInitParams、servletConfigInitParams有关
	this.initPropertySources();
}

public WebServer getWebServer(ServletContextInitializer... initializers) {
	Tomcat tomcat = new Tomcat();//创建实例
	File baseDir = this.baseDirectory != null ? this.baseDirectory :      		this.createTempDir("tomcat");//创建目录
	tomcat.setBaseDir(baseDir.getAbsolutePath());//设置tomcat目录
	Connector connector = new Connector(this.protocol);//创建connector
	tomcat.getService().addConnector(connector);//给Service添加connector
	this.customizeConnector(connector);//自定义connector
	tomcat.setConnector(connector);
	tomcat.getHost().setAutoDeploy(false);
	this.configureEngine(tomcat.getEngine());
	Iterator var5 = this.additionalTomcatConnectors.iterator();
	while(var5.hasNext()) {
		Connector additionalConnector = (Connector)var5.next();
		tomcat.getService().addConnector(additionalConnector);
	}
	this.prepareContext(tomcat.getHost(), initializers);
	return this.getTomcatWebServer(tomcat);
}
//GenericWebApplicationContext
//初始化ServletSources
//1
protected void initPropertySources() {
    ConfigurableEnvironment env = this.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment)env).initPropertySources(this.servletContext, (ServletConfig)null);
    }
}
//StandardServletEnvironment
public void initPropertySources(@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
        WebApplicationContextUtils.initServletPropertySources(this.getPropertySources(), servletContext, servletConfig);
}
    
public static void initServletPropertySources(MutablePropertySources sources, @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
    Assert.notNull(sources, "'propertySources' must not be null");
    String name = "servletContextInitParams";
    if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        sources.replace(name, new ServletContextPropertySource(name, servletContext));
    }

    name = "servletConfigInitParams";
    if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
    }

}
```

## 9.10 registerListeners()：向多播事件中注册几个监听者

```java
//主要就做了下面3件事情
this.getApplicationEventMulticaster().addApplicationListener(listener);
this.getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
//需要发布的，这里为0
this.getApplicationEventMulticaster().multicastEvent(earlyEvent);
```

## **9.11 finishBeanFactoryInitialization()：初始化非懒加载beanDefinition**

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    if (beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
    beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
    }
    //不进去
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver((strVal) -> {
            return this.getEnvironment().resolvePlaceholders(strVal);
        });
    }
    //没有
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    String[] var3 = weaverAwareNames;
    int var4 = weaverAwareNames.length;
    for(int var5 = 0; var5 < var4; ++var5) {
        String weaverAwareName = var3[var5];
        this.getBean(weaverAwareName);
    }
    beanFactory.setTempClassLoader((ClassLoader)null);
    //将所有的beanName，放到一个数组中存起来
    beanFactory.freezeConfiguration();
    //初始化一些类，isEagerInit==true
    beanFactory.preInstantiateSingletons();
}
//
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Pre-instantiating singletons in " + this);
    }

    List<String> beanNames = new ArrayList(this.beanDefinitionNames);
    Iterator var2 = beanNames.iterator();

    while(true) {
        String beanName;
        Object bean;
        do {
            while(true) {
                RootBeanDefinition bd;
                do {
                    do {
                        do {
                            if (!var2.hasNext()) {
                                var2 = beanNames.iterator();

                                while(var2.hasNext()) {
                                    beanName = (String)var2.next();
                                    Object singletonInstance = this.getSingleton(beanName);
                                    //如果有bean实例实现了SmartInitializingSingleton会有后置处理触发，不包括延迟加载的。例如：org.springframework.context.event. internalEventListenerProcessor会触发EventListenerMethodProcessor的afterSingletonsInstantiated方法对所有对象（Object的子类）处理
                                    if (singletonInstance instanceof SmartInitializingSingleton) {
                                        SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                        if (System.getSecurityManager() != null) {
                                            AccessController.doPrivileged(() -> {
                                                smartSingleton.afterSingletonsInstantiated();
                                                return null;
                                            }, this.getAccessControlContext());
                                        } else {
                                            smartSingleton.afterSingletonsInstantiated();
                                        }
                                    }
                                }

                                return;
                            }

                            beanName = (String)var2.next();
                            bd = this.getMergedLocalBeanDefinition(beanName);
                        } while(bd.isAbstract());
                    } while(!bd.isSingleton());
                } while(bd.isLazyInit());

                if (this.isFactoryBean(beanName)) {
                    bean = this.getBean("&" + beanName);
                    break;
                }

                this.getBean(beanName);
            }
        } while(!(bean instanceof FactoryBean));

        FactoryBean<?> factory = (FactoryBean)bean;
        boolean isEagerInit;
        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            SmartFactoryBean var10000 = (SmartFactoryBean)factory;
            ((SmartFactoryBean)factory).getClass();
            isEagerInit = (Boolean)AccessController.doPrivileged(var10000::isEagerInit, this.getAccessControlContext());
        } else {
            isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
        }

        if (isEagerInit) {
            this.getBean(beanName);
        }
    }
}
```

## 9.12 finishRefresh()：初始化生命周期，开启Web容器

```java
//ServletWebServerApplicationContext
protected void finishRefresh() {
    super.finishRefresh();
    //开启容器
    WebServer webServer = this.startWebServer();
    if (webServer != null) {
        //发布事件
        this.publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }
}
//AbstractApplicationContext
protected void finishRefresh() {
    //执行resourceCaches.clear()，这个map中放一些业务类信息
    this.clearResourceCaches();
    //初始化生命周期处理器，逻辑是判断beanFactory中是否已经注册了lifecycleProcessor，没有就new一个DefaultLifecycleProcessor
    this.initLifecycleProcessor();
    //执行上面初始化生命周期处理器的onRefresh，
    this.getLifecycleProcessor().onRefresh();
    //发布（执行）ApplicationListener类的invokeListener方法，不知道干啥的
    this.publishEvent((ApplicationEvent)(new ContextRefreshedEvent(this)));
    //注册ApplicationContext，不知道干啥的
    LiveBeansView.registerApplicationContext(this);
}
//DefaultLifecycleProcessor
public void onRefresh() {
    this.startBeans(true);
    this.running = true;
}
private void startBeans(boolean autoStartupOnly) {
    //获取不是自身的lifecycleBeans
    Map<String, Lifecycle> lifecycleBeans = this.getLifecycleBeans();
    //后面对lifecycleBeans做一些操作，不懂
}
```

## 9.13 resetCommonCaches()

```java
//AbstractApplicationContext
protected void resetCommonCaches() {
    ReflectionUtils.clearCache();
    AnnotationUtils.clearCache();
    ResolvableType.clearCache();
    CachedIntrospectionResults.clearClassLoader(this.getClassLoader());
}
```

## 9.14 registerShutdownHook()：注册应用关闭时的销毁流程

```java
//AbstractApplicationContext
//可以继承这个，实现自己的钩子
public void registerShutdownHook() {
    if (this.shutdownHook == null) {
        this.shutdownHook = new Thread() {
            public void run() {
                synchronized(AbstractApplicationContext.this.startupShutdownMonitor) {
                    AbstractApplicationContext.this.doClose();//注册销毁流程
                }
            }
        };
        Runtime.getRuntime().addShutdownHook(this.shutdownHook);
    }

}
```

# 10 afterRefresh()：空实现

```java
//SpringApplication
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
}
```

# 11 listeners.started()：通知Run监听者应用已开启

```java
//SpringApplicationRunListeners
public void started(ConfigurableApplicationContext context) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.started(context);
    }

}
```

# 12 callRunners()：调用 ApplicationRunner 或者 CommandLineRunner 的运行方法

```java
//SpringApplication
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    Iterator var4 = (new LinkedHashSet(runners)).iterator();

    while(var4.hasNext()) {
        Object runner = var4.next();
        if (runner instanceof ApplicationRunner) {
            this.callRunner((ApplicationRunner)runner, args);
        }

        if (runner instanceof CommandLineRunner) {
            this.callRunner((CommandLineRunner)runner, args);
        }
    }

}
```

# 13 listeners.running()：通知Run监听者应用已运行

```java
//SpringApplicationRunListeners
public void running(ConfigurableApplicationContext context) {
    Iterator var2 = this.listeners.iterator();
    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.running(context);
    }
}
```

```java
//初始化BeanDefinition加载器，关键：其中有三个子加载器
//1. xmlReader：解析加载XML里的bean 
//2. annotateReader：解析加载注解的bean
//3. scanner：解析加载scanner的bean
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    this.xmlReader = new XmlBeanDefinitionReader(registry);
    if (isGroovyPresent()) {
        this.groovyReader = new GroovyBeanDefinitionReader(registry);
    }
    this.scanner = new ClassPathBeanDefinitionScanner(registry);
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```

# 14 销毁

- 关闭lifecycleProcessor实例
- 销毁beanFactory中所有的实例

```java
//AbstractApplicationContext
public void close() {
    Object var1 = this.startupShutdownMonitor;
    synchronized(this.startupShutdownMonitor) {
        //在这里执行
        this.doClose();
        if (this.shutdownHook != null) {
            try {
                Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
            } catch (IllegalStateException var4) {
                ;
            }
        }

    }
}
//开始销毁
protected void doClose() {
    //确保只有一个线程进入
    if (this.active.get() && this.closed.compareAndSet(false, true)) {

        //关闭lifecycleProcessor实例
        this.lifecycleProcessor.onClose();

		//销毁beanFactory中所有的实例
        this.destroyBeans();
        //关闭bean工程
        this.closeBeanFactory();
        this.onClose();
        this.active.set(false);
    }

}
```
## 14.1 销毁beanFactory中所有的实例

```java
protected void destroyBeans() {
    this.getBeanFactory().destroySingletons();
}
public void destroySingletons() {
    super.destroySingletons();//主要是这个
}
public void destroySingletons() {
    Map var2 = this.disposableBeans;//取出所有的disposableBeans
    String[] disposableBeanNames;
    synchronized(this.disposableBeans) {
        disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
    }
	//遍历销毁所有的disposableBeans
    for(int i = disposableBeanNames.length - 1; i >= 0; --i) {
        this.destroySingleton(disposableBeanNames[i]);
    }
}
public void destroySingleton(String beanName) {
    synchronized(this.disposableBeans) {
        //pop
        disposableBean = (DisposableBean)this.disposableBeans.remove(beanName);
    }
    this.destroyBean(beanName, disposableBean);
}
protected void destroyBean(String beanName, @Nullable DisposableBean bean) {
    bean.destroy();//即调用DisposableBean接口的destroy
}
//实现了DisposableBean接口
public interface DisposableBean {
    void destroy() throws Exception;
}
```

# 15 参考文献

spring-boot:2.1.1.RELEASE

[spring boot实战(第十篇)Spring boot Bean加载源码分析](https://blog.csdn.net/liaokailin/article/details/49107209)
[spring boot 源码解析11-ConfigurationClassPostProcessor类加载解析](https://blog.csdn.net/qq_26000415/article/details/78917682)
[Spring如何加载和解析@Configuration标签](https://www.cnblogs.com/jiaoqq/p/7678037.html)
[spring容器加载分析 三Configuration类解析](https://www.jianshu.com/p/b61809506d0b)
[Spring Boot启动过程（二）](https://www.cnblogs.com/saaav/p/6292524.html)
[内置Tomcat启动](https://www.cnblogs.com/saaav/p/6323350.html#4245351)