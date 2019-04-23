## Set
> 元素不重复的集合。这里元素重复是根据元素的equalse()方法的结果判断的。

### HashSet
HashSet是我们最常用的一个Set实现类。**它通过封装HashMap实现元素的唯一。**我们知道,HashMap的key是唯一的，我们向HashSet添加一个新元素时，它向内部维护的一个HashMap对象添加一个<E, new Object>的键值对。

它的初始容量为16，初始负载因子为0.75f。在我们新添加一个元素时，先判断当前实际大小是否超过 容量*负载因子，如果是，则对map进行扩容，并进行rehash（因为扩容后HashMap对象中的桶数发生变化，从而需要重新计算每个元素与桶索引的对应关系，具体请参考[HashMap]() TODOHashMap链接。）

### LinkedHashSet
LinkedHashSet对HashSet的特性做了补充，其内部的实现也由HashMap切换成了对应的LinkedHashMap。
