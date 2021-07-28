refresh()方法一直跟进会跟进到ConfigurationClassParser类中的parse()方法

```java
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
    // 加载默认的配置--->>（对springboot项目来说这里就是自动装配的入口了）
    processDeferredImportSelectors();
}
```

## @SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}
```

接着看@EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

**@Import导入了一个重要的类：AutoConfigurationImportSelector**

## AutoConfigurationImportSelector

```java
// AutoConfigurationImportSelector类
//自动装配
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
        .loadMetadata(this.beanClassLoader);
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    //获取所有的自动配置类（META-INF/spring.factories中配置的key为org.springframework.boot.autoconfigure.EnableAutoConfiguration的类）
    List<String> configurations = getCandidateConfigurations(annotationMetadata,attributes);
    configurations = removeDuplicates(configurations);
    //需要排除的自动装配类（springboot的主类上 @SpringBootApplication(exclude = {com.demo.starter.config.DemoConfig.class})指定的排除的自动装配类）
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    //将需要排除的类从 configurations remove掉
    configurations.removeAll(exclusions);
    configurations = filter(configurations, autoConfigurationMetadata);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return StringUtils.toStringArray(configurations);
}
```

看看getCandidateConfigurations()方法是如何拿到这些自动配置类的。

```java
// AutoConfigurationImportSelector类
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
     List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
             getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
     Assert.notEmpty(configurations,
             "No auto configuration classes found in META-INF/spring.factories. If you "
                     + "are using a custom packaging, make sure that file is correct.");
     return configurations;
 }
```

关键在于这个loadFactoryNames()方法

```JAVA
// SpringFactoriesLoader类
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}
```

从META-INF/spring.factories中加载的类的key是org.springframework.boot.autoconfigure.EnableAutoConfiguration

最终configurations集合里面就存着这些要加载类的名字，这些类就是上面说的这个key定义的，所以如果你自己定义一个starter，也加入到这个key中，springboot也能够扫描到。

**`所以说Starter要生效必须要在META-INF下建立spring.factories文件`**

回到selectImports方法，Set<String> exclusions = getExclusions(annotationMetadata, attributes); 是排除主类上@SpringBootApplication注解上排除的自动装配的类，比如@SpringBootApplication(exclude = {com.demo.starter.config.DemoConfig.class})，那么之后removeAll就会把自定义的这个类移除

之后就是把这些类解析成BeanDefinition并注册进去。

然后跳过refresh方法中的invokeBeanFactoryPostProcessors(beanFactory);方法，最终在beanFactory的BeanDefinitionMap中找到了自定义starter中的自动装配的类。



最后，放一个时序图来看看spring.factories配置文件如何与Spring整合

![image-20200817220057022](D:\Typora\picture\image-20200817220057022.png)

最核心的就是SpringBoot使用Spring提供的BeanDefinitionRegistryPostProcessor扩展点并实现了ConfigurationClassPostProcessor，