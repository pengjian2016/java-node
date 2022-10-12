### ArrayList 

1. 默认容量是多少？每次扩容是原来多少倍？扩容过程？

答：ArrayList 默认容量是10，之后每次扩容的时候是原来的1.5倍。

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



### ArrayList 与 LinkedList 区别

1. 结构: ArrayList 内部是一个数组，LinkedList是一个双向链表

2. 
