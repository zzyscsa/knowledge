## 怎么通过配置XML文件加载bean?

```java
BeanFactory bf = new XmlBeanFactory (new ClassPathResource ("beanFactoryTest.xml")); //这个写法废除了
MyTestBean bean = (MyTestBean)bf.getBean("myTestBean");
```

### 首先是XmlBeanFactory

Spring的配置文件读取通过ClassPathResource 封装，Spring中设计了一个Resource接口来封装底层资源。这个ClassPathResource 就是把配置文件封装成**Resource类型**的实例方法。

封装完后XmlBeanFactory初始化会调用构造函数里面XmlBeanDefinitionReader类型有个方法**loadBeanDefinitions(resource)，这是真正的加载资源的方法**。这个方法先对resource使用EncodedResource封装，然后从resource中获取对应的输入流InputStream并构造InputSource。通过构造InputSource实例和Reource实例继续调用最核心的函数**doLoadBeanDefinitions(inputSource,**
**encodedResource.getResource ())。**

这个函数做了三件事：

- 获取对XML文件的验证模式：验证模式就是判断xml文件是否规范的一种定义
- 加载XML文件，得到对应的<font color='cornflowerblue'>Document</font>
- 根据返回的Doucument加载Bean信息
  - 调用doRegisterBeanDefinitions方法真正开始解析！这个方法先解析profile，就是用的是哪个环境（dev?、pro?）的配置。
  - 然后调用里面的parseBeanDefinitions解析并注册BeanDefinition，当然这里面又调用了很多。

<font color='red'>BeanDefinition是配置文件\<bean>标签元素在容器内部的表示形式（承载实例）</font>，是一个接口。\<bean>标签元素拥有class、scope、lazy-init等属性，BeanDefinition则提供了相应的beanClass、scope、lazyInit属性，与之**一一对应**！配置文件中父\<bean>和子\<bean>分别对应RootBeanDefinition和ChildBeanDefinition两种实现类，没有父\<bean>就用RootBeanDefinition表示。它还有个实现类GenericBeanDefinition。

所以解析属性就是首先要创造出这个BeanDefinition实例（GenericBeanDefinition），然后解析bean信息就是给这个实例字段填充。

Spring将这些BeanDefinition注册到**BeanDefinitionRegistry**中，这个BeanDefinitionRegistry就像是Spring配置信息的内存数据库，<font color='green'>主要以map的形式保存</font>。后续直接操作读取配置信息。使用beanName作为key，BeanDefinition作为value，还可以通过alias 配置的别名作为key，有个别名映射map，get的时候冲刺那个想一次

### 现在来看getBean，这是加载Bean

#### <font color='purple'>**先来看看FactoryBean**</font>

一般来说Spring**通过反射利用bean的class属性**指定实现来实例化bean。某些特殊情况实例化bean比较复杂，需要提供非常多配置信息，这个FactoryBean工厂类接口可以让用户定制实例化bean的逻辑。



回过头来看getBean，会调用doGetBean(final String name , final Class<T> requiredType , final Object[] args ,boolean typeCheckOnly)方法，及其复杂，大概有这么几个步骤

- **<font color='cornflowerblue'>转换对应的beanName</font>**：传入的name可能不是beanName，可能是别名，需要转换
- **<font color='cornflowerblue'>尝试从缓存中加载单例</font>**：首先从singletonObjects缓存（一级）获取实例，获取不到再从earlySingletonObjects缓存（二级）中获取，不成功再次尝试从singletonFactories （三级缓存）中加载，从singletonFactories里面获取beanName对应的ObjectFactory，调用其getObject()来创建bean。并且放到earlySingletonObjects里面去，然后从singletonFactories里面remove这个ObjectFatory。（三级缓存升级到二级缓存！）

<font color='red'>**补充：这个singletonFactories 是Spring的三级缓存**</font>

```java
/** Cache of singleton objects: bean name to bean instance. */
// 用于存储完整的bean，接下来称之为【一级缓存】 bean name->bean instance
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of early singleton objects: bean name to bean instance. */
// 用于存储不完整的bean，即只是new出来，并没有属性值的bean，接下来称之为【二级缓存】
//bean name->ObjectFactory
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

/** Cache of singleton factories: bean name to ObjectFactory. */
//用于存储bean工厂对象，接下来称之为【三级缓存】
//也是保存BeanName 和创建bean 实例之间的关系与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了,其目的是用来检测循环引用!
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

- **<font color='cornflowerblue'>bean实例化</font>**：如果从缓存中得到bean的原始状态，则需要实例化getObjectForBeanlnstance
- **<font color='cornflowerblue'>原型模式依赖检查</font>**：单例情况下的循环依赖问题
- **<font color='cornflowerblue'>检测parentBeanFactory</font>**：如果当前加载的XML 配置文件中不包含beanName 所对应的配置，就只能到父类工厂parentBeanFactory去尝试下了，然后再去递归的调用getBean 方法。
- **<font color='cornflowerblue'>将存储XML 配置文件的GernericBeanDefinition 转换为RootBeanDefinition</font>**：xml读取的信息在GernericBeanDefinition 中，但后续bean处理都是针对RootBeanDefinition
- **<font color='cornflowerblue'>寻找依赖</font>**：先加载依赖的bean
- **<font color='cornflowerblue'>针对不同的scope进行bean的创建</font>**：根据不同的配置进行不同的初始化策略
- **<font color='cornflowerblue'>类型转换</font>**：通常requiredType 参数为null，如果不为空就转换成requiredType指定的类型

## 创建Bean

常规Bean实例创建用的是doCreateBean方法，大概的流程：

- 如果是单例则需要先清除缓存
- **<font color='red'>实例化Bean</font>**，调用createBeanInstance方法将BeanDefinition转换为BeanWrapper

createBeanInstance这个函数参数有一个mdb，这个mdb是BeanDefinition(bd)和父类(如果有)合并的结果，这个mdb包含了父类的一些信息。就是上面所说的RootBeanDefinition

得到这个mdb后看当前bean有没有配置depends-on依赖，如果配置了就预先实例化依赖的bean

根据bean中定义的class信息还有构造方法信息，寻找合适的构造方法，查找过程非常复杂，总之最后拿到合适的构造方法，然后通过反射来完成实例化对象的流程，这时候这个对象就创建出来了！但是还是有很多逻辑需要处理

- **<font color='red'>依赖处理</font>**
- 属性填充：populateBean函数
  - 依赖注入：提取依赖的bean，并统一存入PropertyValues中
    - 一个是autowireByName另一个是autowireByType，无外乎就是寻找BeanWrapper中需要依赖注入的属性，遍历这些属性寻找匹配的bean
  - 属性填充：applyPropertyValues函数，到这里获取了所有需要注入的bean，但是是以PropertyValues形式存在，那么现在就应用到已经实例化的bean中
- **<font color='red'>初始化bean</font>**，配置bean中有init-method属性作用就是在bean实例化，并且属性填充后调用用户设定的初始化方法，不仅如此，还有些其他工作
  - **激活Aware方法**：Spring提供了很多Aware接口，实现Aware接口的bean在初始化后能获得一些相对应的资源，比如实现BeanFactoryAware接口的bean可以获得Spring中的BeanFactory，原理就是Spring容器会自动注入BeanFactory实例，其他的也大同小异
  - **处理器的应用**：**<font color='blue'>BeanPostProcessor</font>**，使用位置就是在这里，在init-method前后会分别调用BeanPostProcessor的postProcessBeforelnitialization和postProcessAfterlnitialization方法，使用
    户可以根据自己的业务需求进行响应的处理。BeanPostProcessor是Spring的核心扩展点技术了，Spring也提供了很多这个接口的实现类，比如上面说的Aware就是Bean后处理器某一个实例完成的，再比如AOP，解析自动装配注解@Autowire等等。
- 注册DisposableBean：如果配置了destroy-method，这里需要注册以便于在销毁时候调用
- 完成创建并返回

## 循环依赖问题如何解决

### 单例构造器循环依赖

无法解决，抛出BeanCurrentlylnCreationException异常表示循环依赖。

Spring容器把每个正在创建的bean标识符放在一个“当前创建bean池”中，只有bean创建完毕才会移出去，如果创建过程中发现自己已经在这个池里面了，就会抛出异常。

**<font color='#C4C'>那么Spring如何发现这个循环依赖的呢？</font>**

用一个set保存当前线程在创建的beanName，比如先创建A，然后把A添加到set中。然后通过反射调用A构造方法，发现依赖参数是B类型，触发执行getBean(B)，发现Spring缓存中没有实例化B，然后就去构建B，同时B的构造方法又依赖A，触发getBean(A)，因为A尚未实例化完毕，缓存也没有A的实例，又触发创建A，然后添加到set中发现已经有A了，抛异常！

### setter注入循环依赖

通过Spring容器**提前暴露**刚完成构造器注入但未完成其他步骤（如setter注入）的bean来完成的，而且只能解决单例作用域的bean循环依赖。通过提前暴露一个单例工厂方法，从而使其他bean能引用到
该bean。

#### 细说一下这个ObjectFactory

```java
// 解决循环依赖的关键步骤
// earlySingletonExposure：是否”提前暴露“原始对象的引用
// 因为不论这个bean是否完整，他前后的引用都是一样的，所以提前暴露的引用到后来也指向完整的bean
boolean earlySingletonExposure = (mbd.isSingleton() &&this.allowCircularReferences &&isSingletonCurrentlyInCreation(beanName));
// 如果需要提前暴露单例bean，则将该bean工厂放入【三级缓存】中
if (earlySingletonExposure) {
    // 将刚创建的bean工厂放入三级缓存中singleFactories(key是beanName，value是FactoryBean)
    // 同样也会移除【二级缓存】中对应的bean，即便没有
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));  }
```

一开始的doCreateBean这个函数中有earlySingletonExposure这么一个变量，这个变量是判断是否要提前曝光的，如果这个**BeanDefinition是单例**、**允许循环依赖**、**这个beanName正在创建**，三个条件满足就执行addSingletonFactory方法。举一个AB循环依赖的例子：

![image-20200811221527575](D:\Typora\picture\image-20200811221527575.png)

创建A的时候会把beanA的工厂加入缓存，关键是在getBean(A)的时候，不会直接实例化A，而是检查缓存中是否有已经创建好对应的bean或者已经创建好的ObjectFactory，此时A的ObjectFactory早以创建好，直接调用ObjectFactory创建A。

所以就是B创建要依赖A，就用缓存中ObjectFactory工厂提供的方法来中断A的属性填充，使得B中持有仅仅刚开始初始化而未填充属性的A，然后A属性填充自然也可以通过B中的A获取，因为是单例的！

**<font color='dfdenghdfvn'>总结来说处理循环依赖：</font>**

1. 初始化 Bean 之前，将这个 bean 的 name 放入**三级缓存**

2. 创建 Bean 将准备创建的 Bean 放入 **singletonsCurrentlyInCreation** （正在创建的 Bean ）

3. `createNewInstance`方法执行完后执行`addSingletonFactory`，将这个实例化但没有属性赋值的 Bean**放入三级缓存，并从二级缓存中移除**

   > 一般情况下初次创建的 bean 不会存在于二级缓存，故该步骤可以简单理解为仅仅是放入了三级缓存而已

4. 属性赋值&自动注入时，引发关联创建

5. 关联创建时：

   1. 检查“正在被创建的 Bean ”中是否有即将注入的 Bean
   2. 如果有，检查二级缓存中是否有当前创建好但没有赋值初始化的 Bean
   3. 如果没有，检查三级缓存中是否有正在创建中的 Bean
   4. 至此一般会有，将这个 Bean **放入二级缓存，并从三级缓存中移除**

6. 之后 Bean 被成功注入，最后执行 `addSingleton`，将这个完全创建好的Bean**放入一级缓存，从二级缓存和三级缓存移除**，并记录已经创建了的单实例Bean

### prototype(原型)作用范围的依赖处理

```xml
<bean id="hello" class="com.scsa.pojo.Hello" scope="prototype">
    <property name="str" value="hello spring!"/>
</bean>
```

没法处理，Spring容器不缓存“prototype”作用域的bean，无法提前暴露一个创建中的bean。

**<font color='#C4C'>那么Spring如何发现这个循环依赖的呢？</font>**

prototype作用域是在每次get的时候创建新的对象，而不是从容器中拿。

Spring为了发现循环依赖问题，专门在容器上提供了一个threadLocal类型的set，用来记录当前线程正在创建的beanName，每个对象开始执行创建逻辑的时候都会把对应的beanName放到这个set中，如果A对象执行到依赖注入环节，那么这个set就有A字符串，那么A依赖B，Spring先检查缓存中是否有B的实例可以用，如果没有就去根据B的BeanDefinition创建B的实例，处理到B的依赖注入的时候，发现B依赖A，Spring同样检查缓存是否有A实例可以用，但是Spring使用getBean(A)获取A对象的时候，发现BeanDefinition表示A是prototype类型，需要创建新的，又走上创建A实例的流程，在创建之前会检查threadLocal-set是否有A字符串，如果有 ，循环依赖！抛异常！



**<font color='#C4C'>补充：为什么需要三级缓存，不能直接用二级缓存？</font>**

1. 一级缓存：singletonObjects（创建好的单例bean）
2. 二级缓存：earlySingletonObjects（早期暴露的实例，还未初始化以及属性注入）
3. 三级缓存：singletonFactories（早期暴露的一个单例工厂，二级缓存中存的就是这个工厂中获取的对象）

如果没有AOP，那么三级缓存和二级缓存没有什么差别。如果结合了AOP，A的ObjectFactory给B注入的是A的**代理对象**！这个三级缓存这个工厂目的在于**延迟对实例化阶段生成的对象的代理**，只有真正发生循环依赖的时候，才提前创建代理对象，否则只会创建一个工厂并放入三级缓存，而不会通过这个工厂去创建真正对象。

即使没有循环依赖，都会把这个工厂添加进三级缓存，Spring也不确定到底有没有循环依赖。假如这里直接使用二级缓存，那么意味着所有的Bean都要马上生成代理放入二级缓存，这不仅没必要而且违背Spring在结合AOP跟Bean生命周期的设计！

如果有三级缓存

创建A->实例化A->为A注入属性->开始创建B->实例化B->为B注入属性->**为A创建代理**->初始化B->结束创建B->初始化A->结束创建A

如果没有三级缓存

创建A->实例化A->**为A创建代理**->为A注入属性->开始创建B->实例化B->为B注入属性->初始化B->结束创建B->初始化A->结束创建A

## 使用ApplicationContext方式加载xml

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
Hello hello = (Hello) context.getBean("hello");
```

其中有一个refresh函数最为重要

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1.设置context属性，验证必要属性是否缺失
        prepareRefresh();

        // 2.  1.createBeanFactory 2.loadBeanDefinitions(beanFactory)
        //这里是在子类中启动 refreshBeanFactory() 的地方
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3.设置beanfactory的一些属性
        prepareBeanFactory(beanFactory);

        try {
            // 4.由子类实现 BeanFactory 初始化后置处理
            postProcessBeanFactory(beanFactory);

            //5.注册并调用BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6.注册BeanPostProcessors
            registerBeanPostProcessors(beanFactory);

            // 7.Initialize message source for this context.
            initMessageSource();

            // 8.初始化上下文事件广播器
            initApplicationEventMulticaster();

            // 9.由子类实现初始化其他特殊的bean
            onRefresh();

            // 10.查找listener并注册为bean，然后注册到广播器中
            registerListeners();

            // 11.初始化非延迟加载单例，预先实例化
            finishBeanFactoryInitialization(beanFactory);

            // 12.发布响应事件
            finishRefresh();
```

这是常用的加载方式，BeanFactory默认就是使用XmlBeanFactory加载bean，但是ApplicationContext扩展的BeanFactory的功能，它的大概流程

- 环境准备：例如对系统属性及环境变量的初始化及验证。

- **加载BeanFactory**：ApplicationContext是对BeanFactory功能的扩展，因此它包含其所有功能。
  - 增加@Qualifier 与＠Autowired注解支持
  - 加载BeanDefinition，和之前套路一样
  
- 现在完成了配置文件的解析，进入prepareBeanFactory函数，扩展功能，主要有几个方面
  - 增加SqEL语言支持
  - 增加属性注册编辑器，可以自定义属性编辑器，不然原来DI注入Date类型无法识别
  - 添加ApplicationContextAwareProcessor处理器：其中注册了BeanPostProcessor，ApplicationContextAwareProcessor就是使得实现Aware接口的bean初始化后可以取得对应资源
  - 设置忽略依赖：实现Aware接口的bean要忽略了，不是普通的bean了
  
- **BeanFactory后处理**

- **初始化非延迟加载单例**（预先实例化）

  ApplicationContext 实现的默认行为就是在启动时将所有单例bean提前进行实例化。因为这样在配置中的任何错误就会即刻被发现，也可提升运行效率。也就是说容器启动完毕后我们代码get出来的bean都是缓存拿过来的，除非配置的是懒加载（比如一些非常耗费资源的）。

## BeanFactory后处理

对BeanFactory的扩展

### 激活调用注册的BeanFactoryPostProcessor

Bean容器后处理器是在实例化单例前执行

这个接口和BeanPostProcessor类似，可以对bean的定义（配置元数据）进行处理，也就是说IOC容器允许BeanFactoryPostProcessor在容器实际实例化任何其他bean之前读取配置元数据，并有可能修改它。它的作用范围是容器级的。它有一个接口方法，可以将当前容器的引用传递进去，拿到这个引用它能做的事情就不受限制了，因为到这一步单例实例化还未执行，可以手动修改移除现有的bd信息。使用的话可以写到配置文件里面去

### 注册BeanPostProcessor

注册了才能在实例化的时候使用！Bean后处理器是在实例化每个实例过程中穿插执行

### 初始化ApplicationEventMulticaster事件广播器

如果用户没有自定义事件广播器，默认就使用ApplicationEventMulticaster

事件广播器会在Spring生命周期内，接收相应的事件并广播。这里面**存放着监听器**，产生Spring事件的时候就遍历所有监听器，使用监听器中的方法对事件处理。比如某个BeanDefinition信息处理完后，也就是创建出对应的单实例保存到缓存以后，会发出事件到事件广播器里，广播器把事件广播出去监听器就收到了，就可以做一些事情。

监听者只要写java类，实现Spring监听者接口，xml中配置这个bean就可以提供给事件广播器了。Spring容器启动过程中会根据这些bd信息实例化后注册到广播器里

## 单例Bean是线程安全的吗？如何保证线程安全

Spring中默认的bean作用域是单例，不是线程安全的

Spring使用ThreadLocal将共享变量给每个线程一个副本，变为独享的，有状态的。

## 注入的三种方式

- 构造方法注入
- setter注入
- 注解注入

## Spring容器中bean是什么时候被实例化的

 第一：如果你使用BeanFactory作为Spring Bean的工厂类，则所有的bean都是在第一次使用该Bean的时候实例化 



 第二：如果你使用ApplicationContext作为Spring Bean的工厂类，则又分为以下几种情况： 
    （1）：如果bean的scope是singleton的，并且lazy-init为false（默认是false，所以可以不用设置），则 ApplicationContext启动的时候就实例化该Bean，并且将实例化的Bean放在一个map结构的缓存中，下次再使 用该 Bean的时候，直接从这个缓存中取 
    （2）：如果bean的scope是singleton的，并且lazy-init为true，则该Bean的实例化是在第一次使用该Bean的时候进 行实例化 
    （3）：如果bean的scope是prototype的，则该Bean的实例化是在第一次使用该Bean的时候进行实例化 

## Spring bean构造器注入是怎么选择的？

spring中查找构造器是使用determineCandidateConstructors方法完成的

如果没有@Autowired注解，若构造器只有一个，那么把此构造器作为适用的构造器返回出去，使用此构造器进行实例化。若构造器只有两个，且存在无参构造器，将直接使用无参构造器初始化。若大于两个构造器，将返回一个空集合，也就是没有找到合适的构造器，将会使用无参构造器进行实例化。此时若没有无参构造器会抛出异常，实例化Bean失败。

如果有@Autowired注解，如果只有一个构造器有@Autowired注解，那么就用这个构造器；如果有两个构造器@Autowired注解（required=true），那么抛异常，如果一个是false一个是true，也抛异常；如果两个为false，那么这两个都会被放入candidates集合中返回，还会把默认无参构造器也放入。然后Spring会遍历集合里面的构造器，从参数多的开始依次遍历。如果参数不是Spring的Bean（是否被IOC容器管理），跳过下一个，如果不满足，那就选无参构造器。