

# ArrayList

> 继承于AbstractList类，实现List接口，RandomAccess接口（可随机访问），Cloneable接口（可拷贝），Serializable接口（可序列化）；AbstractList又继承于AbstractCollection类，AbstractCollection实现Collection接口

## 核心属性

- 默认初始化空间10

```java
private static final int DEFAULT_CAPACITY = 10;
```

- 用于空对象初始化，如果使用 0 作为初始化容量的时候，ArrayList就会使用这个对象来初始化

```java
private static final Object[] EMPTY_ELEMENTDATA = {};
```

- 也是初始化空数组，用于无参构造（jdk8）， 如果使用默认的构造函数的话，就会使用这个对象初始化

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

- 存储数组，底层就是一个Object数组结构！

```java
transient Object[] elementData;
```

- 最大数组容量

```Java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

## 构造方法

- 无参构造

```java
public ArrayList() {
    //jdk1.8中如果是无参构造，使用的是DEFAULTCAPACITY_EMPTY_ELEMENTDATA空数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

- 传递初始容量

```Java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        //创建一个Object数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        //jdk1.8中，如果给定初始大小是0，使用的是EMPTY_ELEMENTDATA空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

- 传递一个集合来初始化

```Java
public ArrayList(Collection<? extends E> c) {
    //调用Collection接口的toArray方法，返回一个数组
    elementData = c.toArray();
    //如果集合非空
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        //是否成功转换成Object类型数组，比如通过Arrays.asList转换的List通过toArray转换的数组还是原来的类型
        if (elementData.getClass() != Object[].class)
            //如果不是Object类型数组就进行复制，仅复制实际数据长度
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        //如果集合是空，那么直接设置一个空数组
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

## add&addAll方法

```Java
public boolean add(E e) {
    //判断是否能容纳size+1个元素
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

//指定位置添加元素
public void add(int index, E element) {
    //判断下标index是否越界
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //index之后的元素进行一个拷贝，全部往后挪一位
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

来看看判断数组是否能容纳元素调用的方法

```Java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //判断是否是无参构造产生的空数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        //如果是那么就给他默认容量10进行扩容，为什么不是直接给定10而是取最大值呢，因为有可能一开始就是用addAll添加了很多元素。
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
private void ensureExplicitCapacity(int minCapacity) {
    modCount++; //操作次数+1

    // overflow-conscious code
    //如果数组长度小于minCapacity，那么就进行扩容！
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

addAll方法大体和add方法差不多

```Java
public boolean addAll(Collection<? extends E> c) {
    //集合转换成数组
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    //从elementData数组末尾开始往后复制a数组里的元素
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        //先把elementData数组index后的元素挪一挪，给a数组的元素腾出空间
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

## 扩容grow方法

```Java
private void grow(int minCapacity) {
    // overflow-conscious code
    //oldCapacity：老数组容量
    //newCapacity：新数组容量
    int oldCapacity = elementData.length;
    //新容量是旧容量的1.5倍！
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //变成旧容量的1.5倍后还是小于指定容量，那么新容量直接等于指定容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //新容量大于数组指定的最大容量，那么要通过hugeCapacity进行判断
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    //copyOf返回一个新数组，把老数组的元素拷贝到新数组中
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    //如果指定容量大于指定最大容量，那么扩容量就是Integer最大值，否则就是指定的最大容量
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

## get方法

简单

```Java
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

## remove删除元素

```Java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        //把index后面的元素往前移一位
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //最后一位设置为null，方便GC回收
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

## indexOf方法

```java 
//遍历数组查找元素，如果o不是null，就是用类的equals方法比较
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
//和上面那个方法一样，不过是从后往前
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

## jdk1.8和jdk1.7区别

1.8底层数组初始化后是0，**<font color='orange'>添加第一个元素的时候长度变为10</font>**，之后按照1.5倍扩容，因为1.8多用了DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个空数组来判断是否是无参构造产生的，类似于懒汉式创建集合容器；而1.7默认构造直接指定了数组长度是10，类似于饿汉式创建集合容器。

# LinkedList

> 继承AbstractSequentialList类，实现List接口，Deque接口，Cloneable接口，Serializable接口。Deque接口又继承于Queue接口，AbstractSequentialList又继承于AbstractList类

## Node数据结构

通过Node来存储集合中的元素

```Java
//可以看到，是一个双向链表的节点！
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 核心属性

- 链表头结点

```Java
transient Node<E> first;
```

- 链表尾节点

```Java
transient Node<E> last;
```

- 集合大小

```Java
transient int size = 0;
```

## 构造方法

- 无参

```Java
public LinkedList() {
}
```

- 有参，传递一个Collection集合

```Java
public LinkedList(Collection<? extends E> c) {
    this(); //使用无参构造
    addAll(c); //然后添加元素进去
}
```

## add&addAll方法

```Java
public boolean add(E e) {
    linkLast(e);
    return true;
}
//把元素插入到链表的末尾！
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

**addAll方法**

```Java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
//指定位置开始添加一个集合的所有元素
public boolean addAll(int index, Collection<? extends E> c) {
    //判断index是否越界
    checkPositionIndex(index);

    //集合转换为数组！
    Object[] a = c.toArray();
    //集合的大小
    int numNew = a.length;
    //大小为0，直接返回，不需要添加
    if (numNew == 0)
        return false;

    //pred：待添加节点的前一个节点
    //succ：待添加节点的后继节点
    Node<E> pred, succ;
    //如果指定的位置就是末尾，添加节点位置就是null因为没有后继节点，前一个节点就是最后一个节点
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        //否则后继节点就是succ节点，node方法返回指定索引处的节点！
        succ = node(index);
        pred = succ.prev;
    }

    //那么至此，链表相当于就从指定位置index一分为二，成为两个链表，现在开始插入元素
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        //如果前一个节点是null，那么说明这个链表是空的，这个节点就是第一个节点
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    //如果后继节点是null，说明添加到末尾了。
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}

//返回指定索引处的节点
Node<E> node(int index) {
    // assert isElementIndex(index);
	//它这里还优化了一下，判断index更接近链表头还是链表尾，从更近的那一边开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

## get方法

调用的还是之前说的node方法

```Java 
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

## remove方法

```Java
//遍历查找到对应删除的元素，然后调用unlink方法删除
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
//就是一些链表操作，简单！
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        //设置为null可以方便GC
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```