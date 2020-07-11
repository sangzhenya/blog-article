---
title: "Java HashMap"
tags: ["java", "集合"]
categories: ["Java"]
date: "2019-01-04T09:00:00+08:00"
---

HashMap 是一个用于存储 Key-Value 键值对的集合，每个键值对也叫做 Node，这些键值对分散存储在一个数组中，这个数组就是 HaspMap 的骨干。HashMap 默认大小是 16，每次自动扩展或者手动初始的时候长度都是 2 的幂。HashMap 不是线程的安全的，因为在多线程的情况下可能出现循环链表的情况。

HashMap 有四种初始化方式分别是：

```java
// 无参构造函数，默认的容量和默认装载因子
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

// 参数为是初始化容量（生成 Table 的时候会使用大于等于该值最小的 2 的幂值），使用默认的装载因子
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 参数为初始化容量和装载因子
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
    this.threshold = tableSizeFor(initialCapacity);
}
/* 以上三种初始化符串均不会在初始化的时候创建存储数据的 table 而是在第一次 put 的是时候初始化*/

// 使用已经一个存在的 Map 初始化，把原始 Map 中的对象逐个放入到新的 Map 中
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

 HaspMap 的 put 的方法调用的过程如下：

```java
public V put(K key, V value) {
    // 首先调用 hash 函数得到 key 计算后的 hash 值。然后调用 putval 方法存放 value
    return putVal(hash(key), key, value, false, true);
}
```

hash 方法如下（这么做的原因可以参考 [JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617/answer/111577937) ）：

```java
// 返回 key 的 hashCode 与 hashCode 无符号右移 16 位值异或操作的结果
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

putVal 方法如下：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 判断当前存储数据的 table 是否为空，如果为空先初始化 table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // n 是 table 的长度，是2的幂，其减 1 的值与 hash 值进行与操作，相当于一个低位掩码。得到的结果即是该 value 对应的下标。如果该位置的值为空，则新建一个 Node 放入到 table 中
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果该下标对应的值不为空，继续处理
    else {
        Node<K,V> e; K k;
        // 判断当前存储的值对应的 key 和 要插入的 key 是否相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 将要修改的 Node 设置为找到的 Node
            e = p;
        // 不相等则判断当前节点是否是 TreeNode，即当前位置是否是由 红黑树构建的
        else if (p instanceof TreeNode)
            // 调用树的方法找到需要修改的Node，找不到就加入到树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 当前节点是由链表构建
        else {
            // 遍历链表找到对应的 Node，找不到就加入到链表的末尾
            for (int binCount = 0; ; ++binCount) {
                // 如果到链表的时候仍然没有找到相等的则将需要插入的元素放到链表末尾。
                if ((e = p.next) == null) {
                    // 将需要插入的元素放到链表末尾
                    p.next = newNode(hash, key, value, null);
                    // 判断当前节点对应的链表中数据量是否大于等于8[TREEIFY_THRESHOLD]. 如果是则将链表转换为树来存储
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 判断链表中的当前节点是否等于需要插入的数据
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果找到了对应的 Node，则修改 对应 Node Value 将 Old Value 返回。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 修改记录修改数量的 modCount 加一，为了 Fail-Fast 机制
    // 这个标志一是为了多线程时候，如果其他线程修改了当前 map 那么将抛出 ConcurrentModificationException
    // 二是通过迭代器遍历的时候，使用map本身方法修改数据的时候抛出 ConcurrentModificationException
    ++modCount;
    // 如果插入的值之后 table 的 size 大于阈值，则进行 resize， 对 table 扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

resize 方法如下：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 如果大于0 即表示是扩容
    if (oldCap > 0) {
        // 旧 table 的容量大于最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 阈值设置为最大值
            threshold = Integer.MAX_VALUE;
            // 无法扩容返回当前的 table
            return oldTab;
        }
        // 设置新容量为旧容量的 2， 且新容量小于最大值，并且旧容量大于默认值
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 将新的阈值设置为旧阈值的 2 倍
            newThr = oldThr << 1; // double threshold
    }
    // 如果旧的阈值大于0 表名设置了初始容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 设置新的容量为旧的阈值
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 否则使用默认的容量和阈值
        newCap = DEFAULT_INITIAL_CAPACITY; // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 16*0.75 = 12
    }
    // 针对于设置了初始容量初始化的情况。设置新的阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        // 如果初始化容量小于最大容量且计算的阈值小于最大阈值。否则设置为最大值
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 更新阈值属性
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 旧 table 不为空则需要搬移数据
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
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
    // 返回新的 table
    return newTab;
}
```







参考文章：

[什么是HashMap](https://zhuanlan.zhihu.com/p/31610616)

[高并发下的HashMap](https://zhuanlan.zhihu.com/p/31614195?from_voters_page=true)

[Java7/8中的HashMap和ConcurrentHashMap全解析](https://juejin.im/post/5c933cdb6fb9a070b45dcdda#heading-17)

