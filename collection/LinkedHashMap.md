### LinkedHashMap 和 LinkedHashSet

#### LinkedHashMap 部分源码

```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>{
        
    transient LinkedHashMap.Entry<K,V> head;

    transient LinkedHashMap.Entry<K,V> tail; 
    
    //新增元素和修改的方法
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
 
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

    final boolean accessOrder;

    // link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
    
    //其中的一个构造函数
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

    
    //访问元素的方法
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
        
}


```

#### 常见面试问题

##### 1. LinkedHashMap 的数据结构是什么？

源码中可以看到它是继承自HashMap，所以内部数据结构仍然是数组加链表或者红黑树，只不过是它的每个元素又同时在一个它自己维护的一个双向链表上，参考源码中它内部的两个变量head和tail，而newNode()和newTreeNode()两个方法是重写了HashMap中的Node创建方法，会在元素创建之后调用linkNodeLast()方法将元素加入到链表中。

##### 2. LinkedHashMap 的两个顺序是什么？

源码中有个构造函数（其他构造函数默认accessorder为false），其中一个 accessOrder 参数，这个是控制LinkedHashMap两个顺序的变量。两种顺序分别是，插入顺序和访问顺序。

插入顺序：默认情况不指定accessOrder或者为false时，是插入顺序，当我们变量输出元素时，实际上是使用它内部的链表从head向下遍历。

访问顺序：当我们设置accessOrder为true时，则为访问顺序，首先插入仍然是有顺序的，其次，当我们get方法访问元素时，它会将访问过的通过afterNodeAccess()方法移动到链表的末尾，如果是修改元素的时候，我们知道在HashMap的putVal访问中，当查到key值相同的元素时也会调用afterNodeAccess()方法：

HashMap中的putVal方法源码：

```
if (e != null) { // existing mapping for key
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
        e.value = value;
    afterNodeAccess(e);
    return oldValue;
}
```
所以插入或修改或访问都将把元素添加到链表的末尾，说到这里，大家肯定想到它能做什么事情了，就是下面的第3个问题

##### 3.如何使用LinkedHashMap实现LRU，即淘汰最近最少使用的元素？

前面我们知道，LinkedHashMap有两种顺序，那么实现LRU，只需要开启访问顺序，同时实现removeEldestEntry方法即可

参考代码

```
import java.util.LinkedHashMap;
import java.util.Map;

public class LRULinkedHashMap<K, V> extends LinkedHashMap<K, V> {
    private int maxSize;

    public LRULinkedHashMap(int maxSize) {
        super(16, 0.75f, true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > this.maxSize;
    }
}

```

测试代码

```
public static void main(String[] args) {
    LRULinkedHashMap<String, String> map =
            new LRULinkedHashMap<>(2);
    map.put("first","1");
    map.put("second","2");
    System.out.println("=======第1次遍历 start ");
    map.forEach((k,v)->System.out.println(k+"="+v));
    System.out.println("=======第1次遍历 end ");
    System.out.println();

    //容量不足时
    map.put("third","3");
    System.out.println("=======第2次遍历 start ");
    map.forEach((k,v)->System.out.println(k+"="+v));
    System.out.println("=======第2次遍历 end ");
    System.out.println();

    //访问first之后
    map.put("first","1");
    map.put("second","2");
    map.get("first");
    map.put("third","3");
    System.out.println("=======第3次遍历 start ");
    map.forEach((k,v)->System.out.println(k+"="+v));
    System.out.println("=======第3次遍历 end ");
}
```

输出结果：

```
=======第1次遍历 start 
first=1
second=2
=======第1次遍历 end 

=======第2次遍历 start 
second=2
third=3
=======第2次遍历 end 

=======第3次遍历 start 
first=1
third=3
=======第3次遍历 end 
```


#### LinkedHashSet 继承自 HashSet
