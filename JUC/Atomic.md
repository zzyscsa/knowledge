# AtomicInteger

**cas+volatile实现的**

```Java
private volatile int value;  //volatile修饰保证可见性！！
```

Atomic类的底层都是使用unsafe包的工具实现的。Unsafe类是CAS的核心，Java无法直接访问底层操作系统，是通过本地（native）方法来访问。不过尽管如此，JVM还是开了个后门，JDK中的<font color='red'>Unsafe类提供了硬件级别的原子操作</font>

```Java
private static final Unsafe unsafe = Unsafe.getUnsafe(); //获得unsafe工具
private static final long valueOffset;
```

这个静态代码块，在类加载的时候就会被执行，从Atomic文件中获取了value的偏移量，**<font color='cornflowerblue'>valueOffset表示value在内存中的偏移地址，因为Unsafe是根据内存偏移地址获取数据原值的</font>**，这样我们就能通过unsafe来实现CAS

```Java
static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

## getAndSet(int newValue)

```java
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
```

调用的是**unsafe包**里面的getAndSetInt方法。

## compareAndSet(int expect, int update)

```Java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

## getAndIncrement()

```Java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

原理类似。

## getAndAddInt

看一看unsafe类的getAndAddInt

```Java
public final int getAndAddInt(Object o, long valueOffset, int modify) {
    int expect;
    do {
        expect = this.getIntVolatile(o, valueOffset);
    } while(!this.compareAndSwapInt(o, valueOffset, expect, expect + modify));
    return expect;
}
```

compareAndSwapInt明显就是一个CAS的体现，是一个native方法

我们看看expect获取到了什么，通过unsafe的getIntVolatile(o, valueOffset)，这是个navive方法，其实就是获取o类中，valueOffset偏移处的值，o就是AtomicInteger类，valueOffset就是value的偏移量，因此获取到的就是value的值。

重点是这个<font color='blue'>compareAndSwapInt(o, valueOffset, expect, expect + modify)</font>，解释一下，参数是：**AtomicInteger类、value的偏移量、期望值expect、需要修改之后的值。**

我们写成这样compareAndSwapInt(obj, value, expect, update)

如果obj类中value和expect相等，说明没有被其他线程该过，那么这个变量更新为update，如果这次CAS没有成功，那么采用自旋的方式继续CAS

所以getAndAddInt方法核心就是`自旋+CAS`

 **<font color='green'>那么对应的CPU指令是什么呢？</font>**

**<font color='red'>cmpxchg(x, addr, e)</font>**：x是原值，addr是需要cas变量的内存地址，e是更新的值

# AtomicIntegerFieldUpdater注意点

- 可见范围：因为用了反射，所以不支持private的变量
- 不支持static修饰的变量



# LongAdder

**<font color='red'>为什么LongAdder在高并发下比AtomicLong效率高？</font>**

因为AtomicLong线程每一次加法都需要刷新到主内存，然后另一个线程从主内存读取到本地内存，需要做一次同步。而LongAdder不需要，它每个线程有自己的计数器，然后汇总

LongAdder引入**分段累加**的概念，内部有一个base变量和Cell[]数组共同参与计数。

`base变量`：竞争不激烈，直接CAS累加到该变量上

`Cell[]数组`：竞争激烈，各个线程分散累加到自己的槽Cell[i]中。而Cell[]数组全局只有一个，每个线程根据hash计算出自己属于哪个槽，最后求和就是base值加上Cell[]数组里面的值

## sum方法

```Java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

## add方法

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // 如果是第一次执行，则直接cas操作base
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        //as数组为空(null或者size为0)
        //或者当前线程取模as数组大小为空
        //或者cas更新Cell失败。
        //就该扩容了
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

Cell数组是懒加载，在第一次发生竞争（CAS失败）就会创建一个大小为2的cells数组，每次扩容都是加倍（加倍操作由longAccumulate方法判断并执行），直到达到CPU核数。



在低争用情况下，两者效率差不多，高争用情况下，LongAdder效率高，但消耗更多空间。LongAdder适合统计求和计数的场景，而且LongAdder基本只提供add方法，但AtomicLong还有cas方法

# CAS

cas最终是要利用CPU的特殊指令，CPU保证了他们的原子性。