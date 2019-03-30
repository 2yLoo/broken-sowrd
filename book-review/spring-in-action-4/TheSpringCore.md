## Spring，启动！

Spring的核心即依赖注入（Dependency Injection, DI）与面向切面编程（Aspect-Oriented Programming, AOP）。它从4种关键策略入手

- 基于POJO的轻量级和最小侵入编程
- 通过依赖注入和切面面向接口实现松耦合
- 基于切面和惯例进行声明式编程
- 通过切面和模板减少样板式代码

从这两点出发，Spring搭建出一套以简单性、可测试性和松耦合为切入点的模块全家桶。

### 依赖注入
使用依赖注入带来的最大收益即松耦合。如果一个对象只通过接口来表明依赖关系，这种依赖即可在对象本身毫不知情的情况下，用不同的实现方式进行替换。

本书第一个吸引到我的例子便是关于骑士冒险的例子。骑士（类比于Controller）需要执行Quest（类比于Service）任务，而Quest任务是一个接口。通过依赖注入的方式，在我们初始化骑士的时候，不需要对Quest进行紧耦合的初始化实现，我们十分轻巧的就可使用到Quest提供的功能。

依赖注入可分为3个步骤： 创建Bean -> 将其扫描到Spring容器中 -> 在我们需要的地方进行注入。实现了基础的DI功能后，我们还可根据需要进行进一步的配置。

#### 实现注入

我们使用依赖注入的第一步即声明一个Bean到Spring容器中。有3种实现方式：
1. 通过组件扫描与自动装配的方式实现DI

  这是我们应该首先考虑的一种实现方式。通过两个注解实现：@Component与@ComponentScan。其中@Component将目的Bean声明工作准备好，再通过@ComponentScan扫描目的目录，从而将带有@Component注解的类添加到Spring容器中。

2. 通过Java Config配置类声明Bean

  第一种实现方式也有自己的局限性：至少我们需要可以在Bean上添加相应注释才行。而有时我们需要第三方库的Bean，这时候Java Config就登场了。通过@Configuration注解，Spring会扫描该类中带有@Bean注解的方法，并将返回的对象保存到容器中。

3. 通过XML配置Bean

  XML陪伴了Spring许久许久，并且不限于DI，在切面、数据连接等场景下也能完成工作。但现在基于Spring Boot开发时，XML的应用场景很少，虽然本书常有提及，仅作为了解，此处不进行详细的阐述。

在配置Bean时，我们应先考虑自动配置与Java配置这两种实现方式。完成Bean的声明后，通过@Autowired注解就可实现Bean的自动装配。

#### 依赖注入进阶

许多时候我们在开发过程中止步于此，但Spring也为我们考虑到Bean的生效环境、Bean名冲突等问题，让我们能将此技术灵活运用。

我们或许会遇到这样的场景：我们配置了一个Redis缓存，但仅希望它在生产环境生效；或者我们的Excel解析类在Windows/Linux上初始化执行的语句不同···

Spring提供了优雅的实现方式，让我们可以在运行时决定某个Bean是否创建：

1. 使用@Profile注解

  通过在Bean上添加该注解来指定该Bean的生效环境，简约方便，但这也限定了它的约束能力，用法如下：
  ```
  @Configuration
  @Profile("prod")
  public class DemoConfig {
    @Bean
    public DemoBean demo { return new DemoBean();}
  }
  ```

2. 使用@Conditional注解

  @Conditional注解提供更强大的条件约束能力

  我们首先需要实现一个Condition接口，该接口只有一个matches方法：
  ```
  public class DemoCondition implements Condition {
    public boolean matches (ConditionContext ctxt, AnnotatedTypeMetadata metadata) {
      return ture;
    }
  }
  ```
  然后在Bean声明处进行注解，仅当DemoCondition的matches方法返回true时该Bean才会被创建
  ```
  @Bean
  @Conditional(DemoCondition.class)
  public DemoBean demo(){ return new DemoBean();}
  ```
  它的强大之处蕴藏在两个参数上，我们可以点进这两个传参的类中查看它能获取什么信息，这些信息也就是我们在运用@Conditional注解可以掌控的点，此处不进行展开。

除此之外，Spring也提供一些方法避免Bean冲突的问题。当我们对一个接口进行了不同的实现并均声明为Bean，如果不做任何处理，在获取Bean时会抛出NoUniqueBeanDefinitionException异常，为了避免这样的问题，我们有几种选择：

1. 使用@Primary声明首选Bean，但道高一尺魔高一丈，混乱的Bean声明可能使@Primary使用在两个同名Bean上。我们应积极处理Bean冲突的问题，而不是单纯的指定首选Bean。

2. 使用@Qualifier限定注入Bean的ID，当找不到该Bean时注入失败。

3. @Qualifier的另一作用就是在Bean声明时指定ID，并且可以作为元注解用在其他自定义注解上。

在开发Spring应用过程中，我们应用的大多数Bean都以单例的形式创建（默认），这样在不同地方调用时都注入同一个Bean实例。我们声明在Control层、Service层、Dao层的Bean通常是无状态的，数据流窜与这些类的函数之间，但是没有给该对象带来状态上的改变，使用单例模式可以节省开销。但如果我们需要类的某些状态，例如获取Session中的key，单例模式可能并不理想。Spring定义了4种作用域：
1. 单例（Singleton）：整个应用中只创建Bean的一个实例。
2. 原型（Prototype）：每次注入或通过Spring应用上下文获取时都创建新的Bean实例。
3. 会话（Session）：在Web中为每个会话创建一个Bean实例。
4. 请求（Request）：在Web中为每个请求创建一个Bean实例。

我们可以通过@Scope注解来指定Bean的作用域。例如：
```
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) // 原型
// @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON) // 单例
// @Scope(WebApplicationContext.SCOPE_SESSION) // 会话
// @Scope(WebApplicationContext.SCOPE_REQUEST) // 请求
public class DemoBean{ ... }

// @Scope也可指定代理的模型，此例中 DemoBean为一个接口类
@Component
@Scope(
  value = WebApplicationContext.SCOPE_REQUEST,
   proxyMode = ScopedProxyMode.INTERFACES)
public DemoBean demoBean{ ... }
```

当我们在依赖注入需要将某个Bean的属性设置为指定值时，也有多种实现方式。硬编码提供最Ugly的开发风格，除此之外，也可考虑XML配置、通过@PropertySource从配置文件加载、SpEL（Spring Expression Language）等方式来实现，在此不进行扩展。

### 面向切面编程
继续以骑士为例，当我们需要一个“吟游诗人（日志）”来记录骑士的英勇事迹，如果每一个骑士携带一个吟游诗人，的确可以实现我们的目的，诗人的记录动作其实是一个可重用部分，我们在塑造一个新的骑士时，将吟游诗人的角色透明化看似更加优雅，并且有利于面对后期的扩展与改动。

面向切面编程可以给我们带来巨大的收益。不仅可以减少代码的冗余，也可让我们把注意力集中在类自身的功能。AOP并不是Spring特有的能力，它描述的是一种思想。AspectJ是一个很强大的AOP框架，而Spring AOP与AspectJ之间有大量协作，为我们的工作提供强有力的技术支持。

日志是切面最常见的应用场景，除此之外，声明式事务、安全和缓存也是不错的切面。除了面向切面编程，我们也可通过继承的形式完成一部分功能的重用，但Java中的继承是单继承（只有接口可以实现多继承），**如果整个应用中都使用相同的基类，死板的继承往往会导致一个脆弱的对象体系！**

书中开门见山的介绍了AOP术语，这些术语十分具有专业性，但总给我一种在玩网游的幻觉，让我对此十分兴奋，不知大家有无同感：

1. 通知（Advice）：在切面做的具体工作逻辑。

2. 连接点（Join Point）：能应用通知的所有地方的统称。

3. 切点（Pointcut）：有助于缩小切面所通知的连接点范围。

4. 切面（Aspect）：通知与切点的结合。

5. 织入（Weaving）：把切面应用到目标对象的过程。
> 在目标对象的生命周期里有多个点可以进行织入：
>
> 编译期：切面在目标类编译是被织入，AspectJ以此方式织入切面
>
> 类加载期： 切面在目标类加载到JVM时被织入，AspectJ5支持此方式
>
> 运行期：切面在运用运行的某个时刻被织入，Spring AOP以此方式织入切面

6. 引入（Introduction）：引入（名词）允许我们向现有类添加新方法或属性。

Spring提供4种类型的AOP支持：

1. 基于代理的经典Spring AOP；

2. 纯POJO切面；

3. @AspectJ注解驱动的切面；

4. 注入式AspectJ切面

Spring AOP构建在动态代理基础之上，**Spring对AOP的支持局限于方法拦截**。我的理解认为，Spring的DI是对Bean多做了一层包装。以切面缓存为例（后续章节介绍），当我们在某个Bean的方法A上添加了@Cachable注解，当外部通过该Bean调用A时，Spring AOP技术帮助我们在查询缓存，查询不到时才进入此方法。而该Bean的B方法对A方法进行调用时，不会触发缓存，这也侧面说明了该点。

Spring支持5种注解来指定切面：

1. @After：通知方法会在目标方法返回或抛出异常后调用

2. @AfterReturning：通知方法会在目标方法成功返回后调用

3. @AfterThrowing：通知方法会在目标方法抛出异常后调用

4. @Around：通知方法会将目标方法封装起来

5. @Before：通知方法会在目标方法调用之前执行

我们举几个买肥宅快乐水的栗子把它们都运用起来：
```
// 先定义一个具有提供七喜功能的售货机接口
public interface VendingMachine {
  void give7Up(int num);  
}

// 声明一个肥宅切面Bean
@Aspect
public class FatNerd {

  // 在此处声明我们的切点名，以方便下面的织入，并且此处可以将切点的参数应用到通知动作中
  @Pointcut(
    "execution(* com.yy.VendingMachine.give7Up(int))" +
    "&& args(num)")
  public void buy7up(int num) { }

  @After("buy7up(num)")
  public void leave(int num) {
    System.out.println("2y买" + num + "瓶快乐水~");
  }

  @AfterReturning("buy7up(num)")
  public void drink(int num) {
    System.out.println("2y成功买到" + num + "瓶快乐水~");
  }

  @AfterThrowing("buy7up(num)")
  public void wtf(int num) {
    System.out.println(num + "瓶快乐水都没有？辣鸡！");
  }

  @Around("buy7up(num)")
  public void drink(ProceedingJoinPoint jp, int num) {
  try {
    System.out.println("2y冷静的想想买瓶装还是罐装~");
    System.out.println("2y摸摸左口袋钱够不够~");
    System.out.println("2y从右口袋掏出100元硬踏踏的钞票塞进售卖机~");

    jp.proceed();

    System.out.println("2y从售卖机拿出" + num + "瓶七喜~");
    System.out.println("2y屁溜屁溜的离开了~");
  } catch (Throwable e) {
    System.out.println("快乐失败 :( ");
  }

  }

  @Before("buy7up(num)")
  public void checkMoney(int num) {
    System.out.println("2y检查一下自己的钱够不够买" + num + "瓶快乐水");
  }

}
```

至此我们的切面已经声明好了，并且从中我们可以看出切面方法是可以获取到切面传参的。但它还只是一个Bean，等着我们激活：
```
@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class FatNerdConfig {

  @Bean
  public FatNerd fatNerd() {
    return new FatNerd();
  }

}
```

Spring AOP的作用不像例子中的单调，它甚至还可给我们的Bean添加新的方法。你绝对想象不到一台售卖机还能充QB，但我们可以在代码的世界里实现它：
```
// 定义我们要引入的行为
public interface Chargeable {
  void chargeQB();
}

@Aspect
public class ChargeableIntroducer {
  @DeclareParents(
    value="com.yy.VendingMachine+",
    defaultImpl=DefaultChargeable.class)
  public static Chargeable chargeable;
}
```

前面提到Spring只支持方法级别的连接点，在它无法满足我们的需求时，可以考虑用更强大的AspectJ框架技术，不过此处不作补充。

Spring AOP帮助我们把分散的某些行为集成到一个可重用的模块中，从而减少代码冗余，让我们在开发时可以更关注一些类本身的行为。现在我们已经了解了Spring的基石：依赖注入与面向切面编程。接下来看一下Spring Web应用的前世今生。
