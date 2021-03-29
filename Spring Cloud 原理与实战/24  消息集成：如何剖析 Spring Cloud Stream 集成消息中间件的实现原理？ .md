[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4769)



> Spring Cloud Stream 中的内容比较多，今天我们重点关注的是如何实现 Spring Cloud Stream 与其他消息中间件的整合过程，因此只介绍消息发送和接收的主流程。我们将分别从**Spring Cloud Stream**以及**消息中间件**的角度出发，分析如何基于这一主流程，完成两者之间的无缝集成。
>
> ### Spring Cloud Stream 中的 Binder
>
> 通过前面几个课时的介绍，我们明确了 Binder 组件是 Spring Cloud Stream 与各种消息中间件进行集成的核心组件，而 Binder 组件的实现过程涉及一批核心类之间的相互协作。接下来，我们就对 Binder 相关的核心类做源码级的展开。
>
> #### BindableProxyFactory
>
> 我们知道在发送和接收消息时，需要使用 @EnableBinding 注解，该注解的作用就是告诉 Spring Cloud Stream 将该应用程序绑定到消息中间件，从而实现两者之间的连接。我们来到 org.springframework.cloud.stream.binding 包下的 BindableProxyFactory 类。根据该类上的注释，BindableProxyFactory 是用于初始化由 @EnableBinding 注解所提供接口的工厂类，该类的定义如下所示：
>
> 复制代码
>
> ```
> public class BindableProxyFactory implements MethodInterceptor, FactoryBean<Object>, Bindable, InitializingBean
> ```
>
> 注意到 BindableProxyFactory 同时实现了 MethodInterceptor 接口和 Bindable 接口。其中前者是 AOP 中的方法拦截器，而后者是一个标明能够绑定 Input 和 Output 的接口。我们先来看 MethodInterceptor 中用于拦截的 invoke 方法，如下所示：
>
> 复制代码
>
> ```
> @Override
> public synchronized Object invoke(MethodInvocation invocation) throws Throwable {
>         Method method = invocation.getMethod();
>  
>         Object boundTarget = targetCache.get(method);
>         if (boundTarget != null) {
>             return boundTarget;
>         }
>  
>         Input input = AnnotationUtils.findAnnotation(method, Input.class);
>         if (input != null) {
>             String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(input, method);
>             boundTarget = this.inputHolders.get(name).getBoundTarget();
>             targetCache.put(method, boundTarget);
>             return boundTarget;
>         }
>         else {
>             Output output = AnnotationUtils.findAnnotation(method, Output.class);
>             if (output != null) {
>                 String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(output, method);
>                 boundTarget = this.outputHolders.get(name).getBoundTarget();
>                 targetCache.put(method, boundTarget);
>                 return boundTarget;
>             }
>         }
>         return null;
> }
> ```
>
> 这里的逻辑比较简单，可以看到 BindableProxyFactory 保存了一个缓存对象 targetCache。如果所调用方法已经存在于缓存中，则直接返回目标对象。反之，会根据 @Input 和 @Output 注解从 inputHolders 和 outputHolders 中获取对应的目标对象并放入缓存中。这里使用缓存的作用仅仅是为了加快每次方法调用的速度，而系统在初始化时通过重写 afterPropertiesSet 方法，已经将所有的目标对象都放置在 inputHolders 和 outputHolders 这两个集合中。至于这里提到的这个目标对象，暂时可以把它理解为就是一种 MessageChannel 对象，后面会对其进行展开。
>
> 然后我们来看 Bindable 接口的定义，如下所示：
>
> 复制代码
>
> ```
> public interface Bindable {
>   default Collection<Binding<Object>> createAndBindInputs(BindingService adapter) {
>       return Collections.<Binding<Object>>emptyList();
>   }
>   default void bindOutputs(BindingService adapter) {}
>   default void unbindInputs(BindingService adapter) {}
>   default void unbindOutputs(BindingService adapter) {}
>   default Set<String> getInputs() {
>       return Collections.emptySet();
>   }
> 
>   default Set<String> getOutputs() {
>       return Collections.emptySet();
>   }
> }
> ```
>
> 显然，这个接口提供了对 Input 和 Output 的绑定和解绑操作。在 BindableProxyFactory 中，对以上几个方法的实现过程基本都类似，我们随机挑选一个 bindOutputs 方法进行展开，如下所示：
>
> 复制代码
>
> ```
> @Override
> public void bindOutputs(BindingService bindingService) {
>         for (Map.Entry<String, BoundTargetHolder> boundTargetHolderEntry : this.outputHolders.entrySet()) {
>             BoundTargetHolder boundTargetHolder = boundTargetHolderEntry.getValue();
>             String outputTargetName = boundTargetHolderEntry.getKey();
>             if (boundTargetHolderEntry.getValue().isBindable()) {
>                 if (log.isDebugEnabled()) {
>                     log.debug(String.format("Binding %s:%s:%s", this.namespace, this.type, outputTargetName));
>                 }
>                 bindingService.bindProducer(boundTargetHolder.getBoundTarget(), outputTargetName);
>             }
>         }
> }
> ```
>
> 这里需要引入另一个重要的工具类 BindingService，该类提供了对 Input 和 Output 目标对象进行绑定的能力。但事实上，通过类上的注释可以看到，这也是一个外观类，它将底层的绑定动作委托给了 Binder。我们以绑定生产者的 bindProducer 方法为例展开讨论，该方法如下所示：
>
> 复制代码
>
> ```
> public <T> Binding<T> bindProducer(T output, String outputName) {
>         String bindingTarget = this.bindingServiceProperties
>                 .getBindingDestination(outputName);
>         Binder<T, ?, ProducerProperties> binder = (Binder<T, ?, ProducerProperties>) getBinder(
>                 outputName, output.getClass());
>         ProducerProperties producerProperties = this.bindingServiceProperties
>                 .getProducerProperties(outputName);
>         if (binder instanceof ExtendedPropertiesBinder) {
>             Object extension = ((ExtendedPropertiesBinder) binder)
>                     .getExtendedProducerProperties(outputName);
>             ExtendedProducerProperties extendedProducerProperties = new ExtendedProducerProperties<>(
>                     extension);
>             BeanUtils.copyProperties(producerProperties, extendedProducerProperties);
>             producerProperties = extendedProducerProperties;
>         }
>         validate(producerProperties);
>         Binding<T> binding = doBindProducer(output, bindingTarget, binder, producerProperties);
>         this.producerBindings.put(outputName, binding);
>         return binding;
> }
> ```
>
> 显然，这里的 doBindProducer 方法完成了真正的绑定操作，如下所示：
>
> 复制代码
>
> ```
> public <T> Binding<T> doBindProducer(T output, String bindingTarget, Binder<T, ?, ProducerProperties> binder,
>             ProducerProperties producerProperties) {
>         if (this.taskScheduler == null || this.bindingServiceProperties.getBindingRetryInterval() <= 0) {
>             return binder.bindProducer(bindingTarget, output, producerProperties);
>         }
>         else {
>             try {
>                 return binder.bindProducer(bindingTarget, output, producerProperties);
>             }
>             catch (RuntimeException e) {
>                 LateBinding<T> late = new LateBinding<T>();
>                 rescheduleProducerBinding(output, bindingTarget, binder, producerProperties, late, e);
>                 return late;
>             }
>         }
> }
> ```
>
> 从这个方法中，我们终于看到了 Spring Cloud Stream 中最核心的概念 Binder，通过 Binder 的 bindProducer 方法完成了目标对象的绑定。
>
> #### Binder
>
> Binder 是一个接口，分别提供了绑定生产者和消费者的方法，如下所示：
>
> 复制代码
>
> ```
> public interface Binder<T, C extends ConsumerProperties, P extends ProducerProperties> {
>     Binding<T> bindConsumer(String name, String group, T inboundBindTarget, C consumerProperties);
>  
>     Binding<T> bindProducer(String name, T outboundBindTarget, P producerProperties);
> }
> ```
>
> 在介绍 Binder 接口的具体实现类之前，我们先来看一下如何获取一个 Binder，getBinder 方法如下所示。
>
> 复制代码
>
> ```
> protected <T> Binder<T, ?, ?> getBinder(String channelName, Class<T> bindableType) {
>         String binderConfigurationName = this.bindingServiceProperties.getBinder(channelName);
>         return binderFactory.getBinder(binderConfigurationName, bindableType);
> ```
>
> 显然，这里用到了个工厂模式。工厂类 BinderFactory 的定义如下所示：
>
> 复制代码
>
> ```
> public interface BinderFactory {
>  
>     <T> Binder<T, ? extends ConsumerProperties, ? extends ProducerProperties> getBinder(String configurationName,
>             Class<? extends T> bindableType);
> }
> ```
>
> BinderFactory 只有一个方法，根据给定的配置名称 configurationName 和绑定类型 bindableType 获取 Binder 实例。而 BinderFactory 的实现类也只有一个，即 DefaultBinderFactory。在该实现类的 getBinder 方法中对配置信息进行了校验，并通过 getBinderInstance 获取真正的 Binder 实例。在 getBinderInstance 方法中，我们通过一系列基于 Spring 容器的步骤构建了一个上下文对象 ConfigurableApplicationContext，并通过该上下文对象获取实现了 Binder 接口的 Java bean，核心代码就是下面这句：
>
> 复制代码
>
> ```
> Binder<T, ?, ?> binder = binderProducingContext.getBean(Binder.class);
> ```
>
> 当然，对于 BinderFactory 而言，缓存也是需要的。在 DefaultBinderFactory 中存在一个 binderInstanceCache 变量，使用了一个 Map 来保存配置名称所对应的 Binder 对象。
>
> #### AbstractMessageChannelBinder
>
> 既然我们已经能够获取 Binder 实例，接下去就来讨论 Binder 实例中对 bindConsumer 和 bindProducer 方法的实现过程。在 Spring Cloud Stream 中，Binder 接口的类层关系如下所示，注意到这里还展示了 spring-cloud-stream-binder-rabbit 代码工程中的 RabbitMessageChannelBinder 类，这个类在本课时讲到 Spring Cloud Stream 与 RabbitMQ 进行集成时会具体展开：
>
> ![图片1.png](https://s0.lgstatic.com/i/image/M00/81/B6/Ciqc1F_RmKqAehbuAAUIZHxCJ9g730.png)
>
> Binder 接口类层结构图
>
> Spring Cloud Stream 首先提供了一个 AbstractBinder，这是一个抽象类，提供的 bindConsumer 和 bindProducer 方法实现如下所示：
>
> 复制代码
>
> ```
> @Override
> public final Binding<T> bindConsumer(String name, String group, T target, C properties) {
>         if (StringUtils.isEmpty(group)) {
>             Assert.isTrue(!properties.isPartitioned(), "A consumer group is required for a partitioned subscription");
>         }
>         return doBindConsumer(name, group, target, properties);
> }
>  
> protected abstract Binding<T> doBindConsumer(String name, String group, T inputTarget, C properties);
>  
> @Override
> public final Binding<T> bindProducer(String name, T outboundBindTarget, P properties) {
>         return doBindProducer(name, outboundBindTarget, properties);
> }
>  
> protected abstract Binding<T> doBindProducer(String name, T outboundBindTarget, P properties);
> ```
>
> 可以看到，它对 Binder 接口中相关方法只是提供了空实现，并把具体实现过程通过 doBindConsumer 和 doBindProducer 抽象方法交由子类进行完成。显然，从设计模式上讲，AbstractBinder 应用了很典型的模板方法模式。
>
> AbstractBinder 的子类是 AbstractMessageChannelBinder，它同样也是一个抽象类。我们来看它的 doBindProducer 方法，并对该方法中的核心语句进行提取和整理：
>
> 复制代码
>
> ```
> @Override
> public final Binding<MessageChannel> doBindProducer(final String destination, MessageChannel outputChannel,
>         final P producerProperties) throws BinderException {
> 	 
> 	…
> final MessageHandler producerMessageHandler;
>         final ProducerDestination producerDestination;
>         try {
>             producerDestination = this.provisioningProvider.provisionProducerDestination(destination,
>                     producerProperties);
>             SubscribableChannel errorChannel = producerProperties.isErrorChannelEnabled()
>                     ? registerErrorInfrastructure(producerDestination) : null;
>             producerMessageHandler = createProducerMessageHandler(producerDestination, producerProperties,
>                     errorChannel);
> 
> 	…
> postProcessOutputChannel(outputChannel, producerProperties);
>         ((SubscribableChannel) outputChannel).subscribe(
>                 new SendingHandler(producerMessageHandler, HeaderMode.embeddedHeaders
>                         .equals(producerProperties.getHeaderMode()), this.headersToEmbed,
> 	                     producerProperties.isUseNativeEncoding()));
>  
> Binding<MessageChannel> binding = new DefaultBinding<MessageChannel>(destination, null, outputChannel,
>                 producerMessageHandler instanceof Lifecycle ? (Lifecycle) producerMessageHandler : null) {
> 	…
> };
>  
>         doPublishEvent(new BindingCreatedEvent(binding));
>         return binding;
> }
> ```
>
> 上述代码的核心逻辑在于，Source 里的 output 发送消息到 outputChannel 通道之后会被 SendingHandler 这个 MessageHandler 进行处理。从设计模式上讲，SendingHandler 是一个静态代理类，因此它又将这个处理过程委托给了由 createProducerMessageHandler 方法所创建的 producerMessageHandler，这点从 SendingHandler 的定义中可以得到验证，如下所示的 delegate 就是传入的 producerMessageHandler：
>
> 复制代码
>
> ```
> private final class SendingHandler extends AbstractMessageHandler implements Lifecycle {
>  
> private final MessageHandler delegate;
>  
>         @Override
>         protected void handleMessageInternal(Message<?> message) throws Exception {
>             Message<?> messageToSend = (this.useNativeEncoding) ? message
>                     : serializeAndEmbedHeadersIfApplicable(message);
>             this.delegate.handleMessage(messageToSend);
>         }
>  
>         // 省略其他方法
> }
> ```
>
> 请注意，同样作为一个模板方法类，AbstractMessageChannelBinder 具有三个抽象方法，即 createProducerMessageHandler、postProcessOutputChannel 和 afterUnbindProducer，这三个方法都需要由它的子类进行实现。也就是说，SendingHandler 所使用的 producerMessageHandler 需要由 AbstractMessageChannelBinder 子类负责进行创建。
>
> 需要注意的是，作为统一的数据模型，SendingHandler 以及 producerMessageHandler 中使用的都是 Spring Messaging 组件中的 Message 消息对象，而 createProducerMessageHandler 内部会把这个 Message 消息对象转换成对应中间件的消息数据格式并进行发送。
>
> 下面转到消息消费的场景，我们来看 AbstractMessageChannelBinder 的 doBindConsumer 方法。该方法的核心语句是创建一个消费者端点 ConsumerEndpoint，如下所示：
>
> 复制代码
>
> ```
> MessageProducer consumerEndpoint = createConsumerEndpoint(destination, group, properties);
> consumerEndpoint.setOutputChannel(inputChannel);
> ```
>
> 这两行代码有两个注意点。首先，createConsumerEndpoint 是一个抽象方法，需要 AbstractMessageChannelBinder 的子类进行实现。与 createProducerMessageHandler 一样，createConsumerEndpoint 需要把中间件对应的消息数据结构转换成 Spring Messaging 中统一的 Message 消息对象。
>
> 然后，我们注意到这里的 consumerEndpoint 类型是 MessageProducer。MessageProducer 在 Spring Integration 中代表的是消息的生产者，它会把从第三方消息中间件中收到的消息转发到 inputChannel 所指定的通道中。基于 @StreamListener 注解，在 Spring Cloud Stream 中存在一个 StreamListenerMessageHandler 类，用于订阅 inputChannel 消息通道中传入的消息并进行消费。
>
> 作为总结，我们可以用如下所示的流程图来概括整个消息发送和消费流程：
>
> ![图片2.png](https://s0.lgstatic.com/i/image/M00/81/C1/CgqCHl_RmIOAbYcHAAHhiR5WQIE310.png)
>
> 消息发送和消费整体流程图
>
> ### Spring Cloud Stream 集成 RabbitMQ
>
> 到目前为止，Spring Cloud Stream 提供了对 RabbitMQ 和 Kafka 这两款主流消息中间件的集成。今天，我们选择使用 RabbitMQ 作为示例，讲解如何通过 Spring Cloud Stream 所提供的 Binder 完成与具体消息中间件的整合。
>
> Spring Cloud Stream 团队提供了 spring-cloud-stream-binder-rabbit 作为与 RabbitMQ 集成的代码工程。这个工程只有四个类，我们需要重点关注的就是实现了 AbstractMessageChannelBinder 中几个抽象方法的 RabbitMessageChannelBinder 类。
>
> #### 集成消息发送
>
> 首先找到 RabbitMessageChannelBinder中 的 createProducerMessageHandler 方法，我们知道该方法用于完成消息的发送。我们在 createProducerMessageHandler 中找到了以下核心代码：
>
> 复制代码
>
> ```
> final AmqpOutboundEndpoint endpoint = new AmqpOutboundEndpoint(         buildRabbitTemplate(producerProperties.getExtension(), errorChannel != null));
> endpoint.setExchangeName(producerDestination.getName());
> ```
>
> 首先，在 buildRabbitTemplate 方法中，我们看到了 RabbitTemplate 的构建过程。RabbitTemplate 是 Spring Amqp 组件中提供的专门用于封装与 RabbitMQ 底层交互 API 的模板类。在构建 RabbitTemplate 的整个过程中，涉及设置与 RabbitMQ 相关的 ConnectionFactory 等众多参数。
>
> 然后，我们发现 RabbitMessageChannelBinder 也是直接集成了 Spring Integration 中用于整合 AQMP 协议的 AmqpOutboundEndpoint。AmqpOutboundEndpoint 提供了如下所示的 send 方法进行消息的发送：
>
> 复制代码
>
> ```
> private void send(String exchangeName, String routingKey,
>             final Message<?> requestMessage, CorrelationData correlationData) {
>         if (this.amqpTemplate instanceof RabbitTemplate) {
>             MessageConverter converter = ((RabbitTemplate) this.amqpTemplate).getMessageConverter();
>             org.springframework.amqp.core.Message amqpMessage = MappingUtils.mapMessage(requestMessage, converter,
>                     getHeaderMapper(), getDefaultDeliveryMode(), isHeadersMappedLast());
>             addDelayProperty(requestMessage, amqpMessage);
>             ((RabbitTemplate) this.amqpTemplate).send(exchangeName, routingKey, amqpMessage, correlationData);
>         }
>         else {
>             this.amqpTemplate.convertAndSend(exchangeName, routingKey, requestMessage.getPayload(),
>                     message -> {
>                         getHeaderMapper().fromHeadersToRequest(requestMessage.getHeaders(),
>                                 message.getMessageProperties());
>                         return message;
>                     });
>         }
> }
> ```
>
> 可以看到这里依赖于 Spring Amqp 提供的 AmqpTemplate 接口进行消息的发送，而 RabbitTemplate 是 AmqpTemplate 的一个实现类。同时，通过 Spring Integration 组件中提供的 MessageConverter 工具类完成了从 org.springframework.messaging.Message 到 org.springframework.amqp.core.Message 这两个消息数据结构之间的转换。
>
> #### 集成消息消费
>
> RabbitMessageChannelBinder 中与消息消费相关的是 createConsumerEndpoint 方法。这个方法中大量使用了 Spring Amqp 和 Spring Integration 中的工具类。该方法最终返回的是一个 AmqpInboundChannelAdapter 对象。在 Spring Integration 中，AmqpInboundChannelAdapter 是一种 InboundChannelAdapter，代表面向输入的通道适配器，提供了消息监听功能，如下所示
>
> 复制代码
>
> ```
> protected class Listener implements ChannelAwareMessageListener, RetryListener {
>         @Override
>         public void onMessage(final Message message, final Channel channel) throws Exception {
>  
>         //省略相关实现
> 	 }
> }
> ```
>
> 在这个 onMessage 方法中，调用了 createAndSend 方法完成消息的创建和发送，如下所示：
>
> 复制代码
>
> ```
> private void createAndSend(Message message, Channel channel) {
>             org.springframework.messaging.Message<Object> messagingMessage = createMessage(message, channel);
>             setAttributesIfNecessary(message, messagingMessage);
>             sendMessage(messagingMessage);
> }
> 	 
> 	 
> private org.springframework.messaging.Message<Object> createMessage(Message message, Channel channel) {
>             Object payload = AmqpInboundChannelAdapter.this.messageConverter.fromMessage(message);
>             Map<String, Object> headers = AmqpInboundChannelAdapter.this.headerMapper
>                     .toHeadersFromRequest(message.getMessageProperties());
>             if (AmqpInboundChannelAdapter.this.messageListenerContainer.getAcknowledgeMode()
>                     == AcknowledgeMode.MANUAL) {
>                 headers.put(AmqpHeaders.DELIVERY_TAG, message.getMessageProperties().getDeliveryTag());
>                 headers.put(AmqpHeaders.CHANNEL, channel);
>             }
>             if (AmqpInboundChannelAdapter.this.retryTemplate != null) {
>                 headers.put(IntegrationMessageHeaderAccessor.DELIVERY_ATTEMPT, new AtomicInteger());
>             }
>             final org.springframework.messaging.Message<Object> messagingMessage = getMessageBuilderFactory()
>                     .withPayload(payload)
>                     .copyHeaders(headers)
>                     .build();
>             return messagingMessage;
> }
> ```
>
> 显然，在上述 createMessage 方法中，我们完成了消息数据格式从 org.springframework.amqp.core.Message 到 org.springframework.messaging.Message 的反向转换。
>
> ### 小结与预告
>
> Binder 是 Spring Cloud Stream 的核心组件，通过这个组件，Spring Cloud Stream 完成了与第三方消息中间件的集成。在本课时中，我们花了较大篇幅系统分析了 Binder 组件相关的核心类。然后，基于这些核心类以及 RabbitMQ，我们给出了 Spring Cloud Stream 集成RabbitMQ 的实现原理。
>
> 这里给你留一道思考题：在 Spring Cloud Stream 中，Binder 组件对于消息发送和消费做了哪些抽象？
>
> 介绍完 Spring Cloud Stream 之后，我们又将启动一个新的话题，即安全性。在微服务架构中，安全性的重要性往往被忽略，值得我们系统的进行分析和实现。下一课时，我们首先关注如何理解微服务访问的安全需求和实现方案。