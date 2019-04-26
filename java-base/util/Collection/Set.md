## Set
> 元素不重复的集合。这里元素重复是根据元素的equalse()方法的结果判断的。

### HashSet
HashSet是我们最常用的一个Set实现类。**它通过封装HashMap实现元素的唯一。**我们知道，HashMap的key是唯一的，我们向HashSet添加一个新元素时，它向内部维护的一个HashMap对象添加一个<E, new Object>的键值对。

与HashMap相同，它的初始容量为16，初始负载因子为0.75f。关于容量与负载因子的关系，具体请参考[HashMap](https://github.com/2yLoo/broken-sowrd/blob/master/java-base/util/Map/HashMap.md)。

而它对Set接口的实现，即在内部转为对HashMap中key的方法操作。

### LinkedHashSet
LinkedHashSet对HashSet的特性做了补充，与HashSet相似，LinkedHashMap的底层实现依赖于LinkedHashMap。但我们可以指定LinkedHashMap的顺序（插入有序or访问有序），而LinkedHashSet只能做到插入有序。
