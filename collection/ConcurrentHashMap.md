## ConcurrentHashMap
 
### 1. HashMap是线程安全的吗？在多线程情况下要使用安全的Map有解决方案吗？有没有线程安全的并发容器？

HashMap多线程情况下是不安全的，会有数据丢失的可能，具体请看HashMap那一节，多线程情况下使用ConcurrentHashMap来解决并发问题。
这个问题其实很简单，往往只是用来引出后面的ConcurrentHashMap而问的。

### 2. ConcurrentHashMap与HashMap有什么区别？它为什么能解决并发时的安全问题？

ConcurrentHashMap最主要的区别在put过程中，其他包括底层数据结构（数组+链表/红黑树），get方法基本上与HashMap类似。下面主要分析put方法展现两者的不同：
- 2.1 调用put方法这里与HashMap不同的是不会先进行hashcode扰动。
```
public V put(K key, V value) {
   return putVal(key, value, false);
}
```
- 2.2 putVal方法中 首先判断key或者value是否为null，否则抛出异常，这里也是跟HashMap不同的地方，接下来才是取key的hashcode，然后进行扰动，扰动的方法也比HashMap多了``` & HASH_BITS``` 操作，然后是循环判断数组是否为空，否则进行初始化过程。
```
//hash扰动方法
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
//putval方法 2.2
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();

```
- 2.3 (n - 1) & hash 计算元素的位置，如果没有元素，则使用cas替换null为新元素，并发的情况，可能替换失败，因为外侧有for循环，会在下一次走相应的逻辑。

```
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
         if (casTabAt(tab, i, null,
            new Node<K,V>(hash, key, value, null)))
            break;   // no lock when adding to empty bin
    }
```
- 2.4 如果当前位置不为空，判断是否在进行库容，如果是，则帮助库容（多线程库容）
```
    else if ((fh = f.hash) == MOVED)
           tab = helpTransfer(tab, f);
```
- 2.5 否则的话，锁定当前节点，接下来的一些列操作基本上与HashMap一样，是插入到链表还是红黑树上，链表达到阈值进行变换等。
 
```
        else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
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
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
```

### 3. ConcurrentHashMap 如何计算元素个数？
源码中可以看到有size()和mappingCount()两种方法，不过差别不大（代码注释中更推荐mappig方法，元素个数有可能超过integer最大值），最后都是调用sumCount()方法，其中最主要的就是通过baseCont 和 CounterCell数组计算总数。

```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}

/**
 * Returns the number of mappings. This method should be used
 * instead of {@link #size} because a ConcurrentHashMap may
 * contain more mappings than can be represented as an int. The
 * value returned is an estimate; the actual count may differ if
 * there are concurrent insertions or removals.
 *
 * @return the number of mappings
 * @since 1.8
 */
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}

```
### 4. ConcurrentHashMap 中为何key和value 都不能为null，而HashMap却没有限制？
源码中可以看到，当key或者value为null时会抛异常，当key为null的时候不能辨别是key不存在还是本身就为null，value也同样如此，即并发安全的容器中不允许null值的出现的主要原因是他可能会在并发的情况下带来难以容忍的二义性。而HashMap可以通过contains(key)来判断是否包含某个key。HashMap与ConcurrentHashMap两个原作者对于null值的讨论，ConcurrentHashMap认为允许null是错误的设计，HashMap的作者也发现null是一个非常令人头疼的问题。具体参考下面的文章，讲解的最详细：

[ConcurrentHashMap key value都不允许为null](https://segmentfault.com/a/1190000021105716)

## ConcurrentSkipListMap

#### ConcurrentSkipListMap 数据结构

说到跳跃表，大家可能都有点印象，它是个具有多级的有序链表结构，存取时间都是o(log(n))，支持并发操作。

![输入图片说明](https://images.gitee.com/uploads/images/2020/1103/093659_4a4b2dc3_8076629.png "屏幕截图.png")

#### 相比与ConcurrentHashMap 它有什么优势或者特点

前面的问题我们知道它是个有序的链表，需要有序的操作时使用它效率更好，另一方面它能支持更高的并发，在一定数据量以下的情况它可能不如ConcurrentHashMap，但是随着数据越来越多并发情况越高的情况下，它的优势就比较明显了。

参考：
[Java多线程（四）之ConcurrentSkipListMap深入分析](https://blog.csdn.net/iteye_1168/article/details/82536800)


## CopyOnWriteArrayList

它是多线程操作下的ArrayList，源码中可以看到，其内部仍然是一个数组结构，get方法没有任何的锁，而add方法和remove方法都是通过ReentrantLock 加锁，而这两个方法都是在添加元素或者删除元素时copy一个新的数组，然后再set回去。

它适用的场景主要就是读多写少的情况，因为写的时候会复制新的数组，如果写的情况较多，频繁的创建新数组，且随着数据越来越多，复制的效率也就越低。

另外在读的时候有一定的延迟，因为读的可能是旧的数组。

#### 1. 为什么引入一个新的数组操作，而不是在原来的数组上直接操作？
 
1.1

> volatile 关键字修饰的是数组，如果我们简单的在原来数组上修改其中某几个元素的值，是无法触发可见性的，我们必须通过修改数组的内存地址才行，也就说要对数组进行重新赋值才行。
在新的数组上进行拷贝，对老数组没有任何影响，只有新数组完全拷贝完成之后，外部才能访问到，降低了在赋值过程中，老数组数据变动的影响。比如经典的 ConcurrentModificationException 异常问题。参考：[Java 经典面试题：聊一聊 JUC 下的 CopyOnWriteArrayList](https://www.cnblogs.com/jamaler/p/12843126.html)

1.2

> CopyOnWriteArrayList 的写操作进行了加锁。如果 CopyOnWriteArrayList 只有写操作，那么这里确实只通过加锁就可以保证安全，不需要进行复制。但是 CopyOnWriteArrayList 还有读操作，而且大多数情况下，List 都是读多写少的。所以这里本质上也依然是并发读写的问题：
若没有复制，写时加锁，读时不加锁，那么就会发生并发读写问题，产生不可预期的异常，即 ConcurrentModificationException；
若没有复制，写时加锁，读时也需要加锁，这样就相当于退化为 SynchronizedList，读性能大大减弱。
而写时复制，则可以很好的处理并发读写问题，而且还保障了性能：
写时加锁，不会产生并发写的问题，保证了写操作的安全性；
实际的写操作，是在复制的新数组上进行；而同一时刻的读操作，是在原数组进行的，所以这里的读操作不会产生并发读写问题，也不需要加锁；
新数组操作完成后，将原数组替换，这里则是通过 volatile 关键字保障了新数组的线程可见性。
这样，引入写时复制的原因就说清楚了。实际上，这是 volatile、锁、写时复制三者共同作用的结果，既保证了并发读写的安全性，也保证了读的性能，三者缺一不可，可谓精妙。
参考:[关于 CopyOnWriteArrayList 中的写时复制](https://patchouli-know.com/2020/05/16/copy-on-write-array-list/)


部分源码:

```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8673264195747942595L;

    //锁
    final transient ReentrantLock lock = new ReentrantLock();

    //数据存储
    private transient volatile Object[] array;
    
    final Object[] getArray() {
        return array;
    }

     
    final void setArray(Object[] a) {
        array = a;
    }

    private E get(Object[] a, int index) {
        return (E) a[index];
    }

    //get 方法
    public E get(int index) {
        return get(getArray(), index);
    }
    
    //add 方法
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }

    //remove 方法
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
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

