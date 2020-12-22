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
下面将结合ReentrantLock，ReentrantReadWriteLock等来详细的介绍AQS如果实现独占锁、共享锁、公平和非公平锁，等待队列如何工作的，释放锁后是怎么样的流程，条件队列等

#### 2. ReentrantLock 如何实现公平锁和非公平锁的？获取锁的过程？

2.1 ReentrantLock 公平锁

ReentrantLock  依赖于AQS，其核心就是内部类对AbstractQueuedSynchronizer的实现

```
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
       
        abstract void lock();

        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

    }

    /**
     * 非公平锁实现
     */
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
    /**
     * 公平锁实现
     */
    static final class FairSync extends Sync {
     
        final void lock() {
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    public void lock() {
        sync.lock();
    }
    public void unlock() {
        sync.release(1);
    }
}

```
1). 源码中可以看到，构造方法中默认使用非公平锁（NonfairSync），当然也可以指定为公平锁（FairSync），先看一下公平锁的加锁过程，当调用 ReentrantLock 中的 lock() 方法时，实际上会调用sync.lock()，查看FairSync中lock的实现就是非常简单的调用 acquire(1) 方法，这个方法是AQS中的一个方法，具体如下：

```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
2). 首先是调用tryAcquire方法，这个方法在FairSync内部有实现，看上面的源码，在tryAcquire方法中，首先获取当前线程，然后判断state是否是0（0表示无锁状态），如果是0即，没有加锁的情况下，在if判断中先调用hasQueuedPredecessors()方法，该方法也在AQS中，具体如下：

```
    public final boolean hasQueuedPredecessors() {
        /*
        双向链表中，第一个节点为虚节点，其实并不存储任何信息，只是占位。真正的第一个有数据的节点，是在第二个节点开始的。
        当h != t时： 如果(s = h.next) == null，等待队列正在有线程进行初始化，但只是进行到了Tail指向Head，没有将Head指向Tail，此时队列中有元素，需要返回True。
        如果(s = h.next) != null，说明此时队列中至少有一个有效节点。
        如果此时s.thread == Thread.currentThread()，说明等待队列的第一个有效节点中的线程与当前线程相同，那么当前线程是可以获取资源的；
        如果s.thread != Thread.currentThread()，说明等待队列的第一个有效节点线程与当前线程不同，当前线程必须加入进等待队列。
        */
        Node t = tail; 
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
3). 如果队列中没有等待的线程，则会调用compareAndSetState 设置锁状态，设置成功后调用setExclusiveOwnerThread方法，即设置当前线程为拥有锁的线程，当state不为0的时候，则判断拥有锁的线程是否是当前线程，如果是，则把state加1，这也是可重入锁的实现逻辑。再回到acquire方法中，当tryAcquire方法返回失败时（成功时则流程结束），即未获取到锁时会调用acquireQueued(addWaiter(Node.EXCLUSIVE), arg) ，这里面是两个方法，先调用addWaiter 方法将当前线程加入到队列尾部并返回包含当前线程的Node节点，acquireQueued会把放入队列中的线程不断去获取锁，直到获取成功或者不再需要获取（中断）。

以上是ReentrantLock公平锁的加锁过程，过程虽然不复杂，但是也需要一定的思考，特别是加锁失败时的逻辑，如果进入队列，如何在队列中获取锁的，其核心就在acquireQueued方法中，这个会在后面详细介绍，这里先介绍整体流程，方便理解。总结公平锁加锁流程如下：

![公平锁](https://images.gitee.com/uploads/images/2020/1218/143045_ca6770a7_8076629.png "屏幕截图.png")


2.2 非公平锁 NonfairSync 

源码中可以看到，当调用lock方法时，它会先进行一次抢锁修改state状态，成功则直接获取锁并把当前线程设置为有效线程即拥有锁的线程；如果失败，则会调用acquire(1)方法，这个方法都是AQS中的实现，上面已经介绍过，它会先调用tryAcquire方法，实际上会调用nonfairTryAcquire方法，里面的逻辑大部分与公平锁的实现一致，只是不会判断队列是否为空。

总的来讲，它与公平锁的区别就是，先进行一次抢锁，成功则获取锁，不成功则继续判断state，当state=0时，它不会判断当前队列是否为空，这就是两个主要的区别。

![非公平锁](https://images.gitee.com/uploads/images/2020/1218/144302_4cb133f1_8076629.png "屏幕截图.png")


以上就是ReentrantLock的核心内容，它是独占锁，其内部实现了公平锁和非公平锁两种机制，它实现了Lock接口，所以当面试官问起java中lock与synchronized区别时，主要就是指ReentrantLock，那么它们有哪些区别呢？







AbstractQueuedSynchronizer（AQS）、ReentrantLock、ReentrantReadWriteLock、compareAndSwap(CAS)