## AbstractQueuedSynchronizer（AQS）

#### 1. 什么是AQS？原理是什么？

近年来AQS这个框架被面试的几率越来越高，说实话早些年的时候我也不知道它是个什么玩意，因为项目中没有亲自写过，所以对它一片茫然，后来去看了下它的源码，发现它只不过是个抽象类，那些基于它的实现才是突出它重要的原因，这些具体实现类包括ReentrantLock、ReentrantReadWriteLock、Semaphore、CountDownLatch等，当然，它们不是直接基础AbstractQueuedSynchronizer，而是在内部定义了NonfairSync，FairSync这种类去实现它。同时它也是模板方法设计模式的经典应用。下面是AbstractQueuedSynchronizer的部分源码：


```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer{

    //用于
    private volatile int state;

    protected final int getState() {
        return state;
    }

    protected final void setState(int newState) {
        state = newState;
    }
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // 四个核心的模板方法，tryAcquire，tryRelease是独占锁的加锁与释放方法；tryAcquireShared，tryReleaseShared是共享锁的加锁与释放方法
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
        
}

```




#### 2. 


AbstractQueuedSynchronizer（AQS）、ReentrantLock、ReentrantReadWriteLock、compareAndSwap(CAS)