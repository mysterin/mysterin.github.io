---
title: Set 实现类介绍
date: 2018-05-15 17:37:41
tags: [Set, HashSet, LinkedHashSet, TreeSet]
---

*本文代码来自JDK8*

Set 继承于 Collection 继承于 Iterable, 作用是存储不重复的元素, 常用的实现类有 HashSet, LinkedHashSet, TreeSet.

#### 实现方式
Set 实际的实现方式都是在内部维护一个 Map 实例, 然后将新增的元素作为 key, 用一个 Object 实例作为 value, 保存到 Map 实例. 因为 Map 是不会存在重复的 key, 因此这种保存方式就不会造成重复元素存在. 可以看下下面的代码:
```java
// HashSet, LinkedHashSet, TreeSet 都是这样新增元素的
// 如果 key 不存在, put 方法返回 null, add 方法就返回 true
// 如果 key 已存在, put 方法返回旧值, add 方法就返回 false

private static final Object PRESENT = new Object();
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

#### 遍历方式
Set 的遍历实际就是对内部 map 的 key 进行遍历, 最终都是通过调用 map.keySet().iterator() 方法返回迭代器, 然后对 key 进行遍历. 当然这里面会根据 map 不同遍历结果也不同, 比如 HashSet 内部是用 HashMap 保存, LinkedHashSet 内部是用 LinkedHashMap 保存, TreeSet 内部是用 TreeMap 保存. 因此, HashSet 不会排序, LinkedHashSet 会根据先后插入顺序排序遍历, TreeSet 会根据元素比较排序遍历, 反正最后都是根据 map 来遍历的.

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---