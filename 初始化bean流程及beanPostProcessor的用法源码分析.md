[TOC]

#### 1 finishBeanFactoryInitialization->getBean

```java
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
	    //9.4 注册Scope相关的类，扫描包和注解进行注册
        this.postProcessBeanFactory(beanFactory);
        //9.5 和业务类、自动配置类有关------这里是一个关键（将所有的bean注册成beanDefinition以便后面的初始化）
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
        //9.11 主要是初始化beanDefinition
        //实例化所有的不是延迟加载（延迟加载的只有在使用的时候才会实例化）的bean实例
        this.finishBeanFactoryInitialization(beanFactory);
        //9.12 主要是开启Web容器
        this.finishRefresh();
	    //9.13 清除一些缓存
        this.resetCommonCaches();
    }
}
```

#### 2 getBean->doGetBean->createBean

1. 检查缓存中是否已经存在了bean实例
   1. 如果有，判断bean是否正在创建，打相应的日志
2. 如果bean不存在，分情况处理
   1. bean是单例，createBean()
   2. bean是多例，createBean()
   3. bean是其它（Session）
3. 如果!requiredType.isInstance(bean)，要做什么转换吧，否则直接返回bean

```java
//finishBeanFactoryInitialization最终会遍历beanDefinition，调用getBean方法初始化bean
public Object getBean(String name, Object... args) throws BeansException {
    return this.doGetBean(name, (Class)null, args, false);
}

protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
    //这里是通过别名获取beanName（这样就可以注册别名了）
    String beanName = this.transformedBeanName(name);
    //检查缓存中是否已经存在了bean实例
    Object sharedInstance = this.getSingleton(beanName);
    Object bean;
    //如果有，判断bean是否正在创建，打相应的日志
    if (sharedInstance != null && args == null) {
        if (this.logger.isTraceEnabled()) {
            if (this.isSingletonCurrentlyInCreation(beanName)) {
                this.logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
            } else {
                this.logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
    } else {
        //如果是多例模式，正在创建，那肯定有问题，因为多例是互不相关的，直接报错
        if (this.isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        //在父类上下文中找该bean，找到了就返回(代码略)
        
        //获取缓存的BeanDefinition对象并合并其父类和本身的属性
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

        //递归初始化该类的依赖类
        String[] dependsOn = mbd.getDependsOn();
        for (String dep : dependsOn) {
            getBean(dep);
        }
        
        if (mbd.isSingleton()) {
            //单例初始化
            sharedInstance = createBean();
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        } else if (mbd.isPrototype()) {
            //多例初始化，差不多
        } else {
            //其它类型（如Session等）
            String scopeName = mbd.getScope();
            Scope scope = (Scope)this.scopes.get(scopeName);
            //初始化
        }
    }
    if (requiredType != null && !requiredType.isInstance(bean)) {
        //如果!requiredType.isInstance(bean)，要做什么转换吧
        T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
        return convertedBean;
    } else {
        return bean;
    }
}
```
#### 3 createBean->doCreateBean 

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    RootBeanDefinition mbdToUse = mbd;
	//设置一些mbdToUse属性（代码略）
    
    //如果有子父类有相同的方法，设置override属性
    mbdToUse.prepareMethodOverrides();

    Object beanInstance;
    //if (targetType != null) {
    //    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
    //    if (bean != null) {
    //        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
    //    }
    // }
    beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
    //如果这边就已经可以初始化了，那就直接返回了，不会有依赖处理不来的问题（因为之前已经处理依赖了）
    if (beanInstance != null) {
        return beanInstance;
    }
	//4
    beanInstance = this.doCreateBean(beanName, mbdToUse, args);
    return beanInstance;
}
```

#### 4 doCreateBean->(createBeanInstance,populateBean,initializeBean,registerDisposableBeanIfNecessary)

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        //相当于pop
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }

    if (instanceWrapper == null) {
        //如果没有，就创建，通过反射创建wrapper实例
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }
	//包装类
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    Object var7 = mbd.postProcessingLock;
    synchronized(mbd.postProcessingLock) {
        //如果还没处理
        if (!mbd.postProcessed) {
            //调用所有的MergedBeanDefinitionPostProcessors
            this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }
	//提前暴露引用，允许循环引用，一般为true
    boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
    if (earlySingletonExposure) {
		//打日志代码略
        this.addSingletonFactory(beanName, () -> {
            return this.getEarlyBeanReference(beanName, mbd, bean);
        });
    }
    Object exposedObject = bean;
    //5
    this.populateBean(beanName, mbd, instanceWrapper);
    //6
    exposedObject = this.initializeBean(beanName, exposedObject, mbd);

	//处理earlySingletonExposure代码略

    //注册注解的DisposableBean方法
    this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
    return exposedObject;

}
```

#### 5 populateBean

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
    } else {
        boolean continueWithPropertyPopulation = true;
        if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
            Iterator var5 = this.getBeanPostProcessors().iterator();

            while(var5.hasNext()) {
                BeanPostProcessor bp = (BeanPostProcessor)var5.next();
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                        continueWithPropertyPopulation = false;
                        break;
                    }
                }
            }
        }

        if (continueWithPropertyPopulation) {
            PropertyValues pvs = mbd.hasPropertyValues() ? mbd.getPropertyValues() : null;
            if (mbd.getResolvedAutowireMode() == 1 || mbd.getResolvedAutowireMode() == 2) {
                MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                if (mbd.getResolvedAutowireMode() == 1) {
                    this.autowireByName(beanName, mbd, bw, newPvs);
                }

                if (mbd.getResolvedAutowireMode() == 2) {
                    this.autowireByType(beanName, mbd, bw, newPvs);
                }

                pvs = newPvs;
            }

            boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
            boolean needsDepCheck = mbd.getDependencyCheck() != 0;
            PropertyDescriptor[] filteredPds = null;
            if (hasInstAwareBpps) {
                if (pvs == null) {
                    pvs = mbd.getPropertyValues();
                }

                Iterator var9 = this.getBeanPostProcessors().iterator();

                while(var9.hasNext()) {
                    BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                        PropertyValues pvsToUse = ibp.postProcessProperties((PropertyValues)pvs, bw.getWrappedInstance(), beanName);
                        if (pvsToUse == null) {
                            if (filteredPds == null) {
                                filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                            }

                            pvsToUse = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
                            if (pvsToUse == null) {
                                return;
                            }
                        }

                        pvs = pvsToUse;
                    }
                }
            }

            if (needsDepCheck) {
                if (filteredPds == null) {
                    filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }

                this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
            }

            if (pvs != null) {
                this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
            }

        }
    }
}
```

#### 6 initializeBean

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(() -> {
            this.invokeAwareMethods(beanName, bean);
            return null;
        }, this.getAccessControlContext());
    } else {
        //就3中AwareMethods（BeanNameAware，BeanClassLoaderAware，BeanFactoryAware）
        //统一调用bean.setxxxAware;如果该类实现了某个aware接口的话
        this.invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        //初始化之前调用，这里调用了@PostConstruct注解的初始化方法！通过CommonAnnotationPostprocessor
        wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    }

    //调用初始化方法，继承了InitializingBean（调用它的afterPropertiesSet方法）或指定了InitMethod（通过注解等）先调用afterPropertiesSet()，后调用initMethod(@Bean(initMethod="..."))
    this.invokeInitMethods(beanName, wrappedBean, mbd);

    if (mbd == null || !mbd.isSynthetic()) {
        //初始化之后调用
        wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

#### 6 registerDisposableBeanIfNecessary

```java
//关键在DisposableBeanAdapter中解析了类的destroy方法，不具体分析了
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
   AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
   if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
      if (mbd.isSingleton()) {
         // Register a DisposableBean implementation that performs all destruction
         // work for the given bean: DestructionAwareBeanPostProcessors,
         // DisposableBean interface, custom destroy method.
         registerDisposableBean(beanName,
               new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
      }
      else {
         // A bean with a custom scope...
         Scope scope = this.scopes.get(mbd.getScope());
         if (scope == null) {
            throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
         }
         scope.registerDestructionCallback(beanName,
               new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
      }
   }
}
```

#### 7 applyBeanPostProcessorsBeforeInitialization

```
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
    Object result = existingBean;

    Object current;
    for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
        BeanPostProcessor processor = (BeanPostProcessor)var4.next();
        current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
    }

    return result;
}
```



#### 8 applyMergedBeanDefinitionPostProcessors

```
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
    Iterator var4 = this.getBeanPostProcessors().iterator();

    while(var4.hasNext()) {
        BeanPostProcessor bp = (BeanPostProcessor)var4.next();
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor)bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
        }
    }

}
```



#### 参考文献

[深入理解spring生命周期与BeanPostProcessor的实现原理](https://blog.csdn.net/varyall/article/details/82257202)

[Spring源码：bean创建（三）:createBeanInstance](https://blog.csdn.net/finalcola/article/details/81451019)

[依赖注入之Bean实例化前的准备](https://blog.csdn.net/jzq114/article/details/50831499)