# Java中的问题记录

## 单元测试运行失败
> 在一个新项目中，使用单元测试调试Repository时运行报错，但启动Application时Mongo交互正常。
### 细节
```
java.lang.IllegalArgumentException: Given type must be an interface!
    at ····
Caused by: java.lang.IllegalArgumentException: Given type must be an interface!
    at ····
```
### 代码
```
// Mongo配置类
@Configuration
@EnableMongoRepositories(basePackages={"com.lyy.repository"})
public class MongodbConfig extends AbstractMongoConfiguration {...}

// Repository接口
package com.lyy.repository;

public interface DemoRepository extends CrudRepository<Moment, String> {...}

// 测试类
package com.lyy.repository;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = {MomentApplication.class})
public class MomentRepositoryTest {...}
```

### 问题发现
由于之前没出现过此类情况，考虑新项目下是否测试用例创建步骤有差错。原来创建的测试用例包名与Repository接口包名一致（测试用例在test目录下，Repository在java目录下），导致Mongo配置类在扫描包时，扫描到测试类，而@EnableMongoRepositories注解配置的包名会扫描到测试用例，所以运行测试类方法时抛出```Given type must be an interface!```异常。

### 解决
更改测试包名结构，避免配置类扫描到测试用例。

## 项目运行失败
> 引入公司第三方包后，通过其实现日志输出。本地运行正常，通过Jenkins打包测试环境时项目启动失败。

### 细节
```
Caused by: java.lang.UnsupportedClassVersionError: com/tcl/mibc/log/CommonInfoLogAnno : Unsupported major.minor version 52.0
```

### 问题发现
打印运行日志后发现上述错误，造成原因是第三方包中使用的是Java8版本，而测试服务器以及项目使用的Java版本为7.本地不报错原因是因为本地Java版本为8，只是通过IDEA指定该项目Java等级为7。

### 解决
第三方包内并没用到Java8特性，经沟通将第三方包的Java版本指定为1.7，重新deploy后，本项目通过Jenkins重新打包并在测试机重启服务后运行正常。

## 多数据源配置失败
使用多MongoDB数据源时，配置相应数据库失效。

### 代码
第一个配置类如下：
```
@Configuration
@EnableMongoRepositories(basePackages = {"com.2y.demo.repository.one"},
        mongoTemplateRef = OneMongodbConfig.MONGO_TEMPLATE)
public class OneMongodbConfig extends AbstractMongoConfiguration {

    @Value("${spring.data.one.mongodb.uri}")
    private String uri;

    @Value("${spring.data.one.mongodb.database}")
    private String dbName;

    static final String MONGO_TEMPLATE = "oneMongoTemplate";

    @Override
    public String getDatabaseName() {
        return dbName;
    }

    @Override
    public Mongo mongo() {
        MongoClientURI mongoClientURI = new MongoClientURI(uri);
        return new MongoClient(mongoClientURI);
    }

    @Override
    @Bean(name = MONGO_TEMPLATE)
    public MongoTemplate mongoTemplate() throws Exception {
        MappingMongoConverter converter = mappingMongoConverter();
        converter.setTypeMapper(new DefaultMongoTypeMapper(null));
        return new MongoTemplate(mongoDbFactory(), converter);
    }
}
```

第二个配置类如下：
```
@Configuration
@EnableMongoRepositories(basePackages = {"com.2y.demo.repository.two"},
        mongoTemplateRef = TwoMongodbConfig.MONGO_TEMPLATE)
public class TwoMongodbConfig extends AbstractMongoConfiguration {

    @Value("${spring.data.two.mongodb.uri}")
    private String uri;

    @Value("${spring.data.two.mongodb.database}")
    private String dbName;

    static final String MONGO_TEMPLATE = "twoMongoTemplate";

    @Override
    public String getDatabaseName() {
        return dbName;
    }

    @Override
    public Mongo mongo() {
        MongoClientURI mongoClientURI = new MongoClientURI(uri);
        return new MongoClient(mongoClientURI);
    }

    @Override
    @Bean(name = MONGO_TEMPLATE)
    public MongoTemplate mongoTemplate() throws Exception {
        MappingMongoConverter converter = mappingMongoConverter();
        converter.setTypeMapper(new DefaultMongoTypeMapper(null));
        return new MongoTemplate(mongoDbFactory(), converter);
    }
}
```

而相应的one和two两个包下的Repository只使用了其中的一个数据源。

### 问题发现
真正与MongoDB建立连接是配置在MongoDbFactory中，这两个数据源使用了同一个MongoDbFactory。如果SpringBoot启动时，先生效OneMongodbConfig，此时其父类会实例化一个MongoDbFactory，等SrpingBoot加载第二个配置类时，发现已有MongoDbFactory的Bean，则不再实例化另一个MongoDbFactory。

### 解决方案
在两个配置类中都重写方法`mongoDbFactory()`，并配置不同的Bean名：
```
@Override
@Bean(name = "oneMongoDBFactory")
public MongoDbFactory mongoDbFactory() throws Exception {
    return super.mongoDbFactory();
}
```

## Mongo Repository 查询过慢
在使用Spring Clouds微服务时，某个服务调用失败，通过Feign熔断打印异常。后通过Postman直接调用该服务器接口，虽然可以返回数据，但响应时间非常长。也是这个原因导致熔断触发。

在代码中插入日志监控该接口中各个方法请求时间，发现某个方法访问数据库时耗时 20-30 秒！

数据库为MongoDB，当前该集合中一共有1200+w条记录。查询时携带的字段与排序字段都加了索引。通过MongoDB客户端直接查询耗时也很短，于是把问题定位到该访问方法上。

该方法是通过继承MongoRepository后编写的接口方法，没有进行实现（由框架实现），具体如下：
```
public interface MomentRepository extends MongoRepository<Moment, String>, UpdateOperations<Moment>, FindOperations {
    /**
     * 分页获取用户发布的动态
     *
     * @param userId   用户id
     * @param status   动态状态
     * @param type     动态类型
     * @param date     发布时间
     * @param pageable 分页传入参数
     * @return 分页动态结果
     */
    Page<Moment> findAllByUserIdAndStatusInAndTypeAndPublishTimeBeforeOrderByPublishTimeDesc(String userId, List<Integer> status, Integer type, Date date, Pageable pageable);

}
```

该方法的调用处实际需求是查询 **指定个数** 的动态集，而如果想使用Repository特性编程，在方法中需要构造Pageable参数，并将size传入其中。（伏笔）

换成MongoTemplate实现：
```
public List<Moment> customFind(String userId, List<Integer> status, Integer type, Date date, Pageable pageable) {
    Query query = new Query();
    Sort sort = new Sort(Sort.Direction.DESC, "publishTime");
    query.addCriteria(
            Criteria.where("userId")
                    .is(userId).and("status")
                    .in(status).and("type")
                    .is(type).and("publishTime").lt(date)).with(pageable).with(sort);
    return mongoTemplate.find(query, Moment.class);
}
```

之前提到实际需求是获取List\<Moment>，而不是Page\<Moment>。切换实现方式后，接口响应时间迅速缩短到30ms左右。

为了更深入的了解两种实现方式的区别，开启了Spring Mongo Data的DEBUG日志，即在配置文件中加入下面的配置：
```
logging:
  level:
    org:
      springframework:
        data:
          mongodb:
            core: DEBUG
```

但两种实现方式的调试日志相同：
```
// Repository
10:59:57.285 [http-nio-9004-exec-2] DEBUG o.s.data.mongodb.core.MongoTemplate - find using query: { "userId" : "test", "status" : { "$in" : [1, 2, 3] }, "type" : 0, "publishTime" : { "$lt" : { "$date" : 1563343181779 } } } fields: Document{{}} for class: class com.tcl.mibc.social.common.entity.Moment in collection: moment

// MongoTemplate
10:59:57.285 [http-nio-9004-exec-2] DEBUG o.s.data.mongodb.core.MongoTemplate - find using query: { "userId" : "test", "status" : { "$in" : [1, 2, 3] }, "type" : 0, "publishTime" : { "$lt" : { "$date" : 1563343181779 } } } fields: Document{{}} for class: class com.tcl.mibc.social.common.entity.Moment in collection: moment
```

仔细比较两种实现，前者（Repository）的返回参数中还会返回总条数。后通过jstack查看方法调用，发现该方法会隐式的调用cout方法，这也是造成该方法请求速度过慢的原因。

但由于大部分地方都用到了MomentRepository特性，想尽量减少MongoTemplate实现，否则项目结构会变复杂。

在查阅 [Spring Data MongoDB](https://docs.spring.io/spring-data/mongodb/docs/2.1.9.RELEASE/reference/html/) 文档时，如要使用方法命名的方式访问数据库，可通过两种方式限制返回集合数量大小：
```
// 1. 通过TopX或FirstX命名
List<T> findTop10ByUserIdIn(List<String> ids)

// 2. 传入Pageable参数
Page<T> findAllByUserIdIn(List<String> ids, Pageable pageable)
```
其中第一种方式适用于查询固定数量的集合，不易于扩展，第二种方式扩展性好，但返回 `Page` 对象时，会做一次 `count` 操作查询符合条件的元素个数。

本来想通过 `@Query` 注解自定义查询，后经过尝试，直接修改方法的返回类就可以解决当前问题：
```
List<Moment> findAllByUserIdAndStatusInAndTypeAndPublishTimeBeforeOrderByPublishTimeDesc(String userId, List<Integer> status, Integer type, Date date, Pageable pageable);
```
