---
title: TreeMap 详解四
date: 2018-04-25 17:44:03
tags: [TreeMap]
---

#### 遍历

TreeMap 使用中序遍历, 遍历的方式与 HashMap, LinkedHashMap 相同, 通过 entrySet().iterator() 方法返回 Iterator 实例遍历.<!-- more -->
```java
public Set<Map.Entry<K,V>> entrySet() {
    EntrySet es = entrySet;
    return (es != null) ? es : (entrySet = new EntrySet());
}
```

```java
// 迭代器和 TreeMap 实例绑定一起, 所以用非静态内部类
class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public Iterator<Map.Entry<K,V>> iterator() {
    	// 注意这里传入了一个参数, 前面说到是用中序遍历方式, 所以这个方法是返回最左节点
        return new EntryIterator(getFirstEntry());
    }
    ...
}

// 返回最左节点作为遍历开始的节点
final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    if (p != null)
        while (p.left != null)
            p = p.left;
    return p;
}
```

```java
final class EntryIterator extends PrivateEntryIterator<Map.Entry<K,V>> {
    EntryIterator(Entry<K,V> first) {
        super(first);
    }
    // 这里调用的是父类的 nextEntry 方法返回下一个节点
    public Map.Entry<K,V> next() {
        return nextEntry();
    }
}
```

#### PrivateEntryIterator
```java
/**
 * 作为 EntryIterator 的父类, 这个类同样是 TreeMap 的内部类
 */
abstract class PrivateEntryIterator<T> implements Iterator<T> {
    Entry<K,V> next;
    Entry<K,V> lastReturned;
    int expectedModCount;

    // 一开始 next 指向最左节点
    PrivateEntryIterator(Entry<K,V> first) {
        expectedModCount = modCount;
        lastReturned = null;
        next = first;
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Entry<K,V> nextEntry() {
        Entry<K,V> e = next;
        if (e == null)
            throw new NoSuchElementException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 中序遍历返回下一个节点
        next = successor(e);
        lastReturned = e;
        return e;
    }
    ...
}
```

---

#### successor
TreeMap 之所以用中序遍历来迭代, 是因为左子树比节点要小, 右子树比节点要大, 只有中序遍历方式才会输出从小到大的节点. 在前面已经把最左节点传入给迭代器了, 这是遍历开始的第一个节点, 接下来主要工作就是如何确认节点的后继, 可以分两种情况讨论:
1. 节点有右子树
该节点的后继就是它右子树的最左节点, 如下图, 0005 节点的后继就是 0008.
![](/images/c437439a04c0b2056eebe8aef4806b0.png)
2. 节点没有右子树
该节点的后继就是它所在的左子树的第一个父节点, 如下图, 0006 节点的后继就是 0007. 而如果向上追溯发现该节点不是处于左子树, 那么这个节点就是最后节点, 如 0009 就是最后节点.
![](/images/1ec868d1ffee7d42f8bb0203d34311f.png)
```java
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;

    // 有右子树, 对右子树进行最左遍历, 返回最左节点
    else if (t.right != null) {
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } 

    // 没有右子树, 向上追溯第一个左子树节点
    else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        // 这里通过节点是否父节点的右子树来找到第一个左子树
        // 如果 p 为空, 说明已经追溯到根节点了, 没有后继了, 直接返回空值 p
        // 如果 ch != p.right, 说明 ch 是 p 的左孩子, p 就是要找的后继了
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*