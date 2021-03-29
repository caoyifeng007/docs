[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4722)



> 你好，欢迎来到第 22 讲，今天我们来学习 Session 的相关内容。
>
> 当我们使用 Spring Boot 加 JPA 的时候，会发现 Spring 帮我们新增了一个 spring.jpa.open-in-view 的配置，但是 Hibernate 本身却没有这个配置，不过其又是和 Hibernate 中的 Session 相关的，因此还是很重要的内容，所以这一讲我们来学习一下。
>
> 由于 Session 不是 JPA 协议规定的，所以官方对于这方面的资料比较少，从业者只能根据个人经验和源码来分析它的本质，那么接下来我就以我个人的经验为你介绍这部分的概念。首先了解 Session 是什么。
>
> ### Session 是什么？
>
> 我们通过一个类的关系图来回顾一下，看看 Session 在什么样的位置上。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/71/66/Ciqc1F--J7aAPxYpAAApgD8vr5o823.png)
>
> 其中，SessionImpl 是 Hibernate 实现 JPA 协议的 EntityManager 的一种实现方式，即实现类；而 Session 是 Hibernate 中的概念，完全符合 EntityManager 的接口协议，同时又完成了 Hibernate 的特殊实现。
>
> 在 Spring Data JPA 的框架中，我们可以狭隘地把 Session 理解为 EntityManager，因为其对于 JPA 的任何操作都是通过 EntityManager 的接口进行的，我们可以把 Session 里面的复杂逻辑当成一个黑盒子。即使 SessionImpl 能够实现 Hibernate 的 Session 接口，但如果我们使用的是 Spring Data JPA，那么实现再多的接口也和我们没有任何关系。
>
> 除非你不用 JPA 的接口，直接用 Hibernate 的 Navite 来实现，但是我不建议你这么做，因为过程太复杂了。那么 SessionImpl 对使用 JPA 体系的人来说，它主要解决了什么问题呢？
>
> ### SessionImpl 解决了什么问题？
>
> 我们通过源码来看一下，请看下面这张图。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/71/66/Ciqc1F--J7-AcywEAAYEtGdc-RE017.png)
>
> 通过 SessionImpl 的源码和 Structure 的视图，我们可以“简单粗暴”地得出如下结论。
>
> 1. SessionImpl 是 EntityManager 的实现类，那么肯定实现了 JPA 协议规定的 EntityManager 的所有功能。比如我们上一课时讲解的 Persistence Context 里面 Entity 状态的所有操作，即管理了 Entity 的生命周期；EntityManager 暴露的 flushModel 的设置；EntityManager 对 Transaction 做了“是否开启新事务”“是否关闭当前事务”的逻辑。
> 2. 如上图所示，实现 PersistenceContext 对象实例化的过程，使得 PersistenceContext 生命周期就是 Session 的生命周期。所以我们可以抽象地理解为，Sesession 是对一些数据库的操作，需要放在同一个上下文的集合中，就是我们常说的一级缓存。
> 3. Session 有 open 的话，那么肯定有 close。open 的时候做了“是否开启事务”“是否获取连接”等逻辑；close 的时候做了“是否关闭事务”“释放连接”等动作；
> 4. Session 的任何操作都离不开事务和连接，那么肯定用当前线程保存了这些资源。
>
> 当我们清楚了 SessionImpl、EntityManager 的这些基础概念之后，那么接着来看看 open-in-view 是什么，它都做了什么事情呢？
>
> ### JPA 里面的 open-in-view 是做什么的？
>
> open-in-view 是 Spring Boot 自动加载 Spring Data JPA 提供的一个配置，全称为 spring.jpa.open-in-view=true，它只有 true 和 false 两个值，默认是 true。那么它到底有什么威力呢？
>
> #### open-in-view 的作用
>
> 我们可以在 JpaBaseConfiguration 中找到关键源码，通过源码来看一下 open-in-view 都做了哪些事情，如下所示。
>
> 复制代码
>
> ```
> public abstract class JpaBaseConfiguration implements BeanFactoryAware {
> @Configuration(proxyBeanMethods = false)
> @ConditionalOnWebApplication(type = Type.SERVLET)
> @ConditionalOnClass(WebMvcConfigurer.class)
> //这个提供了一种自定义注册OpenEntityManagerInViewInterceptor或者OpenEntityManagerInViewFilter的可能，同时我们可以看到在Web的MVC层打开session的两种方式，一种是Interceptor，另外一种是Filter；这两个类任选其一即可，默认用的是OpenEntityManagerInViewInterceptor.class;
> @ConditionalOnMissingBean({ OpenEntityManagerInViewInterceptor.class, OpenEntityManagerInViewFilter.class })
> @ConditionalOnMissingFilterBean(OpenEntityManagerInViewFilter.class)
> //这里使用了spring.jpa.open-in-view的配置，只有为true的时候才会执行这个配置类，当什么都没配置的时候，默认就是true，也就是默认此配置文件就会自动加载；我们可以设置成false，关闭加载；
> @ConditionalOnProperty(prefix = "spring.jpa", name = "open-in-view", havingValue = "true", matchIfMissing = true)
> protected static class JpaWebConfiguration {
>    private static final Log logger = LogFactory.getLog(JpaWebConfiguration.class);
>    private final JpaProperties jpaProperties;
>    protected JpaWebConfiguration(JpaProperties jpaProperties) {
>       this.jpaProperties = jpaProperties;
>    }
> //关键逻辑在OpenEntityManagerInViewInterceptor类里面；加载OpenEntityManagerInViewInterceptor用来在MVC的拦截器里面打开EntityManager，而当我们没有配置spring.jpa.open-in-view的时候，看下面代码spring容器会打印warn日志警告我们，默认开启了open-in-view，提醒我们需要注意影响面，具体有哪些影响面，希望你可以在此篇文章中找到答案，并欢迎留言；
>    @Bean
>    public OpenEntityManagerInViewInterceptor openEntityManagerInViewInterceptor() {
>       if (this.jpaProperties.getOpenInView() == null) {
>          logger.warn("spring.jpa.open-in-view is enabled by default. "
>                + "Therefore, database queries may be performed during view "
>                + "rendering. Explicitly configure spring.jpa.open-in-view to disable this warning");
>       }
>       return new OpenEntityManagerInViewInterceptor();
>    } 
>   //利用WebMvcConfigurer加载上面的OpenEntityManagerInViewInterceptor拦截器进入到MVC里面；
>    @Bean
>    public WebMvcConfigurer openEntityManagerInViewInterceptorConfigurer(  OpenEntityManagerInViewInterceptor interceptor) {
>       return new WebMvcConfigurer() {
>          @Override
>          public void addInterceptors(InterceptorRegistry registry) {
>             registry.addWebRequestInterceptor(interceptor);
>          }
>       };
>    }
> }
> .....//其他不重要的代码省略
> ```
>
> 通过上面的源码我们可以看到，spring.jpa.open-in-view 的主要作用就是帮我们加载 OpenEntityManagerInViewInterceptor 这个类，那么我们再打开这个类的源码，看看它帮我们实现的主要功能是什么？
>
> #### OpenEntityManagerInViewInterceptor 源码分析
>
> 打开这一源码后，可以看到下图所示的界面。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/71/71/CgqCHl--J86AIwh6AATgTHk0WxE893.png)
>
> 我们可以发现，OpenEntityManagerInViewInterceptor 实现了 WebRequestInterceptor 的接口中的两个方法：
>
> 1. public void preHandle(WebRequest request) 方法，里面实现了在每次的 Web MVC 请求之前，通过 createEntityManager 方法创建 EntityManager 和 EntityManagerHolder 的逻辑；
> 2. public void afterCompletion(WebRequest request, @Nullable Exception ex) 方法，里面实现了在每次 Web MVC 的请求结束之后，关闭 EntityManager 的逻辑。
>
> 我们如果继续看 createEntityManager 方法的实现，还会找到如下关键代码。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/71/71/CgqCHl--J9yANzVgAAMLiLh9kQQ355.png)
>
> 上图可以看到，我们通过 SessionFactoryImpl 中的 createEntityManager() 方法，创建了一个 EntityManager 的实现 Session；通过拦截器创建了 EntityManager 事务处理逻辑，默认是 Join 类型（即有事务存在会加入）；而 builder.openSession() 逻辑就是 new SessionImpl(sessionFactory, this)。
>
> 所以这个时候可以知道，通过 open-in-view 配置的拦截器，会帮我们的每个请求都创建一个 SessionImpl 实例；而 SessionImpl 里面存储了整个 PersistenceContext 和各种事务连接状态，可以判断出来 Session 的实例对象比较大。
>
> 并且，我们打开 spring.jap.open-in-view=true 会发现，如果一个请求处理的逻辑比较耗时，牵涉到的对象比较多，这个时候就比较考验我们对 jvm 的内存配置策略了，如果配置不好就会经常出现内存溢出的现象。因此当处理比较耗时的请求和批量处理请求的时候，需要考虑到这一点。
>
> 到这里，经常看源码的同学就应该会好奇了，都有哪些时候需要调用 openSession 呢？那是不是也可以知道 EntityManager(Session) 的打开时机了？
>
> #### EntityManager(Session) 的打开时机及扩展场景
>
> 我们通过 IDEA 开发者工具，直接点击右键查 public Session createEntityManager() 此方法被使用到的地方即可，如下图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/71/71/CgqCHl--J-SAFH9yAAUc9mOMYYk555.png)
>
> 其中，EntityManagerFactoryAccessor 是 OpenEntityManagerInViewInterceptor 的父类，从图上我们可以看得出来，Session 的创建（也可以说是 EntityManager 的创建）对我们有用的时机，目前就有三种。
>
> 第一种：Web View Interceptor，通过 spring.jpa.open-in-view 控制。
>
> 第二种：Web Filter，这种方式是 Spring 给我们提供的另外一种应用场景，比如有些耗时的、批量处理的请求，我们不想在请求的时候开启 Session，而是想在处理简单逻辑后，需要用到延迟加载机制的请求时 Open Session。因为开启 Session 后，我们写框架代码的时候可以利用 lazy 机制。而这个时候我们就可以考虑使用 OpenEntityManagerInViewFilter，配置请求 filter 的过滤机制，实现不同的请求以及不同 Open Session 的逻辑了。
>
> 第三种：JPA Transaction，这种方式就是利用 JpaTransactionManager，实现在事务开启的时候打开 Session，在事务结束的时候关闭 Session。
>
> 所以默认情况下，Session 的开启时机有两个：每个请求之前、新的事务开启之前；而 Session 的关闭时机也是两个：每个请求结束之后、事务关闭之后。
>
> 此外，EntityManager(Session) 打开之后，资源存储在当前线程里面 （ThreadLoacal），所以一个 Session 中即使开启了多个事务，也不会创建多个 EntityManager 或者 Session。
>
> 而事务在关闭之前，也会检查一下此 EntityManager / Session 是不是我这个事务创建的，如果是就关闭，如果不是就不关闭，不过其不会关闭在事务范围之外创建的 EntityManager / Session。
>
> 这个机制其实还给我们一些额外思考：我们是不是可以自由选择开启 / 关闭 Session 呢？不一定是 view / filter / 事务，任何多事务组合的代码模块都可以。只要我们知道什么时间开启，保证一定能 close 就没有问题。
>
> 下面我们通过日志来看一下两种打开、关闭 EntityManager 的时机。
>
> #### 验证 EntityManager 的创建和释放的日志
>
> 第一步：我们新建一个 UserController 的方法，用来模拟请求两段事务的情况，代码如下所示。
>
> 复制代码
>
> ```
> @PostMapping("/user/info")
> public UserInfo saveUserInfo(@RequestBody UserInfo userInfo) {
>    UserInfo u2 = userInfoRepository.findById(1L).orElse(null);
>    if (u2!=null) {
>       u2.setLastName("jack"+userInfo.getLastModifiedTime());
>       //更新u2，新开启一个事务
>       userInfoRepository.save(u2);
>    }
>    //更新userInfo，新开启一个事务
>    return userInfoRepository.save(userInfo);
> }
> ```
>
> 可以看到，里面调用了两个 save 操作，没有指定事务。但是我之前讲过，因为 userInfoRepository 的实现类 SimpleJpaRepository 的 save 方法上面有 @Transactional 注解，所以每个 userInfoRepository.save() 方法就会开启新的事务。我们利用这个机制在上面的 Controller 里面模拟了两个事务。
>
> 第二步：打开 open-in-view，同时修改一些日志级别，方便我们观察，配置如下述代码所示。
>
> 复制代码
>
> ```
> ## 打开open-in-view
> spring.jpa.open-in-view=true
> ## 修改日志级别
> logging.level.org.springframework.orm.jpa.JpaTransactionManager=trace
> logging.level.org.hibernate.internal=trace
> logging.level.org.hibernate.engine.transaction.internal=trace
> ```
>
> 第三步：启动项目，发送如下请求。
>
> 复制代码
>
> ```
> #### update
> POST /user/info HTTP/1.1
> Host: 127.0.0.1:8087
> Content-Type: application/json
> Cache-Control: no-cache
> {"ages":10,"id":3,"version":0}
> ```
>
> 然后我们查看一下日志，关键日志如下图所示。
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/71/71/CgqCHl--KAKAWO2aAAL2Xu59B5I687.png)
>
> 可以看到，我们请求了 user/info 之后就开启了 Session，然后在 Controller 方法执行的过程中开启了两段事务，每个事务结束之后都没有关闭 Session，而是等两个事务都结束之后，并且 Controller 方法执行完毕之后，才 Closing Session 的。中间过程只创建了一次 Session。
>
> 第四步：其他都不变的前提下，我们把 open-in-view 改成 false，如下面这行代码所示。
>
> 复制代码
>
> ```
> spring.jpa.open-in-view=false
> ```
>
> 我们再执行刚才的请求，会得到如下日志。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/71/66/Ciqc1F--KAuAQPOwAATpoE3jbT0924.png)
>
> 通过日志可以看到，其中开启了两次事务，每个事务创建之后都会创建一个 Session，即开启了两个 Session，每个 Session 的 ID 是不一样的；在每个事务结束之后关闭了 Session，关闭了 EntityManager。
>
> 通过上面的事例和日志，我们可以看到 spring.jpa.open-in-view 对 session 和事务的影响，那么它对数据库的连接有什么影响呢？我们看一下 hibernate.connection.handling_mode 这个配置。
>
> ### hibernate.connection.handling_mode 详解
>
> 通过之前讲解的类 AvailableSettings，可以找到如下三个关键配置。
>
> 复制代码
>
> ```
> // 指定获得db连接的方式，hibernate5.2之后已经不推荐使用，改用hibernate.connection.handling_mode配置形式
> String ACQUIRE_CONNECTIONS = "hibernate.connection.acquisition_mode";
> // 释放连接的模式有哪些？hibernate5.2之后也不推荐使用，改用hibernate.connection.handling_mode配置形式
> String RELEASE_CONNECTIONS = "hibernate.connection.release_mode";
> //指定获取连接和释放连接的模式，hibernate5.2之后新增的配置项，代替上面两个旧的配置
> String CONNECTION_HANDLING = "hibernate.connection.handling_mode";
> ```
>
> 那么 hibernate.connection.handling_mode 对应的配置有哪些呢？Hibernate 5 提供了五种模式，我们详细看一下。
>
> #### PhysicalConnectionHandlingMode 的五种模式
>
> 在 Hibernate 5.2 里面，hibernate.connection.handling_mode 这个 Key 对应的值在 PhysicalConnectionHandlingMode 枚举类里面有定义，核心代码如下所示。
>
> 复制代码
>
> ```
> public enum PhysicalConnectionHandlingMode {
>    IMMEDIATE_ACQUISITION_AND_HOLD( IMMEDIATELY, ON_CLOSE ),
>    DELAYED_ACQUISITION_AND_HOLD( AS_NEEDED, ON_CLOSE ),
>    DELAYED_ACQUISITION_AND_RELEASE_AFTER_STATEMENT( AS_NEEDED, AFTER_STATEMENT ),
>    DELAYED_ACQUISITION_AND_RELEASE_BEFORE_TRANSACTION_COMPLETION( AS_NEEDED, BEFORE_TRANSACTION_COMPLETION ),
>    DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION( AS_NEEDED, AFTER_TRANSACTION )
>    ;
>    private final ConnectionAcquisitionMode acquisitionMode;
>    private final ConnectionReleaseMode releaseMode;
>    PhysicalConnectionHandlingMode(
>       ConnectionAcquisitionMode acquisitionMode,
>       ConnectionReleaseMode releaseMode) {
>       this.acquisitionMode = acquisitionMode;
>       this.releaseMode = releaseMode;
>    }
> ......//不重要代码先省略}
> ```
>
> 我们可以看到一共有五组值，也就是把原来的 ConnectionAcquisitionMode 和 ConnectionReleaseMode 分开配置的模式进行了组合配置管理，我们分别了解一下。
>
> **IMMEDIATE_ACQUISITION_AND_HOLD：立即获取，一直保持连接到 Session 关闭。** 其可以代表如下几层含义：
>
> - Session 一旦打开就会获取连接；
> - Session 关闭的时候释放连接；
> - 如果 open-in-view=true 的时候，也就是说即使我们的请求里面没有做任何操作，或者有一些耗时操作，会导致数据库的连接释放不及时，从而导致 DB 连接不够用，如果请求频繁的话，会产生不必要的 DB 连接的上下文切换，浪费 CPU 性能；
> - 容易产生 DB 连接获取时间过长的现象，从而导致请求响应时间变长。
>
> **DELAYED_ACQUISITION_AND_HOLD：延迟获取，一直保持连接到 Session 关闭。** 其可以代表如下几层含义：
>
> - 表示需要的时候再获取连接，需要的时候是指进行 DB 操作的时候，这里主要是指事务打开的时候，就需要获取连接了（因为开启事务的时候要执行“AUTOCOMMIT=0”的操作，所以这里的按需就是指开启事务；我们也可以关闭事务开启的时候改变 AUTOCOMMIT 的行为，那么这个时候的按需就是指执行 DB 操作的时候，不一定开启事务就会获得 DB 的连接）；
> - 关闭连接的时机是 Session Colse 的时候；
> - 一个 Session 里面只有一个连接，而一个连接里面可以有多段事务；比较适合一个请求有多段事务的场景；
> - 这个配置解决了，当没有 DB 操作的时候，即没有事务的时候不会获取数据库连接的问题；从而可以减少不必要的 DB 连接切换；
> - 但是一旦一个 Session 在进行了 DB 操作之后，又做了一些耗时的操作才关闭，那么也会导致 DB 连接释放不及时，从而导致 DB 连接的利用率低、高并发的时候请求性能下降。
>
> **DELAYED_ACQUISITION_AND_RELEASE_AFTER_STATEMENT：延迟获取，Statement 执行完释放。** 其可以代表如下几层含义：
>
> - 表示等需要的时候再获取连接，不是 session 一打开就会获取连接；
> - 在每个 Statement 的 SQL 执行完就释放连接，一旦有事务每个 SQL 执行完释放满足不了业务逻辑，我们常用的事务模式就不生效了；
> - 这种方式适合没有事务的情景，工作中不常见，可能分布式事务中有场景需要。
>
> **DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION：延迟获取，事务执行之后释放。** 其可以代表如下几层含义：
>
> - 表示等需要的时候再获取连接，不是 Session 一打开就会获取连接；
> - 在事务执行完之后释放连接，同一个事务共享一个连接；
> - 这种情况下 open-in-view 的模式对 DB 连接的持有和事务一样了，比较适合一个请求里面事务模块不多请求的情况；
> - 如果事务都控制在 Service 层，这个配置就非常好用，其对 Connection 的利用率比较高，基本上可以做到不浪费；
> - 这个配置不适合一个 Session 生命周期里面有很多独立事务的业务模块，因为这样就会使一个请求里面产生大量没必要的获取连接、释放连接的过程。
>
> **DELAYED_ACQUISITION_AND_RELEASE_BEFORE_TRANSACTION_COMPLETION：延迟获取，事务执行之前释放。** 其可以代表如下几层含义：
>
> - 表示等需要的时候再获取连接，不是 Session 一打开就会获取连接；
> - 在事务执行完之前释放连接，这种不保险，也比较少用。
>
> 现在你知道了 handling_mode 的五种模式，那么通常会默认用哪一种呢？
>
> #### 默认的模式是哪个？如何修改默认值？
>
> 我们打开源码 HibernateJpaVendorAdapter 类里面可以看到如下加载方式。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/71/71/CgqCHl--KCuAcIttAANGE2zTL8g522.png)
>
> Hibernate 5.2 以上使用的是 DELAYED_ACQUISITION_AND_HOLD 模式，即按需获取、Session 关闭释放，如下面这段代码。
>
> 复制代码
>
> ```
> jpaProperties.put("hibernate.connection.handling_mode", "DELAYED_ACQUISITION_AND_HOLD");
> ```
>
> 而 Hibernate 5.1 以前是通过设置 release_mode 等于 ON_CLOSE 的方式，也是 Session 关闭释放，如下面这段代码。
>
> 复制代码
>
> ```
> jpaProperties.put("hibernate.connection.release_mode", "ON_CLOSE");
> ```
>
> 那么，如何修改默认值呢？直接在 application.properties 文件里面做如下修改即可。
>
> 复制代码
>
> ```
> ## 我们可以修改成按需获取连接，事务执行完之后释放连接
> spring.jpa.properties.hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
> ```
>
> 说了这么多，我们通过日志来看一下常用的两个配置对数据库连接的影响是什么？
>
> #### handling_mode 的配置对连接的影响
>
> 第一步：验证一下 DELAYED_ACQUISITION_AND_HOLD，即默认情况下，连接池的情况是什么样的？
>
> 我们对配置文件做如下配置。
>
> 复制代码
>
> ```
> ## 在拦截MVC层开启Session，模拟默认情况，这条可以不需要配置，我只是为了给你演示得清晰一点
> spring.jpa.open-in-view=true
> ## 采用默认情况DELAYED_ACQUISITION_AND_HOLD，这条也不需要配置，我只是为了演示得清晰一点
> spring.jpa.properties.hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_HOLD
> ## 开启hikair的数据库连接池的监控：
> logging.level.com.zaxxer.hikari=TRACE
> ```
>
> 在 UserInfoController 的如下方法里面，通过 Thread.sleep（2 分钟）模拟耗时操作，代码如下。
>
> 复制代码
>
> ```
> @PostMapping("/user/info")
> public UserInfo saveUserInfo(@RequestBody UserInfo userInfo) throws InterruptedException {
>    UserInfo u2 = userInfoRepository.findById(1L).orElse(null);
>    if (u2!=null) {
>       u2.setLastName("jack"+userInfo.getLastModifiedTime());
>       userInfoRepository.save(u2);
>       System.out.println("模拟事务执行完之后耗时操作........");
>       Thread.sleep(1000*60*2L);
>       System.out.println("耗时操作执行完毕.......");
>    }
>    return userInfoRepository.save(userInfo);
> }
> ```
>
> 项目启动，我们做如下请求。
>
> 复制代码
>
> ```
> #### update
> POST /user/info HTTP/1.1
> Host: 127.0.0.1:8087
> Content-Type: application/json
> Cache-Control: no-cache
> {"ages":10,"id":3,"version":0}
> ```
>
> 这个时候打开日志控制台，可以看到如下日志。
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/71/66/Ciqc1F--KEGAOpmsAAFjWvgQG2M712.png)
>
> 可以看到，我们在 save 之后，即事务提交之后，HikariPool 里面的数据库连接一直没有归还，而如果我们继续等待的话，在整个 Session 关闭之后，数据库连接才会归还到连接池里面。
>
> 试想一下，如果我们实际工作中有这样的耗时操作，是不是用不了几个这样的请求，连接池就不够用了？但其实数据库连接没做任何 DB 相关的操作，白白被浪费了。
>
> 第二步：验证一下 DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION 模式。
>
> 我们只需要对配置文件做如下修改。
>
> 复制代码
>
> ```
> spring.jpa.properties.hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
> ```
>
> 其他代码都不变，我们再请求刚才的 API 请求，这个时候可以得到如下日志。
>
> ![Drawing 9.png](https://s0.lgstatic.com/i/image/M00/71/72/CgqCHl--KFqAEpxiAAGZnguqMBM395.png)
>
> 从日志中可以看到，当我们执行完 save(u2)，事务提交之后，做一些耗时操作的时候，发现此时整个 Session 生命周期是没有持有数据库连接的，也就是事务结束之后就进行了释放，这样大大提高了数据库连接的利用率，即使大量请求也不会造成数据库连接不够用。
>
> 下面是我的一些 Hrkari 数据源连接池下， DB 连接获得的时间参考值。
>
> 其中，对连接的池的持有情况如下图所示，这是正常情况，几乎监控不到 DB 连接不够用的情况。
>
> ![Drawing 10.png](https://s0.lgstatic.com/i/image/M00/71/72/CgqCHl--KGKAUrM_AAGuA1rvXks679.png)
>
> 对 DB 连接利用率的监控，如下图所示，连接的 Creation、Acquire 基本上是正常的，但是连接的 Usage>500ms 就有些不正常了，说明里面有一些耗时操作。
>
> ![Drawing 11.png](https://s0.lgstatic.com/i/image/M00/71/66/Ciqc1F--KGiAQnscAAFqjef-KBY275.png)
>
> 所以，一般在实际工作中，我们会在 DELAYED_ACQUISITION_AND_HOLD 和 DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION 之间做选择；通过日志和监控，我们也可以看得出来 DELAYED_ACQUISITION_AND_HOLD 比较适合一个 Session 里面有大量事务的业务场景，这样不用频繁切换数据库连接。
>
> 而 DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION 比较适合日常的 API 业务请求，没有大量的事务，事务结束就释放连接的场景。
>
> 下面再结合我们前几讲的基础知识，总结一下 Session 需要关心的关键关系有哪些。
>
> ### Session、EntityManager、Connection 和 Transaction 的关系
>
> #### Connection 和 Transaction 的关系
>
> 1. 事务是建立在 Connection 之上的，没有连接就没有事务。
> 2. 以 MySQL InnoDB 为例，新开一个连接默认开启事务，默认每个 SQL 执行完之后自动提交事务。
> 3. 一个连接里面可以有多次串行的事务段；一个事务只能属于一个 Connection。
> 4. 事务与事务之间是相互隔离的，那么自然不同连接的不同事务也是隔离的。
>
> #### EntityManager、Connection 和 Transaction 的关系
>
> 1. EntityManager 里面有 DataSource，当 EntityManager 里面开启事务的时候，先判断当前线程里面是否有数据库连接，如果有直接用。
> 2. 开启事务之前先开启连接；关闭事务，不一定关闭连接。
> 3. 开启 EntityManager，不一定立马获得连接；获得连接，不一定立马开启事务。
> 4. 关闭 EntityManager，一定关闭事务，释放连接；反之不然。
>
> #### Session、EntityManager、Connection 和 Transaction 的关系
>
> 1. Session 是 EntityManager 的子类，SessionImpl 是 Session 和 EntityManager 的实现类。那么自然 EntityManager 和 Connection、Transaction 的关系同样适用 Session、EntityManager、Connection 和 Transaction 的关系。
> 2. Session 的生命周期决定了 EntityManager 的生命周期。
>
> #### Session 和 Transaction 的关系
>
> 1. 在 Hibernate 的 JPA 实现里面，开启 Transaction 之前，必须要先开启 Session。
> 2. 默认情况下，Session 的生命周期由 open-in-view 决定是请求之前开启，还是事务之前开启。
> 3. 事务关闭了，Session 不一定关闭。
> 4. Session 关闭了，事务一定关闭。
>
> ### 总结
>
> 以上就是这一讲的内容了。本讲中我们通过源码分析了 spring.jpa.open-in-view 是什么、干什么用的，以及它对事务、连接池、EntityManager 和 Session 的影响。
>
> 到这一讲你应该已经掌握了 Spring Data JPA 的核心的原理里面最重要的五个时机，即 Session（Entity Manager）的 Open 和 Close 时机、数据库连接的获取和释放时机、事务的开启和关闭时机以及我们上一讲介绍的 Persistence Context 的创建和销毁时机、Flush 的触发时机。希望你可以好好掌握并牢记我在其中提到的要点。
>
> 那么 open-in-view 对 lazy 的影响是什么呢？我们将在第 24 讲详细介绍。而下一讲我会通过一个实际案例，和你一起通过原理分析一些疑难杂症。
>
> 关于每一讲的内容，希望你可以提出一些自己的看法，在下方留言，让志同道合之士一起讨论，共同成长。再见。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa