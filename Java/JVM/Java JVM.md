## Java JVM

### MinorGC 的过程 

复制 --> 清空 --> 互换

1. eden, SurviorFrom 复制到 SurviorTo, 年龄 + 1

   首先 当 Eden 区满的时候回触发一次 GC，把仍然存活的对象复制到 SurviorFrom 区，当 Eden 去再次触发GC的时候会扫描 Eden区和From区域，对这个两个区域进行垃圾回收，经过这次回收后还存活的对象则直接复制到 To 区域（如果对象的年龄已经达到了老年的标准，则复制到 老年代），同时把这些对象的年龄 + 1.

2. 清空 eden, SurviorFrom

   然后清空 Eden 和 SurviorFrom区中的对象，也即复制之后有交换，谁空谁是to

3. SurviorTo 和 SurviorFrom 互换

   最后 SurviorFrom 和 SurviorTo 互换，原SurviorTo区域成为下一次 GC 时的 SurviorFrom 区域。部分对象会在 From 和 To 区域中复制来复制去，如此交换 15次，如果仍然存活则放到老年代。该参数由 MaxTenuringThreshold 决定，默认是 15



### 可达性分析

为了解决引用计数法的循环引用的问题，Java 使用了可达性分析的方法。通过 GC Root 遍历引用。所谓的 GC Roots 是一组必须活跃的引用。其基本思路就是通过一系列名为 “GC Roots” 的对象作为起始点，从这个被 成为 GC Roots 的对象开始向下搜索，如果一个对象到 GC Roots 没有任何引用链相连时，则说明此对象不可用。也即给定一个集合的引用作为根出发，通过引用关系遍历对象图，能被遍历到（可达到的）对象就被判定为存活，没有被遍历到的则判断为死亡。

GC Roots 可以是以下类型的对象：

1. 虚拟机栈（栈帧中的局部变量区）中引用的对象。
2. 方法区中的类静态属性引用的对象。
3. 方法区中常量引用的对象的。
4. 本地方法栈中 JNI 引用的对象。

```java
public class GCRootDemo {
    private byte[] bytes = new byte[100 * 1024 * 1024];
    private static GCRootDemo2 demo2;	// case 2
    private static final GCRootDemo3 demo3 = new GCRootDemo3(); // case 3
   	public static void m1() {
        GCRootDemo1 demo1 = new GCRootDemo1(); // case 1
    }
    public static void main(String[] args) {
        m1();
    }
}
```



### JVM 参数

#### 标配参数

```shell
-version
-help
java -showversion
```

#### X 参数

```shell
-Xint	# 解释执行
-Xcomp	# 第一次使用就变异成本地代码
-Xmixed	# 混合模式
```

####  XX 参数

##### Boolean 类型

```shell
-XX: 
# + 或者- 某个属性
# + 表示开启
# - 表示关闭

--XX:+PrintGCDetails  # 打印 GC 信息
--XX:+UseSerialGC		# 使用串行垃圾回收器

#----

jps -l # 查看运行的程序
jinfo -flag <参数名称> <程序进程号>
```

+ jps 查看当前运行的 Java 程序

![查看当前运行的程序](https://i.loli.net/2019/06/15/5d04338a4778c90439.png)

+ jinfo 查看 Java 程序信息

  ![Java 程序信息](https://i.loli.net/2019/06/15/5d0435fa5cb0a15680.png)
  
+ jmap 内存映像工具

+ jstat 统计信息监视工具

+ jstack 堆栈异常跟踪工具

+ jvisualvm

+ jconsole

  
  
  

##### KV 设置类型

```shell
# KV 设值类型
--XX:MetaspaceSize			# 元空间大小
--XX:MaxTenuringThreshold	# 新生代到老年代年龄限制
-Xms	# = -XX:InitialHeapSize	默认是物理内存的 1/64
-Xmx	# = -XX:MaxHeapSize		默认是物理内存的 1/4

#-------------
jinfo -flags <进程号>	# 查看 Java 程序的参数设置
java -XX:+PrintFlagsInital  # 查看默认参数设置
java -XX:+PrintFlagsFinal	# 查看参数最终值
java -XX:+PrintCommandLineFlags -version # 查看参数默认设置

UseAES = true
UseCompressedClassPointers := true # := 表示被人为或者系统修改过

```

+ 查看 Java 程序默认参数

![Java 程序默认参数](https://i.loli.net/2019/06/15/5d0436cbe460468773.png)

+ 查看 Java 全局默认参数

  ![](https://i.loli.net/2019/06/15/5d043868a0b9f71065.png)

+ 查看默认设置

![查看默认参数设置](https://i.loli.net/2019/06/15/5d043bb91ce8e64159.png)

#### 常用的 JVM 参数

通过程序查看 JVM 的一些信息

```java
long totalMemory = Runtime.getRuntime().totalMemory();
long maxMemory = Runtime.getRuntime().maxMemory();
System.out.println(totalMemory/1024/1024);
System.out.println(maxMemory/1024/1024);
```

常用的参数

```shell
-Xms		# 初始大小内存 默认为物理内存的 1/64 等价于 -XX:InitialHeapSize
-Xmx		# 最大分配内存 默认为物理内存的 1/4 等价于 -XX:MaxHeapSize
-Xss		# 设置单个线程栈的大小 默认为 512k ~ 1024k 
			# 等价于 -XX:ThreadStackSize  默认值依赖于平台
-Xmn		# 新生区内存的大小
-XX:MetaspaceSzie	# 设置元空间大小
-XX:+PrintGCDetails	# 打印 GC 的详细信息
-XX:SurviorRatio	# 设置新生代中 eden 和 s0/s1 空间的比例
					# 默认是 -XX:SurvivorRatio=8, Eden:s0:s1=8:1:1
					# 假如 -XX:SurvivorRatio=4, Eden:s0:s1=4:1:1
					# SurvivorRatio 值就是设置 Eden 区的比例占多少，s0和s1 相同
-XX:NewRatio		# 配置年轻代与老年代在堆结构的占比
					# 默认 -XX:NewRatio=2 新生代占1，老年代占2，年轻代占整个对的 1/3
					# 假如 -XX:NewRatio=4 新生代占1，老年代占4.
					# NewRatio 值就是设置老年代的占比，剩下的 1 分给新生代
-XX:MaxTenuringThreshold	# 设置垃圾最大年龄


#----
-Xms128m -Xmx4096m -Xss1024k -XX:Metaspace=512m 
-XX:+PrintCommandLineFlags -XX:+PrintGCDetails -XX:+UseSerialGC

#---
YungGC
# [GC类型 (状态)   		
[GC (Allocation Failure) 
# [新生代： 回收前内存 -> 回收后内存（总内存）]
[PSYoungGen: 1927K->496K(2560K)] 
# 回收前 JVM 堆内存 -> 回收后 JVM 堆内存(JVM 堆大小) GC 耗时]
1927K->820K(9728K), 0.0020666 secs] 
# [耗时：user=用户耗时 sys=系统耗时, real=实际耗时]
[Times: user=0.00 sys=0.00, real=0.00 secs] 

FullGC
# [GC 类型（状态） , 
[Full GC (Allocation Failure) 
# [新生代：回收前内存 -> 回收后内存（总内存）]
[PSYoungGen: 448K->0K(2560K)] 
# [老年代：回收前内存大小 -> 回收后内存大小] 回收前 JVM 堆内存大小 -> 回收后堆内存大小（JVM 堆大小）
[ParOldGen: 324K->697K(7168K)] 772K->697K(9728K), 
# [元空间：回收前大小 -> 回收后大小], GC 耗时]
[Metaspace: 3278K->3278K(1056768K)], 0.0066242 secs] 
# [耗时：user=用户耗时 sys=系统耗时, real=实际耗时]
[Times: user=0.11 sys=0.00, real=0.01 secs] 

```

![Java 8 堆栈信息](https://i.loli.net/2019/06/15/5d044275bff1c67699.png)


