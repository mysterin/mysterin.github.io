---
title: Hashtable 详解一
date: 2018-04-26 11:26:47
tags: [Hashtable]
---

*本文代码来自JDK8*

Hashtable 和 HashMap 十分类似, 它是继承 Dictionary, 这个类已经不推荐使用了. 下面列举几点 Hashtable 与 HashMap 的不同之处.
<!-- more -->
1. Hashtable 是线程安全的;
2. Hashtable 的数组长度可以自定义, 不一定是 2 的幂次方, 默认初始长度是 11;
3. Hashtable 同样用链表保存, 但不会转换成树结构;
4. Hashtable 的 key, value 都不允许为空.

---

#### put
```java
/**
 * 注意这里使用了 synchronized 关键字, 说明插入方法是线程安全的
 */
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();

    // 这里计算索引的方法跟 HashMap 是不同的
    // HashMap 是通过哈希值与数组长度减一进行与运算直接得出
    // 这里则是对数组长度求余运算得出
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];

    // 遍历链表, 如果有相同 key 的元素, 就替换旧值
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    // 添加新值
    addEntry(hash, key, value, index);
    return null;
}
```

#### addEntry
```java
private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;

    // 对于元素数量大于阈值, 需要对数组扩容
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    // 这里创建新节点, 插入到链表头部
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

#### rehash
```java
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    // 新数组长度变成原来的 2 倍再加 1
    int newCapacity = (oldCapacity << 1) + 1;
    // 对可能超过最大长度做处理
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

    // 遍历旧数组
    for (int i = oldCapacity ; i-- > 0 ;) {
        // 遍历链表
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            // 计算新索引值, 然后把元素插入到新链表头部
            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*