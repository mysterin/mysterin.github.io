---
title: HashMap 详解二
date: 2018-03-13 16:23:45
tags: [HashMap, tableSizeFor]
---

#### tableSizeFor 方法
初始化 HashMap 的长度大小, 会调用 tableSizeFor 方法赋值给 threshold
```java
// 构造函数
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;

    //主要看这个
    this.threshold = tableSizeFor(initialCapacity);
}


// HashMap 要求长度是 2 的幂, 这个方法目的是返回大于等于 cap 的最小 2 的幂
// cap=0, 结果返回 1
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

---
#### 假设不减一
`int n = cap - 1` 目的是防止 cap 开始值就为 2 的幂, 结果就返回 cap 的 2 倍了, 实际应该返回 cap 才对,比如 cap = 0000 0001 0000 0000, 不执行减一就操作下面右移:
**第一次右移**
`n |= n >>> 1;`
n = 0000 0001 1000 0000
**第二次右移**
`n |= n >>> 2;`
n = 0000 0001 1110 0000
**第三次右移**
`n |= n >>> 4;`
n = 0000 0001 1111 1110
**第四次右移**
`n |= n >>> 8;`
n = 0000 0001 1111 1111
**第五次右移**
`n |= n >>> 16;`
n = 0000 0001 1111 1111

**最后返回**
n = n + 1 = 0000 0010 0000 0000 = 2 * cap

---
#### 对于非 2 次幂
对于其他大小, 可以假设执行减一后是: 001x xxxx xxxx xxxx, 那么一系列右移后结果变成: 0011 1111 1111 1111, 然后执行加一操作就变成: 0100 0000 0000 0000 0000, 这个就是大于 cap 的最小 2 次幂了.

---
#### tableSizeFor 返回
关于这个方法的返回直接赋值给 threshold, 而不是乘以加载因子, 这个可以去看 resize 方法, 因为这里开始并没有初始化数组, 所以数组还是空, 当新增元素时就会调用 resize 方法, 里面会把 threshold 作为新的长度大小来初始化数组, 同时`长度 * 加载因子`会赋值给 threshold, 关于 resize 方法具体可以看下一 part.

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---