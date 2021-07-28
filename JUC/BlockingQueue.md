阻塞队列是具有阻塞功能的队列，一端是给生产者放数据用，另一端给消费者拿数据用。是**线程安全**的，生产者和消费者都可以多线程。

队列如果为空，生产者阻塞等，队列满，消费者阻塞

![image-20200913161410215](D:\Typora\picture\image-20200913161410215.png)

# ArrayBlockingQueue

## 构造方法

```Java
// 1. 传入队列长度
public ArrayBlockingQueue(int capacity) {
    this(capacity, false); //默认非公平锁
}
// 2. 传入队列长度、可重入锁是否是公平锁
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity]; // 初始化Object数组
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition(); //初始化信号量
    notFull =  lock.newCondition();
}
```

## 实例变量

```Java
//保存队列元素的数组
final Object[] items;

//取出元素的位置
int takeIndex;

//添加元素的位置
int putIndex;

//队列中元素的数量
int count;

//锁对象
final ReentrantLock lock;

//不空的信号量
private final Condition notEmpty;

//不满的信号量
private final Condition notFull;

/**
 * Shared state for currently active iterators, or null if there
 * are known not to be any.  Allows queue operations to update
 * iterator state.
 */
transient Itrs itrs = null;
```

## put()

```Java
public void put(E e) throws InterruptedException {
    checkNotNull(e); //判断元素是否为null
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); //可中断的获取锁
    try {
        while (count == items.length) //当前队列中元素如果等于队列(数组)长度
            notFull.await(); //阻塞当前线程
        enqueue(e); //不等就添加元素到队列
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    // 把待插入的元素放在数组的putIndex位置
    items[putIndex] = x;
    //如果执行完插入后putIndex等于数组的长度，说明队列已经满了
    if (++putIndex == items.length) 
        //把putIndex的值置为0，即下次插入的位置为0，下次要插入成功的必要条件是取出了一个元素，取出的位置为takeIndex。因为队列先进先出，所以下一次插入到0
        putIndex = 0;
    count++;
    notEmpty.signal(); //唤醒notEmpty阻塞的线程
}
```

## take()

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) //如果当前队列是空的
            notEmpty.await(); //取元素线程阻塞等待
        return dequeue(); //否则取出元素
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    // 取出takeIndex处的元素
    E x = (E) items[takeIndex]; 
    // 此位置引用置空
    items[takeIndex] = null;
    // 判断takeIndex长度如果等于数组长度，说明已经取到最后一个元素，下次从0开始取
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal(); //唤醒notFull阻塞的线程
    return x;
}
```

## offer/add、poll/peek

offer添加元素进队列，如果队列满，返回false；add如果队列满会抛出异常

poll会取出队头元素，如果队列空，返回null；peek返回队头元素但不取出

take和put方法都会阻塞直至成功或者被中断！

# LinkedBlockingQueue

- <font color='red'>无界</font>
- <font color='red'>容量Integer.MAX_VALUE</font>
- 内部结构：Node、<font color='red'>两把锁</font>

## 构造方法

```Java
// 1. 不指定容量默认就是Integer.MAX_VALUE，即无界队列
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
// 2. 指定队列容量大小
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    // 初始化last和head结点，都指向一个元素为null的结点
    last = head = new Node<E>(null);
}
```

## 重要属性

```Java
/**
 * 结点Node类，用于存储数据
 */
static class Node<E> {
    E item;

    Node<E> next; // 指向下一结点的指针

    Node(E x) { item = x; }
}

/** 阻塞队列大小，默认为Integer.MAX_VALUE */
private final int capacity;

/** 当前阻塞队列中元素的个数，是一个原子类 */
private final AtomicInteger count = new AtomicInteger();

/**
 * 阻塞队列头结点
 * Invariant: head.item == null
 */
transient Node<E> head;

/**
 * 阻塞队列尾结点
 * Invariant: last.next == null
 */
private transient Node<E> last;

/** 获取并移除元素时使用的锁，如take, poll... */
private final ReentrantLock takeLock = new ReentrantLock();

/** notEmpty条件对象，当队列没有数据时用于挂起执行删除的线程 */
private final Condition notEmpty = takeLock.newCondition();

/** 添加元素时使用的锁如 put, offer... */
private final ReentrantLock putLock = new ReentrantLock();

/** notFull条件对象，当队列数据已满时用于挂起执行添加的线程 */
private final Condition notFull = putLock.newCondition();
```

我们可以知道，<font color='cornflowerblue'>每个数据被封装成Node结点</font>，添加到链表队列中，head和last分别指向链表的头尾结点。值得注意的是，LinkedBlockingQueue用了**两把锁**，takeLock和putLock，这点和ArrayBlockingQueue不同，也就是说，删除和添加操作并不是互斥操作，可以同时进行。每一个Lock都提供了一个Condition用来挂起和唤醒其他线程。

**为什么要用两把锁？**

个人猜想这是设计者考虑到ArrayBlockingQueue底层是使用的数组，而LinkedBlockingQueue底层使用的是链表。LinkedBlockingQueue添加元素时有一个构造节点的时间，为了尽量减少这部分时间占比，使用一个读锁一个写锁可以实现并发存取的优化。而ArrayBlockingQueue在添加元素时不需要开辟空间等等（创建时指定数组大小），所以反而是加锁解锁时间占比较大，如果继续使用这种读写锁分离的设计，增加的时间损耗大于并发读写带来的收益。

## put()

```java 
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 获取添加元素的中断锁
    putLock.lockInterruptibly();
    try {
        // 判断队列是否已满，已满则阻塞等待
        while (count.get() == capacity) {
            notFull.await();
        }
        // 把node放入队列中
        enqueue(node);
        c = count.getAndIncrement(); //c获取的是count+1之前的值
        //再次判断是否有可用空间
        if (c + 1 < capacity)
            notFull.signal(); //队列不满，通知添加线程可以开始添加
    } finally {
        putLock.unlock();
    }
    // 放入元素之前队列如果是空的，那么可能有消费线程阻塞等待，那现在有一个新元素，唤醒消费线程消费
    if (c == 0)
        signalNotEmpty();
}

// 添加到链表尾部，并且更新last指针
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}

// 为什么这么写， 因为signal的时候需要获得该signal对应对象的锁
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

## take()

```Java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        // 队列为空，阻塞等待
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 取出元素
        x = dequeue();
        // c同样是count-1之前的值
        c = count.getAndDecrement();
        // 如果队列中还有值，唤醒下一个线程消费
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 如果之前c为队列容量，说明可能有添加线程阻塞了，唤醒添加线程
    if (c == capacity)
        signalNotFull();
    return x;
}

//head指向的队头结点出队
private E dequeue() {
    // 获取到head节点
    Node<E> h = head;
    // 获取到head节点指向的下一个节点
    Node<E> first = h.next;
    // head节点原来指向的节点的next指向自己，等待下次gc回收
    h.next = h; // help GC
    // head节点指向新的节点
    head = first;
    // 获取到新的head节点的item值
    E x = first.item;
    // 新head节点的item值设置为null
    first.item = null;
    return x;
}
```

总体而言，和put方法十分相似，只不过是逆向处理。

补充一点：为什么h.next不直接指向null而是h本身？

这其实和它的迭代器中的nextNode方法有关

```Java
private Node<E> nextNode(Node<E> p) {
    for (;;) {
        Node<E> s = p.next;
        if (s == p)
            return head.next;
        if (s == null || s.item != null)
            return s;
        p = s;
    }
}
```

为了正确遍历，使用了s == p判断，当下一个元素是自己本身时，返回head的下一个结点

# PriorityBlockingQueue

- 支持<font color='red'>优先级</font>
- 自然顺序（不是先进先出）
- 无界队列

# SynchronousQueue

- <font color='red'>容量为0</font>
- 不需要去持有元素，所做的就是直接传递
- 效率很高
- 是一个极好的用来<font color='red'>直接传递</font>的并发数据结构