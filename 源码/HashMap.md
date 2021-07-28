# HashMap

> HashMap继承于抽象类AbstractMap，实现Map接口、Cloneable接口、Serializable接口，AbstractMap实现Map接口。

## Node数据结构

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
```

这个hash就是经过扰动的hash，之后再说。每一个值都封装为一个Node

## 底层存储结构初探

![image-20200715220710066](D:\Typora\picture\image-20200715220710066.png)

链表长度达到8，并且当前数组长度大于64，转换成红黑树。



## 核心属性

-  默认数组table大小是16

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

- table最大长度

```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```

- 缺省负载因子0.75（为什么？）

这是一个**提高空间利用率**和**减少查询成本**的折中

StackOverFlow有这么一个回答， 一个bucket空和非空的概率为0.5，通过牛顿二项式等数学计算，得到这个loadfactor的值为log（2），约等于0.693. 同回答者所说，可能小于0.75 大于等于log（2）的factor都能提供更好的性能。

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

- 树化阈值，超过这个值变红黑树（为什么是8）

HashMap源码的注释中告诉我们

```java
/* Because TreeNodes are about twice the size of regular nodes, we
* use them only when bins contain enough nodes to warrant use
* (see TREEIFY_THRESHOLD). And when they become too small (due to
* removal or resizing) they are converted back to plain bins.  In
* usages with well-distributed user hashCodes, tree bins are
* rarely used.  Ideally, under random hashCodes, the frequency of
* nodes in bins follows a Poisson distribution
* (http://en.wikipedia.org/wiki/Poisson_distribution) with a
* parameter of about 0.5 on average for the default resizing
* threshold of 0.75, although with a large variance because of
* resizing granularity. Ignoring variance, the expected
* occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
* factorial(k)). The first values are:
*
* 0:    0.60653066
* 1:    0.30326533
* 2:    0.07581633
* 3:    0.01263606
* 4:    0.00157952
* 5:    0.00015795
* 6:    0.00001316
* 7:    0.00000094
* 8:    0.00000006
* more: less than 1 in ten million
```

TreeNode节点是常规节点的两倍，很占用空间。

就是说使用随机哈希码，节点出现的频率在hash桶中遵循泊松分布，同时给出了桶中元素个数和概率的对照表。当桶中元素大于8的时候，概率就非常小了。因此选择0.75的时候，每个碰撞位置的链表长度超过８个是几乎不可能的。

```java
static final int TREEIFY_THRESHOLD = 8;
```

- 树降级成为链表的阈值（在哈希表扩容时，如果发现链表长度小于 6，则会由树重新退化为链表）

```java
static final int UNTREEIFY_THRESHOLD = 6;
```

- 树化的另一个参数。所有元素达到64个才能够升级为Tree

```java
static final int MIN_TREEIFY_CAPACITY = 64;
```

- 哈希表、大小、结构修改次数

```java
transient Node<K,V>[] table;
transient int size;
transient int modCount; //插入一个元素，删除一个元素都会计数，但是替换不会计数
```

- 扩容阈值，超过就扩容

```java
int threshold;
```

- 负载因子：threshold = capacity * loadFactor

```java
final float loadFactor;
```

## 构造方法

有四个构造方法

- 传递初始容量和负载因子

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    //数组长度超过最大值不行，直接给你最大值
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //负载因子必须大于0
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //因为数组长度必须是2的次方数
    this.threshold = tableSizeFor(initialCapacity);
}

//tableSizeFor返回一个大于等于当前值cap的数字，并且这个数字是2的次方数
static final int tableSizeFor(int cap) {
    int n = cap - 1; //如果没有减一会得到cap的两倍结果
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

- 只传递容量

```java
public HashMap(int initialCapacity) {
	this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

- 无参（使用默认值）

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

- 使用一个Map构造

```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

## put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

put一个key为null，就放在数组下标0。

先来看一下这个`扰动函数hash()`

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

作用：**让key的值的高16位也参与运算**。**减少碰撞，进一步降低hash冲突的几率**

再来看一下这个`putVal方法`

```java
	/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     如果key存在了，那么就不插入了，一般都是false，有则特换，没有就插入新的
     在putIfAbsent这个方法里面传的是true
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    //tab：引用当前HashMap的散列表
    //p：当前散列表的元素
    //n：散列表数组的长度
    //i：寻址的结果值
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    //第一次put插入数据的时候才初始化最耗费内存的散列表！延迟初始化逻辑
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    //这里的(n - 1) & hash就是寻址算法！如果这个地址对应的桶位是null，就直接把kv的Node存进去
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //p就是if中赋值的当前散列表中的对应元素，说明这个桶位的元素和当前的元素key一致，需要在之后进行替换操作
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //就是当前桶位已经红黑树了！
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //链表的情况！
            for (int binCount = 0; ; ++binCount) {
                //循环找到了最后一个位置，都没找到和我key一致的，那就加到链表后面
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //循环从0开始，因此大于等于7就说明大于8个了，需要红黑树树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到了链表中e和我key一致，break出去准备做替换
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //替换操作
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //散列表结构被修改，可以看到替换操作是不会增加这个值的！
    ++modCount;
    //size自增，如果大于扩容阈值，需要扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**可以看到，是put第一个值的时候，才初始化散列表！**

## resize方法

```java
final Node<K,V>[] resize() {
    //oldTab：引用扩容之前的table
    Node<K,V>[] oldTab = table;
    //oldCap：表示扩容之前table数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //oldThr：扩容之前的阈值，触发本次扩容的阈值
    int oldThr = threshold;
    //newCap：扩容之后的大小；newThr：扩容之后的扩容阈值
    int newCap, newThr = 0;
    //>0表示散列表已经初始化过了，现在是一次正常扩容
    if (oldCap > 0) {
        //如果大于了table允许的最大值，把扩容阈值设置为最大，表示不再扩容了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //oldCap左移1位，也就是翻倍赋值给newCap，而且oldCap大于等于16，那么新的阈值是原来阈值的翻倍。
        //这里如果你初始化map有指定容量并且小于16那么第二个条件就不满足了
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 	oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //oldCap==0，说明还未初始化
    //除了无参构造，其他三种构造都是有给threshold赋值的
    //赋值给newCap，相当于指定了table的大小！
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //oldCap==0，使用无参构造
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY; //16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); //12
    }
    //也是一样的，指定了table的值，那么就要计算新的扩容阈值是多少
    //两种情况：1. 上面oldCap >= DEFAULT_INITIAL_CAPACITY条件不满足
    //	      2. 使用了除了无参构造外的三种构造方法初始化
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && 
  		        ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //说明扩容之前，table不为null
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //说明当前这个桶位里面有数据：有可能是单个数据，链表，红黑树
            if ((e = oldTab[j]) != null) {
                //这个桶位指向null，方便GC回收内存
                oldTab[j] = null;
                
                //说明是单个数据
                if (e.next == null)
                    //算出在新数组中的位置
                    newTab[e.hash & (newCap - 1)] = e;
                
                //说明是红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                
                //链表的情况
                //主要思想是把当前链表拆成两个链表。比如当前长度是16，那么15位置上的所有元素共同特点就是hash值的最后4位都是1111，那么这个时候&上(32-1)，后五位会出现两种情况：01111和11111，就分别放在新table的15位置和31位置！
                else { // preserve order
                    //低位链表：存放扩容之后数组下标和当前数组下标一致！
                    Node<K,V> loHead = null, loTail = null;
                    //高位链表：存放扩容之后数组下标：当前数组下标+扩容之前数组长度！
                    Node<K,V> hiHead = null, hiTail = null;
                    
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //这里就比较巧妙了。使用原来的数组长度(16：10000)，就可以区							分出来倒数第五位的0和1！这样就拆分出高位链和低位链
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        //低位链末尾指向null，因为老链表可能指向着其他元素
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```



## get方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //这里不仅判断key是否相同，还判断这个元素的hash是否相同，就是防止你有时候没有重写自己的hashcode()
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //说明当前桶位不止一个元素
        if ((e = first.next) != null) {
            //第二种情况：桶位升级成了红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //第三种情况：桶位形成链表，遍历链表查找呗！
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



## remove方法

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
```

```java
//matchValue如果是true表示key和value都必须匹配上才删除
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //找到桶位位置赋值给p
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        //说明桶位是有数据的，需要进行查找操作，并且删除
        //node：查找到的结果；e：当前node的下一个元素
        Node<K,V> node = null, e; K k; V v;
        //第一种情况：当前桶位元素即为需要删除的元素。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;  
		//表示当前桶位不止一个元素，可能红黑树或链表
        else if ((e = p.next) != null) {
            //第二种情况：红黑树
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //第三种情况：链表。遍历查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //如果查找到了需要删除的元素，matchValue就是控制是否走后面的逻辑
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            //第一种情况：红黑树的删除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            //第二种情况：删除的元素是当前桶位的第一个元素
            else if (node == p)
                tab[index] = node.next;
            //第三种情况，删除链表元素
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

## replace方法

```java
@Override
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}

@Override
public V replace(K key, V value) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
    return null;
}
```

# HashMap中的红黑树

## TreeNode数据结构

hashmap中，红黑树的节点是一个TreeNode结构

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    //指向前一个节点，原链表中的前一个节点
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;  //是否是红色
```

TreeNode继承自LinkedHashMap的内部类LinkedHashMap.Entry，这个Entry又继承自Node。



## 树化

在putVal中，桶位元素超过了8个，就树化

### treeifyBin

```java
/**
* Replaces all linked nodes in bin at index for given hash unless
//除非表太小，也就是小于64个元素，就不树化，改为resize扩容
* table is too small, in which case resizes instead.
*/
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果数组长度小于64，那么就先扩容吧
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //找到桶位的第一个元素e
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //hd是双向链表的第一个节点
        TreeNode<K,V> hd = null, tl = null;
        do {
            //把Node节点替换为TreeNode节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //如果t1为，null，说明当前树节点就是第一个根节点！
            if (tl == null)
                hd = p;
            //这里把这个单链表转换乘双向链表结构了，t1是p的前驱，p是当前节点
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        //对这个TreeNode链表进行树化！并且数组index位置就是双向链表的第一个元素hd
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

//把Node节点替换为TreeNode节点
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}

```

以上的操作是把链表中的Node节点全部转换为TreeNode节点，并转换成双向链表！然后再进行树化

### treeify

```java
final void treeify(Node<K,V>[] tab) {
    //树的根节点
    TreeNode<K,V> root = null;
    //x是当前节点，next是后继节点
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        //如果根节点是null，当前节点设置为根节点，是黑色的！
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            //循环遍历，找到插入位置，进行红黑树的插入
            //x：待插入的节点；k：x的key；h：x的hash
            //p：遍历中的当前节点
            for (TreeNode<K,V> p = root;;) {
                //dir：表示x节点于p节点的比较，-1表示比p小，1表示比p大。不存在相等情况，因为hashmap不存在两个key完全一致的情况
                //ph：p的hash；pk：p的key
                int dir, ph;
                K pk = p.key;
                //要插入的x比p小，dir=-1
                if ((ph = p.hash) > h)
                    dir = -1;
                //要插入的x比p大，dir=1
                else if (ph < h)
                    dir = 1;
                //如果x和p的hash值相等，那么判断k是否实现了comparable接口，如果实现了comparable接口就使用compareTo进行比较，如果仍旧相等或者没有实现comparable接口，则在tieBreakOrder中比较。总之我无论如何都要给你比出一个结果！
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                //x比p小就去左边，比p大就去右边，判断是否是null，如果是null就该插入了
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    //进入循环p就是null了
                    //xp就是x的父节点！
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    //进行插入自平衡处理！
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    //确保给定节点是桶中的第一个元素
    moveRootToFront(tab, root);
}

//这里不是为了整体排序，而是为了在插入中保持一致的顺序
static int tieBreakOrder(Object a, Object b) {
    int d;
    //用两者的类名进行比较，如果相同则使用对象默认的hashcode进行比较
    if (a == null || b == null ||
        (d = a.getClass().getName().
         compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
             -1 : 1);
    return d;
}
```

这里的逻辑就是，循环遍历当前树，然后找到可以该节点可以插入的位置，依次和遍历节点比较，比它大则跟其右孩子比较，小则与其左孩子比较，依次遍历，直到找到左孩子或者右孩子为null的位置进行插入。然后自平衡处理balanceInsertion

### moveRootToFront

```java
//将root节点移动到桶中的第一个元素，也就是链表的首节点
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        //first指向链表第一个节点
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        //如果root不是第一个节点，则将root放到第一个首节点位置
        if (root != first) {
            Node<K,V> rn;
            tab[index] = root;
            //这里的操作是把root这个节点从双向链表中提出来，插入到链表的头部！
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            if (first != null)
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        //这里是防御性编程，校验更改后的结构是否满足红黑树和双链表的特性
        //因为HashMap并没有做并发安全处理，可能在并发场景中意外破坏了结构
        assert checkInvariants(root);
    }
}
```

这么看来，红黑树的TreeNode里面还维护着一个双向链表，在插入删除的时候都需要维护这个链表，那么这个链表有啥用呢？



## TreeNode的左旋和右旋

### 左旋

![img](https://img-blog.csdnimg.cn/20181125155223734.gif)

```java
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                      TreeNode<K,V> p) {
	//p：对应上图的E节点
    //r：E的右节点S
    //pp：p的父节点，即E的父节点
    //rl：r的左节点，即S的左节点，需要移动的
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        //p的右节点指向r的左节点，并且rl指向这个节点，联系gif图
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        //将p的父节点的孩子节点指向r
        if ((pp = r.parent = p.parent) == null)
            //如果没有pp节点，那么这个节点就是根节点！要染成黑色！
            (root = r).red = false;
        //如果有pp节点，那么就把原来的p节点替换成r节点
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}
```

### 右旋

![img](https://img-blog.csdnimg.cn/20181125160017452.gif)

```java
//方法和左旋一样的
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                       TreeNode<K,V> p) {
    //p：对应上图的S节点
    //l：S的左节点E
    //pp：p的父节点，即S的父节点
    //lr：l的右节点，即E的右节点，需要移动的
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) {
        if ((lr = p.left = l.right) != null)
            lr.parent = p;
        if ((pp = l.parent = p.parent) == null)
            (root = l).red = false;
        else if (pp.right == p)
            pp.right = l;
        else
            pp.left = l;
        l.right = p;
        p.parent = l;
    }
    return root;
}
```

## TreeNode的插入

### putTreeVal

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    //标识是否已经遍历过一次树，未必是从根节点遍历的，但是遍历路径上一定已经包含了后续需要比对的所有节点。
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;
    //循环遍历，找到插入位置，进行红黑树的插入，和treeify里面的基本一样，有些许不同
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        //这表示当前节点和当前元素的key一致，那么返回，外层的putVal会对其替换值操作
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
		
        //key不一样，但是hash一样，那么继续比较
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            //重写了comparable接口使用compareTo进行比较后还是相同，那么遍历树查找是否有key相同的节点，遍历只遍历一次，通过searched来控制

            //左右子树递归遍历查找，是否有和key的hash相同并且equals相同的节点
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                //调用find方法进行递归查找，如果没有找到，就返回null
                //短路运算，左侧找到了右侧就不需要找了！
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            //到这里说明找不到与之key相同的节点，下次也不会遍历树查找了！
            dir = tieBreakOrder(k, pk);
        }

        
        TreeNode<K,V> xp = p;
        //在这里p赋值成下一个节点了，如果不为null，那么继续走大循环查找！！
        //如果是null，那么当前位置就是插入位置，创建新的节点插入！
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            //当前节点的next节点
            Node<K,V> xpn = xp.next; 
            //没找到一样的节点，必须新建节点插入！
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            //平衡之后把root节点移到链表头部！
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

### 插入自平衡balanceInsertion

```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    //插入的元素默认是红色！
    x.red = true;
    //xp：插入节点父节点
    //xpp：插入节点爷爷节点
    //xppl：插入节点的叔叔（left）节点
    //xppr：插入节点的叔叔（right）节点
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        //情况1：插入节点的父节点为null，说明我就是根节点，染成黑色返回
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        //情况2：父节点是黑色或者爷爷节点是null，那么刚好满足红黑树条件，返回
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        //情况3：父节点是红色
        //情况3.1：父节点是爷爷节点的左节点
        if (xp == (xppl = xpp.left)) {
            //情况3.1.1：如果叔叔节点存在且为红色
            if ((xppr = xpp.right) != null && xppr.red) {
                //叔叔设置为黑色
                xppr.red = false;
                //父节点设置为黑色
                xp.red = false;
                //爷爷设置为红色
                xpp.red = true;
                //爷爷节点设置为当前节点
                x = xpp;
            }
            //情况3.1.2：叔叔节点不存在或者为黑色
            else {
                //情况3.1.2.1：新插入节点是其父节点的右节点（LR双红）
                //父节点xp左旋，p设置为新插入节点x！进入LL双红的操作
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    //给xp，xpp重新赋值下，因为更新了插入节点
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                //情况3.1.2.2：LL双红
                if (xp != null) {
                    //1. 变色：父节点设置为黑色，爷爷设置为红色
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        //2. 对爷爷节点右旋
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        //情况3.2：父节点是爷爷节点的右节点
        else {
            //情况3.2.1：叔叔节点存在且为红色
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            //情况3.2.2：叔叔节点不存在或者为黑色
            else {
                //情况3.2.2.1：新插入结点，是其父结点的左节点（RL双红）
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                //情况3.2.2.2：RR双红
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

## TreeNode的get获取

```java
final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
}

//find函数在putTreeVal中也有用到，就是遍历整棵树，寻找对应key的值
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}
```



## TreeNode的扩容

在resize()方法中，如果遇到了这个桶位已经升级为了红黑树，那么会调用split方法

```java
//说明是红黑树
else if (e instanceof TreeNode)
    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
```

我们就来看一看这个split方法

```java
//map：当前hashmap
//tab：扩容后的新Node数组
//index：当前桶位在老Node数组里面的下标位置
//bit：扩容之前老数组的长度
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    //这个for循环的操作就是拆分成高低位的TreeNode链表，和链表的操作是相同的
    //这就是TreeNode节点之间还维护着双向链表的原因！！
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        //如果低位链表长度小于等于6，那么就把红黑树结构转换为链表结构
        if (lc <= UNTREEIFY_THRESHOLD)
            //将原索引位置的节点设置为对应的头结点
            tab[index] = loHead.untreeify(map);
        else {
            //如果长度大于6，那么继续保持红黑树结构
            tab[index] = loHead;
            //如果高位链不为null，那么说明拆分了，需要重新构建红黑树，否则没有拆分，继续维持即可
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

这样我们就知道了为什么TreeNode还要维护双向链表，就是为了方便红黑树扩容的时候拆分！

区别在于**<font color='green'>拆分完会判断链长度是否大于6，若大于则继续红黑树，否则就转换成链表！</font>**



## TreeNode删除

### removeTreeNode

```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                          boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    if (pred == null)
        tab[index] = first = succ;
    else
        pred.next = succ;
    if (succ != null)
        succ.prev = pred;
    if (first == null)
        return;
    if (root.parent != null)
        root = root.root();
    if (root == null || root.right == null ||
        (rl = root.left) == null || rl.left == null) {
        //转换为普通链表结构
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        TreeNode<K,V> s = pr, sl;
        //找到后继节点
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        TreeNode<K,V> sr = s.right;
        TreeNode<K,V> pp = p.parent;
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        }
        else {
            TreeNode<K,V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;
    //p是待删除节点，replacement用于后续的红黑树调整，指向的是p或者p的替代者。
    //如果p是叶子节点，p==replacement，否则replacement为p的替代节点
    if (replacement != p) {
        //若p不是叶子节点，则让replacement的父节点指向p的父节点
        TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    //若待删除的节点p时红色的，则树平衡未被破坏，无需进行调整。
    //否则删除节点后需要进行调整
    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    //p为叶子节点，则直接将p从树中清除
    if (replacement == p) {  // detach
        TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        moveRootToFront(tab, r);
}
```

### balanceDeletion

```java
//x：待删除节点的替换节点（最终需要删除的节点）
//xp：替换节点的父节点
//xpl：替换节点的兄弟节点（x为父节点的右节点时）
//xpr：替换节点的兄弟节点（x为父节点的左节点时）
//sl：兄弟节点的左节点
//sr：兄弟节点的右节点
static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                           TreeNode<K,V> x) {
    for (TreeNode<K,V> xp, xpl, xpr;;)  {
        if (x == null || x == root)
            return root;
        //父节点null，说明替换节点是根节点，染成黑色，直接返回
        else if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        //替换节点是红色，不需要调整，直接返回
        else if (x.red) {
            x.red = false;
            return root;
        }
        //替换节点是父节点的左节点
        else if ((xpl = xp.left) == x) {
            //有兄弟节点且兄弟节点是红色
            if ((xpr = xp.right) != null && xpr.red) {
                //兄弟节点设置为黑色
                xpr.red = false;
                //父节点设置为红色
                xp.red = true;
                //对父节点左旋
                root = rotateLeft(root, xp);
                //重新xp指向x的父节点，xpr指向xp的新右节点，准备进入接下来的替换节点兄弟节点的子节点都是黑色的情况！
                xpr = (xp = x.parent) == null ? null : xp.right;
            }
            //没有兄弟节点，向上继续调整
            if (xpr == null)
                x = xp;
            else {
                //替换节点的兄弟节点是黑色，得到兄弟节点的左右子节点sl、sr
                TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                //兄弟节点左右节点都是黑色
                if ((sr == null || !sr.red) &&
                    (sl == null || !sl.red)) {
                    //兄弟节点设置为红色
                    xpr.red = true;
                    //父节点作为删除节点重新进行处理
                    x = xp;
                }
                else {
                    //兄弟节点右节点是黑色，左节点是红色
                    if (sr == null || !sr.red) {
                        if (sl != null)
                            //兄弟左节点设置为黑色
                            sl.red = false;
                        //兄弟节点设置为红色
                        xpr.red = true;
                        //对兄弟节点右旋
                        root = rotateRight(root, xpr);
                        //重新设置下连接关系，准备进入兄弟节点右节点是红色，左节点任意颜色的情况！
                        xpr = (xp = x.parent) == null ?
                            null : xp.right;
                    }
                    //兄弟节点右节点是红色，左节点任意颜色
                    if (xpr != null) {
                        //兄弟的颜色设置为父节点的颜色
                        xpr.red = (xp == null) ? false : xp.red;
                        if ((sr = xpr.right) != null)
                            //兄弟右节点设置为黑色
                            sr.red = false;
                    }
                    if (xp != null) {
                        //父节点设置为黑色
                        xp.red = false;
                        //对父节点左旋
                        root = rotateLeft(root, xp);
                    }
                    x = root;
                }
            }
        }
        //替换节点是父节点的右节点，和上面的相反
        else { // symmetric
            if (xpl != null && xpl.red) {
                xpl.red = false;
                xp.red = true;
                root = rotateRight(root, xp);
                xpl = (xp = x.parent) == null ? null : xp.left;
            }
            if (xpl == null)
                x = xp;
            else {
                TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                if ((sl == null || !sl.red) &&
                    (sr == null || !sr.red)) {
                    xpl.red = true;
                    x = xp;
                }
                else {
                    if (sl == null || !sl.red) {
                        if (sr != null)
                            sr.red = false;
                        xpl.red = true;
                        root = rotateLeft(root, xpl);
                        xpl = (xp = x.parent) == null ?
                            null : xp.left;
                    }
                    if (xpl != null) {
                        xpl.red = (xp == null) ? false : xp.red;
                        if ((sl = xpl.left) != null)
                            sl.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateRight(root, xp);
                    }
                    x = root;
                }
            }
        }
    }
}
```





# JDK1.7和JDK1.8HashMap的区别

- 1.7的结构是数组+链表；1.8是数组+链表+红黑树
- 1.7封装的对象是Entry；1.8是Node
- 1.7的扩容时机是当前存放的元素大于阈值并且存放新值的时候这个桶位存在hash碰撞；1.8的扩容时机是当前元素个数大于阈值或者是当前链表里面元素大于8个且数组长度小于64
- 1.7的扩容方法是建立一个新数组，然后把原来的元素存放到新数组中，采用的是头插法，会有成环风险；1.8是把链表拆成高低位链，使得元素更加分散，并且采用的是尾插法