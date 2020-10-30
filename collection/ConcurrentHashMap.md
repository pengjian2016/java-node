## ConcurrentHashMap

[ConcurrentHashMap 部分源码](#ConcurrentHashMap部分源码)

常见面试问题

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
- 2.2 putVal方法中 首先判断key或者value是否为null，否则抛出异常，这里也是跟HashMap不同的地方，接下来才是取key的hashcode，然后进行一次扰动，扰动的方法也比HashMap多了``` & HASH_BITS``` 操作，然后是循环判断数组是否为空，否则进行初始化过程。
```
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}

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

### 3. ConcurrentHashMap 如何计算元素个数，即size()方法：

```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

```





















# ConcurrentHashMap部分源码

> 源码6000多行，全部放在文章里影响阅读，这里只把重点的几个方法拿过来，便于大家参考（JDK 1.8）

```
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
     
    //获取元素的方法
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }    

    // 添加元素的方法    
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();//1.初始化方法与HashMap有区别 
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //2.设置null的时候与HashMap有区别 
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);//正在扩容的情况，帮助库容
            else {
                V oldVal = null;
                synchronized (f) {//3.不为空的时候与HashMap有区别 
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
        }
        addCount(1L, binCount);
        return null;
    }       
    //扩容方法
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }           
       
}


```

