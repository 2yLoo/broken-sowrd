# MongoDB简介
MongoDB是一款代表性很强的NoSQL数据库，进一步说，MongoDB是一个文档型数据库。不同于传统关系型数据库中的行/列的概念，在MongoDB中，数据被保存在一个类JSON的文档结构中。

例如保留一个学生的数据，在关系型数据库中，它可能是这样的：

| id  | name | sex  | age |
| --- | ---- | ---- | --- |
| 1   | eco  | male | 20  |

但在MongoDB中，对应的文档可能是这样子的：

```
{
  "_id": ObjectId("5fb2a5ba17d3c67fdffab06a"),
  "name": eco,
  "sex": "male",
  "age": 20
}
```

MongoDB有许多特性，我认为最大的特点就是 **灵活、天生的分布式** 。

- 灵活

  在传统型数据库中，新增一个纬度的数据需要预先修改表，再进行插入/更新。但在MongoDB中，对数据的结构没有那么强力的约束，两个文档之间的结构可以是完全不同的，并且可以内嵌文档。

  > 这是一把双刃剑，滥用这个能力会给项目、数据库带来一团糟。

- 分布式

  MongoDB是一个天生的分布式数据库，它支持副本集、分片， **Mongo会为每一条文档默认创建一个 `_id` 字段，它是由时间戳、机器、进程ID、计数器4个纬度的因素决定的** ，因此不用担心在分布式部署时，插入的两份文档持有相同 `_id` 的问题。

同时作为数据库它也支持索引，还有一些其他能力，会在后面继续介绍。

# MongoDB安装与部署

我一直认为最好的安装向导就是来自官网的那一份。一般官方的安装文档都是简洁、快速实践的风格，因此这里首先推荐[MongoDB安装指南](https://docs.mongodb.com/manual/administration/install-community/)。

其次在Docker生态上推荐[Docker Hub Mongo安装指南](https://hub.docker.com/_/mongo)。

## Mac安装与部署

在Mac上推荐使用Docker进行安装与部署，启动Docker后，直接在终端执行`docker pull mongo`拉取MongoDB最新、最稳定的版本镜像即可。latest版对于简单的上手与使用来说都已经足够了。

镜像拉取完成后可以通过命令`docker images`查看当前机器下的所有镜像：
```
➜  ~ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mongo               latest              bcef5fd2979d        10 months ago       386MB
```

然后执行命令`docker run -d -p 27017:27017 --name my-mongo mongo`启动MongoDB容器，该命令会根据刚拉取的mongo镜像部署一个新的容器，容器名为"my-mongo"，对外暴露端口27017，至此MongoDB的部署就完成了。

部署完成之后，执行`docker exec -it my-mongo mongo`即可连接到刚启动的MongoDB服务。

# MongoDB基础

MongoDB会有一些不同于传统SQL的专有名词，但描述的内容是可以通过联想记忆的：

| MySQL     | MongoDB          |
| --------- | ---------------- |
| 行(row)   | 文档(Document)   |
| 表(table) | 集合(Collection) |
| 数据库    | 数据库           |

## 文档

文档就是 **键值对** 的一个 **有序集** 。数据是以K-V的形式聚集在同一个文档中。其中值可以是字符串、数字、数字、内嵌文档等类型。

以这个文档的例子来说：

```
{
  "_id": ObjectId("5fb2a5ba17d3c67fdffab06a"),
  "name": eco,
  "sex": "male",
  "age": 20
}
```

`"_id"、"name"、"sex"、"age"`都是键(Key)，而它们的值分别为冒号后的内容。这种格式的可读性很好。需要注意的是键是大小写敏感的，`name`和`Name`属于两个不同的键，并且文档中不能存在两个相同的键，而值是区分类型的，`20`是数字，`"20"`是字符，两者的类型不同。

有序集表明下面两个文档是不能划等号的：
```
{
  "name": eco,
  "sex": "male"
}

{
  "sex": "male",
  "name": eco
}
```

## 集合

集合是文档的上一级概念。一个集合内会存在多个文档，之前提到MongoDB的灵活性，在同一个集合中，文档的结构是可以不同的。但异构的文档并不适宜存储在同一个集合中，有以下几个原因：

- 不易维护

  当集合中的文档杂乱无章，也就失去了管理的意义。在同一个集合中读取异构的文档对应用层来说是很头大的一件事。

- 查询缓慢

  由于所有文档都聚合在一个集合中，查询到指定一个文档时，需要牺牲许多时间排除掉完全不符合要求的文档（例如在一群学生、动物的文档中找到具体某一个学生，不相关的动物们也会被纳入搜索范围）。

- 索引模糊

  索引是优化查询的常用手段，但在异构的文档中建立合适的索引是一个难以实现的事情。

MongoDB中的集合是存在命名规范的，"system."是系统保留的集合名前缀，对于复杂的存储设计，可以通过符号"."对集合名进行管理，如"ugc.picture", "ugc.video"两个集合可以表示为用户生产内容中的图片子集与视频子集。

在向新集合插入文档时，是不需要提前创建集合的，在第一次执行insert后集合会被自动创建。

> 虽然可以使用 UgcPicture、ugc_picture这样的命名格式，但这样在MongoDB中会显得有些突兀。

## 数据库

数据库用来管理集合。在数据库的命名上，禁止使用"."等特殊字符（否则 "mydb.user.student.grade"中数据库名到底是"mydb"还是"mydb.user"呢？）

在MongoDB中有几个系统级的数据库：

- admin

  添加到admin数据库中的用户，会拥有其他所有数据库的操作权限，后续会介绍到MongoDB中的权限系统，可以创建只拥有某个数据库权限的用户。

- local

  在MongoDB的副本集里，从机会复制主机的数据，但local中的数据属于实例本地，不会在副本集中被复制。

- config

  在MongoDB中，分片信息会存储在config数据库中。

可以通过命令`use {$db_name}`来使用指定数据库，如`use my_db`。如果该数据库不存在，在第一次执行插入文档时，会被自动创建。

## 数据类型

### BSON，数据存储的基础

MongoDB使用类JSON存储数据，称之为BSON。

> JSON：JavaScript Object Notation，从JavaScript中发扬出来，具有很高的可读性、灵活性，被广泛应用在API、配置文件、日志、存储等方面上。
>
> BSON：Binary JSON，轻量级的二进制JSON格式，支持更丰富的数据类型，是数据库可理解的、落盘的格式。

从表现上看，BSON支持更多的数据类型，例如日期，多种多样的数字（浮点、32位整形、64位整形甚至128位整形）。更本质上来说， **JSON属于文本类型，而BSON的存储是二进制**。

为何MongoDB会选择BSON而不是JSON呢？

- 更小的存储成本

  BSON比JSON占用的存储空间更小，BSON是经过序列化后的二进制，而JSON是字符串。

- 更低的索引成本

  BSON序列化后会在头部存储该文档的长度，因此在查询时，可以很迅速的跨越到下一个文档，而JSON只能从头到尾遍历。查询是衡量数据库能力的一项重要指标，BSON能带来更快的查询速度。

- 更多的数据类型

  日期是第一个例子，JSON中的日期是字符串，对于计算、转换、修改而言都是重伤，而BSON中存储的是实在的日期对象，这是存储场景中很大的一个加分项。另一个加分项是数字类型，BSON支持比JSON更丰富的数据类型，例如浮点、双精度浮点、布尔值、32位/64位/128位整形，这使MongoDB会更贴切应用层，可以满足更多的应用场景。

- 高效的编解码

  BSON可以快速进行编码与解码。这一点对于应用程序来说十分重要，高效的编解码的速度是选择MongoDB的关键原因之一，但也十分隐晦。

BSON支持的数据类型包括以下几种：

| 数据类型   | 说明 ｜                                                                                 |
| ---------- | --------------------------------------------------------------------------------------- |
| null       | 表示空值或不存在的字段                                                                  |
| 布尔型     | 只存在true/false两种类型                                                                |
| 数值       | **在shell中默认使用64位的浮点型数值** ，还有NumberInt(32位)、NumberLong(64位)等具体类型 |
| 字符串     | 通过双引号""扩起来的字符串                                                              |
| 日期       | 内部为存储为时间戳，从UTC时间 1970/1/1 00:00:00 开始计算的 **毫秒数**                   |
| 正则表达式 | 可用以作为查询条件                                                                      |
| 数组       | 由[]表示，可以内嵌数组、数组中的元素可为不同类型                                        |
| 内嵌文档   | {"profile":{"name":"eco", "age":11}}，禁止套娃！                                        |
| 对象id     | 一个12字节的ID，常作为文档的唯一标识                                                    |
| 二进制数据 | 任意字节的字符串                                                                        |
| 代码       | 查询和文档中可以包含JavaScript代码                                                      |

我们在使用mongo客户端或在应用中通过MongoDriver读写MongoDB时，是经过了序列化与反序列化的步骤，才完成了“可读性BSON”与真实的二进制BSON之间的转换。 **数据在MongoDB服务中，是以二进制的形式接收、传输与存储的** 。

> 推荐阅读：[MongoDB-JSON与BSON]([https://www.mongodb.com/json-and-bson])

### ObjectId，分布式基础

ObjectId是文档中 "\_id" 的默认类型（在插入新文档时无需指定"\_id"，MongoDB会自动生成该字段）。在分布式系统中，相较于常规的自增主键id，ObjectId更轻巧的避免了主键冲突的情况。在A、B两个服务中避免自增主键id冲突的方式有许多，但都有一定的局限性：

- 全局同步

  在并发时的同步开销成本与效率是很大的短板。

- 给A、B预留不同的id段

  给A预留[0, 1000)的id段，给B预留[1000, 2000)的id段…容易入侵业务，不便扩展与维护。

- 给A、B不同的起始值

  A的id从0,2,4,6,8等差递增，B的id从1,3,5,7,9等差递增，伸缩性差，入侵业务。

ObjectId给了MongoDB作为分布式数据库的底气。它从以下几个方面保证唯一性：

| 维度   | 所占字节 | 表现                                   | 说明                                                                 |
| ------ | -------- | -------------------------------------- | -------------------------------------------------------------------- |
| 时间戳 | 4字节    | 不同时间的数据id必不相同               | 32位时间戳，单位为秒                                                 |
| 机器   | 3字节    | 不同机器上的id必不相同                 | 实例所在机器的机器名散列值                                           |
| 进程ID | 2字节    | 相同机器上不同进程（实例）的id必不相同 | 进程的唯一标识符                                                     |
| 计数器 | 3字节    | 同一实例同一时间的id必不相同           | 保证单个实例同一秒内插入的2^24个文档拥有不同id，目前远远达不到此规格 |

其中id的高4字节为时间戳，可近似保证秒级的插入顺序，但应用侧不应通过这4个字节获取文档的插入时间，更合理的方式是维护一个类似"createTime"的字段进行表示。

# 操作MongoDB

## 善用help

使用shell操作MongoDB时，help是一个很强的指令。直接在命令行输入help可以查看相关的操作帮助。另一个技巧是，发送带()版本的指令，会执行具体操作，而发送不带括号的版本，则会输出该方法的定义到终端：

```
> db.coll.findOne()
{
	"_id" : ObjectId("5fb2a5ba17d3c67fdffab06a"),
	"x" : "c",
	"y" : 99.5,
	"z" : 1.5,
	"a" : [
		11,
		12
	]
}
> db.coll.findOne
function(query, fields, options, readConcern, collation) {
    var cursor = this.find(query, fields, -1 /* limit */, 0 /* skip*/, 0 /* batchSize */, options);

    if (readConcern) {
        cursor = cursor.readConcern(readConcern);
    }

    if (collation) {
        cursor = cursor.collation(collation);
    }

    if (!cursor.hasNext())
        return null;
    var ret = cursor.next();
    if (cursor.hasNext())
        throw Error("findOne has more than 1 result!");
    if (ret.$err)
        throw _getErrorWithCode(ret, "error " + tojson(ret));
    return ret;
}
```

## 创建 & 插入

使用`db.{$collection_name}.insert(...)`可以创建一条文档，其中`{$collection_name}`为具体的集合名，"..."中填入类JSON格式的键值对字符串，如`"name":"eco"`。insert区分大小写。

在执行`db.coll.insert{"name":"eco"}`时，会在集合"coll"中插入一条"name"为"eco"的数据。在MongoDB中，集合是可以自动创建的，如果不存在"coll"集合，则会自动创建。

在创建一条文档时如果没有指定其"\_id"字段，MongoDB会自动分配一个ID。如果之前接触关系型数据库较多，可能会偏向使用自增类型的主键id，但在MongoDB中不推荐这么做，除非有特定的业务场景要求。

像MySQL一样，也可以为集合添加一些特定索引，添加索引的内容在新篇章中介绍。

## 删除

使用`db.{$collection_name}.remove(...)`可以删除指定文档。大多数时候我们要删除的都是满足特定条件的文档，例如"name"等于"eco"。因此在删除命令中，需要添加删除（查询）条件，用类JSON表达式进行过滤。

执行`db.coll.remove({"name":"eco"})`会删除集合"coll"中"name"等于"eco"的所有文档。在其他MongoDB基本操作，如修改中，也会常用这种方式。

如果执行完删除后，该集合中已经没有文档，则该集合还会暂时保留。

如果需要删除整个集合，可以使用`drop()`命令，如`db.coll.drop()`会删除整个coll集合以及其中的文档。

## 修改

使用`db.{$collection_name}.remove(...)`可以更新指定文档。与删除不同的是，在方法的"..."中，除了要传入查询条件，还有需要执行的更新动作。

执行`db.coll.update({"name":"eco"},{"age":2})`会找到集合"coll"中一条名称为"eco"的文档，并将这条文档的 **整个内容替换** 为`{"age":2}`：

```
// 先插入一条文档 {"name":"eco","age":1}
> db.coll.insert({"name":"eco","age":1})

// 通过查询验证插入结果，并获得它的_id
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe754043466cdeb2152fae5"), "name" : "eco", "age" : 1 }

// 此时通过上述语法更新文档
> db.coll.update({"name":"eco"},{"age":2})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

// 通过查询 _id 字段验证我们的更新结果："name"字段不见了，"age"被更改为了2
> db.coll.find({"_id" : ObjectId("5fe754043466cdeb2152fae5")})
{ "_id" : ObjectId("5fe754043466cdeb2152fae5"), "age" : 2 }
```

这通常不是我们要的更新结果，如果我们期望的结果是`{"name":"eco","age":2}`，则应该使用另一种语法——修改器。

### 修改器

我们可能希望更新某一个具体字段的值，而不是替换全部文档文档。除此之外，在一些特定的修改场景，例如希望删除某一个字段，或者更新文档中的数组内容，修改器会非常有用。

#### $set 修改字段值

"$set"可以用于更新某一个字段的值，如果该字段不存在，则会直接新增该字段。改写上面的例子，则具体的命令为：`db.coll.update({"name":"eco"},{"$set":{"age":2}})`，看下实际的执行效果：
```
// 准备一条文档进行演示
> db.coll.insert({"name":"eco","age":1})

// 查询该文档内容
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 1 }

// 通过 $set 修改器修改 "age" 字段的值
> db.coll.update({"name":"eco"},{"$set":{"age":2}})

// 查询该文档更新后的内容："name"字段不变，"age"的值更新为了2
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 2 }
```

#### $unset 删除字段

"$unset"可以用于删除文档中的具体字段。

执行`db.coll.update({"name":"eco"}, {"$unset":{"age":3}})`会找到集合"coll"中"name"为"eco"的文档，并将其"age"字段删除。

> 删除字段时，需要填入完整的K-V内容，但对这个字段的值不会进行校验，即使填的是错误值也没有关系。在上面的命令中，即使匹配到的文档"age"值不为3，也不会影响执行结果。我认为这可能是MongbDB为了更好的利用通配的解析命令函数，所以才规定必须填入K-V结构，否则命令解析失败。

#### $inc 处理数值类型的增加与减少

"$inc"可以用于更新文档中的数值类型（整形、长整型、双精度浮点型）字段，如果字段不存在，MongoDB会新建字段并赋值为增加/减少的值。

执行`db.coll.update({"name":"eco"}, {"$inc":{"age":1}})`会找到集合"coll"中"name"为"eco"的文档，并将其年龄"age"加1。

> 有人可能会疑问为什么有"$set"了还需要"$inc"，我认为"$inc"是有必要的。它是在既有的数值字段上进行增量修改，并不需要提前知道该字段目前的值。例如某人在生日过后需要将年龄+1，如果使用"$set"，需要分三步进行：获取他的当前年龄 > 将当前年龄加1 > 将新的年龄写入MongoDB。而如果使用"$inc"，则只有一步：将他的当前年龄加1。

### 修改器高阶玩法

其实以上的几个修改器特性已经可以满足大部分场景的需求，但在MongoDB中支持保存数组，所以针对数组，衍生了几个更特别的修改器："$push"、"$each"、"$slice"、"$sort"、"$addToSet"、"$pop"和"$pull"。

#### $push 向数组末尾添加元素
"$push"向数组字段中添加一个元素，如果该字段不存在，则新增数组字段，并将元素添加到其中。

执行`db.coll.update({"name":"eco"}, {"$push":{"qq":12345}})`会找到集合"coll"中"name"为"eco"的文档，并将12345添加到"qq"中。

#### $each 向数组批量添加元素
如果需要添加多个"qq"，则可以组合"$push"与"$each"形成新命令：`db.coll.update({"name":"eco"}, {"$push":{"qq":{"$each":[11111,22222]}}})`。这样会将两个号码11111,22222都添加到"qq"中。`update()`的第二个参数已经开始变复杂，为了更好的观察其结构，可以将其进行JSON格式化：
```
{
	"$push": {
		"qq": {
			"$each": [11111, 22222]
		}
	}
}
```

> 这是本文的第一次将参数内容初始化，目的是为了方便观察各个修改器在修改语句中的角色，帮助理解修改器语法，后文也会多次使用这种方式展示方法参数。

#### $slice 保留数组中后n位元素
如果一个账号只能保留3个"qq"，则在下一次添加新"qq"时，可以通过组合"$push"与"$slice"的方式执行命令：`db.coll.update({"name":"eco"}, {"$push":{"qq":{"$each":[33333,44444], "$slice":-3}}})`，同样将参数格式化的结构如下：
```
{
	"$push": {
		"qq": {
			"$each": [33333, 44444],
			"$slice": -3
		}
	}
}
```

#### $sort 对数组中的元素进行排序
$sort也可与$slice一起使用，-1表示逆序排序，1表示正序排序，执行`db.coll.update({"name":"eco"}, {"$push":{"qq":{"$each":[11111,11112], "$slice":-3, "$sort":-1}}})`会将数字批量插入到"qq"中，再进行排序，最后裁剪到只剩末尾3个元素，格式化后：
```
{
	"$push": {
		"qq": {
			"$each": [11111, 11112],
			"$slice": -3,
			"$sort": -1
		}
	}
}
```

#### $addToSet 添加不重复的元素到数组
$addToSet只会将原数组中不存在的元素插入到其中，但如果原数组中已存在两个相同的元素，并不会剔除原数组中的重复元素。我们先初始化一下"qq"的值：
```
// 先删除qq字段
> db.coll.update({"name":"eco"},{"$unset":{"qq":1}})

// 向qq字段批量插入3个数，其中两个相同
> db.coll.update({"name":"eco"}, {"$push":{"qq":{"$each":[11111,11112,11111], "$slice":-3, "$sort":-1}}})

// 获取现在的文档数据
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 3, "qq" : [ 11112, 11111, 11111 ] }
>
```

$addToSet可与$each搭配使用，进行批量插入：
```
// 向qq数组批量插入3个数，其中11111与11112在原数组中已存在
> db.coll.update({"name":"eco"}, {"$addToSet":{"qq":{"$each":[11111,11112,11113]}}})

// 查看插入结果，只有11113被添加进去
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 3, "qq" : [ 11112, 11111, 11111, 11113 ] }
>
```

#### $pop 删除数组中的头/尾元素
$pop会将数组中的头部或者尾部 **1** 个元素从数组中删除，具体删除位置与传参有关。

假设我们当前的文档内容如下：
```
{
	"_id": ObjectId("5fe758433466cdeb2152fae6"),
	"name": "eco",
	"age": 3,
	"qq": [11111, 11112, 11113, 11114]
}
```

执行`db.coll.update({"name":"eco"}, {"$pop":{"qq":1}})`会将数组尾部的11114从数组中删除。

执行`db.coll.update({"name":"eco"}, {"$pop":{"qq":-1}})`会将数组头部的11111从数组中删除。

#### $pull 删除数组中匹配的元素
$pull会将数组中，所有匹配的元素全都从数组中剔除。

执行`db.coll.update({"name":"eco"}, {"$pull":{"qq":11111}})`会找到集合"coll"中一条名称为"eco"的文档，并将"qq"数组中所有值为1111的元素全都剔除。

#### 访问数组中的指定索引
假设我们当前的文档内容如下：
```
{
	"_id": ObjectId("5fe758433466cdeb2152fae6"),
	"name": "eco",
	"age": 3,
	"qq": [11111, 11112, 11112, 11113]
}
```

如果我们希望修改"qq"中第4个元素的值，可以执行`db.coll.update({"name":"eco"}, {"$inc":{"qq.3":2}})`将11113加2。其中"qq.3"表示数组qq的第3个索引元素（数组下标以0开头）。

如果数组中有多个匹配查询条件的元素，而只想更改第一个匹配查询条件的元素值，则可利用$表示查询匹配的第一条。执行`db.coll.update({"qq":11112}, {"$inc":{"qq.$":2}})`可以查询到数组qq中包含11112的文档，并将数组中第一个11112的值加2。

结合上面的命令执行结果如下：
```
// 查询当前文档的内容
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 3, "qq" : [ 11111, 11112, 11112, 11113 ] }

// 更新数组中索引为3的元素值
> db.coll.update({"name":"eco"}, {"$inc":{"qq.3":2}})

// 查看更新结果1⃣️
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 3, "qq" : [ 11111, 11112, 11112, 11115 ] }

// 更新数组qq中第一个匹配查询条件"等于11112"的元素值
> db.coll.update({"qq":11112}, {"$inc":{"qq.$":2}})

// 查看更新结果2⃣️
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe758433466cdeb2152fae6"), "name" : "eco", "age" : 3, "qq" : [ 11111, 11114, 11112, 11115 ] }
```

### update的高阶操作
除了修改器参数的多样性，MongoDB还在方法级别上提供了一些有用特性。通过命令`db.coll.update`可以查看该方法的定义（注意是不带括号的命令）：
```
> db.coll.update
function(query, updateSpec, upsert, multi) {
  ...
}
```

其中`upsert`和`multi`传入的都是布尔值，默认值均为false。分别表示更新插入特性与批量修改特性

在执行update命令后，如果命令执行正常，会输出类似`WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })`的信息，分别表示匹配到的文档数量、查询不到时插入的文档数量以及查询到时更改的文档数量。这对分析update执行upsert或批量修改时的结果都有帮助。

#### upsert 更新或插入
upsert是MongoDB中一种特有的更新操作，它的语义是查找到匹配文档时进行更新，否则插入新文档（update or insert）。它的特点是 **性能与原子性** 。

- 性能：完成同样的目标，传统做法需要先查询是否存在，再根据查询结果决定是插入还是更新，而upsert一次交互即可完成这样的语义。
- 原子性：整个执行过程是原子的，如果设置的字段拥有唯一索引，则在一个请求执行upsert时，不会被另一个请求的insert或upsert操作干扰。

它并不是MongoDB Client所支持的一种命令，使用方式如下：
```
// 先确保zill不存在
> db.coll.find({"name":"zil"})

// 通过update第三个参数传入true开启upsert功能
> db.coll.update({"name":"zil"}, {"age":0}, true)
WriteResult({
	"nMatched" : 0,
	"nUpserted" : 1,
	"nModified" : 0,
	"_id" : ObjectId("5fe76da198d1a26e17ce21d7")
})

// 查询upsert结果1⃣️
> db.coll.find({"name":"zil"})
> db.coll.find({"age":0})
{ "_id" : ObjectId("5fe76da198d1a26e17ce21d7"), "age" : 0 }

// 再对该文档执行一次upsert
> db.coll.update({"_id" : ObjectId("5fe76da198d1a26e17ce21d7")}, {"age":1}, true)
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

// 查询upsert结果2⃣️
> db.coll.find({"_id" : ObjectId("5fe76da198d1a26e17ce21d7")})
{ "_id" : ObjectId("5fe76da198d1a26e17ce21d7"), "age" : 1 }
```

从查询结果1⃣️可以说明，当字段不存在时，插入的新文档只会包含需要更新的字段，而不会包含查询字段。

而两次upsert操作的输出是不同的，第一次查询不到，新增了文档，输出结果为：
```
WriteResult({
	"nMatched" : 0,
	"nUpserted" : 1,
	"nModified" : 0,
	"_id" : ObjectId("5fe76da198d1a26e17ce21d7")
})
```

第二次查询到了，对该文档进行了修改，输出结果为：
```
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

#### 批量修改
在默认情况下，当更新时，查询到多条匹配的文档，只会更新其中的一条，如果需要全量更新，需要在调用update时将参数`multi`设置为true。

假设当前一共有3条"name"为"eco"的文档：
```
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe77339061d6ea0f9d64a64"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe7733b061d6ea0f9d64a65"), "name" : "eco", "age" : 2 }
{ "_id" : ObjectId("5fe7733f061d6ea0f9d64a66"), "name" : "eco", "age" : 3 }
```

在执行默认的更新操作后，只会更改其中的一个文档：
```
db.coll.update({"name":"eco"}, {"$set":{"age":4}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

但如果将参数`multi`设置为true，则会更新匹配查询条件的全部文档：
```
> db.coll.update({"name":"eco"}, {"$set":{"age":5}}, false, true)
WriteResult({ "nMatched" : 3, "nUpserted" : 0, "nModified" : 3 })
```

### findAndModify操作
findAndModify是另一种 **原子** 的更新操作，与update不同的是，findAndModify执行后会返回更新前的文档，如果文档找不到则返回null。

可以通过client执行`db.coll.findAndModify({query:{"name":"eco"},update:{$inc:{"age":1}}})`找到"name"为"eco"的文档，并将其"age"的值加1，执行该命令后，在终端会输出该文档更新前的值。

除了query和update，findAndModify还支持以下几个类别的行为：
- sort: 对结果进行排序
- remove: 与update不同的是，它将查找文档，并将其删除
- new: 决定返回的是更新/删除前的文档还是更新/删除后的文档，默认返回的是更新/删除前的文档。
- field: 返回指定的文档字段
- upsert: 在update操作中是否加入upsert语义

### 修改的效率
在插入文档时，依次插入的文档在磁盘上的位置是相邻的，但如果修改后，文档变大了，原来的位置可能无法容纳该文档，此时文档就会被迁移到磁盘的其他位置，此时原磁盘的位置就被空了出来。空闲的空间可用于前一个文档的扩充，或者当该空间可满足新文档需求时，被新文档所使用。

由于数据在磁盘上的迁移（读写）是很慢的，每次插入新文档时，可以为其预留一部分磁盘空间，尽量避免因磁盘空间不足导致的文档整体迁移。在MongoDB中，有一个动态调控的配置，称为填充因子，遵循公式：
```
实际使用空间 = 文档所需空间 * 填充因子
```
当填充因子为1时，所有文档紧挨在一起，当文档因体积增大而移动到磁盘的其他位置，填充因子也随着变大，此时每次新增文档时，都会为该文档预留更多的磁盘空间，而当文档不再移动（不管是文档大小趋于稳定还是预分配空间可满足文档大小变化），填充因子的值又会缓慢降低。

## 查询

我认为查询语句是基础操作语法中最重要的一部分，因为查询是更新和删除操作中的前置动作，它们都会使用查询语法来匹配文档，而有时一些特定的场景下，查询并不是像`{"name":"eco"}`这样简单的等于关系，例如像表达年龄大于18或者朋友中包含eco这样的复杂查询。

查询的接口有两个，分别是`find()`和`findOne()`。`find()`会查询返回所有（该描述不准确，后文会介绍）匹配的文档，如果不传入查询条件，则返回集合中的全部文档。`findOne()`会查询集合中第一个匹配查询条件的方法，如果不传入查询条件，则返回集合中的第一个文档。在执行查询时，可以在`find()`或`findOne()`后面加上`.pretty()`，使查询结果格式化为JSON格式，文档可读性更高。

find()定义如下：
```
function(query, fields, limit, skip, batchSize, options)
```

其中常用的几个字段对应的关系为：

| 参数名    | 作用                                                                                           |
| --------- | ---------------------------------------------------------------------------------------------- |
| query     | 查询时的过滤条件                                                                               |
| fields    | 控制返回的字段集                                                                               |
| limit     | 限定匹配的文档个数量                                                                                   |
| skip      | 跳过n个 **匹配的** 文档再执行查询                                                              |
| batchSize | 批量返回的文档大小，当匹配到到文档过多时，并不会被一次返回，可以通过该参数调整每次返回的文档数 |

### query 指定查询条件

查询支持许多复杂的查询条件，最基础的就是判断字段值是否相等，以及将不同的查询条件并行组合：

```
// 查询集合中所有name为eco的文档
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }

// 查询集合中所有name为eco且age等于1的文档
> db.coll.find({"name":"eco", "age":1})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
```

### fields 指定返回的键

在调用`find()`或`findOne()`时，默认会返回匹配文档的全部字段，包括它的"\_id"字段，如果不需要返回全部字段，可以在查询时指定返回内容，以此节省数据传输量以及BSON解码的时间、空间消耗。具体做法如下：

```
// 返回name字段，默认会将_id字段也返回
> db.coll.find({},{"name":1})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco" }

// 如果需要剔除_id字段，可以显示禁止其返回
> db.coll.find({},{"name":1, "_id":0})
{ "name" : "eco" }

// 也可指定需要剔除的字段
> db.coll.find({},{"name":0})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "age" : 1 }

// 但不可同时指定需要字段与不需要字段（除非是剔除的_id）
> db.coll.find({},{"name":0, "age":1})
Error: error: {
	"ok" : 0,
	"errmsg" : "Projection cannot have a mix of inclusion and exclusion.",
	"code" : 2,
	"codeName" : "BadValue"
}
```

### limit & skip 分页查询的基础

limit和skip可用于分页查找场景，类似于MySQL中`SELECT ... LIMIT offset,limit`，具体用法如下：
```
// 若匹配的文档有10个，则从第0个开始返回2个文档
> db.coll.find({"name":"eco"},{},2,0)
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }

// 分页的另一种查询方式
> db.coll.find({"name":"eco"}).skip(3).limit(3)
{ "_id" : ObjectId("5fe8adfc938232c61bee2b43"), "name" : "eco", "age" : 4 }
{ "_id" : ObjectId("5fe8ae04938232c61bee2b44"), "name" : "eco", "age" : 5 }
{ "_id" : ObjectId("5fe8ae06938232c61bee2b45"), "name" : "eco", "age" : 6 }
```

skip的值不宜过大，因为skip会跳过n个已匹配的文档，当skip过大时，MongoDB需要先匹配到这n个文档并丢弃，再从该位置开始匹配查询条件并返回。这对查询的性能影响很大。

如果我们的查询条件是有指定排序的，可以利用这一特性，在下一次查找时，加上一个过滤条件。例如我们的查询是按文档中的日期排序，其中第1000条的日期是具体的时间t，比起`find({...},{},100,1000)`跳过前1000个已匹配的文档，使用`find({..., "date": {"$gt": t}},{},100,0)`直接查询日期大于t的100个文档的效率可能会更高，如果在"date"上能利用索引，效果会更明显。

在使用MongoDB Shell查询时，如果查询匹配的文档超过20个，则在首次执行查询时只会展示20个，如果需要查看更多文档，则可以输入`it`获取[20, 40)的内容：
```
> db.coll.find({"name":"eco"})
{ "_id" : ObjectId("5fea18f7938232c61bee2b8c"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fea18f9938232c61bee2b8d"), "name" : "eco", "age" : 2 }
{ "_id" : ObjectId("5fea18fb938232c61bee2b8e"), "name" : "eco", "age" : 3 }
{ "_id" : ObjectId("5fea18fe938232c61bee2b8f"), "name" : "eco", "age" : 4 }
{ "_id" : ObjectId("5fea1900938232c61bee2b90"), "name" : "eco", "age" : 5 }
{ "_id" : ObjectId("5fea1901938232c61bee2b91"), "name" : "eco", "age" : 6 }
{ "_id" : ObjectId("5fea1904938232c61bee2b92"), "name" : "eco", "age" : 7 }
{ "_id" : ObjectId("5fea1906938232c61bee2b93"), "name" : "eco", "age" : 8 }
{ "_id" : ObjectId("5fea1909938232c61bee2b94"), "name" : "eco", "age" : 9 }
{ "_id" : ObjectId("5fea190b938232c61bee2b95"), "name" : "eco", "age" : 10 }
{ "_id" : ObjectId("5fea190d938232c61bee2b96"), "name" : "eco", "age" : 11 }
{ "_id" : ObjectId("5fea190f938232c61bee2b97"), "name" : "eco", "age" : 12 }
{ "_id" : ObjectId("5fea1911938232c61bee2b98"), "name" : "eco", "age" : 13 }
{ "_id" : ObjectId("5fea1913938232c61bee2b99"), "name" : "eco", "age" : 14 }
{ "_id" : ObjectId("5fea1914938232c61bee2b9a"), "name" : "eco", "age" : 15 }
{ "_id" : ObjectId("5fea1916938232c61bee2b9b"), "name" : "eco", "age" : 16 }
{ "_id" : ObjectId("5fea1918938232c61bee2b9c"), "name" : "eco", "age" : 17 }
{ "_id" : ObjectId("5fea191a938232c61bee2b9d"), "name" : "eco", "age" : 18 }
{ "_id" : ObjectId("5fea191c938232c61bee2b9e"), "name" : "eco", "age" : 19 }
{ "_id" : ObjectId("5fea191f938232c61bee2b9f"), "name" : "eco", "age" : 20 }
Type "it" for more
> it
{ "_id" : ObjectId("5fea1920938232c61bee2ba0"), "name" : "eco", "age" : 21 }
```

### 使用sort对结果排序

我们可以根据查询结果的某个字段或者多个字段进行排序。

执行`db.coll.find({...}).sort({"age":1})`可以对查询的结果根据"age"字段进行正序排序。

> 字段后的值1表示正序排序，-1表示逆序排序。

如果想根据多个字段排序，则可以这样做：
```
// 根据"name"正序排序，"name"相同时根据"age"逆序排序
db.coll.find({...}).sort({"name":1, "age":-1})
```

但MongoDB并没有约束文档中某一个字段的具体类型，所以在排序时，可能碰到不止一个字段类型（我认为除了null以外的两种类型混用，极大可能是使用上的问题），但MongoDB考虑了这种情况，并给出了优先级的大小顺序：

1. 最小值
2. null
3. 数字（整型、长整型、双精度）
4. 字符串
5. 对象/文档
6. 数组
7. 二进制数据
8. 对象ID，即ObjectId类型
9. 布尔型
10. 日期型
11. 时间戳
12. 正则表达式
13. 最大值

### 高阶查询语法

前面作为入门只介绍了等于的使用以及多个查询条件组合的使用。但查询场景可能更复杂，接下来介绍其他查询方式。

#### 比较查询

两个值的关系除了等于，也可以是不等于、大于、大于等于、小于、小于等于中的一种。

以"age"字段为例，下面是各个比较关系的查询例子：
```
// 使用$gt查询coll集合中age大于8的文档
> db.coll.find({"age":{"$gt":8}})
{ "_id" : ObjectId("5fe8ae0d938232c61bee2b48"), "name" : "eco", "age" : 9 }

// 使用$gte查询coll集合中age大于等于8的文档
> db.coll.find({"age":{"$gte":8}})
{ "_id" : ObjectId("5fe8ae09938232c61bee2b47"), "name" : "eco", "age" : 8 }
{ "_id" : ObjectId("5fe8ae0d938232c61bee2b48"), "name" : "eco", "age" : 9 }

// 使用$lt查询coll集合中age小于2的文档
> db.coll.find({"age":{"$lt":2}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }

// 使用$lte查询coll集合中age小于等于2的文档
> db.coll.find({"age":{"$lte":2}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }

// 使用$ne查询coll集合中age不等于2的文档
> db.coll.find({"age":{"$ne":2}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8adfa938232c61bee2b42"), "name" : "eco", "age" : 3 }

// 多个比较查询条件搭配使用
> db.coll.find({"age":{"$lt":3, "$gt":1}})
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }
```

| 操作符 | 含义     | 全称                |
| ------ | -------- | ------------------- |
| $gt    | 大于     | great than          |
| $gte   | 大于等于 | great than or equal |
| $lt    | 小于     | less than           |
| $lte   | 小于等于 | less than or equal  |
| $ne    | 不等于   | not equal           |

查询语句结构：
```
{
	"age": {
		"$ne": 2
	}
}
```

#### OR查询

对于多个条件的或查询，可以使用$in($nin)或者$or。

$in的语义是，如果某一个Key的值在给定集合中，而$or的语义更接近条件之间的或关系。

执行`db.coll.find({"age":{"$in":[1,2]}})`会返回集合coll中，"age"等于1或者2的所有文档：
```
// 使用$in对age字段进行或查询，查询语义为age字段的值在给定数组中
> db.coll.find({"age":{"$in":[1,2]}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }
```

也可使用$nin查询某一个Key的值不在给定集合中的文档：
```
// 使用$nin对age字段进行或查询，查询语义为age字段的值不在给定数组中
> db.coll.find({"age":{"$nin":[3,4,5,6,7,8,9]}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }
```

如果多个条件所描述的并不是同一个字段，可使用$or对多个条件进行或运算。执行`db.coll.find({"$or":[{"age":1}, {"name":"zill"}]})`会返回集合coll中，"age"等于1或者"name"等于"zill"的所有文档。
```
// 使用or查询两个条件，满足任一条件即可
> db.coll.find({"$or":[{"age":1}, {"name":"zill"}]})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
```

除了$or，也有亦或操作符$nor，使用方式与$or类似：
```
// 使用$nor查询age不大于2且name不等于eco的文档
> db.coll.find({"$nor":[{"age":{"$gt":2}}, {"name":"eco"}]})
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
```

#### 数值字段取模与非条件

针对数值运算，还有更花哨的查询方式，例如取模。

执行`db.coll.find({"age":{"$mod":[3,1]}})`会返回集合coll中"age"字段除以3之后模1的所有文档。实际效果如下：

```
// if (age\3==1) than 匹配成功
> db.coll.find({"age":{"$mod":[3,1]}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8adfc938232c61bee2b43"), "name" : "eco", "age" : 4 }
{ "_id" : ObjectId("5fe8ae08938232c61bee2b46"), "name" : "eco", "age" : 7 }
```

如果条件是`age\3!=2`，需要加上$not操作符：
```
> db.coll.find({"age":{"$not":{"$mod":[3,2]}}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8adfa938232c61bee2b42"), "name" : "eco", "age" : 3 }
{ "_id" : ObjectId("5fe8adfc938232c61bee2b43"), "name" : "eco", "age" : 4 }
{ "_id" : ObjectId("5fe8ae06938232c61bee2b45"), "name" : "eco", "age" : 6 }
{ "_id" : ObjectId("5fe8ae08938232c61bee2b46"), "name" : "eco", "age" : 7 }
{ "_id" : ObjectId("5fe8ae0d938232c61bee2b48"), "name" : "eco", "age" : 9 }
```

$not可以用于与正则表达式以及其他操作符搭配使用，查询与之不匹配的所有文档。下面两个查询的实际效果相同：
```
// 使用操作符$not以及$gt获取age不大于2的文档
> db.coll.find({"age":{"$not":{"$gt":2}}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }

// 使用操作符$lte获取age小于等于2的文档
> db.coll.find({"age":{"$lte":2}})
{ "_id" : ObjectId("5fe8a08b061d6ea0f9d64a67"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fe8a09a061d6ea0f9d64a68"), "name" : "zill", "age" : 2 }
{ "_id" : ObjectId("5fe8ad14938232c61bee2b41"), "name" : "eco", "age" : 2 }
```

#### 查询null
为了更好的演示本节内容，先将集合的文档初始化为以下形式：
```
> db.coll.find()
{ "_id" : ObjectId("5fea082d938232c61bee2b7e"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fea085d938232c61bee2b7f"), "name" : "eco", "age" : 2, "like" : null }
{ "_id" : ObjectId("5fea0873938232c61bee2b80"), "name" : "zill", "age" : 3, "like" : "footbal" }
{ "_id" : ObjectId("5fea0a24938232c61bee2b81"), "name" : "zily", "age" : 4, "like" : "swimming" }
```

如果我们用普通的查询方式查询集合中"like"字段为null的文档，除了会返回"like"等于null的文档，也会返回不存在"like"字段的文档：
```
// 查询like字段为null的文档
> db.coll.find({"like":null})
{ "_id" : ObjectId("5fea082d938232c61bee2b7e"), "name" : "eco", "age" : 1 }
{ "_id" : ObjectId("5fea085d938232c61bee2b7f"), "name" : "eco", "age" : 2, "like" : null }
```

如果我们希望查询有"like"字段，且值为null的文档，则需要借助一点技巧：
```
// 使用操作符$in以及$exists作为查询条件
> db.coll.find({"like":{"$in":[null], "$exists":true}})
{ "_id" : ObjectId("5fea085d938232c61bee2b7f"), "name" : "eco", "age" : 2, "like" : null }
```

#### 使用正则表达式查询

有时为了查询集合中某字段符合某个规律（正则）的文档，可以借助正则表达式语法进行查询。

初始集合数据：
```
{ "_id" : ObjectId("5fea085d938232c61bee2b7f"), "name" : "eco", "age" : 2, "like" : null }
{ "_id" : ObjectId("5fea0873938232c61bee2b80"), "name" : "zill", "age" : 3, "like" : "footbal" }
{ "_id" : ObjectId("5fea0a24938232c61bee2b81"), "name" : "zily", "age" : 4, "like" : "swimming" }
```

例如想查询"name"字段为zi开头的所有文档，可以这样：
```
> db.coll.find({"name":/zi/i})
{ "_id" : ObjectId("5fea0873938232c61bee2b80"), "name" : "zill", "age" : 3, "like" : "footbal" }
{ "_id" : ObjectId("5fea0a24938232c61bee2b81"), "name" : "zily", "age" : 4, "like" : "swimming" }
```

在查询条件中，`/zi/i`不需要加上双引号，其中i是正则表达式的标识，在"regular_expression"中填入表达式内容即可：`/{$regular_expression}/i`。

> MongoDB使用Perl兼容的正则表达式（PCRE）库来匹配正则表达式。

#### 查询数组

与modify一样，数组的查询也支持额外的操作符。

为了更好的演示本节内容，先将集合的文档初始化为以下形式：
```
> db.coll.find()
{ "_id" : ObjectId("5fea0fe4938232c61bee2b82"), "name" : "eco", "age" : 1, "qq" : 11111 }
{ "_id" : ObjectId("5fea0ff6938232c61bee2b84"), "name" : "eco", "age" : 2, "qq" : [ 11111 ] }
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
{ "_id" : ObjectId("5fea1029938232c61bee2b86"), "name" : "eco", "age" : 4, "qq" : [ 33333, 44444, 55555 ] }
{ "_id" : ObjectId("5fea1194938232c61bee2b87"), "name" : "eco", "age" : 3, "qq" : [ 22222, 33333, 44444 ] }
```

如果想要查询数组中包含某个具体元素的文档，可以直接使用这样的查询方式：
```
// 查询qq为11111或者qq的数组中包含11111的元素
> db.coll.find({"qq":11111})
{ "_id" : ObjectId("5fea0fe4938232c61bee2b82"), "name" : "eco", "age" : 1, "qq" : 11111 }
{ "_id" : ObjectId("5fea0ff6938232c61bee2b84"), "name" : "eco", "age" : 2, "qq" : [ 11111 ] }
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
```

如果想查完全匹配的数组，可以在查询的值前后加上[]符号：
```
> db.coll.find({"qq":[11111]})
{ "_id" : ObjectId("5fea0ff6938232c61bee2b84"), "name" : "eco", "age" : 2, "qq" : [ 11111 ] }
> db.coll.find({"qq":[11111,22222]})
> db.coll.find({"qq":[11111,22222,33333]})
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 3, "qq" : [ 11111, 22222, 33333 ] }
```

如果想查询数组中是否存在某个子集，则需要使用操作符$all：
```
// 使用$all操作符查询qq字段是否包含[22222,33333]子集
> db.coll.find({"qq":{"$all":[22222,33333]}})
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
{ "_id" : ObjectId("5fea1194938232c61bee2b87"), "name" : "eco", "age" : 3, "qq" : [ 22222, 33333, 44444 ] }
```

可以通过$size查询该字段的数组是否有n个元素：
```
// 使用$size操作符查询数组元素数量匹配的文档
> db.coll.find({"qq":{"$size":3}})
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
{ "_id" : ObjectId("5fea1029938232c61bee2b86"), "name" : "eco", "age" : 4, "qq" : [ 33333, 44444, 55555 ] }
{ "_id" : ObjectId("5fea1194938232c61bee2b87"), "name" : "eco", "age" : 3, "qq" : [ 22222, 33333, 44444 ] }
```

如果在查询数组的过程中存在条件判断，例如$gt或$lt，则需要特别注意，因为普通的查询判断方式是——数组中任意一个元素匹配查询条件，则匹配：
```
// qq数组中任意元素匹配查询条件，则被返回
> db.coll.find({"qq":{"$gt":10000, "$lt":20000}})
{ "_id" : ObjectId("5fea0fe4938232c61bee2b82"), "name" : "eco", "age" : 1, "qq" : 11111 }
{ "_id" : ObjectId("5fea0ff6938232c61bee2b84"), "name" : "eco", "age" : 2, "qq" : [ 11111 ] }
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
```

如果对上面的案例来说，希望查询的是数组类型而非普通的数字类型，则需要使用$elemMatch操作符：
```
> db.coll.find({"qq":{"$elemMatch":{"$gt":10000, "$lt":20000}}})
{ "_id" : ObjectId("5fea0ff6938232c61bee2b84"), "name" : "eco", "age" : 2, "qq" : [ 11111 ] }
{ "_id" : ObjectId("5fea1019938232c61bee2b85"), "name" : "eco", "age" : 5, "qq" : [ 11111, 22222, 33333 ] }
```

#### 查询内嵌文档

MongoDB的数据类型中，有一类是内嵌文档，因此也会有对于内嵌文档的查询。

为了更好的演示本节内容，先将集合的文档初始化为以下形式：
```
> db.coll.find()
{ "_id" : ObjectId("5fea165d938232c61bee2b8a"), "name" : "eco", "age" : 1, "grade" : { "class" : "math", "score" : 90 } }
{ "_id" : ObjectId("5fea1665938232c61bee2b8b"), "name" : "eco", "age" : 1, "grade" : { "class" : "english", "score" : 80 } }
```

在"grade"的内嵌文档中，存在两个字段："class"和"score"，下面的查询是OK的：
```
// 查询name为eco且score等于80的文档
> db.coll.find({"name":"eco","grade.score":80})
{ "_id" : ObjectId("5fea1665938232c61bee2b8b"), "name" : "eco", "age" : 1, "grade" : { "class" : "english", "score" : 80 } }
```

对于查询内嵌文档中的字段，需要使用符号"."将外层文档与内层文档联系起来。
