---
title: "Java 四种引用类型"
tags: ["Java", "JVM"]
categories: ["Java"]
date: "2020-02-11T09:00:00+08:00"
---

![四种引用类型](https://i.loli.net/2019/06/16/5d063bd4cf73280638.png)

### 强引用

当内存不足的时候， JVM 开始回收垃圾，对于强引用的对象，就算是出现了 OOM 也不会对该对象进行回收，死都不回收。

强引用是我们常见的普通对象引用，只要还有一个强引用指向一个对象，就表明对象还活着，垃圾收集器不会回收这种对象。在 Java 中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，是不会被垃圾收集机制回收的，即使该对象以后容易不会被用到 ， JVM 也不会回收。因此强引用是造成 Java 内存泄漏的主要原因之一。

对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或显示地将引用赋值为 null，一般认为就是可以被垃圾收集了。

### 软引用

软引用是一种相对引用弱化了一些的引用，需要用 `java.lang.ref.SoftReference` 类 来实现，可以让对象豁免一些垃圾收集器，对于只有软引用的对象来说。当系统内存充足的时候，其不会回收。当系统内存不足的时候，其会被回收。软引用通常用于对内存敏感的程序中，比如高速缓存就有用到软引用，内存够用的时候就保留，不够用就回收。

### 弱引用

虚引用需要使用 `java.lang.ref.WeakReference` 类来实现，它比软引用生存周期更短。对于只有软引用的对象来说，只要垃圾回收一运行，不管 JVM 的内存空间是否足够，都会回收该对象。

#### WeakHashMap

```java
public static void main(String[] args) {
    myHashMap();
    System.out.println("----------");
    myWeakHashMap();
}

private static void myWeakHashMap() {
    WeakHashMap<Integer, String> hashMap = new WeakHashMap<>();
    Integer key = new Integer(1);
    String value = "HashMap";
    hashMap.put(key, value);
    System.out.println(hashMap);
    key = null;
    System.out.println(hashMap);
    System.gc();
    System.out.println(hashMap);
}

private static void myHashMap() {
    HashMap<Integer, String> hashMap = new HashMap<>();
    Integer key = new Integer(1);
    String value = "HashMap";
    hashMap.put(key, value);
    System.out.println(hashMap);
    key = null;
    System.out.println(hashMap);
    System.gc();
    System.out.println(hashMap);
}

/** Console Print
{1=HashMap}
{1=HashMap}
{1=HashMap}
----------
{1=HashMap}
{1=HashMap}
{}
*/
```

### 虚引用

虚引用需要引用 `java.lang.ref.PhantomReference` 类来实现。顾名思义，就是形同虚设，与其他几种引用不同，虚引用并不会决定对象的生命周期。

如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收，它不能单独使用也不能通过它访问对象，虚引用必须和队列 `ReferenceQueue` 联合使用。

虚引用的主要作用是跟踪对象的被垃圾回收的状态。仅仅提供了一种确保对象被 finalize 之后，做某些事情的机制。 `PhantomReference` 的 `get` 方法总是返回 `null`，因此无法访问对应的引用对象。其意义在于说明一个对象已经进入`finalization` 阶段，可以被 `gc` 回收，用实现比 `finalization` 机制更灵活的回收操作。

换句话说，设置虚引用的唯一目的是，就是在这个对象被收集器回收的时寒冰收到一个系统通知或者后续添加进一步的处理。当然 Java 也允许使用 `finalize()` 方法再垃圾收集器将对象从内存中清楚出去之前做必要的清理工作。

#### ReferenceQueue

弱引用使用 `ReferenceQueue`

```java
public static void main(String[] args) {
    Object o1 = new Object();
    ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
    WeakReference<Object> weakReference = new WeakReference<>(o1, referenceQueue);

    System.out.println(o1);
    System.out.println(weakReference.get());
    System.out.println(referenceQueue.poll());
    System.out.println("--------------------");
    o1 = null;
    System.gc();
    System.out.println(o1);
    System.out.println(weakReference.get());
    // 被回收前会放到 ReferenceQueue 中
    System.out.println(referenceQueue.poll());
}
/** Console Print
java.lang.Object@6e0be858
java.lang.Object@6e0be858
null
--------------------
null
null
java.lang.ref.WeakReference@61bbe9ba
*/
```

虚引用使用 `ReferenceQueue`

```java
public static void main(String[] args) {
    Object o1 = new Object();
    ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
    PhantomReference<Object> phantomReference = new PhantomReference<>(o1, referenceQueue);
    System.out.println(o1);
    System.out.println(phantomReference.get());
    System.out.println(referenceQueue.poll());
    System.out.println("--------------");
    o1 = null;
    System.gc();
    System.out.println(o1);
    System.out.println(phantomReference.get());
    System.out.println(referenceQueue.poll());
}
/** Console Print
java.lang.Object@6e0be858
null
null
--------------
null
null
java.lang.ref.PhantomReference@61bbe9ba
*/
```

### 软引用和弱引用使用场景

例如有一个应用需要读取大量的本地图片，如果每次读取图片都从硬盘读取则会影响性能。如果一次性全部加载到内存中又容易导致内存溢出。此时使用软引用或者弱引用解决。

设计思路：用一个 HashMap 来保存图片的路径和对应图片对象关联的软引用或者弱引用之前映射关系，在内存不足时，JVM 会自动回收这些缓存的图片对象所占用的空间，从而有效的避免了 OOM 的问题。