---
title: HashMap 详解三
date: 2018-03-14 10:53:23
tags: [HashMap, resize]
---

#### resize 方法
数组为空或者元素数量超过阈值, 将会执行 `resize()` 方法, 结果是将数组的长度加倍
```java
final Node<K,V>[] resize() {
    // 设置旧数组, 旧长度, 旧阈值
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;

    // 初始新长度 新阈值为 0
    int newCap, newThr = 0;

    // 对原来已经有元素的数组进行处理
    if (oldCap > 0) {
    	// 元素数目超出最大范围, 把阈值调整到最大值, 同时返回原数组
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 对于长度没有超过最大范围, 长度加倍, 阈值也加倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }

    // 旧数组长度为 0, 但是旧阈值不为 0, 那么新阈值还是等于旧阈值 
    // 这种是在指定初始长度构造 HashMap 的情况, 可以通过上一 part 了解下
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;

    // 对于旧数组长度为 0, 阈值也为 0 的情况, 
    // 初始化新长度为默认长度 16, 
    // 新阈值为`默认长度*默认加载因子` 16*0.75 = 12
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }

    // 只有指定初始长度构造 HashMap 才会出现新阈值为 0 这一情况, 
    // 同样通过长度*加载因子计算出新阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;

    // 新建新数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    // 把旧数组的元素复制到新数组, 对于元素处理有三种情况
    // 1. 只有一个元素, 没有形成链表或者树, 直接把元素放入新数组
    // 2. 该元素是树节点, 那么按照树的方式处理
    // 3. 如果是链表结构, 那么需要遍历链表
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 只有一个元素
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 树节点
                // 因为涉及到红黑树, 具体复制过程留到以后新增元素后再具体分析
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 链表节点, 这部分具体实现放到下面来讲
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

---
#### 复制旧链表到新数组
由于元素所在索引值是通过哈希值和数组长度减一进行与操作来得到, 当数组扩容一倍后, 元素在新数组的索引值就有两种情况, 一种是跟旧数组索引值一样, 另一种是旧数组索引值加旧数组长度.
举个例子, 因为数组长度一直都是 2 次幂, 那么假设旧数组长度是 10000, 减一是 **1111**, 新数组长度 100000, 减一就是 **11111**. 元素在旧数组的索引值是 xxxx, 在新数组的索引值便是 0xxxx 或者 1xxxx = xxxx + 10000(旧数组长度), 因此可以直接通过**旧索引值+旧数组长度**来得出新索引值.

那么现在的问题就是如何判断元素是在原来的索引位置还是新的索引位置, 其实就是判断哈希值的高位是 0 还是 1, 这里可以通过元素的哈希值和旧数组长度进行与运算得出, 如果结果是 0, 对应的就是原来位置, 非 0 对应新位置.

1. 源代码用头尾节点来记录链表, 因为要生成两个链表, 所以有两对头尾节点, 如下:
```java
// 旧索引位置对应的头尾节点
Node<K,V> loHead = null, loTail = null;
// 新索引位置对应的头尾节点
Node<K,V> hiHead = null, hiTail = null;
// 记录下一个节点
Node<K,V> next;
```


2. 开始遍历旧数组链表, 源代码用了 do...while() 方式循环遍历, 如下:
```java
do {
	next = e.next;
	...

} while ((e = next) != null)
```

3. 判断该元素是放到旧索引位置还是新索引位置, 判断方法是上面所说的哈希值和旧数组长度做与运算, 如下:
```java
// 与运算为 0, 说明是旧索引位置
if ((e.hash & oldCap) == 0) {
    // 尾节点是空, 说明这是第一个元素, 用头节点指向这个元素
    if (loTail == null)
        loHead = e;
    // 否则就把这个元素连接到链表的最后
    else
        loTail.next = e;
    // 最后尾节点指向链表新增的节点, 也是最后一个节点
    loTail = e;
}
// 否则是新索引位置
else {
    // 同上
    if (hiTail == null)
        hiHead = e;
    else
        hiTail.next = e;
    hiTail = e;
}
```

4. 当一个链表遍历完成了, 我们将得到两个新链表, 把这两个新链表保存到数组就完成了, 如下:
```java
// 尾节点不为空, 链表是存在的, 把头节点保存到新数组的旧索引值
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}

// 把头节点保存到新数组的新索引值
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---