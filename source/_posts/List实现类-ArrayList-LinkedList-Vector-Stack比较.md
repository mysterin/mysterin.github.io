---
title: 'List实现类 ArrayList,LinkedList,Vector,Stack比较'
date: 2018-05-15 14:46:58
tags: [List, ArrayList, LinkedList, Vector, Stack]
---

*本文代码来自JDK8*

List 继承于 Collection 继承于 Iterable, 常用实现类有 ArrayList, LinkedList, Vector, Stack, 它们各有不同, 下面会从各方面进行比较.

#### 保存元素方式
##### ArrayList
```java
transient Object[] elementData;
```
使用一个 Object 的数组保存, 所以可以通过索引直接访问.
##### LinkedList
```java
transient Node<E> first;
transient Node<E> last;
```
因为是链表结构, 所以用头结点, 尾节点表示链表.
##### Vector
```java
protected Object[] elementData;
```
使用数组方式保存.
##### Stack
Stack 是继承 Vector, 所以同样是用数组方式保存.

---

#### 初始化
##### ArrayList
三个构造函数, 一个默认空数组, 一个指定大小数组, 一个根据其他集合初始化.
```java
/**
 * 默认初始化成一个空数组
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * 当然也可以指定初始化数组大小
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

/**
 * 通过其他集合实例初始化也是允许的
 * 实际就是读取集合元素复制到数组中
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
##### LinkedList
两个构造函数, 一个什么不做, 另一个根据提供的集合实例初始化.
```java
/**
 * 默认构造什么也没做, 毕竟是链表结构, 一开始头尾节点都是空的
 */
public LinkedList() {
}

/**
 * 通过其他集合实例初始化
 * 实际就是读取集合元素插入到链表中
 */
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
##### Vector
四个构造函数, 一个默认数组大小, 一个指定数组大小, 一个指定数组大小和扩容大小, 最后一个同样是根据集合初始化.
```java
/**
 * capacityIncrement 是每次增长的长度大小, 默认是 0
 */
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}

public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}

/**
 * 默认初始化长度为 10 的数组
 */
public Vector() {
    this(10);
}

public Vector(Collection<? extends E> c) {
    elementData = c.toArray();
    elementCount = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
}
```
##### Stack
只有一个构造函数, 什么不做, 还是要调用父类无参构造函数来初始化.
```java
/**
 * 只有一个默认构造函数, 主要还是通过父类 Vector 初始化
 */
public Stack() {
}
```

---

#### 扩容
##### ArrayList
每次增长原来的 1.5 倍.
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
##### LinkedList
因为是链表结构, 没有扩容一说.
##### Vector
如果指定了 capacityIncrement, 那么就按这个大小扩容; 如果没有指定, 那么扩容为原来的 2 倍.
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
##### Stack
因为继承于 Vector, 所以扩容是用父类方法扩容.

---

#### 添加元素
##### ArrayList
第一步判断是否要扩容, 第二步直接把元素插入数组即可, 很简单.
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
##### LinkedList
第一步创建节点, 第二步把节点插入到链表末尾, 也很简单. 值得注意的是链表是双向链表.
```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```
##### Vector
插入方式和 ArrayList 一样, 不同之处在于这是个线程安全的方法, 这也是 ArrayList 和 Vector 的不同.
```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```
##### Stack
Stack 是一个栈, 所以添加元素是调用 push 方法, 元素插入到数组后面
```java
public E push(E item) {
    addElement(item);
    return item;
}
```

---

#### 读取元素
##### ArrayList
因为是数组, 所以可以直接根据索引读取元素, 当然遍历方式也是这样读取.
##### LinkedList
因为是链表, 所以遍历方式需要从头开始遍历, 当然它也支持通过索引值读取, 看下面代码:
```java
/**
 * 这里用了一个巧妙方式读取索引位置的值
 * 判断索引是在链表的前一半还是后一半
 * 如果是前面就从头节点开始向后遍历, 直到索引位置
 * 如果是后一半就从尾节点开始向前遍历, 直到索引位置
 */
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
##### Vector
同样是数组读取方式, 当然是线程安全的.
##### Stack
栈的读取方式是出栈 pop, 读取数组最后一个元素, 同时把这个元素删除, 删除方式就是索引位置置为空, 数组长度减一, 让 gc 回收这个对象即可.
```java
public synchronized E pop() {
    E       obj;
    int     len = size();

    obj = peek();
    removeElementAt(len - 1);

    return obj;
}
```

---
**如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨**

---