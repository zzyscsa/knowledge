zookeeper他是个数据库，文件存储系统，并且有监听通知机制（观察者模式）

- ZooKeeper主要**服务于分布式系统**，可以用ZooKeeper来做：统一配置管理、统一命名服务、分布式锁、集群管理。
- 使用分布式系统就无法避免对节点管理的问题(需要实时感知节点的状态、对节点进行统一管理等等)，而由于这些问题处理起来可能相对麻烦和提高了系统的复杂性，ZooKeeper作为一个能够**通用**解决这些问题的中间件就应运而生了。

**为什么zookeeper能做这些？**

# 原理

## 数据结构

ZooKeeper的数据结构，跟Unix文件系统非常类似，可以看做是一颗**树**，每个节点叫做**ZNode**。每一个节点可以通过**路径**来标识，结构图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJjsfAkwgiaM65BadDL48V0tZOYgAWnKaZdadIJVC737xDyuC03euxGGg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那ZooKeeper这颗"树"有什么特点呢？？ZooKeeper的节点我们称之为**Znode**，Znode分为**两种**类型：

- **短暂/临时(Ephemeral)**：当客户端和服务端断开连接后，所创建的Znode(节点)**会自动删除**
- **持久(Persistent)**：当客户端和服务端断开连接后，所创建的Znode(节点)**不会删除**

**ZooKeeper和Redis一样，也是C/S结构(分成客户端和服务端)**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJmuRxxsRwItFUKDia5G8CH3cf0KKBCicF16JfrMlTtFtACqPufScuTrJA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 监听器

**常见**的监听场景有以下两项：

- 监听Znode节点的**数据变化**
- 监听子节点的**增减变化**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJ2MPKwsv4m7rLooZ3eInFNjvpVudhcLouCszUDJDA5RVEjWDOIibVHSQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJsIBicWhwAJIcNwYPvXfN7mmgh09RBSbTQd6AfsyfOfFqeqsCrnxPOKg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 统一配置管理

我们可以将`common.yml`这份配置放在ZooKeeper的Znode节点中，系统A、B、C监听着这个Znode节点有无变更，如果变更了，**及时**响应。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJkMbN7WO6ITfDPHtO09zibW532otIiaLlw5vAsvtPth0FNrz4dInibPEKA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 统一命名服务

统一命名服务的理解其实跟**域名**一样，是我们为这某一部分的资源给它**取一个名字**，别人通过这个名字就可以拿到对应的资源。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJDyPibE3x7OFicib01XFjibQKUygiaPTZz5F0vPkedlanemyqKCg7JVyLDlg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 分布式锁

系统A、B、C都去访问`/locks`节点

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJNqt17v32q9icicE67f9UVYHbseicaUYZgmy1ObqichHm54LLicXRFSGBcMQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

访问的时候会创建**带顺序号的临时/短暂**(`EPHEMERAL_SEQUENTIAL`)节点，比如，系统A创建了`id_000000`节点，系统B创建了`id_000002`节点，系统C创建了`id_000001`节点。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJczzKQ6bLPE3Buwib1YJeqluPWicmZUbPadvFCU6UopDDkajKQu1FLO3A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接着，拿到`/locks`节点下的所有子节点(id_000000,id_000001,id_000002)，**判断自己创建的是不是最小的那个节点**

- 如果是，则拿到锁。

- - 释放锁：执行完操作后，把创建的节点给删掉

- 如果不是，则监听比自己要小1的节点变化

# 集群状态

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/2BGWl1qPxib1WCfFk2icxudIMHNPQMHIDJ4lnbuRg5lEDmjlSTmdarCs8Dq7Pjg213pAq7QlXxzc7dIklkGuAWYQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

只要系统A挂了，那`/groupMember/A`这个节点就会删除，通过**监听**`groupMember`下的子节点，系统B和C就能够感知到系统A已经挂了。(新增也是同理)

除了能够感知节点的上下线变化，ZooKeeper还可以实现**动态选举Master**的功能。(如果集群是主从架构模式下)

原理也很简单，如果想要实现动态选举Master的功能，Znode节点的类型是带**顺序号的临时节点**(`EPHEMERAL_SEQUENTIAL`)就好了。

- Zookeeper会每次选举最小编号的作为Master，如果Master挂了，自然对应的Znode节点就会删除。然后让**新的最小编号作为Master**，这样就可以实现动态选举的功能了。

# Paxos 算法

Paxos算法也是解决数据一致性问题的。是基于**消息传递且具有高度容错特性的一致性算法**，是目前公认的解决分布式一致性问题最有效的算法之一，**其解决的问题就是在分布式系统中如何就某个值（决议）达成一致** 。

在 `Paxos` 中主要有三个角色，分别为 `Proposer提案者`、`Acceptor表决者`、`Learner学习者`。`Paxos` 算法和 `2PC` 一样，也有两个阶段，分别为 `Prepare` 和 `accept` 阶段。

## prepare阶段

- `Proposer提案者`：负责提出 `proposal`，每个提案者在提出提案时都会首先获取到一个 **具有全局唯一性的、递增的提案编号N**，即在整个集群中是唯一的编号 N，然后将该编号赋予其要提出的提案，在**第一阶段是只将提案编号发送给所有的表决者**。
- `Acceptor表决者`：每个表决者在 `accept` 某提案后，会将该提案编号N记录在本地，这样每个表决者中保存的已经被 accept 的提案中会存在一个**编号最大的提案**，其编号假设为 `maxN`。每个表决者仅会 `accept` 编号大于自己本地 `maxN` 的提案，在批准提案时表决者会将以前接受过的最大编号的提案作为响应反馈给 `Proposer` 。

![paxos第一阶段](https://img-blog.csdnimg.cn/img_convert/22e8d512d954676bdf0cc92d200af8ef.png)

## accept阶段

当一个提案被 `Proposer` 提出后，如果 `Proposer` 收到了超过半数的 `Acceptor` 的批准（`Proposer` 本身同意），那么此时 `Proposer` 会给所有的 `Acceptor` 发送真正的提案（你可以理解为第一阶段为试探），这个时候 `Proposer` 就会发送提案的内容和提案编号。

表决者收到提案请求后会再次比较本身已经批准过的最大提案编号和该提案编号，如果该提案编号 **大于等于** 已经批准过的最大提案编号，那么就 `accept` 该提案（此时执行提案内容但不提交），随后将情况返回给 `Proposer` 。如果不满足则不回应或者返回 NO 。

![paxos第二阶段1](https://img-blog.csdnimg.cn/img_convert/b82536f956f70a584c6a20c10113f225.png)

当 `Proposer` 收到超过半数的 `accept` ，那么它这个时候会向所有的 `acceptor` 发送提案的提交请求。需要注意的是，因为上述仅仅是超过半数的 `acceptor` 批准执行了该提案内容，其他没有批准的并没有执行该提案内容，所以这个时候需要**向未批准的 `acceptor` 发送提案内容和提案编号并让它无条件执行和提交**，而对于前面已经批准过该提案的 `acceptor` 来说 **仅仅需要发送该提案的编号** ，让 `acceptor` 执行提交就行了。

而如果 `Proposer` 如果没有收到超过半数的 `accept` 那么它将会将 **递增** 该 `Proposal` 的编号，然后 **重新进入 `Prepare` 阶段** 。

# ZAB协议

Zab（Zookeeper Atomic Broadcast）是为ZooKeeper协设计的**崩溃恢复原子广播协议**，它<font color='red'>保证zookeeper集群数据的一致性和命令的全局有序性。</font>

## 概念

- **集群角色**

1. Leader：同一时间集群总只允许有一个Leader，提供对客户端的读写功能，负责将数据同步至各个节点；
2. Follower：提供对客户端读功能，写请求则转发给Leader处理，当Leader崩溃失联之后参与Leader选举；
3. Observer：与Follower不同的是但不参与Leader选举。

- **服务状态**

1. LOOKING：当节点认为群集中没有Leader，服务器会进入LOOKING状态，目的是为了查找或者选举Leader；
2. FOLLOWING：follower角色；
3. LEADING：leader角色；
4. OBSERVING：observer角色；

- **ZAB状态**：Zookeeper还给ZAB定义的4中状态，反应Zookeeper从选举到对外提供服务的过程中的四个步骤。

1. ELECTION: 集群进入选举状态，此过程会选出一个节点作为leader角色；
2. DISCOVERY：连接上leader，响应leader心跳，并且检测leader的角色是否更改，通过此步骤之后选举出的leader才能执行真正职务；
3. SYNCHRONIZATION：整个集群都确认leader之后，将会把leader的数据同步到各个节点，保证整个集群的数据一致性；
4. BROADCAST：过渡到广播状态，集群开始对外提供服务。

- <font color='red'>**ZXID**</font>

Zxid是极为重要的概念，它是一个long型（64位）整数，分为两部分：纪元（epoch）部分和计数器（counter）部分，是一个**全局有序**的数字。

epoch代表当前集群所属的哪个leader，leader的选举就类似一个朝代的更替，你前朝的剑不能斩本朝的官，用epoch代表当前命令的有效性，counter是一个递增的数字。

ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction id, 该id称为zxid. 由于zxid的递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生。

创建任意节点, 或者更新任意节点的数据, 或者删除任意节点, 都会导致Zookeeper状态发生改变, 从而导致zxid的值增加。

## 选举

### 时机

- **选举发生的时机**Leader发生选举有两个时机，一个是服务启动的时候当整个集群都没有leader节点会进入选举状态，如果leader已经存在就会告诉该节点leader的信息，自己连接上leader，整个集群不用进入选举状态。

- 还有一个就是在服务运行中，可能会出现各种情况，服务宕机、断电、网络延迟很高的时候leader都不能再对外提供服务了，所有当其他几点通过心跳检测到leader失联之后，集群也会进入选举状态。

### 规则

zab协议是按照几个比较规则来进行投票的筛选，如果你的票比我更好，就修改自身的投票信息，改投你当leader。

当其他节点的纪元比自身高投它，如果纪元相同比较自身的zxid的大小，选举zxid大的节点，这里的zxid代表节点所提交事务最大的id，**zxid越大代表该节点的数据越完整。**

最后如果epoch和zxid都相等，则比较服务的serverId，这个Id是配置zookeeper集群所配置的，所以我们配置zookeeper集群的时候可以把服务性能更高的集群的serverId配置大些，让性能好的机器担任leader角色。

### 流程

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzX3UD9iaiaozvbZib41ib8AxLMmRFMibA6QdwqgEQF36eLoBPVGboYcGrZVrF1cEuibT4Ut6APHhoymTCA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 所有节点第一票先选举自己当leader，将投票信息广播出去；
- 从队列中接受投票信息；
- 按照规则判断是否需要更改投票信息，将更改后的投票信息再次广播出去；
- 判断是否有<font color='red'>超过一半</font>的投票选举同一个节点，如果是选举结束根据投票结果设置自己的服务状态，选举结束，否则继续进入投票流程。

## 广播

集群在经过leader选举之后还会有连接leader和同步两个步骤。那么集群对外提供服务如何保证各个节点数据的一致性？

zab在广播状态中保证以下特征

- **可靠传递:** 如果消息m由一台服务器传递，那么它最终将由所有服务器传递。
- **全局有序:** 如果一个消息a在消息b之前被一台服务器交付，那么所有服务器都交付了a和b，并且a先于b。
- **因果有序:** 如果消息a在因果上先于消息b并且二者都被交付，那么a必须排在b之前。

**有序性**是zab协议必须要保证的一个很重要的属性，因为zookeeper是以类似目录结构的数据结构存储数据的，必须要求命名的有序性。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzX3UD9iaiaozvbZib41ib8AxLMod5pbG2Ij0DdfhQQ0U3I4ay4JqnG1YTXStsSuTBpbmoicBLXOVww8og/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，整个写请求类似一个二阶段的提交。

当收到客户端的写请求的时候会经历以下几个步骤：

1. Leader收到客户端的写请求，生成一个事务（Proposal），其中包含了zxid；
2. Leader开始广播该事务，需要注意的是所有节点的通讯都是由一个FIFO的队列维护的；
3. Follower接受到事务之后，将事务写入本地磁盘，写入成功之后返回Leader一个ACK；
4. Leader收到过半的ACK之后，开始提交本事务，并广播事务提交信息
5. 从节点开始提交本事务。

有以上流程可知，zookeeper通过二阶段提交来保证集群中数据的一致性，因为**只需要收到过半的ACK就可以提交事务，所以zookeeper的数据并不是强一致性。**

zab协议的有序性保证是通过几个方面来体现的，

- 第一，服务之前用<font color='gdreen'>TCP协议</font>进行通讯，保证在网络传输中的有序性；
- 第二，节点之前都维护了一个<font color='gdreen'>FIFO队列</font>，保证全局有序性；第三，通过全局递增的zxid保证因果有序性。

# Zookeeper节点数据操作流程

1.在Client向Follwer发出一个写的请求

2.Follower把请求发送给Leader

3.Leader接收到以后开始发起投票并通知Follower进行投票

4.Follower把投票结果发送给Leader

5.Leader将结果汇总后如果需要写入，则开始写入同时把写入操作通知给Follower，然后commit;

6.Follower把请求结果返回给Client

# Zookeeper工作原理

Zookeeper的核心是原子广播，这个机制保证了各个server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是`恢复模式`和`广播模式`。

当服务启动或者在领导者崩溃后，Zab就进入了`恢复模式`，当领导者被选举出来，且大多数server的完成了和leader的状态同步以后，恢复模式就结束了。

状态同步保证了leader和server具有相同的系统状态

一旦leader已经和多数的follower进行了状态同步后，他就可以开始广播消息了，即进入`广播状态`。这时候当一个server加入zookeeper服务中，它会在恢复模式下启动，发现leader，并和leader进行状态同步。待到同步结束，它也参与消息广播。Zookeeper服务一直维持在Broadcast状态，直到leader崩溃了或者leader失去了大部分的followers支持。

`广播模式`需要保证**proposal被按顺序处理**，因此zk**采用了递增的事务id号(zxid)来保证**。**所有的提议(proposal)都在被提出的时候加上了zxid。**

当leader崩溃或者leader失去大多数的follower，这时候zk进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的server都恢复到一个正确的状态。每个Server启动以后都询问其它的Server它要投票给谁。对于其他server的询问，server每次根据自己的状态都回复自己推荐的leader的id和上一次处理事务的zxid（系统启动时每个server都会推荐自己）收到所有Server回复以后，就计算出**zxid最大**的那个Server，并将这个Server相关信息设置成下一次要投票的Server。计算这过程中获得票数最多的的sever为获胜者，如果获胜者的票数超过半数，则改server被选为leader。否则，继续这个过程，直到leader被选举出来。leader就会开始等待server连接，Follower连接leader，将最大的zxid发送给leader，Leader根据follower的zxid确定同步点，完成同步后通知follower 已经成为uptodate状态，Follower收到uptodate消息后，又可以重新接受client的请求进行服务了

# zookeeper是如何避免脑裂的？

什么是脑裂？

集群（Master-Slave的情况）通常是发生在节点通信不可达(分区)的情况下，集群会分裂成不同的小集群，小集群各自选举出多个master节点的情况。

如何避免？

leader选举，要求节点的投票数量>总节点数量/2，即过半数，有这个选举原则保证了集群出现分区，无论如何最多只能有一个小集群选出leader。

M-S 模式的集群节点个数为何推荐是奇数个？

首先，偶数个节点的集群一旦出现对半分区(比如4个节点分区成两个节点和两个节点的情况)，整个集群无法选举出leader，集群无法提供服务。
其次，在容错能力相同的情况下，奇数节点比偶数节约资源。比如，5个节点挂了2个还能选出leader，而6个节点最多也只能挂2个节点才能保证选举出leader。

# zookeeper 是如何保证事务的顺序一致性的？

zookeeper 采用了全局递增的事务 Id 来标识，所有的 proposal（提议）都在被提出的时候加上了 zxid，zxid 实际上是一个 64 位的数字，高 32 位是 epoch（ 时期; 纪元; 世; 新时代）用来标识 leader 周期，如果有新的 leader 产生出来，epoch会自增，低 32 位用来递增计数。当新产生 proposal 的时候，会依据数据库的两阶段过程，首先会向其他的 server 发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。


# ZAB 和 Paxos 算法的联系与区别？

相同点：

（1）两者都存在一个类似于 Leader 进程的角色，由其负责协调多个 Follower 进程的运行

（2）Leader 进程都会等待超过半数的 Follower 做出正确的反馈后，才会将一个提案进行提交

（3）ZAB 协议中，每个 Proposal 中都包含一个 epoch 值来代表当前的 Leader周期，Paxos 中名字为 Ballot

不同点：

ZAB 用来构建高可用的分布式数据主备系统（Zookeeper），Paxos 是用来构建分布式一致性状态机系统。

# zookeeper怎么做同步的

https://www.cnblogs.com/youngchaolin/p/13211752.html#_label1