## AbstractQueuedSynchronizer（AQS）

#### 1. 什么是AQS？原理是什么？

AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的框架。近年来AQS这个框架被面试的几率越来越高，正如它的名称中包含的一样它是个抽象类，那些基于它的实现正是突出它重要的原因，这些具体实现类包括ReentrantLock、ReentrantReadWriteLock、Semaphore、CountDownLatch等，当然，它们不是直接继承AbstractQueuedSynchronizer，而是在内部定义了NonfairSync，FairSync这样的类去实现它。同时它也是模板方法设计模式的经典应用。

那么AQS的原理时什么呢？

> AQS核心思想是，如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。CLH：Craig、Landin and Hagersten队列，是单向链表，AQS中的队列是CLH变体的虚拟双向队列（FIFO），AQS是通过将每条请求共享资源的线程封装成一个节点来实现锁的分配。[美团技术团队-从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)


下面对AbstractQueuedSynchronizer的部分源码进行简单的说明：

1.1 四个核心的模板方法，tryAcquire，tryRelease是独占锁的加锁与释放方法；tryAcquireShared，tryReleaseShared是共享锁的加锁与释放方法，具体的实现由实现类来完成

```
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

```
1.2 加锁过程或释放锁过程，主要通过state字段来完成，比如独占锁情况下，通过getState()方法判断state是否是0，0表示无锁状态，然后通过compareAndSetState()方法尝试加锁等

```
    private volatile int state;

    protected final int getState() {
        return state;
    }

    protected final void setState(int newState) {
        state = newState;
    }

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

```
1.3 CLH 队列中的数据结构，其中：waitStatus表示当前节点在队列中的等待状态，0-节点初始化时的默认状态，1-CANCELLED获取资源等待超时或线程被中断时，该线程将从CLH队列中取消等待，-1-SIGNAL 表示当前节点已经准备好了（已被唤醒），等它的前驱节点释放锁后，该线程就可以执行了，-2-CONDITION 该状态与Condition条件有关

```
     static final class Node {
        
        volatile int waitStatus;

        volatile Node prev;

        volatile Node next;

        volatile Thread thread;

        Node nextWaiter;
    }


```


#### 2. 


AbstractQueuedSynchronizer（AQS）、ReentrantLock、ReentrantReadWriteLock、compareAndSwap(CAS)