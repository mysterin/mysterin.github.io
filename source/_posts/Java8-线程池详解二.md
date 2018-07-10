---
title: Java8 线程池详解二
date: 2018-05-28 17:08:43
tags: [ThreadPoolExecutor]
---

上一 part 我们了解了线程池的成员变量, 接下来我们来说说如何跑一个线程.

<!-- more -->

#### submit
submit 既接收 Runable 类型参数也接收 Callable 类型参数, 最终还是构造一个 Runable 参数传递给 execute 方法执行, 区别在于 submit 方法是有返回值的.
```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

#### execute
execute 接收一个 Runable 类型参数, 也就是一个任务, 最终是通过线程池构建新线程执行这个任务.
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 对于线程数小于 corePoolSize
    // 通过调用 addWorker() 这个方法尝试用新线程执行任务, 成功就返回了
    // 如果新增线程失败, 那么会继续向下执行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 这里把任务加入到任务队列
    // 再次检查线程池状态, 这是为了防止其他线程修改了状态
    // 不是 RUNNING 状态, 就会执行拒绝策略
    // 如果还是 RUNNING 状态, 但是线程数为 0, 需要从任务队列中取任务执行
    // addWorker(null, false) 就是读任务队列执行
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }

    // 这里主要是处理任务队列已经满了的情况
    // 会尝试新建线程执行任务
    // 当然失败的话还是要执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

#### addWorker
```java
/**
 * firstTask 表示要添加的任务
 * core 如果是 true, 那么用 corePoolSize 作为边界大小, 否则用 maximumPoolSize
 * 这个方法的目的就是创建一个新线程执行任务
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    // 虽然线程池是用来创建管理线程的, 但它也支持多个线程对线程池操作的
    // 所以当很多线程同时往线程池添加任务时, 就会来到下面的死循环了
    // 只有被允许执行任务的线程才能跳出循环
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 如果是停止, 整理, 结束状态, 不接受新任务, 也不跑旧任务, 直接返回
        // 如果是关闭状态并且有新任务, 也返回, 因为关闭状态不接受新任务
        // 如果是关闭状态并且队列为空, 还是返回, 因为这时没有旧任务要执行了
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            // 检查边界, 超出范围就返回
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;

            // 通过 CAS 算法尝试让 ctl 加 1, 也就是线程数加 1
            // 成功就说明允许执行这个任务
            // 几个线程在这里竞争, 只有一个能成功跳出循环
            // 其他线程就继续循环, 继续竞争
            if (compareAndIncrementWorkerCount(c))
                break retry;

            // 还要判断是否被其他线程改变状态, 如果是就从 retry 重新开始循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 经过上面两个循环, 来到这里就说明能添加任务了
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
    	// 这里用 Worker 这个类封装任务
    	// 这个类有个成员变量 thread, 它是由一个线程工厂创建的
    	// 当然这个 thread 只是创建, 并没有执行, 它接收 Worker 本身作为参数
    	// 所以可以把这个 Worker 看成一个任务, 也可以看成一个线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 这里要加锁, 因为 workers 的操作只能有一个线程
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                // 这里分两种情况
                // 一是线程池处于运行状态, 任务可以随便加
                // 二是线程池虽然是关闭状态, 但是任务为空, 还是可以执行旧任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 把这个新线程加入到线程池中
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 任务添加成功, 开始执行线程跑任务
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
经过上面分析, 我们大致明白了线程池会在什么情况下才会新增一个线程来跑任务, 也明白了什么情况下会拒绝一个任务, 但还是会有些疑问, 在 execute 方法中没有得到线程执行而是加入到任务队列的那些任务是怎么执行的? `addWorker(null, false)` 这个方法又是怎么保证会读任务队列的任务然后执行? 毕竟在 addWorker 的方法里面并没有看到读任务队列, 反而是会创建一个 Worker 实例, 传了空任务, 然后创建一个线程来跑这个空任务, 这样是不是没有意义? 要了解这些, 我们需要来分析分析 Worker 这个类才行.

#### Worker
```java
/**
 * 这是一个非静态内部类, 因为要用到外部类的成员方法, 所以是非静态
 * 注意这里继承了 AbstractQueuedSynchronizer 这个类, 参考上一 part 介绍的重入锁
 * 因此 Worker 具有锁功能. 至于为什么要让它具有锁功能? 这个需要在关闭线程池那里再说
 */
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    final Thread thread;
    Runnable firstTask;

    /**
     * 构造函数, 接受 Runable 的参数
     * 然后自己通过线程工厂新建一个线程, 接受自己作为参数
     * 构造方式跟 addWorker 方法说的一样, 最后都是通过 thread 执行任务的
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /**
     * 因为 Worker 实现了 Runable 接口, 所以需要实现 run 方法
     * 因此 thread 最后是调用这个 run 方法执行任务的
     * 而这个方法实际是调用外部类也就是线程池的 runWorker 方法, 下面详细分析这个方法
     */
    public void run() {
        runWorker(this);
    }

    ...
}
```

#### runWorker
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();

    // 提取 Worker 的任务, 成员变量 firstTask 设置为空
    // 因为只有 firstTask 为空才会读任务队列的任务出来执行
    // 这里设置为空是为了下一次任务读取做准备
    Runnable task = w.firstTask;
    w.firstTask = null;

    // 这里释放锁说是为了允许中断, 因为在构造 Worker 时 state 就被设置为 -1
    // 所以这里释放锁就是把 state 设置为 0, 为什么之前要禁止中断在 shutdown 方法说
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
    	// 接下来就执行 Worker 本身的任务或者读取任务队列的任务, getTask 方法下面再说
        while (task != null || (task = getTask()) != null) {
            // 这里加锁表示开始线程执行任务, 该线程不是空闲线程了
            // 但是一个 Worker 对应一个线程, 那么这里的加锁有什么意义
            // 貌似没有其他线程跟它竞争, 原因放到 shutdown 方法说
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 如果是停止状态, 就确保线程设置了中断
            // 如果不是停止状态, 就确保线程没有设置中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {

            	// 这个是保留方法, 留着继承扩展的
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 终于可以开始跑任务了
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

#### getTask
```java
/**
 * 这个方法会判断是否启用线程空闲超时
 * 一般是线程数大于核心线程数才会启用
 * 然后如果任务队列一直没有任务过来, 就返回空, 从而让多出的线程结束掉
 * 当线程数降低到核心线程数就不启用线程空闲超时了
 * 就会一直在等待, 等新的任务过来才返回
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 如果是停止状态, 返回空, 线程数减一
        // 如果是关闭状态并且任务队列没有任务了, 也返回空, 线程数减一
        // 返回空是会导致线程结束
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 是否要线程空闲超时
        // allowCoreThreadTimeOut 默认是 false, 一般可以忽略
        // 所以当线程数大于核心线程数才会有线程空闲超时一说
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 线程数一般小于 maximumPoolSize, 所以第一个条可以忽略
        // timeOut 只有在下面读取任务队列返回空才是 true
        // 因此线程数大于核心线程数以及任务队列返回空就尝试把线程数减一, 然后返回空
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 如果设置了超时, 那就按超时时间读取任务队列
            // 否则就等待一直到任务队列有新任务过来
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

---

#### shutdown
```java
/**
 * 这个方法是用来关闭线程池
 * 不再接受新任务, 但是还是会执行队列的任务
 * 当然会把空闲的线程中断
 * 还有一个 shutdownNow 方法是用来停止线程池, 它会中断所有线程, 实际操作差不多, 不细说了
 */
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 修改线程池状态为关闭
        advanceRunState(SHUTDOWN);
        // 主要说这个关闭空闲线程的方法, 因为它涉及到上面说的 Worker 加锁问题
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

private void interruptIdleWorkers() {
    // 实际是调用这个方法
    interruptIdleWorkers(false);
}

/**
 * onlyOne 表示是否只关闭一个空闲线程
 * 当然上面的调用是传了 false, 所以还是会关闭所有空闲的线程
 */
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 遍历线程池, 找出没有设置中断标识的线程
        // 尝试获取 Worker 的锁, 因为上面执行任务前都会对 Worker 加锁
        // 这里如果能获取成功, 就说明没有执行任务, 是空闲的线程
        // 所以就可以把这个空闲的线程设置中断标识了
        // 一般来说, 空闲线程都是阻塞在 getTask 这个方法
        // 这里设置中断标识会导致阻塞线程抛出异常, 从而结束掉
        // 这是我个人的推测, 并没有验证过, 有不同意见的同学可以在 Issues 留言
        // 在这里还可以得出为什么 Worker 要在 runWorker 方法开头释放锁的答案
        // 因为一开始 Worker 只是创建线程, 并没有执行它
        // 如果不加锁, 在线程执行之前执行 shutdown 方法, 会在这里把这个线程中断
        // 然而线程还没执行, 是不会产生中断标识的, 所以才要在构造线程时禁止中断
        // 等到线程开始执行了才允许中断, 也就是释放锁
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

经过上面分析, 我们大概明白了线程池的工作流程. 下面再总结一次:
1. 一开始线程池是没有线程的, 只有来了任务才会新建线程;
2. 当到了核心线程数就暂时不会继续增加线程了, 而是把任务都放到任务队列;
3. 线程完成一个任务会自动从任务队列读取任务出来执行;
4. 当任务队列满了, 才会继续新建线程执行任务, 但线程数不能超过最大线程数, 当然这些新线程在执行完任务同样要从任务队列读取任务继续执行;
5. 当到了最大线程数还继续来新的任务, 那么就执行拒绝策略, 一般是抛出异常;
6. 当队列的任务都执行完了, 超过核心线程数的线程就会自动结束;
7. 小于核心线程数的线程则继续保留, 等待新的任务到来.

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*
