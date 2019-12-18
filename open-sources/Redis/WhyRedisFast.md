# 为什么Redis这么快？

![](https://redis.io/images/redis-white.png)

> Redis的数据运行在内存中，这是Redis极致性能的起点，而不是终点。

为了保证单线程Redis的高效，作者Antirez在数据结构上的优化下了很多心血。

Redis有5种基础数据结构： *string（字符串）、list（列表）、hash（字典）、set（集合）、zset（有序集合）* 。这是它向外暴露的接口，而在实现上，这几种数据结构又分别存在不同的优化空间。下面依次介绍以上几种数据结构在Redis中的已有优化，优化点由 **粗体字** 表示。

## String

Reids的string结构如下：

```
struct SDS {
  T capacity;      // 数组容量
  T len;           // 数组长度
  byte flags;      // 特殊标志符
  byte[] content;  // 数组内容
}
```

SDS是Simple Dynamic String的简称，封装的是一种可动态修改的字符串。值得注意的是 **其中的字段 `capacity` 和 `len` 使用了泛型T，当字符串较短时， `capacity` 和 `len` 可以用 `byte` 和 `short` 表示，从而节省内存** 。

虽然SDS可动态修改， **Redis在创建字符串时 `capacity` 和 `len` 一样长，不会分配冗余空间** 。

Redis的字符串有两种存储方式，在字符串较短时（字符长度不超过44字节）使用 `embstr` ，较长时使用 `raw`。

![Redis embstr & raw 存储形式](http://assets.processon.com/chart_image/5dbbda9ce4b04913a285ee2a.png?_=1572592576800)

在 `embstr` 中，RedisObject的头结构与SDS对象连续存在一起，使用 `malloc` 分配一次内存，而 `raw` 需要使用两次 `malloc` 方法，地址是不连续的。

对象头信息中存储了该对象的类型、存储形式等重要信息。
```
struct RedisObject {
  int4 type;       // 对象类型，string、list、set...
  int4 encoding;   // 存储形式，同一个类型会存在不同的存储形式
  int24 lru;       // LRU信息，用于lru与lfu内存回收策略
  int32 refcount;  // 引用计数，为0时被销毁，内存回收
  void * ptr;      // 内容指针，地址为对象内容的具体存储位置
}
```

为何区分 `embstr` 与 `raw` 的节点是44字节的字符串大小呢？

Redis使用的内存分配器 `jemalloc` 或者 `tcmalloc` 在分配内存大小时的单位都是 2/4/8/16/32/64 字节。 **如果字符串总长度超过64字节（包括对象头信息等），Redis就不再用 `embstr` 的形式存储，改用 `raw` 形式** 。

![Redis字符串存储结构](http://assets.processon.com/chart_image/5dbbd4f2e4b0ea86c41d59ad.png?_=1572599720108)

因为内存分配器为字符串一次分配为64字节时，除去Redis对象头信息的19字节，以及字符串结尾符NULL占用的1字节，还剩 64-19-1 = 44字节可用。

`string` 另一点优化就是， **在字符串长度小于1MB时，每次扩容采用加倍策略；当字符串长度超过1MB，每次扩容只会新增1MB的冗余空间** 。

## List
在Java中，最常用的List实现分为两种：数组与链表。Redis中的 `list` 是基于链表实现的，这样保证了较好的 `push` 、 `pop` 性能。具体的说，是 *快速链表（quiklist）* 。快速链表的节点包装了一个 *压缩列表（ziplist）* 。

> 由于Redis中 `list` 由链表实现，所以在实际使用过程中，要慎用 `lindex` 、 `ltrim` 、 `lrange` 等随机查询的命令

![Redis快速链表结构](http://assets.processon.com/chart_image/5dbbf64fe4b0ece75946e81c.png?_=1572847889385)

`quicklist` 结构如下：
```
struct quicklist {
  quicklistNode* head;  // 指向第一个quicklist节点
  quicklistNode* tail;  // 指向最后一个quicklist节点
  long count;           // 元素总数
  int nodes;            // ziplist节点个数
  int compressDepth;    // LZF算法压缩深度
  ...
}

struct quicklistNode {
  quicklistNode* prev;  // 指向前一个quicklist节点
  quicklistNode* next;  // 指向后一个quicklist节点
  ziplist* zl;          // 指向包装内的ziplist
  int32 size;           // ziplist的字节总数
  int16 count;          // ziplist重点元素数
  int2 encoding;        // 存储形式，标志该节点为原生字节数组还是LZF压缩存储
  ...
}
```

优化： **在 `list` 元素较少时，所有元素存储在一个ziplist中，只有当数据量较多时才改成快速链表** 。
> 由于链表需要额外的空间存储前后两个指针，会存在空间浪费，加重内存的碎片化，所以使用链表与 `ziplist` 结合的方式，保证了写的性能，也不会出现太大空间冗余。

### ZipList的实现原理
在 `ziplist` 中，所有元素紧挨着一起存储，分配一块连续的内存。它在Redis中不仅用在 `list` 中， `hash` 和 `zset` 中也有所运用。

![Redis ziplist存储结构](http://assets.processon.com/chart_image/5dbbf659e4b0893e9a6da977.png?_=1572599739040)

`ziplist` 数据结构如下：
```
struct ziplist {
  int32 zlbytes;        // ziplist占用字节数
  int32 zltail_offset;  // 最后一个元素相对列表起始位的偏移量，用于定位最后一个节点
  int16 zllength;       // 元素个数
  T[] entries;          // 元素内容数组
  int8 zlend;           // 结束符标志，在ziplist最末位置
}
```

其中字段 `zltail_offset` 存在的目的是为了快速定位最后一个元素，以便逆序遍历。

`ziplist` 中 `entry` 的结构如下：
```
struct entry {
  int<var> prevlen;         // 前一个 entry 长度
  int<var> encoding;        // 元素类型编码
  optional byte[] content;  // 元素内容
}
```

字段 `prevlen` 表示前一个 `entry` 的字节长度，在逆序遍历时用于快速定位元素位置，如果前一个元素内容小于254字节， `prevlen` 用1个字节存储，否则用5个字节存储。

> 由于相邻元素之间的耦合，使得 `ziplist` 中前一个元素的大小被修改时，可能发生级联更新。例如某个 `entry` 的大小由253字节变成254字节时，后一个 `entry` 的 `prevlen` 字段需要从1个字节扩展为5个字节。如果后一个 `entry` 大小也为253，则它后面的 `entry` 需要继续更新，直到信息同步为止。

字段 `encoding` 存储了元素内容的编码类型信息， `ziplist` 通过该字段决定 `content` 的形式。字段 `content` 是可选择的，当该 `entry` 用于存储很小的数时(1~13)，内容并非存在 `content` 中，而是在 `encoding` 的后4位。此时 `encoding` 的格式为 `1111xxxx`。
> *为什么不是 0~15？*
>
> 上面提到的 `1111xxxx` 中 `xxxx` 的范围为 0001~1101，因为0000、1110、1111在encoding中结合前4位会代表其他元素类型。

`ziplist` 是十分紧凑的，没有冗余空间，添加元素时，需要调用 `realloc` 扩展内存。根据内存分配器算法和当前的 `ziplist` 内存大小， `realloc` 可能重新分配内存空间，或在原有地址上扩展。

> 当 `ziplist` 占据内存较大，重新分配内存会产生很大的消耗，所以 `ziplist` 不适合存储大型字符串，也不宜拥有过多元素。

## Hash
Redis的 `hash` 与Java中的 `HashMap` 相似，由 数组 + 链表 实现，也称为 `hashtable`。

> 在Java中，数组中的元素被称为“桶”，在Java8中，当“桶”中元素过多，为提高性能，会使用红黑树替代链表。这一点目前未在Redis中体现。

在Java中， `rehash` 行为是一次性动作。Redis为追求高性能，采用的是 **渐进式`rehash`** 策略。

它维护了两份 `hashtable` —— 旧数组与新数组。在 `rehash` 过程中，会将旧数组中的元素逐步迁移到新数组中，以保证在 `rehash` 发生时可以继续提供服务。

![Redis hash存储结构](http://assets.processon.com/chart_image/5dbfd5e9e4b06b7d7021c8c3.png?_=1572853681265)

并且， **当hash元素较少时，使用 `ziplist` 进行存储** ，其中 `key` 与 `value` 作为两个 `entry` 相邻存储。

## Set
**在 `set` 集合容纳的元素都是整数并且元素个数较少时，Redis会使用 `intset` 来存储集合元素** 。 `intset` 是紧凑的数组结构，支持16位、32位和64位的整数。

![Redis intset存储结构](http://assets.processon.com/chart_image/5dbfd007e4b0ea86c420386e.png?_=1572851847479)

```
struct intset<T> {
  int32 encoding;   // 决定整数位宽为16、32还是64位
  int32 length;     // 元素个数
  int<T> contents;  // 整数数组
}
```

当 `set` 中元素较多，或向 `set` 中添加非整数值时，会使用 `hashtable` 存储，值统一为 `NULL`。

## ZSet
在 `zset` 元素较少时，Redis同样使用 `ziplist` 存储，其中 `value` 与 `score` 作为两个 `entry` 相邻存储。否则使用复合结构 `hash` + `skiplist` 存储。其中 `hash` 用于存储 `value` 与 `score` 的对应关系，而排序功能由 `skiplist` 实现。

`skiplist` 称为跳跃链表，可高效率的维护一个有序元素集合。

一个比较理想的跳跃列表结构如下：

![Redis skiplist存储结构](http://assets.processon.com/chart_image/5dbfdb83e4b0c555374ce245.png?_=1572856873522)

```
struct zslnode {
  string value;         // 节点值
  double score;         // 节点分数
  zslnode*[] forwards;  // 多层连接指针
  zslnode* backward;    // 回溯指针
}

struct zsl {
  zslnode* header;           // 跳跃列表头指针
  int maxLevel;              // 跳跃列表当前的最高层
  map<string, zslnode*> ht;  // hash结构的所有键值对
}
```

在Redis中，每个节点可上升的层数由随机计算得出，获取越高层的概率越低。

在 `skiplist` 中节点的查找过程如下：
![Redis skiplist查找过程](http://assets.processon.com/chart_image/5dbfe58fe4b04913a2890198.png?_=1572857965086)

理解起来比较绕，中心思想其实是在做一个循环——记节点在A层的下一个元素为A.next， 节点在A层的下一层为A'，伪代码表示为：
```
find(x) {
  node A,B;
  boolean find = false;
  for(A = head; find; ) {
    B = A.next;
    if (B > x) {
      A = B';
    } else {
      A = A';
    }
  }
}
```
> 代码中未补充循环结束条件（已找到 or 找不到）。

`skiplist` 的实现不在本文讨论中。之前看Java源码的时候，`TreeMap`基于红黑树思想实现了可排序的功能。为何Redis不采用平衡树还是选择了 `skiplist` 呢？

Redis作者 Antirez 是这么说的：
> There are a few reasons:
>
> 1) They are not very memory intensive. It’s up to you basically. Changing parameters about the probability of a node to have a given number of levels will make then less memory intensive than btrees.
>
> 2) A sorted set is often target of many ZRANGE or ZREVRANGE operations, that is, traversing the skip list as a linked list. With this operation the cache locality of skip lists is at least as good as with other kind of balanced trees.
>
> 3) They are simpler to implement, debug, and so forth. For instance thanks to the skip list simplicity I received a patch (already in Redis master) with augmented skip lists implementing ZRANK in O(log(N)). It required little changes to the code.

我的理解如下：
> 改变参数时，红黑树需要保持平衡， `skiplist`在性能上表现更好
>
> Redis对内存的“斤斤计较”没必要使用红黑树

本总结参考自老钱的《Redis深度历险》。
## 参考文档
- [[个] 《Redis深度历险》](https://juejin.im/book/5afc2e5f6fb9a07a9b362527)
