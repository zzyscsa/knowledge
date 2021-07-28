SpringCloud是什么

SpringCloud是一个开发工具集，含多个子项目。利用SpringBoot开发的便利实现很多功能，诸如服务注册与发现、负载均衡、数据监控、RESTApi发布方式



**什么是RESTful风格？**

用 URL 表示要操作的资源，用不同的 HTTP 请求（GET，POST，PUT，DELETE）描述对资源的操作，通过 HTTP 的状态码来判断此次对资源操作的结果，这就是 Restful风格。



## 注册中心Eureka

如果没有注册中心，分布式系统就像是无头苍蝇，因此注册中心一般都是集群

假如B是服务提供商，在注册中心注册了自己的服务，A是调用服务的，它有两种方式获得B的服务：

- 客户端发现（Eureka）：由A发起，注册中心告诉A所有可用的B，A自己从中挑选一个
  - 优点：简单直接
  - 缺点：必须自己实现一套逻辑把B找出来
- 服务端发现（Nginx，Zookeeper，Kubernetes）：代理帮助A从众多B中找到一个B
  - 优点：B和注册中心对于A是不可见的，A只需要发个请求找代理即可



## 应用通信Feign

Spring Cloud中服务之间通过restful方式调用，有以下两种

- RestTemplate
- <font color='red'>Feign</font>

Eureka属于客户端发现，它的负载均衡是软负载，它会向注册中心拉取可用的服务信息，然后根据负载均衡策略直接命中哪一台服务器发送请求，都是在客户端完成的，这些是通过Ribbon组件。RestTemplate、Feign、Zuul都使用到了Ribbon组件

Ribbon实现软负载均衡核心：

1. 服务发现：依据服务名字把该服务所有实例找出来
2. 服务选择规则：依据规则策略，从多个服务中选择出有效的服务
3. 服务监听：检测失效的服务，做到高效剔除

Ribbon主要组件：

1. ServerList
2. IRule
3. ServerListFilter

**流程**：首先通过ServerList获取所有可用服务列表，然后通过ServerListFilter过滤一部分地址，最后剩下的地址中通过IRule选择一个实例作为最终结果

看一看这个源码：

```Java
ServiceInstance serviceInstance = loadBalancerClient.choose("PRODUCT"); //参数是注册在eureka的服务名字
```

进入到这个choose

```Java
@Override
public ServiceInstance choose(String serviceId) {
   Server server = getServer(serviceId);
   if (server == null) {
      return null;
   }
   return new RibbonServer(serviceId, server, isSecure(server, serviceId),
         serverIntrospector(serviceId).getMetadata(server));
}
```

继续跟踪getServer()

```Java
protected Server getServer(ILoadBalancer loadBalancer) {
   if (loadBalancer == null) {
      return null;
   }
   return loadBalancer.chooseServer("default"); // TODO: better handling of key
}
```

可以看到它传入了一个ILoadBalancer类，这是一个接口，就是属于Ribbon组件里面的，它有一个方法getAllServers()就是获取所有的服务！最终这个List集合里面是服务的地址和端口。这个是在启动后会自己去注册中心获取的

```Java
public List<Server> getAllServers() {
    return Collections.unmodifiableList(allServerList);
}
```

回到getServer()的chooseServer方法

```Java
public Server chooseServer(Object key) {
    if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            return rule.choose(key);
        } catch (Exception e) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
        }
    }
}
```

主要就是看负载均衡的规则rule，默认规则就是轮询

```java
protected IRule rule = DEFAULT_RULE; 
private final static IRule DEFAULT_RULE = new RoundRobinRule();
```

### Feign

- 声明式REST客户端（伪RPC）

- 采用了基于接口的注解
- 本质还是使用HTTP客户端，通过Feign使得HTTP调用对开发者完全透明
- Feign内部也使用Ribbon做负载均衡

## Config 统一配置中心

### 为什么需要？

- 不方便维护，比如配置
- 配置内容安全与权限：线上配置不会对开发公开，尤其是数据库账号密码，需要隔离配置文件
- 更新配置项目必须重启



统一配置中心也是一个**单独的微服务**，它也有server端和client端。

server把配置从远端git上面拉下来，然后放到本地git，如果远端访问不了，就从本地git拉取。我们的微服务从配置中心拿到配置

我们的服务会首先去（配置文件为bootstrap.yml）注册中心寻找config微服务，然后通过这个这个config拉取git上面的配置文件。如果我们的服务关于注册中心eureka的配置也写在git上面，那么它默认去8761端口找eureka，如果不是8761就报错了。

关于配置的自动刷新，使用的是**`SpringCloud Bus`**，使用了这个组件后，会连上rabbitmq，如果没有配置，默认是

```java
public class RabbitProperties { 
    private String host = "localhost"; 
    private int port = 5672; 
    private String username = "guest"; 
    private String password = "guest";
```

如果有需要，就要配置

```yml
spring: 
 rabbitmq: 
  host: 
  port: 
  username: 
  password:
```

我们需要配置远程仓库的Webhooks，这样一旦仓库修改了就会给指定的url发送一个消息，相当于rabbitmq通知我们服务修改了配置

## 服务网关

要素：

- 稳定性、高可用
- 性能、并发性
- 安全性
- 扩展性

常见网关方案：

Nginx+Lua、Kong、Tyk、Spring Cloud Zuul

### Zuul

#### Zuul特点：

-  路由+过滤器 = Zuul
- 核心是一系列过滤器

#### Zuul四种过滤器API

- 前置（Pre）：限流（分布式限流可以用Sentinel）、鉴权、参数检验
- 后置（Post）：统计、记录日志
- 路由（Route）
- 错误（Error）

#### HTTP请求的生命周期

![image-20200824194528094](D:\Typora\picture\image-20200824194528094.png)

Zuul底层是HTTP转发，网关没有做协议转换，一直都是http

### Zuul高可用

- 多个Zuul结点注册到Eureka Server
- Nginx和Zuul“混搭”：Nginx对外暴露url，Nginx把请求转发到Zuul上，Nginx继续做负载均衡

## Hystrix

防雪崩利器

- 服务降级：
  - 优先核心服务，非核心服务不可用或弱可用
  - 通过HystrixCommand注解指定
  - fallbackMethod（回退函数）中具体实现降级逻辑
- 依赖隔离
- 监控
- 服务熔断：

微服务和分布式中，容错的办法通常 有两种：

1. 重试机制：预期的，短暂的故障问题，使用重试
2. 断路模式：长时间的故障问题，将受保护的服务封装在一个可以监控故障的断路器里面，当故障达到一定的值，断路器跳闸，断路对象返回错误

Circuit Breaker：断路器

断路器状态机![image-20200826160133031](D:\Typora\picture\image-20200826160133031.png)

1. Closed：熔断器关闭状态，失败次数累计到达一定的阈值，就会启动熔断机制，到达open状态
2. Open：熔断器打开状态，对服务都直接返回错误，设置了一个时钟选项，到了这个时间，会进入半熔断状态
3. Half Open：半熔断状态，允许定量的服务请求，如果调用都成功或者一定的比例成功，则认为服务恢复了，关闭熔断器，返回关闭状态，否则认为服务还没好，又回到打开状态

参数：

```Java
//服务熔断
@HystrixCommand(commandProperties = {
        @HystrixProperty(name = "circuitBreaker.enabled", value = "true"), //设置熔断
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"), //设置在时间窗口中，断路器最小请求数，至少请求10次才触发熔断
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), //时间窗口，断路器打开对主逻辑启动熔断，Hystrix会启动一个休眠时间窗，这个时间窗内降级逻辑临时成为主逻辑，当休眠时间窗到期（10s）,断路器进入半熔断状态，如果失败，重新计时
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),//断路器打开的错误百分比条件
})
```



## 链路监控sleuth

引入依赖后调用服务可以看到order服务打印如下日志

`INFO [order,698e7dcfb0805ebb,698e7dcfb0805ebb,false] 18020 -....`

product服务打印如下日志

`INFO [product,698e7dcfb0805ebb,c89f2cea51e22a6f,false] 14528 -`

第一个值代表服务名字

第二个值表示**traceid**，一条链路有唯一的ID，因此这两个服务相同

第三个值表示**spanid**，是一个基本工作单元，每个traceid会有若干个spanid，为了记录调用了哪些服务，以及每次调用的消耗时间等信息，在每次调用服务时，填入一个调用记录

第四个值true表示这个信息需要输出到其他服务进行收集展示，false表示不收集

