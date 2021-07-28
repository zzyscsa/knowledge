# 架构组成

他主要有四大核心组成部分：**NameServer**、**Broker**、**Producer**以及**Consumer**四部分。![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4qEfYsQxwVmhcwpqxrvDFGJJ5kNNS0QGOIrXB2QrNSd9hs4fnnyKhUQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以看到**RocketMQ**啥都是**集群**部署的，这是他**吞吐量大**，**高可用**的原因之一

## NameServer

主要负责对于源数据的管理，包括了对于**Topic**和路由信息的管理。它其实也是一个 **注册中心** ，主要提供两个功能：**Broker管理** 和 **路由信息管理** 。说白了就是 `Broker` 会将自己的信息注册到 `NameServer` 中，此时 `NameServer` 就存放了很多 `Broker` 的信息(Broker的路由表)，消费者和生产者就从 `NameServer` 中获取路由表然后照着路由表的信息和对应的 `Broker` 进行通信(生产者和消费者定期会向 `NameServer` 去查询相关的 `Broker` 的信息)。

## Producer

消息生产者，负责产生消息，一般由业务系统负责产生消息。

消息由**Producer**通过多种负载均衡模式发送到**Broker**集群，发送低延时，支持快速失败。提供了三种方式发送消息：**同步、异步和单向**

## Broker

消息中转角色，负责**存储消息**，转发消息。

- **Broker**是具体提供业务的服务器，单个Broker节点与所有的NameServer节点保持长连接及心跳，并会定时将**Topic**信息注册到NameServer，顺带一提底层的通信和连接都是**基于Netty实现**的。
- **Broker**负责消息存储，以Topic为纬度支持轻量级的队列，单机可以支撑上万队列规模，支持消息推拉模型。
- 官网上有数据显示：具有**上亿级消息堆积能力**，同时可**严格保证消息的有序性**。

## Consumer

消息消费者，负责消费消息，一般是后台系统负责异步消费。

**Consumer**也由用户部署，支持PUSH和PULL两种消费模式，支持**集群消费**和**广播消息**，提供**实时的消息订阅机制**。

- **Pull**：拉取型消费者（Pull Consumer）主动从消息服务器拉取信息，只要批量拉取到消息，用户应用就会启动消费过程，所以 Pull 称为主动消费型。
- **Push**：推送型消费者（Push Consumer）封装了消息的拉取、消费进度和其他的内部维护工作，将消息到达时执行的回调接口留给用户应用程序来实现。所以 Push 称为被动消费类型，但从实现上看还是从消息服务器中拉取消息，不同于 Pull 的是 Push 首先要注册消费监听器，当监听器处触发后才开始消费消息。

# 消息领域模型

![图片](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4BqkagB4km60k4fpJuynd2awiciciad45MvFib5Wiaf50cJiczzbHABwNjKKg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## Message

**Message**（消息）就是要传输的信息。

一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。

一条消息也可以拥有一个可选的标签（Tag）和额处的键值对，它们可以用于设置一个业务 Key 并在 Broker 上查找此消息以便在开发期间查找问题。

## Topic

**Topic**（主题）可以看做**消息的规类**，它是<font color='red'>**消息的第一级类型**</font>。比如一个电商系统可以分为：交易消息、物流消息等，一条消息必须有一个 Topic 。

**Topic** 与生产者和消费者的关系非常松散，一个 Topic 可以有0个、1个、多个生产者向其发送消息，一个生产者也可以同时向不同的 Topic 发送消息。

## Tag

**Tag**（标签）可以看作子主题，它是**<font color='red'>消息的第二级类型</font>**，用于为用户提供额外的灵活性。使用标签，**同一业务模块不同目的的消息就可以用相同 Topic 而不同的 Tag 来标识**。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 **Tag** 。

标签有助于保持您的代码干净和连贯，并且还可以为 **RocketMQ** 提供的查询系统提供帮助。

## Group

分组，一个组可以订阅多个Topic。

分为ProducerGroup，ConsumerGroup，代表某一类的生产者和消费者，一般来说同一个服务可以作为Group，同一个Group一般来说发送和消费的消息都是一样的

## Queue

在**Kafka**中叫Partition，**<font color='red'>每个Queue内部是有序的</font>**，在**RocketMQ**中分为读和写两种队列，一般来说**读写队列数量一致**，如果不一致就会出现很多问题。

## Message Queue

**Message Queue**（消息队列），主题被划分为一个或多个子主题，即消息队列。

一个 Topic 下可以设置多个消息队列，发送消息时执行该消息的 Topic ，RocketMQ 会轮询该 Topic 下的所有队列将消息发出去。

消息的物理管理单位。一个Topic下可以有多个Queue，Queue的引入使得消息的存储可以分布式集群化，具有了水平扩展能力。

## Offset

在**RocketMQ** 中，**所有消息队列都是持久化，长度无限的数据结构**，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用Offset 来访问，Offset 为 java long 类型，64 位，理论上在 100年内不会溢出，所以认为是长度无限。

也可以认为 Message Queue 是一个长度无限的数组，**Offset** 就是<font color='red'>下标</font>。每次消费者组消费完会返回一个成功的响应，然后队列再把维护的消费位移加一，这样就不会出现刚刚消费过的消息再一次被消费了。

# 一次完整的通信流程

Producer 与 NameServer集群中的其中一个节点（随机选择）建立长连接，定期从 NameServer 获取 **Topic** 路由信息，并向提供 Topic 服务的 **Broker Master** 建立长连接，且定时向 **Broker** 发送心跳。

**Producer** 只能将消息发送到 Broker master，但是 **Consumer** 则不一样，它同时和提供 Topic 服务的 Master 和 Slave建立长连接，既可以从 Broker Master 订阅消息，也可以从 Broker Slave 订阅消息。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo48iaqJp9gibF4rNXcLfiaJOzceLF2BdTibPs01nfZeibibrfj2U1Iich1443SQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **Producer**是消息发送方，怎么发送的呢？

**需要先从 `NameServer` 获取关于 `Broker` 的路由信息**，然后通过**轮询**，**Producer**轮询某个**Topic**下面的所有队列实现发送方的负载均衡

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4X3YnHWfibB2MXgLvGA6MDrNm6jWnMUXDMmxEh9gYopqxLlbBBRtlIDQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## Consumer怎么接收消息

消费者通过 `NameServer` 获取所有 `Broker` 的路由信息后，向 `Broker` 发送 `Pull` 请求来获取消息数据。`Consumer` 可以以两种模式启动—— **广播（Broadcast）和集群（Cluster）**。广播模式下，一条消息会发送给 **同一个消费组中的所有消费者** ，集群模式下消息只会发送给一个消费者。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4WJiaqkSTX7xRdLNntBg9ShtF0iaW1aCeoXTqnjT8GJNxksxNxrDib6wQw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4HibXwoQLz818SWIVbu7OlKveLJngfibytMwUWNiaXsXLwArd5Nlx0bmwQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

消费端会通过**RebalanceService**线程，10秒钟做一次基于**Topic**下的所有队列负载。

# 重复消费

去重原则：使用业务端逻辑保持幂等性

保证每条消息都有唯一编号（**比如唯一流水号）**，且保证消息处理成功与去重表的日志同时出现。

# 顺序消费

因为每一个队列内部是有序的，所以可以通过<font color='red'>Hash取模法</font>来保证同一个订单在同一个队列中就行了。

**RocketMQ**的topic内的队列机制,可以保证存储满足**FIFO**（First Input First Output 简单说就是指先进先出）,剩下的只需要消费者顺序消费即可。

# 分布式事务

## Half Message(半消息)

**是指暂不能被Consumer消费的消息**。Producer 已经把消息成功发送到了 Broker 端，但此消息被标记为`暂不能投递`状态，处于该种状态下的消息称为半消息。需要 Producer对消息的`二次确认`后，Consumer才能去消费它。

## 消息回查

由于网络闪段，生产者应用重启等原因。导致 **Producer** 端一直没有对 **Half Message(半消息)** 进行 **二次确认**。这是**Brock**服务器会定时扫描`长期处于半消息的消息`，会

主动询问 **Producer**端 该消息的最终状态(**Commit或者Rollback**),该消息即为 **消息回查**。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpy6iciaD6rYR19SPHCBETqSo4llYyibTR0OOKsSwb4BQmBL5Qd0HTIZ8SSbEHtdCpE2X3ibaTEgicCZIFA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. A服务先发送个Half Message给Brock端，消息中携带 B服务 即将要+100元的信息。

2. 当A服务知道Half Message发送成功后，那么开始第3步执行本地事务。

3. 执行本地事务(会有三种情况1、执行成功。2、执行失败。3、网络等原因导致没有响应)

4. 如果本地事务成功，那么Product像Brock服务器发送Commit,这样B服务就可以消费该message。

5. 如果本地事务失败，那么Product像Brock服务器发送Rollback,那么就会直接删除上面这条半消息。

6. 如果因为网络等原因迟迟没有返回失败还是成功，那么会执行RocketMQ的回调接口,来进行事务的回查。

   # Broker的Buffer

   Broker的**`Buffer`**通常指的是Broker中一个队列的内存Buffer大小，这类**`Buffer`**通常大小有限。

   另外，RocketMQ没有内存**`Buffer`**概念，RocketMQ的队列都是持久化磁盘，数据定期清除。

   RocketMQ同其他MQ有非常显著的区别，RocketMQ的内存**`Buffer`**抽象成一个无限长度的队列，不管有多少数据进来都能装得下，这个无限是有前提的，Broker会定期删除过期的数据。

   例如Broker只保存3天的消息，那么这个**`Buffer`**虽然长度无限，但是3天前的数据会被从队尾删除。

# 消息堆积

消息中间件的主要功能是异步解耦，还有个重要功能是挡住前端的数据洪峰，保证后端系统的稳定性，这就要求消息中间件具有一定的消息堆积能力，消息堆积分以下两种情况：

- 消息堆积在内存**`Buffer`**，一旦超过内存**`Buffer`**，可以根据一定的丢弃策略来丢弃消息，如CORBA Notification规范中描述。适合能容忍丢弃消息的业务，这种情况消息的堆积能力主要在于内存**`Buffer`**大小，而且消息堆积后，性能下降不会太大，因为内存中数据多少对于对外提供的访问能力影响有限。
- 消息堆积到持久化存储系统中，例如DB，KV存储，文件记录形式。当消息不能在内存Cache命中时，要不可避免的访问磁盘，会产生大量读IO，读IO的吞吐量直接决定了消息堆积后的访问能力。

# 刷盘

## 同步刷盘和异步刷盘

![img](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-11/16ef387fba311cda.jpg)

在同步刷盘中需要等待一个刷盘成功的 `ACK` ，同步刷盘对 `MQ` 消息可靠性来说是一种不错的保障，但是 **性能上会有较大影响** ，一般地适用于金融等特定业务场景。

异步刷盘往往是开启一个线程去异步地执行刷盘操作。消息刷盘采用后台异步线程提交的方式进行， **降低了读写延迟** ，提高了 `MQ` 的性能和吞吐量，一般适用于如发验证码等对于消息保证要求不太高的业务场景。

# 存储机制

`RocketMQ` 消息存储架构中的三大角色——`CommitLog` 、`ConsumeQueue` 和 `IndexFile` 。

主要就是前两个

- `CommitLog`： **消息主体以及元数据的存储主体**，存储 `Producer` 端写入的消息主体内容，消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是**顺序写入日志文件**，当文件满了，写入下一个文件。
- `ConsumeQueue`： 消息消费队列，**引入的目的主要是提高消息消费的性能**(我们再前面也讲了)，由于`RocketMQ` 是基于主题 `Topic` 的订阅模式，消息消费是针对主题进行的，如果要遍历 `commitlog` 文件中根据 `Topic` 检索消息是非常低效的。`Consumer` 即可根据 `ConsumeQueue` 来查找待消费的消息。其中，`ConsumeQueue`（逻辑消费队列）**作为消费消息的索引**，保存了指定 `Topic` 下的队列消息在 `CommitLog` 中的**起始物理偏移量 `offset` ，消息大小 `size` 和消息 `Tag` 的 `HashCode` 值。`consumequeue` 文件可以看成是基于 `topic` 的 `commitlog` 索引文件**，故 `consumequeue` 文件夹的组织方式如下：topic/queue/file三层组织结构，具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}。同样 `consumequeue` 文件采取定长设计，每一个条目共20个字节，分别为8字节的 `commitlog` 物理偏移量、4字节的消息长度、8字节tag `hashcode`，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个 `ConsumeQueue`文件大小约5.72M；
- `IndexFile`： `IndexFile`（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。

`ConsumeQueue` 你可以大概理解为 `Topic` 中的队列。

![img](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-11/16ef3884c02acc72.png)

`RocketMQ` 采用的是 **混合型的存储结构** ，即为 `Broker` 单个实例下所有的队列共用一个日志数据文件来存储消息。有意思的是在同样高并发的 `Kafka` 中会为每个 `Topic` 分配一个存储文件。这就有点类似于我们有一大堆书需要装上书架，`RockeMQ` 是不分书的种类直接成批的塞上去的，而 `Kafka` 是将书本放入指定的分类区域的。

而 `RocketMQ` 为什么要这么做呢？原因是 **提高数据的写入效率** ，不分 `Topic` 意味着我们有更大的几率获取 **成批** 的消息进行数据写入，但也会带来一个麻烦就是读取消息的时候需要遍历整个大文件，这是非常耗时的。

所以，在 `RocketMQ` 中又使用了 `ConsumeQueue` 作为每个队列的索引文件来 **提升读取消息的效率**。我们可以直接根据队列的消息序号，计算出索引的全局位置（索引序号*索引固定⻓度20），然后直接读取这条索引，再根据索引中记录的消息的全局位置，找到消息。

就是相当于消费者从`ConsumeQueue` 中读到的是索引，然后这个索引指示这条消息在commitLog里面的位置，然后自己去取出来。

# 为啥用NameServer而不是zookeeper，nameserver缺点？

1. ZooKeeper 作为支持顺序一致性的中间件，在某些情况下，它为了满足一致性，会丢失一定时间内的可用性，RocketMQ 需要注册中心只是为了发现组件地址，在某些情况下，RocketMQ 的注册中心可以出现数据不一致性，这同时也是 NameServer 的缺点，因为 NameServer 集群间互不通信，它们之间的注册信息可能会不一致
2. 另外，当有新的服务器加入时，NameServer 并不会立马通知到 Produer，而是由 Produer 定时去请求 NameServer 获取最新的 Broker/Consumer 信息（这种情况是通过 Producer 发送消息时，负载均衡解决）