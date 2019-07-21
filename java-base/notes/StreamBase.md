## Stream 基础

### 基础介绍
Stream（流）是Java8版本的一个新特性。学习流时需与I/O中的输入输出流区别开来。

操作流让我们的关注点从“怎么做”转移到了“做什么”。它有很高的可读性，代码量也少于以往的循环迭代方式。并且，Java8针对流提供了许多十分便捷的调用方法，例如排序（sorted）、查找元素（max/min/findFirst···）、收集（collect）等。

假设需求为获取字符串列表中以"A"开头的字符串，并仅获取前3个，使用普通方式实现如下：
```
public List<String> method() {
    List<String> list = Arrays.asList("A1", "A2", "B1", "C1", "A3", "A4", "B2", "D1");
    List<String> result = new ArrayList<>();
    for (String str : list) {
        if (str.startsWith("A")) {
            result.add(str);
            if (result.size() == 3) {
                break;
            }
        }
    }
    return result;
}
```

如果获取list的流并对其操作，代码如下：
```
public List<String> method() {
    List<String> list = Arrays.asList("A1", "A2", "B1", "C1", "A3", "A4", "B2", "D1");
    return list
        .stream()
        .filter(str -> str.startsWith("A"))
        .limit(3)
        .collect(Collectors.toList());
}
```
在foreach版本中，我们只有跟随开发者思维，走进循环体，才可得知这一块代码的作用。而使用Stream后，通过追加的方法名，即可很轻松的读取代码含义。开发者也从“怎么做”的思维转化为“做什么”。

流不是集合的扩展，以下是Stream与集合的不同点（摘自Oracle与《Java核心技术卷 Ⅱ》）：
- 流不存储元素。它通过管道从源（像数据结构、数组、生成器函数或I/O通道）传递元素。
- 流的操作不会修改数据源。对流做的操作不会修改源，而是生成一个新的流。
- 流的操作是惰性执行的。直到需要其结果时，操作才会执行。
- 流没有边界的限制。集合中的元素个数是有限的，而Java允许生成一个无线的流。
- 流是可消费的。由第一点得知流本身不会存储元素，元素在流的生命周期里只会被访问一次，想再次访问相同元素需生成一个新的流。

流有许多的获取方式。
- 集合的 `stream()` 方法与 `parallelStream()` 可分别用于获取一个顺序流或并行流；
- 调用 `Arrays.stream(Object[])` 获取一个数组的流；
- 通过静态工厂方法获取流（ `Stream.of(Object[])` 、 `IntStream.range(int, int)` 等）；
- 通过 `BufferedReader.lines()` （非静态方法）获取一个文件的流；
- 通过 `Random.ints() ` （非静态方法）获取一个随机数的流；
- 在JDK中定义的其他获取流的方法。

操作流的方法可分为两种类型：
- 中间操作：对流执行该类型操作会返回一个新的流，例如 filter、distinct、limit等；
- 终结操作：对流执行该类型操作会返回一个非流值，例如 forEach、sum、collect等。

### 基础应用

#### 获取流
- 从集合获取流
```
public void method() {
    Collection<String> collection = new ArrayList<>();
    // add some elements to collection
    Stream<String> stream = collection.stream();
}
```
由于我们常用的List、Set也属于集合，代码中的Collection也可用相应的子类（接口）替代。

- 从数组获取流
```
public void method() {
    String[] arr = new String[5];
    // add some elements to array
    Stream<String> stream = Arrays.stream(arr);
}
```
有时候我们操作的是一个基础类型的数组，JDK也提供了对应 `IntStream` 、 `LongStream` 、 `DoubleStream` ：
```
public void method() {
    int[] arr = new int[5];
    // add some elements to array
    IntStream stream = Arrays.stream(arr);
}
```

- 获取一个空流
```
public void method() {
    Stream stream = Stream.empty();
}
```

- 获取一个无限流
```
public void method() {
    Stream<Double> stream = Stream.generate(Math::random);
}
```
获取无限流的重点是 `Stream.generate(Supplier s)`，在例子中我们产生的是一个(0,1)之间的小数无限流。

#### 中间操作
- 元素过滤
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Stream<String> originalStream = list.stream();
    Stream<String> newStream = originalStream.filter(s -> s.startsWith("A"));
}
```
代码中 `originalStream` 为 `list` 的流，通过调用 `filter()` 方法过滤掉原流中不以字符"A"开头的元素。

- 获取前n个元素的流
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Stream<String> originalStream = list.stream();
    Stream<String> newStream = originalStream.limit(10);
}
```
调用 `limit(long n)` 方法会获取原流中前n个元素的流，如果原流中的元素个数小于n，则提前结束。该方法入参为一个 `long` 类型的参数。

- 获取跳过前n个元素的流
```
public void method() {
    List<String> list = new ArrayList<>();
    list.add("A1");
    // add some elements to list
    Stream<String> originalStream = list.stream();
    Stream<String> newStream = originalStream.skip(10);
}
```

- 获取元素唯一的流
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Stream<String> originalStream = list.stream();
    Stream<String> newStream = originalStream.distinct();
}
```
`distinct()` 方法会剔除掉原流中的相同元素。

- 元素排序
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Stream<String> originalStream = list.stream();
    Stream<String> newStream = originalStream.sorted();
    // Stream<String> newStream = originalStream.sorted(Comparator.comparing(String::length));
}
```
`sorted()` 方法会将元素进行排序并返回新流，排序元素需实现 `Comparable` 接口。或者传入一个自定义的 `Comparator` 。

- 对元素执行操作
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Stream<String> stream = list.stream().peek(s -> System.out.println("Fetching "+ s));
    stream.collect(Collectors.toSet());
}
```
`peek()` 方法会将流中的元素传递给action方法中。

#### 终结操作
- 执行方法
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    list.stream().forEach(System.out::println);
}
```
`forEach()` 将把流的每个元素都传入执行动作里，该方法返回值为 `void`。

- 获取元素个数
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    long size = list.stream().count();
}
```
单独调用 `count()` 方法的意义不大，但是可以组合一些中间操作后（例如 `filter()` ）后再调用该方法。

- 收集为数组
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    Object[] arr = list.stream().toArray();
    String[] strArr = list.stream().toArray(String[]::new);
}
```
`toArray()` 方法会返回一个 `Object[]` 数组，当我们想转化为指定类型的数组时，可传入指定的数组构造器。

- 收集为集合
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some elements to list
    List<String> newList = list.stream().collect(Collectors.toList());
    Set<String> newSet = list.stream().collect(Collectors.toSet());
}
```
