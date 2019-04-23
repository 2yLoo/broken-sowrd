### 单元测试运行失败
> 在一个新项目中，使用单元测试调试Repository时运行报错，但启动Application时Mongo交互正常。
#### 细节
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
