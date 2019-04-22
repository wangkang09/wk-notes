- 第一步生成`sqlSessionFactory`
  1. 将mybatis.xml解析成doc形式
  2. 将doc映射成Configuration类
```java
public void main() throws IOException {
    InputStream inputStream = Resources.getResourceAsStream("mybatis-cfg.xml");

    //构建分为两步：
    //1、XMLConfigBuilder解析xml文件，设置Configuration类。（包括mybatis.xml/mapper.xml)
    //2、使用Configuration对象创建SqlSessionFactory。默认实现是DefaultSqlSessionFactory
    //SqlSessionFactory只有两个方法：getConfiguration()，openSession(...);
    //所以，这行代码最关键的就是构建了一个Configuration类
    //这一步是最繁琐的地方，主要就是解析配置文件，设置到Configuration类中
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

    //这里最主要是得到executor，并封装到SqlSession中。executor才是整个数据库操作的核心！！
    SqlSession sqlSession = sqlSessionFactory.openSession();
    
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    User user = userMapper.selectById(1L);
    System.out.println(user.toString());
}
//SqlSessionFactory只有两个方法，一个是Con
public interface SqlSessionFactory {
  SqlSession openSession(...);
  Configuration getConfiguration();
}
```

```java
//默认调用这个方法，build的后两个属性就是null了
public SqlSessionFactory build(InputStream inputStream) {
  return build(inputStream, null, null);
}
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
      //1.用XPathParser类将inputStream解析成docment类，放入XPathParser类的Docment属性中
      //2.生成一个Xml(doc)解析类
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);

      //解析doc，并生成 Configuration对象
      //构建一个DefaultSqlSessionFactory，里面的属性就有Configuration
      // parser.parse()是最最关键的！！！解析XML生成Configuration类！！！
      return build(parser.parse());
  }
//最关键的就是，下面这个config类，设置到SqlSessionFactory中，
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```


- 第2步获得`sqlSession`

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
      //获得mybatis.xml中的environment节点(数据库连接和事务属性等信息)属性
      final Environment environment = configuration.getEnvironment();

       //通过环境获取到事务工厂
       //JDBC实现JdbcTransactionFactory
       //<transactionManager type="JDBC" />
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);

      //创建事务对象，主要就是getConnection，commit，rollback，close几个方法
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

      //Executor 是什么呢？它其实是一个执行器，SqlSession 的操作会交给 Executor 去执行
      //核心的数据库操作逻辑，封装在 executor中！
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
  }
```

- 第3步，**核心！**，生成executor

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {

    // 如果手动设置过默认执行器类型就使用默认执行器类型（setDefaultExecutorType）
    //默认为Simple
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }

    // 一级级缓存，如果开启缓存就用缓存执行器包装一层。（默认开启）
    if (cacheEnabled) {
        //通过代理形式，将CachingExecutor实例设置到executor中
        executor = new CachingExecutor(executor);
    }

    // 拦截器链（应用插件）：关键的扩展点
    // 作用在执行增删改查前后的拦截器
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

- 第4步，获得对应的mapper接口

```java
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
}
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //这个mapperRegistry在Configuration初始化的时候旧通过mapper.xml生成了
    return mapperRegistry.getMapper(type, sqlSession);
}
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //获取mapper代理工厂，这个mapperRegis
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
	//通过特定的mapper和对应的sqlSession生成代理对象mapperProxy
    //通过反射生成对象
    //MapperProxyFactory和SqlSession是分开来的！！
     return mapperProxyFactory.newInstance(sqlSession);
}
public T newInstance(SqlSession sqlSession) {
    //关键是这个：MapperProxy为InvocationHandler的实现类
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    //真实生成代理
    return newInstance(mapperProxy);
}
```

- 第5步，执行（一级缓存是关不掉的）

```java
User user = userMapper.selectById(1L);
//mapperProxy的关键反射方法，这个方法在调用mapper接口方法时，调用
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    // 如果是 Object类中的方法，直接调用
    if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
        // 见 https://github.com/mybatis/mybatis-3/issues/709 ，支持 JDK8 default 方法
    } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
    }

    //尝试从缓存methodCache中获取，没有就新建一个并缓存起来
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //真正的处理在这里，通过MapperMethod对象调用方法
    return mapperMethod.execute(sqlSession, args);
}

//这个一级缓存是关不掉的
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;

    // 在缓存中，添加占位对象。此处的占位符，和延迟加载有关，可见 `DeferredLoad#canLoad()` 方法
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // 从缓存中，移除占位对象
        localCache.removeObject(key);
    }
    localCache.putObject(key, list);

    // 忽略，存储过程相关
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }

    return list;
}
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }

    // 清空本地缓存，如果 queryStack 为零，并且要求清空本地缓存。
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;

        // 从一级缓存中，获取查询结果
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            //存储过程相关
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 获得不到，则从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        // 执行延迟加载
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        // 如果缓存级别是 LocalCacheScope.STATEMENT ，则进行清理
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}

```



[SpringBoot事务管理](http://www.importnew.com/24138.html)

[禁用Mybatis1级缓存](https://blog.csdn.net/j7636579/article/details/73647885)