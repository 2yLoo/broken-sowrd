# Stream 扩展
第一章[《Stream 基础》](https://github.com/2yLoo/broken-sowrd/blob/master/java-base/notes/StreamBase.md)介绍了Java8 Stream的基础知识与常用打开方式。不过Java8允许我们把Stream玩的更优（花）雅（哨）。

`String` 类型的元素十分简单，为了更好的体现后文中许多方法的特性，使用新类 `Person` 作为元素类型。
```
class Person {
    /**
     * 唯一id
     */
    String id;
    /**
     * 名称
     */
    String name;
    /**
     * 年龄
     */
    int age;
    /**
     * 居住城市
     */
    String city;
    /**
     * 存款
     */
    long deposit;

    // Getter & Setter
}
```

## 并行流
并行流的处理效率更高（以空间换时间），并且 **无法保证线程安全** 。

### 获取并行流
获取并行流有两种方式：调用集合的 `parallelStream()` 方法，或者通过 `parallel()` 从指定流获取一个新的并行流。下面代码展示了并行流的获取方式：
```
public void method() {
    List<String> list = new ArrayList<>();
    // add some element to list
    Stream<String> stream = list.stream();
    Stream<String> paralletStream = stream.parallel();
    Stream<String> anotherParalletStream =  list.parallelStream();
}
```

### 顺序流 VS 顺序流
下面的例子在顺序流和并行流中做相同处理，并收集耗时结果：
```
public void method() {
    List<String> list1 = new ArrayList<>();
    List<String> list2 = new ArrayList<>();
    long node1 = System.currentTimeMillis();
    Stream.generate(Math::random).limit(1000).forEach(n -> list1.add(String.valueOf(n)));
    long node2 = System.currentTimeMillis();
    Stream.generate(Math::random).parallel().limit(1000).forEach(n -> list2.add(String.valueOf(n)));
    long node3 = System.currentTimeMillis();
    System.out.println();
    System.out.println("顺序流处理耗时：" + (node2 - node1)); // 63
    System.out.println("并行流处理耗时：" + (node3 - node2)); // 6
    System.out.println("顺序流list1元素数量：" + list1.size()); // 1000
    System.out.println("并行流list2元素数量：" + list2.size()); // 961
}
```

为了在并行流下保证现成安全，我们可以在声明list2时，做如下修改：
```
public void method() {
  // 使用现成安全的List
  List<String> list = Collections.synchronizedList(new ArrayList<>());
  long start = System.currentTimeMillis();
  Stream.generate(Math::random).parallel().limit(1000).forEach(n -> list.add(String.valueOf(n)));
  long end = System.currentTimeMillis();
  System.out.println("并行流处理耗时：" + (start - end)); // 6
  System.out.println("并行流list元素数量：" + list.size()); // 1000
}
```

## 在流中的简约操作
### 计算最大最小值
通过 `max()` （ 或`min()` ）即可获取流中最大（或最小）值。不过前提是我们需要定义比较器，例如，获取人群中年龄最大的人：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Optional<Person> optional = people.stream().max(Comparator.comparing(Person::getAge));
}
```

### 返回一个元素
通过 `findFirst()` 或 `findAny()` 可返回流中的单个元素。

获取第一个元素信息：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Optional<Person> optional = people.stream().findFirst();
}
```

### 审查流
我们也可通过 `anyMatch()` 、 `allMatch()` 与 `noneMatch()` 审查流中的元素。

例如查询人群中是否有人名为"Eco"：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Boolean hasEco = people.stream().anyMatch(p -> "Eco".equals(p.getName()));
}
```

## Optional类型
`Optional<T>` 对象是一种包装器对象，要么包装了类型T的对象，要么没有包装任何对象。正确的使用 `Optional` 可以规避头疼的空指针异常。

它有如下常用方法：

| 方法名                                                                            | 说明                                                                                             |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| T get()                                                                           | 在Optional为空时，抛出异常 `NoSuchElementException`                                              |
| T orElse(T other)                                                                 | 返回包装中的值，当Optional为空时，返回other                                                      |
| T orElseGet(Supplier<? extends T> other)                                          | 返回包装中的值，当Optional为空时，产生调用other的结果                                            |
| \<X extends Throwable> T orElseThrowable(Supplier<? extends X> exceptionSupplier) | 返回包装中的值，当Optional为空时掏出调用exceptionSupplier的结果                                  |
| void ifPresent<Consumer<? extends T> consumer                                     | 当Optional不为空时，将其传递给consumer                                                           |
| \<U> Optional\<U> map(Function<? super T, ? extends U> mapper)                    | 产生将该Optional的值传递给mapper的结果，只要Optional不为空且结果不为null，否则产生一个空Optional |
| boolean isPersent()                                                               | 如果该Optional不为空，返回true                                                                   |
| static \<T> Optional<T> of(T value)                                               | 产生具有给定值的Optional。如果value为null，抛出NullPointerException                              |
| static \<T> Optional<T> ofNuallable(T value)                                      | 产生具有给定值的Optional。如果value为null，产生一个空Optional                                    |

## 收集为Map

### 基本实现
刚使用Stream时，一直很苦恼如何将集合转化为Map。例如现在已经获取了一个 `List<Person>` 列表，希望按照id为键，对象为值的方式将其收集到一个Map中，有什么办法呢？

丑陋示例：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Person> map = new HashMap<>(people.size());
    people.stream().forEach(person -> map.put(person.getId(), person));
}
```

有没有更优雅的实现呢？答案是肯定的：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Person> map = people.stream().collect(Collectors.toMap(Person::getId, Function.identity()));
}
```
其中 `Function.identity()` 表示当前元素对象。

### 处理id冲突
在转化为Map的过程中，如果出现键冲突的情况，上述代码中，程序会抛出异常并停止运行，异常为： `java.lang.IllegalStateException: Duplicate key` 。不过 `Collectors.toMap()` 有几个变体重载方法。我们改写一下上面的方法：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Person> map = people
            .stream()
            .collect(Collectors.toMap(Person::getId, Function.identity(), (oldValue, newValue) -> newValue));
}
```
`toMap()` 中的第三个表达式表示出现key冲突时，保留旧值还是新值。

### 指定Map类型
默认情况下，通过 `collect()` 将结果收集为Map时，会将结果收集为HashMap。我们常用的Map还有LinkedHashMap、TreeMap， `toMap()`有一个重载方法允许我们指定收集的Map具体实现。
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Person> map = people
            .stream()
            .collect(Collectors.toMap(
                    Person::getId,
                    Function.identity(),
                    (oldValue, newValue) -> newValue,
                    TreeMap::new));
}
```
此例中，我们收集到的Map即为TreeMap。

### 返回线程安全的Map
我们也可调用 `Collectors.toConcurrentMap()` 方法获取一个现成安全的Map，它也有3个重载版本，传参与 `toMap()` 方法一一对应。
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Person> map = people.stream().collect(Collectors.toConcurrentMap(Person::getId, Function.identity()));
}
```

### 按条件分组/分区
Collectors中也支持将人群按照 城市 - 人群 映射关系进行分组。
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
      Map<String, List<Person>> map = people.stream().collect(Collectors.groupingBy(Person::getCity));
}
```
同样，我们也可按照年龄，甚至名称首字母进行分组，在此不进行展示。

默认情况下，我们得到的Map结构为 `<String, List<Person>>` ，若想得到一个 `<String, Set<Person>>` 结构的Map，也是可以的：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Set<Person>> map = people.stream().collect(Collectors.groupingBy(Person::getCity, Collectors.toSet()));
    System.out.println(map);
}
```

除了分组，我们还可通过 `Collectors.partitioningBy()` 方法，将人群分为满足条件与不满足条件的两个区。下例将人群按是否成年分为两部分：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<Boolean, List<Person>> map = people.stream().collect(Collectors.partitioningBy(p -> p.getAge() >= 18));
}
```

### 统计各分组集合数
下例展示了统计各城市人群数量对应关系：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Long> map = people.stream().collect(Collectors.groupingBy(Person::getCity, Collectors.counting()));
}
```

### 统计各分组指定数据
下例展示了各城市总人群存款：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Long> map = people.stream().collect(Collectors.groupingBy(Person::getCity, Collectors.summingLong(Person::getDeposit)));
    System.out.println(map);
}
```

下例展示了统计各城市首富信息：
```
public void method() {
    List<Person> people = new ArrayList<>();
    // add some elements to people
    Map<String, Optional<Person>> map = people.stream().collect(
            Collectors.groupingBy(
                    Person::getCity,
                    Collectors.maxBy(Comparator.comparing(Person::getDeposit))));
    System.out.println(map);
}
```

## 基本类型流
虽然Java支持自动拆箱、装箱操作，即支持int与Integer、long与Long等的转换。但将整数包装到对象中是很低小的。流库中有专门的类型 `IntStream` 、 `LongStream` 和  `DoubleStream` ，可用于直接存储基本类型值。在基础类型中，short、char、byte和boolean都可使用 `IntStream` ， float可使用 `DoubleStream`。

通过 `toArray()` 方法可返回基本类型数组。

产生可选结果的方法会返回一个 `OptionalInt` 、 `OptionalLong` 或 `OptionalDouble` 。与 `Optional` 的 `get()` 对应，分别具有 `getAsInt()` 、 `getAsLong()` 和 `getAsDouble()` 方法。

由于基本类型可用于计算，与对象流不同，它们具有方法sum、average、max和min，分别对应求和、求平均值、求最大值和求最小值。

并且，通过 `summaryStastics` 方法可产生一个类型为 `IntSummaryStatistics` 、 `LongSummaryStatistics` 或 `DoubleSummaryStatistics` 的对象， **同时** 统计流的总和、平均值与最大最小值。
