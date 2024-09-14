# **什么是线程池**

线程池其实是一种池化的技术实现，池化技术的核心思想就是实现资源的复用，避免资源的重复创建和销毁带来的性能开销。线程池可以管理一堆线程，让线程执行完任务之后不进行销毁，而是继续去处理其它线程已经提交的任务。

## 使用线程池的好处

- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

# **线程池的构造**

Java 主要是通过构建 ThreadPoolExecutor 来创建线程池的。接下来我们看一下线程池是如何构造出来的

ThreadPoolExecutor 的构造方法：

```java
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
    // 核心线程数
    this.corePoolSize = corePoolSize;
    // 最大线程数
    this.maximumPoolSize = maximumPoolSize;
    // 任务队列
    this.workQueue = workQueue;
    // 空闲线程的最大存活时间
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    // 线程池内部创建线程所用的工厂
    this.threadFactory = threadFactory;
    // 线程池的拒绝策略
    this.handler = handler;
}
```

所以我们刚创建出来的线程池里面是没有线程的，只初始化了工作队列和一些线程池参数。

# 线程池的状态

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池的状态保存在`ctl`这个变量中，这个变量中同时还保存了当前线程池的线程个数，变量的高3位保存的是线程池状态，低29位保存的是线程数。

`workerCountOf(int c)` 方法从变量中取出线程数；

`runStateOf(int c)` 方法从变量中取出线程池状态；

# 提交任务

提交任务调用的是线程池的`execute(Runnable command)` 方法：

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
         
        // c是当前线程池的状态值
        int c = ctl.get();
        
        // 如果当前运行中的线程数 < 核心线程数，那么就调用addWorker方法，增加核心线程来执行任务
        // 注意：这个方法有可能会return false，因为并发情况下核心线程数有可能满，或者线程池状态不符合了
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        
        // 如果核心线程数满了，检查线程池的运行状态，并且尝试把任务放进任务队列
				~~~~if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            
            // 任务放进队列以后还要再检查一次，避免任务放进去以后线程池状态又变了
            // 如果线程池状态不符合，直接走拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池运行状态没问题，这里还要再检查一次是否还有线程再运行中，
            // 如果没有线程再运行中了会导致没有线程来执行刚才放进去的任务了
            // 注意这里没有把command传进去是因为command已经放进队列了，worker会直接从队列拿
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        
        // 如果队列也放不进去，那就尝试新增工作线程来执行任务；
        // 如果工作线程也新增失败，那么就走拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

接下来时调用到的`addWorker(Runnable firstTask, boolean core)`方法：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 如果线程池状态不是RUNNING（从前面可以知道RUNNING状态是一个复数，SHUTDOWN是0，
        // 其他状态都是正数），需要return false；但是一种情况例外：
        // SHUTDOWN状态 & 初始任务为空 & 任务队列不为空，没有提交初始任务，
        // 并且队列不为空，说明这个worker是要用来执行队列中的剩余任务的，应该通过
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
            return false;

        for (;;) {
		        // 获取线程数
            int wc = workerCountOf(c);
            // 如果线程数>=最大线程数，或者：
            // 新增核心线程数时>=核心线程数；新增工作线程时 >= 最大线程数，新增失败
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // compareAndIncrementWorkerCount方法对ctl变量+1，意思时线程池的最大线程数+1
            // 成功的话跳出循环，失败的话从头再来
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
		    // 新增一个worker，worker的构造方法会初始化thread变量
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
		        // 加锁，这里因为workers用的是HashSet，线程不安全，所以要先加锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                // 线程池状态为RUNNING(rs < SHUTDOWN)
                // 或者线程池状态为SHUTDOWN，但是并没有提交firstTask说明这个worker是要执行队列任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 理论上线程刚创建出来还没有执行过任务，不应该是alive状态
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 把worker加入工作线程
                    workers.add(w);
                    // 记录线程池达到过的最大线程数
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // worker添加标志位置为true
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
		            // 真的启动线程
                t.start();
                // worker开始执行标志位置为true
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            // 线程启动失败，需要从工作线程中移除对应的Worker
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

# Worker

上面的代码中创建了一个worker，从worker中拿到了他的成员变量thread，并且开启了线程，Worker的定义如下：

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    // 执行任务的线程
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    // 待执行的第一个任务，有些场景会传递firstTask，例如第一次创建worker时
    Runnable firstTask;
    /** Per-thread task counter */
    // 当前worker完成的任务计数
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // newThread的时候worker把自身传递进去了，所以thread.start()的时候实际上会调用
        // worker的run()方法
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

从上面的方法可以看出worker的run方法主要就是调用了 `runWorker(Worker w)`方法，此时就已经是在工作线程中执行了：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 拿到初始任务
    Runnable task = w.firstTask;
    // 上面已经拿到初始任务，这里把worker的引用置空，避免下次又执行这个任务
    w.firstTask = null;
    w.unlock(); // allow interrupts
    // worker被异常中断的标识，最后正常结束会修改为false
    boolean completedAbruptly = true;
    try {
		    // 这里循环获取任务，task是出师任务，getTask是从任务队列获取任务，getTask很重要，后面会详解
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
		            // 任务执行前的切面
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
		                执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
		                // 任务执行后的切面
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 任务完成数+1
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
		    // 获取不到任务了才会走到这里来，执行worker退出的操作
        processWorkerExit(w, completedAbruptly);
    }
}
```

这里我们可以看到，worker如果获取不到任务就直接退出循环，在finally块里面调用了worker退出的方法，那工作线程的等待时间是怎么实现的呢，答案就在getTask方法中；

# getTask

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 这里是判断如果当前线程获取不到任务，当前线程是否可以退出：
        // allowCoreThreadTimeOut是否允许核心线程退出，如果是的话那当前线程肯定是可退出的；
        // wc > corePoolSize线程总数是否大于核心线程数，如果是的话那当前线程也是可以退出的；
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
		        // 如果当前线程是可推出的，就阻塞获取，阻塞时间是keepAliveTime，否则不阻塞
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                // 获取到任务了就返回任务，说明当前线程还需要继续执行任务
                return r;
            // 获取不到任务把timedOut置为true，下一次循环就可以判断线程是否可以退出了
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```