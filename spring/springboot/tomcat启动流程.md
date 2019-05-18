```java
protected void onRefresh() {
   //1 
   createWebServer();
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        //2 
        ServletWebServerFactory factory = getWebServerFactory();
        //5 这里getWebServer的参数是一个lamda表达式，通过getSelfInitializer()获取lamda表达式对应的具体方法
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        getSelfInitializer().onStartup(servletContext);
    }
    initPropertySources();
}
//2 
protected ServletWebServerFactory getWebServerFactory() {
    // Use bean names so that we don't consider the hierarchy
    // 去 beanDefinitionMap中找，ServletWebServerFactory的类
    // 找到了一个 tomcatServletWebServerFactory：这个bean是在ServletWebServerFactoryConfiguration中引入的
    String[] beanNames = getBeanFactory()
        .getBeanNamesForType(ServletWebServerFactory.class);
    // 3 初始这个bean
    return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}
//3 后置处理器处理 是 WebServerFactory 的类，调用5个类的customize方法
private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
    LambdaSafe
        .callbacks(WebServerFactoryCustomizer.class, getCustomizers(),
                   webServerFactory)
        .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
        .invoke((customizer) -> customizer.customize(webServerFactory));
}
//4 返回了5个WebServerFactoryCustomizer类：tomcatWebSocketServletWebServerCustomizer/ServletWebServerFactoryCustomizer/TomcatServletWebServerFactoryCustomizer/TomcatWebServerFactoryCustomizer/HttpEncodingAutoConfiguration$LocaleCharsetMappingsCustomizer
private Collection<WebServerFactoryCustomizer<?>> getCustomizers() {
    if (this.customizers == null) {
        // Look up does not include the parent context
        this.customizers = new ArrayList<>(getWebServerFactoryCustomizerBeans());
        this.customizers.sort(AnnotationAwareOrderComparator.INSTANCE);
        this.customizers = Collections.unmodifiableList(this.customizers);
    }
    return this.customizers;
}
//5 这个是核心类，初始化了tomcat中的connect，这个connect负责接收发生请求，其中的ProtocolHandler实现了线程
public WebServer getWebServer(ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
        : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    //将配置文件中的属性配置到connector中
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
}
```

```java
//TomcatWebSocketServletWebServerCustomizer
public void customize(TomcatServletWebServerFactory factory) {
   factory.addContextCustomizers((context) -> context
         .addApplicationListener(WsContextListener.class.getName()));
}
//ServletWebServerFactoryCustomizer：给 factory 设置 serverProperties
public void customize(ConfigurableServletWebServerFactory factory) {
    PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
    map.from(this.serverProperties::getPort).to(factory::setPort);
    map.from(this.serverProperties::getAddress).to(factory::setAddress);
    map.from(this.serverProperties.getServlet()::getContextPath)
        .to(factory::setContextPath);
    map.from(this.serverProperties.getServlet()::getApplicationDisplayName)
        .to(factory::setDisplayName);
    map.from(this.serverProperties.getServlet()::getSession).to(factory::setSession);
    map.from(this.serverProperties::getSsl).to(factory::setSsl);
    map.from(this.serverProperties.getServlet()::getJsp).to(factory::setJsp);
    map.from(this.serverProperties::getCompression).to(factory::setCompression);
    map.from(this.serverProperties::getHttp2).to(factory::setHttp2);
    map.from(this.serverProperties::getServerHeader).to(factory::setServerHeader);
    map.from(this.serverProperties.getServlet()::getContextParameters)
        .to(factory::setInitParameters);
}
//TomcatServletWebServerFactoryCustomizer：给 tomcat 设置 serverProperties
public void customize(TomcatServletWebServerFactory factory) {
    ServerProperties.Tomcat tomcatProperties = this.serverProperties.getTomcat();
    if (!ObjectUtils.isEmpty(tomcatProperties.getAdditionalTldSkipPatterns())) {
        factory.getTldSkipPatterns()
            .addAll(tomcatProperties.getAdditionalTldSkipPatterns());
    }
    if (tomcatProperties.getRedirectContextRoot() != null) {
        customizeRedirectContextRoot(factory,
                                     tomcatProperties.getRedirectContextRoot());
    }
    if (tomcatProperties.getUseRelativeRedirects() != null) {
        customizeUseRelativeRedirects(factory,
                                      tomcatProperties.getUseRelativeRedirects());
    }
}
//
public void customize(ConfigurableTomcatWebServerFactory factory) {
    ServerProperties properties = this.serverProperties;
    ServerProperties.Tomcat tomcatProperties = properties.getTomcat();
    PropertyMapper propertyMapper = PropertyMapper.get();
    propertyMapper.from(tomcatProperties::getBasedir).whenNonNull()
        .to(factory::setBaseDirectory);
    propertyMapper.from(tomcatProperties::getBackgroundProcessorDelay).whenNonNull()
        .as(Duration::getSeconds).as(Long::intValue)
        .to(factory::setBackgroundProcessorDelay);
    customizeRemoteIpValve(factory);
    //向connector中设置maxThreads
    propertyMapper.from(tomcatProperties::getMaxThreads).when(this::isPositive)
        .to((maxThreads) -> customizeMaxThreads(factory,
                                                tomcatProperties.getMaxThreads()));
    propertyMapper.from(tomcatProperties::getMinSpareThreads).when(this::isPositive)
        .to((minSpareThreads) -> customizeMinThreads(factory, minSpareThreads));
    propertyMapper.from(this::determineMaxHttpHeaderSize).whenNonNull()
        .asInt(DataSize::toBytes).when(this::isPositive)
        .to((maxHttpHeaderSize) -> customizeMaxHttpHeaderSize(factory,
                                                              maxHttpHeaderSize));
    propertyMapper.from(tomcatProperties::getMaxSwallowSize).whenNonNull()
        .asInt(DataSize::toBytes)
        .to((maxSwallowSize) -> customizeMaxSwallowSize(factory, maxSwallowSize));
    propertyMapper.from(tomcatProperties::getMaxHttpPostSize).asInt(DataSize::toBytes)
        .when((maxHttpPostSize) -> maxHttpPostSize != 0)
        .to((maxHttpPostSize) -> customizeMaxHttpPostSize(factory,
                                                          maxHttpPostSize));
    propertyMapper.from(tomcatProperties::getAccesslog)
        .when(ServerProperties.Tomcat.Accesslog::isEnabled)
        .to((enabled) -> customizeAccessLog(factory));
    propertyMapper.from(tomcatProperties::getUriEncoding).whenNonNull()
        .to(factory::setUriEncoding);
    propertyMapper.from(properties::getConnectionTimeout).whenNonNull()
        .to((connectionTimeout) -> customizeConnectionTimeout(factory,
                                                              connectionTimeout));
    propertyMapper.from(tomcatProperties::getMaxConnections).when(this::isPositive)
        .to((maxConnections) -> customizeMaxConnections(factory, maxConnections));
    propertyMapper.from(tomcatProperties::getAcceptCount).when(this::isPositive)
        .to((acceptCount) -> customizeAcceptCount(factory, acceptCount));
    customizeStaticResources(factory);
    customizeErrorReportValve(properties.getError(), factory);
}

public void customize(ConfigurableServletWebServerFactory factory) {
    if (this.properties.getMapping() != null) {
        factory.setLocaleCharsetMappings(this.properties.getMapping());
    }
}
```