---
title: ConcurrentHashMap 详解一
date: 2018-05-02 17:07:56
tags: [ConcurrentHashMap]
---

*本文代码来自JDK8*

1. ConcurrentHashMap 实现了线程安全;
2. 虽然可以通过 Hashtable 或者 Collections.synchronizedMap 来生成一个线程安全的 Map 实例, 但这是全局锁方式, 性能不行;
<!-- more -->
3. ConcurrentHashMap 的存储方式和 HashMap 十分类似, 都是用数组 + 链表 + 红黑树的结构, 差别在于一些操作需要做线程同步处理;
4. 在 JDK7 ConcurrentHashMap 使用 Segment 分段锁的方式实现线程安全, 而在 JDK8 就抛弃这种做法, 采用 CAS 算法来保证线程安全, 这里就不展开讨论分段锁的方式了, 有兴趣可以去找 JDK7 的源码分析.


#### 内存模型
要了解 Java 的线程安全, 首先要知道它的内存模型. 内存模型是一个概念, 简单来讲就是它定义了虚拟机里多线程如何访问变量. 这里涉及到两个概念: 主内存和工作内存. 线程之间的共享变量都是保存在主内存, 每个线程如果想要访问变量, 那么需要先把变量从主内存加载到私有的工作内存, 然后对工作内存的变量进行操作, 最后再更新到主内存中. 注意区分这里的内存模型跟硬件内存不是一码事, 简单理解为主内存和工作内存都可以包含 CPU 寄存器, CPU 缓存, RAM.

#### volatile
根据内存模型, 假设有一个成员变量 value, 线程 A 修改了它, 实际是修改线程 A 私有的工作内存里面的 value 的副本, 还没有更新到主内存的 value, 因此线程 B 读取的 value 还是旧的 value, 这样就不同步了.
而使用 volatile 修饰的成员变量 value 有个特点, 那就是假如线程 A 修改了这个变量的值, 那么它会通知其他线程应该去主内存读取新的 value, 这就是可见性.

#### CAS
CAS 全称是比较并交换, 是一种乐观锁机制, 它要求变量对其他线程是可见的, 所以需要用 volatile 修饰, 当然也不是一定要这样, 也可以用 synchronized 来实现, 毕竟这只是个算法, 实现方式多种多样. 涉及到三个参数: 内存值, 期待值, 更新值. 算法有三个步骤:
1. 读取内存值;
2. 比较内存值和期待值;
3. 两者相同就把更新值更新上去.

这三个步骤是一个原子操作, 实际是通过 CPU 总线加锁或者缓存加锁方式来实现, 我也没有具体深入了解, 只知道是个原子操作就行.
在 Java 中 CAS 的实现是在 *Unsafe* 类中, 它有个本地方法 `public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);`, var1 表示对象, var2 表示对象地址, var4 表示期待值, var5 表示更新值. 

---

ConcurrentHashMap 的插入方式跟 HashMap 是一样的, 唯一的区别就是多了线程安全处理, 所以接下来主要是对线程安全详细讲解.

#### 变量
1. `transient volatile Node<K,V>[] table;`
使用 volatile 修饰的数组, 针对该数组引用具有可见性, 对于数组元素没有可见性. 为了保证数组元素也有可见性, 这里就用 volatile 修饰 Node 类的 val 和 next.
2. `private transient volatile Node<K,V>[] nextTable;`
用来扩容时的临时数组
3. `private transient volatile int sizeCtl;`
一个控制字段, -1 表示在初始化数组, -N 表示有 N-1 个线程在扩容, table 初始化后保存下一个次扩容的大小, 跟阈值一样. 这里要说明一下, 虽然都说 -N 表示有 N-1 个线程扩容, 我也没仔细研究是不是这样, 但是在代码里面是这样来设置的: 第一个线程需要扩容, 它会用一个非常大的负数对 sizeCtl 设置, 此后每多一个线程扩容, sizeCtl 就会加一, 然后扩容完成后 sizeCtl 减一, 最后把它设置成新容量的阈值(正数).

#### 初始化
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl 小于 0, 表示数组正在被其他线程进行初始化, 那么挂起本线程
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin

        // U 是 Unsafe 类的实例, 用来操作 CAS 算法
        // 这里使用 CAS 算法把 sizeCtl 改成 -1
        // 如果修改成功会返回 true, 然后初始化 table
        // 因为这个 CAS 条件, 这里的初始化只有一个线程执行, 其他线程是不会进入这里
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 相当于 0,75*n
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

#### 扩容
ConcurrentHashMap 是支持多线程扩容方式, 比如线程 A 把旧数组的索引位置 1 复制到新数组, 同时线程 B 也把旧数组的索引位置 2 复制到新数组, 效率更加快. 另外每个线程都是负责一整个索引区域, 所以扩容原理是先找出本线程应该负责的索引区域, 然后遍历这个区域, 将其中的节点复制到新数组.
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 单个线程允许处理的最少 table 桶首节点数量
    // 即每个线程的任务量
    // 大概是根据 CPU 数量来算出, 为什么这样算我还没弄明白
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    // nextTab 作为临时数组先扩容一倍
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;

    // 这是一个特殊的节点, hash 值设置为 -1, 也就是常量 MOVED
    // 扩容过程中遇到索引位置为空就设置成该节点
    // 或者索引位置不为空, 但是已经处理复制后也把索引位置设置为该节点
    // 目的是为了告诉其他线程不需要再处理该索引位置
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);

    // 表示索引 i 节点是否被复制成功
    boolean advance = true;
    // 表示所有节点复制完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // 这个循环目的很简单
        // 首先我们要知道扩容是一批一批的复制到新数组的
        // 比如把索引范围 [10, 16) 的节点复制到新数组
        // 而这里是逆序扩容, 比如原来数组范围是 [0, 16), 首先是对 [10, 16) 进行复制
        // 还有变量 stride 就是区间大小, 比如这里就是 6
        // 所以这个循环目的就是为了找出允许线程扩容的索引范围 [bound, i]
        // 这里只有更新共享变量 transferIndex 才用到 CAS 算法, 其他操作就不需要了
        while (advance) {
            int nextIndex, nextBound;
            // 满足 [bound, i] 这个区间或者已经完成扩容, 跳出这个循环
            if (--i >= bound || finishing)
                advance = false;
            // nextIndex 是边界 i 的临时保存, 如果小于 0, 说明没有要复制的节点了
            // transferIndex 是共享变量, 保存区间范围的上限, 初始值是旧数组长度
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }

            // 尝试更新 transferIndex
            // 如果成功, 当前线程就负责复制 [nextBound, nextIndex) 范围的节点
            // transferIndex 变成 nextBound
            // 注意这里 i=nextIndex-1, 所以 [nextBound, nextIndex) 也是 [bound, i]
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }

        // 下面开始复制 [bound, i] 范围的节点, 逆序复制, 从 i 开始

        // 对于扩容完成处理
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // sizeCtl 设置为总大小的 0.75
                // 至于这里为什么不用 CAS 更新值, 可能是没有必要吧, 重复更新也没关系
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 扩容完成, sizeCtl 减一      
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
            	// 扩容前 sizeCtl 会设置成 resizeStamp(n) << RESIZE_STAMP_SHIFT + 2
            	// 如果不相等说明有其他线程执行扩容完成的操作了, 本线程不需要重复操作了
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }

        // 对于 i 的节点为空, 那么设置指向特殊节点 ForwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 当前线程判断到这个节点的 hash 值是 MOVED
        // 说明是特殊节点, 已经有其他线程操作了, 可以跳过这个节点
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed

        // 如果 i 既不是空值, 也不是特殊节点, 说明这是个普通节点
        // 那么就开始对这个链表或者树进行复制, 首先是把它锁上, 防止其他线程同时操作它
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;

                    // 下面就是对链表或者树复制的过程, 具体可以参考 HashMap
                    // 值得注意的是对于树结构, 这里索引位置不是直接指向树的根节点
                    // 而是用 TreeBin 构造红黑树, 然后指向这个 TreeBin
                    // TreeBin 的 hash 值设置为 -2, 而其他节点的 hash 值都是大于 0
                    // 节点 hash 值的计算可以参考 spread 方法
                    // 因此可以通过 hash 值大于等于 0 来判断是链表还是树结构
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 复制完成后用特殊节点代替原来节点
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 这里创建 TreeBin 来构造红黑树, 具体构造过程可以参考 HashMap
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 复制完成后用特殊节点代替原来节点
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

/**
 * 计算 key 的 hash 值, 其中 HASH_BITS 是 0x7fffffff, 所以 hash 值一定大于等于 0
 */
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*