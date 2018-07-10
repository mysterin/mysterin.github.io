---
title: HashMap 详解七
date: 2018-04-23 10:31:01
tags: [HashMap, 遍历]
---

#### 使用 Iterator 遍历
通过 HashMap.entrySet().iterator() 方法获取迭代器, 使用 next 方法对 HashMap 进行遍历.

<!-- more -->

```java
HashMap<String, String> map = new HashMap<>();
Iterator it = map.entrySet().iterator();
while(it.hasNext()) {
	Map.Entry<String, String> entry = it.next();
}
```
下面详细讲解各个方法的作用, 其实迭代器之所以能遍历元素节点, 主要是应用了内部类. 通过内部类可以访问外部类的变量和方法, 从而完成遍历节点.

---

#### entrySet()
```java
/**
 * 直接返回 EntrySet 的实例
 * 注意这里 entrySet 不是静态方法, 而 EntrySet 是非静态的内部类, 所以可以直接 new 实例
 */
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

---

#### EntrySet
```java
/**
 * EntrySet 继承于 AbstractSet
 */
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    ...

    /**
     * 返回 EntryIterator 实例, 这也是属于 HashMap 的非静态内部类
     */
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    ...
}
```

---

#### EntryIterator
```java
/**
 * HashMap 的非静态内部类
 */
final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
    /**
     * next 方法调用父类 HashIterator 的 nextNode 方法, 返回下一个元素
     */
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```

---

#### HashIterator
```java
/**
 * HashMap 的内部抽象类
 */
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    /**
     * 构造函数, 从 0 开始遍历 HashMap 的保存数组, 一直到非空元素
     */
    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();

        // 从根节点开始遍历链表, 其中树也当成链表结构来遍历, 一直到尾节点
        if ((next = (current = e).next) == null && (t = table) != null) {
            // 链表遍历完全后, 重新读取数组的下一个非空元素
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }
}
```

以上就是 HashMap 的遍历方法, 它不是按照插入节点的先后顺序进行遍历, 而是按照数组结构来遍历.

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*