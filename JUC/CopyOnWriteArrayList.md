CopyOnWriteArrayList允许并发读，读操作是无锁的；至于写操作，比如向容器中添加一个元素，则首先将当前容器复制一份，然后在新副本上执行写操作，结束之后再将原容器的引用指向新容器。

![img](https://images2015.cnblogs.com/blog/1024555/201705/1024555-20170521150621182-1289355418.png)

优点：读操作性能很高，因为无需任何同步措施，适用于<font color='red'>读多写少</font>的并发场景。

缺点：<font color='red'>一是内存占用</font>，毕竟需要拷贝；<font color='red'>二是无法保证实时性</font>，Vector对于读写操作均加锁同步，可以保证读和写的强一致性。而CopyOnWriteArrayList读写在不同容器上，写操作的执行过程中，读不会阻塞但是读取的是老容器的数据。

## 初始化

```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);   //创建一个大小为0的Object数组作为array初始值
}
final void setArray(Object[] a) {
    array = a; //array就是一个全局的Object数组
}
```

## add()

```Java
public boolean add(E e) {
    //加锁，保证线程安全
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //拷贝原来数组，长度为原数组+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //在副本上执行添加操作
        newElements[len] = e;
        //将原数组引用指向新的副本数组
        setArray(newElements);
        return true;
    } finally {
        //解锁
        lock.unlock();
    }
}
```

## get()

```Java
public E get(int index) {
    return get(getArray(), index);
}
private E get(Object[] a, int index) {
    return (E) a[index];
}
```

直接读就好了，也不用加锁。。。简单

## remove()

```java 
public E remove(int index) {
    //加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            //如果要删除的是列表末端数据，拷贝前len-1个数据到新副本上，再切换引用
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            //否则，将除要删除元素之外的其他元素拷贝到新副本中，并切换引用
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

总之删除也是把除了要删除的那个元素之外的所有元素拷贝到新的副本中，然后再切换引用