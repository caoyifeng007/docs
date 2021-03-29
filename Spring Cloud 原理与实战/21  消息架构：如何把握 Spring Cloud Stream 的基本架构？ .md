[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4766)



> 上一课时中，我们介绍了事件驱动架构的基本原理，以及 Spring 中对消息传递机制的抽象和对应的开发框架。要想在 SpringHealth 案例系统中添加消息发送和接收的效果有很多种实现方法，我们完全可以直接使用诸如 RabbitMQ、Kafka 等消息中间件来实现消息传递，这种解决方案的主要问题在于需要开发人员考虑不同框架的使用方式以及框架之间存在的功能差异性。而 Spring Cloud Stream 则不同，它在内部整合了多款主流的消息中间件，为开发人员提供了一个平台型解决方案，从而屏蔽各个消息中间件在技术实现上的差异。在今天的内容中，我将首先介绍 Spring Cloud Stream 的基本架构，并给出它与目前主流的各种消息中间件之间的整合机制。
>
> ### Spring Cloud Stream 基本架构
>
> Spring Cloud Stream 对整个消息发布和消费过程做了高度抽象，并提供了一系列核心组件。我们先介绍通过 Spring Cloud Stream 构建消息传递机制的基本工作流程。区别于直接使用 RabbitMQ、Kafka 等消息中间件，Spring Cloud Stream 在消息生产者和消费者之间添加了一种桥梁机制，所有的消息都将通过 Spring Cloud Stream 进行发送和接收，如下图所示：
>
> ![图片1.png](https://s0.lgstatic.com/i/image/M00/73/90/Ciqc1F_GFU2Ae0qTAAHKrvTf9sk254.png)
>
> Spring Cloud Stream 工作流程图
>
> 在上图中，我们不难看出 Spring Cloud Stream 具备四个核心组件，分别是 Binder、Channel、Source 和 Sink，其中 Binder 和 Channel 成对出现，而 Source 和 Sink 分别面向消息的发布者和消费者。
>
> - Source 和 Sink
>
> 在 Spring Cloud Stream 中，Source 组件是真正生成消息的组件，相当于是一个输出（Output）组件。而 Sink 则是真正消费消息的组件，相当于是一个输入（Input）组件。根据我们对事件驱动架构的了解，对于同一个 Source 组件而言，不同的微服务可能会实现不同的 Sink 组件，分别根据自身需求进行业务上的处理。
>
> 在 Spring Cloud Stream 中，Source 组件使用一个普通的 POJO 对象来充当需要发布的消息，通过将该对象进行序列化（默认的序列化方式是 JSON）然后发布到 Channel 中。另一方面，Sink 组件监听 Channel 并等待消息的到来，一旦有可用消息，Sink 将该消息反序列化为一个 POJO 对象并用于处理业务逻辑。
>
> - Channel
>
> Channel 的概念比较容易理解，就是常见的通道，是对队列的一种抽象。根据上一课时所讨论的结果，我们知道在消息传递系统中，队列的作用就是实现存储转发的媒介，消息生产者所生成的消息都将保存在队列中并由消息消费者进行消费。通道的名称对应的往往就是队列的名称。
>
> - Binder
>
> Spring Cloud Stream 中最重要的概念就是 Binder。所谓 Binder，顾名思义就是一种黏合剂，将业务服务与消息传递系统黏合在一起。通过 Binder，我们可以很方便地连接消息中间件，可以动态的改变消息的目标地址、发送方式而不需要了解其背后的各种消息中间件在实现上的差异。关于 Binder 是如何与不同的消息中间件进行整合的实现原理我们在后续课程中会有源码级的专题进行讨论。
>
> ### Spring Cloud Stream 集成 Spring 消息处理机制
>
> 结合上一课时中了解到的关于 Spring Messaging 和 Spring Integration 的相关概念，我们就不难理解 Spring Cloud Stream 中关于 Source 和 Sink 的定义。Source 和 Sink 都是接口，其中 Source 接口的定义如下
>
> 复制代码
>
> ```
> import org.springframework.cloud.stream.annotation.Output;
> import org.springframework.messaging.MessageChannel;
> 	 
> public interface Source {
>  
>     String OUTPUT = "output";
>  
>     @Output(Source.OUTPUT)
>     MessageChannel output();
> }
> ```
>
> 注意到这里通过 MessageChannel 来发送消息，而 MessageChannel 类来自 Spring Messaging 组件。我们在 MessageChannel 上发现了一个 @Output 注解，该注解定义了一个输出通道。
>
> 类似的，Sink 接口定义如下：
>
> 复制代码
>
> ```
> import org.springframework.cloud.stream.annotation.Input;
> import org.springframework.messaging.SubscribableChannel;
> 	 
> public interface Sink{
>  
>     String INPUT = "input";
>  
>     @Input(Sink.INPUT)
>     SubscribableChannel input();
> }
> ```
>
> 同样，这里通过 Spring Messaging 中的 SubscribableChannel 来实现消息接收，而 @Input 注解定义了一个输入通道。
>
> 注意到 @Input 和 @Output 注解使用通道名称作为参数，如果没有名称，会使用带注解的方法名字作为参数，也就是默认情况下分别使用“input”和“output”作为通道名称。从这个角度讲，一个 Spring Cloud Stream 应用程序中的 Input 和 Output 通道数量和名称都是可以任意设置的，我们只需要在这些通道的定义上添加 @Input 和 @Output 注解即可。例如在如下所示的代码中，我们定义了 SpringHealthChannel 接口并声明了一个 Input 通道和两个 Output 通道，说明使用该通道的服务会从外部的一个通道中获取消息并向外部的两个通道发送消息：
>
> 复制代码
>
> ```
> public interface SpringHealthChannel {
> 	 
> 	    @Input
> 	    SubscribableChannel input1();
> 	 
> 	    @Output
> 	    MessageChannel output1();
> 	 
> 	    @Output
> 	    MessageChannel output2();
> }
> ```
>
> 可以看到上述接口定义中同时使用到了 Spring Messaging 中的 SubscribableChannel 和 MessageChannel。Spring Cloud Stream 对 Spring Messaging 和 Spring Integration 提供了原生支持。在常规情况下，我们不需要使用这些框架中提供的API就能完成常见的开发需求。但如果确实有需要，我们也可以使用更为底层 API 直接操控消息发布和接收过程。
>
> ### Spring Cloud Stream 集成消息中间件
>
> 对于 Spring Cloud Stream 而言，最核心的无疑是 Binder 组件。Binder 组件是服务与消息中间件之间的一层抽象，但我们知道各种消息中间件在消息传递机制的设计和实现上存在一定的差异性，那么 Spring Cloud Stream 如何屏蔽这些差异性从而打造自身的消息模型呢？在接下来的内容中，我们将梳理 Spring Cloud Stream 中的消息传递模型，并给出 Binder 与消息中间件如何进行整合的过程。
>
> #### Spring Cloud Stream 中的消息传递模型
>
> Spring Cloud Stream 将消息发布和消费抽象成如下三个核心概念，并结合目前主流的一些消息中间件对这些概念提供了统一的实现方式。
>
> - 发布-订阅模型
>
> 我们知道点对点模型和发布-订阅模型是传统消息传递系统的两大基本模型，其中点对点模型实际上可以被视为发布-订阅模型在订阅者数量为 1 时的一种特例。因此，在 Spring Cloud Stream 中，统一通过发布-订阅模型完成消息的发布和消费，如下所示：
>
> ![图片2.png](https://s0.lgstatic.com/i/image/M00/73/90/Ciqc1F_GFXqAPudEAAHBljDWmY4700.png)
>
> 消息发布-订阅模型示意图
>
> - 消费者组
>
> 设计消费者组（Consumer Group）的目的是应对集群环境下的多服务实例问题。显然，如果采用发布-订阅模式就会导致一个服务的不同实例都消费到了同一条消息。为了解决这个问题，Spring Cloud Stream 中提供了消费者组的概念。一旦使用了消费组，一条消息就只能被同一个组中的某一个服务实例所消费。消费者的基本结构如下图所示（其中虚线表示不会发生的消费场景）：
>
> ![图片3.png](https://s0.lgstatic.com/i/image/M00/73/90/Ciqc1F_GFYKAOmpjAAHSXAgoO8s789.png)
>
> 消费者组结构示意图
>
> - 消息分区
>
> 假如我们希望相同的消息都被同一个微服务实例来处理，但又有多个服务实例组成了负载均衡结构，那么通过上述的消费组概念仍然不能满足要求。针对这一场景，Spring Cloud Stream 又引入了消息分区（Partition）的概念。引入分区概念的意义在于，同一分区中的消息能够确保始终是由同一个消费者实例进行消费。尽管消息分区的应用场景并没有那么广泛，但如果想要达到类似的效果，Spring Cloud Stream 也为我们提供了一种简单的实现方案，消息分区的基本结构如下图所示：
>
> ![图片4.png](https://s0.lgstatic.com/i/image/M00/73/9C/CgqCHl_GFY-Acs5MAAHDDsj-fd8866.png)
>
> 消息分区结构示意图
>
> #### Binder 与消息中间件
>
> Binder 组件本质上是一个中间层，负责与各种消息中间件的交互。目前，Spring Cloud Stream 中集成的消息中间件包括 RabbitMQ和Kafka。在具体介绍如何使用 Spring Cloud Stream 进行消息发布和消费之前，我们先来结合消息传递机制给出 Binder 对这两种不同消息中间件的整合方式。
>
> - RabbitMQ
>
> RabbitMQ 是 AMQP（Advanced Message Queuing Protocol，高级消息队列协议）协议的典型实现框架。在 RabbitMQ 中，核心概念是交换器（Exchange）。我们可以通过控制交换器与队列之间的路由规则来实现对消息的存储转发、点对点、发布-订阅等消息传递模型。在一个 RabbitMQ 中可能会存在多个队列，交换器如果想要把消息发送到具体某一个队列，就需要通过两者之间的绑定规则来设置路由信息。路由信息的设置是开发人员操控 RabbitMQ 的主要手段，而路由过程的执行依赖于消息头中的路由键（Routing Key）属性。交换器会检查路由键并结合路由算法来决定将消息路由到哪个队列中去。下图就是交换器与队列之间的路由关系图：
>
> ![图片5.png](https://s0.lgstatic.com/i/image/M00/73/9C/CgqCHl_GFZiAJmwzAAKqP6viw60190.png)
>
> RabbitMQ 中交换器与队列的路由关系图
>
> 可以看到一条来自生产者的消息通过交换器中的路由算法可以发送给一个或多个队列，从而分别实现点对点和发布订阅功能。同时，我们基于上图也不难得出消费者组的实现方案。因为 RabbitMQ 里每个队列是被消费者竞争消费的，所以指定同一个组的消费者消费同一个队列就可以实现消费者组。
>
> - Kafka
>
> 从架构上讲，在 Kafka 中，生产者使用推模式将消息发布到服务器，而消费者使用拉模式从服务器订阅消息。在 Kafka 中还使用到了 Zookeeper，其作用在于实现服务器与消费者之间的负载均衡，所以启动 Kafka 之前必须确保 Zookeeper 正常运行。同时，Kafka 也实现了消费者组机制，如下图所示：
>
> ![图片6.png](https://s0.lgstatic.com/i/image/M00/73/9C/CgqCHl_GFaOANReSAAKuXoszyk4239.png)
>
> Kafka 消费者分组
>
> 可以看到多个消费者构成了一种组结构，消息只能传输给某个组中的某一个消费者。也就是说，Kafka 中消息的消费具有显式的分布式特性，天生就内置了 Spring Cloud Stream 中的消费组概念。
>
> 在 SpringHealth 案例中，我们将同时基于 RabbitMQ 和 Kafka 来展示 Spring Cloud Stream 的各项功能特性。
>
> ### 小结与预告
>
> Spring Cloud Stream 是 Spring Cloud 中针对消息处理的一款平台型框架，该框架的核心优势在于在内部集成了 RabbitMQ、Kafka 等主流消息中间件，而对外则提供了统一的 API 接入层。通过今天的课程，我们知道 Spring Cloud Stream 是通过 Binder 来实现了这一目标。同时，针对消息处理场景下的消费者分组、消息分区等需求，该框架也内置了抽象层并完成与不同消息中间件之间的整合。
>
> 这里给你留一道思考题：在 Spring Cloud Stream 中，消费者组和消费分区分别用于解决什么应用场景？
>
> 在明确了 Spring Cloud Stream 的基本架构之后，在接下来的两个课时中，我们将介绍如何使用它来实现消息发布者和消费者的详细步骤和方法。