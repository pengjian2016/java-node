###TreeMap

TreeMap 部分源码（jdk 1.8）

```
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
{
    // 比较器
    private final Comparator<? super K> comparator;
    // 根节点
    private transient Entry<K,V> root;

    public TreeMap() {
        comparator = null;
    }
    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }

    // put方法
    public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
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
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }

    private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED;

        while (x != null && x != root && x.parent.color == RED) {
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {
                    if (x == rightOf(parentOf(x))) {
                        x = parentOf(x);
                        rotateLeft(x);
                    }
                    setColor(parentOf(x), BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    rotateRight(parentOf(parentOf(x)));
                }
            } else {
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

    // get方法
    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }
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

}

```

##### 1. TreeMap 数据结构？

从源码中，可以看到，其内部只有一个root变量，维护的是一个红黑树，在fixAfterInsertion() 方法中可以看到，插入节点后会进行一系列红黑树的变换。
时间复杂度o(log(n))

##### 2. TreeMap 中  key为何要实现comparator接口？如何保证有序性？

put方法中，当我们初始化TreeMap时未指定比较器，则使用key的比较器进行比较，所以key并非一定要实现比较器，只是TreeMap的默认构造函数中不指定比较器的时候，是以key的比较器进行比较确认元素要插入的位置。

顺序的保证也是通过比较器比较key值来保证的。

##### 3. TreeMap 如何实现一致性hash算法？ 

待完善（写过一版，但是了解的不够深入，没体现TreeMap的具体作用）

参考[关于TreeMap和一致性hash](https://zhuanlan.zhihu.com/p/20270435)

#### TreeSet

TreeSet 部分源码

```
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    
    private transient NavigableMap<E,Object> m;

    private static final Object PRESENT = new Object();
    
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }

}
```

##### 1. TreeSet 数据结构
构造方法中使用TreeMap 初始化了内部变量，所以实际上还是 TreeMap 的红黑树数据结构。TreeSet的作用是保存无重复的数据，不过还对这些数据进行了排序。

##### 2. 不管时HashSet还是TreeSet 他们都是以一个空对象（PRESENT）作为value值，为何这样做？为何不直接使用null还能节省空间？

HashSet内部使用的是HashMap，而TreeSet内部使用的是TreeMap，这两个Map，当第一次插入元素时，返回的是null，而修改元素时返回的是key对应的value值，假如set中使用null作为value值，那么第一次调用add方法后返回的是true（return m.put(e, null)==null），而当再修改的时候，由于value值存放的是null，那么add方法返回的也是true，与它要表现的与行为是不一致的，即插入重复的元素返回失败，也就是说再次插入重复的元素时，无法判断是不是已经重复了，还是说本来就没有。


