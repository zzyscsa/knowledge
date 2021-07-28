## 关于分布式锁

使用set命令，key是锁，value是一个随机值，还要设置过期时间

解锁比较value的值判断是否是同一个人加的锁，如果是就解锁，但是这两步不是原子的，因此使用lua脚本。

```java
@Component
public class RedisLock {

    private String LOCK_KEY = "redis_lock";

    protected long INTERNAL_LOCK_LEASE_TIME = 3;

    private long timeout = 1000;

    private SetParams params = SetParams.setParams().nx().px(INTERNAL_LOCK_LEASE_TIME);

    JedisPool jedisPool = new JedisPool("127.0.0.1", 6379);

    /**
     * 加锁
     *
     * @param id
     * @return
     */
    public boolean  lock(String id) {
        Long start = System.currentTimeMillis();
        Jedis jedis = jedisPool.getResource();
        try {
            for (; ; ) {
                //SET命令返回OK ，则证明获取锁成功
                String lock = jedis.set(LOCK_KEY, id, params);
                if ("OK".equals(lock)) {
                    return true;
                }
                //否则循环等待，在timeout时间内仍未获取到锁，则获取失败
                long l = System.currentTimeMillis() - start;
                if (l >= timeout) {
                    return false;
                }
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        } finally {
            jedis.close();
        }
    }

    /**
     * 解锁
     *
     * @param id
     * @return
     */
    public boolean unlock(String id) {
        Jedis jedis = jedisPool.getResource();
        String script =
            "if redis.call('get',KEYS[1]) == ARGV[1] then" +
            "   return redis.call('del',KEYS[1]) " +
            "else" +
            "   return 0 " +
            "end";
        try {
            String result = jedis.eval(script, Collections.singletonList(LOCK_KEY), Collections.singletonList(id)).toString();
            return "1".equals(result) ? true : false;
        } finally {
            jedis.close();
        }
    }
}
```

以上在单机Redis中没什么问题。

但是在Redis集群中，使用的是**Redisson**，Redission也是一个Redis客户端。

Redission封装了锁的实现，其继承了java.util.concurrent.locks.Lock的接口，让我们像操作我们的本地Lock一样去操作Redission的Lock。

![img](https://user-gold-cdn.xitu.io/2018/10/8/16652989862ce5af?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

说说这个tryLock：

这个tryLock方法尝试加锁，为了保证原子性，使用的是lua脚本。他并没有使用我们的sexNx来进行操作，而是使用的hash结构，我们的每一个需要锁定的资源都可以看做是一个HashMap，锁定资源的节点信息是Key,锁定次数是value。通过这种方式可以很好的实现可重入的效果，只需要对value进行加1操作，就能进行可重入锁。

如果尝试加锁失败，判断是否超时，如果超时则返回false。

如果加锁失败之后，没有超时，那么需要在名字为redisson_lock__channel+lockName的channel上进行**订阅**，用于订阅解锁消息，然后一直阻塞直到超时，或者有解锁消息。



解锁操作也是通过lua脚本进行解锁，如果是可重入锁，只是减1。如果是非加锁线程解锁，那么解锁失败。



如果想象一个这样的场景当机器A申请到一把锁之后，如果Redis主宕机了，这个时候从机并没有同步到这一把锁，那么机器B再次申请的时候就会再次申请到这把锁，为了解决这个问题Redis作者提出了`RedLock红锁`的算法,在Redission中也对RedLock进行了实现。

![img](https://user-gold-cdn.xitu.io/2018/10/8/16652bd95e11a8b3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- 首先生成多个Redis集群的Rlock，并将其构造成RedLock。

- 依次循环对三个集群进行加锁

- 如果循环加锁的过程中加锁失败，那么需要判断加锁失败的次数是否超出了最大值，这里最大值是根据集群的个数，比如三个就只允许失败一个，五个的话只允许失败两个，要保证多数成功。

- 加锁的过程中需要判断是否加锁超时，有可能我们设置加锁只能用3ms，第一个集群加锁已经消耗了3ms了。那么也算加锁失败。
- 3，4步里面加锁失败的话，那么就会进行解锁操作，解锁会对所有的集群在请求一次解锁。

RedLock基本原理是利用多个Redis集群，用多数的集群加锁成功，减少Redis某个集群出故障，造成分布式锁出现问题的概率。

## 如果其他线程要删除redis锁怎么办

可以存key-value的时候value存线程的ID号，删除的时候取出value比较一下是否是自己的线程ID，如果是的话就删除，不是就不能删除。

## 如何保证Redis中的数据都是热点数据

redis 内存数据集大小上升到一定大小的时候，会施行数据淘汰策略。

redis 提供 6种数据淘汰策略：

redis 内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。redis 提供 6种数据淘汰策略：
volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
no-enviction（驱逐）：禁止驱逐数据

## 为什么Redis单线程还能支持高并发

1. 完全基于内存操作，加上其特有的数据结构（优化的数据结构）使得其访问速度非常快。
2. 单线程模式单省却了CPU上下文切换带来的开销问题，也不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗。
3. 多路IO复用，使其可以可以高效的处理多个连接请求，既支持了高并发，同时也减少了网络IO的消耗时间。
4. Redis采用自己实现的事件分离器，效率比较高，内部采用非阻塞的执行方式，吞吐能力比较大

## 布隆过滤器常见场景

- 防止Redis缓存穿透
- 爬虫过滤已抓到的url就不再抓
- 垃圾邮件过滤

## 保证数据库和redis缓存一致性

使用先更新数据库，再删除缓存的方法。

会出现几种情形：

- A线程进行写操作，更新数据库，还未淘汰缓存；B线程从缓存中可以读取到旧数据，**此时数据不一致**，A线程完成淘汰缓存操作，这时候数据一致，因此数据不一致时间很短，最终是一致的
- A进行写操作，但是更新较慢，缓存也没有淘汰；此时B进行读操作，读到的是旧数据，这是缓存和数据库都是旧数据，不存在不一致情况
- 缓存没有数据，B从数据库读数据，还未写入缓存，这时候A更新数据库，然后将数据写入缓存，这时候B读到的已经是旧数据，它再次写入缓存，**造成数据不一致**。不过由于读操作比写操作快，这种情况发生概率极低

在这种方法下，唯一会产生的问题是：

`如果删除缓存失败了，就会造成数据不一致`

解决办法可以是使用“重试机制”，如果删除失败就报错，然后重试删除操作。重试可以通过MQ实现



## 为什么Redis的key不能过长

大key相关的删除或者自动过期时，会出现qps突降或者突升的情况，极端情况下会造成主从复制异常，Redis服务阻塞无法响应请求，因为大key删除耗时长而Redis又是单线程的



## redis对于lru的优化

Redis 使用了一种基于采样的近似LRU算法来替代普通的LRU算法，从而避免普通的LRU算法带来的过高的资源消耗，而像Redis这样的程序，并不需要驱逐精确的那个最久没有访问的键。

简述：**Redis随机采样一小部分键值，并选中其中最久没有访问的键，然后淘汰。**

