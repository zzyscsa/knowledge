# TreeMap

> 继承AbstractMap抽象类，实现NavigableMap接口；NavigableMap继承SortedMap接口，SortedMap继承Map接口；

SortedMap这个接口决定了HashMap的key是无序的，TreeMap的key是有序的

**<font color='green'>TreeMap底层使用红黑树实现，没有数组，就是一个红黑树</font>**



## 结点数据结构

```Java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    //默认是黑色结点
    boolean color = BLACK;
```

key不能为null，和HashTable一样

## 核心属性

- Comparator自定义比较器，默认是null

```Java
private final Comparator<? super K> comparator;
```

- 唯一的根节点

```Java
private transient Entry<K,V> root;
```

- Entry的数量

```Java
private transient int size = 0;
```

- 结构调整的次数

```Java
private transient int modCount = 0;
```

## 构造方法

- 无参构造，自然顺序排序，所谓自然顺序就是采用key类型的compareTo方法比较

```Java
public TreeMap() {
    comparator = null;
}
```

- 传递一个比较器，自定义外部比较器排序

```Java
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
```

- 传递一个Map构造，按照默认规则排序

```Java
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}
```

- 传递一个Map，按照传递的map规则排序

```Java
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```

## 插入操作

### put方法

```Java
public V put(K key, V value) {
    //t是根节点
    Entry<K,V> t = root;
    //如果根节点是null，说明还没有建立起红黑树
    if (t == null) {
        compare(key, key); // type (and possibly null) check
		//那么创建一个根节点
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    //这个变量存储对象比较的结果！
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    //cpr表示自定义的比较器
    Comparator<? super K> cpr = comparator;
    //如果有自定义的比较器
    if (cpr != null) {
        //二分搜索红黑树查找插入的位置
        do {
            //当前节点赋值给parent
            parent = t;
            //插入的结点和当前结点比较
            cmp = cpr.compare(key, t.key);
            //如果插入的结点更小，那么往左子树继续搜索
            if (cmp < 0)
                t = t.left;
            //如果插入的结点更大，那么往右子树继续搜索
            else if (cmp > 0)
                t = t.right;
            //否则两个结点相等，那么就替换value的值
            else
                return t.setValue(value);
        } while (t != null);
    }
    //如果没有自定义比较器，那就是自然排序，用类自己的compareTo方法比较
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    //查找完成，没有相同的key，那么就创建新的结点Entry，并且插入位置的父节点就是parent
    Entry<K,V> e = new Entry<>(key, value, parent);
    //如果比父节点小，插到左边
    if (cmp < 0)
        parent.left = e;
    //否则插到右边
    else
        parent.right = e;
    //自平衡操作！
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

### fixAfterInsertion自平衡

```Java
private void fixAfterInsertion(Entry<K,V> x) {
    //插入结点都是红色
    x.color = RED;
	//如果父节点是黑色，那么不需要调整，只有父节点是红色才进入这个循环调整
    while (x != null && x != root && x.parent.color == RED) {
        //1.父节点是爷爷结点的左孩子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            //y表示叔叔结点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            //1.1如果叔叔结点为红色
            if (colorOf(y) == RED) {
                //父节点和叔叔结点设置为黑色
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                //爷爷结点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                //爷爷结点设置为新的插入结点！
                x = parentOf(parentOf(x));
            } else { //1.2 叔叔结点为黑色或者为null
                //1.2.1插入的结点时父节点的右孩子
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                //1.2.2插入结点是父节点的左孩子（LL双红）
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else { //2.父节点是爷爷结点的右孩子
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```

## get操作

```Java
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}
```

```Java
//就是一个红黑树二分查找的过程
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

## 删除操作

### remove方法

```Java
public V remove(Object key) {
    Entry<K,V> p = getEntry(key); //查找到需要删除的结点
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p); //调用删除结点的方法
    return oldValue; //返回被删除的结点
}
```

### deleteEntry方法

```Java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //如果删除的这个结点既有左孩子又有右孩子，那么用后继结点替换，然后删除后继结点
    if (p.left != null && p.right != null) {
        //s表示后继结点
        Entry<K,V> s = successor(p); 
        //替换k-v值
        p.key = s.key;
        p.value = s.value;
        //被删除的结点更新为这个后继结点！这个后继结点肯定没有左孩子！
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    //replacement表示替代结点，到这里被删除的结点可能是叶子结点，可能是只有左节点或者只有右节点
    //如果是叶子节点就直接删除，如果是只有左节点或者只有右节点那么就把左节点或者右节点拿来替换它，然后删除它的左节点或者右节点
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);
	//有左节点或者右节点
    if (replacement != null) {
        // Link replacement to parent
        replacement.parent = p.parent;
        //替换节点替换被删除的结点位置
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        //删除这个被删除结点的左右分支以及父节点引用，方便GC回收
        p.left = p.right = p.parent = null;

        // Fix replacement
        //如果替换节点是红色，那么不需要平衡处理，否则需要自平衡
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    }
    //是根节点，那么就是唯一的结点
    else if (p.parent == null) { // return if we are the only node.
        root = null;
    }
    //替换结点可以直接删除！
    else { //  No children. Use self as phantom replacement and unlink.
        //同理，删除的是黑色需要自平衡
        if (p.color == BLACK)
            fixAfterDeletion(p);
        //删除这个被删除结点的左右分支以及父节点引用，方便GC回收
        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

### fixAfterDeletion自平衡

```Java
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        if (x == leftOf(parentOf(x))) {
            Entry<K,V> sib = rightOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }

            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```