![img](https://camo.githubusercontent.com/e11a2ff9575abc290657ba3fdbff5d36f1594e7add67a72e0eda32e449508eef/68747470733a2f2f647562626f2e6170616368652e6f72672f696d67732f6172636869746563747572652e706e67)

调用关系说明：

1. 服务容器负责启动，加载，运行服务提供者。
2. 服务提供者在启动时，向注册中心**<font color='red'>注册</font>**自己提供的服务。
3. 服务消费者在启动时，向注册中心**<font color='red'>订阅</font>**自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于**长连接**推送变更数据给消费者。
5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟8. 发送一次统计数据到监控中心。

重要知识点总结：

- 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
- 监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示
- 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
- **注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者**
- 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，**消费者在本地缓存了提供者列表**
- 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者
- 服务提供者无状态，任意一台宕掉后，不影响使用
- 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

# 如何服务发现？如何知道服务提供者宕机？

zookeeper提供了“**心跳检测**”功能（不懂是leader发给follower 还是 follower 发给leader，不过好像说是客户端发给server），它会定时向各个服务提供者发送一个请求（实际上建立的是一个 socket 长连接），如果长期没有响应，服务中心就认为该服务提供者已经“挂了”，并将其剔除，比如100.19.20.02这台机器如果宕机了，那么zookeeper上的路径就会只剩/HelloWorldService/1.0.0/100.19.20.01:16888。

服务消费者会去监听相应路径（/HelloWorldService/1.0.0），一旦路径上的数据有任务变化（增加或减少），**zookeeper都会通知服务消费方服务提供者地址列表已经发生改变**，从而进行更新。

# 为什么使用zookeeper？

是满足CP的，最终一致性。**而到底是用AP还是CP，是由业务决定的。**比如你是一个文件上传的服务器，用户可能上传几个g的文件，那么如果用一个AP的系统，拿到的可能是不可用的节点，这样返回给客户端重试，客户端肯定得疯掉，这时候就需要用CP。

比如现在zk里存的两个节点的ip，分别是ipA ipB，现在zk发现ipB不可用了，于是把ipB删掉，假设ZK是AP的，无法保存数据一致性，并且有两台ZK的机器，其中一台已经把ipB删掉了，另一台没删，这时候客户端过来拿ip，有可能拿到不可用的ipB. 这里的节点不可用里的不可用，和AP的A 可用性，指的不是一个对象。

而像 rpc 调用，调用失败了重试就好，成本代价都不大，这时候，用AP可能会更合适。

Zookeeper的数据模型很简单，是一个多叉树的结构，有一系列被称为**ZNode的数据节点组成**，与传统的磁盘文件系统不同的是，zk将全量数据存储在内存中，可谓是高性能，而且支持集群，可谓高可用，另外支持事件监听。进行服务注册，实际上就是在zookeeper中创建了一个znode节点，该节点存储了该服务的IP、端口、调用方式(协议、序列化方式)等。该节点承担着最重要的职责，它由服务提供者(发布服务时)创建，以供服务消费者获取节点中的信息，从而定位到服务提供者真正网络拓扑位置以及得知如何调用。他的结点分为两种类型，一种是**临时节点**，当客户端和服务端断开连接后，所创建的Znode(节点)**会自动删除**。一种是持久结点，当客户端和服务端断开连接后，所创建的Znode(节点)**不会删除**

<font color='cornflowerblue'>以及他的watcher机制，可以监听Znode节点的**数据变化**以及监听子节点的**增减变化**</font>

**`ZooKeeper和Redis一样，也是C/S结构(分成客户端和服务端)`**

zookeeper优势

1.当提供程序意外停止时，注册表服务器可以自动删除其信息。
2.注册表服务器重新启动时，可以自动恢复所有注册数据和订阅请求。
3.会话过期后，可以自动恢复所有注册数据和订阅请求。

redis做注册中心

使用Redis的Key/Map结构存储数据。
    主Key为服务名和类型。
    Map中的Key为URL地址。
    Map中的Value为过期时间

redis是通过发布订阅模式完成消息的通知 但是存在几个问题:

1.服务的非自然下线需要监护中心来维护
2.redis做注册中心服务器时间必需同步，否则出现时间不对被强制过期(删除key)!
3.zookeeper支持监听，redis不支持，因此需要客户端启动多个线程进行订阅监听，对服务器有一定压力

# Dubbo工作原理

![image-20210302155150464](D:\Typora\picture\image-20210302155150464.png)

各层说明：

- 第一层：service层，接口层，给服务提供者和消费者来实现的
- 第二层：config层，配置层，主要是对dubbo进行各种配置的
- 第三层：**proxy层，服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton**
- 第四层：**registry层，服务注册层，负责服务的注册与发现**
- 第五层：**cluster层，集群层，封装多个服务提供者的路由以及负载均衡，将多个实例组合成一个服务**
- 第六层：monitor层，监控层，对rpc接口的调用次数和调用时间进行监控
- 第七层：**<font color='red'>protocol层</font>，远程调用层，封装rpc调用**
- 第八层：**exchange层，信息交换层，封装请求响应模式，同步转异步**
- 第九层：**transport层，网络传输层，抽象mina和<font color='red'>netty</font>为统一接口**
- 第十层：**serialize层，数据序列化层。网络传输需要**

**对于上述各层之间关系的描述，如下所示：**

- 在 RPC 中，Protocol 是核心层，也就是只要有Protocol + Invoker + Exporter 就可以完成非透明的RPC调用，然后在Invoker的主过程上Filter拦截点。
- 图中的 Consumer 和Provider是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用Client和Server的原因是Dubbo在很多场景下都使用Provider、Consumer、Registry、Monitor划分逻辑拓普节点，保持统一概念。
- 而Cluster是外围概念，所以Cluster的目的是将多个Invoker伪装成一个Invoker，这样其它人只要关注Protocol层Invoker即可，加上Cluster或者去掉Cluster对其它层都不会造成影响，因为只有一个提供者时，是不需要Cluster的。
- Proxy层封装了所有接口的透明化代理，而在其它层都以Invoker为中心，只有到了暴露给用户使用时，才用Proxy将Invoker转成接口，或将接口实现转成Invoker，也就是去掉Proxy层RPC是可以Run的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
- 而Remoting 实现是Dubbo协议的实现，如果你选择RMI协议，整个Remoting都不会用上，Remoting内部再划为Transport传输层和Exchange信息交换层，Transport层只负责单向消息传输，是对Mina、Netty、Grizzly的抽象，它也可以扩展UDP传输，而Exchange层是在传输层之上封装了Request-Response语义。
- Registry和Monitor实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。

# 如何进行服务调用

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpysOH9vvW1SMXroibXRCiaJz3u5xjBRTDRn4nT8ZAhmxSE6T3fLvOuybibDzx2sttWv615UV7rbeOLwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![image-20210302172927536](D:\Typora\picture\image-20210302172927536.png)

上图中，蓝色的表示与业务有交互，绿色的表示只对Dubbo内部交互。上述图所描述的调用流程如下：

- 服务提供方发布服务到服务注册中心；
- 服务消费方从服务注册中心订阅服务；
- 服务消费方调用已经注册的可用服务；

将上面抽象的调用流程图展开，详细如图所示：

![image-20210302173025998](D:\Typora\picture\image-20210302173025998.png)

![img](https://img-blog.csdn.net/20170813145153166?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3V6aGVuZ2ZlaTExMTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 暴露\注销服务的流程

针对服务提供方而言的。

![image-20210302174055098](D:\Typora\picture\image-20210302174055098.png)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpz82arDLk6S32wLdibQBnsiblwxBsUibsTPN5VM0JQXzbzcHiaibhVjgFtSyAmiaLelVPtBFibtzGmmZD3BA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



总的而言服务暴露的过程起始于 Spring IOC 容器刷新完成之时，具体的流程就是根据配置组装得到 URL，然后根据 URL 的参数来进行本地或者远程调用。

通过 proxyFactory.getInvoker，利用javassist 动态代理封装 ref (你写的服务真实实现类)，统一暴露出 Invoker 使得调用方便，屏蔽底层实现细节，然后根据URL参数选择对应的协议封装成 exporter 存储到一个Map中，等待消费者的调用，并且会将 URL 注册到注册中心，使得消费者可以获取服务提供者的信息。

在第一次暴露的时候会调用 createServer 来创建 Server，默认是 NettyServer。

### invoker是什么

在 Dubbo 中 invoker 其实就是一个具有调用功能的对象，在服务暴露端封装的就是真实的服务实现，把真实的服务实现封装一下变成一个 invoker。

在服务引入端就是从注册中心得到服务提供者的配置信息，然后一条配置信息对应封装成一个 invoker，这个 invoker 就具备远程调用能力，当然要是走的是 injvm 协议那真实走的还是本地的调用。

然后还有个 ClusterInvoker ，它也是个 invoker ，它封装了服务引入生成的 invoker 们，赋予其集群容错等能力，这个 invoker 就是暴露给消费者调用的 invoker。

所以说 Dubbo 就是搞了个统一模型，将能**调用的服务的对象都封装成 invoker**。

### 为什么要封装成 invoker

至于为什么要**封装成 invoker 其实就是想屏蔽调用的细节，统一暴露出一个可执行体**，这样调用者简单的使用它，向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

### 为什么要搞个本地暴露呢

因为可能存在同一个 JVM 内部引用自身服务的情况，因此**暴露的本地服务在内部调用的时候可以直接消费同一个 JVM 的服务避免了网络间的通信**。

## 订阅\取消服务的流程

服务的引入时机有两种，第一种是饿汉式，第二种是懒汉式。

饿汉式就是加载完毕就会引入，懒汉式是只有当这个服务被注入到其他类中时启动引入流程，默认是懒汉式。

![image-20210302174135641](D:\Typora\picture\image-20210302174135641.png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrpk1g5xYFUKsBNiaXQPibnYicl8bhiawpzKlgloDmoAUEZCPSpTDiaWPSkzg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过配置组成 URL ，如果有注册中心那么会构建 RegistryDirectory 向注册中心注册自己的信息，然后订阅注册中心相关信息，得到远程 `provider`的 ip 等信息，封装 Invoker，再通过`netty`客户端进行连接。

并且通过`directory` 和  `cluster` 进行底层多个服务提供者的屏蔽、容错和负载均衡等，最终得到封装好的 `invoker`再通过动态代理封装得到代理类，让接口调用者无感知的调用方法。



### <font color='red'>怎么进行动态发现？</font>

dubbo zk 注册中心采用是事件通知与客户端拉取方式。服务第一次订阅的时候将会拉取对应目录下全量数据，然后在订阅的节点注册一个 watcher。一旦目录节点下发生任何数据变化，zk 将会通过 watche r 通知客户端。客户端接到通知，将会重新拉取该目录下全量数据，并重新注册 watcher。利用这个模式，dubbo 服务就可以就做到服务的动态发现。

### 怎么注销服务？

## 代理对象如何调用

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzRZ8zcIIAfAouFVhHEibwPfhlxzLSibIkJhiaIe9hVOngIlhIjOgq39CXh4L7SQ0yhsouBREO95RAgQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先客户端调用接口的某个方法，实际调用的是代理类，代理类会通过 cluster 从 directory 中获取一堆 invokers(如果有一堆的话)，然后进行 router 的过滤（其中看配置也会**添加 mockInvoker 用于服务降级**），然后再通过 SPI 得到 loadBalance 进行一波负载均衡。

这里要强调一下默认的 cluster 是 FailoverCluster ，会进行容错重试处理。

现在我们已经得到要调用的远程服务对应的 invoker 了，此时根据具体的协议构造请求头，然后将参数根据具体的序列化协议序列化之后构造塞入请求体中，再通过 NettyClient 发起远程调用。

服务端 NettyServer 收到请求之后，根据协议得到信息并且反序列化成对象，再按照派发策略派发消息，默认是 All，扔给业务线程池。

业务线程会根据消息类型判断然后得到 serviceKey 从之前服务暴露生成的 exporterMap 中得到对应的 Invoker ，然后调用真实的实现类。

最终将结果返回，因为请求和响应都有一个统一的 ID， 客户端根据响应的 ID 找到存储起来的 Future， 然后塞入响应再唤醒等待 future 的线程，完成一次远程调用全过程。

# Java序列化框架对比

|                | **优点**                 | 缺点                                         |
| -------------- | ------------------------ | -------------------------------------------- |
| **Kryo**       | 速度快，序列化后体积小   | 跨语言支持较复杂                             |
| **Hessian**    | 跨语言，占用空间小       | 较慢，但是比jdk好                            |
| **Protostuff** | 速度快，占空间小，跨语言 | 需要进行环境搭建安装                         |
| **FastJson**   | 速度快，使用简单         |                                              |
| **Java**       | 使用方便，可序列化所有类 | 速度慢，占空间，不适用于网络传输，无法跨语言 |
| **Thrift**     | 二进制，高效             | 向后兼容有一定的约定限制                     |

java原生反射加锁耗时，而且他可能一个对象要用多个字节表示，而**Hessian**中Integer就用I表示。

**<font color='red'>Java序列化</font>**会把要序列化的对象类的**元数据和业务数据**全部序列化为字节流，而且是把**整个继承**关系上的东西全部序列化了。它序列化出来的字节流是对那个对象结构到内容的完全描述，包含所有的信息，因此**效率较低而且字节流比较大**。但是由于确实是序列化了所有内容，所以可以说什么都可以传输，因此也更可用和可靠。

**<font color='red'>Hessian</font>**基于Http协议进行传输。（1）把本地格式的数据编码为二进制数据，仅用一个字符作为结构化标记，减少了编码、解码操作对内存的占用量。（2）采用定长的字节记录值。用定长的字节记录值，解码时，就可以使用位操作从固定长度的位获得值。这样不仅操作简单，而且可以获得较高的性能。（3）采用引用取代重复遇到的对象。使用引用取代重复遇到的对象可以避免对重复对象的编码，而且也减少了编码后的数据量。它是把对象序列化为二进制流的形式在http信道中传输，那么对于安全性高的应用不应该采用hessian(比如网上支付等)。缺乏安全机制，传输没有加密处理。

它的实现机制是**着重于数据，附带简单的类型信息的方法**。就像Integer a = 1，hessian会序列化成I 1这样的流，I表示int or Integer，1就是数据内容。而对于**复杂对象，通过Java的反射机制，hessian把对象所有的属性当成一个Map来序列化**，产生类似M className propertyName1 I 1 propertyName S stringValue（大概如此，确切的忘了）这样的流，包含了基本的类型描述和数据内容。而**在序列化过程中，如果一个对象之前出现过，hessian会直接插入一个R index这样的块来表示一个引用位置，从而省去再次序列化和反序列化的时间**。这样做的代价就是hessian需要对不同的类型进行不同的处理（因此hessian直接偷懒不支持short），而且遇到某些特殊对象还要做特殊的处理（比如StackTraceElement）。而且同时因为并没有深入到实现内部去进行序列化，所以在某些场合会发生一定的不一致，比如通过Collections.synchronizedMap得到的map。



<font color='red'>**Fastjson**</font>是一个Java语言编写的高性能的JSON处理器,由阿里巴巴公司开发。无依赖，不需要例外额外的jar，能够直接跑在JDK上。**FastJson在复杂类型的Bean转换Json上会出现一些问题，可能会出现引用的类型，导致Json转换出错，需要制定引用，会有漏洞。**FastJson采用独创的算法，将parse的速度提升到极致，超过所有json库。



为什么<font color='red'>**Kryo**</font>序列化这么快？因为Kryo是一个无协议的序列化包，用起来定制性很强；PB有的优化Kryo基本都有；特别的是它把Class注册为一个数字，再注册一个序列化器，这样序列化时直接使用用户自定义的序列化器，输出内容格式顺序大小完全由用户控制，传输时只要一个数字代替class。

Kryo序列化后比Hessian小很多，由于Kryo没有将类field的描述信息序列化，所以Kryo需要以自己加载该类的filed。这意味着如果该类没有在kryo中注册，或者该类是第一次被kryo序列化时，kryo需要时间去加载该类（hessian优于kryo），如果该类已经被kryo加载过，那么kryo保存了其类的信息，就可以很快的将byte数组填入到类的field中，而hessian则需要解析序列化后的byte数组中的field信息，对于序列化过的类，kryo优于hessian。hessian使用了固定长度存储int和long，而kryo则使用的变长，实际中，很大的数据不会经常出现。(kryo优于hessian)，他通过最后一位的特殊操作而非写入长度来标记字段（x70）的范围。

它也有一个致命的弱点：生成的byte数据中部包含field数据，对类升级的兼容性很差！所以，若用kryo序列化对象用于C/S架构的话，两边的Class结构要保持一致。

# Dubbo解决TCP粘包半包问题

**当发生TCP拆包问题时候**

TCP的拆包和粘包主要是decoder部分，在`NettyCodecAdapter`中的decoder是由`InternalDecoder`来实现这里假设之前还没有发生过任何数据交互，系统刚刚初始化好，那么这个时候在InternalDecoder里面的**buffer**属性会是EMPTY_BUFFER。当发生第一次inbound数据的时候，第一次在InternalDecoder里面接收的肯定是dubbo消息头的部分（这个由TCP协议保证），由于发生了拆包情况，那么此时接收的inbound消息可能存在一下几种情况

1、当前inbound消息只包含dubbo协议头的一部分

2、当前inbound消息只包含dubbo的协议头

3、当前inbound消息只包含dubbo消息头和部分playload消息

通过上面的讨论，我们知道发生上面三种情况，都会触发ExchangeCodec返回NEED_MORE_INPUT，由于在DubboCountCodec对余返回NEED_MORE_INPUT会回滚读索引，所以此时的buffer里面的数据可以当作并没有发生过读取操作，并且DubboCountCodec的decode也会返回NEED_MORE_INPUT，在InternalDecoder对于当判断返回NEED_MORE_INPUT，也会进行读索引回滚，并且退出循环，最后会执行finally内容，这里会判断inbound消息是否还有可读的，由于在DubboCountCodec里面进行了读索引回滚，所以次数的buffer里面是完整的inbound消息，等待第二次的inbound消息的到来，当第二次inbound消息过来的时候，再次经过上面的判断。

**当发生TCP粘包的时候**
当发生粘包的时候是tcp将一个以上的dubbo协议栈放在一个tcp包中，那么有可能发生下面几种情况

1、当前inbound消息只包含一个dubbo协议栈

2、当前inbound消息包含一个dubbo协议栈，同时包含部分另一个或者多个dubbo协议栈内容

如果发生只包含一个协议栈，那么当前buffer通过ExchangeCodec解析协议之后，当前的buffer的readeIndex位置应该是buffer尾部，那么在返回到InternalDecoder中message的方法readable返回的是false,那么就会对buffer重新赋予EMPTY_BUFFER实体，而针对包含一个以上的dubbo协议栈，当然也会解析出其中一个dubbo协议栈，但是经过ExchangeCodec解析之后，message的readIndex不在message尾部，所以message的readable方法返回的是true。那么则会继续遍历message，读取下面的信息。最终要么message刚好整数倍包含完整的dubbo协议栈，要不ExchangeCodec返回NEED_MORE_INPUT,最后将未读完的数据缓存到buffer中,等待下次inbound事件，将buffer中的消息合并到下次的inbound消息中，种类又回到了拆包的问题上。

# zookeeper宕机与dubbo直连的情况

在实际生产中，假如 zookeeper 注册中心宕掉，一段时间内服务消费方还是能够调用提供方的服务的，实际上它使用的**本地缓存进行通讯**，这只是 dubbo 健壮性的一种提现。

**dubbo的健壮性表现：**

1. 监控中心宕掉不影响使用，只是丢失部分采样数据
2. 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
3. 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
4. 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
5. 服务提供者无状态，任意一台宕掉后，不影响使用
6. 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小。所以，我们可以完全可以绕过注册中心——采用 **dubbo 直连** ，即在服务消费方配置服务提供方的位置信息。

# Dubbo怎么实现集群容错

**集群容错是消费者端实现的**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpx8UqZjGaZ8qB95qbicY4v5dnoYL7wEwibLkeLhyaUAicM8vSEd7dSibl0oiaSibvC6pwK7Eex54dhIKjmA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

服务目录也就是 Directory，实际上它是一堆 invoker 的集合，还实现了监听注册中心的功能（指的是 RegistryDirectory ）。

 RegistryDirectory 一共有三大作用：

1. 获取 invoker 列表：RegistryDirectory 实现的父类抽象方法 doList，其目的就是得到 invoker 列表，而其内部的实现主要是做了层方法名的过滤，通过方法名找到对应的 invokers。
2. 监听注册中心的变化：通过实现 NotifyListener 接口能感知到注册中心的数据变更，这其实是在服务引入的时候就订阅的。
3. 刷新 invokers：其实就是根据监听变更的 invokerUrls 做一波操作，`refreshInvoker(invokerUrls)`, 根据配置更新 invokers。

 cluster 就是一个中间层，方便的替换各种集群容错措施

Dubbo 默认的 cluster 实现有很多，主要有以下几种：

![图片](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpx8UqZjGaZ8qB95qbicY4v5d06KHl2gGcf8oJhNy312zzMMJQkjcYljFu5AGic0uicnMZDQHbN9bJakQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

每个 Cluster 内部其实返回的都是 XXXClusterInvoker。

- **Failover** - 失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数(不含第一次)。
- **Failfast** - 快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。
- **Failsafe** - 失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。
- **Failback** - 失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
- **Forking** - 并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。

# Dubbo怎么实现负载均衡

## RPC调用中使用随机算法和轮转算法做负载均衡的优缺点

我觉得加权随机算法可以充分发挥每台机器的能力吧。



 Dubbo 也有自己的负载均衡，即 LoadBalance

![图片](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpx8UqZjGaZ8qB95qbicY4v5dI5PQP01Am8ia5EaONep94UacsIuTmajL3SiahpKSicgkibXPSicOulz1TCQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最后怎么串联起来呢？

![图片](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpx8UqZjGaZ8qB95qbicY4v5dciaia6OuBZpKD4RKghAoP9KO4IssibIUQJ7Kf1Do9EHDalxicKJcGic9pfg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先在服务引入的时候，如果没有降级mock逻辑，就会调用Directory#list方法获取所有该服务提供者的地址列表，将多个远程调用都塞入 Directory 中，然后通过 Cluster 来封装这个目录，封装的同时提供各种容错功能，比如 FailOver、FailFast（默认是FailoverClusterInvoker） 等等，最终暴露给消费者的就是一个 invoker。

然后调用 Router 的 route 方法进行路由，过滤掉不符合路由规则的 Invoker。当 FailoverClusterInvoker 拿到 Directory 返回的 Invoker 列表后，再由 loadBalance 来进行负载均衡选择一个 invoker，最终将参数传给选择出的Invoker 实例的 invoke 方法发起调用。

这种过程其实是在 Cluster 的内部发起的，所以能在发起调用出错的情况下，用上容错的各种措施。

# Dubbo怎么实现服务降级

可以通过配置文件

```xml
<dubbo:reference mock="true"  .../>
<dubbo:reference mock="com.xxxx" .../>
<dubbo:reference mock="return null" .../>
<dubbo:reference mock="throw xxx" .../>
<dubbo:reference mock="force:return .." .../>
<dubbo:reference mock="force:throw ..." .../>
```

其中，最主要的两种形式是：
 1） `mock='force:return+null'`表示消费对该服务的方法调用都直接返回null值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
 2） 还可以改为`mock=fail:return+null`表示消费方对该服务的方法调用在失败后，再返回null。用来容忍不重要服务不稳定时对调用方的影响。

在客户端调用代理对象的时候，会调用`MockClusterInvoker#invoke`方法，

```java
public class MockClusterInvoker<T> implements Invoker<T> {
    
    private final Invoker<T> invoker;
    
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        // 获取 mock 配置值
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            // 无 mock 逻辑，直接调用其他 Invoker 对象的 invoke 方法，
            // 比如 FailoverClusterInvoker
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            // force:xxx 直接执行 mock 逻辑，不发起远程调用
            result = doMockInvoke(invocation, null);
        } else {
            // fail:xxx 表示消费方对调用服务失败后，再执行 mock 逻辑，不抛出异常
            try {
                // 调用其他 Invoker 对象的 invoke 方法
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                } else {
                    // 调用失败，执行 mock 逻辑
                    result = doMockInvoke(invocation, e);
                }
            }
        }
        return result;
    } 
    // 省略其他方法
}
```

# Dubbo怎么实现限流

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpysOH9vvW1SMXroibXRCiaJz3u5xjBRTDRn4nT8ZAhmxSE6T3fLvOuybibDzx2sttWv615UV7rbeOLwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还是这张图，限流是在Filter的时候做的。作用于提供方

Dubbo默认使用令牌桶算法实现限流。某段时间内，桶里面只能放进n个令牌，然后来一个请求就减少一个令牌，如果桶里面的令牌没有了，则不能继续执行请求。

## 漏桶和令牌桶区别

 漏桶算法思路很简单，水（请求）先进入到漏桶里，漏桶以一定的速度出水，当水流入速度过大会直接溢出，可以看出漏桶算法能强行限制数据的传输速率。

 对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。

并不能说明令牌桶一定比漏洞好，她们使用场景不一样。令牌桶可以用来保护自己，主要用来对调用者频率进行限流，为的是让自己不被打垮。所以如果自己本身有处理能力的时候，如果流量突发（实际消费能力强于配置的流量限制），那么实际处理速率可以超过配置的限制。而漏桶算法，这是用来保护他人，也就是保护他所调用的系统。主要场景是，当调用的第三方系统本身没有保护机制，或者有流量限制的时候，我们的调用速度不能超过他的限制，由于我们不能更改第三方系统，所以只有在主调方控制。这个时候，即使流量突发，也必须舍弃。因为消费能力是第三方决定的。
总结起来：如果要让自己的系统不被打垮，用令牌桶。如果保证被别人的系统不被打垮，用漏桶算法。

# Dubbo怎么实现路由

Dubbo 一共有三种路由分别是：条件路由 ConditionRouter、脚本路由 ScriptRouter 和标签路由 TagRouter。最常用的就是条件路由

路由的配置一样是通过 RegistryDirectory 的 notify 更新和构造的，然后路由的调用在是刷新 invoker 的时候，具体是在调用 `toMethodInvokers` 的时候会进行服务级别的路由和方法级别的路由。Directory 还经过了路由规则的过滤。

# 心跳怎么实现的

**`Dubbo服务端和客户端都会发起心跳。`**目的是维持provider和consumer之间的长连接

dubbo心跳时间heartbeat默认是60s，超过heartbeat时间没有收到消息，就发送心跳消息(provider，consumer一样)，如果连着3次（heartbeatTimeout为heartbeat*3）没有收到心跳响应，provider会关闭channel，而consumer会进行重连；不论是provider还是consumer的心跳检测都是通过**启动定时任务**的方式实现；

## Provider启动心跳检测

```java
public HeaderExchangeServer(Server server) {
    if (server == null) {
        throw new IllegalArgumentException("server == null");
    }
    this.server = server;
    this.heartbeat = server.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
    //心跳超时时间默认为心跳时间的3倍
    this.heartbeatTimeout = server.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
    //如果心跳超时时间小于心跳时间的两倍则抛异常
    if (heartbeatTimeout < heartbeat * 2) {
        throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
    }
    startHeatbeatTimer();
}
```

startHeatbeatTimer的实现 

先停止已有的定时任务，启动新的定时任务：

```java
private void startHeatbeatTimer() {
    // 停止原有定时任务
    stopHeartbeatTimer();
    // 发起新的定时任务
    if (heartbeat > 0) {
        heatbeatTimer = scheduled.scheduleWithFixedDelay(
            new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
                public Collection<Channel> getChannels() {
                    return Collections.unmodifiableCollection(HeaderExchangeServer.this.getChannels());
                }
            }, heartbeat, heartbeatTimeout),
            heartbeat, heartbeat, TimeUnit.MILLISECONDS);
    }
}
```

HeartBeatTask的实现：

遍历所有的channel，检测心跳间隔，如果超过心跳间隔没有读或写，则发送需要回复的心跳消息，最有判断是否心跳超时(heartbeatTimeout)，如果超时，provider关闭channel，consumer进行重连

```java
public void run() {
    try {
        long now = System.currentTimeMillis();
        for (Channel channel : channelProvider.getChannels()) {
            if (channel.isClosed()) {
                continue;
            }
            try {
                Long lastRead = (Long) channel.getAttribute(HeaderExchangeHandler.KEY_READ_TIMESTAMP);
                Long lastWrite = (Long) channel.getAttribute(HeaderExchangeHandler.KEY_WRITE_TIMESTAMP);
                // 读写的时间，任一超过心跳间隔，发送心跳
                if ((lastRead != null && now - lastRead > heartbeat)
                    || (lastWrite != null && now - lastWrite > heartbeat)) {
                    Request req = new Request();
                    req.setVersion("2.0.0");
                    req.setTwoWay(true); // 需要响应的心跳事件
                    req.setEvent(Request.HEARTBEAT_EVENT);
                    channel.send(req);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Send heartbeat to remote channel " + channel.getRemoteAddress()
                                     + ", cause: The channel has no data-transmission exceeds a heartbeat period: " + heartbeat + "ms");
                    }
                }
                // 最后读的时间，超过心跳超时时间
                if (lastRead != null && now - lastRead > heartbeatTimeout) {
                    logger.warn("Close channel " + channel
                                + ", because heartbeat read idle time out: " + heartbeatTimeout + "ms");
                    // 客户端侧，重新连接服务端
                    if (channel instanceof Client) {
                        try {
                            ((Client) channel).reconnect();
                        } catch (Exception e) {
                            //do nothing
                        }
                        // 服务端侧，关闭客户端连接
                    } else {
                        channel.close();
                    }
                }
            } catch (Throwable t) {
                logger.warn("Exception when heartbeat to remote channel " + channel.getRemoteAddress(), t);
            }
        }
    } catch (Throwable t) {
        logger.warn("Unhandled exception when heartbeat, cause: " + t.getMessage(), t);
    }
}
```

## Comsumer启动心跳检测

```java
public HeaderExchangeClient(Client client, boolean needHeartbeat) {
    if (client == null) {
        throw new IllegalArgumentException("client == null");
    }
    this.client = client;
    // 创建 HeaderExchangeChannel 对象
    this.channel = new HeaderExchangeChannel(client);
    // 读取心跳相关配置
    String dubbo = client.getUrl().getParameter(Constants.DUBBO_VERSION_KEY);
    this.heartbeat = client.getUrl().getParameter(Constants.HEARTBEAT_KEY, dubbo != null && dubbo.startsWith("1.0.") ? Constants.DEFAULT_HEARTBEAT : 0);
    this.heartbeatTimeout = client.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
    if (heartbeatTimeout < heartbeat * 2) { // 避免间隔太短
        throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
    }
    // 发起心跳定时器
    if (needHeartbeat) {
        startHeatbeatTimer();
    }
}
```

# RPC调用中服务器宕机之后发生了什么

# 什么是SPI

SPI 是 Service Provider Interface，主要用于框架中，框架定义好接口，不同的使用者有不同的需求，因此需要有不同的实现，而 SPI 就通过定义一个特定的位置，Java SPI 约定在 Classpath 下的 META-INF/services/ 目录里创建一个**以服务接口命名的文件**，然后**文件里面记录的是此 jar 包提供的具体实现类的全限定名**。

所以就可以通过接口找到对应的文件，获取具体的实现类然后加载即可，做到了灵活的替换具体的实现类。

## 为什么dubbo不用JDK的spi

因为 Java SPI 在查找扩展实现类的时候遍历 SPI 的配置文件并且**将实现类全部实例化**，假设一个实现类初始化过程比较消耗资源且耗时，但是你的代码里面又用不上它，这就产生了资源的浪费。

因此 Dubbo 就自己实现了一个 SPI，给每个实现类配了个名字，通过名字去文件里面找到对应的实现类全限定名然后加载实例化，按需加载。

# Dubbo连接的协议

## dubbo协议

Dubbo缺省协议采用**单一长连接**和NIO异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。Dubbo缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

连接个数：单连接； 

连接方式：长连接； 

传输协议：TCP； 

传输方式：NIO异步传输；

序列化：Hessian二进制序列化

## Hessian 协议

Hessian协议用于集成Hessian的服务，Hessian底层采用Http通讯，采用Servlet暴露服务，Dubbo缺省内嵌Jetty作为服务器实现基于Hessian的远程调用协议。

连接个数：多连接

连接方式：短连接

传输协议：HTTP 传输方式：同步传输 

序列化：Hessian二进制序列化 

适用范围：传入传出参数数据包较大，提供者比消费者个数多，提供者压力较大，可传文件 

适用场景：页面传输，文件传输，或与原生hessian服务互操作

## HTTP 协议

采用Spring的HttpInvoker实现

连接个数：多连接 

连接方式：短连接 

传输协议：HTTP 传输方式：同步传输 

序列化：表单序列化（JSON） 

适用范围：传入传出参数数据包大小混合，提供者比消费者个数多，可用浏览器查看，可用表单或URL传入参数，暂不支持传文件 

适用场景：需同时给应用程序和浏览器JS使用的服务。

## RMI协议

RMI协议采用JDK标准的java.rmi.*实现，采用阻塞式短连接和JDK标准序列化方式

连接个数：多连接 

连接方式：短连接 

传输协议：TCP 

传输方式：同步传输 

序列化：Java标准二进制序列化 

适用范围：传入传出参数数据包大小混合，消费者与提供者个数差不多，可传文件。

 适用场景：常规远程服务方法调用，与原生RMI服务互操作

## Thrift协议

当前 dubbo 支持的 thrift 协议是对 thrift 原生协议的扩展，在原生协议的基础上添加了一些额外的头信息，比如service name，magic number等。

## WebService 协议

连接个数：多连接 

连接方式：短连接 

传输协议：HTTP 

传输方式：同步传输 

序列化：SOAP文本序列化 

适用场景：系统集成，跨语言调用。