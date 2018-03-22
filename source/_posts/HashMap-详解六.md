---
title: HashMap 详解六
date: 2018-03-21 17:39:43
tags: [HashMap, treeifyBin]
---

#### 链表转树结构
根据详解四, 当链表长度大于 8 时, 为了更高效的查询, 需要转成红黑树结构, 使用的方法是 treeifyBin. 过程是先把链表结构调整为双向链表结构, 再把双向链表结构调整为红黑树结构.
```java
/**
 * tab: 数组
 * hash: 新节点 key 的哈希值, 通过运算就可以得出需要转成树的对应链表的对应索引值
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    
    // 对于数组为空或者数组长度小于阈值, 需要扩容
    // 这里阈值是 64, 数组长度不是很长, 但是却需要转成树结构,
    // 说明很多冲突了, 先扩容看看情况有没有改善, 大概吧
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        // 遍历链表, 然后把链表变成双向链表
        do {
            // 创建树节点, 表示当前节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // hd 是头结点, t1 用来指向下一个节点
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

/**
 * 通过普通链表节点构造树节点
 */
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}

/**
 * 这个方法不是 HashMap 的方法, 是它内部类 TreeNode 的方法, 将双向链表转成红黑树结构
 */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        // 设置根节点
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;

            // p 指向树的当前节点
            // x 表示双向链表的当前节点
            // 开始遍历树
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;

                // 双向链表的当前节点哈希值和树的当前节点哈希值比较
                // 小于就把 dir 设置 -1
                // 大于就把 dir 设置 1
                // 等于就把 dir 设置 0
                // 这里可以参考上一 part 的 putTreeVal 方法同样的遍历方式
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;

                // 根据 dir 决定向左遍历还是向右遍历, 一直到叶子节点
                // 然后把双向链表的当前节点插入到叶子节点
                // 最后通过 balanceInsertion 方法调整红黑树, 参考上一 part
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }

    // 调整数组索引值指向的根节点, 参考上一 part
    moveRootToFront(tab, root);
}
```

到这里, HashMap 插入键值对的过程就基本介绍完了, 至于获取键值对就不仔细展开讨论了, 涉及到的遍历都是一样的.