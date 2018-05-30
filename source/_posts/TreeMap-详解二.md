---
title: TreeMap 详解二
date: 2018-04-25 15:32:02
tags: [TreeMap]
---

TreeMap 插入 key-value 同样是用 put 方法
#### put
```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    // 对于空树, 插入新节点作为根节点
    // 红黑树定义根节点必须是黑色, 这里构造新节点默认就是黑色了, 不需要修改颜色
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;

    // 如果构造 TreeMap 时传入了自定义的比较器, 那么使用该比较器来遍历树
    // 小于 0 向左遍历, 大于 0 向右遍历, 等于 0 直接覆盖值
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    // 如果没有传入自定义的比较器, 那么会使用 key 的 compareTo 方法比较
    // 所以这里就要求 key 不能为空了
    // 同样根据小于, 等于, 大于 0 来决定遍历方向
    // 同时使用 parent 变量保存新节点的父节点
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    // 构造新节点, 根据情况保存到父节点的左右子树
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 这里就是重新调整树, 使之符合红黑树的性质, 下面具体来看
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

---

#### fixAfterInsertion
这里我们把这个方法的代码拆分来看, 至于红黑树的调整原理可以看 *HashMap-详解五*, 这里不在赘述.
1. 插入的新节点要是红色, 所以要先修改节点颜色
```java
x.color = RED;
```
2. 只有红父才需要调整, 所以节点不会是根节点, 循环结构如下:
```java
while (x != null && x != root && x.parent.color == RED) {
	...
}
```
3. 对于新节点是位于祖父节点的左子树, 叔叔节点是红色, 那么只要改变颜色即可, 同时当前节点指向祖父节点, 循环向上遍历
```java
if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
    Entry<K,V> y = rightOf(parentOf(parentOf(x)));
    if (colorOf(y) == RED) {
        setColor(parentOf(x), BLACK);
        setColor(y, BLACK);
        setColor(parentOf(parentOf(x)), RED);
        x = parentOf(parentOf(x));
    }
}
```
4. 对于新节点是位于祖父节点的左子树, 叔叔节点是黑色, 如果新节点位于父节点的右子树, 先左旋, 然后统一右旋, 当然也要改变节点颜色
```java
if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
    Entry<K,V> y = rightOf(parentOf(parentOf(x)));
    if (colorOf(y) == RED) {
        ...
    } else {
    	// 看这里
    	// 左旋之后, 当前节点是原来的父节点
    	// 这样就可以相当于插入一个新的左节点, 跟下面的右旋完美结合
        if (x == rightOf(parentOf(x))) {
            x = parentOf(x);
            rotateLeft(x);
        }
        setColor(parentOf(x), BLACK);
        setColor(parentOf(parentOf(x)), RED);
        rotateRight(parentOf(parentOf(x)));
    }
}
```
5. 对于新节点是位于祖父节点的右子树, 叔叔节点是红色, 直接修改颜色即可
```java
Entry<K,V> y = leftOf(parentOf(parentOf(x)));
if (colorOf(y) == RED) {
    setColor(parentOf(x), BLACK);
    setColor(y, BLACK);
    setColor(parentOf(parentOf(x)), RED);
    x = parentOf(parentOf(x));
}
```
6. 对于新节点是位于祖父节点的右子树, 叔叔节点是黑色, 如果新节点位于父节点的左子树, 先右旋, 然后统一左旋, 同样要修改颜色
```java
Entry<K,V> y = leftOf(parentOf(parentOf(x)));
if (colorOf(y) == RED) {
    ...
} else {
    if (x == leftOf(parentOf(x))) {
        x = parentOf(x);
        rotateRight(x);
    }
    setColor(parentOf(x), BLACK);
    setColor(parentOf(parentOf(x)), RED);
    rotateLeft(parentOf(parentOf(x)));
}
```
完整代码如下:
```java
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED;

    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
    
```

---

左旋和右旋实际是修改节点的左右子树和父节点, 可以通过自己画图来更好理解节点的交换过程
#### rotateLeft
```java
// 左旋
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> r = p.right;
        p.right = r.left;
        if (r.left != null)
            r.left.parent = p;
        r.parent = p.parent;
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}
```

#### rotateRight
```java
// 右旋
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---