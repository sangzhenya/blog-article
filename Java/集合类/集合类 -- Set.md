## 集合类 -- Set

```java
// 用于存储数据的  hashMap
private transient HashMap<E,Object> map;
// 作为 Value 的 dummy 值
private static final Object PRESENT = new Object();

public HashSet() {
    // 默认的 HashMap 用于存储数据
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    // 如果初始Collection 长度的 4/3 和 16 之间的大的值作为初始化容量
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

public HashSet(int initialCapacity, float loadFactor) {
    // 使用默认的初始容量，默认的负载因子
    map = new HashMap<>(initialCapacity, loadFactor);
}

public boolean add(E e) {
    // 直接调用 map 的 add 方法，dummy 值作为 value
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    // 直接调用 map 的 remove
    return map.remove(o)==PRESENT;
}
```

