1

```java
//import(DataSourceInitializerInvoker)
DataSourceInitializerInvoker(ObjectProvider<DataSource> dataSource, DataSourceProperties properties, ApplicationContext applicationContext) {
    this.dataSource = dataSource;
    this.properties = properties;//关键，这里的properties是从spring.datasource中得到的！！
    // @ConfigurationProperties(
    //	 prefix = "spring.datasource"
    // )
    this.applicationContext = applicationContext;
}
//最后调用这个方法，这个方法就是初始化schema和data的
public void afterPropertiesSet() {
    //这个过后，dataSource变成了@primary指定的数据源！！
    DataSourceInitializer initializer = this.getDataSourceInitializer();
    if (initializer != null) {
        boolean schemaCreated = this.dataSourceInitializer.createSchema();
        if (schemaCreated) {
            this.initialize(initializer);
        }
    }
}
//
public boolean createSchema() {
    //关键：是从这里定位到Scripts资源的！！
    List<Resource> scripts = this.getScripts("spring.datasource.schema", this.properties.getSchema(), "schema");
    if (!scripts.isEmpty()) {
        if (!this.isEnabled()) {
            logger.debug("Initialization disabled (not running DDL scripts)");
            return false;
        }

        String username = this.properties.getSchemaUsername();
        String password = this.properties.getSchemaPassword();
        //有一个关键！！！这里运行scripts资源
        this.runScripts(scripts, username, password);
    }

    return !scripts.isEmpty();
}
//如何获取Scripts，默认获取地址是哪个
private List<Resource> getScripts(String propertyName, List<String> resources, String fallback) {
    if (resources != null) {
        return this.getResources(propertyName, resources, true);
    } else {
        //默认获取地址：classpath*:schema-all.sql,classpath*:schema.sql
        String platform = this.properties.getPlatform();
        List<String> fallbackResources = new ArrayList();
        fallbackResources.add("classpath*:" + fallback + "-" + platform + ".sql");
        fallbackResources.add("classpath*:" + fallback + ".sql");
        return this.getResources(propertyName, fallbackResources, false);
    }
}

private void runScripts(List<Resource> resources, String username, String password) {
    if (!resources.isEmpty()) {
        ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
        populator.setContinueOnError(this.properties.isContinueOnError());
        populator.setSeparator(this.properties.getSeparator());
        if (this.properties.getSqlScriptEncoding() != null) {
            populator.setSqlScriptEncoding(this.properties.getSqlScriptEncoding().name());
        }

        Iterator var5 = resources.iterator();

        while(var5.hasNext()) {
            Resource resource = (Resource)var5.next();
            populator.addScript(resource);
        }
	    //这里获取的还是@primary指定的数据源
        DataSource dataSource = this.dataSource;
        //关键是这一步，如果username这两个多有的话，数据源就变成spring.datasource.url了，哈哈，这是干啥，直接用@primary数据源不就行了吗，但是也好，初始化schema的数据源，独立于其他的数据源
        //如果要用@primary的数据源，不指定data-username，schema-username即可即可
        if (StringUtils.hasText(username) && StringUtils.hasText(password)) {
            dataSource = DataSourceBuilder.create(this.properties.getClassLoader()).driverClassName(this.properties.determineDriverClassName()).url(this.properties.determineUrl()).username(username).password(password).build();
        }
		//
        DatabasePopulatorUtils.execute(populator, dataSource);
    }
}
```

