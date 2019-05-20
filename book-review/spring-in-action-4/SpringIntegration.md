## Spring集成

### 使用远程服务
远程调用是客户端与服务端之间的会话，即我们经常听到的RPC（Remote Procudure Call）。它与本地方法类似，都是同步操作，会阻塞代码执行，直到调用过程执行结束。我们使用Java开发时可选择不同类型的PRC技术：
- 远程方法调用（Remote Method Invocation，RMI）
- Caucho的Hessian和Burlap
- Spring基于HTTP的远程服务
- 使用JAX-PRC和JAX-WS的Web Service

Spring通过多种调用技术支持RPC：

| PRC模型             | 适用场景                                                                                     |
| ------------------- | -------------------------------------------------------------------------------------------- |
| 远程方法调用（RMI） | 不考虑网络限制时（例如防火墙），访问/发布基于Java的服务                                      |
| Hessian或Burlap     | 考虑网络限制时，通过HTTP访问/发布基于Java的服务。Hessian是二进制协议，Burlap基于XML          |
| HTTP invoker        | 考虑网络限制时，并希望使用基于XML或专有序列化机制实现Java序列化时，访问/发布基于Spring的服务 |
| JAX-RPC和JAX-WS     | 访问/发布平台独立、基于SOAP的Web服务                                                         |

Spring针对这几种模型提供了风格一致，掌握了其中一种模型，即可降低其他模型的学习成本。并且易于对模型选型的切换。可以发现上一章讲到Spring持久化技术时，一旦掌握其中一种持久化技术，学习和切换的成本也是非常低的。

基本的远程调用可分为三个模块：客户端模块、服务端模块以及公共模块。其中公共模块负责管理抽象接口以及传输实体类。

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/15fig02.jpg)

如图所示，客户端通过代理来获取服务。再由代理来负责客户端与远程服务的通信。这样子客户端在调用处无需关心服务源在哪，可以当做本地服务类一样进行调用。而且各个模型中的远程异常是不可控因素，代理类会将其捕获并重新抛出非检查型异常RemoteAceessException。

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/15fig03_alt.jpg)

服务端将实现了远程服务接口的Bean发布出去，只需关心发布结果是否成功。

注意：**参与远程服务传输的类需要实现序列化**

#### RMI
RMI是一个具有历史的RPC技术，从JDK1.1开始即被引入到Java中。下面直接通过代码来展示其用法：

1. 在公共模块准备接口与传输实体类
```
// 接口
public interface DemoService {
    UserDTO findByUserName(String name);
}
// DTO
public class UserDTO implements Serializable {

    private String name;

    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserDTO{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
2. 在服务端实现该接口
```
@Service
public class DemoServiceImpl implements DemoService {
    @Override
    public UserDTO findByUserName(String name) {
        System.out.println("RMI方法调用成功！");
        UserDTO dto = new UserDTO();
        dto.setName(name);
        dto.setAge(18);
        return dto;
    }
}
```
3. 在服务端注册该服务
```
@Configuration
public class ServerConfig {
    /* 在写Demo的过程，客户端一直无法获取服务，
       当时服务端启动正常，客户端启动报错，我把重心一直放在客户端，
       后来查看了本机端口占用情况才发现服务端的服务一直没有注册上，
       找到办法通过代码创建注册中心，才将服务注册成功。
       本人第一次使用RMI，感觉这种注册中心的创建方式还是不合适，
       不知道是否有更优雅的实现。*/
    static {
        try {
            LocateRegistry.createRegistry(1099);
        }catch (RemoteException e){
            System.out.println("注册端口创建失败");
            System.out.println(e.getMessage());
        }
    }

    @Bean
    public RmiServiceExporter rmiExporter(DemoService demoService) {
        RmiServiceExporter rmiExporter = new RmiServiceExporter();
        rmiExporter.setService(demoService);
        rmiExporter.setServiceName("DemoService");
        rmiExporter.setServiceInterface(DemoService.class);
        // 设置远程服务注册地址
        rmiExporter.setRegistryHost("localhost");
        // 设置远程服务注册端口
        rmiExporter.setRegistryPort(1099);
        // 设置远程服务传输端口
        rmiExporter.setServicePort(1199);
        return rmiExporter;
    }
}
```
4. 在客户端创建代理
```
@Configuration
public class DemoFactory {

    @Bean
    public RmiProxyFactoryBean demoService() {
        RmiProxyFactoryBean proxyFactoryBean = new RmiProxyFactoryBean();
        proxyFactoryBean.setServiceUrl("rmi://localhost:1099/DemoService");
        proxyFactoryBean.setServiceInterface(DemoService.class);
        return proxyFactoryBean;
    }
}
```
5. 通过依赖注入调用服务
```
public class ClientTest extends RpcClientApplicationTests{

    @Autowired
    private DemoRmiService demoRmiService;

    @Test
    public void test() throws RemoteException{
        UserDTO userDTO = demoRmiService.findByUserName("2yLoo");
        System.out.println(userDTO);
    }
}
```
此处为了方便调试，我直接在测试用例中调用该服务，继承了RpcClientApplicationTests会在测试用例运行前加载项目环境，所以可以激活代理获取到Bean。

RMI虽然可实现远程服务交互，但其使用受防火墙限制，并且要求客户端与服务端都使用相同Java版本开发（RMI使用了Java的序列化机制）。Caucho公司提供了两种解决方案：Hessian与Burlap，它们解决了防火墙问题，使用私有的序列化机制。

#### Hessian 和 Burlap
它们都基于HTTP协议实现。Hessian使用二进制协议，可移植到其他非Java语言的应用中；Burlap基于XML实现，可支持所有能解析XML的语言上。因为其消息传输协议不同，Hessian在带宽上更具优势，而Burlap的可读性更好。

配置步骤与RMI相似，需将对应的服务端Exporter与客户端ProxyFactoryBean替换为HessianServiceExporter/BurlapServiceExporter、HessianProxyFactoryBean/BurlapProxcyFactoryBean。在上面RMI的例子中，客户端访问DemoService时是无法感知其是否是一个远程服务的，这也是面向接口编程的实际应用。当我们切换为Hessian或Burlap时，只需在远程服务的暴露与接入过程动刀。后面介绍到的其他RPC模型也是如此。

也许历史上Hessian与Burlap的确有辉煌的时候，毕竟它解决了部分RMI用户的痛点，但也由于其私有序列化机制，在9102年的现在，它们已经是一个没那么热门的技术，Spring Web可以解决客户端与服务端之间的通信，而服务端内部如果需要远程调用，也有如今大热的Spring Cloud与Dubbo框架任君选择。所以在此不展示Hessian与Burlap的实战案例。但RPC模型的历史进阶过程还是值得了解一下。

#### HttpInvoker
Spring 的HTTP invoker填补了Hessian与Burlap的空白，它也基于HTTP实现（解决了RMI防火墙问题），并使用Java的序列化机制（解放了Hessian与Burlap的序列化痛点）。

使用时对应的服务端Expoter为HttpInvokerServiceExporter，客户端Proxy为HttpInvokerProxyFactoryBean。

不过HTTP invoker并不是没有缺点，它由Spring提供，使用Java的序列化机制，要求客户端与服务端的Java版本相同。

#### JAX-WS
JAX-WS全称为Java API for XML Web WebService。其远程服务被称为端点，端点的声明周期由JAX-WS运行时管理，而不是Spring。

它有两个概念：
- 端点（类），由@WebService标注，类似前面RMI暴露的接口
- 操作（方法）：由@WebMethod标注，类似于暴露的接口中的方法

让端点类继承SpringBeanAutowiringSupport即可使用@Autowire来标注端点的属性。运用这一点，我们可以在端点中装配我们指定的Spring Bean（Service），而在操作中委托其进行具化实现。这是JAX-WS与其他（以上）技术暴露服务的差别。

虽然远程服务在使用上与本地服务无差，但从本质出发它通常比本地服务更低效（网络传输、序列化等消耗），所以还是应限制其调用来规避性能瓶颈。

### 使用Spring Web创建 REST API
REST：以信息为中心的表属性状态转移（Representational State Transfer），一种围绕资源展开的架构风格。简言之，它就是在客户端与服务端之间，将资源的状态以最适合客户端或服务端的形式进行传输。

REST已成为替换传统SOAP Web服务的流行方案。SOAP关注行为和处理，REST关注要处理的数据。

REST与RPC不同，RPC面向服务，关注行为和动作；REST面向资源，强调描述应用程序的事物和名词。

REST也有行为，它们是通过HTTP方法定义的。主要包括GET、POST、PUT、DELETE、PATCH等。这些HTTP方法与CRUD动作的匹配关系如下：
- Create：POST
- Read：GET
- Update：PUT/PATCH
- Delete：DELETE

但这并不是严格限制，只是一种规范。有时服务端也会约定GET匹配以读为主的操作，而POST匹配以写为主的操作。

Spring提供了对REST API的良好支持，包括对REST方法的处理、URL参数化、模型渲染类型（XML、JSON、Atom等）、@RequestBody、@ResponseBody等等方面。

下面这个控制器中的方法即体现了Spring对REST的支持：
```
@RequestMapping(value = "/demo/{value}", method = RequestMethod.GET)
public String demo(@PathVariable String value){
    return "OK";
}
```
还可从RequestMethod的源码查看控制器支持的HTTP方法类型：
```
package org.springframework.web.bind.annotation;

public enum RequestMethod {
    GET,
    HEAD,
    POST,
    PUT,
    PATCH,
    DELETE,
    OPTIONS,
    TRACE;

    private RequestMethod() {
    }
}
```
当我们从后台获取到了一份数据，可选择使用JSON或者HTML等格式展现出去。而资源没有变化，只是它的表述方式发生改变。

Spring提供两个方案来转换资源表述形式：
- 内容协商：选择一个视图，它能将模型渲染为呈现给客户端的表述形式
- 消息转换器：通过一个消息转换器将控制器返回的对象转换为呈献给客户端的表述形式

Spring提供了一个ResponseEntity实体类来加强服务端与客户端之间的通信能力。由于HTTP的状态码与业务无关，我们可以自定义返回码来给客户端提更明确的响应信息（状态码对应信息如有更新应及时与客户端沟通，并且避免用HTTP预留状态码，如2xx、3xx、4xx等）。

Spring还提供了一个RestTemplate模板类，它定义了36个与REST资源交互的方法，对应着HTTP的方法类型。其中有11个独立方法，与25个重载方法。

| 方法              | 描述                                                             |
| ----------------- | ---------------------------------------------------------------- |
| delete()          | 在特定的URL上对资源执行HTTP DELETE操作                           |
| exchange()        | 在URL上执行特定的HTTP方法，返回包含对象的ResponseEntity          |
| execute()         | 在URL上执行特定的HTTP方法，返回一个从响应提映射得到的对象        |
| getForEntity()    | 发送HTTP GET请求，返回的ResponseEntity包含了响应提所映射成的对象 |
| getForObject()    | 发送HTTP GET请求，返回的请求体将映射为一个对象                   |
| headForHeaders()  | 发送HTTP HEAD请求，返回包含特定资源URL的HTTP头                   |
| optionsForAllow() | 发送HTTP OPSIONS请求，返回对特定URL的Allow头信息                 |
| postForEntity()   | POST数据到一个URL，返回包含一个对象的ResponseEntity              |
| postForLocation() | POST数据到一个URL，返回新创建资源的URL                           |
| postForObject()   | POST数据到一个URL，返回根据响应体匹配形成的对象                  |
| put()             | PUT资源到指定URL                                                 |

RestTemplate可以帮助我们实现绝大多数与REST资源交互的操作，而REST只是应用间通信的方法之一，Spring还支持借助消息实现异步通信。

### Spring 异步消息
#### 什么是异步消息？
前面提到的通信机制都是同步通信：

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/17fig01.jpg)

从调用者的角度理解，同步通信即调用后需等待方法返回才能继续执行。大家回想起自己读书的时候，自习时有班主任或老师来镇守班级吗？如果自习铃响了，老师啥也不干，守着学生们自习，自习结束后才回家，这就是“同步”的老师。

异步通信如下：

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/17fig02.jpg)

调用者发出调用后，不等待其返回，直接执行下去。“异步”的老师会怎么做呢？自习铃响时，委托班长组织自习纪律，自己就可以好好利用这个时间。

在异步通信时，调用者会将消息交给一个服务，再由服务确保将消息投递给被调用者。异步消息有两个主要概念：消息代理（Message Broker）与目的地（Destination）。通过代理的工作，来解耦调用者与被调用者之间的关系。现在广为人知的消息代理（很多时候称为Message Queue，MQ，消息中间件）包括RabbitMQ、RocketMQ、ActiveMQ、Kafka等，它们提供了不同的消息路由模式，但有两种通用模式：队列（Queue）与主题（Topic）。

队列是种点对点的模型，每一条消息都有一个发送者和一个接收者：

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/17fig03_alt.jpg)

主题是发布——订阅模型，每一条消息还是由一个发送者发出，但是订阅了该主题的接收者都会收到该条消息。

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/17fig04_alt.jpg)

同步通信虽然好理解、好搭建，但它的也有弊端：
- 同步通信需等待
- 客户端通过服务接口与远程服务相耦合
- 客户端与远程服务的位置耦合
- 客户端与服务的可用性相耦合

异步通信的优点则有：
- 客户端无需等待消息处理
- 异步通信面向消息，客户端与服务端解耦
- 位置独立，不关心服务的位置
- 确保投递，消费失败的消息将被重新投递

但并不是说异步通信比同步通信更优，它们的侧重点不同，做选择时还是应贴合实际应用场景选择。

#### 使用JMS
Java消息服务（Java Message Service, JMS）是一个Java标准，定义了使用消息代理的通用API。Spring通过基于模板类的抽象为JMS功能提供支持，也就是JmsTemplate。

使用传统的JMS与使用JDBC一样，在实现许多简单工作时，也需要不断做重复的获取、释放链接以及捕获异常。Spring提供的JmsTemplate类似于JdbcTemplate，不仅处理了JMSException异常，并且抛出有信息价值的非检查型异常。并且使用它时我们只用关心发送、接收的消息，而无需频繁获取释放连接。

使用消息队列可分3个步骤实现：
1. 启动服务

  ActiveMQ是一个基于JMS的消息队列。我在Mac上直接使用Homebrew的命令```brew install activemq```进行的安装，其他环境的安装步骤可以[查看官网](https://activemq.apache.org/)。

  安装后运行，运行成功后访问```http://localhost:8161/```即可看到Active的主页。
2. 发送消息

  在发送消息前，需要在项目中集成ActiveMQ。我用了SpringBoot框架，通过ActiveMQ的starter集成：
  ```
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
  </dependency>
  ```
  通过Maven集成后，在零配置的情况下就可使用ActiveMQ（零配置是由Spring Boot的自动默认配置办到的）：
  ```
  @Component
  public class ActiveMQClient {

    @Autowired
    private JmsTemplate jmsTemplate;

    public void send(String message) {
        jmsTemplate.convertAndSend("queue.yy", message);
    }
  }
  ```
3. 接收消息

  使用注解@JmsListener获取指定队列中的消息
  ```
  @Component
  public class ActiveMQServer {
      @JmsListener(destination = "queue.yy")
      public void receive(String message) {
          System.out.println("ActiveMQ 收到消息：" + message);
      }
  }
  ```

很多情况发送一条String的消息不能真切满足我们的需求，则需要使用Spring JMS中的消息转换器：

| 消息转换器                      | 功能                                                                                                                                                       |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MappingJacksonMessageConverter  | 使用Jackson JSON库实现消息与JSON格式之间的互相转换                                                                                                         |
| MappingJackson2MessageConverter | 使用Jackson 2 JSON库实现消息与JSON格式之间的相互转换                                                                                                       |
| MarshallingMessageConverter     | 使用JAXB库实现消息与XML格式之间的相互转换                                                                                                                  |
| SimpleMessageConverter          | 实现String与TextMessage之间的相互转换，字节数组与ByteMessage之间的相互转换，Map与MapMessage之间的相互转换以及Serializable对象与ObjectMessage之间的相互转换 |

默认情况下JmsTemplate在convertAndSend()方法中会使用SimpleMessageConverter，**如果我们传输的是对象，则对应类应实现序列化** 。

JMS是Java应用中主流的消息解决方案，但对Java与Spring开发者来说，JMS并不是唯一的选择。*高级消息队列协议*(Advanced Message Queuing Protocal, AMQP)也得到了Spring的支持。

#### 使用AMQP
AMQP有JMS所不具备的一些特性（优点）。

其一，JMS定义的是API规范，JMS的API协议能确保所有的实现都通过通用的API使用，但不能保证某个JMS实现所发送的消息能被不同的JMS实现所使用。AMQP为消息定义了线路层的协议，该协议规范了消息的格式，消息的生产者与消费者都会遵循此格式，从而AMQP在协作方面要更强大，它不仅能夸不同的AMQP实现，还能跨语言和平台。

其二，AMQP的消息模型更加灵活与透明。前文提到JMS仅有两种模型，点对点与发布-订阅。AMQP不仅支持这两种模型的实现，还提供了更多的发送消息的方式，这是通过生产者与队列解耦来实现的。

AMQP在消息生产与传递之间引入了一种间接机制——Exchange：

![](https://dpzbhybb2pdcj.cloudfront.net/walls5/Figures/17fig08_alt.jpg)

消息的生产者不再关注消息的目的队列，而是和Exchange对接，由Exchange来决定消息的下一站，它们对接时会用到routing key和参数，它由生产者提供。Exchange拿到routing key和参数后会通过已配置的与队列之间绑定的routing key和参数进行对比。并用自己的路由算法来决定消息发送的队列。AMQP定义了4种不同路由算法的Exchange：

| 算法名  | 释义                                                                                    |
| ------- | --------------------------------------------------------------------------------------- |
| Direct  | 如果消息的routing key与binding的routing key直接匹配的话，消息将路由到该队列上           |
| Topic   | 如果消息的routing key与binding的routing key符合通配符匹配的话，消息将会被路由到该队列上 |
| Headers | 如果消息参数表中的头信息和值都与binding参数表中匹配，消息将会路由到该队列上             |
| Fanout  | 不管消息的routing key和参数表的头信息是什么，消息将会路由到所有队列上                   |

其实Exchange就是一层对队列的代理，而通过这层代理，不仅可以轻松实现点对点和发布-订阅的方式，还能解锁更多自定义的路由模式。

RabbitMQ是实现了AMQP协议的一个开源消息队列。站在Spring的肩膀上使用RabbitMQ也非常简单。

启动好RabbitMQ后，通过配置类声明队列：
```
@Configuration
public class RabbitConfig {

    public static final String DEMO_QUEUE = "demo.yy.queue";

    @Bean
    public Queue demoQueue() {
        return new Queue(DEMO_QUEUE);
    }
}
```

消息生产者通过Amqp模板类发送消息：
```
@Component
public class DemoSender {

    private AmqpTemplate rabbitTemplate;

    @Autowired
    public DemoSender(AmqpTemplate amqpTemplate) {
        this.rabbitTemplate = amqpTemplate;
    }

    public void publish(String message){
        this.rabbitTemplate.convertAndSend(RabbitConfig.DEMO_QUEUE, message);
    }
}
```

消费者通过@RabbitHandler与@RabbitListener消费指定队列中的消息：
```
@Component
public class DemoReceiver {

    @RabbitHandler
    @RabbitListener(queues = RabbitConfig.DEMO_QUEUE)
    public void receive(String message){
      System.out.println(message);
    }
}
```

AMQP中的消费者关注点仍然为队列，但消息到达队列前的步骤发生变化，我们通过配置类来配置各个Exchange，在这个节点上将预配置好每个Exchange处理的routing key与队列，而生产者的关注点则从队列转移到routing key上。
