## Java 8 Stream
### 流与集合
#### 流

流是 Java 的新 API，允许以声明的方式处理集合。简单来说你可以把流看作是遍历数据集的高级迭代器。此外流还可以透明的并行处理。Java 8的集合支持一个新的 stream 方法，这个方法会返回一个流。

#### 流与集合

粗略的来说流与集合的差异在于什么时候进行计算。集合是一个内存数据结构，包含数据结构中目前所有值。流则是概念上的数据结构，其元素是按需计算的。从另外一个角度说流就是一个延迟创建的集合。另外和迭代器类似，流只能遍历一次，遍历完表示该流已经被消费掉了，只能重新获取然后再次迭代。

#### 内部迭代和外部迭代

```java
List<String> names = new ArrayList<>();

// 外部迭代：使用增强 for 循环进行迭代
for (Dog dog : dogList) {
    names.add(dog.getName());
}

// 外部迭代：使用迭代器进行迭代
Iterator<Dog> iterator = dogList.iterator();
while (iterator.hasNext()) {
    Dog dog = iterator.next();
    names.add(dog.getName());
}

// 内部迭代：使用流
names = dogList.stream().map(Dog::getName).collect(Collectors.toList());
```

内部迭代和外部迭代如下图所示：

![内部迭代和外部迭代](https://i.loli.net/2019/04/25/5cc1bad68c7c0.png)



### 中间操作和终端操作
一个简单的流操作
```java
List<String> dogNames = dogList.stream()
    .filter(d -> d.getWeight() > 99)
    .map(Dog::getName)
    .limit(2)
    .collect(Collectors.toList());
```

图示如下：

![流操作示意图](https://i.loli.net/2019/04/25/5cc1bcde06769.png)



图中可以看到两类操作：

1. filter, map 和 limit 组成一条流水线。
2. collect 触发流水线执行并关闭它。

#### 中间操作
`java.util.stream.Stream`中的 Stream定义了很多操作，可以分为两类，其中可以连接起来的流操作称为中间操作。诸如 filter 或 sorted 等操作会返回一个流，这让多个操作连接起来的形成一个查询。另外流水线上至少有一个终端操作，否则中间操作不会执行任何处理。
中间操作如下列表：

| 操作     | 返回类型  | 操作数         | 函数描述符     |
| -------- | --------- | -------------- | -------------- |
| filter   | Stream<T> | Predicate<T>   | T -> boolean   |
| map      | Stream<T> | Function<T, R> | T -> R         |
| limit    | Stream<T> |                |                |
| sorted   | Stream<T> | Comparator<T>  | (T, T) -> int  |
| distinct | Stream<T> |                |                |
| skip     | Stream<T> | long           |                |
| flatMap  | Stream<T> | Function<T, R> | T -> Stream<R> |

#### 终端操作
Stream 中另外一类操作称为终端操作，其会从流的流水线生成一个结果，其结果是任何不是流的值。
终端操作如下列表：

| 操作      | 返回类型    | 操作数             | 函数描述符   |
| --------- | ----------- | ------------------ | ------------ |
| forEach   | void        | Consumer<T>        | T -> void    |
| count     | long        |                    |              |
| collect   | R           | Collector<T, A, B> |              |
| anyMatch  | boolean     | Predicate<T>       | T -> boolean |
| noneMatch | boolean     | Predicate<T>       | T -> boolean |
| allMatch  | boolean     | Predicate<T>       | T -> boolean |
| findAny   | Optional<T> |                    |              |
| findFirst | Optional<T> |                    |              |
| reduce    | Optional<T> | BinaryOperator<T>  | (T, T) -> T  |



### 筛选和切片
#### filter

filter 方法接受一个谓词作为参数，并返回一个包括所有符合谓词的元素的流。例如：

```java
List<Dog> myDog = dogList.stream()
    .filter(Dog::isOwn)
    .collect(Collectors.toList());
```
#### distinct

distinct 会返回元素各异的流。例如：

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 4, 3, 2);
numbers.stream()
    .filter(i -> i % 2 == 0)
    .distinct()
    .forEach(System.out::println);
```
#### limit

limit方法返回一个不超过指定长度的流，所需的长度通过参数传递给limit。例如：

```java
List<Dog> limitDog = dogList.stream()
                .filter(d -> d.getWeight() > 99)
                .limit(1)
                .collect(Collectors.toList());
```
#### skip

skip 方法返回一个扔掉前 n 个元素的流。如果长度不够，则返回一个空流。例如：

```java
List<Dog> skipDog = dogList.stream()
                .filter(d -> d.getWeight() > 99)
                .skip(1)
                .collect(Collectors.toList());
```



### 映射

#### map

流支持 map 方法其接受一个函数，并将该函数应用在每个元素上，映射成一个新的元素。例如：

```java
List<Integer> dogNameLength = dogList.stream()
                .map(Dog::getName)
                .map(String::length)
                .collect(Collectors.toList());
```
#### flatMap

如果有一个数组的列表想要生成一个扁平的元素的列表则可以使用  flatMap。其效果就是各个数组不是分别映射成一个流，而是映射成为流的内容。例如：

```java
String[] words = {"Java", "Stream"};
Arrays.stream(words)
    .map(word -> word.split(""))
    .flatMap(Arrays::stream)
    .distinct()
    .collect(Collectors.toList());
```

简单来说 flatMap 方法就是把流中每一个值都换成另外一个流，然后把所有的流连接起来成为一个流。



### 查找和匹配

#### match

**anyMatch**：至少一个match
**allMatch**：所有的match
**noneMatch**：没有一个match

```java
 if (dogList.stream().anyMatch(Dog::isOwn)) {
     System.out.println("There is at least one of mine");
 }
if (dogList.stream().allMatch(Dog::isOwn)) {
    System.out.println("There all dog of mine");
}
if (dogList.stream().noneMatch(Dog::isOwn)) {
    System.out.println("There is no dog of mine");
}
```
#### find

**findAny**：返回当前流中的任一元素。
**findFirst**：返回当前流中的第一个元素。
例如：

```java
Optional<Dog> myAnyDog = dogList.stream()
    .filter(Dog::isOwn)
    .findAny();
Optional<Dog> myFirstDog = dogList.stream()
    .filter(Dog::isOwn)
    .findFirst();
```



### 规约 reduce

reduce 将流中的所有元素反复结合得到一个值。
例如：

```java
// 使用原始方法求和
int sum = 0;
for (Integer number : numbers) {
	sum += number;
}

// 使用流的 reduce 方法求和
sum = numbers.stream().reduce(0, (a, b) -> a + b);

// 求最大和最小值
numbers.stream().reduce(Integer::max);
numbers.stream().reduce(Integer::min);
```
### 收集 collect

1. 最大值最小值

   ```java
   Optional<Dog> heaviestDog = dogList.stream()
       .collect(Collectors
       	.maxBy(Comparator
       		.comparingDouble(Dog::getWeight)));
   Optional<Dog> heaviestDog = dogList.stream()
       .max(Comparator
       	.comparingDouble(Dog::getWeight));
   ```

2. 求和

   ```java
   Double weight = dogList.stream()
       .collect(Collectors
       	.summingDouble(Dog::getWeight));
   Double weight = dogList.stream()
       .mapToDouble(Dog::getWeight).sum();
   ```

3. 连接字符串

   ```java
   dogList.stream()
       .map(Dog::getName)
       .collect(Collectors.joining(","));
   ```

4. 分组

   ```java
   // 单级分组
   Map<String, List<Dog>> dogMap = dogList.stream()
                   .collect(Collectors
                           .groupingBy(Dog::getGender));
   // 多级分组
   Map<String, Map<String, List<Dog>>> dogMap = dogList.stream()
       .collect(Collectors
       .groupingBy(Dog::getGender, Collectors.groupingBy(dog -> {
       	if (dog.getWeight() > 99) {
       		return "SMALL";
       	} else {
       		return "Big";
       	}
       })));
   
   // 按子组收集数据
   dogList.stream()
   	.collect(Collectors.groupingBy(Dog::getGender, 
   		Collectors.counting()));
   
   dogList.stream()
       .collect(Collectors.groupingBy(Dog::getGender, 
   			Collectors.maxBy(Comparator
   				.comparingDouble(Dog::getWeight))));
   ```

5. 分区

   分区是分组的特殊情况，由一个谓词即返回 boolean 的函数作为分类函数，即分区函数。

   ```java
   dogList.stream()
       .collect(Collectors
       	.partitioningBy(Dog::isOwn));
   ```

6. Collectors 类的静态工厂方法

| 工厂方法          | 返回类型              | 用于                                                         |
| ----------------- | --------------------- | ------------------------------------------------------------ |
| toList            | List<T>               | 把流中所有的项目收集到一个 List                              |
| toSet             | Set<T>                | 把流中所有的项目收集到一个 Set                               |
| toCollection      | Collection<T>         | 把流中所有的项目收集到给定的供应源创建的集合                 |
| counting          | Long                  | 计算流中元素的个数                                           |
| summingInt        | Integer               | 对流中的项目的一个整数属性求和                               |
| averagingInt      | Double                | 计算流中项目的 Integer 属性的平均值                          |
| summarizingInt    | IntSummaryStatistics  | 收集关于流中项目 Integer 属性的统计值，例如最大、最小、总和与平均值 |
| joining           | String                | 连接对流中每个项目调用 toString 方法产生的字符串             |
| maxBy             | Optional<T>           | 一个包裹了流中按照给定的比较器选出的最大元素的 Optional，或如果流为空为 Optional.empty() |
| minBy             | Optional<T>           | 一个包裹了流中按照给定的比较器出选出最小元素的Optional， 或如果流为空为 Optional.empty() |
| reducing          | 规约操作产生的类型    | 一个作为累加器的初始值的开始，利用 BinaryOperator 与流中元素逐个结合，从而将流规约为单个值 |
| collectingAndThen | 转换函数返回的类型    | 包裹另外一个收集器，对其结果应用转换函数                     |
| groupingBy        | Map<K, List<T>>       | 根据项目的一个属性的值对流中的项目进行分组，并记得属性值作为结果 Map 的键 |
| partitioningBy    | Map<Boolean, List<T>> | 根据对流中的项目的应用谓词的结果进行分区                     |



### 数值流
Java 8 提供了三个原始类型的特化流来解决操作数值时可能暗含的装箱成本。分别是：IntStream, DoubleStream, LongStream  分别来流中的元素特化位为 int，double 和 long。例如：
```java
// 使用包装类型求和
double totalWeigh = dogList.stream()
    .map(Dog::getWeight)
    .reduce(0d, Double::sum);

// 使用数值类型流求和
// 映射到数值流
totalWeigh = dogList.stream()
    .mapToDouble(Dog::getWeight)
    .sum();

// 转换回对象流
dogList.stream()
    .mapToDouble(Dog::getWeight)
    .boxed();
```

对于求最大值的情况，流中可能并不存在最大值，对于这种情况使用默认值是 0 的话，显然是错误的。这个时候需要引入一个新的类 `OptionalDouble` 。

```java
// 查找流中最大值
OptionalDouble maxWeight = dogList.stream()
    .mapToDouble(Dog::getWeight)
    .max();

// 如果没有最大值，可以显示定义一个默认值
maxWeight.orElse(1);

// 生成一定范围数值的数字
IntStream.rangeClosed(1, 100).filter(n -> n % 2 == 0);
```



### 构建流

#### 由值构建

```java
Stream.of("Java", "Stream")
    .map(String::toUpperCase)
    .forEach(System.out::println);
```
#### 由数组构建

```java
int[] numbers = {1, 2, 3, 4, 5};
int sum = Arrays.stream(numbers).sum();
```
#### 由文件生成流

```java
long uniqueWords = 0;
try (Stream<String> lines = Files.lines(Paths.get("data.txt"), Charset.defaultCharset())) {
    uniqueWords = lines.flatMap(line -> Arrays.stream(line.split(" "))).distinct().count();  
} catch (IOException e) {}
```
#### 由函数生成流

Stream 提供了两个静态方法来从函数中生成流：Stream.iterate 和 Stream.generate。 这两个操作可以创建所谓的无限流：即由 iterate 和 generate 产生的流会用给定的函数按需创建，可以无穷尽的计算下去，一般来说应该使用 limit(n) 来对这种流加以限制。

```java
// 简单的迭代
Stream.iterate(0, n -> n + 2)
    .limit(10)
    .forEach(System.out::println);

// 简单的生成
Stream.generate(Math::random)
    .limit(5)
    .forEach(System.out::println);
```



### 收集器接口

Collector 接口包含了一系列方法，为实现具体的规约操作提供了样本。其定义了以下五个接口：

#### supplier

supplier 方法用于建立新的结果容器。其返回一个结果为空的 Supplier，也就是一个无参数函数，在调用时它会产生一个空的累加器实例，供数据收集过程使用。

#### accumulator

accumulator 方法用于将元素添加到结果容器。其返回执行规约操作的函数。当遍历到流中第 n 个元素时，这时会有两个参数：保存规约结果的累加器，还有第 n 个元素本身。该函数将返回 void，因为累加器是原位更新，即函数的执行改变了它的内部状态以提现遍历元素的效果。

#### finisher

finisher 方法用于对结果容器应用最终转换。在遍历完流后，finisher 方法必须返回在累加过程的最后一个调用的函数，以便将累加器对象转换为整个集合操作的最终结果。

#### combiner

combiner 方法用于合并两个结果容器。其返回一个供规约操作使用的函数，其定义了对流的各个子部分的进行并行处理时，各个子部分规约所得累加器要如何合并。

#### characteristics 

characteristics 方法返回一个不可变的 Characteristcs 集合，其定义了收集器的行为，尤其是流是否可以并行规约，以及可以使用哪些优化的提示。包含以下三个枚举：

1. UNORDERED ：规约结果不受流中项目遍历和累积顺序的影响。
2. CONCURRENT ：accumulator 函数可以从多个线程调用，且收集器可以并行规约流，如果收集器没有标为 UNORDERED，那它仅用于无序数据源时才可以并行归纳。
3. IDENTITY_FINISH ：表明完成器方法返回的函数是一个恒等函数可以跳过，这种情况下累加器对象将会直接用作规约过程的最终结果。意味着累加器 A 不加检查转换为结果 R 是安全的。



一个过程图如下：



![一个规约过程图](https://i.loli.net/2019/04/27/5cc32e23b6749.png)