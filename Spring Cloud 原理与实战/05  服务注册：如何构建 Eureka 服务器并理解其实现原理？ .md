[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4750)



> 上一课时，我们全面介绍了服务治理的解决方案，引出了 **Spring Cloud Netflix Eureka** 组件。Eureka 分为**服务器端组件**和**客户端组件**，今天我们将讨论 Eureka 服务器的构建方式及其实现原理。
>
> ### 基于 Eureka 构建注册中心
>
> 基于 Eureka 构建服务注册中心涉及两大部分内容，首先我们将给出构建单个 Eureka 服务器的方法。但是，Eureka 服务器不能保证高可用，因此在生产环境中，我们一般都还需要**构建 Eureka 服务器集群**。
>
> #### 1. 构建单点 Eureka 服务器
>
> 我们将创建一个新的 Maven 工程并命名为 eureka-server。eureka-server 是一个 Spring Boot 项目。同时我们引入了 spring-cloud-starter-eureka-server 依赖，该依赖是 Spring Cloud 中实现 Spring Cloud Netflix Eureka 功能的主体 jar 包：
>
> 复制代码
>
> ```
> <dependency>
>      <groupId>org.springframework.cloud</groupId>
>      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
> </dependency>
> ```
>
> 引入 Maven 依赖之后就可以创建 Spring Boot 的启动类，在示例代码中，我们把该启动类命名为 EurekaServerApplication，代码如下所示。
>
> 复制代码
>
> ```
> @SpringBootApplication
> @EnableEurekaServer
> public class EurekaServerApplication {
> 	public static void main(String[] args) {
> 	 
>         SpringApplication.run(EurekaServerApplication.class, args);
>     }
> }
> ```
>
> 请注意，在上面的代码中，我们在启动类上加了一个 @EnableEurekaServer 注解。在 Spring Cloud 中，包含 @EnableEurekaServer 注解的服务意味着就是一个 Eureka 服务器组件。
>
> 我们运行这个 EurekaServerApplication 类并访问网站http://localhost:8761/，如果得到如下图中所示的 Eureka 服务监控页面，则意味着 Eureka 服务器已经启动成功。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/58/E5/CgqCHl9wTn-AADzaAACz58XWQz4286.png)
>
> Eureka 服务监控页面
>
> 虽然目前还没有任何一个服务注册到 Eureka 中，但从上图中，我们还是得到了关于 Eureka 服务器内存、CPU 等的有用信息。
>
> 同时，Eureka 也为开发人员提供了一系列的配置项。这些配置项可以分成三大类，一类用于**控制 Eureka 服务器端行为**，以 **eureka.server** 开头；一类则是从客户端角度出发**考虑配置需求**，以 **eureka.client** 开头；而最后一类则关注于注册到 **Eureka 的服务实例本身**，以 **eureka.instance** 开头。请注意，Eureka 除了充当服务器端组件之外，实际上也可以作为客户端注册到 Eureka 本身，这时候它使用的就是客户端配置项。
>
> Eureka 的配置项很多，我们无意一一进行展开。在日常开发过程中，使用的最多的还是客户端相关的配置，所以这里以客户端配置为例。现在，我们尝试在 eureka-server 工程的 application.yml 文件中添加了如下配置信息。
>
> 复制代码
>
> ```
> server:
>   port: 8761
>  
> eureka:
>   client:
>     registerWithEureka: false
>     fetchRegistry: false
>     serviceUrl:
>       defaultZone: http://localhost:8761
> ```
>
> 在这些配置项中，我们看到了三个以 eureka.client 开头的客户端配置项，它们分别是**registerWithEureka、fetchRegistry** 和**serviceUrl**。从配置项的命名上我们不难看出，registerWithEureka 用于指定是否把当前的客户端实例注册到 Eureka 服务器中，而 fetchRegistry 则用于指定是否从 Eureka 服务器上拉取已注册的服务信息。这两个配置项默认都是 true，但这里都将其设置为 false。因为在微服务体系中，包括 Eureka 服务在内的所有服务对于注册中心来说都可以算作客户端，而 Eureka 服务显然不同于业务服务，我们不希望 Eureka 服务对自身进行注册。而 serviceUrl 配置项用于服务地址，这个配置项在构建 Eureka 服务器集群是很有用，让我们一起来看一下。
>
> #### 2. 构建 Eureka 服务器集群
>
> 前面我们介绍了构建单个 Eureka 服务器的方法，这种运行 Eureka 服务的方式一般称为 Standalone 模式。考虑到单个 Eureka 服务可能存在的单点失效问题，我们通常都需要构建一个 Eureka 服务器集群来确保注册中心本身的可用性。与传统的集群构建方式不同，如果我们把 Eureka 也视为一个服务，也就是说 Eureka服务自身也能注册到其他 Eureka 服务上，从而实现相互注册，并构成一个集群。在 Eureka中，这种实现高可用的部署方式被称为 **Peer Awareness 模式**。
>
> 现在我们准备两个 Eureka 服务实例 **eureka1** 和 **eureka2**。在 Spring Boot 中，我们分别提供 application-eureka1.yml 和 application-eureka2.yml 这两个配置文件来设置相关的配置项。其中 application-eureka1.yml 配置文件的内容如下：
>
> 复制代码
>
> ```
> server:
>   port: 8761
> 
> eureka:
>   instance:
>     hostname: eureka1
>   client
>     serviceUrl
> 	   defaultZone: http:// eureka2:8762/eureka/
> ```
>
> 对应的，application-eureka2.yml 配置文件的内容如下：
>
> 复制代码
>
> ```
> server:
>   port: 8762
> 
> eureka:
>   instance:
>     hostname: eureka2
>   client
>     serviceUrl
> 	   defaultZone: http://eureka1:8761/eureka/
> ```
>
> 这里就出现了一个 **Eureka** 实例管理类配置项 **eureka.instance.hostname**，用于指定当前 Eureka 服务的主机名称。然后，我们注意到 application-eureka1.yml 和 application-eureka2.yml 中的配置项完全一致，区别只是调整了端口和地址的引用。构建 Eureka 集群模式的关键点在于使用客户端配置项 eureka.client.serviceUrl.defaultZone 用于指向集群中的其他 Eureka 服务器。所以 Eureka 集群的构建方式实际上就是将自己作为服务并向其他注册中心注册自己，这样就形成了一组互相注册的服务注册中心以实现服务列表的同步。显然，这个场景下 registerWithEureka 和 fetchRegistry配置项应该都使用其默认的 true 值，所以我们不需要对其进行显式的设置。
>
> 如果你尝试使用本机搭建集群环境，显然 eureka.instance.hostname 配置项中的 eureka1 和 eureka2 是无法访问的，所以需要在本机**hosts 文件**中添加以下信息。
>
> 复制代码
>
> ```
> 127.0.0.1 eureka1
> 127.0.0.1 eureka2
> ```
>
> 现在启动这两个 Eureka 服务，然后分别打开 http://127.0.0.1:8761/ 和 http://127.0.0.1:8762/ 端点可以看到各自的服务注册效果。你可以根据这里的步骤在自己的电脑上演练这个过程，并通过两个 Eureka 服务的启动日志以及控制台界面来验证高可用架构的效果。
>
> ### 理解 Eureka 服务器实现原理
>
> 在介绍完 Eureka 服务器的构建方式之后，我们重点来讲解 Eureka 服务器的实现原理。
>
> #### Eureka 核心概念
>
> 我们在对 Eureka 的内部结构做进一步展开，可以得到如下所示的注册中心细化模型图。
>
> ![Lark20200930-144014.png](https://s0.lgstatic.com/i/image/M00/5A/32/CgqCHl90KMWAEA43AAA7V0eIhi4533.png)
>
> Eureka 细化架构图
>
> 在上图中，Eureka 有以下几个概念与服务治理直接相关，首当其冲的是服务注册。**服务注册**（Register）是服务治理的最基本概念，内嵌了 Eureka 客户端的各个微服务通过向 Eureka 服务器提供 IP 地址、端点等各项与服务发现相关的基本信息完成服务注册操作。
>
> 因为 Eureka 客户端与服务器端通过短连接完成交互，所以在服务续约（Renew）中，Eureka 客户端需要每隔一定时间主动上报自己的运行时状态，从而进行服务续约。
>
> **服务取消（Cancel）\**的意思就是 Eureka 客户端主动告知 Eureka 服务器自己不想再注册到 Eureka 中。当Eureka客户端连续一段时间没有向 Eureka 服务器发送服务续约信息时，Eureka 服务器就会认为该服务实例已经不再运行，从而将其从服务列表中进行\**剔除（Evict）**。
>
> 显然，对于一个注册中心而言，想要理解它的设计理念和实现原理，我们需要分别关注 Eureka 中如何对服务注册信息的存储和管理的具体机制。在接下来的内容中，我们将重点从 Eureka 的服务存储和缓存处理这两个维度出发，基于源码来深入剖析原理。
>
> #### Eureka 服务存储源码解析
>
> 对于一个注册中心而言，我们首先需要关注它的数据存储方法。在 Eureka 中，我们发现 **InstanceRegistry** 接口及其实现类（位于 com.netflix.eureka.registry 包中）承接了这部分职能。InstanceRegistry 的类层结构如下所示：
>
> ![Lark20200930-144023.png](https://s0.lgstatic.com/i/image/M00/5A/27/Ciqc1F90KO2AYINVAANgSO1J2Fs541.png)
>
> InstanceRegistry 类层结构图
>
> 从上图中，不难看出 Spring Cloud 中同样存在一个 InstanceRegistry（位于 org.springframework.cloud.netflix.eureka.server 包中），它实际上是基于 Netflix 中 InstanceRegistry 实现的一种包装。我们在上图中 InstanceRegistry 接口的实现类 AbstractInstanceRegistry 中发现了 Eureka 用于保存注册信息的数据结构，如下所示：
>
> 复制代码
>
> ```
> private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
> ```
>
> 可以看到这是一个**双层的 HashMap**，采用的是 JDK 中线程安全的 **ConcurrentHashMap**。其中第一层的 ConcurrentHashMap 的 Key 为 spring.application.name，也就是服务名，Value 为一个 ConcurrentHashMap；而第二层的 ConcurrentHashMap 的 Key 为 instanceId，也就是服务的唯一实例 ID，Value 为 Lease 对象。Eureka 采用 Lease（租约）这个词来表示对服务注册信息的抽象，Lease 对象保存了服务实例信息以及一些实例服务注册相关的时间，如注册时间 registrationTimestamp、最新的续约时间 lastUpdateTimestamp 等。如果用图形化的表达方式来展示这种数据结构，可以参考下图：
>
> ![Lark20200930-144020.png](https://s0.lgstatic.com/i/image/M00/5A/27/Ciqc1F90KNOATIZoAACHZSqum_8255.png)
>
> 服务注册信息的存储结构示意图
>
> 而对于 InstanceRegistry 本身，它也继承了 Eureka 中两个非常重要的接口，即**LeaseManager 接口**和 **LookupService 接口**。其中 LeaseManager 接口定义如下：
>
> 复制代码
>
> ```
> public interface LeaseManager<T> {
>     void register(T r, int leaseDuration, boolean isReplication);
>     boolean cancel(String appName, String id, boolean isReplication);
>     boolean renew(String appName, String id, boolean isReplication);
>     void evict();
> }
> ```
>
> 显然 **LeaseManager** 做的事情就是 Eureka 注册中心模型中的服**务注册、服务续约、服务取消**和**服务剔除等**核心操作，关注于对服务注册过程的管理。而 LookupService 接口定义如下，关注于对应用程序与服务实例的管理：
>
> 复制代码
>
> ```
> public interface LookupService<T> {
>     Application getApplication(String appName);
>     Applications getApplications();
>     List<InstanceInfo> getInstancesById(String id);
>     InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);
> }
> ```
>
> 在内部实现上，实际上对于注册中心服务器而言，**服务注册、续约、取消**和**剔除**等不同操作所执行的工作流程基本一致，即都是对**服务存储**的操作，并把这一操作同步到其他 Eureka 节点。我们这里选择用于服务注册操作的 register 方法进行展开，register 方法非常长，我们对源码进行裁剪，得出如下所示的核销处理流程：
>
> 复制代码
>
> ```
> public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
>     try { 
>         //从已存储的 registry 获取一个服务定义
>         Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
>         REGISTER.increment(isReplication);
>         if (gMap == null) {
>             //初始化一个 Map<String, Lease<InstanceInfo>> ，并放入 registry 中
>         }
>  
>         //根据当前注册的 ID 找到对应的 Lease
>         Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
>  
>         if (existingLease != null && (existingLease.getHolder() != null)) {
>             //如果 Lease 能找到，根据当前节点的最新更新时间和注册节点的最新更新时间比较
>             //如果前者的时间晚于后者的时间，那么注册实例就以已存在的实例为准
>         } else {
>               //如果找不到，代表是一个新注册，则更新其每分钟期望的续约数量及其阈值
>         }
>  
>         //创建一个新 Lease 并放入 Map 中
>         Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
>         gMap.put(registrant.getId(), lease);
> 
>         //处理服务的 InstanceStatus
>         registrant.setActionType(ActionType.ADDED);
>  
>         //更新服务最新更新时间
>         registrant.setLastUpdatedTimestamp();
>  
>         //刷选缓存
>         invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
>     } 
> }
> ```
>
> AbstractInstanceRegistry 中其他的 cancel、renew 方法也是同样的处理逻辑，这里不再展开。
>
> #### Eureka 服务缓存源码解析
>
> Eureka 服务器端组件的另一个核心功能是**提供服务列表**。为了提高性能，Eureka 服务器会缓存一份所有已注册的服务列表，并通过一定的定时机制对缓存数据进行更新。
>
> 我们知道为了获取注册到 Eureka 服务器上具体某一个服务实例的详细信息，可以访问如下地址：
>
> 复制代码
>
> ```
> http://<eureka-server-ip>:8761/eureka/apps/<APPID>
> ```
>
> 该地址代表的就是一个普通的 HTTP GET 请求。Eureka 中所有对服务器端的访问都是通过**RESTful 风格**的**资源（Resource）** 进行获取，ApplicationResource 类（位于com.netflix.eureka.resources 包中）提供了根据应用获取注册信息的入口。我们来看该类的 getApplication 方法，核心代码如下所示：
>
> 复制代码
>
> ```
> Key cacheKey = new Key(
>        Key.EntityType.Application,
>        appName,
>        keyType,
>        CurrentRequestVersion.get(),
>        EurekaAccept.fromString(eurekaAccept)
> );
>  
> String payLoad = responseCache.get(cacheKey);
>  
> if (payLoad != null) {
>       logger.debug("Found: {}", appName);
>       return Response.ok(payLoad).build();
> } else {
>       logger.debug("Not Found: {}", appName);
>       return Response.status(Status.NOT_FOUND).build();
> }
> ```
>
> 可以看到这里是构建了一个 **cacheKey**，并直接调用了 responseCache.get(cacheKey) 方法来返回一个字符串并构建响应。从命名上看，不难想象这里使用了缓存机制。我们来看 ResponseCache 的定义，如下所示，其中最核心的就是这里的 get 方法：
>
> 复制代码
>
> ```
> public interface ResponseCache {
>     void invalidate(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress);
>  
>     AtomicLong getVersionDelta();
>     AtomicLong getVersionDeltaWithRegions();
>     String get(Key key);
>     byte[] getGZIP(Key key);
> }
> ```
>
> 从类层关系上看，ResponseCache 只有一个**实现类 ResponseCacheImpl**，我们来看它的 get 方法，发现该方法使用了如下处理策略：
>
> 复制代码
>
> ```
> Value getValue(final Key key, boolean useReadOnlyCache) {
>         Value payload = null;
>         try {
>             if (useReadOnlyCache) {
>                 final Value currentPayload = readOnlyCacheMap.get(key);
>                 if (currentPayload != null) {
>                     payload = currentPayload;
>                 } else {
>                     payload = readWriteCacheMap.get(key);
>                     readOnlyCacheMap.put(key, payload);
>                 }
>             } else {
>                 payload = readWriteCacheMap.get(key);
>             }
>         } catch (Throwable t) {
>             logger.error("Cannot get value for key : {}", key, t);
>         }
>         return payload;
> }
> ```
>
> 可以看到上述代码中有两个缓存，一个是 **readOnlyCacheMap**，一个是 **readWriteCacheMap**。其中 readOnlyCacheMap 就是一个 JDK 中的 ConcurrentMap，而 readWriteCacheMap 使用的则是 Google Guava Cache 库中的 LoadingCache 类型。在创建 LoadingCache过程中，缓存数据的来源是调用 generatePayload 方法来生成。而在这个 generatePayload 方法中，就会调用前面介绍的 AbstractInstanceRegistry 中的 getApplications 方法获取应用信息并放到缓存中。这样我们就实现了把注册信息与缓存信息进行关联。
>
> 这里有一个设计和实现上的技巧。把缓存设计为一个只读的 readOnlyCacheMap 以及一个可读写的 readWriteCacheMap，可以更好地分离职责。但因为两个缓存中保存的实际上是同一份数据，所以，我们在不断更新 readWriteCacheMap 时，也需要确保 readOnlyCacheMap 中的数据得到同步。为此 ResponseCacheImpl 提供了一个定时任务 CacheUpdateTask，如下所示：
>
> 复制代码
>
> ```
> private TimerTask getCacheUpdateTask() {
>         return new TimerTask() {
>             @Override
>             public void run() {
>                 for (Key key : readOnlyCacheMap.keySet()) {
>                     try {
>                         CurrentRequestVersion.set(key.getVersion());
>                         Value cacheValue = readWriteCacheMap.get(key);
>                         Value currentCacheValue = readOnlyCacheMap.get(key);
>                         if (cacheValue != currentCacheValue) {
>                             readOnlyCacheMap.put(key, cacheValue);
>                         }
>                     } catch (Throwable th) {
>                     }
>                 }
>             }
>         };
> }
> ```
>
> 显然，这个定时任务主要是从 readWriteCacheMap 更新数据到 readOnlyCacheMap。
>
> #### Eureka 高可用源码解析
>
> 我们已经在前面的内容中了解到 Eureka 的高可用部署方式被称为 **Peer Awareness 模式**。对应的，我们在 **InstanceRegistry 的类层**结构中也已经看到了它的一个扩展接口 **PeerAwareInstanceRegistry** 以及该接口的实现类 PeerAwareInstanceRegistryImpl。
>
> 我们还是围绕服务注册这个场景展开讨论，在 **PeerAwareInstanceRegistryImpl** 中同样存在一个 register 方法，如下所示：
>
> 复制代码
>
> ```
> @Override
> public void register(final InstanceInfo info, final boolean isReplication) {
>         int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
>         if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
>             leaseDuration = info.getLeaseInfo().getDurationInSecs();
>         }
>         super.register(info, leaseDuration, isReplication);
>         replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
> }
> ```
>
> 我们在这里看到了一个非常重要的**replicateToPeers 方法**，该方法作就是用来实现服务器节点之间的状态同步。**replicateToPeers 方法的核心代码**如下所示：
>
> 复制代码
>
> ```
> for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
>     //如何该 URL 代表主机自身，则不用进行注册
>     if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
>          continue;
>     }
>     replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
> }
> ```
>
> 为了理解这个操作，我们首先需要理解 Eureka 中的集群模式，这部分代码位于 com.netflix.eureka.cluster 包中，其中包含了代表节点的 PeerEurekaNode 和 PeerEurekaNodes 类，以及用于节点之间数据传递的 HttpReplicationClient 接口。而 replicateInstanceActionsToPeers 方法中则根据不同的 Action 来调用 PeerEurekaNode 的不同方法。例如，如果是 StatusUpdate Action，则会调动 PeerEurekaNode的statusUpdate 方法，而该方法又会执行如下代码;
>
> 复制代码
>
> ```
> replicationClient.statusUpdate(appName, id, newStatus, info);
> ```
>
> 这句代码完成了 PeerEurekaNode 之间的通信，而 replicationClient 是 HttpReplicationClient 接口的实例，该接口定义如下：
>
> 复制代码
>
> ```
> public interface HttpReplicationClient extends EurekaHttpClient {
>     EurekaHttpResponse<Void> statusUpdate(String asgName, ASGStatus newStatus);
>  
>     EurekaHttpResponse<ReplicationListResponse> submitBatchUpdates(ReplicationList replicationList);
> }
> ```
>
> HttpReplicationClient 接口继承自 EurekaHttpClient 接口，而 EurekaHttpClient 接口属于 Eureka 客户端组件，我们会在下一课时介绍 Eureka 客户端基本原理时进行详细介绍。在这里，我们只需要明白 Eureka 提供了 JerseyReplicationClient（位于 com.netflix.eureka.transport 包下）这一基于 Jersey 框架实现的HttpReplicationClient。以 statusUpdate 方法为例，它的实现过程如下：
>
> 复制代码
>
> ```
> @Override
> public EurekaHttpResponse<Void> statusUpdate(String asgName, ASGStatus newStatus) {
>         ClientResponse response = null;
>         try {
>             String urlPath = "asg/" + asgName + "/status";
>             response = jerseyApacheClient.resource(serviceUrl)
>                     .path(urlPath)
>                     .queryParam("value", newStatus.name())
>                     .header(PeerEurekaNode.HEADER_REPLICATION, "true")
>                     .put(ClientResponse.class);
>             return EurekaHttpResponse.status(response.getStatus());
>         } finally {
>             if (response != null) {
>                 response.close();
>             }
>         }
> }
> ```
>
> 这是典型的基于 Resource 的 RESTful 风格的调用方法，用到了 ApacheHttpClient4 工具类。通过以上分析，我们已经从主要维度上掌握了整个 Eureka 服务器端内部的运行机制。
>
> ### 小结与预告
>
> 今天我们讨论的是 Eureka 服务器端组件的相关内容，可以看到基于 Spring Cloud 框架，构建一个 Eureka 注册中心所需要做的事情仅仅只是添加一个注解。但在内部实现上，Eureka 服务器端需要考虑各个微服务实例的存储和获取等核心流程，也需要考虑如何确保注册中心本身的高可用问题。我们基于源码，对这些流程和问题底层的原理进行了详细的分析。
>
> 这里给你留一道思考题：Eureka 是如何实现自身的高可用架构的？
>
> 讲完 Eureka 服务器端组件，下一课时，我将和你一起继续讨论 Eureka 的客户端组件的使用方法和实现原理。