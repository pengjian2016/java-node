# ThreadLocal

### 1. ThreadLocal 是什么？它解决了什么问题？

多线程情况下，实现同一个线程内变量是共享的，不同线程之间是线程隔离的。这就是ThreadLocal的作用。

### 2. ThreadLocalMap 数据结构是什么样的？

当我们创建ThreadLocal变量，调用set方法设置变量时，实际上是往ThreadLocalMap插入数据的

```
    // set方法
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    // get方法
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    } 

```

ThreadlocalMap从那里来的呢？在thread中，每个线程都有变量threadLocals 它就是ThreadLocalMap的实例

```

ThreadLocal.ThreadLocalMap threadLocals = null;

```

在set方法中，当map未创建时，先创建map，并将threadlocal对象（this）作为key，value就是我们的变量，放入map中去。而get方法就是从map中取出value值。

那么这个ThreadLocalMap 是什么结构呢？

```
static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        private Entry[] table;

        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            // 计算要插入的位置
            int i = key.threadLocalHashCode & (len-1);
            // tab[i] 如果没有元素，即e=null的时候，for循环就结束了，否则会循环想下找，在寻找的同时，如果发现key相等则替换掉值，
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // 如果key相等，则替换
                if (k == key) {
                    e.value = value;
                    return;
                }
                // 如果查找的过程中，发现某个key已经被回收导致k为空，则替换这个槽位
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            // 找到空的位置，插入该元素
            tab[i] = new Entry(key, value);
            int sz = ++size;
            // 清理过期的key，如果没有未清理到任何数据，并且元素已达到容量的2/3 时，则会进行库容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
} 
```
结合源码，可以看到ThreadLocalMap 内部维护了一个数组 Entry[] table ，Entry 的key是ThreadLocal 的弱引用。

### 3. ThreadLocalMap hashcode 如何生成的？hash冲突时怎么解决的？key是弱引用的要注意什么问题？

在set方法中有这样一段代码：
```
int i = key.threadLocalHashCode & (len-1);
```
这是取key的hashcode与数组长度减一，计算这个key在数组中存放的位置。key就是ThreadLocal 的对象，而threadLocalHashCode是它的一个属性

```
private final int threadLocalHashCode = nextHashCode();
 private static AtomicInteger nextHashCode =
        new AtomicInteger();

 private static final int HASH_INCREMENT = 0x61c88647;
 private static int nextHashCode() {
     return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
threadLocalHashCode 变量被final修饰，在我们new出来ThreadLocal对象时，这个值就会初始化好（每次加0x61c88647），不会再变动，0x61c88647它是斐波那契数 也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash 分布非常均匀。

虽然如此，如果ThreadLocal对象过多的情况下，仍然会有hash冲突，在HashMap有拉链法，但是这里没有这么做，参考上面的源码set方法中，当前槽位为空，则会直接插入到该位置，否则，向下查找，找到为空的位置插入，当然在查找的过程中会判断是否是同一个key，进行替换等。

注意：因为map中存放的时key的弱引用，在内存不足时会被回收，而value时强引用，这就导致，key对应的value可能没有回收，如果大量的value存活在内存中，就会导致内存不足，这就是所谓的内存泄露。需要在线程结束前调用remove方法。

参考:[万字详解ThreadLocal关键字](https://snailclimb.gitee.io/javaguide/#/docs/java/multi-thread/%E4%B8%87%E5%AD%97%E8%AF%A6%E8%A7%A3ThreadLocal%E5%85%B3%E9%94%AE%E5%AD%97)

