底层使用的是动态代理，如果代理的目标对象实现了至少一个接口，则会使用JDK动态代理。所有该目标类型实现的接口都将被代理。若该目标对象没有实现任何接口，则创建一个CGLIB 代理。也可以配置文件中强制使用CGLIB，但是CGLIB无法通知final方法，且CGLIB二进制包要放在classpath下面

开启AOP使用@EnableAspectJAutoProxy注解，这个注解核心就是向容器中注册了一个 `AnnotationAwareAspectJAutoProxyCreator` 类。

# AOP入口

Spring将AOP抽象代理创建器实现了BeanPostProcessor接口，并在bean后处理过程中向bean中织入增强。

入口在createBean方法中的resolveBeforeInstantiation。这个方法先检查是否有 `InstantiationAwareBeanPostProcessor` ，这是IOC用于组件的创建前后做后置处理的接口。如果有就给后置处理器一些机会，让他创建真正的代理对象。

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    // ......

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 给后置处理器一个机会，来生成一个代理对象，代替普通的目标对象。
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    // doCreateBean ......
}
```

最终执行**<font color='red'>AbstractAutoProxyCreator中的postProcessBeforeInstantiation方法</font>**。

## postProcessBeforeInstantiation

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    // 缓存机制
    Object cacheKey = getCacheKey(beanClass, beanName);
  
    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        // 判断该Bean是否已经被增强（advisedBeans为已经增强过的Bean）
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        // 1、2. 判断是否为基础类型（isInfrastructureClass），或者需要跳过的Bean
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }
    // 如果我们有一个自定义的TargetSource，则在此处创建代理。
 // 这段源码要抑制目标bean的不必要的默认实例化：TargetSource将以自定义方式处理目标实例。
    // 3. 自定义目标资源，对于单实例Bean必定会返回null
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    return null;
}
```

### 1. isInfrastructureClass：判断Bean是否为基础类型

这个方法会调用到父类的方法，而父类的方法会判断Bean的class是否为一些指定的类型（`Advice` 、`PointCut` 、`Advisor` 、`AopInfrastructureBean`）的子类，以及判断Bean的class上是否标注了@Aspect，如果是就返回true

### 2. shouldSkip：Bean是否需要跳过

这一步有一个很重要的操作在这里面一起进行了：<font color='red'>**创建增强器**</font>。

```java
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    // TODO: Consider optimization by caching the list of the aspect names
    // 2.1 
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    for (Advisor advisor : candidateAdvisors) {
        if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
            return true;
        }
    }
    return super.shouldSkip(beanClass, beanName);
}
```

------

#### 2.1 findCandidateAdvisors：加载增强器

```java
// AnnotationAwareAspectJAutoProxyCreator
protected List<Advisor> findCandidateAdvisors() {
    // 添加所有根据父类的规则找到的Spring的增强器
    List<Advisor> advisors = super.findCandidateAdvisors();
    // 给所有BeanFactory中的AspectJ切面构建增强器
    if (this.aspectJAdvisorsBuilder != null) {
        //2.3 
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}
// 父类
// AbstractAdvisorAutoProxyCreator
protected List<Advisor> findCandidateAdvisors() {
    Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
    // 2.2 获取、创建增强器的Bean
    return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

上面子类重写的方法也很简单，先调父类的方法取出一组增强器，再从IOC容器中找出所有标注 `@Aspect` 的组件一起添加上。

父类的方法是用一个 `advisorRetrievalHelper` 组件来调 `findAdvisorBeans` 方法：

#### 2.2 findAdvisorBeans：获取、创建增强器的Bean

这个方法把哪些类型为Advisor的bean都找出来，然后利用getBean把这些增强器创建，初始化出来，放入IOC容器中就完事了。

这个的方法返回后，要准备构建增强器了，它要调用`aspectJAdvisorsBuilder.buildAspectJAdvisors` 方法：

#### 2.3 buildAspectJAdvisors：<font color='red'>创建增强器</font>

```java
// 在当前的BeanFactory中查找带有@AspectJ注解的切面类Bean，然后返回代表它们的增强器列表。为每个AspectJ通知方法创建一个增强器
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    // 提取增强通知
    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new ArrayList<>();
                aspectNames = new ArrayList<>();
                // 获取IOC容器中的所有Bean
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                    this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) {
                        continue;
                    }
                    // 我们必须小心，不要急于实例化bean，因为在这种情况下，IOC容器会缓存它们，但不会被织入增强器
                    // 这一部分的功能是在不创建Bean的情况下获取Bean的类型，防止因为增强器还没有创建，导致对象没有被成功代理
                    Class<?> beanType = this.beanFactory.getType(beanName);
                    if (beanType == null) {
                        continue;
                    }
                    // 如果当前循环的Bean是一个切面类
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        // 包装@Aspect注解的元数据
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        // 默认使用单实例创建切面类
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            // 2.4 如果切面类是一个单实例Bean，则会缓存所有增强器
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            if (this.beanFactory.isSingleton(beanName)) {
                                this.advisorsCache.put(beanName, classAdvisors);
                            }
                            // 否则只会缓存增强器创建工厂，由增强器工厂来创建增强器
                            else {
                                this.aspectFactoryCache.put(beanName, factory);
                            }
                            advisors.addAll(classAdvisors);
                        }
                        else {
                            // Per target or per this.
                            if (this.beanFactory.isSingleton(beanName)) {
                                throw new IllegalArgumentException("Bean with name '" + beanName +
                                                                   "' is a singleton, but aspect instantiation model is not singleton");
                            }
                            MetadataAwareAspectInstanceFactory factory =
                                new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                            this.aspectFactoryCache.put(beanName, factory);
                            advisors.addAll(this.advisorFactory.getAdvisors(factory));
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    // 如果aspectNames不为null，证明之前已经创建过了，直接读缓存即可
    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<Advisor> advisors = new ArrayList<>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}
```

1. 获取IOC容器中的所有Bean
2. 从所有的Bean中找带有 `@Aspect` 注解的Bean
3. 根据Bean中定义的通知（Advice，即被五种通知类型标注的方法），创建增强器
4. 将增强器放入缓存，以备后续加载

这段代码中有一个`advisorFactory.getAdvisors`缓存Bean中的所有增强器。

#### 2.4 getAdvisors：缓存Bean中的所有增强器

这部分的核心就是中间部分的 `getAdvisor`：获取/创建增强器（可以类比**getBean**）。

#### 2.5 getAdvisor：<font color='red'>真正创建增强器</font>

```java
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
        int declarationOrderInAspect, String aspectName) {
    // 校验@Aspect注解等
    validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

    // 2.6 解析切入点
    AspectJExpressionPointcut expressionPointcut = getPointcut(
            candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
    if (expressionPointcut == null) {
        return null;
    }

    // 将切入点和通知包装成一个切面
    return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
            this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

这个方法中先是解析切入点，之后把**切入点**和**通知**包装成一个**切面**（AOP术语）。

#### 2.6  getPointcut：解析切入点

这个方法首先调用AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod根据方法来解析它使用的通知注解，然后就是包装 `AspectJExpressionPointcut` 对象了。

#### 2.7 findAspectJAnnotationOnMethod：获取切入点的注解通知类型

```java
private static final Class<?>[] ASPECTJ_ANNOTATION_CLASSES = new Class<?>[] {
    Pointcut.class, Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class};

protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
    for (Class<?> clazz : ASPECTJ_ANNOTATION_CLASSES) {
        AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) clazz);
        if (foundAnnotation != null) {
            return foundAnnotation;
        }
    }
    return null;
}
```

它会找这里面所有的注解，并确定出一个，返回去。

------

现在回到shouldSkip方法。现在返回了一些加载好的切面，它会循环这些切面，并确定正在创建的Bean的name是否与切面相同，如果发现有相同的，则代表要**<font color='red'>跳过Bean的创建</font>**。

**Debug发现postProcessBeforeInstantiation方法并不会创建一个Bean实例来，那么回到createBean方法，就直接进入doCreateBean方法。**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {
    // ......
    Object exposedObject = bean;
    try {
        populateBean(beanName, mbd, instanceWrapper);
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    // ......
}
```

进到 `initializeBean` 方法，`BeanPostProcessor` 会在这里执行 `postProcessAfterInitialization` 方法。

## postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

这里面关键的方法也是最里头的 `wrapIfNecessary` 方法：

## wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 如果需要增强，创建代理对象
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

在这部分前三个if结构都是跟之前看到的 `postProcessBeforeInstantiation` 部分一样的。

注释的那一行表示要在这里创建代理对象。下面的 `getAdvicesAndAdvisorsForBean` 方法，要先获取该类被切的增强器：

#### 1. getAdvicesAndAdvisorsForBean

```java
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```

这里就是调用`findEligibleAdvisors`方法拿之前收集好的增强器。增强器集合里面就是之前定义的before、after等等增强器。

### 2. findEligibleAdvisors

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

首先要获取所有之前收集好的增强器，然后findAdvisorsThatCanApply方法要过滤出<font color='cornflowerblue'>**可以代理的增强器**</font>（也就是能切该Bean的切面），<font color='red'>这部分就是使用**切入点表达式**与当前Bean中方法进行匹配。</font>

------

回到wrapIfNecessary方法，拿到增强器后，判断增强器不为空后，下面要调用 `createProxy` 方法来真正的创建代理对象。

------

### 3. createProxy

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName, @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    // 创建代理工厂，使用它来创建代理对象
    ProxyFactory proxyFactory = new ProxyFactory();
    // 从当前执行的AbstractAutoProxyCreator中复制一些配置
    proxyFactory.copyFrom(this);

    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 3.1 组合所有增强器
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 将增强器放入代理工厂
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 3.2 真正创建代理对象
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

这里面有两个很重要的步骤：**组合增强器 → 创建代理对象。**

#### 3.1 buildAdvisors：组合所有增强器

这部分的思路也是比较简单的，它除了用参数传递过来的一组增强器，还会去找一些通用的拦截器作为增强器，最后组合在一起封装为一组 `Advisor` 增强器对象，返回出去。

#### 3.2 getProxy：真正创建代理对象

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

分为两步。

##### 3.2.1 createAopProxy

```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}

public AopProxyFactory getAopProxyFactory() {
    return this.aopProxyFactory;
}
```

这里面最后的 return 方法又分为两步，第一步只是取到当前类成员的 `aopProxyFactory` 方法，第二步是调 `ProxyFactory` 的 `createAopProxy` 方法。

我们看看DefaultAopProxyFactory中的`createAopProxy` 方法：

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                                         "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

源码中已经很明显的看到我们熟悉的东西：<font color='red'>**jdk动态代理**</font>，和<font color='red'>**cglib动态代理**</font>了。判断逻辑也很简单，跟我们之前学习两种代理方式的工作场景时一样：<font color='green'>**如果目标对象有接口，用jdk动态代理；没有接口，用cglib动态代理**。</font>

以JDK动态代理为例，看看getProxy方法：

##### 3.2.2 【JDK】getProxy

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

看最后一步：**`Proxy.newProxyInstance`** ，这是最底层的原理，也是我们一开始学动态代理时接触的方式。