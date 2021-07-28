```java
@SpringBootApplication
public class SellApplication {
    public static void main(String[] args) {
        SpringApplication.run(SellApplication.class, args);
    }
}
```

启动无外乎就两步：1. 创建SpringApplication对象。2.run运行

# 创建SpringApplication对象

- 根据classpath下是否存在(ConfigurableWebApplicationContext)判断是否要启动一个web applicationContext。
- SpringFactoriesInstances加载classpath下所有可用的ApplicationContextInitializer
- SpringFactoriesInstances加载classpath下所有可用的ApplicationListener

实例化SpringApplication，并不像平时的Spring Components一样通过注解和扫包完成，而是通过一种不依赖Spring上下文的加载方法，这样才能在Spring完成启动前做各种配置。Spring的解决方法是以接口的全限定名作为key，实现类的全限定名作为value记录在项目的`META-INF/spring.factories`文件中，然后通过`SpringFactoriesLoader`工具类提供静态方法进行类加载并缓存下来。

```kotlin
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
     // 将传入的SellApplication启动类放入primarySources中，这样应用就知道主启动类在哪里，叫什么了
    // SpringBoot一般称呼这种主启动类叫primarySource（主配置资源来源）
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //推断应用类型，后面会根据类型初始化对应的环境。常用的一般都是servlet环境
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //实例化初始化器 ，从类路径下找到META-INFO/spring.factories配置中所有ApplicationContextInitializer；保存到initializers集合中
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //实例化监听器，找到所有ApplicationListener，保存listeners集合中
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //根据调用栈，推断出 main 方法的类名，即主配置类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

比如Spring Boot Starter给被人使用时，就可以使用Factories机制

## 判断应用类型

webApplicationType是个枚举，有三种类型

- WebApplicationType.REACTIVE: classpath下存在org.springframework.web.reactive.DispatcherHandler
- WebApplicationType.SERVLET: classpath下存在javax.servlet.Servlet或者org.springframework.web.context.ConfigurableWebApplicationContext
- WebApplicationType.NONE: 不满足以上条件。

## 设置初始化器

setInitializers方法会将一组类型为 `ApplicationContextInitializer` 的初始化器放入 `SpringApplication` 中。

而这组 **`ApplicationContextInitializer`**，是在构造方法中，通过 `getSpringFactoriesInstances` 得到的。

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[]{});
}

/**
 * 通过指定的classloader 从META-INF/spring.factories获取指定的Spring的工厂实例
 * @param type
 * @param parameterTypes
 * @param args
 * @param <T>
 * @return
 */
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
                              Class<?>[] arameterTypes, Object... args) {
   ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    //通过指定的classLoader从 META-INF/spring.factories 的资源文件中，
    //读取 key 为 type.getName() 的 value
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //通过反射创建这些组件的实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
                                                       classLoader, args, names);
    //对Spring工厂实例排序（org.springframework.core.annotation.Order注解指定的顺序）
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

loadFactoryNames() 这个方法很重要，这个方法是spring-core中提供的从META-INF/spring.factories中获取指定的类（key）的统一入口方法。比如在这里，获取的是key为 org.springframework.context.ApplicationContextInitializer 的类。

`ApplicationContextInitializer`是Spring框架的类**, 这个类的**主要目的**就是在  ConfigurableApplicationContext **调用refresh()方法之前，**回调**这个类的initialize方法。它是在IOC容器之前的回调。

使用方式有三种：

1. 主启动类上手动添加
2. application.properties中配置
3. spring.factories中配置

通过 ConfigurableApplicationContext 的实例**获取容器的环境Environment**，从而实现对配置文件的修改完善等工作。

## 设置监听器

`ApplicationListenr `是spring的事件监听器，典型的观察者模式，通过 ApplicationEvent 类和 ApplicationListener 接口，可以实现对spring容器全生命周期的监听

他的加载套路和之前初始化器一样

## 确定主配置类

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

源码很简单，从 `deduceMainApplicationClass` 方法开始往上爬，哪一层调用栈上有main方法，方法对应的类就是主配置类，就返回这个类。

**<font color='cornflowerblue'>既然main方法中已经将主类传入SpringApplication.run了，为什么还要进行主类推断呢？</font>**

SpringApplication.run方法所传入的是@SpringBootApplication注解标注的类，是primarySources 所对应的类。main方法是启动类，他们可以不一样，只不过我们通常将二者合并到一个类中。

# Run方法

```Java
public ConfigurableApplicationContext run(String... args) {
    //计时器，统计任务的执行时间
   StopWatch stopWatch = new StopWatch();
    //开始执行
   stopWatch.start();
    // 创建空的IOC容器，和一组异常报告器
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 配置与awt相关的信息；主要作用是即使没有检测到显示器也允许其继续启动
   configureHeadlessProperty();
     // 1. 获取SpringApplicationRunListeners启动事件监听器（回调机制）
   SpringApplicationRunListeners listeners = getRunListeners(args);
    // 封装成SpringApplicationEvent事件然后广播出去给SpringApplication中的listeners所监听
    //【回调】首次启动run方法时立即调用。可用于非常早期的初始化（准备运行时环境之前）。
   listeners.starting();
   try {
       // 将main方法的args参数封装到一个对象中
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
       // 2. 构造应用上下文环境
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
       //处理需要忽略的Bean，若有配置spring.beaninfo.ignore，则将该配置设置进系统参数
      configureIgnoreBeanInfo(environment);
       // 打印banner图形
      Banner printedBanner = printBanner(environment);
       // 3、创建ApplicationContext容器！！！！
      context = createApplicationContext();
      // 初始化异常报告器
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
       //4. 初始化容器
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
       //5. 容器上下文刷新！！！
      refreshContext(context);
      afterRefresh(context, applicationArguments);
       // 停止计时器
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
       //发布容器启动完成事件
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
```

## 1. 获取SpringApplicationRunListeners

从META-INF/spring.factories获取<font color='red'>SpringApplicationRunListener</font>的集合，并依次调用的SpringApplicationRunListener的starting方法，监听springApplication的启动.

## 2. 加载SpringBoot应用上下文环境

应用上下文环境包括什么呢？包括计算机的环境，Java环境，Spring的运行环境，Spring项目的配置（在SpringBoot中就是那个熟悉的application.properties/yml）等等。

```java
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 2.1 创建运行时环境。比如web就创建StandardServletEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置运行时环境。根据用户配置，配置 environment系统环境（比如启动哪一个Application配置文件）
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 【回调】SpringApplicationRunListener的environmentPrepared方法（Environment构建完成，但在创建ApplicationContext之前）
    listeners.environmentPrepared(environment);
    // 环境与应用绑定。也就是把配置内容绑定到指定的属性配置类中（类似于 @ConfigurationProperties）
    bindToSpringApplication(environment);
    if (this.webApplicationType == WebApplicationType.NONE) {
        environment = new EnvironmentConverter(getClassLoader())
            .convertToStandardEnvironmentIfNecessary(environment);
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

方法中主要完成的工作，首先是创建并**按照相应的应用类型配置相应的环境**，然后根据用户的配置，配置系统环境，然后启动监听器，并加载系统配置文件。

### 2.1 创建运行时环境

根据当前的应用运行环境类型，创建不同的 `Environment` 。默认 SpringBoot 环境下会创建 `StandardServletEnvironment` 。

```Java
private ConfigurableEnvironment getOrCreateEnvironment() {
   if (this.environment != null) {
      return this.environment;
   }
    // 判断当前Web应用类型
   switch (this.webApplicationType) {
   case SERVLET:
           //web
      return new StandardServletEnvironment();
   case REACTIVE:
      return new StandardReactiveWebEnvironment();
   default:
      return new StandardEnvironment();
   }
}
```

## 3. 创建应用上下文（容器）

应用上下文可以理解成IoC容器的高级表现形式，应用上下文确实是在IoC容器的基础上丰富了一些高级功能。应用上下文对IoC容器是持有的关系。他的一个属性beanFactory就是IoC容器（DefaultListableBeanFactory）。所以他们之间是持有，和扩展的关系。

这里我们就创建了IoC容器DefaultListableBeanFactory

```Java
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
        //  根据Web应用类型决定实例化哪个IOC容器
         switch (this.webApplicationType) {
         case SERVLET: //应用程序应作为基于servlet的web应用程序运行，并应启动嵌入式servlet web（tomcat）服务器。
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE: //应用程序应作为 reactive web应用程序运行，并应启动嵌入式 reactive web服务器。
           contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Unable create a default ApplicationContext, "
                     + "please specify an ApplicationContextClass",
               ex);
      }
   }
    return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
}
```

可以发现都是创建的基于Annotation的 **ApplicationContext**。

SpringBoot项目有三种对应的应用上下文，比如我们的最常用的web项目，对应的上下文为**<font color='yebne'>AnnotationConfigServletWebServerApplicationContext</font>**，通过反射让contextClass得到Class

<font color='red'>我们的beanFactory就是在这个AnnotationConfigServletWebServerApplicationContext继承的类GenericApplicationContext中定义的！**DefaultListableBeanFactory就是IoC容器真面目！**</font>

```JAVA
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
    private final DefaultListableBeanFactory beanFactory;
    ...
    public GenericApplicationContext() {
        this.beanFactory = new DefaultListableBeanFactory();
    }
    ...
}
```

BeanUtils.instantiateClass(contextClass) 这个方法中，不但初始化了AnnotationConfigServletWebServerApplicationContext类，也就是我们的上下文context，同样也触发了GenericApplicationContext类的构造函数，从而IoC容器也创建了。

## 4.初始化容器prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context,
                            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
                            ApplicationArguments applicationArguments, Banner printedBanner) {
    // 将创建好的应用环境设置到IOC容器中
    context.setEnvironment(environment);
    // IOC容器的后置处理
    postProcessApplicationContext(context);
    //执行容器中的 ApplicationContextInitializer。这个方法会获取到所有 Initializer，调用initialize方法。而这些 Initializer，其实就是刚创建 SpringApplication 时准备的那些 ApplicationContextInitializer。
    applyInitializers(context);
    //【回调】SpringApplicationRunListeners的contextPrepared方法（在创建和准备ApplicationContext之后，但在加载之前）
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // Add boot specific singleton beans
    //将main函数中的args参数封装成单例Bean，注册进容器
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    //将 printedBanner 也封装成单例，注册进容器
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // Load the sources
    // 加载主启动类，它要加载 primarySources，就是之前设置的著启动类。
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 4.1 注册主启动类
    load(context, sources.toArray(new Object[0]));
    //【回调】SpringApplicationRunListeners的contextLoaded方法（ApplicationContext已加载但在刷新之前）, 发布容器已加载事件
    listeners.contextLoaded(context);
}
```

### 4.1  注册主启动类

```Java
protected void load(ApplicationContext context, Object[] sources) {
   if (logger.isDebugEnabled()) {
      logger.debug(
            "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
   }
   //创建 BeanDefinitionLoader 
   BeanDefinitionLoader loader = createBeanDefinitionLoader(
         getBeanDefinitionRegistry(context), sources);
   // 设置BeanName生成器，通过Debug发现此时它还没有被注册
   if (this.beanNameGenerator != null) {
      loader.setBeanNameGenerator(this.beanNameGenerator);
   }
   // 设置资源加载器
   if (this.resourceLoader != null) {
      loader.setResourceLoader(this.resourceLoader);
   }
   // 设置运行环境
   if (this.environment != null) {
      loader.setEnvironment(this.environment);
   }
   loader.load();
}
```

#### getBeanDefinitionRegistry

在createBeanDefinitionLoader之前先getBeanDefinitionRegistry，这个做的工作就是**返回IOC容器**

#### createBeanDefinitionLoader

拿到IOC容器之后就进入createBeanDefinitionLoader方法

```java
protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {
    return new BeanDefinitionLoader(registry, sources);
}
```

直接new了一个 `BeanDefinitionLoader` 。看看 `BeanDefinitionLoader` 的构造方法都干了什么

```java
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    // 注册BeanDefinition解析器
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    this.xmlReader = new XmlBeanDefinitionReader(registry);
    if (isGroovyPresent()) {
        this.groovyReader = new GroovyBeanDefinitionReader(registry);
    }
    this.scanner = new ClassPathBeanDefinitionScanner(registry);
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```

这里面发现了几个关键的组件：`AnnotatedBeanDefinitionReader`（**注解驱动**的Bean定义解析器）、`XmlBeanDefinitionReader`（**Xml定义**的Bean定义解析器）、`ClassPathBeanDefinitionScanner`（**类路径下**的Bean定义扫描器），还有一个我们不用的 `GroovyBeanDefinitionReader`（它需要经过isGroovyPresent方法，而这个方法需要判断classpath下是否有 `groovy.lang.MetaClass` 类）。

从基础源（包括XML和JavaConfig）加载bean定义。

#### load

主要是这个load方法，跟进去看的话有很多层，这里就不贴出来了。主要做了个什么事情呢？

<font color='bblse'>**它把主启动类封装成AnnotatedGenericBeanDefinition然后注册到DefaultListableBeanFactory（IOC容器）的beanDefinitonMap中**</font>

那么在刷新容器前，SpringApplication 会根据前面确定好的应用类型，创建对应的IOC容器。IOC容器在刷新之前会进行初始化、加载主启动类等预处理工作。

## 5. 容器上下文刷新

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1.设置context属性，验证必要属性是否缺失
        prepareRefresh();

        // 2. 1.createBeanFactory 2.loadBeanDefinitions(beanFactory)
        //这里是在子类中启动 refreshBeanFactory() 的地方
        //获取BeanFactory，加载所有bean的定义信息（未实例化）
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3.设置beanfactory的一些属性
        prepareBeanFactory(beanFactory);

        try {
            // 4.准备 BeanFactory 初始化后置处理，可以修改BeanDifinition信息
            postProcessBeanFactory(beanFactory);

            // 5.执行BeanFactory创建后的后置处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6.注册Bean的后置处理器 
            registerBeanPostProcessors(beanFactory);

            // 7.初始化MessageSource
            initMessageSource();

            // 8.初始化上下文事件广播器
            initApplicationEventMulticaster();

            // 9.由子类实现初始化其他特殊的bean
            onRefresh();

            // 10.注册监听器
            registerListeners();

            // 11.初始化非延迟加载单例，所有剩下的单例Bean
            finishBeanFactoryInitialization(beanFactory);

            // 12.完成容器的创建工作，发布响应事件
            finishRefresh();
```
### 2. 获取BeanFactory，加载所有bean的定义信息

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 2.1 刷新BeanFactory
    refreshBeanFactory();
    return getBeanFactory();
}
```

obtainFreshBeanFactory()拿到之前创建的DefaultListableBeanFactory容器，而且还刷新BeanFactory

```JAVA
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    //这个CAS表示GenericApplicationContext只允许刷新一次，之前beanFactory就是在GenericApplicationContext里面创建的
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}
```

逻辑很简单，只是设置了 `BeanFactory` 的序列化ID而已。

### 3. prepareBeanFactory：BeanFactory的预处理配置

```java
// Configure the bean factory with context callbacks.
// 3.1 配置一个可回调注入ApplicationContext的BeanPostProcessor
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

这里面引入了一个组件**`BeanPostProcessor`**

它通常被称为 **“Bean的后置处理器”**，它的作用在文档注释中也描述的差不多，**它可以在对象实例化但初始化之前，以及初始化之后进行一些后置处理**。BeanPostProcessor 的切入时机：

![img](https://user-gold-cdn.xitu.io/2019/11/5/16e3bb3c836be8e1?imageslim)

<font color='red'>**这个代码就是注册了ApplicationContextAwareProcessor后置处理器。**</font>

ApplicationContextAwareProcessor是BeanPostProcessor 实现，它将 ApplicationContext 传递给实现 `EnvironmentAware`，`EmbeddedValueResolverAware`，`ResourceLoaderAware`，`ApplicationEventPublisherAware`，`MessageSourceAware` 和/或 `ApplicationContextAware` 接口的bean。

按照上面提到的顺序满足已实现的接口。

IOC容器将自动在其基础bean工厂中注册它。应用程序不直接使用它。

### 4、5. BeanFactory的后置处理、执行BeanFactory创建后的后置处理器

这两个步骤主要完成了**IoC容器初始化**的三个步骤

- **第一步：Resource定位**

在SpringBoot中，我们都知道他的包扫描是从**主类所在的包（所以说启动类要放在最外面啊）**开始扫描的，prepareContext()方法中，会先将**主类解析成BeanDefinition**，然后在refresh()方法的invokeBeanFactoryPostProcessors()方法中解析主类的BeanDefinition获取basePackage的路径。这样就完成了定位的过程。其次SpringBoot的各种starter是通过SPI扩展机制实现的自动装配，SpringBoot的**自动装配同样也是在invokeBeanFactoryPostProcessors()方法中实现的**。还有一种情况，在SpringBoot中有很多的@EnableXXX注解，细心点进去看的应该就知道其底层是@Import注解，在invokeBeanFactoryPostProcessors()方法中也实现了对该注解指定的配置类的定位加载。

常规的在SpringBoot中有三种实现定位，第一个是主类所在包的，第二个是SPI扩展机制实现的自动装配（比如各种starter），第三种就是@Import注解指定的类。（对于非常规的不说了）

- **第二步：BeanDefinition的载入**

在第一步中说了三种Resource的定位情况，定位后紧接着就是BeanDefinition的分别载入。所谓的载入就是通过上面的定位得到的basePackage，SpringBoot会将该路径拼接成：classpath*:org/springframework/boot/demo/**/*.class这样的形式，然后一个叫做PathMatchingResourcePatternResolver的类会将该路径下所有的.class文件都加载进来，然后**遍历判断是不是有@Component注解**，如果有的话，就是我们要装载的BeanDefinition。

- **第三步：注册BeanDefinition**

这个过程通过调用上文提到的BeanDefinitionRegister接口的实现来完成。这个注册过程把载入过程中解析得到的BeanDefinition向IoC容器进行注册。通过上文的分析，我们可以看到，在IoC容器中将BeanDefinition注入到一个ConcurrentHashMap中，IoC容器就是通过这个HashMap来持有这些BeanDefinition数据的。比如DefaultListableBeanFactory 中的beanDefinitionMap属性。

**`来看代码怎么实现的吧.......`**

从invokeBeanFactoryPostProcessors开始跟踪调用栈

```JAVA
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    ...
}
// PostProcessorRegistrationDelegate类
public static void invokeBeanFactoryPostProcessors(
    ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    ...
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    ...
}
// PostProcessorRegistrationDelegate类
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

    for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanDefinitionRegistry(registry);
    }
}
// ConfigurationClassPostProcessor类
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    ...
        processConfigBeanDefinitions(registry);
}
// ConfigurationClassPostProcessor类
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ...
        do {
            parser.parse(candidates);
            parser.validate();
            ...
        }
    ...
}
```

**最终来到了ConfigurationClassPostProcessor类的parse()方法**

```JAVA
// ConfigurationClassParser类
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<>();
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            // 如果是SpringBoot项目进来的，bd其实就是前面主类封装成的 AnnotatedGenericBeanDefinition（AnnotatedBeanDefinition接口的实现类）
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            } else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        } catch (BeanDefinitionStoreException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }
    // 加载默认的配置-->>（对springboot项目来说这里就是自动装配的入口了）
    processDeferredImportSelectors();
}
```

继续沿着parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());方法跟下去

```JAVA
// ConfigurationClassParser类
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
    processConfigurationClass(new ConfigurationClass(metadata, beanName));
}
// ConfigurationClassParser类
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    ...
    // Recursively process the configuration class and its superclass hierarchy.
    //递归地处理配置类及其父类层次结构。
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        //递归处理Bean，如果有父类，递归处理，直到顶层父类
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
// ConfigurationClassParser类
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    // Recursively process any member (nested) classes first
    //首先递归处理内部类，（SpringBoot项目的主类一般没有内部类）
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    // 针对 @PropertySource 注解的属性配置处理
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        } else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    // 根据 @ComponentScan 注解，扫描项目中的Bean（SpringBoot 启动类上有该注解）告诉SpringBoot去哪里扫描Bean
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            // 立即执行扫描，（SpringBoot项目为什么是从主类所在的包扫描，这就是关键了）
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                // 检查是否是ConfigurationClass（是否有configuration/component两个注解），如果是，递归查找该类相关联的配置类。
                // 所谓相关的配置类，比如@Configuration中的@Bean定义的bean。或者在有@Component注解的类上继续存在@Import注解。
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    //递归处理 @Import 注解（SpringBoot项目中经常用的各种@Enable*** 注解基本都是封装的@Import）
    processImports(configClass, sourceClass, getImports(sourceClass), true);

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
```

**主要看doProcessConfigurationClass()方法，这是SpringBoot的包扫描的入口方法！！**

第60行的parse(bdCand.getBeanClassName(), holder.getBeanName()); 会进行递归调用parse方法，因为当Spring扫描到需要加载的类会进一步判断每一个类是否满足是@Component/@Configuration注解的类，如果满足会递归调用parse()方法，查找其相关的类。

68行processImports(configClass, sourceClass, getImports(sourceClass), true);通过@Import注解查找到的类同样也会递归查找其相关的类。

42行Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable( sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class); 解析主类上的@ComponentScan注解，之后会解析该注解并进行包扫描

49行Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName()); 

```JAVA
// ComponentScanAnnotationParser类
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
       componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
    ...
        // 根据 declaringClass （如果是SpringBoot项目，则参数为主类的全路径名）
        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(declaringClass));
        }
    ...
        // 根据basePackages扫描类
        return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

这个declaringClass是我们主类的全路径名，然后basePackages集合只有一个，就是主类所在路径，既然只有一个，为什么用集合？因为还可以用@ComponentScan指定扫描路径。至此Resource完成

那么Spring如何进行扫描的？进入doScan()方法

```JAVA
// ComponentScanAnnotationParser类
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // 从指定的包中扫描需要装载的Bean
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                    AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                //将该 Bean 注册进 IoC容器（beanDefinitionMap）
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

该方法使用`Set<BeanDefinition> candidates = findCandidateComponents(basePackage);`	从basePackage中扫描类并解析成BeanDefinition，拿到所有符合条件的类后使用`registerBeanDefinition(definitionHolder, this.registry);`将该类注册进IoC容器。这个方法完成了BeanDefinition的载入，和BeanDefinition的注册。这就完了

```JAVA
// ClassPathScanningCandidateComponentProvider类
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    ...
        else {
            return scanCandidateComponents(basePackage);
        }
}
// ClassPathScanningCandidateComponentProvider类
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        //拼接扫描路径，比如：classpath*:org/springframework/boot/demo/**/*.class！！！！！
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        //从 packageSearchPath 路径中扫描所有的类
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        //遍历扫描出来的类
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    // //判断该类是不是 @Component 注解标注的类，并且不是需要排除掉的类！！！！！！
                    if (isCandidateComponent(metadataReader)) {
                        //将该类封装成 ScannedGenericBeanDefinition（BeanDefinition接口的实现类）类！！！！
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setResource(resource);
                        sbd.setSource(resource);
                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            candidates.add(sbd);
                        } else {
                            if (debugEnabled) {
                                logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        }
                    } else {
                        if (traceEnabled) {
                            logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    }
                } catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                        "Failed to read candidate component class: " + resource, ex);
                }
            } else {
                if (traceEnabled) {
                    logger.trace("Ignored because not readable: " + resource);
                }
            }
        }
    } catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```

那么这个findCandidateComponents(basePackage)方法正是将basePackage拼接成classpath*:org/springframework/boot/demo/**/*.class，然后扫描，并且判断是否是@Component 注解标注的类，解析成ScannedGenericBeanDefinition。该类是BeanDefinition接口的实现类

```JAVA
protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
}
```

注册BeanDefinition！完工！呃，针对SpringBoot包扫描定位方式的BeanDefinition的定位，加载，和注册过程是这样的。当然还有通过@Import导入的，前面也说了调用了processImports()方法

### 11. finishBeanFactoryInitialization：初始化单实例Bean

这里是Bean的初始化过程

其中有一个preInstantiateSingletons方法比较重要，调用了DefaultListableBeanFactory的preInstantiateSingletons方法。

从<font color='red'>**beanDefinitionNames拿到了所有的BeanNames**</font>，通过names获得**<font color='red'>beanDefinition的bean定义信息</font>**。直接调用**<font color='red'>getBean方法</font>**。源码太长，不放了。说一下做了哪些事情：

<font color='cornflowerblue'>第一</font>：别名-BeanName的映射

<font color='cornflowerblue'>第二</font>：getSingleton：尝试获取单实例Bean（解决循环依赖）：先尝试从之前实例化好的Bean中找有没有这个Bean，如果找到，说明已经被实例化了，可以直接返回。IOC容器会对单实例Bean单独存储

<font color='cornflowerblue'>第三</font>：创建前检查，可以检查是否有循环依赖

<font color='cornflowerblue'>第四</font>：标记准备创建的Bean，主要为了防止多线程同时创建。IOC容器会把所有创建过的Bean的name都存起来。

<font color='cornflowerblue'>第五</font>：合并BeanDefinition，处理显式依赖（@DependsOn的依赖），依赖也是通过getBean创建。

<font color='cornflowerblue'>第六</font>：准备创建Bean。getSingleton就是先从已经加载好的单实例Bean缓存区中获取是否有当前BeanName的Bean。如果没有就createBean创建，创建出来后放入缓存中（一级缓存）。如果当前准备创建的Bean还没有在IOC容器中，就标记一下它正在被创建，这也是解决循环依赖的，可以通过它知道这个bean是否也正在创建，如果正在创建就从二级缓存中找，没有再从三级缓存中找：

```java
// Create bean instance.
// 作用域为singleton，单实例Bean，创建
if (mbd.isSingleton()) {
    // 匿名内部类执行完成后的getSingleton调用
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
```

之后就调用**<font color='green'>createBean</font>**了。这里面有两个很重要的部分：<font color='bule'>**AOP的入口**，**真正创建Bean的入口（doCreateBean）**。</font>也太长了，看看做了什么

<font color='cornflowerblue'>第一</font>：调用createBeanInstance，创建bean对象

<font color='cornflowerblue'>第二</font>：如果这个Bean是一个单实例Bean&&IOC容器允许循环依赖&&正在创建的单实例Bean对象中有当前的这个Bean，那么调用addSingletonFactory，把Bean工厂放入缓存（<font color='red'>**三级缓存**</font>），解决循环引用

<font color='cornflowerblue'>第三</font>：调用populateBean，属性赋值和自动注入。包括@Autowired和setter两种注入流程。

#### **<font color='bjiefj'>怎么通过@Autowired自动注入呢？</font>**

**`AutowiredAnnotationBeanPostProcessor`**这个后置处理器就是实现@Autowired的自动注入。

关键的回调方法 `postProcessProperties` ：

```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

第一句就是构建自动注入的元数据。

第一步会先从缓存中取，如果没有就会构建自动装配的元数据metadata

buildAutowiringMetadata方法通过反射遍历当前类的属性，判断上面是否有`@Autowired` 等类型的注解。并且循环遍历父类寻找。

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
    // Fall back to class name as cache key, for backwards compatibility with custom callers.
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    // Quick check on the concurrent map first, with minimal locking.
    // 首先从缓存中取，如果没有才创建
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                // 构建自动装配的信息
                metadata = buildAutowiringMetadata(clazz);
                // 放入缓存
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }
    return metadata;
}
```

然后之后的 `inject` 方法是真正的自动注入。

```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Collection<InjectedElement> checkedElements = this.checkedElements;
    Collection<InjectedElement> elementsToIterate =
        (checkedElements != null ? checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        for (InjectedElement element : elementsToIterate) {
            if (logger.isTraceEnabled()) {
                logger.trace("Processing injected element of bean '" + beanName + "': " + element);
            }
            element.inject(target, beanName, pvs);
        }
    }
}
```

最底下调了 `element.inject` 方法，这个方法有两个子类重写了这个方法，分别是 `AutowiredFieldElement` 和 `AutowiredMethodElement` 。很明显它们是给属性注入和方法注入的。我们以属性注入为例。

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;
    Object value;
    // 如果这个值在前面的注入中有缓存过，直接取缓存
    if (this.cached) {
        value = resolvedCachedArgument(beanName, this.cachedFieldValue);
    }
    else {
        // 没有缓存，要在下面的try块中利用BeanFactory处理依赖关系
        DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
        desc.setContainingClass(bean.getClass());
        Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
        Assert.state(beanFactory != null, "No BeanFactory available");
        TypeConverter typeConverter = beanFactory.getTypeConverter();
        try {
            // 【关联创建】value应该被找出 / 创建出
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
        }
        synchronized (this) {
            // 处理完成后要对这个属性进行缓存
            if (!this.cached) {
                if (value != null || this.required) {
                    this.cachedFieldValue = desc;
                    // 把这个依赖的Bean添加到BeanFactory的依赖关系映射上缓存起来
                    registerDependentBeans(beanName, autowiredBeanNames);
                    if (autowiredBeanNames.size() == 1) {
                        String autowiredBeanName = autowiredBeanNames.iterator().next();
                        if (beanFactory.containsBean(autowiredBeanName) &&
                            beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                            this.cachedFieldValue = new ShortcutDependencyDescriptor(
                                desc, autowiredBeanName, field.getType());
                        }
                    }
                }
                else {
                    this.cachedFieldValue = null;
                }
                this.cached = true;
            }
        }
    }
    // 如果找到 / 创建好了value，就给它注入
    if (value != null) {
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);
    }
}
```

里面通过`beanFactory.resolveDependency`这个方法关联创建被依赖的Bean。

这个方法里面调用`doResolveDependency`方法来解决依赖。主要是先做一些匹配操作，比如数组，集合等，什么Bean都匹配不到就创建关联，就是调用getBean。

创建好后回到inject方法最后，利用反射注入属性值。

<font color='cornflowerblue'>第四</font>：调用initializeBean，初始化Bean。**在初始化前执行了后置处理器**。

![img](https://img2018.cnblogs.com/blog/1066538/201909/1066538-20190902000437259-1068766043.png)

Bean的生命周期：

- 实例化：从BeanDefinationMap中读取bean属性，通过反射读取构造方法创建实例，对应方法：`AbstractAutowireCapableBeanFactory`中的`createBeanInstance`方法
- 初始化
  - 属性填充：对应方法：`AbstractAutowireCapableBeanFactory`的`populateBean`方法
  - 初始化：包括将初始化好的bean放入Spring缓存中，填充我们预设的属性进一步做后置处理等，对应方法：`AbstractAutowireCapableBeanFactory`的`initializeBean`
- 使用和销毁

------

- Bean容器找到配置文件中 Spring Bean 的定义。
- Bean容器利用Java Reflection API创建一个Bean的实例。
- 如果涉及到一些属性值 利用set方法设置一些属性值。
- 如果Bean实现了BeanNameAware接口，调用setBeanName()方法，传入Bean的名字。
- 如果Bean实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。
- 如果Bean实现了BeanFactoryAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。
- 与上面的类似，如果实现了其他*Aware接口，就调用相应的方法。
- 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessBeforeInitialization()方法
- 如果Bean实现了InitializingBean接口，执行afterPropertiesSet()方法。
- 如果Bean在配置文件中的定义包含init-method属性，执行指定的方法。
- 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessAfterInitialization()方法
- 当要销毁Bean的时候，如果Bean实现了DisposableBean接口，执行destroy()方法。
- 当要销毁Bean的时候，如果Bean在配置文件中的定义包含destroy-method属性，执行指定的方法。