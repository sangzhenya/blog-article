## Java ConcurrentHashMap

### JDK 8 中 ConcurrentHashMap 为什么放弃分段锁

在 Java 8 中 `ConcurrentHashMap` 放弃了分段锁而是使用 `synchronized` 加 CAS 的来实现并发控制的。一方面是 JVM 对 `synchronized` 已经进行了大量的优化措施，例如锁粗化，锁消除，自适应自旋锁，偏向锁等等。另一方面使用分段锁浪费内存空间，且 map 放入时竞争同一个锁的概率很小，分段锁反而可能造成更新操作长时间等待，同时为了提高 GC 的效率。

### HashMap 并发情况下可能出现的 死循环

在 JDK 1.7 版本中并发的情况下使用 HashMap 可能会出现死循环的情况，主要的是在 HashMap 扩容之后会通过 `transfer` 方法，把元素从老链表中转移到新的链表中。

```java
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

`resize` 中出现的死锁的过程如下，如果一个 table 中某个 key 对应如下的链表：

```java
key ==> A -> B -> C
```

正常的情况 `resize` 之后应该是

```java
key ==> B -> A
key1 ==> C
```

多线程的时候 resize 过程可能如下（请结合上面 transfer 方法看）：

第一步：

开始前链表状态

```java
A -> B -> C 
```

线程1：

```java
e = A
next = B
---
newTable[key] -> null
```

线程2：

```java
e = A
next = B
---
newTable[key] -> null
```

第二步：

开始前链表状态

```java
A -> B -> C 
```

线程 1 （未被执行）

```java
e = A
next = B
---
newTable[key] -> null
```

线程 2

```java
e = B
next = C
---
newTable[key] -> A
```

第三步：

开始前链表状态

```java
A -> null
B -> C -> null
```

线程 1  （未被执行）

```java
e = A
next = B
---
newTable[key] -> null
```

线程 2

```java
e = C
next = null
---
newTable[key] -> B -> A
```

第四步：

开始前链表状态

```java
B -> A -> null
C -> null
```

线程 1 

```java
e = B
next = A
---
newTable[key] -> A
```

线程 2 （未被执行）

```java
e = C
next = null
---
newTable[key] -> B -> A
```

第五步：

开始前链表状态

```java
B -> A -> null
C -> null
```

线程 1

```java
e = A
next = null
---
newTable[key] -> B -> A
```

线程 2 (未被执行)

```java
e = C
next = null
---
newTable[key] -> B -> A
```

第六步：

开始前链表状态

```java
B -> A -> null
A -> null
```

线程 1

```java
e = null
next = null
---
newTable[key] -> B <-> A
```

线程 2 (未被执行)

```java
e = C
next = null
---
newTable[key] -> B -> A
```

结束的链表状态

```java
B <-> A 
C -> null
```

这个时候就发现在 key 的位置出现了循环链表了。

在 JDK 8 的 HashMap 源码中的已经找不到了 `transfer` 方法了，因为 JDK 8 中对于 HashMap 的实现有了较大的改动，对于相同 Hash 值数据存储的方式也由以前的单纯的使用链表转变成了，在数量较少的时候使用链表，当链表长度大于 8 且总 HashMap 的长度大于等于 128 的时候会将链表转换成红黑树进行存储。链表的插入方式也从以前的头插法换成了尾插法（这一点就解决了上述可能出现循环链表的问题）。但是这并不意味着 JDK 8 中的 HashMap 是线程的安全的，在并发环境下 HashMap 还是可能出现缺失数据的问题。所以在并发环境下还是需要使用 ConcurrentHashMap 来保证数据的线程安全。

### 初始化 Table



### Put 操作



### Table 的扩容



### Get 操作



参考：

1. [HashMap 多线程下死循环分析及JDK8修复](http://www.voidcn.com/article/p-hcuguoxo-bob.html)