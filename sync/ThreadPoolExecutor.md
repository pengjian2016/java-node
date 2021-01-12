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
- workQueue: 等待队列
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

参考:[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

### 2. 等待队列（阻塞队列）有哪些？

无界队列：
- PriorityQueue，LinkedTransferQueue，DelayQueue

有界队列：
- ArrayBlockingQueue，LinkedBlockingQueue，SynchronousQueue，LinkedBlockingDeque

![美团有界](https://images.gitee.com/uploads/images/2021/0112/175955_8f42b4a2_8076629.png "屏幕截图.png")

### 3. 拒绝策略有哪些？

- 丢弃新提交的任务并抛出异常，默认策略；
- 丢弃任务但不抛异常；
- 丢弃等待时间最长的任务，提交该任务；
- 有提交任务的线程执行，即执行ThreadPoolExecutor.execute()方法所在的线程，执行；

![美团拒绝策略](https://images.gitee.com/uploads/images/2021/0112/180552_124ec43c_8076629.png "屏幕截图.png")

### 4. 线程池解决的是什么问题？

### 5. 达到最大线程数后线程如果回收？

