---
title: Java8 线程池详解一
date: 2018-05-24 15:24:37
tags: [ThreadPoolExecutor]
---

*本文代码来自 JDK8*

一般调用线程是通过 Thread 或者 Runable, 这样做就要自己管理线程, 不方便, 所以线程池就出现了. 简单来讲, 线程池保存几个线程, 然后有任务过来了, 就调用线程执行这个任务. 在 JDK 中就默认通过 ThreadPoolExecutor 实现了线程池, 它继承了 AbstractExecutorService 抽象类, 实现了 ExecutorService 和 Executor 接口, 通过 execute 或者 submit 方法提交任务给线程池执行.

<!-- more -->

---

#### 重入锁 ReentrantLock
在开始讲解线程池之前, 我们需要了解重入锁, 这里只是简单说明, 需要深入了解的同学可以找资料研究.
所谓重入锁, 是指一个线程可以对同一个资源重复加锁, 为什么要这样定义? 因为比如方法 1 用了锁, 方法 2 也用了锁, 当方法 1 调用方法 2 时, 如果不允许重入锁, 那么就造成死锁了, 所以才需要锁能重入. 当然加多少次锁就要释放多少次锁. 为了保证一定能释放锁, 释放锁的操作一定要放到 finally 块中.
ReentrantLock 实现的原理是 AbstractQueuedSynchronizer, 队列同步器, 简称 AQS, 基础也是基于 CAS 算法.
ReentrantLock 用 state 表示是否加锁, 0 表示没有锁, 当前线程可以持有锁, 1 表示有其他线程持有锁, 那么需要把当前线程加入到同步队列等待(这里实际是用双向链表表示队列). 重复加锁会让 state 加一, 至于这里对 state 变量操作, 当然是用 CAS 算法操作, 这样才能保证只有一个线程可以修改成功, 也就只有一个线程可以持有锁. 
下面简单说下获取锁的过程:
1. 调用 ReentrantLock 的 lock() 方法;
2. 该方法通过 CAS 算法把 state 从 0 更新为 1, 成功就说明没有其他线程持有锁, 本线程就持有锁了, 同时会用一个变量保存这个持有锁的线程;
3. 更新失败就判断本线程是否持有锁的线程, 是的话就把 state 加 1, 然后就可以开心地跑任务了;
4. 前面的判断都不是那就只能把本线程加入到同步队列中了;
5. 接下来用一个死循环让线程不断轮询, 对于同步队列的线程就阻塞(实际是调用 LockSupport.park() 方法阻塞, 具体怎么阻塞就不讨论了), 等待释放锁后的唤醒, 然后同步队列的下一个线程才能获取锁执行, 其他线程还是在等待中.

接着再说说释放锁的过程:
1. 调用 ReentrantLock 的 unlock() 方法;
2. 然后会让 state 减一, 注意这里不必用 CAS 算法更新 state 的值, 因为前面的加锁操作已经决定了只有一个线程可以持有锁, 那么释放锁的操作肯定也只有一个线程在执行了, 所以不必用 CAS 保证原子操作;
3. 如果 state 为 0, 说明锁全部释放了, 需要把之前保存持有锁线程的变量设置为空, 然后把同步队列的后继线程唤醒.

以上就是 AQS 的大概实现原理, 在这里我们只需要理解可以通过 ReentrantLock.lock() 获取锁, 通过 ReentrantLock.unlock() 释放锁就行, 以后有机会再来详细探讨吧.

---

#### 线程中断 interrupt
除了要了解重入锁, 我们还需要懂得线程中断的一些知识, 这里也简单介绍中断, 并不会详细说明, 喜欢深入研究的同学可以找相关资料研究. 主要介绍的是 Thread 类的三个中断方法:
1. `public void interrupt()`
这个就是产生线程中断的方法, 但并不是说调用这个方法就能让线程停止了, Java 中是不会允许直接停止一个线程的, 毕竟说不定线程打开了其他资源, 如果直接停止那么这些资源就没有关闭了, 很危险的操作. 
这个方法实际是产生一个中断标志, 用来表示调用者希望这个线程能够中断, 这时如果线程是处于阻塞, 那么线程就抛出一个中断异常, 同时把这个中断标识清除. 
因此我们真正要处理的是判断是否有这个中断标识, 有的话就可以做一些操作, 比如不再执行下去; 或者捕捉是否有中断异常, 有的话也可以做一些收尾操作等等, 具体要做什么都是自己决定的.  那么异常捕捉就不用说啦, 我们又是怎么判断是否有中断异常呢? Thread 类提供了两个方法来判断.
2. `public static boolean interrupted()`
这是个静态方法, 用来判断当前线程是否有中断标识, 如果有就返回 true, 而且同时会把中断标识清除, 这也意味着如果通过 interrupt() 方法产生中断标识, 然后连续执行 interrupted() 方法, 第一次返回 true, 之后的都是返回 false.
3. `public boolean isInterrupted()`
这是个成员方法, 需要通过 Thread 的实例调用, 也是用来判断线程是否有中断标识, 有就返回 true, 不过它跟 interrupted() 方法不同, 它不会把中断标识清除. 因此如果有中断标识, 连续执行 isInterrupted() 都是会返回 true. 

---

#### 状态和线程数变量
1. `private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));`
状态计数器, 又是典型的一个变量多种使用, AtomicInteger 是使用 CAS 算法的线程安全类, 一个整型有 32 位, 其中高 3 位用来表示线程池的状态, 低 29 位用来表示线程数.
2. `private static final int COUNT_BITS = Integer.SIZE - 3;`
线程数比特位数, 值为 29, 表示线程数量最多为 2^29-1. 至于为什么是 32-3, 这是因为要用 3 位比特位表示线程池的状态, 线程池的状态有 5 个, 3个比特位足够表示了.
3. `private static final int CAPACITY   = (1 << COUNT_BITS) - 1;`
用来与运算得出线程池的线程数量以及线程池的状态, 值是: 0001 1111 1111 1111 1111 1111 1111 1111.
4. 状态值
```java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
	这里统一左移 29 位, 目的是为了方便 ctl 通过逻辑运算计算状态值, 注意这些状态是按从小到大排序.
	**RUNNING**: 运行状态, 表示正在处理任务和接收任务.
	**SHUTDOWN**: 关闭状态, 表示不接收新任务, 但会继续处理队列的任务. 通过 shutdown() 方法可以进入这个状态.
	**STOP**: 停止状态, 表示不接收新任务, 也不处理队列的任务, 而且还要中断正在执行的任务. 按照上面介绍的中断, 这里的中断任务也不是真的中断, 具体还是看任务是不是阻塞, 如果是阻塞那么就会抛出中断异常停止执行了, 如果没有阻塞那么还要看任务代码里面有没有对中断状态处理, 如果没有任务也还是会执行下去的. 通过 shutdownNow() 方法可以进入这个状态.
	**TIDYING**: 整理状态, 表示所有任务执行完成.
	**TERMINATED**: 结束状态, 表示线程池全部结束.


#### 操作状态和线程数的方法
1. 计算状态值
```java
// c 表示 ctl 的值
// 跟非 CAPACITY 与运算, 实际是得到 ctl 高 3 位的值, 很方便和状态值比较
private static int runStateOf(int c)     { return c & ~CAPACITY; }
```
2. 计算线程数
```java
// c 表示 ctl 的值
// 跟 CAPACITY 与运算, 实际是得到 ctl 低 29 位的值, 也就是线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }
```
3. 计算状态计数器 ctl 的值
```java
// rs 表示状态值, wc 表示线程数
// 两者或运算直接得出 ctl 的值
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
4. 计算当前状态是否小于某个状态
```java
// 这里 c 表示 ctl
// 因为状态值从小到大排序, 可以这样比较
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
```
5. 计算当前状态是否大于等于某个状态
```java
// 这里 c 表示 ctl
// 因为状态值从小到大排序, 可以这样比较
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
```
6. 判断是否运行状态
```java
// 这里 c 表示 ctl
// 因为状态值从小到大排序, 可以这样比较
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

---

#### 任务执行涉及的成员变量
1. `private volatile int corePoolSize;`
核心线程数, 线程池最小线程数量. 任务多时会新建临时线程执行, 任务少时会回落到核心线程数. 当没有任务时, 临时线程等待超时就会结束, 核心线程会一直等待, 直到有任务来执行.
2. `private final BlockingQueue<Runnable> workQueue;`
任务队列, 任务多时会把新任务放到队列等待, 队列满了才会新建临时线程执行, 如果临时线程也满了就执行拒绝策略, 拒绝策略有几种, 以后会说. 注意这个队列允许 poll() 返回 null, 所以需要根据 isEmpty() 判断是否空.
3. `private volatile int maximumPoolSize;`
最大线程数, 核心线程数 + 临时线程数不能超过这个值.
4. `private final ReentrantLock mainLock = new ReentrantLock();`
线程池主锁, 用来线程池内部的一些同步操作.
5. `private volatile RejectedExecutionHandler handler;`
拒绝策略处理器, 可以自定义, 默认是抛出异常.
6. `private volatile long keepAliveTime;`
临时线程空闲超时时间, 超过这个时间就会结束.
7. `private final HashSet<Worker> workers = new HashSet<Worker>();`
保存线程的集合, 也就是线程池. Worker 是一个封装了任务及执行这个任务的线程的类, 可以把 Worker 看成一个线程. 因为 HashSet 不是线程安全的, 所以对它的操作需要持有锁才行.
8. `private volatile boolean allowCoreThreadTimeOut;`
如果是 true, 核心线程空闲超时时间以 keepAliveTime 为准; 如果是 false, 核心线程是不会超时. 默认 false.

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*