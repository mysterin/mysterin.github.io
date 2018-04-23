---
title: LinkedHashMap 详解三
date: 2018-04-23 17:06:14
tags: [LinkedHashMap, 遍历]
---

LinkedHashMap 的遍历方式和 HashMap 的一样, 都是通过 entrySet 方法返回 Set 实例, 然后通过 iterator 方法返回迭代器进行遍历.

#### entrySet
```java
/**
 * 返回 LinkedEntrySet 实例, 这是非静态内部类
 */
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
}

/**
 * 和 HashMap 的 EntrySet 类一样继承 AbstractSet
 * iterator 方法返回 LinkedEntryIterator 实例
 */
final class LinkedEntrySet extends AbstractSet<Map.Entry<K,V>> {
    ...
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new LinkedEntryIterator();
    }
    ...
}


```

---

#### next 和 hasNext
```java
/**
 * next 方法实际是调用父类 nextNode 方法返回节点
 */
final class LinkedEntryIterator extends LinkedHashIterator
        implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}

abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next;
    LinkedHashMap.Entry<K,V> current;
    int expectedModCount;

    /**
     * 构造函数, 从双向链表头节点开始遍历
     */
    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }

    /**
     * 遍历比较简单, 直接读取下一个节点就行
     */
    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after;
        return e;
    }
    ...
}
```