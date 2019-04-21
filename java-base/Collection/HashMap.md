## HashMap

HashMap是一种极高效率的Map实现，它允许键为null，但只接受一个null键。在Java7中，HashMap由 **数组 + 链表** 实现，而Java8中其实现改为了 **数组 + 链表/红黑树**。有时我们也称其数组为*桶*。

HashMap的构造器可以指定其容量与负载因子。

- 容量：容量即桶数组的size。每个桶中都存放了0-n个键值对（Entry）。桶与负载因子的乘积即为该Map的临界值，当我们向HashMap中新增Entry，超过临界值时则会自动扩容。每次扩容一倍，最大值为2^32。并且初始化时我们传入的容量会被转化为2的n次幂：
  ```
  static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
  }
  ```

- 负载因子：对桶的利用率。**负载因子的大小影响到HashMap的空间利用率和访问效率。**负载因子过低会产生许多空桶，从而造成资源的浪费；而因子过高则会造成链表过长，影响访问的效率。默认值0.75是一种空间和时间成本的折中。

以默认值容量16与负载因子0.75f来说，当HashMap实际元素个数等于12(16*0.75f)时，添加新元素Map会自动扩容为32(原容量两倍)。

为了提高HashMap的效率，在new一个HashMap对象时尽量设置好其容量，以避免或减少扩容带来的资源开销。

如果作为键的类非来自第三方库，我们在实现hashCode()方法时尽量保证其区分度，以使我们的键值对更均匀的put到桶中。并且还要考虑到hashCode()资源开销，因为在HashMap中对key.hashCode()的调用还是很频繁的。

Java8帮助我们优化了HashMap的性能，除了对桶内节点的数据结构优化，还优化了hash的计算方法：
```
// Java7 hash计算：
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
    h ^= k.hashCode();
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// Java8 hash计算：
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### Java7中HashMap实现
HashMap在Java7中的实现比较简单，存储在HashMap中的键值对名为Entry，源码中桶和Entry的定义如下：
```
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;

        ···
        // 构造器、getter、setter、hashCode、toString等方法
}
```

在我们调用get()方法时，HashMap先对key进行hash计算，并通过位与（&）当前容量来确定此key所在的桶索引位置，然后再遍历链表进行key匹配。

put()方法也会先确认传入的key是否已存在，步骤与get()相似，存在则覆盖并返回旧值，否则添加新元素，并在添加前判断是否需要扩容。有意思的是，在我们通过构造函数new一个HashMap时，table（桶数组）并没跟进初始化（Java8也是如此），而是在第一次put()时进行的初始化。

有兴趣的童靴可以从源码中查看其具体实现。😁

### Java8中的HashMap实现
相比于Java7的HashMap，Java8的HashMap效率更高（~~可惜可读性下降严重~~）。在最乐观的情况下我们访问每个key的复杂度都为1，此时每个桶内最多只有一个元素，我们只需要计算key对应的桶索引即可定位到该键值对。但事不尽如人意，很多时候我们无法控制桶中元素个数。所以其访问的复杂度就受到了桶内元素的数据结构影响。**Java8在桶中引入了红黑树（复杂度为logN），在链表（复杂度为N）长度超过8时将其优化为红黑树。**

下面是Java8中的桶与Entry部分源码：
```
transient Node<K,V>[] table;

// 当桶内节点为链表结构时
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    ···
    // 构造器、getter、setter、hashCode、toString等方法
}

// 当桶内节点为红黑树结构时
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    ···
    // 除了类似Node中的方法，还包括了红黑树的旋转、平衡等方法
}
```

其中HashMap的Entry在Java8中更名为Node，表示链表节点，TreeNode表示红黑树节点，并继承自Node（LinkedHashMap.Entry继承了HashMap.Node）。但与TreeMap（红黑树思想实现的排序树）的节点相比，它还维护了链表顺序prev/next。

> 不知道大家会不会和我一样在此联想到TreeMap，不过此处Java8中HashMap的红黑树相关代码都封装在内部类TreeNode中，足足有589行代码，虽然实现方式接近，但与TreeMap无直接联系。

源码中HashMap的put()方法如下：
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // 第一次put时进入此代码块
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果key对应的桶为空，生成新节点存入此桶
        tab[i] = newNode(hash, key, value, null);
    else {
        // 进入此代码块时我们需要对桶中的元素与key匹配
        // 如果有匹配的元素则会替换旧值，否则添加新节点
        Node<K,V> e; K k; // e：与key对应的老节点 k：
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 桶中第一个元素与key匹配赛时
            e = p;
        else if (p instanceof TreeNode)
            // 在红黑树节点下进行put操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 当在链表节点下进行put操作
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 当链表长度大于等于8时将其转化为树结构
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) {
            // 找到旧节点，执行替换逻辑
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        // 自动扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

相比Java7版本的put操作，它新增了对红黑树的判断，以及在链表向红黑树的转换。

在红黑树中put时，需要先遍历树中是否有对应key的节点，如果有，直接返回该节点，并在putVal()方法中进行值替换，否则新增一个节点，在新增节点后还需要进行红黑树的平衡，这部分逻辑与TreeMap相似。我们先不考虑红黑树的具体实现，看下源码如何在树下新增一个节点：
```
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;

        // hash值相等但key值不相等时，需要进一步向下遍历以寻找key相等的节点
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

值得注意的是，其红黑树节点不仅维护了红黑树当中的父子节点、左右子节点，还维护了双向链表结构。

put和remove都有可能破坏原有红黑树的平衡，这一部分通过内部的balanceInsertion和balanceDeletion方法来保持红黑树的平衡，在此不进行扩展，思想与TreeMap类似。在TreeMap的介绍中我们再进行平衡的展开。

### HashMap的遍历
HashMap提供了对EntrySet、KeySet、Values的访问，根据具体场景我们可以直接获取相应的Set进行遍历。直接获取键值的方法有2种：

- 通过Iterator进行遍历
```
Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
while (iterator.hasNext()) {
    Map.Entry<String, String> entry = iterator.next();
    System.out.println(entry.getKey() + entry.getValue());
}
```

- 通过for循环进行遍历
```
for (Map.Entry<String, String> entry : map.entrySet()) {
    System.out.println(entry.getKey() + entry.getValue());
}
```

我们也可通过遍历KeySet来逐个获取value值，但这样会带来更重的查询负担以及代码量，画蛇添足，完全不推荐：
```
Iterator<String> iterator1 = map.keySet().iterator();
while (iterator1.hasNext()) {
    String next = iterator1.next();
    System.out.println(next + map.get(next));
}

for (String key : map.keySet()){
    System.out.println(key + map.get(key));
}
```
