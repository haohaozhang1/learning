**ThreadPoolExecutor**是一个有固定核心线程数的线程池，下面根据源码来详细介绍一下**ThreadPoolExecutor**的设计和思想

首先看一下**ThreadPoolExecutor**的类图

![这里写图片描述](http://img.blog.csdn.net/20160328210120970)

首先了解一下ThreadPoolExecutor的5种状态
```
// runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

1. RUNNING    表示线程池可以接受任务，并且可以运行队列中的任务
2. SHUTDOWN 停止线程池接受任务，还可以继续运行队列中的任务
3. STOP  停止线程池接受任务，不可以运行队列中的任务，并尝试去中断正在执行的任务
4. TIDYING 表示所有的任务都已经通知，核心线程数为0，转到TIDYING的线程即将运行terminated()方法
5. TERMINATED  表示terminated（）执行完毕

5种状态的转换有以下几种方式。 
RUNNING -> SHUTDOWN：调用了shutdown方法，或者线程池实现了finalize方法，在里面调用了shutdown方法。 
(RUNNING or SHUTDOWN) -> STOP：调用了shutdownNow方法 
SHUTDOWN -> TIDYING：当队列和线程池均为空的时候 
STOP -> TIDYING：当线程池为空的时候 
TIDYING -> TERMINATED：terminated()钩子方法调用完毕
```
ThreadPoolExecutor executors = (ThreadPoolExecutor) Executors.newFixedThreadPool(size);
```
这是创建**ThreadPoolExecutor**的代码，**size**为固定的核心线程数，看一下源码最终都会走到

```

public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

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
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

 1. corePoolSize  线程核心数
 2. maximumPoolSize 最大的线程数（包括线程核心数+非核心数）,发现Android sdk中直接把最大线程数==线程核心数
 3. keepAliveTime  当线程执行完，等到多长时间销毁  
 4. unit keepAliveTime  的单位 
 5. workQueue  工作线程队列
 6. threadFactory 创建线程的工厂类
 7. handler reject的任务的处理就是靠这个来完成的(当workQueue满了并且达到了最大线程的个数的时候会拒绝加进来的任务，或者调用了shutdown函数之后再加入任务也是会reject的)。

启动ThreadPoolExecutor有俩种方式

```
ThreadPoolExecutor executors = (ThreadPoolExecutor) Executors.newFixedThreadPool(size);
executors.submit(new Runnable());
executors.execute(new Runnable());
```

首先看一下submit源码

```
/**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```

 RunnableFuture<T> ftask = newTaskFor(task, result); 这一行代码只是设置了FutureTask（也继承了Runnable）的回调task和回调以后返回的值result。不管你submit的时候传入的是Runnable还是Callable最后RunnableFuture(FutureTask)里面都会生成Callable对象。任务调用的时候调用RunnableFuture(FutureTask)的run方法，run方法调用Callable对象的call方法。

**下面主要看一下execute（runnable）**

```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        //线程数小与核心线程数
        if (workerCountOf(c) < corePoolSize) {
	        //添加work到线程池,返回true代表加入成功并运行
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //如果c是正在运行的，就加入workQueue等待执行的队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //再次判断是否为running，如果不是就remove掉
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //是running状态，然后判断线程池中的核心线程数 如果为0，代表没有指定核心线程数size为0
            else if (workerCountOf(recheck) == 0)
	            //添加非核心线程到线程池
                addWorker(null, false);
        }
        //只有俩种情况 1：不是running  2：是running但是核心线程数达到最大
        //添加到非核心线程，如果还返回false，那就直接reject
        else if (!addWorker(command, false))
            reject(command);
    }
```
看上面的注释就是意思

然后看一下addWorker做了什么

```
private boolean addWorker(Runnable firstTask, boolean core) {
        ...一些线程池的状态判断 省略

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
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

**上面代码最主要的逻辑就是 组装Worker，然后workers.add（worker）。**

看一下Worker类

```
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
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker. */
        public void run() {
            runWorker(this);
        }
```

 上面代码最主要的就是初始化 firstTask和thread，run方法执行runWorker（this）方法
 this.thread = getThreadFactory().newThread(this);下面看一下生成Thread代码

```
static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```
创建Thread，如果对Thread不了解的可以去看一下上一篇[Java和Android的Thread源码分析](http://blog.csdn.net/android_text/article/details/70256166)。
设置thread为非守护线程，设置thread的优先级为默认的5

Work类就剩下public void run() {runWorker(this);}分析

```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
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
```

 beforeExecute(wt, task);和afterExecute(task, thrown);是空实现，开发者可以自由的操作
 第23行 task.run(); 真正每个任务要做的逻辑在这个里面。而且我们前面也说过task是FutureTask，调用FutureTask里面的run方法会调用FutureTask里面Callable的call方法，call方法调用完之后保存住了call的返回值。FutureTask 可以通过get方法得到这个返回值。 

看一下getTask（）

```
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
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
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
```
**getTask（）里是实现核心线程堵塞循环的关键；android sdk中的ThreadPoolExecutor的源码里**

```
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize
```


**会永远返回false；**

```
Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
```

**一定会走到take（）方法。**

LinkedBlockingQueue下的poll
```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

```
while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
```
**当为空的时候，等到keepTime然后就返回null，不会造成堵塞。**

看一下take（）源码

```
public E take() throws InterruptedException {
	E x;
	int c = -1;
	final AtomicInteger count = this.count;
	final ReentrantLock takeLock = this.takeLock;
	takeLock.lockInterruptibly();
	try {
	    while (count.get() == 0) {
	        notEmpty.await();
	    }
	    x = dequeue();
	    c = count.getAndDecrement();
	    if (c > 1)
	        notEmpty.signal();
	} finally {
	    takeLock.unlock();
	}
	if (c == capacity)
	    signalNotFull();
	return x;
}
```

```
while (count.get() == 0) {
	notEmpty.await();
}
x = dequeue();
c = count.getAndDecrement();
```
**当count==0；会notEmpty.await();造成堵塞。
然后核心线程不断的从workQueue里取出work执行。**


**总结流程如下：**
**1.当池子大小小于corePoolSize就新建线程，并处理请求**

**2.当池子大小等于corePoolSize，把请求放入workQueue中，池子里的空闲线程就去从workQueue中取任务并处理，如果workQueue没有就堵塞等待**

**3.当workQueue放不下新入的任务时，新建线程入池，并处理请求，如果池子大小撑到了maximumPoolSize就用RejectedExecutionHandler来做拒绝处理**

**4.另外，当池子的线程数大于corePoolSize的时候，多余的线程会等待keepAliveTime长的时间，如果无请求可处理就自行销毁**

内部结构如下：

![这里写图片描述](http://img.my.csdn.net/uploads/201012/7/0_12917139691L3R.gif)

**从中可以发现ThreadPoolExecutor就是依靠BlockingQueue的阻塞机制来维持线程池，当池子里的线程无事可干的时候就通过workQueue.take()阻塞住。**



