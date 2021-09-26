## HashMap与HashSet

### HashMap 面试重点
如果问到Java基础知识，百分之90的情况都会问到HashMap，可见它的重要性，面试过程中无外乎以下几点：

1. put 过程
2. 扩容过程
3. 加载因子为什么是0.75
4. 为什么容量是2的n次方


#### put 过程
源码：
```
    // put 方法入口
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    // hash 方法
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    // putVal 方法
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```
过程描述：
- 首先取key的hashCode 进行扰动
- 判断数组是否初始化，未初始化，则进行初始化过程；
- 计算hash值与数组长度减一[(n - 1) & hash)]的位置上是否有元素，没有元素，则直接将元素插入到该位置；
- 如果有元素，则比较该元素与要插入的元素的key的hash及key是否相等（(k = p.key) == key || (key != null && key.equals(k)))），如果相等，则说明是同一个元素，直接替换该元素
- 如果不相等，则判断当前元素是不是红黑树，如果是，则遍历树节点查找与要插入的元素的key是否有相等的节点，相等则替换，不相等则插入到红黑树上；
- 如果不是，则说明是链表，遍历链表查找与要插入的元素的key是否有相等的节点，相等则替换，不相等则插入到链表上，最后判断链表长度是否大于阈值(TREEIFY_THRESHOLD-1=7)，将链表转换成红黑树。
- 最后判断数组长度是否大于容量的75%，是则进行扩容

虽然我上面的几个过程，好几处说到相等则替换，实际上只是相等的时候记录该元素，最后再进行替换，实际发生替换的代码如下：
```
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }

```

#### 扩容过程
扩容方法:
```
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```


####  加载因子为什么是0.75

加载因子是决定数组是否需要扩容的主要因素，判断条件是：


```
HashMap 源代码中 putVal() 方法最后

if (++size > threshold)
     resize();

// size  是当前已使用的大小（即已添加的元素）
// threshold 加载因子与容量相乘的结果
// 默认的 加载因子 static final float DEFAULT_LOAD_FACTOR = 0.75f
// 即每次使用到容量的百分之75的时候进行扩容

```

为什么是0.75 而不是其他？
首先0-0.5以下肯定是不合适的，假设是0.5，即每次到容量的一半就扩容，这明显有一半的空间浪费掉了，所以0.5及以下都不合适
那么1呢，如果是1，即每次容量使用完才进行扩容，假设容量是64，那么添加到65个元素的时候进行扩容，但是刚好使得64个元素的hash值正好放在数组对应的位置上，概率是非常小的，那么必然有些位置上出现冲突，这是容量比较小的情况下，那么容量越大的情况下，冲突就越严重，也就是说1的时候，冲突的情况较高（个人理解，有理解比较好的可以兄弟可以说一下，共同进步），那么在0.5-1 之间 为什么取了0.75 而不是0.7或者0.8呢，目前来讲查阅了不少资料但都没有一个让人满意的说法，有的说就是取了个中间值，有的说是空间利用率比较高，有的说什么泊松分布每个碰撞位置的链表长度超过８个是几乎不可能的，感觉完全不是一回事。
这里我目前也没有好的答案，回答的时候一般把0.5以下和1的情况说一下基本上就这样了。

#### 为什么容量是2的n次方
1. hashcode 是一个整数，可以存很多值，要把所有的hash值都放在map数组里面是不现实的，那么必然要把hash值与数组长度取余之后存放即hash%length，而取余运算效率是比较低的，效率最高的就是位运算了，所以要把hash%length 转换为位运算 (length  - 1) & hash 即：hash%length = (length  - 1) & hash 若使得等式成立，则length必须为2的倍数（大家可以自己测试），所以容量是2的n次方
2. 可以使得添加的元素均匀分布在HashMap中的数组上，减少hash碰撞，避免形成链表的结构（查询效率降低了）

参考：
[HashMap初始容量为什么是2的n次幂及扩容为什么是2倍的形式](/https://blog.csdn.net/Apeopl/article/details/88935422)


#### 其他

##### HashMap是线程安全的吗？

不是，多线程情况下推荐使用ConcurrentHashMap

##### HashMap多线程情况下会有什么问题

1.8之前，rehash可能会造成循环链表，导致死循环问题，1.8之后已经修复，但是仍然会有其他问题：
多线程put的时候可能导致元素丢失，主要问题出在addEntry方法的new Entry<K,V>(hash, key, value, e)，如果两个线程都同时取得了e,则他们下一个元素都是e，然后赋值给数组元素的时候有一个成功有一个丢失。


### HashSet

HashSet 存放无序不重复的元素，为什么无序？为什么不重复？

查看HashSet源码这一切都能够很好理解了：

```

private transient HashMap<E,Object> map;

private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

```

在它的内部,实际上是维护了一个HashMap的变量作为它的存储结构

当我们添加元素的时候是把元素作为HashMap的key值，添加到Map中去

无序因为是字典结构，所以是无序的

不重复因为Map中不会有key值相等的元素。
