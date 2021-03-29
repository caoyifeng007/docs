[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4753)



> 上一课时我们介绍了 Spring Cloud Netflix Ribbon 的使用方法。可以看到，基于 Spring Cloud 对 Netflix Ribbon 的完美整合，在微服务系统中实现客户端负载均衡的开发过程对开发人员而言可以说是透明的，我们只需要在 RestTemplate 上添加一个 @LoadBalanced 注解即可。那么，你可能会觉得好奇，Spring Cloud 是如何做到这一点的呢？这就需要我们对 Netflix Ribbon 以及 Spring Cloud Netflix Ribbon 的基本架构以及实现原理进行深入分析。今天我们就来讨论这个内容。
>
> ### Netflix Ribbon 基本架构
>
> 在深入讨论 Netflix Ribbon 之前，我们可以做一个简单的抽象。作为一款客户端负载均衡工具，要做的事情无非就是两件：第一件事情是获取注册中心中的服务器列表；第二件事情是在这个服务列表中选择一个服务进行调用。针对这两个问题，Netflix Ribbon 提供了自身的一套基本架构，并抽象了一批**核心类**，让我们来一起看一下核心类。
>
> #### Netflix Ribbon 中的核心类
>
> Netflix Ribbon 的核心接口 ILoadBalancer 就是围绕着上述两个问题来设计的，该接口位于 com.netflix.loadbalancer 包下，定义如下：
>
> 复制代码
>
> ```
> public interface ILoadBalancer {
>   //添加后端服务
>   public void addServers(List<Server> newServers);
>  
>   //选择一个后端服务
>   public Server chooseServer(Object key); 
>  
> 	//标记一个服务不可用
>   public void markServerDown(Server server);
>  
> 	//获取当前可用的服务列表
>   public List<Server> getReachableServers();
> 	 
> 	//获取所有后端服务列表
>    public List<Server> getAllServers();
> }
> ```
>
> ILoadBalancer 接口的类层结构如下所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/5D/D5/CgqCHl-FUWqAfQXPAAC8k1bRPQ8469.png)
>
> ILoadBalancer 接口的类层结构图
>
> 其中 AbstractLoadBalancer 是个抽象类，只定义了两个抽象方法，并不构成一种模板方法的结构。所以我们直接来看 ILoadBalancer 接口，该接口最基本的实现类是 BaseLoadBalancer，可以说负载均衡的核心功能都可以在这个类中得以实现。这个类代码非常多且杂，我们在理解上需要对其进行裁剪，从而抓住重点。
>
> 我们先来梳理 BaseLoadBalancer 包含的作为一个负载均衡器应该具备的一些核心组件，比较重要的有以下三个。
>
> - **IRule**
>
> IRule 接口是对负载均衡策略的一种抽象，可以通过实现这个接口来提供各种适用的负载均衡算法，我们在上一课时介绍 @RibbonClient 注解时已经看到过这个接口。该接口定义如下：
>
> 复制代码
>
> ```
> public interface IRule{
>     public Server choose(Object key);
>     public void setLoadBalancer(ILoadBalancer lb);
>     public ILoadBalancer getLoadBalancer();
> }
> ```
>
> 显然 choose 方法是该接口的核心方法，我们在下文中会基于该方法对各种负载均衡算法进行具体展开。
>
> - **IPing**
>
> IPing 接口判断目标服务是否存活，定义如下：
>
> 复制代码
>
> ```
> public interface IPing {
>     public boolean isAlive(Server server);
> }
> ```
>
> 可以看到 IPing 接口中只有一个 isAlive() 方法，通过对服务发出"Ping"操作来获取服务响应，从而判断该服务是否可用。
>
> - **LoadBalancerStats**
>
> LoadBalancerStats 类记录负载均衡的实时运行信息，用来作为负载均衡策略的运行时输入。
>
> 注意，在 BaseLoadBalancer 内部维护着 allServerList 和 upServerList 这两个线程的安全列表，所以对于 ILoadBalancer 接口定义的 addServers、getReachableServers、getAllServers 这几个方法而言，主要就是对这些列表的维护和管理工作。以 addServers 方法为例，它的实现如下所示：
>
> 复制代码
>
> ```
> public void addServers(List<Server> newServers) {
>         if (newServers != null && newServers.size() > 0) {
>             try {
>                 ArrayList<Server> newList = new ArrayList<Server>();
>                 newList.addAll(allServerList);
>                 newList.addAll(newServers);
>                 setServersList(newList);
>             } catch (Exception e) {
>                 logger.error("LoadBalancer [{}]: Exception while adding Servers", name, e);
>             }
>         }
> }
> ```
>
> 显然，这里的处理过程就是将原有的服务实例列表 allServerList 和新传入的服务实例列表 newServers 都合并到一个 newList 中，然后再调用 setServersList 方法用这个新的列表覆盖旧的列表。
>
> 针对负载均衡，我们重点应该关注的是 ILoadBalancer 接口中 chooseServer 方法的实现，不难想象该方法肯定通过前面介绍的 IRule 接口集成了具体负载均衡策略的实现。在 BaseLoadBalancer 中的 chooseServer 方法如下所示：
>
> 复制代码
>
> ```
> public Server chooseServer(Object key) {
>         if (counter == null) {
>             counter = createCounter();
>         }
>         counter.increment();
>         if (rule == null) {
>             return null;
>         } else {
>             try {
>                 return rule.choose(key);
>             } catch (Exception e) {
>                  return null;
>             }
>         }
> }
> ```
>
> 果然，这里使用了 IRule 接口的 choose 方法。接下来就让我们看看 Ribbon 中的 IRule 接口为我们提供了具体哪些负载均衡算法。
>
> #### Netflix Ribbon 中的负载均衡策略
>
> 一般而言，负载均衡算法可以分成两大类，即**静态负载均衡算法**和**动态负载均衡算法**。静态负载均衡算法比较容易理解和实现，典型的包括随机（Random）、轮询（Round Robin）和加权轮询（Weighted Round Robin）算法等。所有涉及权重的静态算法都可以转变为动态算法，因为权重可以在运行过程中动态更新。例如动态轮询算法中权重值基于对各个服务器的持续监控并不断更新。另外，基于服务器的实时性能分析分配连接是常见的动态策略。典型动态算法包括源 IP 哈希算法、最少连接数算法、服务调用时延算法等。
>
> 回到 Netflix Ribbon，IRule 接口的类层结构如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/5D/CA/Ciqc1F-FUXmALjkJAABO1WqxOVM788.png)
>
> IRule接口的类层结构图
>
> 可以看到 Netflix Ribbon 中的负载均衡实现策略非常丰富，既提供了 RandomRule、RoundRobinRule 等无状态的静态策略，又实现了 AvailabilityFilteringRule、WeightedResponseTimeRule 等多种基于服务器运行状况进行实时路由的动态策略。在上图中还看到了 RetryRule 这种重试策略，该策略会对选定的负载均衡策略执行重试机制。严格意义上讲重试是一种服务容错而不是负载均衡机制，但 Ribbon 也内置了这方面的功能。
>
> 静态的几种策略相对都比较简单，而像 RetryRule 实际上不算是严格意义上的负载均衡策略，所以这里重点关注 Ribbon 所实现的几种不同的动态策略。
>
> - **BestAvailableRule 策略**
>
> 选择一个并发请求量最小的服务器，逐个考察服务器然后选择其中活跃请求数最小的服务器。
>
> - **WeightedResponseTimeRule 策略**
>
> 该策略与请求的响应时间有关，显然，如果响应时间越长，就代表这个服务的响应能力越有限，那么分配给该服务的权重就应该越小。而响应时间的计算就依赖于前面介绍的 ILoadBalancer 接口中的 LoadBalancerStats。WeightedResponseTimeRule 会定时从 LoadBalancerStats 读取平均响应时间，为每个服务更新权重。权重的计算也比较简单，即每次请求的响应时间减去每个服务自己平均的响应时间就是该服务的权重。
>
> - **AvailabilityFilteringRule 策略**
>
> 通过检查 LoadBalancerStats 中记录的各个服务器的运行状态，过滤掉那些处于一直连接失败或处于高并发状态的后端服务器。
>
> ### Spring Cloud Netflix Ribbon
>
> 正如上一课时中提到的，对于 Netflix Ribbon 组件而言，我们首先需要明确它提供的只是一个辅助工具，这个辅助工具的目的是让你去集成它，而不是说它自己完成所有的工作。而 Spring Cloud 中的 Spring Cloud Netflix Ribbon 就是就专门针对 Netflix Ribbon 提供了一个独立的集成实现。
>
> Spring Cloud Netflix Ribbon 相当于 Netflix Ribbon 的客户端。而对于 Spring Cloud Netflix Ribbon 而言，我们的应用服务相当于它的客户端。Netflix Ribbon、Spring Cloud Netflix Ribbon、应用服务这三者之间的关系以及核心入口如下所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/5D/D5/CgqCHl-FUamAVGCPAABk9QlXx3Y885.png)
>
> 负载均衡三大组件之间的关系图
>
> 这次，我们打算从应用服务层的 @LoadBalanced 注解入手，切入 Spring Cloud Netflix Ribbon，然后再从 Spring Cloud Netflix Ribbon 串联到 Netflix Ribbon，从而形成整个负载均衡闭环管理。
>
> #### @LoadBalanced 注解
>
> 使用过 Spring Cloud Netflix Ribbon 的同学可能会问，为什么通过 @LoadBalanced 注解创建的 RestTemplate 就能自动具备客户端负载均衡的能力？这也是一个面试过程中经常被问到的问题。
>
> 事实上，在 Spring Cloud Netflix Ribbon 中存在一个自动配置类——LoadBalancerAutoConfiguration 类。而在该类中，维护着一个被 @LoadBalanced 修饰的 RestTemplate 对象的列表。在初始化的过程中，对于所有被 @LoadBalanced 注解修饰的 RestTemplate，调用 RestTemplateCustomizer 的 customize 方法进行定制化，该定制化的过程就是对目标 RestTemplate 增加拦截器 LoadBalancerInterceptor，如下所示：
>
> 复制代码
>
> ```
> @Configuration
>     @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
> static class LoadBalancerInterceptorConfig {
>         @Bean
>         public LoadBalancerInterceptor ribbonInterceptor(
>                 LoadBalancerClient loadBalancerClient,
>                 LoadBalancerRequestFactory requestFactory) {
>             return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
>         }
>  
>         @Bean
>         @ConditionalOnMissingBean
>         public RestTemplateCustomizer restTemplateCustomizer(
>                 final LoadBalancerInterceptor loadBalancerInterceptor) {
>             return restTemplate -> {
>                 List<ClientHttpRequestInterceptor> list = new ArrayList<>(
>                         restTemplate.getInterceptors());
>                 list.add(loadBalancerInterceptor);
>                 restTemplate.setInterceptors(list);
>             };
>         }
> }
> ```
>
> 这个 LoadBalancerInterceptor 用于实时拦截，可以看到它的构造函数中传入了一个对象 LoadBalancerClient，而在它的拦截方法本质上就是使用 LoadBalanceClient 来执行真正的负载均衡。LoadBalancerInterceptor 类代码如下所示：
>
> 复制代码
>
> ```
> public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
>  
>     private LoadBalancerClient loadBalancer;
>     private LoadBalancerRequestFactory requestFactory;
>  
>     public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
>         this.loadBalancer = loadBalancer;
>         this.requestFactory = requestFactory;
>     }
>  
>     public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
>         this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
>     }
>  
>     @Override
>     public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
>             final ClientHttpRequestExecution execution) throws IOException {
>         final URI originalUri = request.getURI();
>         String serviceName = originalUri.getHost();
>         Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
>         return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
>     }
> }
> ```
>
> 可以看到这里的拦截方法 intercept 直接调用了 LoadBalancerClient 的 execute 方法完成对请求的负载均衡执行。
>
> #### LoadBalanceClient 接口
>
> LoadBalancerClient 是一个非常重要的接口，定义如下：
>
> 复制代码
>
> ```
> public interface LoadBalancerClient extends ServiceInstanceChooser {
>  
>     <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
>  
>     <T> T execute(String serviceId, ServiceInstance serviceInstance,
>            LoadBalancerRequest<T> request) throws IOException;
>  
>     URI reconstructURI(ServiceInstance instance, URI original);
> }
> ```
>
> 这里有两个 execute 重载方法，用于根据负载均衡器所确定的服务实例来执行服务调用。而 reconstructURI 方法则用于构建服务 URI，使用负载均衡所选择的 ServiceInstance 信息重新构造访问 URI，也就是用服务实例的 host 和 port 再加上服务的端点路径来构造一个真正可供访问的服务。
>
> LoadBalancerClient 继承自 ServiceInstanceChooser 接口，该接口定义如下：
>
> 复制代码
>
> ```
> public interface ServiceInstanceChooser {
>  
>     ServiceInstance choose(String serviceId);
> }
> ```
>
> 从负载均衡角度讲，我们应该重点关注实际上是这个 choose 方法的实现，而提供具体实现的是实现了 LoadBalancerClient 接口的 RibbonLoadBalancerClient，而 RibbonLoadBalancerClient 位于 spring-cloud-netflix-ribbon 工程中。这样我们的代码流程就从应用程序转入到了 Spring Cloud Netflix Ribbon 中。
>
> 在 LoadBalancerClient 接口的实现类 RibbonLoadBalancerClient 中，choose 方法最终调用了如下所示的 getServer 方法：
>
> 复制代码
>
> ```
> protected Server getServer(ILoadBalancer loadBalancer) {
>         if (loadBalancer == null) {
>             return null;
>         }
>         return loadBalancer.chooseServer("default"); 
> }
> ```
>
> 这里的 loadBalancer 对象就是前面介绍的 Netflix Ribbon 中的 ILoadBalancer 接口的实现类。这样，我们就把 Spring Cloud Netflix Ribbon 与 Netflix Ribbon 的整体协作流程串联起来。
>
> ### 小结与预告
>
> 在上一课时的基础上，今天我们系统讲解了与 Ribbon 相关的基本架构和实现原理，涉及两大块内容，一块是作为一个独立组件的 Netflix Ribbon，一块是与 Spring Cloud 进行整合而形成的 Spring Cloud Netflix Ribbon。我们讨论了 Netflix Ribbon 中具备的负载均衡策略，也给出了 @LoadBalanced 注解背后的实现原理。
>
> 这里给你留一道思考题：为什么在 RestTemplate 上添加一个 @LoadBalanced 注解之后就自动具备负载均衡功能呢？
>
> 从下一课时开始，我们将进入一个新的主题，微服务架构中的 API 网关。我们将讨论如何使用 Zuul 来构建 API 网关。