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
源码中可以看到有size()和mappingCount()两种方法，不过差别不大（代码注释中更推荐mappig方法，应为元素个数有可能超过integer最大值），最后都是调用sumCount()方法，其中最主要的就是通过baseCont 和 CounterCell数组计算总数。

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
上面的源码中可以看到，当key或者value为null时会抛异常。当key为null的时候不能辨别是key不存在还是本身就为null，value也同样如此。HashMap是非并发的，可以通过contains(key)来判断。其实，正如


## ConcurrentSkipListMap

#### ConcurrentSkipListMap 实现原理
说到跳跃表，大家可能都有点印象，它是个具有多级的链表结构，















 

