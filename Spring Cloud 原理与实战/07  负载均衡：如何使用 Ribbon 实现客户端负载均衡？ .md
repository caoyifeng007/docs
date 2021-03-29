[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4752)



> 前面我们通过几个课时详细介绍了基于 Eureka 的服务治理构建方式和实现原理。在服务治理技术体系中，服务的发现和调用往往是和**负载均衡**这个概念结合在一起的。Spring Cloud 中同样存在着与 Eureka 配套的负载均衡器，这就是 Ribbon 组件。Eureka 和 Ribbon 的交互方式如下图所示：
>
> ![Lark20201013-151704.png](https://s0.lgstatic.com/i/image/M00/5D/CD/Ciqc1F-FVJKAMpfyAABbXmaXEy0230.png)
>
> Eureka 和 Ribbon 的交互方式示意图
>
> 今天，我们就将结合上图详细介绍如何使用 Ribbon 来实现负载均衡的使用方法。
>
> ### 理解 Ribbon 与 DiscoveryClient
>
> #### Ribbon 的核心功能
>
> Ribbon 组件同样来自 Netflix，它的定位是一款用于提供客户端负载均衡的工具软件。Ribbon 会自动地基于某种内置的负载均衡算法去连接服务实例，我们也可以设计并实现自定义的负载均衡算法并嵌入 Ribbon 中。同时，Ribbon 客户端组件提供了一系列完善的辅助机制用来确保服务调用过程的可靠性和容错性，包括连接超时和重试等。Ribbon 是客户端负载均衡机制的典型实现方案，所以需要嵌入在服务消费者的内部进行使用。
>
> 因为 Netflix Ribbon 本质上只是一个工具，而不是一套完整的解决方案，所以 Spring Cloud Netflix Ribbon 对 Netflix Ribbon 做了封装和集成，使其可以融入以 Spring Boot 为构建基础的技术体系中。基于 Spring Cloud Netflix Ribbon，通过注解就能简单实现在面向服务的接口调用中，自动集成负载均衡功能，使用方式主要包括以下两种：
>
> - **使用 @LoadBalanced 注解。**
>
> @LoadBalanced 注解用于修饰发起 HTTP 请求的 RestTemplate 工具类，并在该工具类中自动嵌入客户端负载均衡功能。开发人员不需要针对负载均衡做任何特殊的开发或配置。
>
> - **使用 @RibbonClient 注解。**
>
> Ribbon 还允许你使用 @RibbonClient 注解来完全控制客户端负载均衡行为。这在需要定制化负载均衡算法等某些特定场景下非常有用，我们可以使用这个功能实现更细粒度的负载均衡配置。
>
> 在今天的课程中，我们会对这两种使用方式做详细展开，而在下一课时中，我们还将会从源码层面讨论其背后的实现机制。事实上，无论使用哪种方法，我们首先需要明确如何通过 Eureka 提供的 DiscoveryClient 工具类查找注册在 Eureka 中的服务，这是 Ribbon 实现客户端负载均衡的基础。上一课时已经介绍了 DiscoveryClient 与 Eureka 服务器的通信机制，今天我们来看看如何通过 DiscoveryClient 获取服务信息。
>
> #### 使用 DiscoveryClient 获取服务实例信息
>
> 通过上一课时，我们知道可以通过 Eureka 提供的 HTTP 端点获取服务的详细信息。基于这一点，假如现在没有 Ribbon 这样的负载均衡工具，我们也可以通过代码在运行时实时获取注册中心中的服务列表，并通过服务定义并结合各种负载均衡策略动态发起服务调用。
>
> 接下来，让我们来演示如何根据服务名称获取 Eureka 中的服务实例信息。通过 DiscoveryClient 可以很容易实现这一点。
>
> 首先，我们获取当前注册到 Eureka 中的服务名称全量列表，如下所示：
>
> 复制代码
>
> ```
> List<String> serviceNames = discoveryClient.getServices();
> ```
>
> 基于这个服务名称列表可以获取所有自己感兴趣的服务，并进一步获取这些服务的实例信息：
>
> 复制代码
>
> ```
> List<ServiceInstance> serviceInstances = discoveryClient.getInstances(serviceName);
> ```
>
> ServiceInstance 对象代表服务实例，包含了很多有用的信息，定义如下：
>
> 复制代码
>
> ```
> public interface ServiceInstance {
>     //服务实例的唯一性 Id
>     String getServiceId();
>     //主机
>     String getHost();
> 	//端口
>     int getPort();
> 	//URI
>     URI getUri();
> 	//元数据
>     Map<String, String> getMetadata();
> 	…
> }
> ```
>
> 显然，一旦获取了一个 ServiceInstance 列表，我们就可以基于常见的随机、轮询等算法来实现客户端负载均衡，也可以基于服务的 URI 信息等实现各种定制化的路由机制。一旦确定负载均衡的最终目标服务，就可以使用 HTTP 工具类来根据服务的地址信息发起远程调用。
>
> 在 Spring 的世界中，访问 HTTP 端点最常见的方法就是使用 RestTemplate 工具类，让我们一起来做一些回顾。在演示 RestTemplate 的使用方法之前，我们先在 SpringHealth 案例的 user-service 添加一个 HTTP 端点，如下所示：
>
> 复制代码
>
> ```
> @RestController
> @RequestMapping(value = "users")
> public class UserController {
>  
>     @RequestMapping(value = "/{userName}", method = RequestMethod.GET)
>     public User getUserByUserName(@PathVariable("userName") String userName) {
>         User user = new User();
>         user.setId(001L);
>         user.setUserCode("mockUser");
>         user.setUserName(userName);
>         return user;
>     } 
> }
> ```
>
> 这是一个典型的 Controller 类，我们构建了一个“users/userName”端点用于根据用户名获取用户详细信息。为了演示简单，这里使用硬编码构造了一个 User 对象并返回。
>
> 然后，我们构建一个测试类来访问这个 HTTP 端点。如果我们能够获取注册中心中的服务定义，我们就可以通过 ServiceInstance 对该服务进行调用，如下所示：
>
> 复制代码
>
> ```
> @Autowired
> RestTemplate restTemplate;
>  
> @Autowired
> private DiscoveryClient discoveryClient;
>  
> public User getUserByUserName(String userName) {
>         List<ServiceInstance> instances = discoveryClient.getInstances("userservice");
>  
>         if (instances.size()==0) 
>          return null;
> 
>         String userserviceUri = String.format("%s/users/%s",instances.get(0).getUri()
> 	.toString(),userName);
> 
>         ResponseEntity<User> user =
>             restTemplate.exchange(userserviceUri, HttpMethod.GET, null, User.class, userName);
> 
>         return result.getBody();
> }
> ```
>
> 可以看到，这里通过 RestTemplate 工具类就可以使用 ServiceInstance 中的 URL 轻松实现 HTTP 请求。在上面的示例代码中，我们通过 instances.get(0) 方法获取的是服务列表中的第一个服务，然后使用 RestTemplate 的 exchange() 方法封装整个 HTTP 请求调用过程并获取结果。
>
> ### 通过 @Loadbalanced 注解调用服务
>
> 如果你掌握了 RestTemplate 的使用方法，那么在 Spring Cloud 中基于 Ribbon 来实现负载均衡非常简单，要做的事情就是在 RestTemplate 上添加一个注解，仅此而已。
>
> 接下来，我们继续使用前面介绍的 user-service 进行演示。因为涉及负载均衡，所以我们首先需要运行至少两个 user-service 服务实例。另一方面，为了显示负载均衡环境下的调用结果，我们在 UserController 中添加日志方便在运行时观察控制台输出信息。重构后的 UserController 的代码如下所示。
>
> 复制代码
>
> ```
> @RestController
> @RequestMapping(value = "users")
> public class UserController {
>  
>     private static final Logger logger = LoggerFactory.getLogger(UserController.class);
>  
>     @Autowired
>     private HttpServletRequest request;
>  
>     @RequestMapping(value = "/{userName}", method = RequestMethod.GET)
>     public User getUserByUserName(@PathVariable("userName") String userName) {
> 
>         logger.info("Get user by userName from port : {} of userservice instance", request.getServerPort());
>  
>         User user = new User();
>         user.setId(001L);
>         user.setUserCode("mockUser");
>         user.setUserName(userName);
>         return user;
>     } 
> }
> ```
>
> 现在我们分别用 8082 和 8083 端口来启动两个 user-service 服务实例，并将它们注册到 Eureka 中。现在 Eureka 监控页面的效果如下所示，我们在上一课时中已经看过这个效果图了。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/5D/D4/CgqCHl-FUQqAXKcfAAB8da4b_EM391.png)
>
> Eureka 中的两个 user-service 实例信息
>
> 准备工作已经就绪，现在让我们来构建 SpringHealth 案例系统的第二个业务微服务。在“案例驱动：如何通过实战案例来学习 Spring Cloud 框架？”中，我们知道 i**ntervention-service 会访问 user-service 以便生成健康干预信息**。对于 user-service 而言，intervention-service 就是它的客户端。我们在 intervention-service 的启动类 InterventionApplication中，通过 @LoadBalanced 注解创建 RestTemplate。现在的 InterventionApplication 类代码如下所示：
>
> 复制代码
>
> ```
> @SpringBootApplication
> @EnableEurekaClient
> public class InterventionApplication {
>  
>     @LoadBalanced
>     @Bean
>     public RestTemplate getRestTemplate(){
>         return new RestTemplate();
>     }
>  
>     public static void main(String[] args) {
>         SpringApplication.run(InterventionApplication.class, args);
>     }
> }
> ```
>
> 对于 intervention-service 而言准备工作已经就绪，现在就可以编写访问 user-service 的远程调用代码。我们在 intervention-service 工程中添加一个新的 UserServiceClient 类并添加以下代码：
>
> 复制代码
>
> ```
> @Component
> public class UserServiceClient {
>  
>     @Autowired
> 	RestTemplate restTemplate;
> 	 
> 	 public UserMapper getUserByUserName(String userName){
> 
>          ResponseEntity<UserMapper> restExchange =
>                 restTemplate.exchange(
>                         "http://userservice/users/{userName}",
>                         HttpMethod.GET,
>                         null, UserMapper.class, userName);
> 
>         UserMapper user = restExchange.getBody();
> 
>         return user;
>     }
> }
> ```
>
> 可以看到以上代码就是注入 RestTemplate，然后通过 RestTemplate 的 exchange() 方法对 user-service 进行远程调用。但是请注意，这里的 RestTemplate 已经具备了客户端负载均衡功能，因为我们在 InterventionApplication 类中创建该 RestTemplate 时添加了 @LoadBalanced 注解。同样请注意，URL“[http://userservice/users/{userName}](http://userservice/users/{username})”中的”userservice”是在 user-service 中配置的服务名称，也就是在注册中心中存在的名称。至于这里的 UserMapper 类，只是一个数据传输对象，用于完成序列化操作。
>
> 为了验证客户端负载均衡功能是否已经生效，我们同样注入这个 UserServiceClient 类并调用 getUserByUserName 方法来对远程调用进行测试。如果我们多次执行这个方法，那么在两个 user-servce 的服务实例中将交替看到如下日志，这意味着负载均衡已经发生效果：
>
> 复制代码
>
> ```
> INFO [userservice,,] 6148 --- [nio-8081-exec-5] c.t.p.controllers. UserController : Get user by userName from 8082 port of userservice instance
> 	 
> INFO [userservice,,] 6148 --- [nio-8081-exec-5] c.t.p.controllers. UserController: Get user by userName from 8083 port of userservice instance
> ```
>
> ### 通过 @RibbonClient 注解自定义负载均衡策略
>
> 在前面的演示中，我们完全没有感觉到 Ribbon 组件的存在。在基于 @LoadBalanced 注解执行负载均衡时，采用的是 Ribbon 内置的负载均衡机制。默认情况下，Ribbon 使用的是轮询策略，我们无法控制具体生效的是哪种负载均衡算法。但在有些场景下，我们就需要对负载均衡这一过程进行更加精细化的控制，这时候就可以用到 @RibbonClient 注解。Spring Cloud Netflix Ribbon 提供 @RibbonClient 注解的目的在于通过该注解声明自定义配置，从而来完全控制客户端负载均衡行为。@RibbonClient 注解的定义如下：
>
> 复制代码
>
> ```
> public @interface RibbonClient {
>     //同下面的 name 属性
>     String value() default "";
>     //指定服务名称
>     String name() default "";
> 	//指定负载均衡配置类
>     Class<?>[] configuration() default {};
> }
> ```
>
> 通常，我们需要指定这里的目标服务名称以及负载均衡配置类。所以，为了使用 @RibbonClient 注解，我们需要创建一个独立的配置类，用来指定具体的负载均衡规则。以下代码演示的就是一个自定义的配置类 SpringHealthLoadBalanceConfig：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthLoadBalanceConfig{
>  
>     @Autowired
>     IClientConfig config;
>  
>     @Bean
>     @ConditionalOnMissingBean
>     public IRule springHealthRule(IClientConfig config) {
> 
>         return new RandomRule();
>     }
> }
> ```
>
> 显然该配置类的作用是使用 RandomRule 替换 Ribbon 中的默认负载均衡策略 RoundRobin。我们可以根据需要返回任何自定义的 IRule 接口的实现策略，关于 IRule 接口的定义放在下一课时进行讨论。
>
> 有了这个 SpringHealthLoadBalanceConfig 之后，我们就可以在调用特定服务时使用该配置类，从而对客户端负载均衡实现细粒度的控制。在 intervention-service 中使用 SpringHealthLoadBalanceConfig 实现对 user-service 访问的示例代码如下所示：
>
> 复制代码
>
> ```
> @SpringBootApplication
> @EnableEurekaClient
> @RibbonClient(name = "userservice", configuration = SpringHealthLoadBalanceConfig.class)
> public class InterventionApplication{
>  
> 	@Bean
>     @LoadBalanced
>     public RestTemplate restTemplate(){
>         return new RestTemplate();
>     }
>  
>     public static void main(String[] args) {
>         SpringApplication.run(InterventionApplication.class, args);
>     }
> }
> ```
>
> 可以注意到，我们在 @RibbonClient 中设置了目标服务名称为 userservice，配置类为 SpringHealthLoadBalanceConfig。现在每次访问 user-service 时将使用 RandomRule 这一随机负载均衡策略。
>
> 对比 @LoadBalanced 注解和 @RibbonClient 注解，如果使用的是普通的负载均衡场景，那么通常只需要 @LoadBalanced 注解就能完成客户端负载均衡。而如果我们要对 Ribbon 运行时行为进行定制化处理时，就可以使用 @RibbonClient 注解。
>
> ### 小结与预告
>
> 在微服务架构中，每个服务都存在多个运行时实例，所以负载均衡是服务治理的必备组件。Ribbon 是一款典型的客户端负载均衡工具，在与 Eureka 无缝集成的同时，也给开发人员提供了非常友好的使用方式。我们可以使用内嵌了负载均衡机制的 @Loadbalanced 注解完成远程调用，也可以使用 @RibbonClient 注解实现自定义的负载均衡策略。
>
> 这里给你留一道思考题：基于 DiscoveryClient 接口，如果让你设计一款简单的负载均衡工具，你会怎么做？
>
> 今天我们对 Ribbon 的使用方式进行了介绍，可以看到这一过程非常简单。下一课时我们将讨论为什么 Ribbon 能够提供如此简单的使用方式，这就涉及 Ribbon 的内置结构和实现原理。