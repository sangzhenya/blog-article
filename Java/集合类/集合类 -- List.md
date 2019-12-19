## 集合类 --- List

### ArrayList

```java
// 用于存储数据的数组
transient Object[] elementData; 
// 默认数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
// 当前已经存储数据的数量，默认值是 0 
private int size;

// 无参构造函数
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 初始化容量的构造函数
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        // 如果初始化参数大于 0 则使用初始容量创建一个 Object 的数组放到 elmenetData 中
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // 如果初始化参数等于0 ，则使用默认的空数组初始化 elmentData 中，和无参构造器一样
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        // 其他情况则直接报错
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

public boolean add(E e) {
    // 添加元素前需要先确保容量足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 将当前的值放到最后一个数据的后一位
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    // 首先要计算容量
    // 然后去确认容量足够
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 如果用于存储的 elmenetData  和默认的 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 数组一样
    // 表明目前 elementData 还是空数组，然计算需要增加的量，计算出大的值，作为容量返回
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    // 如果用于存储的 elementData 不是空数组，那么证明已经初始化过，则直接返回容量的值
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    // 修改的值 +1 主要用于 fast-faile 
    modCount++;

    // 如果需要的容量 大于目前可用的容量，则增加数组的容量，否则什么都不做。
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // 获取目前能够存储最大的容量
    int oldCapacity = elementData.length;
    // 获取新的容量 等于 3/2 旧的容量
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        // 如果新的容量小于需要的容量，则另新的容量等于最小的容量
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        // 如果新的容量大于 最大的容量值 Integer.MAX_VALUE - 8，则通过函数获取新的容量
        newCapacity = hugeCapacity(minCapacity);
    // Copy 当前数组的中数据到一个新容量的数组，并赋值给 elementDta
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    // 如果需要的容量小于0 则抛错
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 否则如果 需要的容量大于 最大的数组容量，则返回 int 的最大值
    // 否则返回 最大的数组容量
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
    MAX_ARRAY_SIZE;
}


public void add(int index, E element) {
    // 首先确保 index 在 0 和 容量之间，否则之间抛错
    rangeCheckForAdd(index);

    //确保数组的容量足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // Copy 数组
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 将新的值放到指定的位置
    elementData[index] = element;
    size++;
}

// index 边界检查
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}


public boolean addAll(Collection<? extends E> c) {
    // 将 Collection 转换为数组
    Object[] a = c.toArray();
    // 获取需要新增的数量
    int numNew = a.length;
    // 确保数组容量足够
    ensureCapacityInternal(size + numNew);  // Increments modCount
    // Copy 元素
    System.arraycopy(a, 0, elementData, size, numNew);
    // 当前存储的量 + 加上新增的量，得到最新的存储数据的量
    size += numNew;
    return numNew != 0;
}

public boolean addAll(int index, Collection<? extends E> c) {
    // 检查 index 是否在 0 到 size 之间
    rangeCheckForAdd(index);

    // Collection 转换为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    // 获取开始移动的位置
    int numMoved = size - index;
    // 大于 0 才去 Copy 数组
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    // Copy 值到新的新的数组
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}

public E get(int index) {
    // 确保 index 在 0 到 size 之间
    rangeCheck(index);
    // 返回该位置的值
    return elementData(index);
}

E elementData(int index) {
    // 直接从数组中返回值
    return (E) elementData[index];
}

public E set(int index, E element) {
    // 确保 index 在 0 到 size 之间
    rangeCheck(index);

    // 获取旧的值
    E oldValue = elementData(index);
    // 将对应位置的值设置为新值
    elementData[index] = element;
    // 返回旧值
    return oldValue;
}

public E remove(int index) {
    // 确保 index 在 0 到 size 之间
    rangeCheck(index);
    // 修改次数 +1, 用于 fast-fail
    modCount++;
    // 通过下标获取旧的值
    E oldValue = elementData(index);

    // 移动开始的位置
    int numMoved = size - index - 1;
    // 需要移动的位置大于 0 的话，Copy 数组。
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 将最后一位设置空，并将当前存储的数量减一
    elementData[--size] = null; // clear to let GC do its work

    // 返回旧值
    return oldValue;
}

public boolean remove(Object o) {
    // 如果 删除的元素是 null 
    if (o == null) {
        // 遍历数组找到第一个是 null 的元素，执行删除操作
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        // 如果删除的元素不是空，则逐个比较元素，找到相同那个删除
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    // 操作次数 +1, 用于 fast-fail
    modCount++;
    // 得到需要移动的数量
    int numMoved = size - index - 1;
    // 如果需要移动的数量大于 0，则移动数组
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 将数组的最后一位清空，并设置当前存储数量减一
    elementData[--size] = null; // clear to let GC do its work
}

 public boolean removeAll(Collection<?> c) {
     Objects.requireNonNull(c);
     return batchRemove(c, false);
 }

private boolean batchRemove(Collection<?> c, boolean complement) {
    //  获取当前的元素数组
    final Object[] elementData = this.elementData;
    // 初始化值用于遍历
    int r = 0, w = 0;
    // 用来标记是否修改了
    boolean modified = false;
    try {
        // 遍历当前的数组
        for (; r < size; r++)
            // complement 的值是 false， 所以意味着，如果需要删除的列表中不包含该值，则 w++ r++
            // 如果包含则 r++ w 保持不变，即可以认为将数组全部前移一位
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        // 如果删除了数据，则将最后一位值之后的值全部重置为空
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            // 设置存储的容量为新的值
            size = w;
            // 设置了修改的标志
            modified = true;
        }
    }
    return modified;
}
```



### Collections.synchronizedList(new ArrayList<>())

```java
// 其实就是对 List 进行一次封装
// 用于存储的 Collection
final Collection<E> c;  // Backing Collection
// 用于锁的类
final Object mutex;     // Object on which to synchronize

SynchronizedCollection(Collection<E> c) {
    // 设置存储的 Collection 为传入的 Collection
    this.c = Objects.requireNonNull(c);
    // 设置锁为当前对象
    mutex = this;
}

// 在原始 Collection 的增删改查上都加锁操作
public E get(int index) {
    synchronized (mutex) {return list.get(index);}
}
public E set(int index, E element) {
    synchronized (mutex) {return list.set(index, element);}
}
public void add(int index, E element) {
    synchronized (mutex) {list.add(index, element);}
}
public E remove(int index) {
    synchronized (mutex) {return list.remove(index);}
}
```



### CopyOnWriteArrayList

```java
private transient volatile Object[] array;
    
public CopyOnWriteArrayList() {
    // 无参构造函数，初始化存储的数组为空数组
    setArray(new Object[0]);
}

public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        // 如果传入的值是 CopyOnWriteArrayList 类型的，则直接获取对应的 array 初始化当前的 array
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        // 否则拿到 collection 的 array 转换为数组
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        // Copy 原始元素到新数组中
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    // 设置数组
    setArray(elements);
}

public CopyOnWriteArrayList(E[] toCopyIn) {
    // Copy 原始数组到新数组中
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

public boolean add(E e) {
    // 获取可重入锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取当前存储数据的数组
        Object[] elements = getArray();
        // 获取数组的长度
        int len = elements.length;
        // Copy 旧数组的数据到新数组中
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 设置新数组的最后一位等于传入的值
        newElements[len] = e;
        // 更新 array
        setArray(newElements);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}

public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 确保 index 在 0 和 size 中间
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        // 计算需要移动的数量
        int numMoved = len - index;
        if (numMoved == 0)
            // 需要移动的数量为0 则，直接 Copy 数组
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            // 否则设置新数组为原始数组的容量 +1 
            newElements = new Object[len + 1];
            // Copy 修改位置前的数组到新数组中
            System.arraycopy(elements, 0, newElements, 0, index);
            // Copy 修改位置后的数组到新数组中
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        // 设置新数组的最后一位是传入的元素
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}

public boolean addAll(Collection<? extends E> c) {
    // 如果是 CopyOnWriteArrayList 类型的 Collection 则直接获取 array,
    // 否则则转换为 array
    Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
        ((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
    // 如果需要新加的元素数量为 0 则返回 false
    if (cs.length == 0)
        return false;
    // 获取可重入锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取当前的数组
        Object[] elements = getArray();
        // 获取数组的长度
        int len = elements.length;
        // 如果数组为空
        if (len == 0 && cs.getClass() == Object[].class)
            // 直接将当前数组设置为需要新加的数组
            setArray(cs);
        else {
            // Copy 原始数组的元素到新数组
            Object[] newElements = Arrays.copyOf(elements, len + cs.length);
            // Copy 新增加的元素到新数组
            System.arraycopy(cs, 0, newElements, len, cs.length);
            // 设置为新的数组
            setArray(newElements);
        }
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}

public boolean addAll(int index, Collection<? extends E> c) {
    // 将新增加的 Collection 转换为数组
    Object[] cs = c.toArray();
    // 获取可重入锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取当前的数组
        Object[] elements = getArray();
        // 获取数组的长度
        int len = elements.length;
        // 确保新增加的位置在 0 到 size 中间
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        // 如果新增加入 0 个元素 则返回 false
        if (cs.length == 0)
            return false;
        // 获取需要移动的数量
        int numMoved = len - index;
        Object[] newElements;
        // 如果需要移动的数量为 0 则直接复制需要新加的元素到新的数组中
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + cs.length);
        else {
            // 复制原有元素到新的数组中
            newElements = new Object[len + cs.length];
            // Copy 原始数组的部分的元素到新数组中
            System.arraycopy(elements, 0, newElements, 0, index);
            // Copy 修改后部分的内容到新数组中
            System.arraycopy(elements, index,
                             newElements, index + cs.length,
                             numMoved);
        }
        // Copy 新添加的元素到数组中
        System.arraycopy(cs, 0, newElements, index, cs.length);
        // 设置新的数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            // 如果原始元素不等于新的元素
            // 获取数组的长度
            int len = elements.length;
            // Copy 原始数组到新数组中
            Object[] newElements = Arrays.copyOf(elements, len);
            // 设置数组的对应位置的元素为新值
            newElements[index] = element;
            // 设置数组
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

public E get(int index) {
    // 直接通过数组下标获取值
    return get(getArray(), index);
}

public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

public boolean remove(Object o) {
    // 获取快照版本
    Object[] snapshot = getArray();
    // 获取对应元素对应的 index
    int index = indexOf(o, snapshot, 0, snapshot.length);
    // 如果找到了则从数组中移除对应位置的元素
    return (index < 0) ? false : remove(o, snapshot, index);
}

private static int indexOf(Object o, Object[] elements,
                           int index, int fence) {
    // 如果元素为 null， 则遍历查找第一个为 null 元素
    if (o == null) {
        for (int i = index; i < fence; i++)
            if (elements[i] == null)
                return i;
    } else {
        // 否则则通过 equals 函数查找第一个元素
        for (int i = index; i < fence; i++)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}

private boolean remove(Object o, Object[] snapshot, int index) {
    // 获取可重入锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取当前的数组
        Object[] current = getArray();
        // 获取当前数组的数量
        int len = current.length;
        // 如果当前快照和原始数组不一致
        if (snapshot != current) findIndex: {
            // 比较获得 index 和 len 小的一个
            int prefix = Math.min(index, len);
            // 遍历 0 到最小的一个值。进行判断
            for (int i = 0; i < prefix; i++) {
                // 如果当前数组中的值和快照中的值不一致 并且 需要删除的元素和当前的元素一致
                // 意味着在需要删除的元素之前有线程删除了值了，通过判断便找到了新的需要删除值的位置
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    // 设置需要插入的 位置index 为当前的位置，结束循环
                    index = i;
                    break findIndex;
                }
            }
            // 如果 index 大于等于 length，表明删除的位置不在当前的数组中，返回插入失败
            if (index >= len)
                return false;
            // 如果当前的 index 等于 需要插入值，则结束循环，证明从 0 到 需要删除值的位置没有修改
            if (current[index] == o)
                break findIndex;
            // 在 index 和 当前的数组长度直接找到新的需要删除的的位置，表明已经删除过了，
            // 需要从当前数组的 index 位置和 len 之间找到新的需要删除值的位置
            index = indexOf(o, current, index, len);
            // 如果找不到则删除失败
            if (index < 0)
                return false;
        }
        // 创建新的数组
        Object[] newElements = new Object[len - 1];
        // Copy 需要删除元素之前的元素到新的数组
        System.arraycopy(current, 0, newElements, 0, index);
        // Copy 需要删除元素之后的元素到新的数组
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        // 设置新的数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

public boolean removeAll(Collection<?> c) {
    // 如果需要 remove 的集合为空的话，直接抛出异常
    if (c == null) throw new NullPointerException();
    // 获取可重入锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取当前数组的元素
        Object[] elements = getArray();
        // 获取当前数组的长度
        int len = elements.length;
        if (len != 0) {
            // temp array holds those elements we know we want to keep
            // 初始化新的数组的长度
            int newlen = 0;
            // 创建一个新的数组
            Object[] temp = new Object[len];
            // 遍历原始数组
            for (int i = 0; i < len; ++i) {
                // 获取对应位置的元素
                Object element = elements[i];
                // 如果需要删除的集合中不包含当前的元素则 newLen++ i++
                // 否则 newLen 不变， i++ 相等于元素组从该位置起，前移一位
                if (!c.contains(element))
                    temp[newlen++] = element;
            }
            // 如果新的数组的长度和原始数组的长度不一致，则设置数组为新的数组
            if (newlen != len) {
                setArray(Arrays.copyOf(temp, newlen));
                return true;
            }
        }
        return false;
    } finally {
        // 解锁
        lock.unlock();
    }
}

```



### LinkedList

```java
// 存储的数量
transient int size = 0;
// 开始位置
transient Node<E> first;
// 结束位置
transient Node<E> last;

public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    // 将元素全全部添加当链表中
    addAll(c);
}

public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

public boolean addAll(int index, Collection<? extends E> c) {
    // 检查需要插入的位置是否在 0 到 size 中间
    checkPositionIndex(index);

    // 转换为数组
    Object[] a = c.toArray();
    // 获取数组长度
    int numNew = a.length;
    // 长度为 0 直接返回插入失败
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    // 插入位置和原始长度相等
    if (index == size) {
        // 设置下一位为空
        succ = null;
        // 设置前面的一位是最后一个位
        pred = last;
    } else {
        // 否则设置下一位为当前index 的元素
        succ = node(index);
        // 上一位为当前 index 元素的上一位
        pred = succ.prev;
    }

    // 遍历数组构建链表
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        // 如果前一位的值为空，则设置 head 为新的元素的
        if (pred == null)
            first = newNode;
        else
            // 否则设置前一位的后一位为新的元素
            pred.next = newNode;
        // 设置前一位为当前 Node
        pred = newNode;
    }

    // 如果后续一位为空
    if (succ == null) {
        // 则最后一位为前一位，其实就是 链表中最后一位
        last = pred;
    } else {
        // 否则设置 链表中最后一位，为刚才记录的下一位
        pred.next = succ;
        succ.prev = pred;
    }

    // 长度增加
    size += numNew;
    modCount++;
    return true;
}

Node<E> node(int index) {
    // 如果 index 小于 长度的 1/2 则从前面开始查找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 如果index 大于等于长度的 1/2 则从最后一位开始查找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    // 获取最后一位
    final Node<E> l = last;
    // 使用传入的值创建一个 Node
    final Node<E> newNode = new Node<>(l, e, null);

    // 设置最后一位为新的 Node
    last = newNode;
    // 如果最后一位为空
    if (l == null)
        // 则 设置 head 也为新值
        first = newNode;
    else
        l.next = newNode; // 否则设置原始最后一位的下一位为新值
    size++;
    modCount++;
}

public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index)); // 传入需要插入的元素，和 index 对应位置的元素
}

void linkBefore(E e, Node<E> succ) {
    // 获取 index 位置对应的 node 的前一位
    final Node<E> pred = succ.prev;
    // 新建当前的 Node
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 设置 index 的前一位是新的 Node 
    succ.prev = newNode;
    // 如果前一位为空，表示在 head 处插入
    if (pred == null)
        // 将 head 设置为当前 node
        first = newNode;
    else
        pred.next = newNode; //否则设置前一位的后一位为当前 node
    // 存储数量 + 1
    size++;
    modCount++;
}
public E get(int index) {
    // 确保 index 在 0 到 size 中间
    checkElementIndex(index);
    return node(index).item;
}

public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

E unlink(Node<E> x) {
    // 获取当前的元素
    final E element = x.item;
    // 获取当前元素下一个元素
    final Node<E> next = x.next;
    // 获取当前元素的上一个元素
    final Node<E> prev = x.prev;

    if (prev == null) {
        // 上一个元素为空，表示删除开头
        // 设置 head  当前元素的下一个元素
        first = next;
    } else {
        // 设置上一个元素的下一个元素为下一个元素的下一个元素
        prev.next = next;
        // 设置当前元素的 前一个元素为空
        x.prev = null;
    }

    if (next == null) {
        // 下一个元素为空，表示删除最后一个元素，则 last 设置为上一个元素
        last = prev;
    } else {
        // 下一个元素的上一个元素设置为上一个元素
        next.prev = prev;
        // 当前元素的下一个元素设置为空
        x.next = null;
    }

    // 当前元素的值设置为空
    x.item = null;
    size--;
    modCount++;
    return element;
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

