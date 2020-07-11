---
title: "Java ConcurrentHashMap"
tags: ["java", "JUC", "集合"]
categories: ["Java"]
date: "2019-01-02T10:00:00+08:00"
# toc: true
---

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

在  JDK 8 中对于 HashMap 的实现有了较大的改动，对于相同 Hash 值数据存储的方式也由以前的单纯的使用链表转变成了，在数量较少的时候使用链表，当链表长度大于 8 且总 HashMap 的长度大于等于 64 的时候会将链表转换成红黑树进行存储。链表的插入方式也从以前的头插法换成了尾插法（这一点就解决了上述可能出现循环链表的问题）。但是这并不意味着 JDK 8 中的 HashMap 是线程的安全的，在并发环境下 HashMap 还是可能出现缺失数据的问题。所以在并发环境下还是需要使用 ConcurrentHashMap 来保证数据的线程安全。

### 主要成员变量

主要成员变量如下

```java
// 存储元素的列表，在第一次插入的时候进行初始化，容量是 2 的幂
transient volatile Node<K,V>[] table;
// 扩容的时候使用到的
private transient volatile Node<K,V>[] nextTable;
// 统计 Map 中存储数据的数量
private transient volatile long baseCount;
// 小于 0 表示 table 正在初始化或者扩容，-1 表示正常在初始化，-(1 + 扩容线程数) 表示正在扩容。
// 当 table 为 null 时为初始化时创立的 table 长度，默认值是 0、
// 初始化后记录下一个元素的值，根据这个值扩容数组，就是阈值
private transient volatile int sizeCtl;
// 1 表示 counterCells 正常创建， 0 表示其他状态
private transient volatile int cellsBusy;
// 递增 baseCount 出现竞争的时候根据 counterCells 中的值进行递增，最终 Map 中的数量是 baseCount 和该数组之和
private transient volatile CounterCell[] counterCells;
```



### Put 操作

测试方法如下：

```java
@Test
public void testConcurrentHashMap() {
    Map<String, String> testConcurrentHashMap = new ConcurrentHashMap<>();
    testConcurrentHashMap.put("Aa", "demo");
    testConcurrentHashMap.put("BB", "demo");
    System.out.println(testConcurrentHashMap);
}
```

先调用 ConcurrentHashMap 的 put 方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

继续调用 putVal 方法真的去存放数据

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 如果 key 或者 value 任意一个为空则抛出异常
    if (key == null || value == null) throw new NullPointerException();
    // 计算 hash 值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果 tab 为空则初始化 tab
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // (n - 1) & hash 计算实际应该存储的位置，tabAt 获取当前位置的元素，如果为空则直接使用 CAS 设置值即可
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果获取到元素的 hash 值为 MOVED 则当前线程协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 哈希值冲突，把当前元素存放到链表或者红黑树中
        else {
            V oldVal = null;
            // 同步锁
            synchronized (f) {
                // 再次获取 i 位置的元素，判断是否等于刚才获取的元素，
                // 如果是则继续执行操作，否则重新走一遍 put 的流程
                if (tabAt(tab, i) == f) {
                    // 存在元素的 hash 值大于 0
                    if (fh >= 0) {
                        // 记录链表长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果 key 值相等则更新value即可
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 插入到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 或者 f 是一个 红黑树元素
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        // 向红黑树中添加元素
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 如果元素 Count 不等于0 则表明已经放入了，可以终止循环了
            if (binCount != 0) {
                // 对于链表如果元素数来大于 阈值 64 则尝试转换链表为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 如果是覆盖则返回旧值
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

// 获取转换后的 Hash 值
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}

// 获取 i 位置的元素
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

// CAS 设置 i 位置元素的值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

```java
// 向 红黑树中 添加元素 
final TreeNode<K,V> putTreeVal(int h, K k, V v) {
     Class<?> kc = null;
     boolean searched = false;
     for (TreeNode<K,V> p = root;;) {
         int dir, ph; K pk;
         if (p == null) {
             first = root = new TreeNode<K,V>(h, k, v, null, null);
             break;
         }
         else if ((ph = p.hash) > h)
             dir = -1;
         else if (ph < h)
             dir = 1;
         else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
             return p;
         else if ((kc == null &&
                   (kc = comparableClassFor(k)) == null) ||
                  (dir = compareComparables(kc, k, pk)) == 0) {
             if (!searched) {
                 TreeNode<K,V> q, ch;
                 searched = true;
                 if (((ch = p.left) != null &&
                      (q = ch.findTreeNode(h, k, kc)) != null) ||
                     ((ch = p.right) != null &&
                      (q = ch.findTreeNode(h, k, kc)) != null))
                     return q;
             }
             dir = tieBreakOrder(k, pk);
         }

         TreeNode<K,V> xp = p;
         if ((p = (dir <= 0) ? p.left : p.right) == null) {
             TreeNode<K,V> x, f = first;
             first = x = new TreeNode<K,V>(h, k, v, f, xp);
             if (f != null)
                 f.prev = x;
             if (dir <= 0)
                 xp.left = x;
             else
                 xp.right = x;
             if (!xp.red)
                 x.red = true;
             else {
                 lockRoot();
                 try {
                     root = balanceInsertion(root, x);
                 } finally {
                     unlockRoot();
                 }
             }
             break;
         }
     }
     assert checkInvariants(root);
     return null;
 }
```

```java
// 尝试链表转红黑树  
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    // 链表不为空
    if (tab != null) {
        // 判断 table 的长度是否小于 64 ，如果小于则进行扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        // 如果 index 位置的元素不为空且 hash 大于 0
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 同步锁
            synchronized (b) {
                // 判断 index 位置的元素是否等于 刚才获取的元素
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 将 Node 转换为 TreeNode
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 替换原本 index 位置为 TreeBin 元素，在 TreeBin 的构造方法中
                    // 完成了链表到红黑树的转换
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

```java
// 修改当前存储值的数量，并检测是否需要扩容
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 更新 BaseCount
    // counterCells 数组中有可用的计数器，去尝试给计数器增加 x
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果 CAS 更新 baseCount 失败则进入 fullAddCount
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        // 计算新的 baseCount 的值
        s = sumCount();
    }
    // 如果 check 大于 0 则检测是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 大于阈值且 table 不为空的时候进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

```java
// addCount 方法在 CAS 操作失败之后进入该方法。
// 在多个统计变量上做递增
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```



### 初始化 Table

ConcurrentHashMap 的初始化也是在 put 一个元素时进行的，而不是在构造函数中进行初始化的。初始化的主要流程如下：

```java
// 初始化 Table
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // Table 为空的时需要初始化
    while ((tab = table) == null || tab.length == 0) {
        // 小于 0 表示正在初始化，所以需要当前线程放弃 CPU
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 设置初始化标记
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 再次判断 table 是否为空
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



### Table 的扩容

```java
// 移动或者复制元素到新的 table
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 计算每个 CPU 需要处理的 Node 数量，最低为 16 个
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //如果新的 Table 为空则新创建一个两倍大的  table 
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    // 获取新 Table 的长度
    int nextn = nextTab.length;
    // 创建 ForwardingNode 作为标志节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // hash 桶操作完成的标志
    boolean advance = true;
   // 扩容完成的标志
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            // nextIndex: 最后一个元素
            // nextBound: table 长度减去 桶的大小 stride，最小是 16
            int nextIndex, nextBound;
            // 如果再当前桶中，直接下一个
            if (--i >= bound || finishing)
                advance = false;
            // 第一次 transferIndex 是原始 table 的长度
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 设置 bound = next - stride 最小为 0，就是当前处理的区间
                bound = nextBound;
                // 应该处理的第一个元素
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 下面是结束的条件
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 如果已经结束则删除 nextTable，并把nextTable 放到 table ，设置新的阈值
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果为空则直接设置为空
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果当前元素的 hash 为 MOVED 则表示当前元素已经被处理过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 同步锁处理非空的坐标
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 获取新的下标
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 如果下标为 0 设置低位为 需要处理的元素
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        // 否则设置高位为需要处理的元素
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 复制元素，分别到高位和低位两个链表中
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 分别设置低位和高位的元素
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 设置当前坐标已经处理过了
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        // 对于树的处理方式
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

```java
// 如果一个线程发现正在扩容，在参与到扩容之中
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 取出 nextTable 进行初始化
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // 确保还没有变化
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 参与到扩容之中
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```



### Get 操作

```java
// 获取元素
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 获取 hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            // 如果找到直接返回
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果小于 0 则从树上找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 如果没有找到表明可能是链表，则遍历找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



参考：

1. [HashMap 多线程下死循环分析及JDK8修复](http://www.voidcn.com/article/p-hcuguoxo-bob.html)
2. [http://footmanff.com/2018/03/13/2018-03-13-ConcurrentHashMap-1/](http://footmanff.com/2018/03/13/2018-03-13-ConcurrentHashMap-1/)
3. [为并发而生的 ConcurrentHashMap（Java 8）](https://www.cnblogs.com/yangming1996/p/8031199.html)