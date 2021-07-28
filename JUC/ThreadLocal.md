## 使用场景

- 每个线程需要一个独享的对象（通常是<font color='red'>工具类</font>，比如SimpleDateFormat和Random，）
- <font color='red'>每个线程内</font>需要保存全局变量（比如拦截器中获取用户信息），可以让不同方法直接使用，<font color='red'>避免参数传递</font>的麻烦。ThreadLocal存储用户的信息，使得后面方法调用可以直接从ThreadLocal里面取

<font color='green'>补充</font>：数据库连接池Connection也是通过threadLocal存的

Class.forName("com.mysql.jdbc.Driver");
java.sql.Connection conn = DriverManager.getConnection(jdbcUrl);

数据库连接池是多次Drivermanager.getConnection(jdbcurl)，获取多个connection放入hashmap中。因为每次获取Connection再释放耗费CPU资源

如果是新线程，则将新的connection放在threadlocal里，再get给到线程。保证每个线程从连接池中获得的都是线程自己的connection。

## 好处

- 达到<font color='red'>线程安全</font>
- <font color='red'>不需要加锁</font>，提高执行效率
- 高效<font color='red'>利用内存</font>、<font color='red'>节省开销</font>：不用每个任务都新建SimpleDateFormat
- 免去传参麻烦

## 原理

![image-20200831190806442](D:\Typora\picture\image-20200831190806442.png)

**<font color='red'>ThreadLocalMap是Thread类的一个成员变量</font>**，所以key-value是保存在线程中而不是ThreadLocal中

```java 
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```



## ThreadLocalMap

### 重要属性

```java
/**
 * 初始容量，必须是2的次方数
 */
private static final int INITIAL_CAPACITY = 16;

/**
 * 存放数据的Entry数组，大小必须为2的幂
 */
private Entry[] table;

/**
 * 数组里面entrys的个数
 */
private int size = 0;

/**
 * 扩容阈值，超过阈值进行扩容
 */
private int threshold; // Default to 0

/**
 * 初始阈值定义为长度的2/3
 */
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

/**
 * 环形意义的下一个索引，开放定地址法解决hash冲突
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

/**
 * 环形意义的上一个索引
 */
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

ThreadLocal有两个方法用于得到上一个/下一个索引，注意这里实际上是环形意义下的上一个与下一个。原因是**ThreadLocalMap使用<font color='red'>线性探测法</font>来解决Hash冲突，所以实际上Entry[]数组在程序逻辑上是作为一个环形存在的。**

线性探测法就是从冲突的数组单元开始，依次往后搜索空单元，如果到数组尾部，再从头开始搜索（环形查找）。所以线性探测里面的下一个结点不是我们理解的+1。

![image-20200831203931170](D:\Typora\picture\image-20200831203931170.png)

### 存储结构Entry

它定义在ThreadLocal类里面，key是ThreadLocal对象，value是Object对象，存储单元是Entry

```java 
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

可以看到key是**<font color='red'>持有ThreadLocal实例的弱引用</font>**，使得ThreadLocal在没有其他强引用的时候被回收掉，这样可以避免因为线程得不到销毁导致ThreadLocal对象无法被回收。

ThreadLocal中并没有ThreadLocalMap的引用，是Thread引用的

### 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //创建16大小的Entry数组
    table = new Entry[INITIAL_CAPACITY]; 
    //计算数组下标索引
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    //因为是延迟加载，所有有第一个value的时候才创建map，因此firstValue存入map
    table[i] = new Entry(firstKey, firstValue);  
    size = 1;
    setThreshold(INITIAL_CAPACITY); //设置阈值
}
```

### Hash函数

`int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`

```java
private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
private static AtomicInteger nextHashCode =
    new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647; 
```

可以知道这里它**<font color='blue'>并不是使用ThreadLocal对象的hashCode()，而是重写了获取hash值方法</font>**

这个threadLocalHashCode在该ThreadLocal被构造的时候就会生成，相当于threadLocalHashCode的ID。而这个值来源于nextHashCode()方法。

当前threadLocalHashCode的ID是在<font color='blue'>上一个被构造出来的ID的基础上加上0x61c88647</font>，这个数的选取与斐波那契数列有关，这样的话目的是得到的结果很平均。

ThreadLocalMap使用的是**线性探测法**，均匀分布的好处在于很快就能探测到下一个临近的可用slot，从而保证效率。

### getEntry()

```Java
private Entry getEntry(ThreadLocal<?> key) {
    //获得索引下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    //对应的entry存在且未失效且弱引用指向的ThreadLocal就是key，则命中返回
    if (e != null && e.get() == key)
        return e;
    else
        // 未找到，但是因为用的是线性探测，所以往后找还是有可能能够找到目标Entry的。
        return getEntryAfterMiss(key, i, e);
}
============================================================================
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
	//基于线性探测法不断向后探测直到遇到空entry
    while (e != null) {
        ThreadLocal<?> k = e.get();
        //找到目标
        if (k == key)
            return e;
        if (k == null)
    //该entry对应的ThreadLocal已被回收，调用expungeStaleEntry清理无效的entry！！！
            expungeStaleEntry(i);
        else
            //环形数组嘛，环形意义下的下一个结点！
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

**总结一下：**

1. 根据入参threadLocal的threadLocalHashCode得到对应的下标index
2. 如果index对应的slot就是需要的threadLocal，直接返回
3. 如果不是，调用getEntryAfterMiss线性探测，过程中每次碰到无效的slot，调用expungeStaleEntry进行清理，如果找到了key，则返回
4. 没找到key，返回null

### set()

```Java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
	//线性探测table查找key
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
		//找到对应的key，覆盖就的Entry
        if (k == key) {
            e.value = value;
            return;
        }
		// threadLocal的强引用已被释放，无法通过这个key获取entry，存在内存泄漏可能性
        if (k == null) {
            // 用当前插入的值替换失效的key
            replaceStaleEntry(key, value, i);
            return;
        }
    }
	// 没有entry就新建一个
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //如果启发式清理没有无效的slot，且数组容量大于阈值了，那进行rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
============================================================================
============================================================================
private void rehash() {
    // 做一次全量清理
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    //因为做了清理，size可能变小。这里的实现是调低阈值来判断是否需要扩容，
    //threshold默认为len*2/3，所以这里的threshold - threshold / 4相当于len/2
    if (size >= threshold - threshold / 4)
        resize(); //扩容
} 
```

**总结一下：**

- 探测过程中slot都不无效，并且顺利找到key所在的slot，直接替换即可
- 探测过程中发现有无效slot，调用replaceStaleEntry，效果是最终一定会把key和value放在这个slot，并且**会尽可能清理无效slot**
  - 在replaceStaleEntry过程中，如果找到了key，则把它放到那个无效slot中，value置为新值
  - 在replaceStaleEntry过程中，没有找到key，直接在无效slot原地放entry
- 探测没有发现key，则在连续段末尾的后一个空位置放上entry，这也是线性探测法的一部分。放完后，做一次启发式清理，如果没清理出去key，并且当前table大小已经超过阈值了，则做一次rehash，rehash函数会调用一次全量清理slot方法也即expungeStaleEntries，如果完了之后table大小超过了threshold - threshold / 4，则进行扩容2倍

#### replaceStaleEntry

```Java
/* 替换失效的key */
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                   int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    int slotToExpunge = staleSlot;
    // 向前扫描，寻找最前一个无效的slot
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    //向后遍历table
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 找到了key，将其与无效的slot交换
        if (k == key) {
            // 更新对应slot的value值
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            //如果向前扫描找到了最前的无效slot，则从那个位置开始做一次清理，否则从i位置开始做清理
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果当前的entry是脏entry，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    // 如果key在table中不存在，则在原地放一个即可
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    // 在探测过程中如果发现任何无效slot，则做一次清理（连续段清理+启发式清理）
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```
首先replaceStaleEntry并不仅仅局限于处理当前已知的脏entry，它认为在出**现脏entry的相邻位置也有很大概率出现脏entry，所以为了一次处理到位，就需要向前环形搜索，找到最前面的脏entry**

1. 前向有脏Entry
   1. 后向环形查找找到可覆盖的entry

      ![](https://upload-images.jianshu.io/upload_images/2615789-ebc60645134a0342.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

   2. 后向环形查找未找到可覆盖的entry

      ![img](https://upload-images.jianshu.io/upload_images/2615789-423c8c8dfb2e9557.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

2. 前向没有脏entry，向后找第一个脏entry

   1. 后向环形查找找到可覆盖的entry。

      ![img](https://upload-images.jianshu.io/upload_images/2615789-018d077773a019dc.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

   2. 后向环形查找未找到可覆盖的entry

      ![img](https://upload-images.jianshu.io/upload_images/2615789-eee96f3eca481ae0.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

      

### 清理

#### 全量清理：expungeStaleEntry()

```Java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    //不是环形数组那样遍历，而是直接遍历table
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

#### 连续段清理slot：expungeStaleEntry(int staleSlot)

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    //1. 这个entry对应的threadlocal已被回收，那么value设置为null，手动断开强引用
    tab[staleSlot].value = null;
    //手动设置entry为null，以便垃圾回收
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    //2. 不要停下来，继续从staleSlot开始向后遍历环形数组
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 清理对应ThreadLocal已经被回收的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //如果这是一个正常数据，重新计算它在slot中的下标
            int h = k.threadLocalHashCode & (len - 1);
            //如果下标不等于当前下标，说明它之前遇到了hash冲突，现在可能之前那个占位的被清除了，重新给它找一个位置存放！！！这位置更接近正确的index
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;java
}
```

#### 启发式清理：cleanSomeSlots

```java
/**
i对应entry是非无效（指向的ThreadLocal没被回收，或者entry本身为空）
n是用于控制控制扫描次数的
正常情况下如果log n次扫描没有发现无效slot，函数就结束了
但是如果发现了无效的slot，将n置为table的长度len，做一次连续段的清理
再从下一个空的slot开始继续扫描

这个函数有两处地方会被调用，一处是插入的时候可能会被调用，另外个是在替换无效slot的时候可能会被调用，
区别是前者传入的n为元素个数，后者为table的容量
*/
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // i在任何情况下自己都不会是一个无效slot，所以从下一个开始判断
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            //发现了无效的slot
            // 扩大扫描范围
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

### 扩容resize()

```Java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    //扩容扩大两倍
    int newLen = oldLen * 2; 
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                //计算在新数组中的下标位置
                int h = k.threadLocalHashCode & (newLen - 1);
                //槽已经被占用，线性探测下一个空的槽
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### remove()

```Java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 找到
        if (e.get() == key) {
            // 弱引用断开，引用赋值null
            e.clear();
            // 做一次连续段清理
            expungeStaleEntry(i);
            return;
        }
    }
}
```

## initialValue()

- 该方法返回当前线程对应的“初始值”，是一个<font color='red'>延迟加载</font>的方法，只有<font color='red'>调用get()</font>的时候，才会触发。没重写的话，默认返回null。get方法如果没有get到map，返回setInitialValue()方法结果，如果之前已经有set方法，那么就不会调用initialValue

```Java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    //得到当前线程的map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        //当前线程还没有map，就创建一个map
        createMap(t, value);
    return value;
}
======================================================================
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
======================================================================
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

可见调用的了initialValue方法。每个线程最多只调用一次initialValue方法，但是如果调用了remove方法，再调用get，则可以再次调用该方法

## set()

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

## get()

```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //当前线程是否有ThreadLocalMap，getMap就返回每个线程的成员变量threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //这个this就是一个ThreadLocal对象，通过它拿到Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //初始化
    return setInitialValue();
}
```

## remove()

```Java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

## 内存泄漏问题

引用关系

![img](https://upload-images.jianshu.io/upload_images/2615789-9107eeb7ad610325.jpg?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

如果threadLocal外部强引用被置为null(threadLocalInstance=null)的话，threadLocal实例就没有一条引用链路可达，很显然在gc的时候势必会被回收，因此entry就存在key为null的情况，无法通过一个Key为null去访问到该entry的value。同时，就存在了这样一条引用链：threadRef->currentThread->threadLocalMap->entry->valueRef->valueMemory,导致在垃圾回收的时候进行可达性分析的时候,value可达从而不会被回收掉，但是该value永远不能被访问到，这样就存在了**内存泄漏**。

当然线程结束的时候还是都会被回收的，但是实际中都是使用线程池，也就是说线程不会结束



```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
它的k赋值是使用super(k)，因此k是一个弱引用，ThreadLocal类k可以被回收。

**如何避免？**

调用remove方法，删除对应entry对象，避免内存泄漏，使用完ThreadLocal就要主动调用remove方法

**ThreadLocalMap的key设置为弱引用，防止内存泄漏，为什么value不设置成弱引用？**

因为不清楚这个value除了map是否还存在其他引用，如果不存在其他引用，GC的时候直接把value回收掉，但此时ThreadLocal还在使用，get的时候就会造成value是null的结果。但是ThreadLocal的话不一样，因为除了map的key弱引用它，他还被栈里的public static ThreadLocal threadlocal = ......强引用着。

## 为什么用弱引用？

假如每个key都强引用指向ThreadLocal的对象，也就是上图虚线那里是个强引用，那么这个ThreadLocal对象就会因为和Entry对象存在强引用关联而无法被GC回收，造成内存泄漏，除非线程结束后，线程被回收了，map也跟着回收。