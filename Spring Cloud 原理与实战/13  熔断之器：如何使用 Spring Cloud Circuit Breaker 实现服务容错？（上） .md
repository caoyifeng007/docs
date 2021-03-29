[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4758)



> 在上一课时中，我们全面梳理了在微服务架构中实现服务容错的设计思想和实现方案，也引出了 Spring Cloud 中专门用于实现服务容错的 Spring Cloud Circuit Breaker 框架。我们知道 Spring Cloud Circuit Breaker 是一个集成性的框架，内部整合了 Netflix Hystrix、Resilience4j、Sentinel 和 Spring Retry 这四款独立的熔断器组件。由于课时有限，我们无意对这四款组件都进行详细的展开，而是更多关注于 Netflix 旗下的 Hystrix，以及受 Hystrix 启发而诞生的 Resilience4j。在今天的课时中，我们先来讨论 Netflix Hystrix。
>
> ### 引入 Hystrix
>
> 要想在微服务中添加对 Netflix Hystrix 的支持，我们首先需要在 Maven 中添加对 spring-cloud-starter-netflix-hystrix 的依赖。过程如下所示：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
> </dependency>
> ```
>
> 另一方面，通过上一课时的介绍，我们也明确了在微服务架构中，服务调用之间势必需要**引入熔断器机制**，**确保服务容错**。所以，Spring Cloud 推出了一个全新的注解，@SpringCloudApplication 注解。该注解用来集成服务治理和服务熔断方面的核心功能。@SpringCloudApplication 注解定义如下所示。
>
> 复制代码
>
> ```
> @Target(ElementType.TYPE)
> @Retention(RetentionPolicy.RUNTIME)
> @Documented
> @Inherited
> @SpringBootApplication
> @EnableDiscoveryClient
> @EnableCircuitBreaker
> public @interface SpringCloudApplication {
> }
> ```
>
> 可以看到 @SpringCloudApplication 是一个组合注解，整合了 @SpringBootApplication、@EnableDiscoveryClient 和 @EnableCircuitBreaker 这三个微服务所需的核心注解。我们可以直接使用该注解来简化代码。因此，从今天开始，在所有的业务服务中，我们都将使用这个新的 @SpringCloudApplication 注解。以 intervention-service 为例，现在的 Bootstrap 类的定义如下所示：
>
> 复制代码
>
> ```
> @SpringCloudApplication
> public class InterventionApplication
> ```
>
> 在 Hystrix 中，最核心的莫过于**HystrixCommand 类**。HystrixCommand 是一个抽象类，只包含了一个抽象方法，即如下所示的 run 方法：
>
> 复制代码
>
> ```
> protected abstract R run() throws Exception;
> ```
>
> 显然，这个方法是让开发人员实现服务容错所需要处理的业务逻辑。在微服务架构中，我们通常在这个 run 方法中添加对远程服务的访问代码。
>
> 同时我们在 HystrixCommand 类中还发现了另一个很有用的方法 getFallback。这个方法用于在 HystrixCommand 子类中设置服务回退函数的具体实现，如下所示：
>
> 复制代码
>
> ```
> protected R getFallback() {
>  
>     throw new UnsupportedOperationException("No fallback available.");
> }
> ```
>
> Hystrix 是一个非常经典而完善的服务容错开发框架，同时支持了上一课时中所提到的服务隔离、服务熔断和服务回退机制。下面的内容，就让我们逐一了解一下这些功能吧。
>
> ### 使用 Hystrix 实现服务隔离
>
> 基于前面对 HystrixCommand 抽象类的理解，我们就可以提供一个该类的子类来实现服务隔离。针对服务隔离，Hystrix 组件在提供了线程池隔离机制的同时，还实现了**信号量隔离**。这里，我们基于最常用的线程池隔离来进行介绍。典型的 HystrixCommand 子类代码风格如下所示：
>
> 复制代码
>
> ```
> public class GetUserCommand extends HystrixCommand<UserMapper> {
> 
>     //远程调用 user-service 的客户端工具类
>     private UserServiceClient userServiceClient;
>  
>     protected GetUserCommand(String name) {
>         super(Setter.withGroupKey(
>             //设置命令组
>             HystrixCommandGroupKey.Factory.asKey("springHealthGroup"))
>                 //设置命令键
>                 .andCommandKey(HystrixCommandKey.Factory.asKey("interventionKey"))
>                 //设置线程池键
>                 .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(name))
>                 //设置命令属性
>                 .andCommandPropertiesDefaults(
>                     HystrixCommandProperties.Setter()
>                         .withExecutionTimeoutInMilliseconds(5000))
>                 //设置线程池属性
>                 .andThreadPoolPropertiesDefaults(
>                     HystrixThreadPoolProperties.Setter()
>                         .withMaxQueueSize(10)
>                         .withCoreSize(2))
>         );
>     }
>  
>     @Override
>     protected UserMapper run() throws Exception {
>         return userServiceClient.getUserByUserName("springhealth_user1");
>     }
>  
>     @Override
>     protected UserMapper getFallback() {
>         return new UserMapper(1L,"user1","springhealth_user1");
>     }
> }
> ```
>
> 上述代码中使用了 Hystrix 中的很多常见的配置项，这些配置项大多数也涉及线程池隔离的相关概念。在 Hystrix 中，从控制粒度上讲，开发人员可以从服务分组和服务本身这两个维度出发，对线程隔离机制进行配置。也就是说我们既可以把一批服务都划分到一个线程池中，也可以把单个服务划分到一个线程池中。上述代码中的 HystrixCommandGroupKey 和 HystrixCommandKey 分别用来配置**服务分组名称**和**服务名称**，然后 HystrixThreadPoolKey 用来配置**线程池**的名称。
>
> 当我们根据需要设置~~了~~分组、服务以及线程池名称后，接下来就需要指定与线程池相关的各个属性。这些属性都包含在 HystrixThreadPoolProperties 中。例如，在上述代码中，我们使用 maxQueueSize 配置线程池队列的最大值，使用 coreSize 配置核心线程池的最大值等。同时，我们也注意到可以使用 withExecutionTimeoutInMilliseconds 配置项来指定请求的超时时间。
>
> 虽然，上述代码有助于我们更好的理解 Hystrix 中线程池隔离实现机制，但在日常开发过程中，一般不建议你通过创建一个 HystrixCommand 子类的方式来实现服务隔离，而是推荐你使用更为简单的 @HystrixCommand 注解。@HystrixCommand 是 Hystrix 为简化开发过程而专门提供的一个注解，定义如下：
>
> 复制代码
>
> ```
> public @interface HystrixCommand {
> 
>     String groupKey() default "";
>     String commandKey() default "";
>     String threadPoolKey() default "";
>     String fallbackMethod() default "";
>     HystrixProperty[] commandProperties() default {};
>     HystrixProperty[] threadPoolProperties() default {};
>     Class<? extends Throwable>[] ignoreExceptions() default {};
>     ObservableExecutionMode observableExecutionMode() default ObservableExecutionMode.EAGER;
>     HystrixException[] raiseHystrixExceptions() default {};
>     String defaultFallback() default "";
> }
> ```
>
> 在上述定义中，我们看到了用于设置分组、服务与线程池名称相关的 groupKey、commandKey 和 threadPoolKey方法，以及与线程池属性相关的 threadPoolProperties 对象。让我们回到案例，并使用 @HystrixCommand 注解进行重构，效果如下：
>
> 复制代码
>
> ```
> @HystrixCommand
> private UserMapper getUser(String userName) {
>  
>     return userClient.getUserByUserName(userName);
> }
> ```
>
> 可以看到这里只使用了 @HystrixCommand 这个注解就完成 HystrixCommand 的创建。当然，我们也可以进一步使用 @HystrixProperty 注解来设置所需的各项属性，如下所示：
>
> 复制代码
>
> ```
> @HystrixCommand(threadPoolKey = "springHealthGroup",
>     threadPoolProperties =
>      {
>          @HystrixProperty(name="coreSize",value="2"),
>          @HystrixProperty(name="maxQueueSize",value="10")
>      }
> )
> private UserMapper getUser(String userName) {
>  
>     return userClient.getUserByUserName(userName);
> }
> ```
>
> 同样可以看到，我们为该 HystrixCommand 设置了 threadPoolKey，也提供了 threadPoolProperties 来设置 coreSize 和 maxQueueSize。
>
> ### 使用 Hystrix 实现服务熔断
>
> 在上一课时中，我们知道熔断器有三个状态，其中**打开**和**半打开状态**会导致**触发熔断机制**。针对服务熔断，我们同样来考虑案例系统中一个服务依赖调用的具体场景，这个场景是对前面介绍的服务隔离的衍生。在 SpringHealth 案例中，我们知道 intervention-service 需要调用 user-service 和 device-service 来生成健康干预记录，该操作的代码流程如下所示：
>
> 复制代码
>
> ```
> public Intervention generateIntervention(String userName, String deviceCode) {
>         logger.debug("Generate intervention record with user: {} from device: {}", userName, deviceCode);
> 
>         Intervention intervention = new Intervention();
>  
>         //获取远程 User 信息
>         UserMapper user = getUser(userName);
>         if (user == null) {
>             return intervention;
>         }
>         logger.debug("Get remote user: {} is successful", userName);
> 
>         //获取远程 Device 信息
>         DeviceMapper device = getDevice(deviceCode);
>         if (device == null) {
>             return intervention;
>         }
>         logger.debug("Get remote device: {} is successful", deviceCode);
>  
>         //创建并保存 Intervention 信息
>         intervention.setUserId(user.getId());
>         intervention.setDeviceId(device.getId());
>         intervention.setHealthData(device.getHealthData());
>         intervention.setIntervention("InterventionForDemo");
>         intervention.setCreateTime(new Date());
>  
>         interventionRepository.save(intervention);
>  
>         return intervention;
> }
> ```
>
> 显然，上述代码中 getUser() 方法和 getDevice() 方法都会涉及微服务之间的相互依赖和调用，示例代码如下所示：
>
> 复制代码
>
> ```
> @Autowired
> private UserServiceClient userClient;
>  
> @Autowired
> private DeviceServiceClient deviceClient;
>  
> @HystrixCommand
> private UserMapper getUser(String userName) {
>     return userClient.getUserByUserName(userName);
> }
>  
> @HystrixCommand
> private DeviceMapper getDevice(String deviceCode) {
>     return deviceClient.getDevice(deviceCode);
> }
> ```
>
> 这里通过注入 UserServiceClient 和 DeviceServiceClient 两个工具类来实现远程调用，这两个工具类都使用了 Ribbon 和 RestTemplate 来实现调用过程的客户端负载均衡，关于客户端负载均衡相关内容我们在《负载均衡：如何使用 Ribbon 实现客户端负载均衡？》中已经进行了介绍。
>
> 在微服务环境下，使用 UserServiceClient 和 DeviceServiceClient 的调用过程可能会出现响应超时等问题，这个时候 intervention-service 作为服务消费者需要做到服务容错。要嵌入 Hystrix 提供的熔断机制，我们只需要在这两个方法上添加 @HystrixCommand 注解即可。在前面的代码我已经做了相应的示例。
>
> 现在我们来模拟一下远程调用超时的场景，调整 getDevice() 方法的代码，通过 Thread.sleep(2000) 来模拟响应时间过长的场景，如下所示。
>
> 复制代码
>
> ```
> @HystrixCommand
> private DeviceMapper getDevice(String deviceCode) {
>  
>         try {
>             Thread.sleep(2000);
>         } catch (InterruptedException e) {
>             e.printStackTrace();
>         }
>  
>         return deviceClient.getDevice(deviceCode);
> }
> ```
>
> 现在我们创建一个端点，如下所示：
>
> 复制代码
>
> ```
> @RestController
> @RequestMapping(value="interventions")
> public class InterventionController {
>     @Autowired
>     private InterventionService interventionService;
> 
>     @RequestMapping(value = "/{userName}/{deviceCode}", method = RequestMethod.POST)
>     public Intervention generateIntervention( @PathVariable("userName") String userName,
>             @PathVariable("deviceCode") String deviceCode) {
>         Intervention intervention = interventionService.generateIntervention(userName, deviceCode);
> 
>         return intervention;
>     }
> }
> ```
>
> 显然，这个端点是用来访问 InterventionService 并生成 Intervention 记录的。现在，让我们访问这个端点：
>
> 复制代码
>
> ```
> http://localhost:8083/interventionss/springhealth_user1/device_blood
> ```
>
> 首先，我们在 intervention-service 的控制台中会看到“java.lang.InterruptedException: sleep interrupted”异常地抛出，而抛出该异常的来源正是 Hystrix。
>
> 然后，我们来查看端点调用的返回值，如下所示：
>
> 复制代码
>
> ```
> {
>      "timestamp":"1601881721343",
>      "status":500,
>      "error":"Internal Server Error",
>      "exception":"com.netflix.hystrix.exception.HystrixRuntimeException",
>      "message":"generate Intervention time-out and fallback failed.",
>      "path":"/interventions/springhealth_user1/device_blood"
>  }
> ```
>
> 在这里，我们发现 HTTP 响应状态为 500，而抛出的异常为 HystrixRuntimeException，从异常信息上可以看出引起该异常的原因是超时。事实上，默认情况下，添加了 @HystrixCommand 注解的方法调用超过了 1000 毫秒就会触发超时异常，显然上例中设置的 2000 毫秒满足触发条件。
>
> 和设置线程池属性一样，在 HystrixCommand 中我们也可以对熔断的超时时间、失败率等各项阈值进行设置。例如我们可以在 getDevice() 方法上添加如下配置项以改变 Hystrix 的默认行为：
>
> 复制代码
>
> ```
> @HystrixCommand(commandProperties = {
>             @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
> })
> private DeviceMapper getDevice(String deviceCode)
> ```
>
> 上面示例中的 execution.isolation.thread.timeoutInMilliseconds 配置项就是用来设置 Hystrix 的超时时间，现在我们把它设置成 3000 毫秒。这时，我们再次访问 [http://localhost:8083/interventions/springhealth_user1/device_blood](http://localhost:8083/interventionsorders/springhealth_user1/device_blood)端点，就会发现请求会正常返回。当然，Hystrix 还提供了一系列的配置项来细化对熔断器的控制。常见的配置项如下所示：
>
> 复制代码
>
> ```
> @HystrixCommand(commandProperties = {
>             @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "12000"),
>             //一个滑动窗口内最小的请求数
>             @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
>             //错误比率阈值
>             @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "75"),
>             //触发熔断的时间值
>             @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "7000"),
>             //一个滑动窗口的时间长度
>             @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "15000"),
>             //一个滑动窗口被划分的数量
>             @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "5") })
> ```
>
> 我们在后续介绍 Hystrix 熔断器实现原理和滑动窗口机制时会对这些配置项的作用做进一步展开。
>
> ### 使用 Hystrix 实现服务回退
>
> Hystrix 在服务调用失败时都可以执行服务回退逻辑。在开发过程上，我们只需要提供一个 Fallback 方法实现并进行配置即可。例如，在 SpringHealth 案例系统中，对于 intervention-service 中访问 user-service 和 device-service 这两个远程调用场景，我们都可以实现 Fallback 方法。回退方法的实现也非常方便，唯一需要注意的就是 Fallback 方法的参数和返回值必须与真实的方法完全一致。如下所示的就是 Fallback 方法的一个示例：
>
> 复制代码
>
> ```
> private UserMapper getUserFallback(String userName) {
>  
>         UserMapper fallbackUser = new UserMapper(0L,"no_user","not_existed_user");
> 
>         return fallbackUser;
> }
> ```
>
> 我们通过构建一个不存在的 User 信息来返回 Fallback 结果。有了这个 Fallback 方法，剩下来要做的就是在 @HystrixCommand 注解中设置“fallbackMethod”配置项。重构后的 getUser 方法如下所示：
>
> 复制代码
>
> ```
> @HystrixCommand(threadPoolKey = "springHealthGroup",
>     threadPoolProperties =
>          {
>              @HystrixProperty(name="coreSize",value="2"),
>              @HystrixProperty(name="maxQueueSize",value="10")
>          },
>          fallbackMethod = "getUserFallback"
> )
> private UserMapper getUser(String userName) {
>  
>         return userClient.getUserByUserName(userName);
> }
> ```
>
> 现在你可以模拟远程方法调用的各种异常情况，并观察这个 Fallback 是否已经生效了。
>
> ### 小结与预告
>
> 本课时对 Hystrix 这款服务容错实现框架进行了详细了讨论，并结合 SpringHealth 案例系统给出了使用该框架的示例代码。Hystrix 是服务容错领域的代表性框架，包含了服务隔离、服务容错和服务回退功能，值得你进行深入的理解并掌握运用。为了帮助你更好的学习 Hystrix 框架，我们在后续课时中还会专门从源码级别分析他的实现原理。
>
> 这里给你留一道思考题：你能说出 Hystrix 中有哪些常见的配置项吗？
>
> 讲完 Hystrix，我们将进一步讲解 Spring Cloud Circuit Breaker 框架。我们知道 Spring Cloud Circuit Breaker 中集成了多种服务容错框架，其中包括 Hystrix 却也不仅包括 Hystrix。下一课时，我们将首先探讨 Spring Cloud Circuit Breaker 中对服务容错的抽象机制，并完成对 Hystrix 使用方式的重构，以及介绍另一款主流的服务熔断工具 Resilience4j。