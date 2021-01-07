## AbstractQueuedSynchronizer（AQS）

### 1. 什么是AQS？原理是什么？

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

### 2. ReentrantLock 如何实现公平锁和非公平锁的？获取锁的过程？

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
2). 首先是调用tryAcquire方法，这个方法在FairSync内部有实现，

```
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
```

在tryAcquire方法中，首先获取当前线程，然后判断state是否是0（0表示无锁状态），如果是0即，没有加锁的情况下，在if判断中先调用hasQueuedPredecessors()方法，该方法也在AQS中：

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

```
    final void lock() {
     if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
     else
        acquire(1);
    }

```

源码中可以看到，当调用lock方法时，它会先进行一次抢锁修改state状态，成功则直接获取锁并把当前线程设置为有效线程即拥有锁的线程；如果失败，则会调用acquire(1)方法，这个方法上面已经介绍过，它会先调用tryAcquire方法，在NonfairSync中实际上会调用nonfairTryAcquire方法，里面的逻辑大部分与公平锁的实现一致，只是不会判断队列是否为空。
```
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
```

总的来讲，它与公平锁的区别就是，先进行一次抢锁，成功则获取锁，不成功则继续判断state，当state=0时，它不会判断当前队列是否为空，这就是两个主要的区别。

![非公平锁](https://images.gitee.com/uploads/images/2020/1218/144302_4cb133f1_8076629.png "屏幕截图.png")


以上就是ReentrantLock的核心内容，它是独占锁，其内部实现了公平锁和非公平锁两种机制，它实现了Lock接口，所以当面试官问起java中lock与synchronized区别时，主要就是指ReentrantLock，那么它们有哪些区别呢？

|      | Synchronized | ReentrantLock |
|------|--------------|---------------|
| 锁的实现 | JVM层面的锁，锁的实现细节不直接对外暴露，其主要是通过监视器来完成加锁过程 | API层面的，依赖于AQS框架，内部实现能直接通过API了解到 |
| 锁的释放 | 方法或代码块执行完成或者异常等，主动释放锁  |   必须要显示的调用unlock()方法   |
| 锁的类型 | 只有非公平锁，它没有CLH队列的概念，其他线程要么在自旋尝试获取锁，要么在阻塞等待唤醒中 | 支持公平锁和非公平锁 |
| 可重入 | 支持可重入 | 支持可重入 |
| 其他区别 | 只关联一个条件队列，即唤醒线程时是通知所有线程 | 可关联多个条件队列，可以选择性通知某些条件下的线程，同时它支持响应中断，超时，尝试获取锁等 |
|  性能 |  做过优化后（引入偏向锁，轻量级锁等），性能已经与ReentrantLock没有太大差别 |  性能无差别  |


### 3 ReentrantReadWriteLock 读写原理知道吗？能介绍一下吗？
3.1 ReentrantReadWriteLock 实现了ReadWriteLock接口，该接口的两个方法分别对应读锁和写锁：
```
public interface ReadWriteLock {
    
    Lock readLock();

    Lock writeLock();
}

```
先看一下写锁的实现，当我们创建 

final ReentrantReadWriteLock rwlock = new ReentrantReadWriteLock();

调用 rwlock.writeLock().lock();方法时，此时是写锁，看一下lock()方法的具体实现：

```
public void lock() {
    sync.acquire(1);
}
// 其中sync 对象是ReentrantReadWriteLock 内部类Sync的实例，acqure方法是aqs中的内容，上面已经列出源码，这里不再多说，它会先调用tryAcquire（）方法
abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final boolean tryAcquire(int acquires) {
            // 当前线程
            Thread current = Thread.currentThread();
            // 锁状态
            int c = getState();
            // 写线程数量
            int w = exclusiveCount(c);
            if (c != 0) {
                // 如果c不为0，w=0 则说明有读锁（state读和写都可以修改），或者w!=0 但是获取写锁的线程不是当前线程（可重入），则获取锁失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //判断同一线程获取写锁是否超过最大次数（65535）
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 可重入，state+1，获取锁成功
                setState(c + acquires);
                return true;
            }
            // c=0 的情况，说明没有被锁，先判断writerShouldBlock()方法，该方法在公平锁FairSync（其实就是判断等待队列是否为空）和非公平锁NonfairSync中被实现，cas设置state 成功则获取锁，否则获取锁失败
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
}

static final class NonfairSync extends Sync {
        final boolean writerShouldBlock() {
            return false; // 不需要等待
        }
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }

static final class FairSync extends Sync {
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }

```
以上是ReentrantReadWriteLock 写锁的加锁过程，可以看到基本上与ReentrantLock类似，只是多了一些判断，另外就是写锁也是独占的。

调用 rwlock.readLock().lock();方法时，看一下lock()方法的具体实现：
```
public void lock() {
    sync.acquireShared(1);
}
...
//AQS中acquireShared 方法，先调用tryAcquireShared 方法，该方法在ReentrantReadWriteLock内部有实现
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final int tryAcquireShared(int unused) {
           
            Thread current = Thread.currentThread();
            // 锁状态
            int c = getState();
            // 如果当前有写锁并且写锁不是当前线程拥有，则获取锁失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            // 读锁数量
            int r = sharedCount(c);
            // 1.readerShouldBlock() 方法，如果是公平锁，则判断是否有等待队列，有的话就需要等待，如果是非公平锁，则是判断队列中的下一个节点是否是写锁，如果是则会等待写锁完成，jdk为了避免写线程过分饥渴，而做出的策略
            // 2.cas获取锁
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            // 在上面的步骤获取失败的情况下，则会自旋尝试获取锁，知道明确的成功或是被，可以看到fullTryAcquireShared方法中的逻辑与上面的执行判断差不多
            return fullTryAcquireShared(current);
        }

        
        final int fullTryAcquireShared(Thread current) {
            
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
}

```
以上就是ReentrantReadWriteLock读锁的主要逻辑。结合写锁的加锁过程，可以知道ReentrantReadWriteLock：读读共享，读写互斥，写写互斥，即只有读的时候是共享的，有写的情况还是独占的。

另外写锁和读锁的释放这里就不在多说，有兴趣的可以看一下源码。

通过 ReentrantLock 我们了解到了独占锁，公平和非公平锁这些概念，通过ReentrantReadWriteLock了解了共享锁的概念，AQS中还有一些重要的东西：

1） 获取不到锁的线程如何加入队列的？加入队列后它在干什么？锁释放后线程又怎么出队列获取锁呢？


2） AQS同步队列和条件队列有什么关系，条件队列是什么？


### 4.什么是CAS？如何实现的？

4.1 CAS：compareAndSwap，判断内存中的值与预期的值是否相等，相等则更新为新值。



AbstractQueuedSynchronizer（AQS）、ReentrantLock、ReentrantReadWriteLock、compareAndSwap(CAS)