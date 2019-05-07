## Spring后端应用
通过Web中的Spring章节，我们看到了Spring Web项目中熟悉的Controller。这是我们开发代码的出入口（不严谨），而在其后面，是我们的业务逻辑与数据访问。第三章*后端中的Spring*介绍了Spring持久化的起源与实战应用，通过Spring这个强大的媒介，我们可以更轻松、更专注的执行我们的数据访问操作。

常常我们开发项目时无法摆脱对持久化的依赖。很多时候我们持久化逻辑提炼到一个专司其职的组件中，而不会把它分散并耦合到各个业务单元，这样的专职组件称为DAO/Repository等。为什么不把它直接与业务逻辑写一起呢？我认为数据访问是一个很特性的单元，将它与业务紧耦合会造成业务类过重，不利于我们的复用与重构，并且解耦后的单元颗粒度更细，便于测试与调试。

**接口是实现松耦合的关键**。我们可以“针对接口编程”实现其解耦，这样做的好处是不仅能使代码更清晰，还可很优雅的对数据源进行替换。

### Spring对数据访问提供的支持
Spring帮我们解决了数据访问的第一大忙，即是对数据库访问异常的动刀。使用原始的JDBC操作数据库时，需要我们捕获许多信息量并不大的异常，以及一些捕获后无法当即解决的异常。Spring JDBC捕获了来自JDBC的异常，然后重新抛出更有信息量的非检查型异常。这不仅解决了前时代不用Spring手撕数据库时异常渗透的痛点，还可以缩小我们在排查时的问题范围。书中提到：**如果无法从SQLException中恢复，那为什么我们还要强制捕获它呢？** 我认为这句话也蕴含了Java Exception机制的哲学。

JDBC是一种偏底层的访问方式，它建立在SQL之上，虽然它让我们与数据库之间薄如蝉翼，但以此为代价我们需要编写代码获取连接、释放链接以及捕获异常等，从而锐减我们的开发效率。Spring JDBC将其封装，将样板代码囊入其中，站在巨人的肩膀上，我们使用JdbcTemplate模板类可以友好便捷的操作数据库。在开发过程中，当我避免不了考虑一些本不该我关注的事情，我总会寻找机会借助框架的力量。

除了JDBC，Spring也提供了类似于Hibernate、iBATIS以及JPA（Java Persistence API）等框架的集成。除了MySQL等关系型数据库，Spring对NoSQL技术的处理也是游刃有余。

### Spring操作关系型数据库
#### Spring集成Hibernate
除了基本的对象关系映射，Hibernate还提供了缓存、延迟加载、预先抓取等功能。
> 延迟加载（Lazy loading）：选择性的获取对象的数据，而非一次性全量的数据。这样可以避免获取某些不必要数据时的开销。
>
> 预先抓取（Eager fetching）：与延迟加载相对，一次获取多个对象间的关联，从而避免多次查询。

在Hibernate中主要使用接口Session来获取基本的CURD功能。基于Spring，我们在Repository中注入Hibernate的Session来获取其CURD的能力。但在此之前，我们需要先配置好Session。

Spring提供了3个Hibernate的Session工厂bean，分别为：
```
org.springframework.orm.hibernate3.LocalSessionFactoryBean
org.springframework.orm.Hibernate3.annotation.AnnotationSessionFactoryBean
org.springframework.orm.hibernate4.LocalSessionFactoryBean
```
我们需根据Hibernate版本、配置方式来选择不同的工厂Bean。但它们的工作都是配置数据源为我们所用。

虽然Hibernate很强大，但它已逐渐淡出开发者的视野。

#### Spring Data JPA
在Spring Data JPA的帮助下，通过继承JpaRepository接口，就可使我们的接口获取部分数据操作能力。如果我们不满足于此，可在接口中自定义方法来进一步满足我们的需求。Spring又一次强有力的解放了我们的双手，作为交换，我们的方法名需符合指定的命名规范。但即便如此，这种能力使用起来就像Apple的Siri一样便捷。

下面的例子展示了一个自定义Repository通过继承JpaRepository的方式来访问数据库：
```
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    boolean existsByName(String name);

    List<User> findUserByCountryOrRegionOrderByAgeASC(String country, String region);

}
```
我们可以通过UserRepository接口的existsByName判断数据库中是否存在某个人，也可根据国家和地球查询用户，并根据年龄升序排序。而更通用的findAll、save等来自父类们的方法则也可顾名思义的推断其用意。更丰富的内容可以直接参考[Spring Data JPA](https://spring.io/projects/spring-data-jpa#learn)。它是如果做到的呢？项目运行时，Spring Data会检查Repository接口的所有方法，解析方法名，并对其进行实现。以Demo中的第二个方法为例，前面提到的命名规范如下：
```
find   User     By     CountryOrRegionOrderByAgeASC
[动词] [主题] [关键词By] [断言]
```
其中，[动词]为固定选项，包括find/delete/count/exists等动作，不可高自由发挥；主题可省略，其意义更多在于可读性，对象类型是通过JpaRepository接口确定，而非主题；在By关键字后的断言可以缩小我们的查询条件，并且指定排序依据。这种方式也有一定的局限性，例如无法进行更新操作，为了进一步扩展其能力，我们也许会用到自定义查询。

- 通过@Query注解查询
  ```
  @Repository
  public interface UserRepository extends JpaRepository<User, Long> {
      @Query("UPDATE User SET age=?1 where id=?2")
      boolean updateAgeById(int age, int id);
  }
  ```
  示例展示了如何通过@Query修改用户年龄。@Query中"?1"与"?2"分别表示第一、二个入参。

- 通过实现类查询

  Spring Data JPA是一种较高层级的JPA实现，为了更友好的开发体验，牺牲了部分功能。如果我们想使用UserRepository做更细化的操作，则需要比Spring Data JPA更底层的实现方式。还是以修改用户年龄为例，我们具体要做哪些工作呢？
  1. 定义一个新接口，该接口定义了某个方法（声明了某种能力），提供给UserRepository继承。由于UserRepository仍然是我们访问数据库的入口，我们需要将```update()```方法提供给UserRepository，但不能将其直接写到UserRepository中，否则在项目启动时就报错。
  ```
  public interface UpdateOperation<T> {
    T update(T t);
  }
  ```
  2. 让目标Repository继承此接口以获取该能力。Java的接口支持多继承，使用此机制可将我们自定义的方法提供到UserRepository中。
  ```
  @Repository
  public interface UserRepository extends JpaRepository<User, Long>, UpdateOperation<User> {
  }
  ```
  3. 编写具体实现类。此类实现了UpdateOperation接口，但类名严格限制为UserRepositoryImpl。这样在程序运行时，才会将其与UserRepository的实现做组合，从而允许我们同时使用两种方式进行数据操作。
  ```
  public class UserRepositoryImpl implements UpdateOperation<User> {

    @Override
    User update(User user) {
      // 具体实现
    }
  }
  ```
  至此我们的自定义实现就完成了。

### Spring操作非关系型数据库
非关系型数据库即NoSQL(另一种称法为Not only SQL)，其结构易于扩展，性能优越，Spring Data也提供了对多种NoSQL的支持。
#### Spring Data MongoDB
MongoDB作为文档型NoSQL的经典实现，自然逃不过Spring的“魔爪”。Spring Data MongoDB提供了3种方式在Spring应用中使用MongoDB：
- 通过注解实现对象-文档映射
- 使用MongoTemplate实现基于模板的数据库访问
- 自动化的运行时Repository生成功能

在使用MongoDB之前，我们至少需要配置主机地址、数据库信息。

下面是一个最基本的MongoDB配置类：
```
@Configuration
@EnableMongoRepositories(basePackages = "com.yy.demo.repository")
public class MongoConfig {

  @Bean
  public MongoFactoryBean mongo() {
      MongoFactoryBean mongo = new MongoFactoryBean();
      mongo.setHost("localhost");
      return mongo;
  }

  @Bean
  public MongoOperations mongoTemplate(Mongo mongo) {
      return new MongoTemplate(mongo, "DemoDB");
  }

}
```

也可通过继承AbstractMongoConfiguration的方式配置：
```
@Configuration
@EnableMongoRepositories(basePackages = "com.yy.demo.repository)
public class MongoConfig extends AbstractMongoConfiguration {

    @Override
    protected String getDatabaseName() {
        return "DemoDB";

    }

    @Override
    public Mongo mongo() throws Exception {
      MongoClientURI uri = new MongoClientURI("mongodb://localhost:27017/DemoDB");
      return new MongoClient(uri);
    }

}
```

其中MongoDB的URI也可携带额外的配置，具体可查看[MongoURI文档](http://api.mongodb.com/java/2.12/com/mongodb/MongoURI.html)。

当应用中的表越来越多，在使用Spring Data JPA时，我们需要维护繁琐的数据对象模型。但在Spring Data MongoDB中，由于MongoDB的存储格式为BSON（一种类JSON的存储格式），Java对象与Mongo文档之间的映射不需要花费我们大量的精力。例如，在MongoDB中不存在User这个Document时，我们直接在应用中使用MongoTemplate的insert方法即可向MongoDB中插入一条User数据。不仅如此，Spring Data MongoDB也提供了一些能带给我们惊喜的注解。

| 注解      | 描述                                             |
| --------- | ------------------------------------------------ |
| @Document | 此注解应用于实体类上，用于绑定文档与实体类的映射 |
| @Id       | 标示类属性为ID域                                 |
| @DbRef    | 标示某个域要引用其他文档                         |
| @Feild    | 用于绑定类中的属性与文档的字段                   |
| @Version  | 标示某个属性用作版本域                           |
| @Index    | 为属性增加索引                                   |
下面通过一个实体类来展示其用法：
```
@Document(collection = "user_demo")
public class User {

  @Id
  private String id;

  @Field("user_name")
  private String name;

  @Index(unique = true)
  private String email;

  @Index(expireAfterSeconds = 100)
  private Date createdTime;

  // Getter & Setter
}
```
在示例中该类会关联到MongoDB中名为"user_demo"的文档中

属性id对应文档的MongoId，可由MongoDB自动生成

属性name对应文档中的"user_name"字段

属性email没有使用@Field注解，则会自动匹配文档的"email"字段。而@Index注解的unique值为true，不仅可以加快对email的检索，还会规定其字段为唯一值。

属性createdTime使用了@Index注解的的expireAfterSeconds属性，设置其过期时间为100秒，60秒后该条记录则会被Mongo删除。此处需要注意，MongoDB检查过期的频率为60s/次，如果MongoDB当前压力较大，则会相应进行延迟，因此此处的过期时间并非严格执行。并且此值发生改变时，需要先同步改变MongoDB中该索引，否则应用将启动失败。

Spring Data MongoDB也支持像Spring Data JPA一样的优秀的“面向方法名编程”。通过继承CrudRepository或MongoRepository都可实现此功能。这里我更推荐继承MongoRepository。MongoRepository是CrudRepository的子类，针对操作MongoDB扩展了更多的方法。

当通过命名方法名无法满足我们的需求时，也有适应于Mongo的@Query注解以及扩展实现，与Spring Data JPA十分类似。@Query注解的使用稍有不同：
```
public interface UserRepository extends MongoRepository<User, String> {
  @Query(value = "{'sex': ?0}", fields = "{'_id':0,'name':1}")
  List<MomentBreed> findUserNamesBySex(String sex);
}
```
例子中@Query中对应更改为Mongo的查询语句，**但参数索引由0开始，而JPA中由1开始，并且无法通过@Query进行更新操作。** 更新操作只能由扩展接口的方式实现。（博文中Spring Data MongoDB版本为2.1.6.RELEASE，我认为无法通过方法名以及注解来实现更新操作是该框架的一大痛点，期待后期版本实现此功能）

#### Spring Data Neo4j
Neo4j是一种图形数据库，以图形结构的形式存储数据。

如果将Neo4j作为应用的一部分，而不是独立服务，可做如下配置：
```
@Configuration
@EnableNeo4jRepositories(basePackages = "com.yy.demo.repository")
public class Neo4jConfig extends Neo4jDataAutoConfiguration {

    public Neo4jConfig() {
        setBasePackage("model");
    }

    public GraphDatabaseService graphDatabaseService() {
        return new GraghDatabaseFactory().newEmbeddedDatabase("/tmp/graphdb");
    }

}
```

Spring Data Neo4j提供的注解很全面：

| 注解                | 描述                                                                           |
| ------------------- | ------------------------------------------------------------------------------ |
| @NodeEntity         | 将Java类型声明为节点实体                                                       |
| @RelationshipEntity | 将Java类型声明为关联实体                                                       |
| @StartNode          | 将属性声明为关联关系实体的开始节点                                             |
| @EndNode            | 将属性声明为关联关系实体的结束节点                                             |
| @Fetch              | 将实体的属性声明为立即加载                                                     |
| @GraphId            | 将属性设置为实体的ID域，该属性为Long                                           |
| @GraphProperty      | 明确声明某个属性                                                               |
| @GraphTraversal     | 声明某个属性会自动添加一个iterable元素，这个元素是图遍历所构建的               |
| @Indexed            | 声明某个属性应被索引                                                           |
| @Labels             | 为@NodeEntity声明标签                                                          |
| @Query              | 声明某个属性会自动添加一个iterable元素，这个元素是执行给定d Cypher查询所构建的 |
| @QueryResult        | 声明某个Java或接口能够持有查询结果                                             |
| @RelatedTo          | 通过某个属性，声明当前的@NodeEntity与另一个@NodeEntity直线的关联关系           |
| @RelatedToVia       | 在@NodeEntity上声明某个属性，指定其引用该节点所属的某一个@RelationshipEntity   |
| @RelationshipType   | 将某个域声明为关联实体类型                                                     |
| @ResultColumn       | 在带有@QueryResult注解的类型上，将某个属性声明为获取查询结果集中的某个特定列   |

Spring中Neo4j的使用与Mongo相似，支持简单粗暴的Repository，也支持@Query注解以及模板类Neo4jOperations的访问形式。

#### Spring Data Redis
Redis是一种Key-Value键值对类型的NoSQL，它的基础数据结构很简单，包括常用的String、List、Hash、Set、ZSet以及扩展了String的BitMap与HyperLogLog。相对于MongoDB、Neo4j等数据库结构，其键值对结构更加简单，因此也不依赖通过Repository访问数据库。

Spring Data Redis为4种Redis客户端实现了连接工厂，这4种客户端分别为：
- JedisConnectionFactory（Redis官方比较推荐的一种客户端，参考[Redis官网](https://redis.io/clients)）
- JredisConnectionFactory
- LettuceConnectionFactory
- SrpConnectionFactory

以配置Jedis客户端为例，配置简洁易懂：
```
@Bean
public RedisConnectionFactory redisCF() {
    JedisConnectionFactory cf = new JedisConnectionFactory();
    cf.setHostName("localhost");
    cf.setPort(6379);
    cf.setPassword("password");
    return cf;
}
```

Spring提供了两个模板类来访问Redis：RedisTemplate与StringRedisTemplate。

通过前面的Redis连接工厂即可提供这两个模板类需要的连接：
```
// RedisTemplate
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisCF) {
    RedisTemplate<String, Object> redis = new RedisTemplate<>();
    redis.setConnectionFactory(redisCF);
    return redis;
}

// StringRedisTemplate
@Bean
public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisCF) {
    return new StringRedisTemplate(redisCF);
}
```
这两个模板类的区别在于我们操作的是String还是具体对象，另外一个区别则在于序列化。

Spring Data Redis支持多种序列化器：
- GenericToStringSerializer：使用Spring转换服务进行序列化
- JacksonJsonRedisSerializer：使用Jackson1将对象序列化为JSON
- Jackson2JsonRedisSerializer：使用Jackson2将对象序列化为JSON
- JdkSerializeationRedisSerializer：使用Java序列化
- OxmSerializer：使用 Spring O/X实现序列化，用于XML序列化
- StringRedisSerializer：序列化String类型的key和value

RedisTemplate默认使用JdkSerializationRedisSerializer，StringRedisTemplate默认使用StringRedisSerializer。在模板类的配置中，我们可以指定key与value的序列化方式。如果我们希望key是String类型，而value序列化为JSON，可进行如下配置：
```
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisCF) {
    RedisTemplate<String, Object> redis = new RedisTemplate<>();
    redis.setConnectionFactory(redisCF);
    redis.setKeySerializer(new StringRedisSerializer());
    redis.setValueSerializer(new Jackson2JsonRedisSerializer<>(Demo.class));
    return redis;
}
```

从各类NoSQL的集成可以看出，Spring Data可以让我们在应用中极大的减少数据源切换带来的代码变动。

### Spring Security
