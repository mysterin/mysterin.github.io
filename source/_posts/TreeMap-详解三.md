---
title: TreeMap 详解三
date: 2018-04-25 17:20:07
tags: [TreeMap]
---

读取节点比较简单, 只是一个遍历过程而已.
#### get
```java
// 实际调用 getEntry 方法
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}
```

#### getEntry
```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    // 自定义比较器, 节点的读取另外处理
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    // key 和当前节点的 key 进行比较, 遍历树, 返回对应节点
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

#### getEntryUsingComparator
```java
final Entry<K,V> getEntryUsingComparator(Object key) {
    @SuppressWarnings("unchecked")
        K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        Entry<K,V> p = root;
        // 通过比较器, key 和当前节点比较, 遍历树, 读取节点
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
    }
    return null;
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---