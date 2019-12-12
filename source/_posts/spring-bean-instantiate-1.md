---
title: Spring容器源码解析之Bean的实例化(一)
date: 2019-12-12 16:03:12
category: spring
tags: [spring]
---

> 提醒：本文是基于Spring 3.0.0.RELEASE 版本进行讲解的，其他版本可能稍有差异，在贴源码的时候，部分不影响流程的代码也在本文省略了

- [Spring容器加载过程源码解析之Resource定位加载](https://me.oopcoder.cn/2019/10/18/spring-resource-load/)
- [Spring容器加载过程源码解析之Resource解析流程](https://me.oopcoder.cn/2019/10/21/spring-resource-parse/)
- [Spring容器加载过程源码解析之默认标签解析](https://me.oopcoder.cn/2019/10/27/spring-resource-parse-default-element/index.html)
- [Spring容器加载过程源码解析之自定义标签解析](https://me.oopcoder.cn/2019/11/01/spring-resource-parse-custom-element/)

经过上面【Spring加载过程源码解析系列】的学习，我们完成了 `Bean `配置的解析和注册过程，容器中存在的是 `Bean` 对应的 `BeanDefinition`,`Bean`并没有完成实例化，接下来我们看看到底是怎么实例化的。

开门见山，`Bean` 的实例化是在使用前完成的，即在方法 `getBean` 中 实例化。

##### 1. getBean
```
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
    return doGetBean(name, requiredType, null, false);
}

public Object getBean(String name, Object... args) throws BeansException {
    return doGetBean(name, null, args, false);
}

public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
    return doGetBean(name, requiredType, args, false);
}
```

四个重载方法都指向了 `doGetBean` 方法

##### 2. doGetBean
```
private <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, 
        boolean typeCheckOnly) throws BeansException {
    
    // 将传入的name转换为实际的beanName, a -> a ; &a -> a; 
    // 如果该bean类型class是FactoryBean, 无论是getBean(a),还是getBean(&a),
    // 都是先实例化FactoryBean, 然后再返回实际需要的bean
    final String beanName = transformedBeanName(name);
    Object bean;
    // 先从缓存中获取单例
    Object sharedInstance = getSingleton(beanName);
    // 缓存中存在单例，并且 args 为空(因为只有prototype才能带参数 args)，才进入 getObjectForBeanInstance
    if (sharedInstance != null && args == null) {
        // 获取实际需要的bean, sharedInstance 可能为 FactoryBean
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // 检查是否为 当前线程正在创建的prototype 实例
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        // 检查是否存在当前beanFactory中，不存在则去父工厂(存在的话)中找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }
        // 检查是否只是为了类型检查而创建bean, 不是的话将beanName添加到已创建bean容器中
        // removeSingletonIfCreatedForTypeCheckOnly() 方法可移除只用来检查类型创建的bean
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }
        // 获取合并后的根 BeanDefinition，BeanDefintiton 根类一样具有继承关系
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd, beanName, args);

        // 将依赖的bean提前创建
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
            for (String dependsOnBean : dependsOn) {
                getBean(dependsOnBean);
                // 注册依赖 bean
                registerDependentBean(dependsOnBean, beanName);
            }
        }
        // 创建单例实例
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, new ObjectFactory() {
                public Object getObject() throws BeansException {
                    try {
                        // 创建bean
                        return createBean(beanName, mbd, args);
                    } catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        //  创建原型实例
        else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
                beforePrototypeCreation(beanName);
                prototypeInstance = createBean(beanName, mbd, args);
            } finally {
                afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }
        //  创建自定义Scope实例, CustomScopeConfigurer / SimpleThreadScope
        else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
                throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
            }
            try {
                Object scopedInstance = scope.get(beanName, new ObjectFactory() {
                    public Object getObject() throws BeansException {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        } finally {
                            afterPrototypeCreation(beanName);
                        }
                    }
                });
                bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {}
        }
    }

    // 检查获取到的bean与参数 requiredType 是否匹配
    if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
    return (T) bean;
}
```

 `doGetBean` 方法里面调用了几个比较重要的方法：
 - getSingleton(String beanName);
 - getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd);
 - getMergedLocalBeanDefinition(String beanName);
 - getSingleton(String beanName, ObjectFactory singletonFactory);
 - createBean(String beanName, RootBeanDefinition mbd, Object[] args)。

##### 3. getSingleton
我们先看看`getSingleton(String beanName, ObjectFactory singletonFactory)`
```
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {
    synchronized (this.singletonObjects) {
        // 先从缓存中获取
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 标记当前 beanName 正在创建
            beforeSingletonCreation(beanName);
            try {
                // 调用 createBean()
                singletonObject = singletonFactory.getObject();
            } catch (BeanCreationException ex) {
                throw ex;
            } finally {
                afterSingletonCreation(beanName);
            }
            // 添加到缓存中
            addSingleton(beanName, singletonObject);
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
}
```

##### 4. createBean
```
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) throws BeanCreationException {
    // Make sure bean class is actually resolved at this point.
    // 将bean标签中class属性解析为Class类
    resolveBeanClass(mbd, beanName);

    // 检查 MethodOverrides, 主要是检查 MethodOverride 里面的 methodName 是否存在，或者是否重载
    try {
        mbd.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException();
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 给BeanPostProcessors机会返回一个代理bean, 替代掉目标bean 
        Object bean = resolveBeforeInstantiation(beanName, mbd);
        if (bean != null) {
            return bean;
        }
    } catch (Throwable ex) {
        throw new BeanCreationException();
    }
    // 创建 bean
    Object beanInstance = doCreateBean(beanName, mbd, args);
    return beanInstance;
}
```
##### 5. doCreateBean
看看真正创建Bean的方法 doCreateBean
```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    // 如果是单例，看是否存在FactoryBean缓存，这个缓存主要是在调用 isTypeMatch() 或 getType() 检查类型匹配或者获取类型后缓存起来的
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 创建Bean实例，并返回Bean的包装实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            // post-processors 可以在此修改definition
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    // 这里的目的就是解决循环依赖的问题
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, new ObjectFactory() {
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 将xml文件配置的各种属性值填充到Bean中
        populateBean(beanName, mbd, instanceWrapper);
        // 初始化bean, 顺序：postProcessBeforeInitialization() -> afterPropertiesSet() -> 
        // 自定义的 init-method -> postProcessAfterInitialization()
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable ex) {}
    
    ......

    return exposedObject;
}
```
Bean实例化的大致流程基本上就是这样了，上面提到的一些重要方法这里还没具体看，我们下次再来分析一波。


 














 