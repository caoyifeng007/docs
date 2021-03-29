[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4759)



> 在上一课时中，我们系统介绍了 Hystrix 所提供了服务隔离、服务容错和服务回退功能。我们发现这个框架确实非常强大，能够灵活处理服务容错的各种场景。事实上，业界也存在一批类似 Hystrix 的框架。Spring Cloud 基于这些框架的共性，专门抽象并开发了一个 Spring Cloud Circuit Breaker 框架。在今天的课程中，我们将引入 Spring Cloud Circuit Breaker 框架，并给出使用这个框架来满足各种服务容错需求的实现方法。
>
> ### 理解 Spring Cloud Circuit Breaker 中的熔断器抽象
>
> 从命名上看，Spring Cloud Circuit Breaker 是对熔断器抽象，内部集成了多款不同的熔断器实现工具，并基于这些工具提取了统一的 API 供应用程序进行调用。
>
> 为了在应用程序中创建一个熔断器，我们可以使用 Spring Cloud Circuit Breaker 中的工厂类 CircuitBreakerFactory，该工厂类的定义如下所示：
>
> 复制代码
>
> ```
> public abstract class CircuitBreakerFactory<CONF, CONFB extends ConfigBuilder<CONF>> extends AbstractCircuitBreakerFactory<CONF, CONFB> {
>  
>     public abstract CircuitBreaker create(String id);
> }
> ```
>
> 可以看到这是一个抽象类，只有一个 create 方法用来创建一个 CircuitBreaker。CircuitBreaker 是一个接口，约定了熔断器应该具有的功能，该接口定义如下所示：
>
> 复制代码
>
> ```
> public interface CircuitBreaker {
>     default <T> T run(Supplier<T> toRun) {
>         return run(toRun, throwable -> {
>             throw new NoFallbackAvailableException("No fallback available.", throwable);
>         });
>     };
>  
>     <T> T run(Supplier<T> toRun, Function<Throwable, T> fallback);
> }
> ```
>
> 这里用到了函数式编程的一些语法，但我们从方法定义上还是可以明显看出包含了 run() 方法和 fallback() 方法。其中的 Supplier 包含了你希望运行在熔断器中的业务代码，而 Function 则代表着回退方法。对比上一课时中介绍的 HystrixCommand，我们发现两者之间存在明显的对应关系。
>
> 在 Spring Cloud Circuit Breaker 中，分别针对 Hystrix、Resilience4j、Sentinel 和 Spring Retry 这四款框架提供了 CircuitBreakerFactory 抽象类的子类。如果我们想要在应用程序中使用这些工具，首先需要引入相关的 Maven 依赖。以 Resilience4j 为例，对应的 Maven 依赖如下所示：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>                <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
> </dependency>
> ```
>
> 不过有一点需要注意，Hystrix 对应的 Maven 依赖名称并不是像其他三个框架一样是在“spring-cloud-starter-circuitbreaker-”之后添加具体的框架名称，而是使用如下所示的依赖关系：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
> 	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
> </dependency>
> ```
>
> 一旦在代码工程的类路径中添加了 starter，系统就会自动创建 CircuitBreaker。也就是说 CircuitBreakerFactory.create 方法会实例化对应框架的一个 CircuitBreaker 实例。
>
> 在引入具体的开发框架之后，下一步工作就是对它们进行配置。在 CircuitBreakerFactory 的父类 AbstractCircuitBreakerFactory 中，我们发现了如下两个抽象方法：
>
> 复制代码
>
> ```
> //针对某一个 id 创建配置构造器
> protected abstract CONFB configBuilder(String id);
>  
> //为熔断器配置默认属性
> public abstract void configureDefault(Function<String, CONF> defaultConfiguration);
> ```
>
> 这里用到了大量的泛型定义，我们可以猜想，在这两个抽象方法的背后，Spring Cloud Circuit Breaker 会针对不同的第三方框架提供了不同的配置实现过程。我们在后续内容中会基于具体的框架对这一过程做展开讨论，首当其冲的就是 Hystrix 框架。
>
> ### 使用 Spring Cloud Circuit Breaker 集成 Hystrix
>
> 让我们回到 Hystrix，来看看在 Spring Cloud Circuit Breaker 中是如何使用统一编程模式集成 Hystrix。
>
> #### 理解 HystrixCircuitBreakerFactory 和 HystrixCircuitBreaker
>
> 我们首先关注实现了 CircuitBreaker 接口的 HystrixCircuitBreaker 类，如下所示：
>
> 复制代码
>
> ```
> public class HystrixCircuitBreaker implements CircuitBreaker {
>  
>     private HystrixCommand.Setter setter;
>  
>     public HystrixCircuitBreaker(HystrixCommand.Setter setter) {
>         this.setter = setter;
>     }
>  
>     @Override
>     public <T> T run(Supplier<T> toRun, Function<Throwable, T> fallback) {
>  
>         HystrixCommand<T> command = new HystrixCommand<T>(setter) {
>             @Override
>             protected T run() throws Exception {
>                 return toRun.get();
>             }
>  
>             @Override
>             protected T getFallback() {
>                 return fallback.apply(getExecutionException());
>             }
>         };
>         return command.execute();
>     }
> }
> ```
>
> 不难想象，这里应该构建了一个 HystrixCommand 对象，并在该对象原有的 run 和 getFallback 方法中封装了 CircuitBreaker 中的统一方法调用，而最终实现熔断操作的还是 Hystrix 原生的 HystrixCommand。
>
> 然后，我们接着来看 HystrixCircuitBreakerFactory，这个类的实现过程也简洁明了，如下所示：
>
> 复制代码
>
> ```
> public class HystrixCircuitBreakerFactory extends
>         CircuitBreakerFactory<HystrixCommand.Setter, HystrixCircuitBreakerFactory.HystrixConfigBuilder> {
>  
>     //实现默认配置
>     private Function<String, HystrixCommand.Setter> defaultConfiguration = id -> HystrixCommand.Setter
>             .withGroupKey(
>                     HystrixCommandGroupKey.Factory.asKey(getClass().getSimpleName()))
>             .andCommandKey(HystrixCommandKey.Factory.asKey(id));
>  
>     public void configureDefault(
>             Function<String, HystrixCommand.Setter> defaultConfiguration) {
>         this.defaultConfiguration = defaultConfiguration;
>     }
>  
>     public HystrixConfigBuilder configBuilder(String id) {
>         return new HystrixConfigBuilder(id);
>     }
>  
>     //创建熔断器
>     public HystrixCircuitBreaker create(String id) {
>         Assert.hasText(id, "A CircuitBreaker must have an id.");
>         HystrixCommand.Setter setter = getConfigurations().computeIfAbsent(id,
>                 defaultConfiguration);
>         return new HystrixCircuitBreaker(setter);
>     }
>  
>     //
>     public static class HystrixConfigBuilder
>             extends AbstractHystrixConfigBuilder<HystrixCommand.Setter> {
>  
>         public HystrixConfigBuilder(String id) {
>             super(id);
>         }
>  
>         @Override
>         public HystrixCommand.Setter build() {
>             return HystrixCommand.Setter.withGroupKey(getGroupKey())
>                     .andCommandKey(getCommandKey())
>                     .andCommandPropertiesDefaults(getCommandPropertiesSetter());
>         } 
>     }
> }
> ```
>
> 上述代码基本就是对原有 HystrixCommand 中关于服务分组等属性的简单封装，你可以结合上一课时的内容做一些回顾。
>
> #### 使用 HystrixCircuitBreakerFactory 设置默认属性
>
> 在应用程序中为熔断器创建默认配置，我们可以使用 Spring Cloud Circuit Breaker 提供的 Customizer工具类。通过传入一个 HystrixCircuitBreakerFactory 对象，然后调用它的 configureDefault 方法就可以构建一个 Customizer 实例。示例代码如下所示：
>
> 复制代码
>
> ```
> Bean
> public Customizer<HystrixCircuitBreakerFactory> defaultConfig() {
>         return factory -> factory.configureDefault(id -> HystrixCommand.Setter
>                 .withGroupKey(HystrixCommandGroupKey.Factory.asKey(id))
>              .andCommandPropertiesDefaults(HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(3000)));
> }
> ```
>
> 这段代码比较容易理解，我们看到了熟悉的服务分组键 GroupKey，以及 Hystrix 命令属性 CommandProperties。这里同样通过 HystrixCommandProperties 的 withExecutionTimeoutInMilliseconds 方法将默认超时时间设置为 3000 毫秒。
>
> 以上方法一般推荐放置在 Spring Boot 的启动类中，这样相当于对 HystrixCircuitBreakerFactory 进行了初始化，接下来就可以使用它来完成服务熔断操作了。
>
> ### 使用 Hystrix 实现服务熔断
>
> 使用 HystrixCircuitBreakerFactory 实现服务熔断的开发流程比较固化。首先，我们需要通过 HystrixCircuitBreakerFactory 创建一个runCircuitBreaker 实例，然后实现具体的业务逻辑并提供一个回退函数，最后执行 CircuitBreaker 的 run 方法。示例代码如下：
>
> 复制代码
>
> ```
> //创建 CircuitBreaker
> CircuitBreaker hystrixCircuitBreaker = circuitBreakerFactory.create("springhealth");
> 
> //封装业务逻辑
> Supplier<UserMapper> supplier = () -> {
>      return userClient.getUserByUserName(userName);
> };
>  
> //初始化回退函数
> Function<Throwable, UserMapper> fallback = t -> {
>      UserMapper fallbackUser = new UserMapper(0L,"no_user","not_existed_user");
> 
>      return fallbackUser;
> };
> 
> //执行业务逻辑
> hystrixCircuitBreaker.run(supplier, fallback);
> ```
>
> 我们可以把上述示例代码进行调整并嵌入到各种业务场景中。
>
> ### 使用 Spring Cloud Circuit Breaker 集成 Resilience4j
>
> 介绍完 Hystrix，我们接下来再来看另一个非常主流的熔断器实现工具 Resilience4j。
>
> #### Resilience4j 基础
>
> Resilience4j 是一款轻量级的服务容错库，其设计灵感正是来自 Hystrix，我们先来看一下 Resilience4j 中定义的几个核心组件。
>
> 当使用 Resilience4j 时，同样需要对熔断器进行配置。而这样配置信息同样分为两部分，一部分是默认配置，一部分是专属于某一个服务的特定配置。典型的 Resilience4j 配置项如下所示：
>
> 复制代码
>
> ```
> resilience4j:
>   circuitbreaker:
>     configs:
>       default:
>         ringBufferSizeInClosedState: 5 # 熔断器关闭时的缓冲区大小
>         ringBufferSizeInHalfOpenState: 2 # 熔断器半开时的缓冲区大小
>         waitDurationInOpenState: 10000 # 熔断器从打开到半开需要的时间
>         failureRateThreshold: 60 # 熔断器打开的失败阈值
>         eventConsumerBufferSize: 10 # 事件缓冲区大小
>         recordExceptions: # 记录的异常
>           - com.example.resilience4j.exceptions.BusinessBException
>           - com.example.resilience4j.exceptions.BusinessAException
>         ignoreExceptions: # 忽略的异常
>           - com.example.resilience4j.exceptions.BusinessAException
>     instances:
>       userCircuitBreaker:
>         baseConfig: default
>       deviceCircuitBreaker:
>         baseConfig: default
>         waitDurationInOpenState: 5000
>         failureRateThreshold: 20
> ```
>
> 可以看到这里，我们先对**全局熔断器**设置好一系列的默认配置。针对不同的业务服务，我们可以配置多个熔断器实例，并对这些实例使用不同的配置或者直接覆盖默认配置。在上述配置项中，我们初始化了两个服务级的 Circuit Breaker 实例 userCircuitBreaker 和 deviceCircuitBreaker，分别作用于 user-service 和 device-service。其中，userCircuitBreaker 完全使用的是默认配置，而 deviceCircuitBreaker 对 waitDurationInOpenState 和 failureRateThreshold 这两个配置项做了覆盖。
>
> 在 Resilience4j 中，存在一个熔断器注册器 CircuitBreakerRegistry。上述配置项会帮我们把 userCircuitBreaker 和 deviceCircuitBreaker 自动注册到这个 CircuitBreakerRegistry 中。而在应用程序中，通过指定熔断器名称就可以从 CircuitBreakerRegistry 中获取熔断器，如下所示：
>
> 复制代码
>
> ```
> CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("userCircuitBreaker");
> ```
>
> 一旦获取了 CircuitBreaker 对象，接下来就是通过该对象所提供的 executeSupplier 方法或 executeCheckedSupplier 方法来执行业务代码，如下所示：
>
> 复制代码
>
> ```
> circuitBreaker.executeCheckedSupplier(userClient::getUser);
> ```
>
> 如果需要对业务代码执行回退，在 Resilience4j 中的实现过程会相对复杂一点。我们需要使用包装器方法 decorateCheckedSupplier，然后再使用 Try.of().recover() 方法进行降级处理，代码示例如下所示：
>
> 复制代码
>
> ```
> CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("userCircuitBreaker");
>  
> CheckedFunction0<UserMapper> checkedSupplier = CircuitBreaker.
> decorateCheckedSupplier(circuitBreaker, userClient::getUser);
>  
> Try<UserMapper> result = Try.of(checkedSupplier).
> .recover(throwable -> {
>     UserMapper fallbackUser = new UserMapper(0L,"no_user","not_existed_user");
>     return fallbackUser;
> });
> return result.get();
> ```
>
> 至此我们演示了基于 Java 代码的方式来使用 Resilience4j，但 Resilience4j 也提供了 @CircuitBreaker 注解。该注解类似 Hystrix 中的 @HystrixCommand 注解。使用方式上也比较类似，一般只需要指定熔断器的名称以及回退方法即可，如下所示：
>
> 复制代码
>
> ```
> @CircuitBreaker(name = "userCircuitBreaker", fallbackMethod = "getUserFallback")
> ```
>
> #### 使用 Resilience4j 实现服务熔断
>
> 现在，让我们回到 Spring Cloud Circuit Breaker，看看该框架如何对 Resilience4j 的使用过程进行封装和集成。
>
> 首先，我们同样需要构建一个 Customizer 实例，来初始化对 Resilience4j 的配置，如下所示：
>
> 复制代码
>
> ```
> @Bean
> public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
>         return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
>                 .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
>                 .build());
> }
> ```
>
> 上述代码似曾相识，但这里的 Customizer 中包装的是 Resilience4JCircuitBreakerFactory 工厂类。同时，这里也构建了一个 Resilience4JConfigBuilder 用来完成与 Resilience4j 相关配置的构建工作。
>
> 而针对 Resilience4JCircuitBreakerFactory 的使用方法，我们会发现与 HystrixCircuitBreakerFactory 是完全一致的。我们也是先通过 Resilience4JCircuitBreakerFactory 创建 CircuitBreaker，然后封装业务逻辑并初始化回调函数，最后通过 CircuitBreaker 的 run 方法执行业务逻辑。相关代码不再重复展开，这种实现方式也是 Spring Cloud Circuit Breaker 作为一个平台化框架提供统一 API 的价值所在。
>
> ### 服务容错集成 API 网关
>
> 最后，我们还是有必要提在 API 网关中集成服务容错机制的实现方法。我们在前面几个课时中系统介绍了 Netflix Zuul 和 Spring Cloud Gateway 这两款 API 网关实现工具，它们都可以完成与 Hystrix 的无缝集成。
>
> 事实上，Hystrix 集成 API 网关唯一所要做的事情就是**在网关的配置文件中添加与 Hystrix 相关的配置项**即可。这里以常见的设置服务访问超时时间的场景为例，给出 Hystrix 配置项，如下所示：
>
> 复制代码
>
> ```yaml
> hystrix:
>   command:
>     default:
>        execution:
>           isolation:
>              thread:
> 	            timeoutInMilliseconds: 5000
> ```
>
> 显然，上述配置信息的效果就是覆写 Hystrix 的默认超时时间为 5000 毫秒。请注意，以上配置项对经由 API 网关中的所有服务均生效。如果我们想要设置具体某一个服务（例如 userservice）的 Hystrix 超时时间，把“hystrix.command.default”段改为“hystrix.command.userservice”即可。
>
> 对于 API 而言，无论是 Netflix Zuul 还是 Spring Cloud Gateway，上述配置项都是一样的。你可以自己进行尝试使用。
>
> ### 小结与预告
>
> 本课时对 Spring Cloud Circuit Breaker 框架进行了展开，并基于该框架重构了上一课时中针对 Hystrix 的使用方法。然后，我们引入了另一个主流的熔断器框架 Resilience4j，并同样基于 Spring Cloud Circuit Breaker 所提供的统一 API 完成了熔断器实现。同时，在结尾部分，我们还给出了 Hystrix 集成 API 网关的配置方法。作为对主流几款熔断器实现技术的统一抽象和封装，Spring Cloud Circuit Breaker 的设计和实现过程值得我们借鉴。
>
> 这里给你留一道思考题：Spring Cloud Circuit Breaker 是如何对各种不同的 Circuit Breaker 的使用方法进行统一抽象的？
>
> 讲完 Spring Cloud Circuit Breaker 的使用方法，我们有必要对熔断器的实现原理做一定的展开。作为一款强大而完善的熔断器工具，Hystrix 内部使用了滑动窗口机制来对运行时度量数据进行采集和计算，从而实现自动熔断。下一课时，就让我们继续围绕 Hystrix 的内部结构和实现机制展开深入分析。