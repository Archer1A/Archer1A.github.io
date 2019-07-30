
# Spring 创建bean的流程（一）
因创建bean的代码复杂，本文探讨bean创建的部分流程，其他流程会后续发布
## 一. Spring 中获取bean

Spring 获取bean的方式是调用getBean（String name），那么其内部的运行是怎么样的呢?

```java
    @Test

public void contextLoads()throws Exception {

    ApplicationContext applicationContext =new AnnotationConfigApplicationContext(BeanConfig.class);

    Color color = (Color) applicationContext.getBean("color");

}
```

## 二. getBean(String beanName) 
1. getBean 是 BeanFacotry.class 的接口定义的一个方法
2. AbstractBeanFactory 实现了BeanFactory接口
3. AbstractBeanFactory 的getBean方法调用了doGetBean 方法

BeanFacotry.class:
```java

ObjectgetBean(String name)throws BeansException;
```
AbstractBeanFactory.class:
```java

@Override

public ObjectgetBean(String name)throws BeansException {

    return doGetBean(name, null, null, false);

}
```

## 三. doGetBean
上一步看到了 getBean并没有可用代码 而是委托给了doGetBean方法,我们看看doGetBean的逻辑吧

总结一下流程：

    1. 提取对应的beanName
    2. 检查缓存或者实例工厂中是否存在对应的实例 若有调用getObjectForBeanInstance后直接返回bean
    3. 若无则开始创建bean 
    4. 检查是否存在循环依赖，只有单例模式下才会尝试解决单例模式
    5. 获取当前的parentBeanFactory ，如果beanDefinitionMap中也就是所有已加载的类中不包括beanName 则尝试从parentBeanFactory中检测
    6. 将GenericBeanDefinition 转化为RootBeanDefinition 如果指定beanName 是子bean的话同时回合并父类的相关属性
    7. 若存在依赖则递归实例化依赖bean
    8. 实例化后根据不同的scope（singleton/prototype/自定义）实例化bean
    9. 再次调用getObjectForBeanInstance 后返回bean
```java
/**
     * Return an instance, which may be shared or independent, of the specified bean.
     * @param name the name of the bean to retrieve
     * @param requiredType the required type of the bean to retrieve
     * @param args arguments to use when creating a bean instance using explicit arguments
     * (only applied when creating a new instance as opposed to retrieving an existing one)
     * @param typeCheckOnly whether the instance is obtained for a type check,
     * not for actual use
     * @return an instance of the bean
     * @throws BeansException if the bean could not be created
     */
    @SuppressWarnings("unchecked")
    protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
            @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
        // 提取对应的beanName
        final String beanName = transformedBeanName(name);
        Object bean;
        // Eagerly check singleton cache for manually registered singletons.
        //检查缓存或实例工厂中是否有对应的实例
        //因为创建单例bean的时候会存在依赖注入，而在创建依赖的时候为了避免循环依赖
        //Spring 创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提早曝光
        // 也就是将ObjectFactory 加入到缓存中，一旦下一个bean创建的时候需要依赖上一个bean则直接使用ObjectFactory
        // 直接尝试从singletonFactories 中的objectFactory中获取
        Object sharedInstance = getSingleton(beanName);

        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            // 返回对应的实例对象，有时候存在诸如BeanFactory 的情况并不是直接返回实例本身 而是返回指定方法返回的实例
            //1. 对factoryBean正确性校验
            //2. 对非factoryBean 不做任何处理
            //3. 对bean进行转换
            //4. 将从factory中解析bean的工作交给getObjectFromFactoryBean
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }

        else {
            // Fail if we're already creating this bean instance:
            // We're assumably within a circular reference.
            // 只有单例模式情况下才会尝试解决循环依赖，原型模式情况下如果存在
            // A中有B的属性，B中有A的属性，那么依赖注入的时候就会产生A还未创建完成的时候
            //就会产生当A还未创建完成的时候因为对于B的创建再次返回创建A 造成循环依赖
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // Check if bean definition exists in this factory.
            BeanFactory parentBeanFactory = getParentBeanFactory();
            // 如果beanDefinitionMap 中也就是所有已加载的类中不包括beanName 则尝试从parentBeanFactory中监测
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    // 递归查找
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else if (requiredType != null) {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
                else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }
            //如果不是仅仅做类型检查 则是创建bean，这里要进行记录
            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                //将GenericBeanDefinition 转化为RootBeanDefinition 如果指定beanName 是子bean的话同时回合并父类的相关属性
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);

                // Guarantee initialization of beans that the current bean depends on.
                String[] dependsOn = mbd.getDependsOn();
                // 若存在依赖则需要递归实例化依赖的bean
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        // 缓存依赖调用
                        registerDependentBean(dep, beanName);
                        try {
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }

                // Create bean instance.
                // 实例化依赖的bean后便可以实例化mdb本身了
                // singleton 模式的创建
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }

                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    //prototype 的方式创建
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }

                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // Check if required type matches the type of the actual bean instance.
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
```
## 四. getObjectForBeanInstance 
在每个流程中都有调用getObjectForBeanInstance方法  那么getObjectForBeanInstance有什么作用呢？
    1. 对factoryBean正确性校验
    2. 对非factoryBean 不做任何处理
    3. 对bean进行转换
    4. 将从factory中解析bean的工作交给getObjectFromFactoryBean
```java
/**
     * Get the object for the given bean instance, either the bean
     * instance itself or its created object in case of a FactoryBean.
     * @param beanInstance the shared bean instance
     * @param name name that may include factory dereference prefix
     * @param beanName the canonical bean name
     * @param mbd the merged bean definition
     * @return the object to expose for the bean
     */
    protected Object getObjectForBeanInstance(
            Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

        // Don't let calling code try to dereference the factory if the bean isn't a factory.
        //如果指定的name是工厂相关（&为前缀）且beanInstance 又不是factoryBean 则验证不通过
        if (BeanFactoryUtils.isFactoryDereference(name)) {
            if (beanInstance instanceof NullBean) {
                return beanInstance;
            }
            if (!(beanInstance instanceof FactoryBean)) {
                throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
            }
        }

        // Now we have the bean instance, which may be a normal bean or a FactoryBean.
        // If it's a FactoryBean, we use it to create a bean instance, unless the
        // caller actually wants a reference to the factory.
        //现在我们有个bean 实例，它可能是一个普通的bean或factoryBean
        // 如果它是factoryBean，使用它创建一个bean实例，
        // 但是如果用户想要直接获取工厂实例而不是工厂的getObject方法对应的实例那么传入的name应该加&前缀
        if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
            return beanInstance;
        }

        Object object = null;
        if (mbd == null) {
            //尝试从缓存中获取bean
            object = getCachedObjectForFactoryBean(beanName);
        }
        if (object == null) {
            // Return bean instance from factory.
            // 前面过滤了不是FactoryBean  到这里beanInstance 一定为FactoryBean
            FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
            // Caches object obtained from FactoryBean if it is a singleton.
            //在所有的加载类中监测是否定义beanName
            if (mbd == null && containsBeanDefinition(beanName)) {
                // GenericBeanDefinition 转化为rootBeanDefinition
                // 如果指定beanName 是子bean的话同时合并父类的相关属性
                mbd = getMergedLocalBeanDefinition(beanName);
            }
            // 是否是用户定义的而不是应用程序本身的
            boolean synthetic = (mbd != null && mbd.isSynthetic());
            object = getObjectFromFactoryBean(factory, beanName, !synthetic);
        }
        return object;
    }

```
### getObjectFromFactoryBean
这个方法逻辑很清晰，根据不同的scope 从FactoryBean中获取实例

```java
/**
     * Obtain an object to expose from the given FactoryBean.
     * @param factory the FactoryBean instance
     * @param beanName the name of the bean
     * @param shouldPostProcess whether the bean is subject to post-processing
     * @return the object obtained from the FactoryBean
     * @throws BeanCreationException if FactoryBean object creation failed
     * @see org.springframework.beans.factory.FactoryBean#getObject()
     */
    protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        // 如果是单例模式
        if (factory.isSingleton() && containsSingleton(beanName)) {

            synchronized (getSingletonMutex()) {
                // 尝试从缓存中获取
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    // 获取实例
                    object = doGetObjectFromFactoryBean(factory, beanName);
                    // Only post-process and store if not put there already during getObject() call above
                    // (e.g. because of circular reference processing triggered by custom getBean calls)
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    }
                    else {
                        if (shouldPostProcess) {
                            if (isSingletonCurrentlyInCreation(beanName)) {
                                // Temporarily return non-post-processed object, not storing it yet..
                                return object;
                            }
                            beforeSingletonCreation(beanName);
                            try {
                                // 使用后置处理器
                                object = postProcessObjectFromFactoryBean(object, beanName);
                            }
                            catch (Throwable ex) {
                                throw new BeanCreationException(beanName,
                                        "Post-processing of FactoryBean's singleton object failed", ex);
                            }
                            finally {
                                afterSingletonCreation(beanName);
                            }
                        }
                        if (containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }
                }
                return object;
            }
        }
        else {
            Object object = doGetObjectFromFactoryBean(factory, beanName);
            if (shouldPostProcess) {
                try {
                    object = postProcessObjectFromFactoryBean(object, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
                }
            }
            return object;
        }
    }
```
## 最后
本文暂时讲bean创建的第一个分支，后续流程在其他文章中讲述。
遗留个问题  开头讲到的BeanFatory和最后讲到的FactoryBean两个有什么区别呢？？？












































