### ArrayList 

##### 1. 默认容量是多少？每次扩容是原来多少倍？扩容过程？

ArrayList 默认容量是10，之后每次扩容的时候是原来的1.5倍。

扩容过程描述：首次创建ArrayList时，未指定容量时，数组其实是一个空对象EMPTY_ELEMENTDATA，并未进行初始化，当第一次添加元素调用add时，会调用  ensureCapacityInternal(size + 1) 方法，去初始化或者执行扩容过程。首先调用 calculateCapacity方法，判断数组是否是空数组，如果是，则返回默认容量大小，否则返回size+1
，之后调用ensureExplicitCapacity方法，判断最小容量是否大于数组长度，如果是，则扩容，新的容量为原来的1.5倍，并调用Arrays.copyOf()方法，将旧的数组复制一份为新的容量。

部分源代码如下：

```
    // 1.添加元素时
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    // 2.扩容或初始化
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    // 3.计算最小需要的容量
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }    
    // 4.需要的容量大于数组长度时，调用grow进行扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    // 5.新的容量为旧的容量+旧的容量的0.5倍，即新的容量为原来的1.5倍
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

```

##### 2. 都说 ArrayList 插入或删除比较慢，是一定的吗？
并不一定

针对插入方法，如果调用的是下面的这种方法，即末尾插入时，我们知道，当数组的容量是够的时候，这个添加过程并不慢，而如果数组的容量不够时，就需要扩容了，扩容过程就是复制新的数组，这个自然就很慢了。

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

```
而如果调用的是插入到某个指定的位置时，则它就必然会很慢了，因为它会复制新的数组：
```
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
对于删除时，如果是末尾删除时，其实是把数组中最后一个元素设置为null，其他情况就需要复制数组了，总之删除大多数情况都是较慢的
```
  public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
### LinkedList 

##### 1. LinkedList 是什么结构的？它需要扩容吗？描述一下插入过程？

LinkedList 是一个双向链表结构，内部有三个属性，其中first是链表头指针，last是尾指针，而Node对象内部保存了当前item和下一个元素next以及上一个元素prev。它可以随意添加，不受容量限制，理论上内存足够大，它就能无限添加。当调用add(e)方法时，它其实是向链表尾部添加元素，添加的过程也很简单，即，创建一个新的Node节点，prev是last节点，然后将last.next节点指向这个新节点，部分源码如下：
```
transient int size = 0;

/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;
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

public boolean add(E e) {
    linkLast(e);
    return true;
}
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


### ArrayList 与 LinkedList 区别

1. 结构: ArrayList 内部是一个数组，LinkedList是一个双向链表

2. ArrayList插入和删除大多数情况下是比较慢的，LinkedList插入和删除比较快

3. ArrayList支持随机访问，查询较快，适合读多写少的场景，LinkedList不支持随机访问、查询较慢，适合写多读少的情况。

4. 相同数量级的情况下，不考虑数组的剩余容量情况下，ArrayList占用内存要比LinkedList少很多，因为LinkedList中每个节点多存储了next和prev节点。


