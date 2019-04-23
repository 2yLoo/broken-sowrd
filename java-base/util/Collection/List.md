## List
> 元素有序的集合，通过整形索引从集合中访问元素。

开发时，有一个很常见的代码，相信大家都有摸过。那就是:

```List<Object> list = new ArrayList<>()```

以这样的方式我们可以创建一个数组List。作为一个继承（扩展）了Collection的接口，除了Collection中提供的方法，它规定了集合的顺序性。给集合中的元素添加了“顺序”的概念，因此我们可以通过索引来对集合进行访问与操作。

### ArrayList
ArrayList是我们接触和应用最多（≠能力最强）的一个List实现类。

它通过**数组**维护了集合的顺序性。集合中的元素地址是连续的。因此当我们进行随机访问时它提供了很好的性能。但如果我们删除的元素e在List的头部或中部，e后的元素在地址上都需要进行一次迁移，带来了更高的消耗，插入操作同理。所以在选择List的实现类时，我们需要考虑对其是否随机删除、插入频繁，如果是，也许我们应该考虑使用下面的LinkedList来对其进行实现。

### LinkedList
LinkedList与ArrayList构成我们最常用的2大List实现类。它的元素通过双向链表进行顺序的维护。在size相同时，它需要的内存资源比ArrayList略大。

它的元素结构如下：
```
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
next和prev即为Java中的引用，其中next保存当前节点的下一个节点对象地址，prev则相反，以这样的结构即可实现双向链表。

显而易见，它的随机访问效率会低于ArrayList，例如我们访问第10位元素时，它是从第1位开始往后查找的。但它的实现方式也给它带来随机删除的高效率。在元素充足的情况下，当我们删除第10位元素时，只需将第9位的元素next指向第11位元素，并将第11位元素的prev指向第9位。

## Tips
### 容量
在阿里的编程规范中，我们在对ArrayList的声明时应尽量指定其容量大小，虽然ArrayList实现了自动扩容（默认初始容量为10，每次扩容1.5倍，最大容量为2^32），但如果我们很肯定List的实际大小，我们在后面的操作时可以避免自动扩容，从而节省资源开支。另一方面，当我们的容量已经达到1000，而接下来我们只会向其再多添加1个元素，自动扩容会帮我们把容量扩充到1500，从而造成499个元素空间的不必要开销。而LinkedList的链表特性并不需要我们关注其容量，我们可以不断的向链表添加新的元素。

我认为，虽然现在硬件条件越来越好，作为一名有理想的开发人员，还是应该保持对硬件资源的敬畏与拮据，选择最适合我们的方式实现我们的需求。

### 选择

曾经在一个代码优化过程中，原List使用了ArrayList实现，但在后面的引用中，除了全量的参数传递，全是对该List的随机删除，我果断弃用了ArrayList，转而用LinkedList实现，好在面向接口编程的过程中，需要改动的代码不多，准确的说，是一行：

```
// List<Object> list = new ArrayList<>()
List<Object> list = new LinkedList<>()
```
从这个例子中我们也可体会到面向接口编程的优雅。

也许有时候我们无法找到十全十美的解决方案，但至少我们应该从已有的解决方案中挑选最适合我们的一个。

在我们使用List时有很多实现类的选择，最基本的该选择ArrayList还是LinkedList也值得我们好好思考。从实际需求出发，我们是否需要在队中或队尾的进行频繁访问（ArrayList）？我们是否在List中部有频繁的删除、新增操作(LinkedList)？或者我们仅需要一个List进行有序访问（ArrayList，因为它的开销比LinkedList更小）？或者我们需要一个线程安全的List···

### 遍历
除了实现类需要我们做出选择，遍历的方式也值得我们留心。我们有好几种方式实现对List的遍历：
```
// 1. 普通for循环
for (int i = 0; i < list.size(); i++) {
  System.out.println(list.get(i).toString());
}

// 2. foreach，内部使用迭代器遍历
for (Object o : list) {
  System.out.println(o.toString());
}

// 3. iterator迭代器
Iterator<Object> iterator = list.iterator();
while(iterator.hasNext()){
   Object next = iterator.next();
   System.out.println(next.toString());
}
```

第一种方式显然更适合ArrayList。当我们在LinkedList上使用第一种for循环，如果我是第100位元素，访问到我前还需向前访问99个元素，我想我会疯掉，更不用说我后面的元素了！第二种方式与第三种方式都使用了迭代器，对LinkedList的支持是非常友好的。

### SubList
我们可以通过subList()方法获取指定List的子列表。但是需要注意的是，我们对子列表的所有操作实际都会操作到父列表上。用实例证明：
```
public static void main(String[] args) {
    List<String> list = new ArrayList<>(); // 切换成LinekdList后结果不变
    for (int i = 0; i < 10; i++) {
        list.add("Father" + i);
    }
    System.out.println(list);
    List<String> sub = list.subList(4, 6);
    for (int i = 0; i < sub.size(); i++) {
        sub.set(i, "Son" + i);
    }
    sub.add("new")
    System.out.println(list);

}

// 控制台输出：
[Dad0, Dad1, Dad2, Dad3, Dad4, Dad5, Dad6, Dad7, Dad8, Dad9]
[Dad0, Dad1, Dad2, Dad3, Son0, Son1, new, Dad6, Dad7, Dad8, Dad9]
```

### 排序
当我们需要对List根据指定规则进行排序时，可以直接调用sort()方法。并传入我们实现的Comperator比较器。而此方法委托Arrays类的sort方法执行排序，Arrays是一个很强的类，在此不进行展开，有兴趣的童靴可以跳转到[Arrays排序分析]()TODO 补充此处。
