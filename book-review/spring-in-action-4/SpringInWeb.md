## Spring Web开发
该章涵盖了Spring MVC相关知识，在看书之前，Spring MVC其实是我的一个盲点。Spring Boot的Web Starter把它“藏”起来了，开发过程中不刻意的话我无法触摸到真实存在的MVC，通过这一章，在没有Spring Boot的“阻挠”下，MVC像是我发现的新大陆。

MVC是3个单词首字母的大写：
- M:Model，前后端交互产生的信息
- V:View，信息展示的视图
- C:Controller，，信息处理端

在传统的网页开发中，一个请求是经历这这些步骤才把页面展现到我们面前的：
![image](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/05fig01.jpg)

它的步骤概念如下：
1. 首先请求到了DispatcherServlet，它是一个调度器
2. 它向处理器映射确定请求的下一步去哪
3. 确定好之后将请求发送给合适的Controller
4. Controller做了逻辑处理的工作，将数据填充到模型中，并将确定的模型和视图名发送给DispatcherServlet
5. DispatcherServlet根据返回的视图名匹配特定的视图实现
6. DispatcherServlet将模型数据交付给指定的视图
7. 视图将模型数据渲染并输出

> 对于3、4步在这我想多说一句，我们在开发后台逻辑时，应尽量简化Controller的工作，把业务逻辑委托给Service实现。在刚接触开发时，我曾疑惑业务逻辑应该写在哪？写在Controller/Service/Dao中JVM都能很完美的处理好我们交代给它的事，但规范的做法应该是写在Service中，让Service专心处理业务逻辑，并有助于提高业务逻辑的重用性。

从图中可以看到，DispatcherServlet是整个流程的核心，由它负责请求的路由与匹配。

### 构建Spring Web

#### 配置DispatcherServlet
DispatcherServlet是整个后端的港口。下面展示了一个很简洁的配置类，而通过继承AbstractAnnotationConfigDispatcherServletInitializer，Spring帮我们省去了很多工作，先看下这个配置的结构：

```
// 注①
public class DempWebAppInitializer
       extends AbstractAnnotationConfigDispatcherServletInitializer {

  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" }; // 注②
  }

  @Override
  // 注③
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] { RootConfig.class }; // 注④
  }

  @Override
  // 注⑤
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { WebConfig.class }; // 注⑥
  }

}
```
这个类通过寥寥几行代码就配置了我们需要的核心:DispatcherServlet，但它有6个需要注意的地方！

- 注①：**扩展了AbstractAnnotationConfigDispatcherServletInitializer的任意类都会自动配置DispatcherServlet和Spring应用上下文。**

- 注②：getServletMappings()方法映射了"/"，它将应用默认的Servlet，处理所有进入应用的请求。

- 注③：getRootConfigClasses()返回带有@Configuration的类，用以配置ContextLoaderLinstener创建的应用上下文的Bean。

- 注④：RootConfig.class是我们下文会展现的一个配置类，它声明了ContextLoaderLinstener所需要的Bean。

- 注⑤：getServletConfigClasses()返回带有@Configuration的类，用以定义DispatcherServlet应用上下文的Bean。

- 注⑥：WebConfig.class是我们下文会展示的另一个配置类，它声明了DispatcherServlet所需要的Bean。


那么这个52字母组成的类是干嘛的呢？比起它的名字，它的能力可能更受人欢迎：

  > 实现了ServletContainerInitializer的类将用于配置Servlet容器，Spring提供了该类实现，名为SpringServletContainerInitializer，它会查找实现了WebApplicationInitializer的类。在Spring 3.2，AbstractAnnotationConfigDispatcherServletInitializer实现了WebApplicationInitializer接口，我们在配置Servlet时通过继承AbstractAnnotationConfigDispatcherServletInitializer实现了WebApplicationInitializer，所以部署到Servlet 3.2时容器会自动发现它，用以配置Servlet上下文。
>
> 简而言之，Spring提供一个名为SpringServletContainerInitializer的类将配置任务交给AbstractAnnotationConfigDispatcherServletInitializer完成，而我们的类扩展了它，以此配置上下文需要的Bean。

接下来看看RootConfig与WebConfig这两个配置类干了什么事情。
```
@Configuration
@ComponentScan(basePackages={"2y"},
    excludeFilters={
        @Filter(type=FilterType.ANNOTATION, value=EnableWebMvc.class)
    })
public class RootConfig {
}
```
RootConfig的基础配置相当简单，它使用@ComponentScan注解扫描指定目录，以此来获取应用上下文的Bean。
```
@Configuration
@EnableWebMvc	// 通过此注解启用Spring MVC
@ComponentScan("2y.web")	 // 启用组件扫描
public class WebConfig
       extends WebMvcConfigurerAdapter {

  @Bean
  // 在这我们配置JSP视图解析器
  public ViewResolver viewResolver() {
    InternalResourceViewResolver resolver =
            new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    resolver.setExposeContextBeansAsAttributes(true);
    return resolver;
  }

  @Override
  // 配置静态资源处理，否则DispatcherServlet会处理所有包括静态资源的请求
  public void configureDefaultServletHandling(
        DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
  }

}
```
WebConfig的配置可以比RootConfig更加简单，但基本本章内容，我们指定一下视图解析器与组件扫描。启动组件扫描后我们可在2y.web包下创建带有@Controller注解的控制器，这样的好处在于我们在新建一个控制器时，不用再花精力在配置类中进行Bean维护。虽然Spring提供了默认的视图解析器，但有时最简单的并不是最适合的，配置我们自己定义的视图解析器可以帮助我们避免一些开发中不必要的问题，例如视图不匹配的情况。

#### 编写控制器
铺垫了这么多，接下来我们看一下控制器的编写。这是我们熟悉的领域。
```
@Controller
public class HomeController {
  @RequestMapping(value = "/home", method=RequestMethod.GET)
  public String home() {
    return "home";
  }
}
```
通过@Controller注解，前文WebConfig配置类即可扫描到该控制器，大部分时候我们也可使用@Component注解替换（@Controller基于@Component实现），但我不推荐这么做，毕竟@Controller具有解释性意义。而@RequestMapping可以指定方法以及类的映射路径，还可用于指定请求的类型。

home()方法返回了一个String，它会被Spring MVC解读为要渲染的视图名，所以下一步，我们需要在路径"/WEB-INF/views/"下新增一个home.jsp文件
```
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ page session="false" %>
<html>
  <head>
    <title>2yLoo'Home Page</title>
    <link rel="stylesheet"
          type="text/css"
          href="<c:url value="/resources/style.css" />" >
  </head>
  <body>
    <h1>2yLoo'Home Page</h1>
    <a href="<c:url value="/hello" />">Say Hello</a>
  </body>
</html>
```

这样，通过请求"localhost:8080/home"即可访问到我们的/WEB-INF/views/home.jsp文件。当我们没有添加指定jsp文件时，我们的请求会返回一个404的```Whitelabel Error Page```。将控制器请求处理的逻辑和视图渲染解耦是Spring MVC的一个重要特性。原书《Spring 实战》基于JSP或Thymeleaf作为视图层展开了许多内容，而移动互联网崛起，JSP的应用受限，接下来我会结合书本上的知识与实际应用开发总结一下其他我感受到的Web项目。

### Spring Web注解应用
#### @RequestMapping
我们在Controller中经常会要用到这个注解，它可以用在类级别与方法级别上使用。通过源码更好的了解这个类的作用：
```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
    String name() default "";

    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    RequestMethod[] method() default {};

    String[] params() default {};

    String[] headers() default {};

    String[] consumes() default {};

    String[] produces() default {};
}
```
通过我们的自定义配置，我们可以通过路由路径、请求类型、请求头与请求参数详情来更明确的指定一个Controller方法的职责范围。

除此之外，@RequestMapping根据请求类型衍生出5种子注解：
```
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@PatchMapping
```
亲身经历告诉我开发中应尽量指定请求类型，否则后续的对接问题将变得更为棘手。

#### @RequestParam
这是我们经常与@RequestMapping搭配使用的一个注解，将它运用在传参上即可对方法参数进一步的细化处理。具体来说，我们通过此参数来映射请求的Params，并且可以指定是否必传以及其默认值。默认情况下携带@RequestParam的注解是不要求必传的。

#### @RequestBody
使用@RequestBody注解帮助我们把请求体中的内容装配到指定对象上，并可以与@RequestParam搭配使用。

#### @ResponseBody
回顾前面编写控制器的例子，在home方法中，我们返回了字符串"home"，这样的返回会被解析为跳转路径。而在移动互联网场景下，有时我们需要的是返回"home"字符串（运用更宽泛的场景下是返回指定复杂对象），使用@RequestBody注解即可实现我们的需求。

#### @RestController
如果我们整个Controller类中的方法都是基于@ResponseBody返回的，那么可以直接在类级别上用@RestController替换@Controller，从而省去方法上的@ResponseBody注解。

#### @Valid
时常我们在控制层会做一些参数校验，以让不满足要求的参数直接返回，而不是进行下一步的业务逻辑执行。而其中有一大部分的校验是比较死板的，@Valid帮助我们的代码更加优美简洁，通过注释的方式完成部分参数校验。

以下以一个添加用户的例子来展示部分注解的实际应用：
```
@RestController
public class UserController {

    @RequestMapping(value = "/add", method = RequestMethod.POST)
    public String add(@RequestBody @Valid User user){
        System.out.println(user);
        return "Success";
    }

}

public class User {
    @Null
    private String id;

    @NotNull
    private String name;

    @Email
    private String email;

    @Digits(integer = 100, fraction = 9)
    private float age;

    @AssertFalse
    private boolean single;

    @Past
    private Date birthday;

    // Getter & Setter
}
```

在Postman的Headers下设置```Content-Type```的值为```application/json;charset=UTF-8```，然后在Body中选择raw，并填入Json数据：
```
{
    "name": "2yLoo",
    "email": "yamolv@qq.com",
    "age": 24.2,
    "single": false,
    "birthday": "1996-02-18"
}
```
在这个简单的例子，我们的add()方法只接收Post请求，请求路径为localhost:8080/add，返回"Success"状态而不是一个JSP路径地址。不仅如此，我们要求id为null、name不为null、email字段为Email格式、年龄的整数部分小于100，小数部分小于9、只接受非单身的用户，并且生日为过去的时间···如果不这么做，请求将返回400 Bad Request错误。如果我们需要自定义的ResponseCode来通知请求者，也许它会显得乏力，但它在我眼里还是一个很强大的注解。

合理使用这些注解可以使我们的开发事半功倍。其中最吸引我的@Valid还有更多用法，这些校验注解都放在```javax.validation.constraints```包下，感兴趣的可以继续探索~
