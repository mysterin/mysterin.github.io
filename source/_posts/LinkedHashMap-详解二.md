---
title: LinkedHashMap 详解二
date: 2018-04-23 15:16:37
tags: [LinkedHashMap, put, newNode, afterNodeAccess, afterNodeInsertion]
---

#### put
LinkedHashMap 的 put 方法也是使用 HashMap 的方法, 不同在于重写了 newNode(), afterNodeAccess 和 afterNodeInsertion 这几个方法, 这几个方法的调用可以看 *HashMap-详解四*, 下面具体讲讲如何重写这几个方法.

---

#### newNode
```java
/**
 * 根据 key-value 创建双向链表节点
 * e 表示下一个节点, 不过这里是空值, 不用理会
 */
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

/**
 * 继承 HashMap 的静态内部类 Node
 */
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

/**
 * 把新节点插入到双向链表尾部
 */
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    // 如果这是空链表, 新节点就是头结点
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

---

#### afterNodeAccess
```java
/**
 * 1. 使用 get 方法会访问到节点, 从而触发调用这个方法
 * 2. 使用 put 方法插入节点, 如果 key 存在, 也算要访问节点, 从而触发该方法
 * 3. 只有 accessOrder 是 true 才会调用该方法
 * 4. 这个方法会把访问到的最后节点重新插入到双向链表结尾
 */
void afterNodeAccess(Node<K,V> e) { // move node to last
    // 用 last 表示插入 e 前的尾节点
    // 插入 e 后 e 是尾节点, 所以也是表示 e 的前一个节点
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
    	// p: 当前节点
    	// b: 前一个节点
    	// a: 后一个节点
    	// 结构为: b <=> p <=> a
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // 结构变成: b <=> p <- a
        p.after = null;

        // 如果当前节点 p 本身是头节点, 那么头结点要改成 a
        if (b == null)
            head = a;
        // 如果 p 不是头尾节点, 把前后节点连接, 变成: b -> a
        else
            b.after = a;

        // a 非空, 和 b 连接, 变成: b <- a
        if (a != null)
            a.before = b;
        // 如果 a 为空, 说明 p 是尾节点, b 就是它的前一个节点, 符合 last 的定义
        else
            last = b;

        // 如果这是空链表, p 改成头结点
        if (last == null)
            head = p;
        // 否则把 p 插入到链表尾部
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

---

#### afterNodeInsertion
```java
/**
 * 插入新节点才会触发该方法
 * 根据 HashMap 的 putVal 方法, evict 一直是 true
 * removeEldestEntry 方法表示移除规则, 在 LinkedHashMap 里一直返回 false
 * 所以在 LinkedHashMap 里这个方法相当于什么都不做
 */
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---