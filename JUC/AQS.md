## 如果没有AQS

需要每个协作工具类自己实现

- 同步状态的原子性管理
- 线程的阻塞与解除阻塞
- 队列的管理

在并发场景下，自己正确且高效实现这些内容，相当有难度，所以AQS帮我们做了这些，我们只需关注业务逻辑

## AQS作用

- 是一个用于构建锁、同步器、协作工具类的工具类（框架）。有了AQS，更多的协作工具类都可以被很方便写出来。
- 有了AQS，构建线程协作类就容易多了。

## AQS三大核心部分

### **<font color='red'>state</font>**

```java
/**
 * The synchronization state.
 */
private volatile int state;
```

state的具体含义，会根据具体实现类的不同而不同，比如在Semaphore里，它表示“<font color='red'>剩余的许可证数量</font>”，而在CountDownLatch里，它表示“<font color='red'>还需要倒数的数量</font>”

state是<font color='red'>volatile</font>修饰，会被并发地修改，所以所有修改state的方法都需要<font color='red'>保证线程安全</font>，比如**getState、setState**以及**compareAndSetState**操作来读取和更新这个状态。这些方法都依赖于atomic包

```Java
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

在<font color='red'>ReentrantLock</font>中，state用来表示“锁”占有（1）情况，包括<font color='red'>可重入计数</font>，当state值为0，标识该锁处于释放状态

### 控制线程抢锁和配合的**<font color='red'>FIFO队列</font>**

这个队列用来<font color='red'>存放“等待的线程”</font>，AQS就是“排队管理器”，当多个线程争用同一把锁时，必须有排队机制将那些没有拿到锁的线程串在一起。锁释放时，锁管理器就会挑选一个合适的线程来占有这个刚刚释放的锁。

AQS会维护一个等待的线程队列，把线程都放到这个队列里，这个队列是一个<font color='red'>双向链表</font>的形式

### 期望协作工具类去实现的**<font color='red'>获取/释放</font>**等重要方法

这里获取和释放方法，是利用AQS的协作工具类里做重要的方法，是<font color='red'>由协作工具类自己实现</font>，并且<font color='red'>含义各不相同</font>

- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

#### 获取方法

- 获取操作依赖于state变量（判断是不是0），经常会阻塞（比如获取不到锁的时候）
- 在<font color='red'>Semaphore</font>中，获取就是<font color='red'>acquire</font>方法，作用是<font color='red'>获取一个许可证</font>
- 而在<font color='red'>CountDownLatch</font>里，获取就是<font color='red'>await</font>方法，作用是<font color='red'>等待直到倒数结束</font>，直到其他方法把state减为0就被唤醒

#### 释放方法

- 释放操作不会阻塞
- 在<font color='red'>Semaphore</font>中，获取就是<font color='red'>release</font>方法，作用是<font color='red'>释放一个许可证</font>
- 而在<font color='red'>CountDownLatch</font>里，获取就是<font color='red'>countDown</font>方法，作用是<font color='red'>倒数1个数</font>



## AQS的用法

1. 写一个类，想好协作逻辑，实现释放/获取方法
2. 内部写一个<font color='red'>内部类Sync类继承AbstractQueuedSynchronizer</font>
3. 根据是否独占来<font color='red'>重写tryAcquire/tryRelease或tryAcquireShared(int acquires)/tryReleaseShared(int releases)</font>等方法，在之前写的获取/释放方法中<font color='red'>调用</font>AQS的acquire/release或者Shared方法。分别对应AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行）和Share（共享，多个线程可同时执行）
   1. tryAcquire/tryRelease：ReentrantLock、ReentrantReadWriteLock
   2. tryAcquireShared(int acquires)/tryReleaseShared(int releases)：CountDownLatch、Semaphore

## AQS在CountDownLatch

### 构造方法

把AQS中的state设置为count

```Java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
Sync(int count) {
    setState(count);
}
protected final void setState(int newState) {
    state = newState;
}
```

### getCount

```java
public long getCount() {
    return sync.getCount();
}
//Sync类
int getCount() {
    return getState();
}
//AQS类
protected final int getState() {
    return state;
}

```

### await()

```Java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
//AQS类
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //中断判断
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg); //当前线程加入等待队列
}

//countdownlatch类中的Sync类实现了tryAcquireShared
protected int tryAcquireShared(int acquires) {
    // 剩余倒数的数量如果=0，说明可以放行了，返回1，表示不需要加入等待队列了！
    // 如果不等于0，说明不可以放行，返回-1，AQS类中当前线程加入等待队列
    return (getState() == 0) ? 1 : -1;
}
```

### countdown()

```java 
public void countDown() {
    sync.releaseShared(1);
}
//AQS类
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared(); //唤醒队列所有线程
        return true;
    }
    return false;
}

//countdownlatch类中的Sync类实现了tryReleaseShared
// 主要工作将count-1
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) //CAS更新count值
            return nextc == 0; //表示这次倒数之后，放行所有线程
    }
}
```

## AQS在Semaphore

### acquire

```java 
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
//同理来到Semaphore的tryAcquireShared方法
public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}
//Syn类
//返回值正数代表获取成功，负数代表获取失败
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        //许可证数量
        int available = getState(); 
        // 当前获取acquires许可证之后剩下的许可证
        int remaining = available - acquires;
        if (remaining < 0 || //如果剩余许可证不够，直接返回负数
            compareAndSetState(available, remaining)) //否则CAS设置当前剩余的值，改变state状态，如果CAS失败，继续下一次尝试。
            return remaining;
    }
}
```

## AQS在ReentrantLock

### unlock

```java 
public void unlock() {
    sync.release(1);
}
// AQS类
public final boolean release(int arg) {
    // true代表锁被释放，false代表锁未被释放
    if (tryRelease(arg)) {
        Node h = head;
        // 从后面等待的节点中去唤醒线程
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
//ReentrantLock类中Sync内部类
protected final boolean tryRelease(int releases) {
    // 重入次数-1
    int c = getState() - releases;
    // 判断当前线程是否是持有锁的线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false; //表示这把锁不是自由
    // c==0表示重入次数为0，需要解锁
    if (c == 0) {
        free = true;//表示这把锁是自由的状态
        // 设置当前持有这把锁的线程
        setExclusiveOwnerThread(null);
    }
    // 更新重入次数
    setState(c);
    return free;
}
```

### lock

以非公平为例

```java
public void lock() {
    sync.lock();
}
final void lock() {
    // CAS设置state值为1
    if (compareAndSetState(0, 1))
        // 当前线程设置为持有锁线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
//AQS类
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //添加到队列中！！！EXCLUSIVE代表是一个互斥锁
        selfInterrupt();
}
//ReentrantLock类
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 判断锁是否被持有
    if (c == 0) {
        // 未被持有，CAS设置state
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程是持有本锁的线程，重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires; //重入次数+1
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## AQS源码

### doAcquireSharedInterruptibly

把当前线程加入等待队列（双向链表）

```Java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 当前需要等待获取资源的线程包装成链表的结点Node，包含了其本身及其等待状态，如是否被阻塞、是否等待唤醒、是否已经被取消等。
    final Node node = addWaiter(Node.SHARED); 
    boolean failed = true;
    try {
        for (;;) {
            // 获取当前结点的前置结点
            final Node p = node.predecessor();
            // 判断前置结点是否是head结点（前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock)；则再次调用 tryAcquireShared 尝试获取一下
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取lock成功, 设置新的head, 并唤醒后继获取readLock的节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //调用 shouldParkAfterFailedAcquire 判断是否需要中断
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) //parkAndCheckInterrupt方法就是让当前线程阻塞挂起
                //若此次唤醒是 通过线程中断, 则直接抛出异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            //清除 node 节点
            cancelAcquire(node);
    }
}
```

### addWaiter

```java
private Node addWaiter(Node mode) {
    //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
    Node node = new Node(Thread.currentThread(), mode);

    //尝试快速方式直接放到队尾。
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }

    //上一步失败则通过enq入队。
    enq(node);
    return node;
}

private Node enq(final Node node) {
    //CAS"自旋"，直到成功加入队尾
    for (;;) {
        Node t = tail;
        if (t == null) { // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {//正常流程，放入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

### acquireQueued(Node, int)

这个函数让线程在等待队列中休息，直到其他线程释放了资源后在唤醒他

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//标记是否成功拿到资源
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过

        //又是一个“自旋”！
        for (;;) {
            final Node p = node.predecessor();//拿到前驱
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                failed = false; // 成功获取资源
                return interrupted;//返回等待过程中是否被中断过
            }

            //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
        }
    } finally {
        if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}
```