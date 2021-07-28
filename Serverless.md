# Serverless

**<font color='red'>侧重描述一类服务</font>**，是一种架构。即，从使用者的角度无需关心背后的计算存储网络内存等实现，也无需关心维护方面。FaaS和BaaS，一定程度来说，都算Serverless。

![file](https://img2018.cnblogs.com/blog/1411156/201910/1411156-20191023143208962-839589233.jpg)

- 将业务函数抽象成一个个 FaaS 函数，将数据库、缓存、加速等服务抽象成 BaaS 服务；
- 上层提供 Restful 或事件触发机制调用，对应到不同的端（PC、移动端）；
- 想要拓展平台能力，只要在端上做开放（组件接入）与 FaaS 服务做开放（后端接入）即可。

# FaaS

 Function as a service。函数即服务。每一个函数都是一个服务，函数可以由任何语言编写。把程序执行的过程，抽象为执行的一个个函数（Function），这些函数可以被打散到各个服务器上去独立执行，开发者只需要实现函数的业务逻辑，无需关注背后的存储网络计算资源等。

- FaaS运行的是后端代码而不是整个后端程序。
- 代码通过事件触发。由于不再有一个长期运行的进程等待或轮询用户请求，代码只能通过特殊的事件触发。这些事件由FaaS框架定义，例如上传文件到对象存储、消息队列收到一条新的消息、API Gateway收到一个新的API请求等。
- 代码的生命周期很短。
- 代码必须做到彻底无状态，两次调用间不能共享内存状态。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/d7YzaYDnrxFcvAVicWQUtLSIrBPbicOdqicbWaa67cQib6zBviaERiashdzdH7kicUbWL7R1GbZrNT08jnFRzlzVL0g5A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当用户第一次访问 HTTP 函数触发器时，函数触发器就会 Hold 住用户的 HTTP 请求，并产生一个 HTTP Request 事件通知函数服务。紧接着函数服务就会检查有没有闲置的函数实例；如果没有函数实例，就去函数代码仓库中拉取你的代码；初始化并启动一个函数实例，执行这个函数，传入这个 HTTP Request 对象作为函数的参数，执行函数。再进一步，函数执行的结果 HTTP Response 返回函数触发器，函数触发器再将结果返回给等待的用户客户端。

## FaaS的启动以及为什么 FaaS 可以极速启动，而应用托管平台 PaaS 不行：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/d7YzaYDnrxFcvAVicWQUtLSIrBPbicOdqicg0cRusEmw1cgEgYLKXrw1Q25FHSwicnMBKY3gIhp9r4m5O09icnrRUag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

PaaS 为了适应用户的多样性，必须支持多语言兼容，还要提供传统后台服务，例如 MySQL、Redis。意味着，应用托管平台 PaaS 在初始化环境时，有大量依赖和多语言版本需要兼容，而且兼容多种用户的应用代码往往也会增加应用构建过程的时间。所以通常应用托管平台 PaaS 无法抽象出轻量的可复用的层级，只能选择服务器或容器方案，从操作系统层开始构建应用实例。

FaaS 设计之初就牺牲了用户的可控性和应用场景，来简化代码模型，并且通过分层结构进一步提升资源的利用率。

## FaaS的分层：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/d7YzaYDnrxFcvAVicWQUtLSIrBPbicOdqicEXgmMHWLr0NaFcOVq65RSibYBrnguPFehdyfViaPH41BI4VyvVbibLRIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

容器层就像是 Windows 操作系统；Runtime 就像是 Windows 里面的播放器暴风影音；你的代码就像是放在 U 盘里的电影。这样分层有什么好处呢？容器层适用性更广，云服务商可以预热大量的容器实例，将物理服务器的计算资源碎片化。Runtime 的实例适用性较低，可以少量预热；容器和 Runtime 固定后，下载你的代码就可以执行了。通过分层，我们可以做到资源统筹优化，这样就能让你的代码快速低成本地被执行。

云服务商负责的就是容器和 Runtime 的准备阶段了。而开发者自己负责的则是函数执行阶段。一旦容器 &Runtime 启动后，就会维持一段时间，这段时间内的这个函数实例就可以直接处理用户数据请求。当一段时间内没有用户请求事件发生（各个云服务商维持实例的时间和策略不同），则会销毁这个函数实例。

## FaaS进程模型

- 用完即毁型：函数实例准备好后，执行完函数就直接结束。这是 FaaS 最纯正的用法。
- 常驻进程型：函数实例准备好后，执行完函数不结束，而是返回继续等待下一次函数被调用。**这里需要注意，即使 FaaS 是常驻进程型，如果一段时间没有事件触发，函数实例还是会被云服务商销毁**

# BaaS

Backend as a service。后端即服务。提供可以使用的后端服务，提供RestFul API，终端客户接触不到。就是集成了许多中间件技术，可以无视环境调用服务，比如数据即服务（数据库服务），缓存服务等。开发者使用这些后端服务就可以轻松开发Application。开发者只需要关注客户端的实现，服务端的实现以及存储网络等都无需关心，

# IaaS

Infrastructure as a Service。基础设施即服务。比如计算机存储、网络、服务器等基建设施以服务的方式提供。也就是**计算/内存/存储/网络资源**是被虚拟化的，多个租户可以自由按需分别使用。

# PaaS

Platform as a service。平台即服务。用户只要上传源代码就可以自动持续集成并享受高可用服务，如果速度足够快，可以认为是类似 Serverless。但随着以 Docker 为代表的容器技术兴起，以容器为粒度的 PaaS 部署逐渐成为主流，是最常用的应用部署方式。比如中间件、数据库、操作系统等。

　　Serverless跟PaaS之间的界线比较模糊，很多人认为Serverless是PaaS的一种。

　　Serverless由BaaS和FaaS两部分构成，<font color='red'>BaaS负责提供业务的依赖服务，FaaS负责业务的部署和生命周期管理</font>，从这个意义上来看，Serverless的角色跟PaaS一样。与传统PaaS的区别在于，传统PaaS是以**程序为粒度**管理应用的生命周期，而Serverless是以**函数粒度**管理应用生命周期。传统PaaS中的应用为常驻内存的进程，而Serverless应用运行完即销毁。此外，使用传统PaaS，用户仍需要关心水平扩展，例如如何配置Auto-Scaling Group，但Serverless没有这个问题，水平扩展是架构天然自带的功能。FaaS 与应用托管 PaaS 平台对比，最大的区别在于资源利用率，这也是 FaaS 最大的创新点。FaaS 的应用实例可以缩容到 0，而应用托管 PaaS 平台则至少要维持 1 台服务器或容器。

# SaaS

Software as a Service。软件即服务。侧重在终端用户可以用的**Application/Software**是跑在云上的，打开网页即可使用。举例，Gmail，Office365，Google Drive，百度云，Github等等。



# 关系

- **IaaS**是比较底层，其他的一般需要依赖IaaS来实现。
- **FaaS**和**BaaS**都可以算作**Serverless**，其中FaaS和Serverless最近，算是一种Serverless。BaaS侧重于提供了RESTFul的API接口，但是FaaS并不提供这类接口（当然也可以提供），Serverless也没有说必须要提供RESTFul的API接口（尽管也可以提供）
- **SaaS**相信很容易理解，开箱即用，就是终端用户可以使用的产品。按使用量收费，比如对于iCloud的备份空间超出5GB是要额外按月购买的。你买一份，那么iCloud就被剥离出去5GB的存储。**注意，**这个和你在爱奇艺上按月订购视频服务是不一样的。区别在于你购买iCloud空间是**一个大的共享资源池的一部分**，而你订购爱奇艺视频，和共享资源池无关，爱奇艺不过是给你发送一份订阅而已，类似于送报纸的逻辑，给你送报纸。iCloud开辟一个空间，更像是一些孵化创业园区的共享办公，划出一个隔间作为A公司的办公室，划出一些隔间作为B公司的办公室，分别按照使用量收钱。

# Serverless与微服务

　Serverless和微服务没有直接关系，但两者有相似之处，例如都需要做业务拆分、强调无状态、具有敏捷特性等。<font color='red'>Serverless在很多方面比微服务粒度更细，要求也更严格。</font>例如微服务以服务为边界拆分业务，**Serverless以函数为边界拆分业务**；微服务可以有跨调用的内存状态共享，Serverless要求调用彻底无状态。此外，Serverless依赖BaaS提供第三方依赖，而微服务可以自由选择第三方依赖来源，例如使用本地搭建的传统中间件栈（如本地MySql和消息总线）。