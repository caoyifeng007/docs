[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4776)



> 在了解了 Spring Cloud Sleuth 的基本工作原理以及与 Zipkin 之间的集成方案之后，我们不禁要想，虽然内置的日志埋点和采集功能已经能够满足日常开发的大多数场景需要，但如果我想在业务系统中重点监控某些业务操作时，是不是有办法来创建自定义的 Span 并纳入可视化监控机制中呢？答案是肯定的，今天的内容我们就围绕如何使用 Spring Cloud Sleuth 底层的 Brave 框架在服务访问链路中添加自定义Span这一话题展开讨论。
>
> ### 使用 Brave 创建自定义 Span
>
> 从 2.X 版本开始，Spring Cloud Sleuth 全面使用 Brave 作为其底层的服务跟踪实现框架。原本在 1.X 版本中通过 Spring Cloud Sleuth 自带的 org.springframework.cloud.sleuth.Tracer 接口创建和管理自定义 Span 的方法将不再有效。因此，想要在访问链路中创建自定义的 Span，需要对 Brave 框架所提供的功能有足够的了解。
>
> 事实上，Brave 是 Java 版的 Zipkin 客户端，它将收集的跟踪信息，以 Span 的形式上报给 Zipkin 系统。我们首先来关注 Brave 中的 Span 类，该类的方法列表如下所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/04/49/CgpVE1_sSpqAX8fHAAAsCx2fAiU688.png)
>
> Span 类的方法列表
>
> 注意到 Span 是一个抽象类，在上面的方法列表中，我们也看到该类的几乎所有方法都是抽象方法，需要子类进行实现。在 Brave 中，该抽象类的子类就是 RealSpan。RealSpan 中的 start 方法如下所示：
>
> 复制代码
>
> ```
> @Override 
> public Span start(long timestamp) {
>     synchronized (state) {
>       state.startTimestamp(timestamp);
>     }
>     return this;
> }
> ```
>
> 这里的 state 是一个可变的 MutableSpan，而上述 start 方法就是为这个 MutableSpan 设置了开始时间。可以想象，对应的 finish 方法也会为 MutableSpan 设置结束时间，如下所示：
>
> 复制代码
>
> ```
> @Override 
> public void finish(long timestamp) {
>     if (!pendingSpans.remove(context)) return;
>     synchronized (state) {
>       state.finishTimestamp(timestamp);
>     }
>     finishedSpanHandler.handle(context, state);
> }
> ```
>
> 对于关闭 Span 的操作而言，上述方法还添加了一个 Handler 以便执行回调逻辑，这也是非常常见的一种实现技巧。我们接着来看另一个非常有用的 annotate 方法，如下所示：
>
> 复制代码
>
> ```
> @Override 
> public Span annotate(long timestamp, String value) {
>     if ("cs".equals(value)) {
>       synchronized (state) {
>         state.kind(Span.Kind.CLIENT);
>         state.startTimestamp(timestamp);
>       }
>     } else if ("sr".equals(value)) {
>       synchronized (state) {
>         state.kind(Span.Kind.SERVER);
>         state.startTimestamp(timestamp);
>       }
>     } else if ("cr".equals(value)) {
>       synchronized (state) {
>         state.kind(Span.Kind.CLIENT);
>       }
>       finish(timestamp);
>     } else if ("ss".equals(value)) {
>       synchronized (state) {
>         state.kind(Span.Kind.SERVER);
>       }
>       finish(timestamp);
>     } else {
>       synchronized (state) {
>         state.annotate(timestamp, value);
>       }
>     }
>     return this;
> }
> ```
>
> 回顾《监控原理：如何理解服务监控和 Spring Cloud Sleuth 的基本原理？》中的介绍的四种监控事件，我们不难理解上述代码的作用就是为这些事件指定类型以及时间，从而为构建监控链路提供基础。
>
> RealSpan 中最后一个值得介绍的方法是如下所示的 tag 方法：
>
> 复制代码
>
> ```
> @Override 
> public Span tag(String key, String value) {
>     synchronized (state) {
>       state.tag(key, value);
>     }
>     return this;
> }
> ```
>
> 该方法为 Span 打上一个标签，其中两个参数分别代表标签的 Key 和 Value，开发人员可以根据需要对任何一个 Span 添加自定义的标签体系。
>
> 了解了 Span 的定义之后，我们就来讨论在业务代码中创建 Span 的两种方法。一种是使用 Brave 中的 Tracer 类，一种是使用注解。
>
> #### 通过 Tracer 类创建 Span
>
> Tracer 是一个工具类，提供了一批方法用于完成与 Span 相关的各种属性和操作。我们同样挑选几个常见的方法进行展开。
>
> 首先，我们来看如何通过 Tracer 创建一个新的根 Span，可以通过如下所示的 newTrace 方法进行实现：
>
> 复制代码
>
> ```
> public Span newTrace() {
>     return _toSpan(newRootContext());
> }
> ```
>
> 这里用到了一个保存跟踪信息的 TraceContext 上下文对象，对于根 Span 而言，这个 TraceContext 就是全新的上下文，没有父 Span。而这里的 _toSpan 方法则最终构建了一个前面提到的 RealSpan 对象。
>
> 复制代码
>
> ```
> Span _toSpan(TraceContext decorated) {
> 	if (isNoop(decorated)) return new NoopSpan(decorated);
>     PendingSpan pendingSpan = pendingSpans.getOrCreate(decorated, false);
>     return new RealSpan(decorated, pendingSpans, pendingSpan.state(), pendingSpan.clock(), finishedSpanHandler);
> }
> ```
>
> 这里多了一个新建的对象叫 PendingSpan ，用于收集一条 Trace 上暂时被挂起的未完成的 Span。
>
> 一旦创建了根 Span，我们就可以在这个 Span 上执行 nextSpan 方法来添加新的 Span，如下所示：
>
> 复制代码
>
> ```
> public Span nextSpan() {
>     TraceContext parent = currentTraceContext.get();
>     return parent != null ? newChild(parent) : newTrace();
> }
> ```
>
> 这里获取当前 TraceContext，如果该上下文不存在，就通过 newTrace 方法来创建一个新的根 Span；如果存在，则基于这个上下文并调用 newChild 方法来创建一个子 Span。newChild 方法也比较简单，如下所示：
>
> 复制代码
>
> ```
> public Span newChild(TraceContext parent) {
>     if (parent == null) throw new NullPointerException("parent == null");
>     return _toSpan(nextContext(parent));
> }
> ```
>
> 当然，在很多场景下，我们首先需要获取当前的 Span，这时候就可以使用 Tracer 类所提供的 currentSpan 方法，如下所示：
>
> 复制代码
>
> ```
> public Span currentSpan() {
>     TraceContext currentContext = currentTraceContext.get();
>     return currentContext != null ? toSpan(currentContext) : null;
> }
> ```
>
> 基于 Tracer 提供的这些常见方法，我们可以梳理在业务代码中添加一个自定义的 Span 模版方法，如下所示：
>
> 复制代码
>
> ```
> @Service
> public class MyService {
> 
>     @Autowired
>     private Tracer tracer;
> 
>     public void perform() {
>          Span newSpan = tracer.nextSpan().name("spanName").start();
>         //ScopedSpan newSpan = tracer.startScopedSpan("spanName");
>         try {
>             //执行业务逻辑
>         }
>         finally{
>           newSpan.tag("key", "value");
>           newSpan.annotate("myannotation");
>           newSpan.finish();
>         }
>     }
> }
> ```
>
> 在上述代码中，我们注入了一个 Tracer 对象，然后通过 nextSpan().name("findByDeviceCode").start() 方法创建并启动了一个“spanName”新的 Span。这是在业务代码中嵌入自定义 Span 的一种方法。另一种方法是使用注释行代码中的 ScopedSpan，ScopedSpan 代表包含一定操作延迟的 Span 对象，可以在操作不脱离当前进程时可以使用。当我们执行完各种业务逻辑之后，可以分别通过 tag 方法和 annotate 添加标签和定义事件，最后通过 finish 方法关闭 Span。这段模版代码可以直接引入到日常的开发过程中。
>
> #### 使用注解创建 Span
>
> 在 Brave 中，除了使用代码对创建 Span 的过程进行控制之外，我们还可以使用另一种更为简单的方法来创建 Span，这种方法就是使用注解。
>
> 我们先来看 @NewSpan 注解，这个注解可以自动创建一个新的 Span，使用方法如下所示：
>
> 复制代码
>
> ```
> @NewSpan
> void myMethod();
> ```
>
> 当然，我们也可以把 @NewSpan 注解和 @SpanTag 注解结合在一起使用，@SpanTag 注解用于自动为通过 @NewSpan 注解所创建的 Span 添加标签，如下所示：
>
> 复制代码
>
> ```
> @NewSpan(name = "myspan")
> void myMethod(@SpanTag("mykey") String param);
> ```
>
> 上述代码示例中，我们定义了一个名为“myspan”的新 Span，并在 myMethod 方法中注入了一个标签并定了标签的键，而该标签的值就是方法的输入参数 param。如果我们执行这个 myMethod(@SpanTag("mykey") String param) 方法，那么将生成一个键为“mykey”，值为 param 的新标签。
>
> 现在，我们已经掌握了创建自定义 Span 的常见方法，让我们把这些方法都串联起来实现日常开发中常见的自定义 Span 的应用场景，并集成 Zipkin 来实现自定义的可视化跟踪效果。
>
> ### 使用 Zipkin 集成自定义跟踪
>
> 在上一课时的介绍中，我们都是基于几个微服务之间的调用关系来讨论 Zipkin 在服务监控可视化过程中发挥的作用，其中完整服务调用链路中的各个 Span 都是采用默认的服务调用结果。在大多数情况下，我们通过这些 Span 就可以分析和排查服务调用链路中可能存在的问题。但在某些特定场景下，我们希望在这些 Span 的基础上能够实现一些定制化的数据收集和展示方式。
>
> 我们来考虑如下场景，假设在服务调用链路中，某一个方法调用时间比较长，但通过默认所创建的基于该方法的 Span，通常无法判断响应时间过长的原因。那么就可能出现一个需求，即通过添加一系列的自定义 Span 的方式进一步对长时间的服务调用进行拆分，把该请求中所涉及的多种操作分别创建 Span，然后找到最影响性能的 Span 并进行优化，这也是服务监控系统实现过程中的一项最佳实践，如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/8C/65/Ciqc1F_sSrKAUJKgAAAtpNjayF4547.png)
>
> 通过自定义 Span 找到性能瓶颈点示意图
>
> ### 使用 Tracer 添加自定义 Span
>
> 让我们回到 SpringHealth 案例系统，来到 device-service。我们知道可以通过 Brave 的 Tracer 工具类创建 Span 并把该 Span 相关信息推送给 Zipkin。现在，我们希望在 DeviceService 的调用过程中添加一个新的 Span 以帮助 device-service 诊断响应时间过长问题，示例代码如下所示：
>
> 复制代码
>
> ```
> @Service
> public class DeviceService {
>  
>     @Autowired
>     private DeviceRepository deviceRepository;
> 
>     @Autowired
>      private Tracer tracer;
> 
>     public Device getDeviceByCode(String deviceCode) {
> 
>          Span newSpan = tracer.nextSpan().name("findByDeviceCode").start();
>  
>         try {
>             return deviceRepository.findByDeviceCode(deviceCode);
>         }
>         finally{
>           newSpan.tag("device", "dababase");
>           newSpan.annotate("deviceInfoObtained");
>           newSpan.finish();
>         }
>     }
> }
> ```
>
> 在上述示例中，我们看到通过以下几个简单的方法调用就可以实现一个自定义 Span。这里基于前面介绍的自定义 Span 模版方法完成了 Span 的创建过程。我们使用 newTrace 方法创建一个自定义的 Span，并为该 Span 命名为“findByDeviceCode”。然后创建了一个键为“device”标签，并把标签值设置为"dababase "指明该标签与数据库操作相关。然后，我们又通过 annotate 方法记录了一个代表设备信息已经被获取的“deviceInfoObtained”事件。最后，我们执行了 finish 方法，在具体操作结束之后必须调用此方法，否则 Span 数据不会被发送到 Zipkin 中。
>
> ### 可视化自定义 Span
>
> 我们先来回顾在不添加上述自定义 Span 之前调用 http://localhost:5555/springhealth/device/devices/device1 时 Zipkin 上所生成的效果图，如下所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/8C/71/CgqCHl_sSr2AH503AABAn4kpV9A498.png)
>
> Zipkin 中系统自动生成 Span 效果界面
>
> 显然，这三个 Span 都是系统自定生成的。现在我们重新启动 device-service，然后再次访问该端口，就会得到如下图所示的可视化效果：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/8C/65/Ciqc1F_sSsiATG_0AABPUjTB7og302.png)
>
> Zipkin中添加自定义Span效果界面
>
> 请注意在上图中，我们看到在原有默认可视化效果的基础上又多了一个名为“findByDeviceCode”的自定义 Span。点击该 Span，我们也将得到这个 Span 对应的各项事件明细数据，如下图所示：
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/8C/65/Ciqc1F_sSs-ACuYCAABBZOB6BPU918.png)
>
> Zipkin 中自定义 Span 中每个关键事件明细数据界面
>
> 这里看到了“deviceObtained”这个自定义事件。同时，基于数据，我们也不难发现在 device-service 处理请求的时间中实际上大部分是消耗在访问数据库以获取设备数据的过程中。同样，我们也可以在其他服务中添加不同的 Span 以实现对服务调用过程更加精细化的管理。
>
> ### 小结与预告
>
> 自定义 Span 是我们在日常开发过程中进程使用的一项工程实践，通过在业务系统中嵌入各种 Span 能够帮助开发人员找到系统中的性能瓶颈点从而为系统重构和优化提供抓手。在 Spring Cloud Sleuth 中，Brave 框架可以用来创建自定义的 Span，而上一课时中介绍的 Zipkin 框架则也可以对这些自定义 Span 实现可视化。本课时对这些具体的开发工作做了详细的介绍并结合 SpringHealth 案例给出了示例代码。
>
> 这里给你留一道思考题：通过 Brave 框架，开发人员创建自定义 Span 有哪些具体的实现方法？
>
> 在介绍完服务监控之后，接下来是整个课程的最后一个主题，即微服务测试。我们将先从微服务系统中与测试相关的需求和解决方案讲起并引出 Spring 家族中的 Spring Cloud Contract 框架。