---
title: LinkedHashMap 详解一
date: 2018-04-23 14:47:33
tags: [LinkedHashMap, 变量]
---

*本文代码来自JDK8*

#### 性质
1. LinkedHashMap 继承于 HashMap, 具备 HashMap 的一切性质;
2. LinkedHashMap 会按先后插入顺序对元素排序遍历;
3. LinkedHashMap 会额外使用双向链表结构来表示插入的元素.

#### 变量
1. **transient LinkedHashMap.Entry<K,V> head**
表示双向链表的头部
2. **transient LinkedHashMap.Entry<K,V> tail**
表示双向链表的尾部
3. **final boolean accessOrder**
true: 表示把最后访问的节点放到双向链表的最后一位, 访问的方式有替换旧节点和读取节点