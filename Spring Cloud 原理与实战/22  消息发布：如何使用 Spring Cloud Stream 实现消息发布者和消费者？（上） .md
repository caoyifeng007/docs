[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4767)



> 从上一课时的内容中，我们对 Spring Cloud Stream 的基本架构有了全面的了解。今天，就让我们回到案例，来看看如何使用 Spring Cloud Stream 来完成消息发布者和消费者的构建。
>
> ### 设计 SpringHealth 中的消息发布场景
>
> 在《消息驱动：如何理解 Spring 中对消息处理机制的抽象过程？》课时中，我们已经给出了在 SpringHealth 案例系统中应用消息处理机制的一个典型场景。类似 SpringHealth 这样的系统中的用户信息变动并不会太频繁，所以很多时候我们会想到通过缓存系统来存放用户信息。而一旦用户信息发生变化，user-service 可以发送一个事件，给到相关的订阅者并更新缓存信息，如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/76/EF/CgqCHl_IvYmAAdzPAABKd1VzQCI715.png)
>
> 用户信息更新场景中的事件驱动架构
>
> 一般而言，事件在命名上通常采用过去时态以表示该事件所代表的动作已经发生。所以，我们把这里的用户信息变更事件命名为 UserInfoChangedEvent。通常，我们也会建议使用一个独立的事件消费者来订阅这个事件，就像上图中的 consumer-service1 一样。但为了保持 SpringHealth 系统的简单性，我们不想再单独构建一个微服务，而是选择把事件订阅和消费的相关功能同样放在了 intervention-service 中，如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/76/E4/Ciqc1F_IvgSAQ-k8AABA--dRdWc848.png)
>
> 简化之后的用户信息更新场景处理流程
>
> 接下来我们关注于上图中的事件发布者 user-service。在 user-service 中需要设计并实现使用 Spring Cloud Stream 发布消息的各个组件，包括 Source、Channel 和 Binder。我们围绕 UserInfoChangedEvent 事件给出 user-service 内部的整个实现流程，如下图所示：
>
> ![3.png](https://s0.lgstatic.com/i/image/M00/76/EF/CgqCHl_IvjSAX_W6AAHB35Qu21g693.png)
>
> user-service 消息发布实现流程
>
> 在 user-service 中，势必会存在一个对用户信息的修改操作，这个修改操作会上图中的触发 UserInfoChangedEvent 事件，然后该事件将被构建成一个消息并通过 UserInfoChangedSource 进行发送。UserInfoChangedSource 就是一种 Spring Cloud Stream 中的具体 Source 实现。然后 UserInfoChangedSource 使用默认的名为“output”的 Channel 进行消息发布。在案例中，我们将同时演示 Kafka 和 RabbitMQ，所以 Binder 组件分别封装了这两个消息中间件。
>
> ### 实现消息发布者
>
> 站在消息处理的角度讲，这个消息发布流程并不复杂，主要的实现过程是如何使用 Spring Cloud Stream 完成 Source 组件的创建、Binder 组件的配置以及如何与 user-service 进行集成，让我们一起来看一下。
>
> #### 使用 @EnableBinding 注解
>
> 无论是消息发布者还是消息消费者，首先都需要引入 spring-cloud-stream 依赖，如下所示：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-stream</artifactId>
> </dependency>
> ```
>
> 而在 SpringHealth 案例中，如果我们使用 Kafka 作为我们的消息中间件系统，那么也需要引入 spring-cloud-starter-stream-kafka 依赖，如下所示。
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-stream-kafka</artifactId>
> </dependency>
> ```
>
> 对应的，RabbitMQ 就需要引入 spring-cloud-starter-stream-rabbit 依赖，如下所示：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
> </dependency>
> ```
>
> 对于消息发布者而言，它在 Spring Cloud Stream 体系中扮演着 Source 的角色，[所以我们需要在 user-service 的 Bootstrap 类中标明这个 Spring](mailto:所以我们需要在user-service的Bootstrap类中添加@enablebinding/(Source.class))Boot 应用程序是一个 Source 组件。调整之后的 UserApplication 类如下所示。
>
> 复制代码
>
> ```
> @SpringCloudApplication
> @EnableBinding(Source.class)
> public class UserApplication {
>                   
>     public static void main(String[] args) {
>         SpringApplication.run(UserApplication.class, args);
>     }
> }
> ```
>
> 可以看到，我们在原有 UserApplication 上我们添加了一个 @EnableBinding(Source.class) 注解，该注解的作用就是告诉 Spring Cloud Stream 这个 Spring Boot 应用程序是一个消息发布者，需要绑定到消息中间件，实现两者之间的连接。@EnableBinding 注解定义比较简单，如下所示：
>
> 复制代码
>
> ```
> public @interface EnableBinding {
>     Class<?>[] value() default {};
> }
> ```
>
> 我们可以使用一个或者多个接口作为该注解的参数。在上面的代码中，我们使用了 Source 接口，表示与消息中间件绑定的是一个消息发布者。在下一课时中，我们在介绍消息消费者时同样也会使用到这个 @EnableBinding 注解。
>
> #### 定义 Event
>
> 接下来，需要给出 UserInfoChangedEvent 的定义。对于事件的定义也存在一些通用的做法，事件类型、事件所对应的操作、事件对应的业务领域对象等是一个完整事件定义所必需的元素。因此，我们将 UserInfoChangedEvent 定义如下：
>
> 复制代码
>
> ```
> public class UserInfoChangedEvent{
>  
>     //事件类型
>     private String type;
>     //事件所对应的操作
>     private String operation;
>     //事件对应的领域模型
>     private User user;
> }
> ```
>
> 定义完事件的数据结构之后，接下来我们就需要通过 Source 接口来具体实现消息的发布。
>
> #### 创建 Source
>
> 在 Spring Cloud Stream 中，Source 是一个接口，包含了一个发送消息的 MessageChannel，让我们简单回顾一下该接口的定义，如下所示：
>
> 复制代码
>
> ```
> public interface Source {
>     String OUTPUT = "output";
>  
>     @Output(Source.OUTPUT)
>     MessageChannel output();
> }
> ```
>
> 使用这个接口的方式也很简单，我们只需要在业务代码中直接进行注入即可，就像在使用一个普通的 Javabean 一样。完整的 UserInfoChangedSource 类如下所示：
>
> 复制代码
>
> ```
> import org.springframework.cloud.stream.messaging.Source;
> import org.springframework.messaging.support.MessageBuilder;
> …
> 	 
> @Component
> public class UserInfoChangedSource {
>     private Source source;
>  
>     private static final Logger logger = LoggerFactory.getLogger(UserInfoChangedSource.class);
>   
>     @Autowired
>     public UserInfoChangedSource(Source source){
>         this.source = source;
>     }
>  
> 	private void publishUserInfoChangedEvent(UserInfoOperation operation, User user){
> 	 
>          logger.debug("Sending message for UserId: {}", user.getId());
>      
>         UserInfoChangedEvent change =  new UserInfoChangedEvent(
>             UserInfoChangedEvent.class.getTypeName(),
>             operation.toString(),
>             user);
>  
>         source.output().send(MessageBuilder.withPayload(change).build());
>     }
>     
>     public void publishUserInfoAddedEvent(User user) {
>          publishUserInfoChangedEvent(UserInfoOperation.ADD, user);
>     }
>     
>     public void publishUserInfoUpdatedEvent(User user) {
>          publishUserInfoChangedEvent(UserInfoOperation.UPDATE, user);
>     }
>     
>     public void publishUserInfoDeletedEvent(User user) {
>          publishUserInfoChangedEvent(UserInfoOperation.DELETE, user);
>     }
> }
> ```
>
> 可以看到我们创建了一个 publishUserInfoChangedEvent 方法，在该方法中，我们首先构建了 UserInfoChangedEvent 事件并通过 Spring Messaging 模块所提供的 MessageBuilder 工具类将它转换为消息中间件所能发送的Message对象。然后，我们调用 Source 接口的 output() 方法将事件发送出去，这里的 output() 方法背后使用的就是一个具体的 MessageChannel。
>
> #### 配置 Binder
>
> 为了通过 UserInfoChangedSource 将代表 UserInfoChangedEvent 的消息发送到正确的地址，我们需要在 application.yml 配置文件中配置 Binder 信息。Binder 信息中存在一些通用的配置项，例如如果要想把消息发布到消息中间件，就需要知道消息所发送的通道或者说目的地 Destination，以及序列化方式，如下所示：
>
> 复制代码
>
> ```
> spring:
>   cloud:
>     stream:
>       bindings:
>         output:
>           destination:  userInfoChangedTopic
>           content-type: application/json
> ```
>
> 另一方面，因为 Binder 完成了与具体消息中间件的整合过程，所以需要针对特定的消息中间件来提供专门的配置项。我们先来看在使用 Kafka 的场景下 Binder 的配置方法，相关配置项如下所示：
>
> 复制代码
>
> ```
> spring:
>   cloud:
>     stream:
>       bindings:
>         output:
>           destination:  userInfoChangedTopic
>           content-type: application/json
>       kafka:
>         binder:
>           zk-nodes: localhost
> 	      brokers: localhost
> ```
>
> 在以上配置项中，除了前面介绍的通用配置型之外，因为 Kafka 的运行依赖于 Zookeeper，所以“kafka”配置段使用 Kafka 作为消息中间件平台，并将其 Zookeeper 地址以及 Kafka 自身的地址都指向了本地。
>
> 相比 Kafka，RabbitMQ 的配置稍微复杂一点，如下所示：
>
> 复制代码
>
> ```
> spring:
>   cloud:
>     stream:
>       bindings:
>         default:
>           content-type: application/json
>           binder: rabbitmq
>         output:
>           destination: userInfoChangedExchange
>           contentType: application/json        
>       binders:
>         rabbitmq:
>           type: rabbit
>           environment:
>             spring:
>               rabbitmq:
>                 host: 127.0.0.1
>                 port: 5672
>                 username: guest
>                 password: guest
>                 virtual-host: /
> ```
>
> 在以上配置项中，我们设置了 destination为userInfoChangedExchange 后会在 RabbitMQ  中创建一个名为“userInfoChangedExchange”的交换器，并把 Spring Cloud Stream 的消息输出通道绑定到该交换器。同时，我们在 bindings 配置段中指定了一个“default”子配置段，用于指定默认所使用的 binder。在这个示例中，我们将这个默认 binder 命名为“rabbitmq”并在“binders”配置段中指定了运行 RabbitMQ 的相关参数。请注意 RabbitMQ 和 Kafka 这两款消息中间件在配置方式上各个配置项的层级以及内容上的差别。
>
> #### 集成服务
>
> 最后，我们要做的事情就是在 user-service 中集成消息发布功能。在前一版本的 UserService 类的基础之上，我们添加对 UserInfoChangedSource 的使用过程，如下所示：
>
> 复制代码
>
> ```
> @Service
> public class UserService {
>     
>     @Autowired
>     private UserRepository userRepository;
>  
>     @Autowired
>     private UserInfoChangedSource userInfoChangedSource;
>  
>     public User getUserById(Long userId) {
>         
>         return userRepository.findById(userId).orElse(null); 
>     }
>     
>     public User getUserByUserName(String userName) {
>         
>         return userRepository.findUserByUserName(userName);
>     }
>  
>     public void addUser(User user){
>          userRepository.save(user);
>         
>          userInfoChangedSource.publishUserInfoAddedEvent(user);
>     }
>  
>     public void updateUser(User user){
>          userRepository.save(user);
>      
>          userInfoChangedSource.publishUserInfoUpdatedEvent(user);
>     }
>  
>     public void deleteUser(User user){
>          userRepository.delete(user);
>      
>          userInfoChangedSource.publishUserInfoDeletedEvent(user);
>     }
> }
> ```
>
> 可以看到，我们在增加、修改和删除用户操作时都添加了发布用户信息变更事件的机制。注意到在 UserService 中我们并没有构建具体的 UserInfoChangedEvent 事件，而是把这部分操作放在了 UserInfoChangedSource中，目的也是为了降低各个层次之间的依赖关系，并封装对事件的统一操作。
>
> 至此，完整的消息发布者实现完毕。接下来，我们来看看消息消费场景应该如何进行设计。
>
> ### 设计 SpringHealth 中的消息消费场景
>
> 我们继续讨论 SpringHealth 案例，根据整个消息交互流程，intervention-service 就是 UserInfoChangedEvent 事件的消费者。作为该事件的消费者，intervention-service 需要把变更后的用户信息更新到缓存中。
>
> 在 Spring Cloud Stream 中，负责消费消息的是 Sink 组件，因此，我们同样围绕 UserInfoChangedEvent 事件给出 intervention-service 内部的整个实现流程，如下图所示：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/76/F0/CgqCHl_IvlKADfCXAAA8UAK4iIs978.png)
>
> intervention-service 消息消费实现流程
>
> 在上图中，UserInfoChangedEvent 事件通过消息中间件发送到 Spring Cloud Stream 中，Spring Cloud Stream 通过 Sink 获取消息并交由 UserInfoChangedSink 实现具体的消费逻辑。可以想象在这个 UserInfoChangedSink 中会负责实现缓存相关的处理逻辑。
>
> 让我们把消息消费过程与 intervention-service 中的业务流程串联起来。我们知道在 intervention-service 中存在 UserServiceClient 类，其核心方法 getUserByUserName 如下所示：
>
> 复制代码
>
> ```
> public UserMapper getUserByUserName(String userName){
>      
>         ResponseEntity<UserMapper> restExchange =
>                 restTemplate.exchange(
>                         "http://zuulservice:5555/springhealth/user/users/username/{userName}",
>                         HttpMethod.GET,
>                         null, UserMapper.class, userName);
>                 
>         UserMapper user = restExchange.getBody();
>         
>         return user;
> }
> ```
>
> 这里我们直接通过调用 user-service 远程获取 User 信息。我们知道用户账户信息变更是一个低频事件，而每次通过 UserServiceClient 实现远程调用的成本很高且没有必要。现在我们可以通过 Spring Cloud Stream 获取用户信息更新的消息了，UserServiceClient 就有了优化的空间。基本思路就是缓存用户信息，并通过消息触发缓存更新，然后我们先从缓存中获取用户信息，只有在缓存中找不到对应的用户信息时才会发起远程调用。下图展示了采用这一设计思想之后的流程图：
>
> ![5.png](https://s0.lgstatic.com/i/image/M00/76/F0/CgqCHl_IvmGADKwmAAFJvHum5Z8651.png)
>
> 用户账户更新流程图
>
> 在上图中，我们看到 user-service 异步发送的 UserInfoChangedEvent 事件会被消费，该消息的处理器 UserInfoChangedSink 所消费，然后 UserInfoChangedSink 将更新后的用户账户信息进行缓存以供 intervertion-service 使用。显然，UserInfoChangedSink 是整个流程的关键。至于如何实现这个 UserInfoChangedSink，我们放在下一课时中进行详细展开并给出代码示例。
>
> ### 小结与预告
>
> 今天，我们基于用户信息更新这一特定业务场景，介绍了使用 Spring Cloud Stream 来完成对 SpringHealth 系统中消息发布消费流程的建模，并提供了针对消息发布者的实现过程。可以看到，只要理解了 Spring Cloud Stream 的基本架构，使用该框架发送消息的开发更多的是配置工作。
>
> 这里给你留一道思考题：在 Spring Cloud Stream 配置不同的 Binder 时，有哪些公共配置项，又有哪些是针对具体消息中间件的特定配置项？
>
> 下一课时将继续讨论基于 Spring Cloud Stream 的开发过程，我们关注于消息消费者的实现，以及自定义消息通道、消费者分组以及消息分区等高级主题的实现方式。