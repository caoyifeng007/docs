[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4751)



> 在介绍完 Eureka 服务器端组件之后，今天我们详细地展开讲解 Eureka 的客户端组件。我们的思路同样是先介绍使用 Eureka 注册和发现服务的使用方法，然后基于源码，剖析 Eureka 客户端的实现原理。
>
> ### 使用 Eureka 注册和发现服务
>
> 现在我们的 SpringHealth 案例中已经有了第一个独立的微服务，即**上一课时构建的 eureka-server 服务**。对于 Eureka 服务器而言，user-service、device-service 和 intervention-service 都是它的客户端，今天我们将先以 user-service 为例来演示如何完成服务的注册和发现。
>
> #### 实现服务注册
>
> 使用 Eureka 注册基于 Spring Boot 构建的 user-service，它非常简单，其主要工作也是通过配置来完成的。在介绍配置内容之前，我们首先需要确保在 Maven 工程中添加对 Eureka 客户端组件 spring-cloud-starter-netflix-eureka-client 的依赖，如下所示。
>
> 复制代码
>
> ```
> <dependency>
>      <groupId>org.springframework.cloud</groupId>
>      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
> </dependency>
> ```
>
> 然后，我们来看 user-service 的 Bootstrap 类，如下所示：
>
> 复制代码
>
> ```
> @SpringBootApplication
> @EnableEurekaClient
> public class UserApplication {
> 	public static void main(String[] args) {
> 	 
>         SpringApplication.run(UserApplication.class, args);
>     }
> }
> ```
>
> 这里引入了一个新的注解 @EnableEurekaClient，该注解用于表明当前服务就是一个 Eureka 客户端，这样该服务就可以自动注册到 Eureka 服务器。当然，随着我们后续内容的演进，你会发现可以使用统一的 @SpringCloudApplication 注解，来实现 @SpringBootApplication 和 @EnableEurekaClient 这两个注解整合在一起的效果。
>
> 这里使用独立的 @EnableEurekaClient 注解是为了帮助你更好地理解该注解的作用，而关于 @SpringCloudApplication 注解我们会在介绍到服务熔断时再进行专门引入。
>
> 接下来就是最重要的配置工作，user-service 中的配置内容如下所示：
>
> 复制代码
>
> ```
> spring:
>   application:
> 	name: userservice 
> server:
>   port: 8081
> 	 
> eureka:
>   client:
>     serviceUrl:
> 	  defaultZone: http://localhost:8761/eureka/
> ```
>
> 显然，这里包含两段配置内容。第一段配置指定了服务的名称和运行时端口。在上面的示例中，user-service 的名称通过“spring.application.name=userservice”进行指定，也就是说 user-service 在注册中心中的名称为 userservice。在后续的示例中，我们会使用这一名称获取 user-service 在 Eureka 中的各种注册信息。
>
> 在 eureka.client 段中，我们设置 Eureka 客户端行为。这里的 serviceUrl 配置项在上一课时中已经介绍过，serviceUrl.defaultZone 指定的就是 Eureka 服务器的地址。
>
> 当然，如果我们同样基于上一课时中介绍的 Peer Awareness 模式构建了 Eureka 服务器集群，那么 eureka.client.serviceUrl.defaultZone 配置项的内容就应该是“http://eureka1:8761/eureka/,http://eureka2:8762/eureka/”，用于指向当前的集群环境。
>
> #### 实现服务发现
>
> 当我们成功创建并启动了 user-service 之后，Eureka 服务器的当前状态如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/58/DB/Ciqc1F9wT66APdccAAB8da4b_EM993.png)
>
> 包含 user-service 服务注册信息的 Eureka 服务监控页面
>
> 可以看到，现在的 Eureka 中注册了两个 user-service 的服务实例，运行端口分别是 8082 和 8083。这时候，你可能会好奇，user-service 在 Eureka 服务器中的注册信息是如何进行表示的呢？为了获取注册到 Eureka 服务器上具体某一个服务实例的详细信息，我们可以访问如下地址：
>
> 复制代码
>
> ```
> http://<eureka-ip-port>:8761/eureka/apps/<APPID>
> ```
>
> 该地址代表的就是一个普通的 HTTP 请求，URL 中的 APPID 就是服务名称。以 user-service 为例，我们发送 HTTP 请求到 http://localhost:8761/eureka/apps/userservice 可以得到如下信息：
>
> 复制代码
>
> ```
> <application> 
>   <name>USERSERVICE</name>
>   <instance> 
>     <instanceId>localhost:userservice:8082</instanceId>
>     <hostName>localhost</hostName>
>     <app>USERSERVICE</app>
>     <ipAddr>192.168.247.1</ipAddr>
>     <status>UP</status>
>     <overriddenstatus>UNKNOWN</overriddenstatus>
>     <port enabled="true">8082</port>
>     <securePort enabled="false">443</securePort>
>     <countryId>1</countryId>
>     <dataCenterInfo class="com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo"> 
>       <name>MyOwn</name> 
>     </dataCenterInfo>
>     <leaseInfo> 
>       <renewalIntervalInSecs>30</renewalIntervalInSecs>
>       <durationInSecs>90</durationInSecs>
>       <registrationTimestamp>1599277974858</registrationTimestamp>
>       <lastRenewalTimestamp>1599278364582</lastRenewalTimestamp>
>       <evictionTimestamp>0</evictionTimestamp>
>       <serviceUpTimestamp>1599277974859</serviceUpTimestamp> 
>     </leaseInfo>
>     <metadata> 
>       <management.port>8082</management.port> 
>     </metadata>
>     <homePageUrl>http://localhost:8082/</homePageUrl>
>     <statusPageUrl>http://localhost:8082/actuator/info</statusPageUrl>
>     <healthCheckUrl>http://localhost:8082/actuator/health</healthCheckUrl>
>     <vipAddress>userservice</vipAddress>
>     <secureVipAddress>userservice</secureVipAddress>
>     <isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>
>     <lastUpdatedTimestamp>1599277974860</lastUpdatedTimestamp>
>     <lastDirtyTimestamp>1599277974520</lastDirtyTimestamp>
>     <actionType>ADDED</actionType> 
>   </instance>
>   <instance>
> 	…
> 	</instance> 
> </application>
> ```
>
> 这里出现了两个标签代表存在两个 user-service 服务实例。根据如上所示的服务实例详细信息，我们可以获取该服务的服务名称、IP 地址、端口、是否可用等基本信息，也可以访问 statusPageUrl、healthCheckUrl 等地址查看当前服务的运行状态，更为重要的是得到了 leaseInfo 等与服务注册过程直接相关的基础数据，这些基础数据有助于我们理解 Eureka 作为注册中心的工作原理。
>
> ### 理解 Eureka 客户端基本原理
>
> 对于 Eureka 而言，微服务的提供者和消费者都是它的客户端，其中服务提供者关注**服务注册、服务续约**和**服务下线**等功能，而服务消费者关注于服务信息的获取。同时，对于服务消费者而言，为了提高服务获取的性能以及在注册中心不可用的情况下继续使用服务，一般都还会具有缓存机制。
>
> 在 Netflix Eureka 中，专门提供了一个客户端包，并抽象了一个客户端接口 EurekaClient。EurekaClient 接口继承自 LookupService 接口，这个 LookupService 接口实际上也是我们上一课时中所介绍的 InstanceRegistry 接口的父接口。EurekaClient 在 LookupService 接口的基础上提供了一系列扩展方法，**这些扩展方法并不是重点，我们还是更应该关注于它的类层机构**，如下所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/58/E7/CgqCHl9wUAmATPQxAAAqMtQEPGk029.png)
>
> 接口 EurekaClient 的类层结构
>
> 可以看到 EurekaClient 接口有个实现类 DiscoveryClient（位于 com.netflix.discovery 包中），该类包含了服务提供者和服务消费者的核心处理逻辑，同时提供了我们在介绍 Eureka 服务器端基本原理时所介绍的 register、renew 等方法。DiscoveryClient 类的实现非常复杂，我们重点关注它构造方法中的这行代码：
>
> 复制代码
>
> ```
> initScheduledTasks();
> ```
>
> 通过分析该方法中的代码，我们看到系统在这里初始化了一批调度任务，具体包含缓存刷新 cacheRefresh、心跳 heartbeat、服务实例复制 InstanceInfoReplicator 等，其中缓存刷新面向服务消费者，而心跳和服务实例复制面向服务提供者。接下来我们将分别从这两个 Eureka 客户端组件出发讨论服务注册和发现的客户端操作。
>
> #### 服务提供者操作源码解析
>
> 服务提供者关注**服务注册、服务续约和服务下线**等功能，它可以使用 Eureka 服务器提供的 RESTful API 完成上述操作。因为篇幅关系，这里同样以服务注册为例给出服务提供者的操作流程。
>
> 在 DiscoveryClient 类中，服务注册操作由register 方法完成，如下所示。为了简单起见，我们对代码进行了裁剪，省略了日志相关等非核心代码：
>
> 复制代码
>
> ```
> boolean register() throws Throwable {
>         EurekaHttpResponse<Void> httpResponse;
>         try {
>             httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
>         } catch (Exception e) {
>             throw e;
>         }
> 
>         return httpResponse.getStatusCode() == 204;
> }
> ```
>
> 上述 register 方法会在 InstanceInfoReplicator 类的 run 方法中进行执行。从操作流程上讲，上述代码的逻辑非常简单，即服务提供者先将自己注册到 Eureka 服务器中，然后根据返回的结果确定操作是否成功。显然，这里的重点代码是eurekaTransport.registrationClient.register()，DiscoveryClient 通过这行代码发起了远程请求。
>
> 首先我们来看 EurekaTransport 类，这是 DiscoveryClient 类中的一个内部类，定义了 registrationClient 变量用于实现服务注册。registrationClient 的类型是 EurekaHttpClient 接口，该接口的定义如下：
>
> 复制代码
>
> ```
> public interface EurekaHttpClient {
>     EurekaHttpResponse<Void> register(InstanceInfo info);
>     EurekaHttpResponse<Void> cancel(String appName, String id);
>     EurekaHttpResponse<InstanceInfo> sendHeartBeat(String appName, String id, InstanceInfo info, InstanceStatus overriddenStatus);
>     EurekaHttpResponse<Void> statusUpdate(String appName, String id, InstanceStatus newStatus, InstanceInfo info);
>     EurekaHttpResponse<Void> deleteStatusOverride(String appName, String id, InstanceInfo info);
>     EurekaHttpResponse<Applications> getApplications(String... regions);
>     EurekaHttpResponse<Applications> getDelta(String... regions);
>     EurekaHttpResponse<Applications> getVip(String vipAddress, String... regions);
>     EurekaHttpResponse<Applications> getSecureVip(String secureVipAddress, String... regions);
>     EurekaHttpResponse<Application> getApplication(String appName);
>     EurekaHttpResponse<InstanceInfo> getInstance(String appName, String id);
>     EurekaHttpResponse<InstanceInfo> getInstance(String id);
>     void shutdown();
> }
> ```
>
> 可以看到这个 EurekaHttpClient 接口定义了 Eureka 服务器的一些底层 REST API，包括 register、cancel、sendHeartBeat、statusUpdate、getApplications 等。在 Eureka 中，关于如何实现客户端与服务器端的远程通信，从工作原理上讲只是一个 RESTful 风格的 HTTP 请求，但在具体设计和实现上可以说是非常考究，因此类层结构上也比较复杂。我们先来看 EurekaHttpClient 接口的一个实现类 EurekaHttpClientDecorator，从命名上看它是一个装饰器（Decorator），如下所示：
>
> 复制代码
>
> ```
> public abstract class EurekaHttpClientDecorator implements EurekaHttpClient {
>  
>     public enum RequestType {
>         Register,
>         Cancel,
>         SendHeartBeat,
>         StatusUpdate,
>         DeleteStatusOverride,
>         GetApplications,
>         …
>     }
>  
>     public interface RequestExecutor<R> {
>         EurekaHttpResponse<R> execute(EurekaHttpClient delegate);
>  
>         RequestType getRequestType();
>     }
>  
>     protected abstract <R> EurekaHttpResponse<R> execute(RequestExecutor<R> requestExecutor);
>  
>     @Override
>     public EurekaHttpResponse<Void> register(final InstanceInfo info) {
>         return execute(new RequestExecutor<Void>() {
>             @Override
>             public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
>                 return delegate.register(info);
>             }
>  
>             @Override
>             public RequestType getRequestType() {
>                 return RequestType.Register;
>             }
>         });
> 	    }
> 	 
> 	//省略其他方法实现
> }
> ```
>
> 可以看到 EurekaHttpClientDecorator 通过定义一个抽象方法 execute(RequestExecutor requestExecutor) 来包装 EurekaHttpClient，这种包装是代理机制的一种表现形式。
>
> 
>  然后我们再来看如何构建一个 EurekaHttpClient，Eureka 也专门提供了 EurekaHttpClientFactory 类来负责构建具体的 EurekaHttpClient。显然，这是工厂模式的一种典型应用。EurekaHttpClientFactory 接口定义如下：
>
> 复制代码
>
> ```
> public interface EurekaHttpClientFactory {
>     EurekaHttpClient newClient();
>     void shutdown();
> }
> ```
>
> Eureka 中存在一批 EurekaHttpClientFactory 的实现类，包括 RetryableEurekaHttpClient 和 MetricsCollectingEurekaHttpClient 等，这些类都位于 com.netflix.discovery.shared.transport.decorator 包下。同时，在 com.netflix.discovery.shared.transport 包下，还存在一个 EurekaHttpClients 工具类，能够创建通过 RedirectingEurekaHttpClient、RetryableEurekaHttpClient、SessionedEurekaHttpClient 包装之后的 EurekaHttpClient。如下所示：
>
> 复制代码
>
> ```
> new EurekaHttpClientFactory() {
>             @Override
>             public EurekaHttpClient newClient() {
>                 return new SessionedEurekaHttpClient(
>                         name,
>                         RetryableEurekaHttpClient.createFactory(
>                                 name,
>                                 transportConfig,
>                                 clusterResolver,
>                                 RedirectingEurekaHttpClient.createFactory(transportClientFactory),
>                                 ServerStatusEvaluators.legacyEvaluator()),
>                         transportConfig.getSessionedClientReconnectIntervalSeconds() * 1000
>                 );
>             }
> };
> ```
>
> 这是 EurekaHttpClient 创建过程中的一条分支，即通过包装器对请求过程进行层层封装和代理。而在执行远程请求时，Eureka 同样提供了另一套体系来完成真正的远程调用，原始的 EurekaHttpClient 通过 TransportClientFactory 进行创建。TransportClientFactory 接口定义如下：
>
> 复制代码
>
> ```
> public interface TransportClientFactory {
>     EurekaHttpClient newClient(EurekaEndpoint serviceUrl);
>     void shutdown();
> }
> ```
>
> TransportClientFactory 同样存在一批实现类，其中有些是实名类，有些是匿名类。以实名的实现类 JerseyEurekaHttpClientFactory 为例，它位于 com.netflix.discovery.shared.transport.jersey 包下，通过 EurekaJerseyClient 获取 Jersey 客户端，而 EurekaJerseyClient 又会使用 ApacheHttpClient4 对象，从而完成 REST 调用。
>
> 作为总结，这里也给你分享一个 Eureka 在设计和实现上的技巧，也就是所谓的高阶（High Level）API和低阶（Low Level）API，如下图所示：
>
> ![Lark20201009-104135.png](https://s0.lgstatic.com/i/image/M00/5B/A4/CgqCHl9_zgeAKVhNAAHaQoJ_1kI602.png)
>
> 高阶 API 和低阶 API 关系示意图
>
> 针对高阶 API，主要是通过装饰器模式进行一系列包装，从而创建目标 EurekaHttpClient。而关于低阶 API 的话，主要是 HTTP 远程调用的实现，Netflix 提供的是基于 Jersey 的版本，而 Spring Cloud 则提供了基于 RestTemplate 的版本，这点我们后面会再讲到。
>
> #### 服务消费者操作源码解析
>
> 我们在介绍注册中心模型时，服务消费者可以配备缓存机制以加速服务路由。对于 Eureka 而言，作为客户端组件的 DiscoveryClient 同样具备这种缓存功能。
>
> Eureka 客户端通过定时任务完成缓存刷新操作，我们已经在前面的内容中提到 DiscoveryClient 中的 initScheduledTasks 方法用于初始化各种调度任务，对于缓存刷选而言，调度器的初始化过程如下所示：
>
> 复制代码
>
> ```
> if (clientConfig.shouldFetchRegistry()) {
>             int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
>             int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
>             scheduler.schedule(
>                     new TimedSupervisorTask(
>                             "cacheRefresh",
>                             scheduler,
>                             cacheRefreshExecutor,
>                             registryFetchIntervalSeconds,
>                             TimeUnit.SECONDS,
>                             expBackOffBound,
>                             new CacheRefreshThread()
>                     ),
>                     registryFetchIntervalSeconds, TimeUnit.SECONDS);
> }
> ```
>
> 显然，这里启动了一个调度任务并通过 CacheRefreshThread 线程完成具体操作。CacheRefreshThread 线程定义如下：
>
> 复制代码
>
> ```
> class CacheRefreshThread implements Runnable {
>         public void run() {
>             refreshRegistry();
>         }
> }
> ```
>
> 对于服务消费者而言，最重要的操作就是**获取服务注册信息**。在这里的 refreshRegistry 方法中，我们发现在进行一系列的校验之后，最终调用了 fetchRegistry 方法以完成注册信息的更新，该方法代码如下。为了简单起见，我们对代码进行了部分裁剪，只保留主流程：
>
> 复制代码
>
> ```
> private boolean fetchRegistry(boolean forceFullRegistryFetch) {
>         try {
>             // 获取应用
>             Applications applications = getApplications();
>             if (…) //如果满足全量拉取条件
>             {
>               // 全量拉取服务实例数据
>                 getAndStoreFullRegistry();
>             } else {
>               // 增量拉取服务实例数据
>                 getAndUpdateDelta(applications);
>             } 
>            // 重新计算和设置一致性hashcode
> 	applications.setAppsHashCode(applications.getReconcileHashCode());
> 
>         } 
>  
>         // 刷新本地缓存
>         onCacheRefreshed();
>         // 更新远程服务实例运行状态
>         updateInstanceRemoteStatus();
>  
>         return true;
> }
> ```
>
> 这里的几个带注释的方法都非常有用，因为 getAndStoreFullRegistry 的逻辑相对比较简单，我们将重点介绍 getAndUpdateDelta 方法，以便学习在 Eureka 中如何实现增量数据更新的设计技巧。裁剪之后的 getAndUpdateDelta 方法代码如下所示：
>
> 复制代码
>
> ```
> private void getAndUpdateDelta(Applications applications) throws Throwable {
>         long currentUpdateGeneration = fetchRegistryGeneration.get();
>  
>         Applications delta = null;
>         //通过 eurekaTransport.queryClient 获取增量信息
>         EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
>         if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
>             delta = httpResponse.getEntity();
>         }
>  
>         if (delta == null) {
>               //如果增量信息为空，就直接发起一次全量更新
>             getAndStoreFullRegistry();
>         } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {//通过CAS来确保请求的线程安全性
>             String reconcileHashCode = "";
>             if (fetchRegistryUpdateLock.tryLock()) {
>                 try {
>                  //比对从服务器端返回的增量数据和本地数据，合并两者的差异数据
>                     updateDelta(delta);
>  
> 	//用合并了增量数据之后的本地数据来生成一致性 hashcode
>                     reconcileHashCode = getReconcileHashCode(applications);
>                 } finally {
>                     fetchRegistryUpdateLock.unlock();
>                 }
>             } else {
>             }
>  
> 	//比较本地数据中的 hashcode 和来自服务器端的 hashcode
>             if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
>                  //如果 hashcode 不一致，就触发远程调用进行全量更新
>                 reconcileAndLogDifference(delta, reconcileHashCode);
>             }
>         } else {
>         }
> }
> ```
>
> 回顾 Eureka 服务器端基本原理，我们知道 Eureka 服务器端会保存一个服务注册列表的缓存。Eureka 官方文档中提到这个数据保留时间是三分钟，而 Eureka 客户端的定时调度机制会每隔 30 秒刷选本地缓存。原则上，只要 Eureka 客户端不停地获取服务器端的更新数据，就能保证自己的数据和 Eureka 服务器端的保持一致。但如果客户端在 3 分钟之内没有获取更新数据，就会导致自身与服务器端的数据不一致，这是这种更新机制所必须要考虑的问题，也是我们自己在设计类似场景时的一个注意点。
>
> 针对上述问题，Eureka 采用了一致性 HashCode 方法来进行解决。Eureka 服务器端每次返回的增量数据中都会带有一个一致性 HashCode，这个 HashCode 会与 Eureka 客户端用本地服务列表数据算出的一致性 HashCode 进行比对，如果两者不一致就证明增量更新出了问题，这时候就需要执行一次全量更新。
>
> 在 Eureka 中，计算一致性 HashCode 的方法如下所示，可以看到这一方法基于服务注册实例信息完成编码计算过程，最终返回一个 String 类型的计算结果：
>
> 复制代码
>
> ```
> public static String getReconcileHashCode(Map<String, AtomicInteger> instanceCountMap) {
>         StringBuilder reconcileHashCode = new StringBuilder(75);
>         for (Map.Entry<String, AtomicInteger> mapEntry : instanceCountMap.entrySet()) {
>             reconcileHashCode.append(mapEntry.getKey()).append(STATUS_DELIMITER).append(mapEntry.getValue().get())
>                     .append(STATUS_DELIMITER);
>         }
>         return reconcileHashCode.toString();
> }
> ```
>
> 作为总结，Eureka 客户端缓存定时更新的流程如下图所示，可以看到它与服务注册的流程基本一致，也就是说在 Eureka 中，服务提供者和服务消费者作为 Eureka 服务器的客户端采用了同一套体系完成与服务器端的交互。
>
> ![Lark20201009-104138.png](https://s0.lgstatic.com/i/image/M00/5B/99/Ciqc1F9_zfGAOViSAAGXRnBlAdc236.png)
>
> Eureka 缓存刷选流程时序图
>
> ### 小结与预告
>
> 延续上一课时内容，今天我们讨论了 Eureka 客户端组件的使用方法和实现原理。在使用方法上，我们同样只需要在 Spring Boot 的启动类中添加一个注解，就可以将服务自身注册到 Eureka 服务器中。而在实现原理上，服务的提供者和服务的消费者都是 Eureka 客户端，但却有不同的操作流程，需要我们分别进行分析。
>
> 这里给你留一道思考题：针对位于 Eureka 服务器上的服务列表信息，Eureka 客户端如何实现注册信息的同步和增量更新？
>
> 负载均衡与服务治理关系密切。下一课时，我们就将基于Eureka 的已知内容，讨论 Spring Cloud 中的客户端负载均衡组件 Ribbon 与 Eureka 之间的交互关系以及使用方法。