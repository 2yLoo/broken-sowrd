## 单元测试运行失败
> 在一个新项目中，使用单元测试调试Repository时运行报错，但启动Application时Mongo交互正常。
#### 细节
```
java.lang.IllegalArgumentException: Given type must be an interface!
    at ····
Caused by: java.lang.IllegalArgumentException: Given type must be an interface!
    at ····
```
#### 代码
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

#### 问题发现
由于之前没出现过此类情况，考虑新项目下是否测试用例创建步骤有差错。原来创建的测试用例包名与Repository接口包名一致（测试用例在test目录下，Repository在java目录下），导致Mongo配置类在扫描包时，扫描到测试类，而@EnableMongoRepositories注解配置的包名会扫描到测试用例，所以运行测试类方法时抛出```Given type must be an interface!```异常。

#### 解决
更改测试包名结构，避免配置类扫描到测试用例。

## 项目运行失败
> 引入公司第三方包后，通过其实现日志输出。本地运行正常，通过Jenkins打包测试环境时项目启动失败。

#### 细节
```
Caused by: java.lang.UnsupportedClassVersionError: com/tcl/mibc/log/CommonInfoLogAnno : Unsupported major.minor version 52.0
```

#### 问题发现
打印运行日志后发现上述错误，造成原因是第三方包中使用的是Java8版本，而测试服务器以及项目使用的Java版本为7.本地不报错原因是因为本地Java版本为8，只是通过IDEA指定该项目Java等级为7。

#### 解决
第三方包内并没用到Java8特性，经沟通将第三方包的Java版本指定为1.7，重新deploy后，本项目通过Jenkins重新打包并在测试机重启服务后运行正常。
