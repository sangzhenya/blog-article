## 集合类 -- Map

### HashMap

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

// 使用已经一个存在的 Map 初始化，把原始 Map 中的对象逐个放入到新的 Map 中
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    // 获取需要新增键值对的大小
    int s = m.size();
    // 大于的 0 的时候才添加
    if (s > 0) {
        // 如果用于存储数据的 table 为空，则需要初始化
        if (table == null) { // pre-size
            // 获取需要新增的 长度的 4/3 倍 + 1 的值
            float ft = ((float)s / loadFactor) + 1.0F;
            // 如果该值小于最大容量，设置该值为容量，否则设置最大值为容量。
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            // 如果得到的值大于阈值，则设置用于存储数据的 table
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            // 如果 数量大于了阈值，则需要 resize 容量
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            // 将值放入到 table 中
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}

// 获取大于传入值的2的最小的指数值
// Returns a power of two size for the given target capacity.
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

public V put(K key, V value) {
    // 首先调用 hash 函数得到 key 计算后的 hash 值。然后调用 putval 方法存放 value
    return putVal(hash(key), key, value, false, true);
}

// 返回 key 的 hashCode 与 hashCode 无符号右移 16 位值异或操作的结果
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K, V>[] tab;
    Node<K, V> p;
    int n, i;
    // 判断当前存储数据的 table 是否为空，如果为空先初始化 table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // n 是 table 的长度，是2的幂，其减 1 的值与 hash 值进行与操作，相当于一个低位掩码。得到的结果即是该 value
    // 对应的下标。如果该位置的值为空，则新建一个 Node 放入到 table 中
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果该下标对应的值不为空，继续处理
    else {
        Node<K, V> e;
        K k;
        // 判断当前存储的值对应的 key 和 要插入的 key 是否相等
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            // 将要修改的 Node 设置为找到的 Node
            e = p;
        // 不相等则判断当前节点是否是 TreeNode，即当前位置是否是由 红黑树构建的
        else if (p instanceof TreeNode)
            // 调用树的方法找到需要修改的Node，找不到就加入到树中
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
        // 当前节点是由链表构建
        else {
            // 遍历链表找到对应的 Node，找不到就加入到链表的末尾
            for (int binCount = 0;; ++binCount) {
                // 如果到链表的时候仍然没有找到相等的则将需要插入的元素放到链表末尾。
                if ((e = p.next) == null) {
                    // 将需要插入的元素放到链表末尾
                    p.next = newNode(hash, key, value, null);
                    // 判断当前节点对应的链表中数据量是否大于等于8[TREEIFY_THRESHOLD]. 
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 判断链表中的当前节点是否等于需要插入的数据
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
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


final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果 当前的 长度小于 64 则扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 使用 TreeNode 代替当前的 Node
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 构建树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}


// Resize table
final Node<K, V>[] resize() {
    // 获取当前存储数据的 table
    Node<K, V>[] oldTab = table;
    // 存储数据的 table 的大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 阈值
    int oldThr = threshold;
    // 初始化新的阈值和容量
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
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 将新的阈值设置为旧阈值的 2 倍
            newThr = oldThr << 1; // double threshold
    }
    // 如果旧的阈值大于0 表明设置了初始容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 设置新的容量为旧的阈值
        newCap = oldThr;
    else { // zero initial threshold signifies using defaults
        // 否则使用默认的容量和阈值
        newCap = DEFAULT_INITIAL_CAPACITY; // 16
        newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 16*0.75 = 12
    }
    // 针对于设置了初始容量初始化的情况。设置新的阈值
    if (newThr == 0) {
        float ft = (float) newCap * loadFactor;
        // 如果初始化容量小于最大容量且计算的阈值小于最大阈值。否则设置为最大值
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ? (int) ft : Integer.MAX_VALUE);
    }
    // 更新阈值属性
    threshold = newThr;
    @SuppressWarnings({ "rawtypes", "unchecked" })
    Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
    table = newTab;
    // 旧 table 不为空则需要搬移数据
    if (oldTab != null) {
        // 遍历原始用于存储数据的 table 数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K, V> e;
            // 如果当前节点不为空才搬移
            if ((e = oldTab[j]) != null) {
                // 原始值设置为空
                oldTab[j] = null;
                // 如果 元素值 的 next 为空。表明没有 hash 值相同的元素在该处
                if (e.next == null)
                    // 为元素值计算新的下标，并设置值为当前元素
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 调用红黑树的搬移方法
                    ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K, V> loHead = null, loTail = null;
                    Node<K, V> hiHead = null, hiTail = null;
                    Node<K, V> next;
                    do {
                        // 设置 next 为当前元素的下一个元素
                        next = e.next;
                        // oldCap = 2^n 次方 oldCap - 1 则是 2^n - 1
                        // 例如原始容量是 16，扩容后是 32
                        // 2^4(16)  10000   ;;  2^4 - 1(15)  1111
                        // 2^5(32)  100000  ;;  2^5 - 1(31)  11111
                        // 元素和容量的与的值只可能是 0 或 1，起作用的是第一位1
                        // 如果结果是 0 则表明扩容后原值不变，放到一个链表中
                        // 如果结果是 1 则表明原值会改变，放到另外一个链表中
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 存放下标不变的链表
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 存放下标修改后的链表
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 例如原始 hash 值是 17
                        // 则原始下标是hash值与 15 与运算之后的值 1
                        // 新的下标值是hash值与 31 与运算之后的值 17
                        // 新的下标值就是原始值加上原本的容量
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 返回新的 table
    return newTab;
}

public V get(Object key) {
    Node<K,V> e;
    // 调用内部方法获取
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    // 声明需要使用到的变量
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 如果存储数据的 table 不为空 并且根据hash值去计算下标对应的值不是 null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 如果第一个就是则直接返回第一个
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 否则逐个遍历得到对应的值
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

public boolean containsKey(Object key) {
    // 根据hash值获取返回获取的value是否为空
    return getNode(hash(key), key) != null;
}

public V remove(Object key) {
    Node<K,V> e;
    // 调用方法 remove node，如果remove 的值不为空，则返回 value
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // table 不为空，且根据hash值计算得到下标位置的值不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 第一个即为 需要找的
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 第一个不是则遍历获取找到需要的
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }

        // 如果找到了值
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 如果是 树，调用树的 remove node 的方法
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果不是树则表明是链表，如果是第一位则直接将值放到table对应的位置
            else if (node == p)
                tab[index] = node.next;
            // 否则需要修改链表的指针
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```



