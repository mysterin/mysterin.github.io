---
title: TreeMap 详解一
date: 2018-04-25 14:42:14
tags: [TreeMap]
---

*本文代码来自JDK8*

1. TreeMap 继承于 AbstractMap, 并不是继承 HashMap, 它跟 HashMap 是同级;
2. TreeMap 直接使用红黑树来保存节点;
3. 在 HashMap 是使用哈希值来比较大小, TreeMap 则不是, 而是需要 key 实现了 Comparable 接口来互相比较大小;
4. 如果没有传入自定义比较器, TreeMap 不能使用 null 作为 key.

---

#### 变量
1. **private final Comparator<? super K> comparator**
比较器, 可以通过构造方法传入, 一旦传入比较器就按这个来排序插入操作, 默认是空
2. **private transient Entry<K,V> root**
红黑树的根节点
3. **private transient int size = 0**
节点数量
4. **private transient int modCount = 0**
树结构改变次数

---

#### compare
TreeMap 使用 compare 方法来比较两个 key 的大小, 如果没有传入比较器 comparator, 那么需要 key 实现 Comparable 接口, 具体代码如下:
```java
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```

---

#### Entry
TreeMap 同样自定义了属于自己的节点内部类, 因为这个节点类是不能依赖外部类的实例, 所以使用静态内部类
```java
// 这里用布尔型表示红色黑色, true是黑色, false是红色
private static final boolean RED   = false;
private static final boolean BLACK = true;

 static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    // 这里默认是黑色, 根据 HashMap-详解五插入时还是要先改成红色
    boolean color = BLACK;

    /**
     * Make a new cell with given key, value, and parent, and with
     * {@code null} child links, and BLACK color.
     */
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    
    // 省略 get/set 等方法
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---