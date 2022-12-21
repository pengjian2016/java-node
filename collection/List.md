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
总之：ArrayList插入和删除受位置的影响，同时插入的时候也受剩余容量的影响。

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

##### 2. LinkedList插入和删除一定是快的吗？

并不一定，当在头或者尾插入和删除时，它的时间复杂度是O(1)，但是如果指定位置插入或删除时，则要先遍历到对应的位置，它的时间复杂度就变成了O(n)

```
    // 指定位置插入
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
    // 指定位置删除
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
```

总之：LinkedList 插入和删除也受位置的影响


### ArrayList 与 LinkedList 区别

1. 结构: ArrayList 内部是一个数组，LinkedList是一个双向链表

2. ArrayList插入和删除大多数情况下是比较慢的，LinkedList插入和删除比较快

3. ArrayList支持随机访问，查询较快，适合读多写少的场景，LinkedList不支持随机访问、查询较慢，适合写多读少的情况。

4. 相同数量级的情况下，不考虑数组的剩余容量情况下，ArrayList占用内存要比LinkedList少很多，因为LinkedList中每个节点多存储了next和prev节点。

5. 项目中很少使用LinkedList，使用LinkedList的场景基本都可以被ArrayList代替，毕竟连LinkedList的作者都说他从来不会用LinkedList。


### ArrayDeque

ArrayDeque 从JDK 1.6开始有的，它是以数组方式实现的双端队列，双端队列即两端都可以插入和弹出元素的队列，它是非线程安全的。

其内部的数据结构是：数组，加上head、tail两个指针。默认容量是16，每次扩容时为原来的2被，数组长度始终保持2的幂次方。

```
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{
    transient Object[] elements; // non-private to simplify nested class access

    transient int head;

    transient int tail;
    
    public ArrayDeque() {
        elements = new Object[16];
    }
}
```

ArrayDeque作为队列使用时的主要方法：

```
    // 添加元素到队列头
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;
        if (head == tail)
            doubleCapacity();
    }
    // 添加元素到队列尾
    public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e;
        if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            doubleCapacity();
    }
    
    // 从队列头出队
    public E pollFirst() {
        int h = head;
        @SuppressWarnings("unchecked")
        E result = (E) elements[h];
        // Element is null if deque empty
        if (result == null)
            return null;
        elements[h] = null;     // Must null out slot
        head = (h + 1) & (elements.length - 1);
        return result;
    }

    // 从队列尾出队
    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
        E result = (E) elements[t];
        if (result == null)
            return null;
        elements[t] = null;
        tail = t;
        return result;
    }
    // 扩容
    private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    }

```

入队和出队的方法有很多，这里主要分析，addFirst、addLast、pollFirst、pollLast这几个方法

队列头入队过程描述(addFirst): 将head指针减1并与数组长度减1取模（防止溢出），在head位置插入元素，判断头和尾相等时进行扩容，新的容量扩为原来的两倍。

队列尾入队过程描述(addLast): 将尾指针的位置插入元素，将尾指针加1，如果与头相等，则扩容。

队列头出队过程描述(pollFirst)：取出head位置元素，将数组中该位置的元素设置为null，移动head指针+1。

队列尾出队过程描述(pollLast)：计算尾指针-1的元素位置，取出该元素，将数组中该位置的元素设置为null，将尾指针-1。

扩容过程：

新的容量为原来的两倍，创建新的数组，用System.arraycopy复制两次到新的数组中去，第一次将head右边的元素复制到数组中去，第二次将head左边的元素复制到数组中去。


ArrayDeque作为栈(后入先出)使用时的主要方法：

```
    public void push(E e) {
        addFirst(e);
    }

    public E pop() {
        return removeFirst();
    }

    public E removeFirst() {
        E x = pollFirst();
        if (x == null)
            throw new NoSuchElementException();
        return x;
    }

```
可以看到push时主要用的是addFirst方法，而弹出时用的是removeFirst方法，而romveFirst方法内部用的是pollFirst方法，这里就不再过度介绍。




参考

https://zhuanlan.zhihu.com/p/64298684
