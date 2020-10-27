## ConcurrentHashMap

[ConcurrentHashMap 部分源码](#ConcurrentHashMap部分源码)

常见面试问题

1. HashMap是线程安全的吗？在多线程情况下要使用安全的Map有解决方案吗？有没有线程安全的并发容器？

HashMap多线程情况下是不安全的，会有数据丢失的可能，具体请看HashMap那一节，多线程情况下使用ConcurrentHashMap来解决并发问题。
这个问题其实很简单，往往只是用来引出后面的ConcurrentHashMap而问的。

2. ConcurrentHashMap与HashMap有什么区别？它为什么能解决并发时的安全问题？

两个问题其实都是一个，底层数据结构都是一样的数组加链表或者红黑树，代码逻辑上大部分都相同，只是在put方法上有些差异，参考源码中put方法，可以看到，当数组未初始化时，进行初始化的过程不一样了，初始化方法里面使用了cas 加锁；判断元素要插入在数组中的位置时，如果为null，则会使用cas将null替换为添加的元素，如果有元素，则使用synchronized锁定这个位置的元素，之后的逻辑，判断是否是同一个元素，是替换还是插入到链表或者红河树上，链表达到阈值转换等，基本上与HashMap一杨























# ConcurrentHashMap部分源码

> 源文件6000多行，全部放在文章里影响阅读，这里只把重点的几个方法拿过来，便于大家参考，JDK 1.8

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

