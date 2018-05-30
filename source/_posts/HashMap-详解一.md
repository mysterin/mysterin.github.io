---
title: HashMap 详解一
date: 2018-03-13 10:46:49
tags: [HashMap, 变量]
---
*本文代码来自JDK8*

#### 实现原理
1. 建立一个数组
2. 根据元素哈希值计算数组索引, 保存到数组
3. 索引号相同的元素通过链表保存
4. 链表长度超过范围转红黑树保存

---

#### 默认常量
* 初始长度大小: DEFAULT_INITIAL_CAPACITY = 1 << 4, 为了区分容量和元素数目, 这里就用长度表示容量
* 最大长度大小: MAXIMUM_CAPACITY = 1 << 30
* 默认加载因子: DEFAULT_LOAD_FACTOR = 0.75f
* 链表转红黑树的阈值: TREEIFY_THRESHOLD = 8
* 红黑树转链表的阈值: UNTREEIFY_THRESHOLD = 6
* 转换红黑树要求最小数组长度: MIN_TREEIFY_CAPACITY = 64
---
#### 变量
* transient Node<K,V>[] table: 保存元素的数组
* transient int size: 元素个数, 不是数组长度
* int threshold: 对数组扩容的阈值, 容量\*加载因子
* final float loadFactor: 加载因子, 越大填充的数据就越多, 冲突也会越多; 越小填充的数据就越少, 冲突也会减少, 为什么取 0.75 默认值就不深入讨论了

---
#### 哈希值计算
```java
// key为空, 对应哈希值为 0
// 否则将 key 的哈希值进行扰动: 高 16 位与低 16 位进行异或
// 这样做的原因是当哈希值与数组长度减一进行与运算得出索引时, 
// 对于数组长度比较小的情况, 如果没有扰动, 哈希值的高位将不参与运算, 最后的索引值很有可能会重复
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
---
关于 tableSizeFor 方法留到下一 part 再讲.

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---