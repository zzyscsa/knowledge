# k8s基本组件

![image-20210118195846117](D:\Typora\picture\image-20210118195846117.png)

## Master组件

- API server：集群统一入口，以restFul方式，交给etcd存储。任何对资源进行增删改查的操作都要       交给APIServer处理后再提交给etcd
- scheduler：节点调度，选择node节点应用部署，负责调度pod到合适的Node上。
- controller-manager：处理集群中常规后台任务，一个资源对应一个controller，controller manager就是负责管理这些控制器的
- etcd：高可用的键值存储系统，用于保存集群相关数据

## Node组件

- kubelet：master派到node的代表，负责维护和管理该Node上面的所有容器。
- kube-proxy：实现了k8s中的**<font color='red'>服务发现</font>**和**<font color='red'>反向代理</font>**功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的**watch机制**，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。
- runtime：runtime指的是容器运行环境，目前Kubernetes支持docker和rkt两种容器。

# Pod

## 基本概念

- 最小部署单元
- 包含多个容器（一组容器集合）
- 一个pod中容器共享网络命名空间
- pod是短暂的，重启可能IP地址就会变化

## 存在意义

- 创建容器使用的是docker，docker是单进程设计，一个docker运行一个应用程序；而Pod是**多进程**设计，运行多个应用程序，一个Pod有多个容器
- Pod存在为了亲密性应用
  - 两个应用之间交互
  - 网络之间调用
  - 两个应用之间需要频繁调用

## 实现机制

### 共享网络

容器本身是相互隔离的，但是他们之间是共享网络的

![image-20210118201157728](D:\Typora\picture\image-20210118201157728.png)

每个Pod有一个Pause容器，又叫做“根容器”，每创建一个业务容器Pause容器都把它加入到Pause容器里面来，让所有业务容器在同一个namespace空间中，实现网络共享

### 共享存储

![image-20210118201636697](D:\Typora\picture\image-20210118201636697.png)

## 镜像拉取策略

- IfNotPresent：默认值，镜像在宿主机上不存在时才拉取
- Always：每次创建Pod都重新拉取镜像
- Never：Pod不会主动拉取，必须手动拉取镜像

## Pod生命周期

Pod被分配到一个Node上之后，就不会离开这个Node，直到被删除。当某个Pod失败，首先会被Kubernetes清理掉，之后ReplicationController将会在其它机器上（或本机）重建Pod，重建之后Pod的ID发生了变化，那将会是一个新的Pod。所以，Kubernetes中Pod的迁移，实际指的是在新Node上重建Pod。

![img](https://upload-images.jianshu.io/upload_images/4728488-f06ac16107147ec5.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

## 资源限制

![image-20210118202515658](D:\Typora\picture\image-20210118202515658.png)

Yaml文件表示Pod调度的容器，限定内存64M，最大不能超过128M

## 重启机制

yaml文件restartPolicy配置重启策略

- Always：当容器终止退出后，总是重启容器，默认策略
- OnFailure：当容器异常退出（退出状态码非0）时，才重启容器
- Never：当容器终止退出，从不重启容器

## 健康检查

- livenessProbe（存活检查）：如果检查失败，杀死容器，根据Pod的restartPolicy配置来操作。
- readinessProbe（就绪检查）：如果检查失败，k8s会把Pod从service endpoints中剔除。

Probe支持以下三种检查方法：

- httpGet：发送http请求，返回200-400范围就是成功
- exec：执行shell命令返回状态码0为成功
- tcpSocket：发起socket连接如果能连接就成功

## 调度策略

### 创建Pod流程

![image-20210119194553923](D:\Typora\picture\image-20210119194553923.png)

#### master节点

- create Pod -> api server -> etcd存储 --> 状态返回给api server
- scheduler在api server中监听有没有pod创建，如果有新的pod创建，通过etcd读取这个pod，通过**调度算法**把pod调度到某个node节点上面，状态返回给api server

#### node节点

- kubelet访问api server，读取etcd拿到分配给当前节点的pod，通过docker创建容器，并把状态返回给api server

### 影响调度的因素

- Pod资源限制的影响
- 节点选择器标签（nodeSelector）的影响，比如可以选择调度到对应开发/生产环境，如1和2节点标签为dev，3、4节点标签为prod
- 节点亲和性：和选择器标签差不多，都是根据节点标签调度
  - 硬亲和性：约束条件必须满足
  - 软亲和性：尝试满足，不保证
- 污点（Taint）和污点容忍：以上两种都是Pod自己的属性，而污点是节点属性，节点不做普通的调度，它可以做专用的节点，比如只能对某些IP地址进行分配。三种污点值：
  - NoSchedule：一定不被调度
  - PreferNoSchedule：尽量不被调度
  - NoExecute：不会调度，并且还会驱逐node中已有的pod

# Controller

## 基本概念

- 在集群上管理和运行容器的对象，是实际存在的

- 确保预期的pod副本数量
- 有状态部署和无状态部署
- 一次性任务以及定时任务

## pod和controller的关系

pod通过controller实现应用的运维，比如伸缩、滚动升级，两者之间通过label标签建立关系，pod上是label标签对应controller上的selector标签

## deployment控制器

- 部署**无状态**应用，如web服务、微服务
- 管理pod和replicaSet（副本）
- 滚动升级，回滚、弹性伸缩功能

# Service

## 基本概念

定义一组Pod的访问规则，Service有自己的IP，是一种虚拟IP对外暴露，外界访问Service，Service进行服务发现和负载均衡。

常用Service类型：

- ClusterIP：集群内部进行使用
- NodePort：对外访问应用使用
- LoadBalancer：对外访问应用使用，公有云

## 存在意义

- 防止Pod失联（服务发现）

pod的ip地址可能在升级的时候就变化了，现在pod会把自己ip注册到service中，如果更新会通知servcie

- 定义一组Pod访问策略（负载均衡）

# List-Watch

由于kubernetes系统的采取Level Trigger而非Edge Trigger的设计理念，所以**各组件只需要感知数据最新的状态**，而不需要担心错过数据的变化过程。list-watch作为kubernentes系统消息通知机制(或者说数据实时通知机制)。

`Etcd`存储集群的数据信息，`apiserver`作为统一入口，任何对数据的操作都必须经过 `apiserver`。客户端(`kubelet`/`scheduler`/`controller-manager`)通过 `list-watch` 监听 `apiserver` 中资源（pod/rs/rc等等）的 `create`, `update` 和 `delete` 事件，并针对`事件类型`调用相应的`事件处理函数`。

**这里 List，一般就是指获取全量数据，一个一次性的请求。Watch，一般指获取增量数据，一个持久链接 (Http streaming) 的请求**

`list-watch`有两部分组成，分别是`list`和 `watch`。

list基于**HTTP短链接**实现；watch则是调用资源的watch API监听资源变更事件，基于**HTTP 长链接**

## List

list非常好理解，就是调用资源的list API罗列资源，

K8S 的`informer` 模块封装 `list-watch API`，用户只需要指定资源，编写事件处理函数，`AddFunc`, `UpdateFunc`和 `DeleteFunc`等。

`informer`首先通过`list API` 罗列资源，然后调用 `watch API`监听资源的变更事件，并将结果放入到一个 `FIFO 队列`，队列的另一头有协程从中取出事件，并调用对应的注册函数处理事件。`Informer`还维护了一个只读的`Map Store` 缓存，主要为了提升查询的效率，降低`apiserver` 的负载。

![img](https://upload-images.jianshu.io/upload_images/3029445-8326e103c0216c83.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

list请求可以保证消息不丢失或者有可靠的重新获取机制(比如说kubelet和kube-apiserver间网络闪断，需要保证网络恢复后kubelet可以收到网络闪断期间产生的消息)

1. watch请求开始之前，先发起一次list请求，获取集群中当前所有该类数据(同时得到最新的ResourceVersion)，之后基于最新的ResourceVersion发起watch请求。
2. 当watch出错时(比如说网络闪断造成客户端和服务端数据不同步)，重新发起一次list请求获取所有数据，再重新基于最新ResourceVersion来watch。

## Watch

Watch是如何通过 `HTTP 长链接`接收`apiserver`发来的`资源变更事件`

实现原理是通过HTTP的 **Chunked transfer encoding（分块传输编码）**：HTTP 分块传输编码允许服务器为动态生成的内容维持 HTTP 持久链接。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。

当客户端调用 `watch API` 时，`apiserver` 在`response` 的 `HTTP Header` 中设置 `Transfer-Encoding`的值为`chunked`，表示采用`分块传输`编码，客户端收到该信息后，便和服务端该链接，并等待下一个数据块，即资源的事件信息。





`list API`可以查询**当前的资源及其对应的状态(即期望的状态)**，客户端通过拿`期望的状态`和`实际的状态`进行对比，纠正状态不一致的资源。让集群现状趋于用户期待。