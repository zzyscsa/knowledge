# JDK1.7

<img src="D:\Typora\picture\image-20200910214806437.png" alt="image-20200910214806437" style="zoom:67%;" />

- 在JDK1.7中，最外层是多个<font color='red'>segment</font>，每个segment的底层数据结构与HashMap类似，仍然是数组和链表组成的<font color='red'>拉链法</font>
- 每个segment<font color='red'>独立上ReentrantLock锁</font>，每个segment之间互不影响，提高了并发效率
- 默认有16个segment，因此<font color='red'>最多可以同时支持16个线程并发写</font>（操作分别分布在不同segment上），默认值初始化可以设置，但是初始化之后不可扩容

# JDK1.8

完全抛弃了1.7中的设计，采用的是<font color='red'>CAS+synchronized</font>来保证线程安全！结构和HashMap基本一样

![image-20200910215446004](D:\Typora\picture\image-20200910215446004.png)

## 重要属性（大多与HashMap相同）

```Java
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final int DEFAULT_CAPACITY = 16;
//默认并发数16
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
private static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64; 
//每次进行转移的最小值
private static final int MIN_TRANSFER_STRIDE = 16;
//生成sizeCtl所使用的bit位数
private static int RESIZE_STAMP_BITS = 16;
// 进行扩容所允许的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
static final int MOVED     = -1; // 表示正在转移扩容
static final int TREEBIN   = -2; // 表示已经转换成树
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

//数组，用来保存元素
transient volatile Node<K,V>[] table;
//转移时候用的数组
private transient volatile Node<K,V>[] nextTable;
//用来控制table初始化和扩容的，默认值为0，如果还未初始化，sizeCtl值为需要初始化的大小；初始化完成或者在初始化的时候指定了大小，sizeCtl为table大小，大小为table的0.75
//当为负的时候，说明table正在初始化或扩容（-1表初始化；-(1+n)表是活动的n个扩容线程）
private transient volatile int sizeCtl;
```

## Node

注意volatile修饰

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }
```

## ForwardingNode

一个特殊的Node节点，<font color='blue'>hash值为-1</font>，其中存储nextTable的引用。
只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动。

```java
final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}
```



## unSafe类的方法

```Java
/*
     * 用来返回节点数组的指定位置的节点的原子操作
     */
@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

/*
     * cas原子操作，在指定位置设定值
     */
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
/*
     * 原子操作，在指定位置设定值
     */
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

## 构造方法

### 无参构造

```Java
public ConcurrentHashMap() {}
```

### 有参构造（传递容量）

```Java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1)); //把传进来的值变成2的n次方
    this.sizeCtl = cap;
}
```

### 有参构造（传递Map）

```java 
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
```

## 初始化

在第一次put操作的时候，才对存储Map元素Node的table进行初始化

```Java
/**
     * 初始化数组table，
     * 如果sizeCtl小于0，说明别的数组正在进行初始化，则让出执行权
     * 如果sizeCtl大于0的话，则初始化一个大小为sizeCtl的数组
     * 否则的话初始化一个默认大小(16)的数组
     * 然后设置sizeCtl的值为数组长度的0.75
*/
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            //如果一个线程发现sizeCtl<0，意味着另外的线程执行CAS操作成功，当前线程只需要让出cpu时间片 ！！！！
            Thread.yield(); // lost initialization race; just spin
        //设置sizeCtl=-1表示正在初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { 
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## put方法

```Java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key不允许为null，和hashmap不同
    if (key == null || value == null) throw new NullPointerException();
    //计算key的hash值
    int hash = spread(key.hashCode());
    //用来保存这个结点共有多少个元素，用来控制扩容或者转移为树
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //table是否初始化
        if (tab == null || (n = tab.length) == 0)
            //第一次put的时候table没有初始化，则进行初始化工作
            tab = initTable();
        //当前位置桶位是null，直接通过cas放进去（没有加锁！）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //创建一个Node添加到数组中去，null表示下一个结点为null
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果当前结点的hash值是MOVED，说明正在进行数组扩张的数据复制阶段
        //当前线程也会参与去复制，通过允许多线程复制的功能，一次来减少数组的复制所带来的性能损失
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
       // 如果在这个位置有元素的话，就采用synchronized的方式加锁！！！，
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //如果当前元素hash大于0，说明是链表
                    if (fh >= 0) {
                        binCount = 1;
                        //遍历链表，同时binCount记录这个结点有多少个元素
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
          //要存的元素的hash，key跟要存储的位置的结点相同的时候，直接替换这个结点value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //没找到相同的，把创建新Node加到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果是红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //当一个结点的数目达到8个，扩张数组或将结点的数据转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //对binCount加1，这个方法还会检查是否超过了sizeCtl阈值需要扩容，或者是正在扩容，如果需要扩容就调用transfer扩容方法，如果正在扩容就帮助扩容，把扩容线程数+1
    addCount(1L, binCount);
    return null;
}
```

总结一下：

1. 当put一个结点的时候，首先判断是否初始化，如果没有就初始化数组

2. 通过计算hash判断放在数组哪个位置，如果这个位置为空则直接<font color='red'>**CAS添加**</font>，不为空就取出这个结点

3. 假如取出来的结点hash值是MOVED(-1)，则说明正在扩容复制到新数组，那么当前线程也帮助扩容

4. 最后一种情况，不在扩容也不为空，则通过<font color='red'>**synchronized来加锁**</font>，进行添加操作
   
   1. 当前取出的结点时链表，那么遍历，如果发现key和hash相同的就替换value，如果没发现就添加到链表末尾
   2. 如果是树，则用putTreeVal方法把此元素添加到树中去
   
   添加完成后，判断该槽处一共有多少个结点（注意是添加前的个数），达到8个以上调用treeifyBin尝试转换为树或者扩容

另外注意一点，获取table对应的索引元素f通过了<font color='red'>tabAt方法</font>

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

为什么不用table[index]，因为是ConcurrentHashMap的弱一致性，java每个线程都有一个工作内存，存着table的副本，虽然table是volatile修饰，但不能保证线程每次都拿到table中的最新元素，Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。

## 树化treeifyBin()

```Java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        //数组长度小于64，数组先扩容一倍
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) 
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            //使用synchronized加锁，将该槽处的链表转化为树
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                //把Node组成的链表，转化为TreeNode的链表，头结点任然放在相同的位置
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    //把TreeNode的链表放入容器TreeBin中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

## 扩容

<font color='red'>扩容的时机：</font>

1. 新增节点后，链表元素达到阈值8，调用treeifyBin转换红黑树的时候，发现数组长度小于阈值64，那么调用tryPresize方法把数组长度扩大到原来的两倍，触发transfer方法，重新调整结点位置。
2. 新增节点后，调用addCount方法记录元素个数，并检查是否需要扩容，当数组元素个数达到阈值sizeCtl时，触发transfer方法，重新调整节点位置。

### tryPresize

```Java
private final void tryPresize(int size) {
    // 如果给定大小大于等于数组最大容量的一半，则直接使用最大容量，否则通过tableSizeFor算出一个2的次方的容量
    //后面table一直要扩容到这个值小于等于sizeCtl(数组长度的3/4)才退出扩容
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        //如果还未初始化，则初始化
        if (tab == null || (n = tab.length) == 0) {
            //初始化大小为sizeCtl和刚刚算出来的c中较大的那个大小
            n = (sc > c) ? sc : c;
//为什么扩张的时候需要有初始化？因为如果不是put单个元素，而是调用putAll方法直接put一个map，那么就没有调用initTable方法去初始化table！
            //把sizeCtl=-1表示正在初始化
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); //数组大小的0.75倍
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        //c <= sc说明已经被扩容过了；n >= MAXIMUM_CAPACITY说明table数组已达到最大容量
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            // 根据容量n生成一个随机数，唯一标识本次扩容操作
            int rs = resizeStamp(n);
            //表明有别的线程正在扩容table，则帮助一起扩容；否则开启新的扩容
            if (sc < 0) {
                Node<K,V>[] nt;
                // 如果当前线程无法协助进行数据转移, 则退出
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //扩容线程数+1，该线程也帮忙扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // sc置为负数, 当前线程自身成为第一个执行transfer(数据转移)的线程
            //通过CAS保证仅有一个线程能够  发起扩容  ！
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                //将第一个参数table中的元素移动到第二个元素的table中去，第二个参数为null时，会创建一个两倍大小的table
                transfer(tab, null);
        }
    }
}
```

在tryPresize方法中，并没有加锁，允许多个线程进入，如果数组正在扩张，则当前线程也去帮助扩容。

### transfer()数据迁移

```Java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // stride可理解成“步长”，即数据迁移时，每个线程要负责旧table中的多少个桶
    // 如果桶较少，那么默认一个CPU（一个线程）处理16个桶
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {    //首次扩容        // initiating
        try {
            // 新建一个两倍长度的数组nextTable
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 更新转移下标，就是 旧的table 的 length
        // [transferIndex-stride, transferIndex-1]表示当前线程要进行数据迁移的桶区间！！
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //初始化ForwardingNode结点，保存了新数组nextTable的引用，在处理完每个槽位的结点之后当作占位结点，表示该槽位已经处理过了；因为多线程都来扩容嘛
    //当旧table的某个桶中的所有结点都迁移完后，用该结点占据这个桶
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 标识一个桶的迁移工作是否完成，advance = true 表示可以进行下一个位置的迁移
    boolean advance = true; 
    // 最后一个数据迁移的线程将该值置为true，并进行本轮扩容的收尾工作
    boolean finishing = false; 
    //for循环处理每个槽位中的链表元素，i指当前处理的桶索引，从大到小
    //bound 表示当前线程可以处理的当前桶区间最小下标，超过这个下标就需要重新领取区间或者结束扩容。
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //每一次自旋前的预处理，主要是定位本轮处理的桶区间
        //正常情况下，预处理完成后：i == transferIndex-1，bound == transferIndex-stride
        //advance表示是否继续递减转移下一个桶，如果为false表示还没有处理好当前桶
        while (advance) {
            int nextIndex, nextBound;
            //如果i-1大于等于bound，表示还需要继续做任务或者刚好完成，修改推进状态为false，表示不能推进，任务成功后才改为true
            if (--i >= bound || finishing)
                advance = false;
            //如果小于0，说明没有区间了，扩容结束了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //通过CAS设置transferIndex属性，给每个线程分配区间
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // CASE1：当前是处理最后一个tranfer任务的线程或出现扩容冲突
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果所有的节点都已经完成复制工作  就把nextTable赋值给table 清空临时对象nextTable
            if (finishing) {
                nextTable = null;
                table = nextTab;
                //扩容阈值设置为原来容量的1.5倍  依然相当于现在容量的0.75倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 利用CAS方法更新这个扩容阈值，在这里面sizectl值减一，说明当前线程已完成自己的transfer任务！
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 判断当前线程是否是本轮扩容中的最后一个线程，如果不是，则直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                /**
                 * 最后一个数据迁移线程要重新检查一次旧table中的所有桶，看是否都被正确迁移到新table了：
                 * ①正常情况下，重新检查时，旧table的所有桶都应该是ForwardingNode;
                 * ②特殊情况下，比如扩容冲突(多个线程申请到了同一个transfer任务)，此时当前线程领取的任务会作废，那么最后检查时，还要处理因为作废而没有被迁移的桶，把它们正确迁移到新table中 */
                i = n; // recheck before commit
            }
        }
        // CASE2：旧桶本身为null，不用迁移，直接则通过CAS插入初始化的ForwardingNode结点告诉其他线程该槽位已经处理过
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // CASE3：该桶已经被处理，取到该节点(ForwardingNode)的hash为MOVED，直接跳过，处理下一个，这是核心！！！！
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // CASE4：该旧桶未迁移完成，进行数据迁移，从这里才开始迁移，之前都是计算下标和处理区间，迁移的过程是同步的
        else {
            // 加锁处理链表结构，和hashmap类似，分为高低链
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // CASE4.1：桶的hash>0，说明是链表迁移
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //构造两个链表  一个是原链表  另一个是原链表的反序排列
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //在nextTable的i位置上插入一个链表 
                        setTabAt(nextTab, i, ln);
                        //在nextTable的i+n的位置上插入另一个链表
                        setTabAt(nextTab, i + n, hn);
                        // 设置ForwardingNode占位
                        setTabAt(tab, i, fwd);
                        // 表示当前旧桶的结点已迁移完毕
                        advance = true;
                    }
                    // CASE4.2：红黑树迁移，也是构造红黑树结点的高低链
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

如何在扩容时，并发地复制与插入？

1. 遍历整个table，当前节点为空，则采用CAS的方式在当前位置放入fwd
2. <font color='red'>当前节点已经为fwd(with hash field “MOVED”)，则已经有有线程处理完了了，直接跳过</font> ，这里是控制并发扩容的核心
3. 当前节点为链表节点或红黑树，重新计算链表节点的hash值，移动到nextTable相应的位置（构建了一个反序链表和顺序链表，分别放置在i和i+n的位置上）。移动完成后，用`Unsafe.putObjectVolatile`在tab的原位置赋为为fwd, 表示当前节点已经完成扩容。
4. 将表拆分，每个线程处理自己的区间



扩容的时候，可不可以对数组进行读写？

是可以的。当在进行数组扩容的时候，如果当前节点还没有被处理（也就是说还没有设置为fwd节点），那就可以进行设置操作。

如果该节点已经被处理了，则当前线程也会加入到扩容的操作中去。

## get方法

```Java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //计算key的hash值
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

如果eh小于0，说明是被扩容移动过，调用ForwardingNode的find方法找

发现一个问题，get操作没有一处加锁！如何保证读到的不是脏数据呢？

原因是**<font color='red'>Node元素的val和next都是volatile修饰！</font>**在多线程环境下线程A修改结点的val或者新增节点的时候是对线程B可见的。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
```

那么我们发现在Node数组上也加了volatile

```java
transient volatile Node<K,V>[] table;
```

这并不能保证数组元素可见性，只能保证数组地址可见性。目的在于扩容的时候对其他线程具有可见性。

