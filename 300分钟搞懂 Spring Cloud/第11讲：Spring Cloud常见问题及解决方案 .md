[300分钟掌握 Spring Cloud 微服务核心 - 《Spring Cloud 微服务》系列书作者尹吉欢- 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=9&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=99)



> 本课时我们主要讲解 Eureka 服务发现慢的原因，Spring Cloud 组件的重试和调优，以及 Zuul 动态路由、Feign 动态日志级别等内容。
>
> # Eureka 服务快速发现的背景
>
> 如果你刚刚接触 Eureka，对 Eureka 的设计和实现都不是很了解，可能就会遇到一些无法快速解决的问题，这些问题包括：新服务上线后，服务消费者不能访问到刚上线的新服务，需要过一段时间后才能访问？或是将服务下线后，服务还是会被调用到，一段时候后才彻底停止服务，访问前期会导致频繁报错？
>
>  
>
> 这些问题还会让你对 Spring Cloud 产生严重的怀疑，这难道不是一个 Bug?
>
>  
>
> 这其实就是服务发现的一个问题，当我们需要调用服务实例时，信息是从注册中心 Eureka 获取的，然后通过 Ribbon 选择一个服务实例发起调用，如果出现调用不到或者下线后还可以调用的问题，原因肯定是服务实例的信息更新不及时导致的，在面试中如果能解决这些问题会是很大的加分项，既能让面试官知道你是有真正实战经验的，又能让面试官对你刮目相看，因为你不仅会简单使用框架，对其背后的原理也相当熟知。
>
>  
>
> 当我们了解了服务发现变慢会产生的问题后，我们接着来具体分析服务发现不及时背后的原因。
>
> # Eureka 服务发现慢的原因
>
> ![img](https://s0.lgstatic.com/i/image3/M01/87/D3/Cgq2xl6UKu-AWCOVAAGDqNRLidI263.png)
>
> Eureka 服务发现慢的原因主要有两个，一部分是因为服务缓存导致的，另一部分是因为客户端缓存导致的。
>
> ## 1. 服务端缓存
>
> 服务注册到注册中心后，服务实例信息是存储在注册表中的，也就是内存中。但 Eureka  为了提高响应速度，在内部做了优化，加入了两层的缓存结构，将 Client 需要的实例信息，直接缓存起来，获取的时候直接从缓存中拿数据然后响应给 Client。
>
>  
>
> 第一层缓存是 readOnlyCacheMap，readOnlyCacheMap 是采用 ConcurrentHashMap
>
>  来存储数据的，主要负责定时与 readWriteCacheMap 进行数据同步，默认同步时间为 30 秒一次。
>
>  
>
> 第二层缓存是 readWriteCacheMap，readWriteCacheMap 采用 Guava 来实现缓存。 缓存过期时间默认为 180 秒，当服务下线、过期、注册、状态变更等操作都会清除此缓存中的数据。
>
>  
>
> Client 获取服务实例数据时，会先从一级缓存中获取，如果一级缓存中不存在，再从二级缓存中获取，如果二级缓存也不存在，会触发缓存的加载，从存储层拉取数据到缓存中，然后再返回给 Client。
>
>  
>
> Eureka 之所以设计二级缓存机制，也是为了提高 Eureka Server 的响应速度，缺点是缓存会导致 Client 获取不到最新的服务实例信息，然后导致无法快速发现新的服务和已下线的服务。
>
> 
>
> 了解了服务端的实现后，想要解决这个问题就变得很简单了，我们可以缩短只读缓存的更新时间（eureka.server.response-cache-update-interval-ms）让服务发现变得更加及时，或者直接将只读缓存关闭（eureka.server.use-read-only-response-cache=false），直接将只读缓存关闭适合服务量小的场景。
>
>  
>
> Eureka Server 中会有定时任务去检测失效的服务，将服务实例信息从注册表中移除，也可以将这个失效检测的时间缩短，这样服务下线后就能够及时从注册表中清除。
>
> ## 2. 客户端缓存
>
> 客户端缓存主要分为两块内容，一块是 Eureka Client 缓存，一块是 Ribbon 缓存。
>
> ### Eureka Client 缓存
>
> Eureka Client 负责跟 Eureka Server 进行交互，在 Eureka Client 中的com.netflix.discovery.DiscoveryClient.initScheduledTasks() 方法中，初始化了一个 CacheRefreshThread 定时任务专门用来拉取 Eureka Server 的实例信息到本地。
>
>  
>
> 所以我们需要缩短这个定时拉取服务信息的时间间隔（eureka.client.registryFetchIntervalSeconds）来快速发现新的服务。
>
> ### Ribbon 缓存
>
> Ribbon 会从 Eureka Client 中获取服务信息，ServerListUpdater 是 Ribbon 中负责服务实例更新的组件，默认的实现是 PollingServerListUpdater，通过线程定时去更新实例信息。定时刷新的时间间隔默认是 30 秒，当服务停止或者上线后，这边最快也需要 30 秒才能将实例信息更新成最新的。我们可以将这个时间调短一点，比如 3 秒。
>
>  
>
> 刷新间隔的参数是通过 getRefreshIntervalMs 方法来获取的，方法中的逻辑也是从 Ribbon 的配置中进行取值的。
>
>  
>
> 将这些服务端缓存和客户端缓存的时间全部缩短后，跟默认的配置时间相比，快了很多。我们通过调整参数的方式来尽量加快服务发现的速度，但是还是不能完全解决报错的问题，间隔时间设置为 3 秒，也还是会有间隔。所以我们一般都会开启重试功能，当路由的服务出现问题时，可以重试到另一个服务来保证这次请求的成功，关于重试的内容在下一课时中我会具体讲解。
>
>  
>
> 上一课时在讲灰度发布的时候，给大家讲过如何进行服务的隔离，像发布这种常见需求，如果我们做了灰度功能，也就不会出现服务发现慢的问题。
>
>  
>
> 服务发布前，首先需要进行隔离，也就意味着不会有请求过来，这样就不会出现下线了还在请求的问题，即使客户端本地有服务信息缓存，也没关系，此时会在 Ribbon 进行选择时被过滤掉。新服务也需要测试一遍，等没问题后再正式发布，这个时间空隙，客户端定时更新服务实例信息都已经做完了，也就解决了上面遇到的问题。
>
> # Spring Cloud 各组件超时
>
> 在 Spring Cloud 中，应用的组件较多，Zuul 中会转发请求到后端服务，后端服务也会调用其他的后端服务，只要涉及通信，就有可能会发生请求超时。那么如何设置超时时间？哪些组件需要设置等一些列问题？便成了面试中最常见的问题。
>
>  
>
> 在 Spring Cloud 中，超时时间只需要关注 Ribbon 和 Hystrix 即可。其他的了解就可以。
>
> ## Zuul
>
> 首先来看 Zuul 中超时的配置，如果在 Zuul 中是通过 URL 方式进行转发的，可以通过设置 zuul.host.connect-timeout-millis 和 zuul.host.socket-timeout-millis 来调整超时时间。
>
> ## Ribbon
>
> 如果采用的是服务发现方式，就可以通过服务名去进行转发，需要配置 Ribbon 的超时。Rbbon 的超时可以配置全局的 ribbon.ReadTimeout 和 ribbon.ConnectTimeout。也可以在前面指定服务名，为每个服务单独配置，比如 user-service.ribbon.ReadTimeout。
>
>  
>
> 其次是 Hystrix 的超时配置，Hystrix 的超时时间要大于 Ribbon 的超时时间，因为 Hystrix 将请求包装了起来，特别需要注意的是，如果 Ribbon 开启了重试机制，比如重试 3 次，Ribbon 的超时为 1 秒，那么 Hystrix 的超时时间应该大于 3 秒，否则就会出现 Ribbon 还在重试中，而 Hystrix 已经超时的现象。
>
> ## Hystrix
>
> Hystrix 全局超时配置就可以用 default 来代替具体的 command 名称。
>
> hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000
>
>  
>
> 如果想对具体的 command 进行配置，那么就需要知道 command 名称的生成规则，才能准确的配置。
>
>  
>
> 如果我们使用 @HystrixCommand 的话，可以自定义 commandKey。
>
>  
>
> 如果使用 Feign Client 的话，可以为 Feign Client 来指定超时时间：
>
> hystrix.command.UserRemoteClient.execution.iso
>
> lation.thread.timeoutInMilliseconds = 3000
>
>  
>
> 如果想对 Feign Client 中的某个接口设置单独的超时，可以在 Feign Client 名称后加上具体的方法：
>
> hystrix.command.UserRemoteClient#getUser(Lon
>
> g).execution.isolation.thread.timeoutInMilliseconds = 3000
>
> ## Feign
>
> Feign 本身也有超时时间的设置，如果此时设置了 Ribbon 的时间就以 Ribbon 的时间为准，如果没设置 Ribbon 的时间但配置了 Feign 的时间，就以 Feign 的时间为准。Feign 的时间同样也配置了连接超时时间（feign.client.config.服务名称.connectTimeout）和读取超时时间（feign.client.config.服务名称.readTimeout）。
>
> # Spring Cloud 各组件重试
>
> spring-retry 是一个用于重试的框架，可以使用代码的方式将业务包裹起来，提供重试功能。也可以整合到 Spring 中，通过注解的方式优雅的进行重试。
>
>  
>
> 在 Spring Cloud 中，重试机制都是依赖于 spring-retry 来实现的，使用重试机制的时候我们需要将 spring-retry 加入到 Maven 的依赖中。
>
>  
>
> 然后配置 Ribbon 的重试规则，需要配置对当前实例的重试次数（ribbon.MaxAutoRetries） ，切换实例的重试次数（ribbon.MaxAutoRetriesNextServer），是否需要对所有操作请求都进行重试（ribbon.OkToRetryOnAllOperations），不配置只会对 Get 请求进行重试等规则。
>
>  
>
> 需要注意的是在 Zuul 中需要开启 Zuul 对重试的支持，通过配置 zuul.retryable=true 开启。
>
>  
>
> 关于重试的使用，给大家一些小小的建议：
>
>  
>
> 重试一定要考虑幂等性，否则就会出现数据重复等问题，当然作为一名优秀的工程师，就算不用重试也需要考虑到幂等性问题，因为你不能要求别人怎么调用你的接口。
>
>  
>
> 请求是包裹在 Hystrix 中执行的，如果触发了重试也就意味着整个请求的时间翻倍了，所以 Hystrix 的超时时间一定要大于重试的总时间。
>
>  
>
> 最后一点是重试次数不宜过多，过多的重试次数会占用资源较长的时间，如果发生大批量重试，对下游服务的压力也会急剧增加，需要考虑和避免雪崩问题。
>
> # Spring Cloud 各组件调优
>
> 想要将 Spring Cloud 用于实际的工作中，除了要会使用全家桶的组件外，还要了解各个组件的原理，更加要懂得如何优化各个组件。
>
> ## Zuul
>
> 当你对网关进行压测时，会发现并发量一直上不去，错误率也很高。因为你用的是默认配置，这个时候我们就需要去调整配置以达到最优的效果。
>
>  
>
> 首先我们可以对容器进行调优，最常见的就是内置的 Tomcat 容器了，可以调整请求队列排队数（server.tomcat.accept-count），最大线程数（server.tomcat.max-threads），最大连接数（server.tomcat.max-connections）等参数。
>
>  
>
> 默认 Zuul 中用的是 Hystrix 的信号量（semaphore）隔离模式，并发量上不去很大的原因都在这里，信号量默认值是 100，也就是最大并发只有 100，超过 100 就得等待。我们可以将这个值（zuul.semaphore.max-semaphores）调大。
>
>  
>
> 还需要配置 Zuul 并发信息，每个路由的连接数（zuul.host.max-per-route-connections）和总连接数（zuul.host.max-total-connections）。
>
>  
>
> 最后还需要调整 Ribbon 的并发配置，单服务并发数（ribbon.MaxConnectionsPerHost）和总并发数（ribbon.MaxTotalConnections）。
>
>  
>
> 如果在 Zuul 中用的是线程隔离模式，容器参数，Zuul 参数，Ribbon 参数这些配置保留，只需要将信号量的配置替换成 Hystrix 线程池的配置，比如核心线程数（hystrix.threadpool.default.coreSize），最大线程数（hystrix.threadpool.default.maximumSize），队列的大小（hystrix.threadpool.default.maxQueueSize）等参数。
>
> ## Feign
>
> 服务之间的调用基本上是用 Feign 来调用的，对于 Feign 可以将默认的HttpURLConnection 替换成 httpclient 来提高性能。还需要调整每个路由的连接数（feign.httpclient.max-connections-per-route）和总连接数（feign.httpclient.max-connections）。
>
> # Zuul动态路由
>
> 网关作为流量的入口，尽量不要频繁地重启，可以用默认的路由规则来访问接口，这样当有新的服务上线时就无需修改路由规则。
>
>  
>
> 默认的路由规则主要是通过服务名来进行转发的，也就是网关地址 + 服务名 + 接口的 URI 来访问。
>
>  
>
> 这样的好处在于对于新增加的服务不用去单独配置路由规则，还有就是通过访问的地址可以很直观的看出来这个请求需要转发到哪个服务上。
>
>  
>
> 默认路由不好的点在于会代理注册中心所有的服务，网关一般都会分为不同的类型，比如给 Web 端提供服务，给小程序提供服务等，每个网关需要暴露的 API 和服务都不相同，所以用默认的方式就会暴露出很多客户端不需要的信息，这个时候自己去配置具体的路由规则就非常合适了。
>
>  
>
> 手动配置路由规则也就意味着每次新增或者减少 API 的时候，都需要进行配置，然后重启发布，所以我们需要能够通过动态刷新路由的方式来避免网关重启。
>
>  
>
> Zuul 中刷新路由只需要发布一个 RoutesRefreshedEvent 事件即可，我们可以监听 Apollo 的修改，当路由发生变化的时候就发送 RoutesRefreshedEvent 事件来进行刷新路由。
>
> # Feign 动态日志级别
>
> 当生产环境出问题后，往往需要更详细的日志来辅助我们排查问题。这个时候会将日志级别调整低一点，比如 debug 级别，这样就能输出更详细的日志信息。目前都是通过修改配置，然后重启服务来让日志的修改生效的。
>
>  
>
> 通过整合 Apollo 我们可以做到在配置中心动态调整日志级别的配置，服务不用重启即可刷新日志级别，非常方便。
>
>  
>
> 最常见的就是我们用 Feign 来调用接口，首先我们需要将 Feign 的日志级别设置成 full, 也就是输出全部的请求信息。那么正常的情况下是不开启 Feign 的日志输出的，当需要排查问题的时候，就可以调整日志级别，查看 Feign 的请求信息以及返回的数据。
>
>  
>
> 我们可以在项目中增加一个日志监听修改刷新的类，当有 loggin.level 相关配置发生修改的时候，就向 Spring 发送 EnvironmentChangeEvent 事件来刷新日志的级别。这样就实现了需要根据请求日志排查问题时可以在 Apollo 中将对应的 Feign Client 的级别设置成 debug，然后就可以输出详细的请求信息了。
>
> 
>
> 好了，本课时的内容就全部讲完了，下一课时也是本专栏的最后一个课时，我分享一个 Spring Cloud 综合案例，记得按时来听课啊，下节课见。