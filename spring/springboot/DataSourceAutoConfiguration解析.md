[TOC]
# DataSourceAutoConfiguration解析

- DataSourceAutoConfiguration 做了以下5件事情：
- 1、初始化 DataSourceProperties 配置文件
- 2、引入EmbeddedDatabaseConfiguration配置类
- 3、引入PooledDataSourceConfiguration配置类
- 4、导入DataSourceInitializationConfiguration配置类
- 5、导入DataSourcePoolMetadataProvidersConfiguration配置类

```java
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
//第1件事
@EnableConfigurationProperties(DataSourceProperties.class)
//第4、5件事
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
      DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

   //第2件事
   @Configuration
   @Conditional(EmbeddedDatabaseCondition.class)
   @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
   @Import(EmbeddedDataSourceConfiguration.class)
   protected static class EmbeddedDatabaseConfiguration {

   }
   //第3件事
   @Configuration
   @Conditional(PooledDataSourceCondition.class)
   @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
   @Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
         DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
         DataSourceJmxConfiguration.class })
   protected static class PooledDataSourceConfiguration {

   }
   /**
    * 只要任何一个条件满足，就满足：1）有spring.datasource.type属性 2）满足PooledDataSourceAvailableCondition：项目中引入了连接池依赖(如，hikari)
    * 说明：如果没有spring.datasource.type属性，就默认查看项目中有没有引入：hikari，tomcat，dbcp2。这样说明如果项目中exclude了这3个，那么就必须使用 spring.datasource.type来指定数据库连接池了
    */
   static class PooledDataSourceCondition extends AnyNestedCondition {
      PooledDataSourceCondition() {
         super(ConfigurationPhase.PARSE_CONFIGURATION);
      }
      @ConditionalOnProperty(prefix = "spring.datasource", name = "type")
      static class ExplicitType {
      }
      @Conditional(PooledDataSourceAvailableCondition.class)
      static class PooledDataSourceAvailable {
      }
   }

   /**
    * 是否能在项目中找到一个数据库连接池的class：项目中有没有引入连接池依赖
    */
   static class PooledDataSourceAvailableCondition extends SpringBootCondition {
      @Override
      public ConditionOutcome getMatchOutcome(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
         ConditionMessage.Builder message = ConditionMessage
               .forCondition("PooledDataSource");
          //getDataSourceClassLoader(context)：内部做class.forName来找项目中的相关class，找到了就不为null啦，一般肯定能找到的，在org.springframework.boot:spring-boot-starter-jdbc中就已经引入了 hikariDatabase，而在 spring.boot:mybatis-spring-boot-starter中引入了 jdbc!
         if (getDataSourceClassLoader(context) != null) {
            return ConditionOutcome
                  .match(message.foundExactly("supported DataSource"));
         }
         return ConditionOutcome
               .noMatch(message.didNotFind("supported DataSource").atAll());
      }
      private ClassLoader getDataSourceClassLoader(ConditionContext context) {
          //在DataSourceBuilder中有个关键的 findType方法来按：hikari，tomcat，dbcp2顺序查找，一查到就返回
         Class<?> dataSourceClass = DataSourceBuilder
               .findType(context.getClassLoader());
         return (dataSourceClass != null) ? dataSourceClass.getClassLoader() : null;
      }

   }

   /**
    * 判断是否使用内置的数据库：H2、Derby、HSQL，注意这里是数据库，不是数据库连接池
    */
   static class EmbeddedDatabaseCondition extends SpringBootCondition {
      private final SpringBootCondition pooledCondition = new PooledDataSourceCondition();
      @Override
      public ConditionOutcome getMatchOutcome(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
         ConditionMessage.Builder message = ConditionMessage
               .forCondition("EmbeddedDataSource");
         if (anyMatches(context, metadata, this.pooledCondition)) {
            return ConditionOutcome
                  .noMatch(message.foundExactly("supported pooled data source"));
         }
         EmbeddedDatabaseType type = EmbeddedDatabaseConnection
               .get(context.getClassLoader()).getType();
         if (type == null) {
            return ConditionOutcome
                  .noMatch(message.didNotFind("embedded database").atAll());
         }
         return ConditionOutcome.match(message.found("embedded database").items(type));
      }

   }
}
```
## 1 初始化 DataSourceProperties 配置文件

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
	private ClassLoader classLoader;
    //数据库名：如果使用内置数据库，默认为testdb
	private String name;
	//Whether to generate a random datasource name
	private boolean generateUniqueName;
    //如果generateUniqueName==true,则不使用name,而使用uniqueName来做数据库名
    private String uniqueName;
    //完整的数据库连接池名。默认从项目中检测出
	private Class<? extends DataSource> type;
    //JDBC driver的完整名，默认从URL中检测出相对应的driver
	private String driverClassName;
    //JDBC URL of the database
	private String url;
    
	//Login username of the database
	private String username;
	//Login password of the database
	private String password;
    //JNDI 数据源的位置：如果指定了，则数据库连接数据将会失效：driverClassName,url,username,password
	private String jndiName;
	//初始化database使用的sql文件的模式，默认是EMBEDDED，如果是NONE就不会执行sql文件
    //如果设置的模式和检测出来的模式不匹配，也不会执行sql文件
	private DataSourceInitializationMode initializationMode = DataSourceInitializationMode.EMBEDDED;

    //执行sql文件相关，schema-${platform}.sql,data-${platform}.sql
    //默认执行不带 platform 的 sql 文件 + 带 platform 的 sql 文件
	private String platform = "all";
    //具体的 schema 文件的位置，如果指定了这个就不会查找默认的sql文件了
	private List<String> schema;
	//执行schema使用数据库的用户名
	private String schemaUsername;
	//执行schema使用数据库的密码，如果schemaUsername和schemaPassword都不指定，就使用 **主数据源** 作为执行目的数据库！
	private String schemaPassword;
	//同schema
	private List<String> data;
	private String dataUsername;
	private String dataPassword;
    
    //如果初始化database时报错，是否继续
	private boolean continueOnError = false;
	//Statement separator in SQL initialization scripts.
	private String separator = ";";
	//SQL scripts encoding.
	private Charset sqlScriptEncoding;
	//默认的内置数据库连接信息：
    //1 NONE(null, null, null)
    //2 H2(EmbeddedDatabaseType.H2, "org.h2.Driver","jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE")
    //3 DERBY(...)
    //4 HSQL(...)
	private EmbeddedDatabaseConnection embeddedDatabaseConnection = EmbeddedDatabaseConnection.NONE;

	private Xa xa = new Xa();
	//从项目中匹配相应的内置数据库，查找是否引入了相应的依赖，如果引入了H2依赖，这里embeddedDatabaseConnection就设置成 H2
	@Override
	public void afterPropertiesSet() throws Exception {
		this.embeddedDatabaseConnection = EmbeddedDatabaseConnection
				.get(this.classLoader);
	}

    // 通过 spring.datasource 属性 初始化一个DataSourceBuilder，用来方便的创建datasource，就是一个封装的方法
	public DataSourceBuilder<?> initializeDataSourceBuilder() {
		return DataSourceBuilder.create(getClassLoader()).type(getType())
				.driverClassName(determineDriverClassName()).url(determineUrl())
				.username(determineUsername()).password(determinePassword());
	}
	//智能的获取DriverClassName
	public String  determineDriverClassName() {
		if (StringUtils.hasText(this.driverClassName)) {
			Assert.state(driverClassIsLoadable(),
					() -> "Cannot load driver class: " + this.driverClassName);
			return this.driverClassName;
		}
		String driverClassName = null;
		if (StringUtils.hasText(this.url)) {
			driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
		}
        //如果走到这，还没识别出 driverClassName，且它为null，就去内置数据库中找匹配的
        //如果项目中没有引入 内置数据库依赖，那就会报错啦
		if (!StringUtils.hasText(driverClassName)) {
			driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
		}
		if (!StringUtils.hasText(driverClassName)) {
			throw new DataSourceBeanCreationException(
					"Failed to determine a suitable driver class", this,
					this.embeddedDatabaseConnection);
		}
		return driverClassName;
	}
    private boolean driverClassIsLoadable() {
		try {
			ClassUtils.forName(this.driverClassName, null);
			return true;
		}
		catch (UnsupportedClassVersionError ex) {
			// Driver library has been compiled with a later JDK, propagate error
			throw ex;
		}
		catch (Throwable ex) {
			return false;
		}
	}
	//Determine the url to use based on this configuration and the environment.
	public String determineUrl() {
		if (StringUtils.hasText(this.url)) {
			return this.url;
		}
		String databaseName = determineDatabaseName();
		String url = (databaseName != null)
				? this.embeddedDatabaseConnection.getUrl(databaseName) : null;
		if (!StringUtils.hasText(url)) {
			throw new DataSourceBeanCreationException(
					"Failed to determine suitable jdbc url", this,
					this.embeddedDatabaseConnection);
		}
		return url;
	}
	//Determine the name to used based on this configuration.
	public String determineDatabaseName() {
		if (this.generateUniqueName) {
			if (this.uniqueName == null) {
				this.uniqueName = UUID.randomUUID().toString();
			}
			return this.uniqueName;
		}
		if (StringUtils.hasLength(this.name)) {
			return this.name;
		}
		if (this.embeddedDatabaseConnection != EmbeddedDatabaseConnection.NONE) {
			return "testdb";
		}
		return null;
	}
	//
	public String determineUsername() {
		if (StringUtils.hasText(this.username)) {
			return this.username;
		}
		if (EmbeddedDatabaseConnection.isEmbedded(determineDriverClassName())) {
			return "sa";
		}
		return null;
	}

	public String determinePassword() {
		if (StringUtils.hasText(this.password)) {
			return this.password;
		}
		if (EmbeddedDatabaseConnection.isEmbedded(determineDriverClassName())) {
			return "";
		}
		return null;
	}

	//XA Specific datasource settings.
	public static class Xa {
		//XA datasource fully qualified name.
		private String dataSourceClassName;
		//Properties to pass to the XA data source.
		private Map<String, String> properties = new LinkedHashMap<>();
    }
    

```

## 2 引入EmbeddedDatabaseConfiguration配置类

```java
@Configuration
//如果项目中找不到引入的数据库连接池，且引入了内置数据库(如H2),才满足条件
@Conditional(EmbeddedDatabaseCondition.class)
//没有DataSource、XADataSource实例
@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
//满足以上条件时，才解析EmbeddedDataSourceConfiguration：这个配置类就是注册了一个EmbeddedDatabase实例
//一般情况下，基本上不会进入这里，因为第一个条件基本上不会满足，mybatis就自动引入了hikari连接池了
@Import(EmbeddedDataSourceConfiguration.class)
protected static class EmbeddedDatabaseConfiguration {}

```

## 3 引入PooledDataSourceConfiguration配置类

```java
@Configuration
//满足其中的任意一个：1）有spring.datasource.type属性 2）满足PooledDataSourceAvailableCondition：项目中引入了连接池依赖
@Conditional(PooledDataSourceCondition.class)
@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
//如果满足上面条件，就解析一下几个配置类（注意顺序，hikari优先）
@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
      DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
      DataSourceJmxConfiguration.class })
protected static class PooledDataSourceConfiguration {}
```

## 4 导入DataSourceInitializationConfiguration配置类

- 这个类主要用来执行 sql 文件

## 5 导入DataSourcePoolMetadataProvidersConfiguration配置类

- https://blog.csdn.net/qq_26000415/article/details/79067214：分析

## 6 总结

### 6.1 从 DataSourceProperties 可以得出

#### 6.1.1 如果设置的不是内置数据库的话

1. 必须的配置有：url，username和password
2. 数据库名不必须
3. driverClassName不是必须的：可以从url推导出
4. type不是必须的：从项目中可以推出，通过DataSourceBuilder#getType方法

#### 6.1.2 如果设置内置数据库的话

1. 必须的配置有：引入内置数据库依赖
2. 其它所有的都可以不配置：有默认的，应该就不配置了，使用默认的，避免使用时配置文件设置冲突

### 6.2 从EmbeddedDatabaseConfiguration可以得出

- 一般情况下，基本上不会进入这里，因为第一个条件基本上不会满足，mybatis就自动引入了hikari连接池了