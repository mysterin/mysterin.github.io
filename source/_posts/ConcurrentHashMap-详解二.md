---
title: ConcurrentHashMap 详解二
date: 2018-05-04 15:26:44
tags: [ConcurrentHashMap]
---

接下来讨论下不同的插入情况

#### 插入元素到索引位置
```java
if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null,
                 new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
}

/**
 * 读取 tab 对应位置 i 的值
 * 这里使用原子读方式, 为什么要这样做暂时没搞明白, 这里给出自己一个假设
 * tab 的元素不是 volatile, 虽然节点把 val 和 next 设置 volatile
 * 但 tab 索引的第一个元素不算 volatile, 如果 tab[i] 方式读取可能是读取工作内存的值
 * 所以用原子读方式读取主内存的值才行
 */
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

/**
 * CAS 算法, 如果 tab 的位置 i 的值与 c 相同, 就用 v 更新它
 */
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

#### 插入元素时正在扩容
有可能线程 A 进行插入时, 线程 B 正在对数组扩容, 这时候我们需要协助扩容
```java
// 当哈希值是 MOVED, 表示其他线程在扩容, 参考上一 part 扩容部分
if ((fh = f.hash) == MOVED)
    // 这个方法放到下面 addCount 方法后面讲
    tab = helpTransfer(tab, f);

/**
 * ForwardingNode 类
 */
static final class ForwardingNode<K,V> extends Node<K,V> {
	// 表示扩容后的新数组
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}


```

#### 插入元素到链表或者树
```java
V oldVal = null;
// f 是头结点, 先把它锁上, 防止其他线程同时操作这个链表或者树
synchronized (f) {
	// 再次判断这个头结点是不是原来的头结点
	// 这是为了防止在锁上前一刻被其他线程修改了
    if (tabAt(tab, i) == f) {
        // hash 值大于等于 0, 说明这是链表, 参考上一 part
        // 然后就是链表节点的插入了, 跟 HashMap 差不多, 可以参考 HashMap 详解
        if (fh >= 0) {
            binCount = 1;
            for (Node<K,V> e = f;; ++binCount) {
                K ek;
                if (e.hash == hash &&
                    ((ek = e.key) == key ||
                     (ek != null && key.equals(ek)))) {
                    oldVal = e.val;
                    if (!onlyIfAbsent)
                        e.val = value;
                    break;
                }
                Node<K,V> pred = e;
                if ((e = e.next) == null) {
                    pred.next = new Node<K,V>(hash, key,
                                              value, null);
                    break;
                }
            }
        }

        // 注意对于树结构, ConcurrentHashMap 是用 TreeBin 进行封装
        // 插入树节点也可以参考 HashMap, 差不多的过程
        else if (f instanceof TreeBin) {
            Node<K,V> p;
            binCount = 2;
            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                           value)) != null) {
                oldVal = p.val;
                if (!onlyIfAbsent)
                    p.val = value;
            }
        }
    }
}
if (binCount != 0) {
    // 链表转树结构, 同样可以参考 HashMap 详解
    if (binCount >= TREEIFY_THRESHOLD)
        treeifyBin(tab, i);
    if (oldVal != null)
        return oldVal;
    break;
}
```

---

#### 是否扩容
节点插入后会调用 addCount 方法来判断是否需要扩容
```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 这里是统计容量 size, 执行加一操作, 不仔细讨论了
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 需要扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // 其他线程正在扩容
            if (sc < 0) {
            	// 扩容完成, 本线程就不需要继续扩容了
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // CAS 把 sizeCtl 成功加一, 本线程开始协助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }

            // 没有其他线程扩容, 说明本线程是第一个扩容的
            // 此时就把 sizeCtl 设置成一个非常大的负数
            // 因为是第一个扩容, 所以新数组是 null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

/**
 * 返回一个标识, 这个标识经过 RESIZE_STAMP_SHIFT 左移必定为负数
 * Integer.numberOfLeadingZeros 返回 n 对应 32 位二进制数左侧 0 的个数
 * 如 9（0000 0000 0000 0000 0000 0000 0000 1001）返回 28
 * 1 << (RESIZE_STAMP_BITS - 1) = 2^15，其中 RESIZE_STAMP_BITS = 16
 * RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS = 16
 */
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

/**
 * 协助扩容方法
 * 经过 addCount 方法, 这个协助扩容就更容易理解了
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // 同样是循环判断是否扩容完成
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 同样是再次判断是否扩容完成
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 同样是把 sizeCtl 加一, 然后协助扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---