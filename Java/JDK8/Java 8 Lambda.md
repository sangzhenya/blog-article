---
title: "Java 8 Lambda"
tags: ["java", "JDK8"]
categories: ["Java"]
date: "2019-01-03T08:00:00+08:00"
---

在编程中可以利用行为参数化来传递代码有助于应对不断变化的需求，即定义一个代码块来表示一个行为，然后传递这个行为。你可以决定在某一个事件发生的时候或者算法中的某个特定的时刻运行该代码，进而编写更为灵活且可重复的代码。而 Lambda 表达式可以让你简洁的表示一个行为或传递代码。Lambda 表达式可以看作匿名功能，基本上就是没有声明名称的方法，和匿名类一样。也可以作为参数传递给一个方法。

### Lambda 基本概念

Lambda 表达式理解为简洁地表示可传递的匿名函数的一种方式，其没有名称，但它有参数列表、函数主题、返回参数类型；可能还有一个可以抛出的异常列表。

```java
Comparator<Apple> byWeight1 = new Comparator<Apple>() {
    @Override
    public int compare(Apple o1, Apple o2) {
        return o1.getWeight() - o2.getWeight();
    }
};
Comparator<Apple> byWeight2 = (Apple o1, Apple o2) -> o1.getWeight() - o2.getWeight();
Comparator<Apple> byWeight3 = Comparator.comparingInt(Apple::getWeight);
```

其中 `byWeight1` 是普通的新建一个 Comparator 方法。`byWeight2` 和 `byWeight3` 则是通过 Lambda 表达式的新建一个 Comparator 方法。

对于 `byWeight2` 的 Lambda 表达式可以分为三个部分：

1. 参数列表：这里采用的是 Comparator 中 compare 方法的参数，两个 Apple
2. 箭头：箭头 -> 把参数列表与 Lambda 主体分隔开
3. Lambda 主体：比较两个 Apple 的重量，表达式就是 Lambda 的返回值。



下面是几个有效的 Lambda 表达式

```java
// 有一个 String 类型的参数并返回一个 int
(String s) -> s.length();

// 有一个 Apple 类型的参数并返回一个 boolean
(Apple a) -> a.getWeight() > 150;

// 有来个具有 int 类型的参数，没有返回值
(int x, int y) -> {
    System.out.println("Result:");
    System.out.println(x + y);
}

// 没有参数，返回一个 int
() -> 42;

// 有来个 Apple 类型的参数，返回一个 int
(Apple a1, Apple a2) -> a1.getWeight() - a2.getWeight(); 
```

基本语法有两种：

1. `(parameters) -> expression`
2. `(parameters) -> { statements}`



### 在何处使用 Lambda

Lambda 使用在函数式接口上，函数式接口就是只定义一个抽象方法的接口。例如 Java 中的 `Comparator` 接口和 `Runnable` 接口。接口有多个默认方法，如果只有一个抽象方法，该接口仍然是函数式接口。

Lambda表达式允许直接以内联的形式为函数式接口的抽象方法的具体实现，并把整个表达式作为函数式接口的实例，类似于匿名内部类的实现。此外函数式接口的抽象方法的签名基本上就是 Lambda 表达式的签名。



### Java 8 中函数式接口

在 JDK 8 中新引入了函数式接口。下面说一下  Predicate、Consumer 和 Function 三个接口。

#### Predicate

`java.util.function.Predicate<T>` 接口定义了一个 `test` 抽象方法，接受泛型 T 对象，并返回一个 boolean。在需要表示涉及一个类型为 T 的布尔表达式时，可以使用该接口。例如：

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}

public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("");
    list.add("0");
    list.add("1");
    list.add("2");
    List<String> results = filter(list, (String s) -> !s.isEmpty());
    System.out.println(results);
}

public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
	List<T> results = new ArrayList<>();
	for (T t : list) {
		if (predicate.test(t)) {
			results.add(t);
		}
	}
	return results;
}
```

#### Consumer

`java.util.function.Consumer<T>` 定义了一个 accept 的抽象方法，接受泛型 T 的对象，没有返回。如果需要访问类型 T 的对象，并对其执行某些操作，则可以使用该接口。例如

```java
@FunctionalInterface
public interface Consumer<T> {
	void accept(T t);
}
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("");
    list.add("0");
    list.add("1");
    list.add("2");
    forEach(list, (String s) -> System.out.println(s));
}

public static <T> void forEach(List<T> list, Consumer<T> consumer) {
    for (T t : list) {
        consumer.accept(t);
    }
}
```

#### Function

`java.util.function.Function<T, R>` 定义了一个 apply 的抽象方法，接受一个泛型 T 的对象，并返回一个泛型 R 对象。如果需要定义一个 Lambda，将输入对象的信息映射到输出，就可以使用该接口。例如

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("0");
    list.add("11");
    list.add("222");
    List<Integer> result = map(list, (String s) -> s.length());
    System.out.println(result);
}

public static <T, R> List<R> map(List<T> list, Function<T, R> function) {
    List<R> result = new ArrayList<>();
    for (T t : list) {
        result.add(function.apply(t));
    }
    return result;
}
```

#### Lambda 及函数式接口的例子 

```java
// 布尔表达式 | Predicate<List<String>>
(List<String> list) -> list.isEmpty();

// 创建对象 | Supplier<Apple>
() -> new Apple();

// 消费一个对象 | Consumer<Apple>
(Apple a) -> System.out.println(a.getWeight());

// 从对象中选择或提取 | FUncation<String, Integer> 或 ToIntFuncation<String>
(String s） -> s.length();
 
// 合并两个值 | IntBinaryOperator
(int a, int b) -> a * b;
 
// 比较来个对象 | Comparator<Apple> 或 BiFuncation<Apple, Apple, Integer> 或 ToIntBiFuncation<Apple, Apple>
(Apple a1, Apple a2) -> a1.getWeigt() - a2.getWeight();
```



### 类型检查及类型推断

#### 类型检查

例子如下：

```java
public static void main(String[] args) {
    List<Apple> list = new ArrayList<>();
    Apple apple1 = new Apple();
    apple1.setWeight(105);
    list.add(apple1);

    Apple apple2 = new Apple();
    apple2.setWeight(115);
    list.add(apple2);

    Apple apple3 = new Apple();
    apple3.setWeight(175);
    list.add(apple3);

    List<Apple> result = filter(list, (Apple a) -> a.getWeight() > 150);
    System.out.println(result);
}

public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
    List<T> results = new ArrayList<>();
    for (T t : list) {
        if (predicate.test(t)) {
            results.add(t);
        }
    }
    return results;
}
```

类型检查可以分为以下几步：

1. 找到 filter 的方法声明：
2. 要求其是 Predicate<Apple> 目标类型对象的第二个正式参数
3. Predicate<Apple> 是一个函数式接口，定义了一个 test 的抽象方法。
4. test 方法描述了一个函数描述符，它可以接受一个 Apple，并返回一个 boolean。
5. filter 的任何实际参数都必须匹配这个要求。
   通过以上几步便可以确定该段代码是有效的。

#### 类型推断

JAVA 编译器回会从上下文推断用了什么函数式接口，即推断出合适 lambda 表达式的签名，因为函数描述符可以通过目标类型来得到。如此我们便可以在 lambda 表达式中省略标注参数类型。如下所示：

```java
// 直接指定参数类型
Comparator<Apple> comparator1 = (Apple a1, Apple a2) -> a1.getWeight() - a2.getWeight();
// 使用参数类型推断
Comparator<Apple> comparator2 = (a1, a2) -> a1.getWeight() - a2.getWeight();
List<Apple> result = filter(list, a -> a.getWeight() > 150);
```

另外 lambda 表达式只有一个类型需要推断的时候，参数名称两边的括号可以省略。



### 方法引用

#### 构建方法引用

方法引用让你可以重复使用现有方法的定义，并想像 lambda 一样传递它们。如下例所示：

```java
// 未使用方法引用
list.sort((a1, a2) -> a1.getWeight() - a2.getWeight());
// 使用方法引用
list.sort(Comparator.comparingInt(Apple::getWeight));
```

方法引用可以看做仅仅调用特定方法的 lambda 的一种快捷写法。即一个lambda 代表的只是直接调用这个方法。方法引用就是让你根据已有的方法实现创建 lambda 表达式。当你使用方法引用时，目标引用放在`::`前，方法名称放在后面。例如 `Apple::getWeight`就是引用了 Apple 类中定义的方法  getWeight 。注意这边不需要括号，因为并没有实际调用该方法。上述表达式其实就是 lambda 表达式  `(Apple a) -> a.getWeight()` 的快捷写法。以下是几个简单的例子：

```java
// Case 1
(Apple a) -> a.getWeight()
Apple::getWeight

// Case 2
() -> Thread.currentThread().getId()
Thread.currentThread()::getId

// Case 3
(String str, int i) -> str.substring(i)
String::substring

// case 4
(String str) -> System.out.println(str)
System.out::println
```

方法引用主要有三类：

1. 指向静态方法的方法引用。例如 `Integer` 的 `parseInt`方法，写作 `Integer::paraeInt`
2. 指向任意类型实例方法的方法引用。例如 `String`的`length`方法，写作 `String::length`
3. 指向现有对象的实例方法引用。
   以上三种等价于下面的 lambda 表达式：

```java
// Case 1
(args) -> ClassName.staticMethod(args)
ClassName::staticMethod    

// Case 2
(arg0, rest) -> arg0.instanceMethod(rest)
ClassName::instanceMethod    

// Case 3
(args) -> expr.instanceMethod(args)
expr::instanceMethod    
```

#### 构造函数引用

对于一个现有构造函数，可以利用它的名字和 new 关键字来创建它的一个引用：`ClassName::new`

```java
// Case 1
Supplier<Apple> supplier1 = Apple::new;
Apple a1 = supplier1.get();
// 等价
Supplier<Apple> supplier2 = () -> new Apple();
Apple a2 = supplier2.get();

// Case 2
Function<Integer, Apple> function1 = Apple::new;
Apple a1 = function1.apply(100);
// 等价
Function<Integer, Apple> function2 = (weight) -> new Apple(weight);
Apple a2 = function2.apply(100);
```

也可以由一个 List 中的每一个元素通过定义的 map 方法传递给 Apple 的构造函数，从而得到一个具有不同属性的对象 List。例如：

```java
public static void main(String[] args) {
    List<Integer> weights = Arrays.asList(100, 115, 170);
    List<Apple> apples = map(weights, Apple::new);
}

public static List<Apple> map(List<Integer> weights, Function<Integer, Apple> function) {
    List<Apple> result = new ArrayList<>();
    for (Integer weight : weights) {
        result.add(function.apply(weight));
    }
    return result;
}
```

对于具有两个参数的构造函数则适合 BiFunction 接口的签名。例如：

```java
BiFunction<Integer, String, Apple> function = Apple::new;
Apple a = function.apply(10, "red");
```



### 复合 lambda 表达式

在 Java 8 中许多函数接口都提供允许你进行复合的方法，即可以把多个简单的 lambda 表达式复合为复杂的表达式。主要有一下几种。

#### 比较器复合

对于根据对象的某个属性比较对象，我们可以使用类似下面的方法处理。

```java
appleList.sort(Comparator.comparingInt(Apple::getWeight))
```

如果需要逆序并不需要创建一个新的 Comparator 对象，仅仅需要使用接口的默认方法 reversed 使给定的比较器逆序即可。如下所示

```java
appleList.sort(Comparator.comparingInt(Apple::getWeight).reversed())
```

如果不仅仅需要使用一种比较器比较，例如对于相同重量的苹果根据国家来排序，那么可以使用 `thenComparing`方法比较。如下

```java
appleList.sort(Comparator.comparingInt(Apple::getWeight).reversed()
                .thenComparing(Apple::getColor));
```

#### 谓词复合

谓词接口包括三个方法：negate, and 和 or，利用这些谓词可以重用已有的 Predicate 来创建更复杂的谓词。
使用 negate 返回一个 predicate 的非。如下：

```java
Predicate<Apple> fullWeightPredicate = apple -> apple.getWeight() > 150;
Predicate<Apple> lessWeightPrediccate = fullWeightPredicate.negate();
```

使用 and 将两个 lambda 进行组合。如下：

```java
Predicate<Apple> fullWeightPredicate = apple -> apple.getWeight() > 150;
Predicate<Apple> redPredicate = apple -> "red".equals(apple.getColor());
Predicate<Apple> fullWeightRedPredicate = fullWeightPredicate.and(redPredicate);
```

或者进一步组合。如下：

```java
Predicate<Apple> fullWeightPredicate = apple -> apple.getWeight() > 150;
Predicate<Apple> redPredicate = apple -> "red".equals(apple.getColor());
Predicate<Apple> greenPredicate = apple -> "green".equals(apple.getColor());
Predicate<Apple> fullWeightRedPredicate = fullWeightPredicate.and(redPredicate).or(greenPredicate);
```

注意 and 和 or 方法按照表达式链中的位置，从左到右确定优先级的。所以 `a.or(b).and(c)`是`(a||b)&&c`

#### 函数复合

Funcation 接口提供了 `andThen`和`compose`两个方法将两个 Function 组合起来返回一个 Function 实例。
`andThen`方法先对输入应用一个函数，再对输出应用另外一个函数。如下所示：

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
Function<Integer, Integer> h = f.andThen(g);
int result = h.apply(1);

// ( x + 1) * 2
```

`compose`方法则是先把给定的函数用作 compose 的参数里面的那个函数，然后再把函数本事用于结果。如下所示：

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
Function<Integer, Integer> h = f.compose(g);
int result = h.apply(1);

// (x * 2) + 1
```

