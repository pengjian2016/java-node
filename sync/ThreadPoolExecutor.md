### 线程有哪几种状态?

- 1. 初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。
- 2. 运行(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。
- 3. 阻塞(BLOCKED)：表示线程阻塞于锁。
- 4. 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
- 5. 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回。
- 6. 终止(TERMINATED)：表示该线程已经执行完毕。

务必记住这张图：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0303/092048_e2322873_8076629.png "屏幕截图.png")


参考：

[Java线程的6种状态及切换](https://blog.csdn.net/pange1991/article/details/53860651)


### A，B两个线程如何保证他们的顺序执行？
1. join方法，B线程中join A线程

```
public class TestThread {
    public static void main(String[] args) {
        try {
            Thread a = new Thread(new JoinRunnable("A"));
            a.start();
            a.join();
            Thread b = new Thread(new JoinRunnable("B"));
            b.start();
        } catch (Exception e) {
        }
    }

    public static class JoinRunnable implements Runnable {
        private String name;

        public JoinRunnable(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            System.out.println("Run thread:" + this.name);
        }
    }
}
```
关于Join的说明：主线程等待子线程的终止。也就是说主线程的代码块中，如果碰到了join()方法，此时主线程需要等待（阻塞），等待子线程结束了(Waits for this thread to die.),才能继续执行join()之后的代码块。所以上面的main方法所在的主线程，需要等到子线程a执行完之后才会继续执行后面的代码。

2. 利用单线程池，提交线程，这样每次只有一个线程会被执行

```
Thread a = new Thread(new JoinRunnable("A"));
Thread b = new Thread(new JoinRunnable("B"));
ExecutorService singleThread = Executors.newSingleThreadExecutor();
singleThread.submit(a);
singleThread.submit(b);

```
单线程池的特点，每次只能执行一个任务，这样，A，B会被按照提交的顺序依次执行

3. 锁和wait，notify方法

```
    private static Object lock = new Object();
    private static int state = 0;

    static class ThreadA extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                System.out.println("This is A");
                state++;
                lock.notifyAll();
            }
        }
    }

    static class ThreadB extends Thread {
        @Override
        public void run() {
            try {
                synchronized (lock) {
                    while (state != 1) {
                        lock.wait();
                    }
                    System.out.println("This is B");
                }
            } catch (Exception e) {
            }
        }
    }

    public static void main(String[] args) {
        new ThreadA().start();
        new ThreadB().start();
    }

```

锁和wait，notify方法 应该是比较基础的方式了，虽然我上面实现的很丑陋，但是大致意思差不多，假如B线程先执行先获得了锁，可是state状态并没有被修改成1的时候，它就会释放锁，等待在那里，直到A执行结束之后才会被唤醒继续执行下去。


4. 其他的方法，如利用Lock 和 Condition 精准唤醒线程，Semaphore 信号量等，大家可以自行去专研。


# ThreadPoolExecutor

### 1. 线程池有使用过吗？原理能讲一下吗？

ThreadPoolExecutor的构造函数：

```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }

```
核心参数:
- corePoolSize: 核心线程数
- maximumPoolSize:最大线程数
- workQueue: 等待队列（阻塞队列），待执行的任务（线程）会放在这里

其他参数：
- keepAliveTime：当线程数超过核心线程数时，等待keepAliveTime时间后仍然没有任务要执行时，回收该线程。
- unit：keepAliveTime等待的时间单位，毫秒，秒，分等
- threadFactory 创建新线程的工程
- handler 拒绝策略

ThreadPoolExecutor 原理分析：

主要通过ThreadPoolExecutor.execute方法提交任务，看一下该方法；

```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        int c = ctl.get();
        // 当前正在工作的线程是否小于核心线程数，如果是，则创建线程（Worker）加入到工作队列中（注意它和等待队列不是一个）并执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 当前线程数大于核心线程数时，则判断等待队列是否已满（workQueue.offer 方法如果元素个数大于数组长度则返回失败，否则加入到数组中），
        // 等待队列没满的情况下则加入到等待队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 如果等待队列已满，则判断是否达到最大线程，没有的情况下则创建线程并执行任务，否则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }

    // 在execute 方法中两次调用addWorker方法，其中core参数是不一样的，其他都一样。
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 判断是大于核心线程还是最大线程数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建工作线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 执行任务
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```

借用一下美团的技术图，描述整体流程：

![美团](https://images.gitee.com/uploads/images/2021/0112/174846_873f1a84_8076629.png "屏幕截图.png")

### 总结线程池的核心原理

1. 当有心的任务提交过来时，先判断核心线程数是否已满，如果未满则直接创建新的线程执行任务；

2. 如果核心线程数已满，则判断等待队列是否已满，如果未满，则加入到等待队列中；

3. 如果等待队列已满，则判断是否达到最大线程数，如果未达到，则创建新的线程执行任务；

4. 如果已达到最大线程数，则执行拒绝策略；


参考:[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

### 2. 等待队列（阻塞队列）有哪些？

无界队列：
- PriorityQueue，LinkedTransferQueue，DelayQueue

有界队列：
- ArrayBlockingQueue，LinkedBlockingQueue，SynchronousQueue，LinkedBlockingDeque

![美团有界](https://images.gitee.com/uploads/images/2021/0112/175955_8f42b4a2_8076629.png "屏幕截图.png")

需要说明的是，当使用无界队列时，maximumPoolSize和拒绝策略都没有了意义，因为无界意味着等待队列无限大，可以容得下所有的待提交的任务。

### 3. 拒绝策略有哪些？

- 丢弃新提交的任务并抛出异常，默认策略；
- 丢弃任务但不抛异常；
- 丢弃等待时间最长的任务，提交该任务；
- 有提交任务的线程执行，即执行ThreadPoolExecutor.execute()方法所在的线程，执行；

![美团拒绝策略](https://images.gitee.com/uploads/images/2021/0112/180552_124ec43c_8076629.png "屏幕截图.png")

### 4. 线程池解决的是什么问题？或者是有什么好处？

- 降低资源消耗：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗
- 提高响应速度：任务到达时，无需等待线程创建即可立即执行
- 提高线程的可管理性：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控
- 提供更多更强大的功能：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行


### 5. 达到最大线程数后工作线程如何回收呢？
在execute方法中我们知道，当创建线程执行任务时，实际上会创建Worker并执行，worker本身也是个线程，它也实现了run方法：

```
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
    final Thread thread;
    
    Runnable firstTask;
    
    // 构造函数
    Worker(Runnable firstTask) {
        setState(-1); 
        // addWorker 时提交的那个要执行的任务
        this.firstTask = firstTask;
        // thread为该work
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        // 在上面的代码中我们知道addWorker后会调用w.thread的start方法启动线程，最后会执行run方法
        runWorker(this);
    }
    /*
        首次提交的任务那个任务firstTask会立即得到执行，执行结束后，下一次循环会调用getTask()方法
        从等待队列(阻塞队列)中获取任务，如果获取不到任务，即task=null后，会调用processWorkerExit方法退出，即回收该工作线程
    */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
    
    private Runnable getTask() {
        boolean timedOut = false; // 

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 线程池状态停止的情况下，需要回收当前线程，运行情况是RUNNING是小于0的
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            // 当前工作线程数
            int wc = workerCountOf(c);

            // 核心线程数如果允许超时或者工作线程数大于核心线程数的情况下
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 如果大于最大线程或者超时了，回收该工作线程，否则一直循环获取任务
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //在大于核心线程数或允许超时时间的情况下，等待keepAliveTime后仍然没获取到任务，则标记timedOut超时，下一次循环可能被回收
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
}
```

### 6. 线程池的几种状态

![来自美团](https://images.gitee.com/uploads/images/2021/0113/101859_0a3a633a_8076629.png "屏幕截图.png")

### 7. 如何在不提交任务的情况下，创建工作线程？
为什么有这样的问题？，我们知道，线程的创建是耗资源的东西，可能我们的应用刚开始上线的时候没啥问题，但是运行一段时间后内存就爆满了等等，也许就是在运行的过程中创建了过多的线程呢。而上面的源码中我们也知道只有在execute提交任务后才会创建工作线程，当然这只是默认情况下，那么有没有办法在项目启动或者不提交任务的时候就把线程池初始化好呢？
下面的两个方法：


```
    // 创建一个核心线程
    public boolean prestartCoreThread() {
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }

    /**
     * 启动所有的核心线程
     *
     */
    public int prestartAllCoreThreads() {
        int n = 0;
        while (addWorker(null, true))
            ++n;
        return n;
    }
```

