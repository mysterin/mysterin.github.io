---
title: Hashtable 详解二
date: 2018-04-26 12:04:36
tags: [Hashtable]
---

#### 遍历
```java
/**
 * 这里是通过 Collections.sysnchronizedSet 方法生成一个线程安全的 Set 实例
 * 实际是重写 Set 方法, 用对 this 也就是 Hashtable 的实例同步代码块包裹实际调用的方法
 * 而实际调用的都是 EntrySet 对应的方法
 * 这里就不展开说了, 详细可以看 Collections 类的内部类 SynchronizedCollection
 */
public Set<Map.Entry<K,V>> entrySet() {
    if (entrySet==null)
        entrySet = Collections.synchronizedSet(new EntrySet(), this);
    return entrySet;
}
```

```java
private class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    /**
     * 通过调用 Hashtable 的 getIterator 方法返回
     * ENTRIES 是一个常量, 表示迭代的是节点, 不是节点的 key 或者节点的 value
     */
    public Iterator<Map.Entry<K,V>> iterator() {
        return getIterator(ENTRIES);
    }
}

/**
 * Hashtable 没有值, 返回一个空的迭代器
 * 否则返回一个 Enumerator 实例
 */
private <T> Iterator<T> getIterator(int type) {
    if (count == 0) {
        return Collections.emptyIterator();
    } else {
        return new Enumerator<>(type, true);
    }
}
```

#### Enumerator
```java
private class Enumerator<T> implements Enumeration<T>, Iterator<T> {
    Entry<?,?>[] table = Hashtable.this.table;
    int index = table.length;
    Entry<?,?> entry;
    Entry<?,?> lastReturned;
    int type;

    /**
     * Indicates whether this Enumerator is serving as an Iterator
     * or an Enumeration.  (true -> Iterator).
     */
    boolean iterator;

    /**
     * The modCount value that the iterator believes that the backing
     * Hashtable should have.  If this expectation is violated, the iterator
     * has detected concurrent modification.
     */
    protected int expectedModCount = modCount;

    Enumerator(int type, boolean iterator) {
        this.type = type;
        this.iterator = iterator;
    }

    // Iterator methods
    public boolean hasNext() {
        return hasMoreElements();
    }

    public T next() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        return nextElement();
    }

    public boolean hasMoreElements() {
        Entry<?,?> e = entry;
        int i = index;
        Entry<?,?>[] t = table;
        /* Use locals for faster loop iteration */
        while (e == null && i > 0) {
            e = t[--i];
        }
        entry = e;
        index = i;
        return e != null;
    }

    public T nextElement() {
        Entry<?,?> et = entry;
        int i = index;
        Entry<?,?>[] t = table;
        /* Use locals for faster loop iteration */
        // et 为空说明已经遍历到链表尾部了, 从数组后面往前遍历, 找到非空元素
        while (et == null && i > 0) {
            et = t[--i];
        }
        entry = et;
        index = i;
        if (et != null) {
            Entry<?,?> e = lastReturned = entry;
            // 用 entry 保存下一个节点
            entry = e.next;
            // 然后根据 type 返回节点, key 或者 value
            return type == KEYS ? (T)e.key : (type == VALUES ? (T)e.value : (T)e);
        }
        throw new NoSuchElementException("Hashtable Enumerator");
    }
    ...
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---