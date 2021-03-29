[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4728)



> 今天我们来聊聊二级缓存相关的话题。
>
> 我们在使用 Mybatis 的时候，基本不用关心什么是二级缓存。而如果你是 Hibernate 的使用者，一定经常听说和使用过 Hibernate 的二级缓存，那么我们应该怎么看待它呢？这一讲一起来揭晓 Cache 的相关概念以及在生产环境中的最佳实践。
>
> ### 二级缓存的概念
>
> 上一讲我们介绍了一级缓存相关的内容，一级缓存的实体的生命周期和 PersistenceContext 是相同的，即载体为同一个 Session 才有效；而 Hibernate 提出了二级缓存的概念，也就是可以在不同的 Session 之间共享实体实例，说白了就是在单个应用内的整个 application 生命周期之内共享实体，减少数据库查询。
>
> 由于 JPA 协议本身并没有规定二级缓存的概念，所以这是 Hiberante 独有的特性。所以在 Hibernate 中，从数据库里面查询实体的过程就变成了：第一步先看看一级缓存里面有没有实体，如果没有再看看二级缓存里面有没有，如果还是没有再从数据库里面查询。那么在 Hibernate 的环境下如何开启二级缓存呢？
>
> #### Hibernate 中二级缓存的配置方法
>
> Hibernate 中，默认情况下二级缓存是关闭的，如果想开启二级缓存需要通过如下三个步骤。
>
> **第一步：引入第三方二级缓存的实现的 jar**。
>
> 因为 Hibernate 本身并没有实现缓存的功能，而是主要依赖第三方，如 Ehcache、jcache、redis 等第三方库。下面我们以 EhCache 为例，利用 gradle 引入 hibernate-ehcace 的依赖。代码如下所示。
>
> 复制代码
>
> ```
> implementation 'org.hibernate:hibernate-ehcache:5.2.2.Final'
> ```
>
> 如果我们想用 jcache，可以通过如下方式。
>
> 复制代码
>
> ```
> compile 'org.hibernate:hibernate-jcache:5.2.2.Final'
> ```
>
> **第二步：在配置文件里面开启二级缓存**。
>
> 二级缓存默认是关闭的，所以需要我们用如下方式开启二级缓存，并且配置 cache.region.factory_class 为不同的缓存实现类。
>
> 复制代码
>
> ```
> hibernate.cache.use_second_level_cache=true
> hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
> ```
>
> **第三步：在用到二级缓存的地方配置 @Cacheable 和 @Cache 的策略**。
>
> 复制代码
>
> ```
> import javax.persistence.Cacheable;
> import javax.persistence.Entity;
> @Entity
> @Cacheable
> @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
> public class UserInfo extends BaseEntity {......}
> ```
>
> 通过以上三步就可以轻松实现二级缓存了，但是这时请你思考一下，这真的能应用到我们实际生产环境中吗？会不会有副作用？
>
> #### 二级缓存的思考
>
> 二级缓存主要解决的是单应用场景下跨 Session 生命周期的实体共享问题，可是我们一定要通过 Hibernate 来做吗？答案并不是，其实我们可以通过各种 Cache 的手段来做，因为 Hibernate 里面一级缓存的复杂度相对较高，并且使用的话实体的生命周期会有变化，查询问题的过程较为麻烦。
>
> 同时，随着现在逐渐微服务化、分布式化，如今的应用都不是单机应用，那么缓存之间如何共享呢？分布式缓存又该如何解决？比如一个机器变了，另一个机器没变，应该如何处理？似乎 Hiberante 并没有考虑到这些问题。
>
> 此外，还有什么时间数据会变更、变化了之后如何清除缓存，等等，这些都是我们要思考的，所以 Hibernate 的二级缓存听起来“高大上”，但是使用起来绝对没有那么简单。
>
> 那么经过这一连串的疑问，如果我们不用 Hibernate 的二级缓存，还有没有更好的解决方案呢？
>
> ### 利用 Redis 进行缓存
>
> 在我们实际工作中经常需要 cache 的就是 Redis，那么我们通过一个例子，来看下 Spring Cache 结合 Redis 是怎么使用的。
>
> #### Spring Cache 和 Redis 结合
>
> 第一步：在 gradle 中引入 cache 和 redis 的依赖，代码如下所示。
>
> 复制代码
>
> ```
> //原来我们只用到了JPA
> implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
> //为了引入cache和redis机制需要引入如下两个jar包
> implementation 'org.springframework.boot:spring-boot-starter-data-redis' //redis的依赖
> implementation 'org.springframework.boot:spring-boot-starter-cache' //cache 的依赖
> ```
>
> 第二步：在 application.properties 里面增加 redis 的相关配置，代码如下。
>
> 复制代码
>
> ```
> spring.redis.host=127.0.0.1
> spring.redis.port=6379
> spring.redis.password=sySj6vmYke
> spring.redis.timeout=6000
> spring.redis.pool.max-active=8
> spring.redis.pool.max-idle=8
> spring.redis.pool.max-wait=-1
> spring.redis.pool.min-idle=0
> ```
>
> 第三步：通过 @EnableCaching 开启缓存，增加 configuration 配置类，代码如下所示。
>
> 复制代码
>
> ```
> @EnableCaching
> @Configuration
> public class CacheConfiguration {
> }
> ```
>
> 第四步：在我们需要缓存的地方添加 @Cacheable 注解即可。为了方便演示，我把 @Cacheable 注解配置在了 controller 方法上，代码如下。
>
> 复制代码
>
> ```
> @GetMapping("/user/info/{id}")
> @Cacheable(value = "userInfo", key = "{#root.methodName, #id}", unless = "#result == null") //利用默认key值生成规则value加key生成一个redis的key值，result==null的时候不进行缓存
> public UserInfo getUserInfo(@PathVariable("id") Long id) {
>    //第二次就不会再执行这里了
>    return userInfoRepository.findById(id).get();
> }
> ```
>
> 第五步：启动项目，请求一下这个 API 会发现，第一次请求过后，redis 里面就有一条记录了，如下图所示。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/03/A1/CgpVE1_gGfGAAYRDAAGxaGz4d8A668.png)
>
> 可以看到，第二次请求之后，取数据就不会再请求数据库了。那么 redis 我们已经熟悉了，那么来看一下 Spring Cache 都做了哪些事情。
>
> #### Spring Cache 介绍
>
> Spring 3.1 之后引入了基于注释（annotation）的缓存（cache）技术，它本质上不是一个具体的缓存实现方案（例如 EHCache 或者 Redis），而是一个对缓存使用的抽象概念，通过在既有代码中添加少量它定义的各种 annotation，就能够达到缓存方法的返回对象的效果。
>
> Spring 的缓存技术还具备相当的灵活性，不仅能够使用 SpEL（Spring Expression Language）来定义缓存的 key 和各种 condition，还提供开箱即用的缓存临时存储方案，也支持主流的专业缓存，例如 Redis，EHCache 集成。而 Spring Cache 属于 Spring framework 的一部分，在下面图片所示的这个包里面。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/03/9F/Cip5yF_gGfmAfvKbAAGFdPiyN9k521.png)
>
> **Spring cache 里面的主要的注解**
>
> **@Cacheable**
>
> 应用到读取数据的方法上，就是可以缓存的方法，如查找方法：先从缓存中读取，如果没有再调用方法获取数据，然后把数据添加到缓存中。
>
> 复制代码
>
> ```
> public @interface Cacheable {
>    @AliasFor("cacheNames")
>    String[] value() default {};
> //cache的名字。可以根据名字设置不同cache处理类。redis里面可以根据cache名字设置不同的失效时间。
>    @AliasFor("value")
>    String[] cacheNames() default {};
> //缓存的key的名字，支持spel
>    String key() default "";
> //key的生成策略，不指定可以用全局的默认的。
>    String keyGenerator() default "";
>    //客户选择不同的CacheManager
>    String cacheManager() default "";
>    //配置不同的cache resolver
>    String cacheResolver() default "";
>    //满足什么样的条件才能被缓存，支持SpEL，可以去掉方法名、参数
>    String condition() default "";
> //排除哪些返回结果不加入缓存里面去，支持SpEL，实际工作中常见的是result ==null等
>    String unless() default "";
>    //是否同步读取缓存、更新缓存
>    boolean sync() default false;
> }
> ```
>
> 下面是@Cacheable 相关的例子。
>
> 复制代码
>
> ```
> @Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result.notNeedCache")//利用SPEL表达式只有当name参数长度小于32的时候再进行缓存，排除notNeedCache的对象
> public Book findBook(String name)
> ```
>
> **@CachePut**
>
> 调用方法时会自动把相应的数据放入缓存，它与 @Cacheable 不同的是所有注解的方法每次都会执行，一般配置在 Update 和 insert 方法上。其源码里面的字段和用法基本与 @Cacheable 相同，只是使用场景不一样，我就不详细介绍了。
>
> **@CacheEvict**
>
> 删除缓存，一般配置在删除方法上面。代码如下所示。
>
> 复制代码
>
> ```
> public @interface CacheEvict {
> //与@Cacheable相同的部分咱我就不重复叙述了。
> ......
> 	//是否删除所有的实体对象
>    boolean allEntries() default false;
>    //是否方法执行之前执行。默认在方法调用成功之后删除
>    boolean beforeInvocation() default false;
> }
> 	@Caching 所有Cache注解的组合配置方法，源码如下：
> 	public @interface Caching {
>    Cacheable[] cacheable() default {};
>    CachePut[] put() default {};
>    CacheEvict[] evict() default {};
> }
> ```
>
> 此外，还有 @CacheConfig 表示全局 Cache 配置；@EnableCaching，表示是否开启 SpringCache 的配置。
>
> 以上是 SpringCache 中常见的注解，下面我们再来看 Spring Cache Redis 里面主要的类都有哪些。
>
> **Spring Cache Redis 里面主要的类**
>
> **org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration**
>
> cache 的自动装配类，此类被加载的方式是在 spring boot的spring.factories 文件里面，其关键源码如下所示。
>
> 复制代码
>
> ```
> @Configuration(proxyBeanMethods = false)
> @ConditionalOnClass(CacheManager.class)
> @ConditionalOnBean(CacheAspectSupport.class)
> @ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
> @EnableConfigurationProperties(CacheProperties.class)
> @AutoConfigureAfter({ CouchbaseDataAutoConfiguration.class, HazelcastAutoConfiguration.class,
>       HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
> @Import({ CacheConfigurationImportSelector.class, CacheManagerEntityManagerFactoryDependsOnPostProcessor.class })
> public class CacheAutoConfiguration {
>   /**
>    * {@link ImportSelector} to add {@link CacheType} configuration classes.
>    */
>   static class CacheConfigurationImportSelector implements ImportSelector {
>      @Override
>      public String[] selectImports(AnnotationMetadata importingClassMetadata) {
>         CacheType[] types = CacheType.values();
>         String[] imports = new String[types.length];
>         for (int i = 0; i < types.length; i++) {
>            imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
>         }
>         return imports;
>      }
>   }
> }
> ```
>
> 通过源码可以看到，此类的关键作用是加载 Cache 的依赖配置，以及加载所有 CacheType 的配置文件，而 CacheConfigurations 里面定义了不同的 Cache 实现方式的配置，里面包含了 Ehcache、Redis、Jcache 的各种实现方式，如下图所示。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image2/M01/03/9F/Cip5yF_gGgWAPkIYAAD6KrApZRs929.png)
>
> **org.springframework.cache.annotation.CachingConfigurerSupport**
>
> 通过此类可以自定义 Cache 里面的 CacheManager、CacheResolver、KeyGenerator、CacheErrorHandler，代码如下所示。
>
> 复制代码
>
> ```
> public class CachingConfigurerSupport implements CachingConfigurer {
>   // cache的manager，主要是管理不同的cache的实现方式，如redis还是ehcache等
>    @Override
>    @Nullable
>    public CacheManager cacheManager() {
>       return null;
>    }
>    // cache的不同实现者的操作方法，CacheResolver解析器，用于根据实际情况来动态解析使用哪个Cache
>    @Override
>    @Nullable
>    public CacheResolver cacheResolver() {
>       return null;
>    }
>    //cache的key的生成规则
>    @Override
>    @Nullable
>    public KeyGenerator keyGenerator() {
>       return null;
>    }
>    //cache发生异常的回调处理，一般情况下我会打印个warn日志，方便知道发生了什么事情
>    @Override
>    @Nullable
>    public CacheErrorHandler errorHandler() {
>       return null;
>    }
> }
> ```
>
> 其中，所有 CacheManager 是 Spring 提供的各种缓存技术抽象接口，通过它来管理，Spring framework 里面默认实现的 CacheManager 有不同的实现类，redis 默认加载的是 RedisCacheManager，如下图所示。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/03/9F/Cip5yF_gGgyAJ21FAADMLk_C6ag487.png)
>
> **org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration**
>
> 它是加载 Cache 的实现者，也是 redis 的实现类，关键源码如下图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/8B/BE/Ciqc1F_gGhKAQGL2AAIaDnIIDQQ607.png)
>
> 我们可以看得出来，它依赖本身的 Redis 的连接，并且加载了 RedisCacheManager；同时可以看到关于 Cache 和 Redis 的配置有哪些。
>
> 通过 CacheProperties 里面 redis 的配置，我们可以设置“key 的统一前缀、默认过期时间、是否缓存 null 值、是否使用前缀”这四个配置。
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/8B/C9/CgqCHl_gGhqAf6niAACSWTdYSNc168.png)
>
> 通过这几个主要的类，相信你已经对 Spring Cache 有了简单的了解，下面我们看一下在实际工作中有哪些最佳实践可以提供参考。
>
> ### Spring Cache 结合 Redis 使用的最佳实践
>
> **不同 cache 的 name 在 redis 里面配置不同的过期时间**
>
> 默认情况下所有 redis 的 cache 过期时间是一样的，实际工作中一般需要自定义不同 cache 的 name 的过期时间，我们这里 cache 的 name 就是指 @Cacheable 里面 value 属性对应的值。主要步骤如下。
>
> 第一步：自定义一个配置文件，用来指定不同的 cacheName 对应的过期时间不一样。代码如下所示。
>
> 复制代码
>
> ```
> @Getter
> @Setter
> @ConfigurationProperties(prefix = "spring.cache.redis")
> /**
>  * 改善一下cacheName的最佳实践方法，目前主要用不同的cache name不同的过期时间，可以扩展
>  */
> public class MyCacheProperties {
>     private HashMap<String, Duration> cacheNameConfig;
> }
> ```
>
> 第二步：通过自定义类 MyRedisCacheManagerBuilderCustomizer 实现 RedisCacheManagerBuilderCustomizer 里面的 customize 方法，用来指定不同的 name 采用不同的 RedisCacheConfiguration，从而达到设置不同的过期时间的效果。代码如下所示。
>
> 复制代码
>
> ```
> /**
>  * 这个依赖spring boot 2.2 以上版本才有效
>  */
> public class MyRedisCacheManagerBuilderCustomizer implements RedisCacheManagerBuilderCustomizer {
>     private MyCacheProperties myCacheProperties;
>     private RedisCacheConfiguration redisCacheConfiguration;
>     public MyRedisCacheManagerBuilderCustomizer(MyCacheProperties myCacheProperties, RedisCacheConfiguration redisCacheConfiguration) {
>         this.myCacheProperties = myCacheProperties;
>         this.redisCacheConfiguration = redisCacheConfiguration;
>     }
>     /**
>      * 利用默认配置的只需要在这里加就可以了
>      * spring.cache.cache-names=abc,def,userlist2,user3
>      * 下面是不同的cache-name可以配置不同的过期时间，yaml也支持，如果以后还有其他属性扩展可以改这里
>      * spring.cache.redis.cache-name-config.user2=2h
>      * spring.cache.redis.cache-name-config.def=2m
>      * @param builder
>      */
>     @Override
>     public void customize(RedisCacheManager.RedisCacheManagerBuilder builder) {
>         if (ObjectUtils.isEmpty(myCacheProperties.getCacheNameConfig())) {
>             return;
>         }
>         Map<String, RedisCacheConfiguration> cacheConfigurations = myCacheProperties.getCacheNameConfig().entrySet().stream()
>                 .collect(Collectors
>                         .toMap(e->e.getKey(),v->builder
>                                 .getCacheConfigurationFor(v.getKey())
>                                 .orElse(RedisCacheConfiguration.defaultCacheConfig().serializeValuesWith(redisCacheConfiguration.getValueSerializationPair()))
>                                 .entryTtl(v.getValue())));
>         builder.withInitialCacheConfigurations(cacheConfigurations);
>     }
> }
> ```
>
> 第三步：在 CacheConfiguation 里面把我们自定义的 CacheManagerCustomize 加载进去即可，代码如下。
>
> 复制代码
>
> ```
> @EnableCaching
> @Configuration
> @EnableConfigurationProperties(value = {MyCacheProperties.class,CacheProperties.class})
> @AutoConfigureAfter({CacheAutoConfiguration.class})
> public class CacheConfiguration {
>     /**
>      * 支持不同的cache name有不同的缓存时间的配置
>      *
>      * @param myCacheProperties
>      * @param redisCacheConfiguration
>      * @return
>      */
>     @Bean
>     @ConditionalOnMissingBean(name = "myRedisCacheManagerBuilderCustomizer")
>     @ConditionalOnClass(RedisCacheManagerBuilderCustomizer.class)
>     public MyRedisCacheManagerBuilderCustomizer myRedisCacheManagerBuilderCustomizer(MyCacheProperties myCacheProperties, RedisCacheConfiguration redisCacheConfiguration) {
>         return new MyRedisCacheManagerBuilderCustomizer(myCacheProperties,redisCacheConfiguration);
>     }
> }
> ```
>
> 第四步：使用的时候非常简单，只需要在 application.properties 里面做如下配置即可。
>
> 复制代码
>
> ```
> # 设置默认的过期时间是20分钟
> spring.cache.redis.time-to-live=20m
> # 设置我们刚才的例子 @Cacheable(value="userInfo")5分钟过期
> spring.cache.redis.cache-name-config.userInfo=5m
> # 设置 room的cache1小时过期
> spring.cache.redis.cache-name-config.room=1h
> ```
>
> **自定义 KeyGenerator 实现，redis 的 key 自定义拼接规则**
>
> 假如我们不喜欢默认的 cache 生成的 key 的 string 规则，那么可以自定义。我们创建 MyRedisCachingConfigurerSupport 集成 CachingConfigurerSupport 即可，代码如下。
>
> 复制代码
>
> ```
> @Component
> @Log4j2
> public class MyRedisCachingConfigurerSupport extends CachingConfigurerSupport {
>     @Override
>     public KeyGenerator keyGenerator() {
>         return getKeyGenerator();
>     }
>     /**
>      * 覆盖默认的redis key的生成规则，变成"方法名:参数:参数"
>      * @return
>      */
>     public static KeyGenerator getKeyGenerator() {
>         return (target, method, params) -> {
>             StringBuilder key = new StringBuilder();
>             key.append(ClassUtils.getQualifiedMethodName(method));
>             for (Object obc : params) {
>                 key.append(":").append(obc);
>             }
>             return key.toString();
>         };
>     }
> }
> ```
>
> **当发生 cache 和 redis 的操作异常时，我们不希望阻碍主流程，打印一个关键日志即可**
>
> 只需要在 MyRedisCachingConfigurerSupport 里面再实现父类的 errorHandler 即可，代码变成了如下模样。
>
> 复制代码
>
> ```
> @Log4j2
> public class MyRedisCachingConfigurerSupport extends CachingConfigurerSupport {
>     @Override
>     public KeyGenerator keyGenerator() {
>         return getKeyGenerator();
>     }
>     /**
>      * 覆盖默认的redis key的生成规则，变成"方法名:参数:参数"
>      * @return
>      */
>     public static KeyGenerator getKeyGenerator() {
>         return (target, method, params) -> {
>             StringBuilder key = new StringBuilder();
>             key.append(ClassUtils.getQualifiedMethodName(method));
>             for (Object obc : params) {
>                 key.append(":").append(obc);
>             }
>             return key.toString();
>         };
>     }
>     /**
>      * 覆盖默认异常处理方法，不抛异常，改打印error日志
>      *
>      * @return
>      */
>     @Override
>     public CacheErrorHandler errorHandler() {
>         return new CacheErrorHandler() {
>             @Override
>             public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
>                 log.error(String.format("Spring cache GET error:cache=%s,key=%s", cache, key), exception);
>             }
>             @Override
>             public void handleCachePutError(RuntimeException exception, Cache cache, Object key, Object value) {
>                 log.error(String.format("Spring cache PUT error:cache=%s,key=%s", cache, key), exception);
>             }
>             @Override
>             public void handleCacheEvictError(RuntimeException exception, Cache cache, Object key) {
>                 log.error(String.format("Spring cache EVICT error:cache=%s,key=%s", cache, key), exception);
>             }
>             @Override
>             public void handleCacheClearError(RuntimeException exception, Cache cache) {
>                 log.error(String.format("Spring cache CLEAR error:cache=%s", cache), exception);
>             }
>         };
>     }
> }
> ```
>
> **改变默认的 cache 里面 redis 的 value 序列化方式**
>
> 默认有可能是 JDK 序列化方式，所以一般我们看不懂 redis 里面的值，那么就可以把序列化方式改成 JSON 格式，只需要在 CacheConfiguration 里面增加默认的 RedisCacheConfiguration 配置即可，完整的 CacheConfiguration 变成如下代码所示的样子。
>
> 复制代码
>
> ```
> @EnableCaching
> @Configuration
> @EnableConfigurationProperties(value = {MyCacheProperties.class,CacheProperties.class})
> @AutoConfigureAfter({CacheAutoConfiguration.class})
> public class CacheConfiguration {
>     /**
>      * 支持不同的cache name有不同的缓存时间的配置
>      *
>      * @param myCacheProperties
>      * @param redisCacheConfiguration
>      * @return
>      */
>     @Bean
>     @ConditionalOnMissingBean(name = "myRedisCacheManagerBuilderCustomizer")
>     @ConditionalOnClass(RedisCacheManagerBuilderCustomizer.class)
>     public MyRedisCacheManagerBuilderCustomizer myRedisCacheManagerBuilderCustomizer(MyCacheProperties myCacheProperties, RedisCacheConfiguration redisCacheConfiguration) {
>         return new MyRedisCacheManagerBuilderCustomizer(myCacheProperties,redisCacheConfiguration);
>     }
>     /**
>      * cache异常不抛异常，只打印error日志
>      *
>      * @return
>      */
>     @Bean
>     @ConditionalOnMissingBean(name = "myRedisCachingConfigurerSupport")
>     public MyRedisCachingConfigurerSupport myRedisCachingConfigurerSupport() {
>         return new MyRedisCachingConfigurerSupport();
>     }
>     /**
>      * 依赖默认的ObjectMapper，实现普通的json序列化
>      * @param defaultObjectMapper
>      * @return
>      */
>     @Bean(name = "genericJackson2JsonRedisSerializer")
>     @ConditionalOnMissingBean(name = "genericJackson2JsonRedisSerializer")
>     public GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer(ObjectMapper defaultObjectMapper) {
>         ObjectMapper objectMapper = defaultObjectMapper.copy();
>         objectMapper.registerModule(new Hibernate5Module().enable(REPLACE_PERSISTENT_COLLECTIONS)); //支持JPA的实体的json的序列化
>         objectMapper.configure(MapperFeature.SORT_PROPERTIES_ALPHABETICALLY, true);//培训
>         objectMapper.deactivateDefaultTyping(); //关闭 defaultType，不需要关心reids里面是否为对象的类型
>         return new GenericJackson2JsonRedisSerializer(objectMapper);
>     }
>     /**
>      * 覆盖 RedisCacheConfiguration，只是修改serializeValues with jackson
>      *
>      * @param cacheProperties
>      * @return
>      */
>     @Bean
>     @ConditionalOnMissingBean(name = "jacksonRedisCacheConfiguration")
>     public RedisCacheConfiguration jacksonRedisCacheConfiguration(CacheProperties cacheProperties,
>                                                                   GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer) {
>         CacheProperties.Redis redisProperties = cacheProperties.getRedis();
>         RedisCacheConfiguration config = RedisCacheConfiguration
>                 .defaultCacheConfig();
>         config = config.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(genericJackson2JsonRedisSerializer));//修改的关键所在，指定Jackson2JsonRedisSerializer的方式
>         if (redisProperties.getTimeToLive() != null) {
>             config = config.entryTtl(redisProperties.getTimeToLive());
>         }
>         if (redisProperties.getKeyPrefix() != null) {
>             config = config.prefixCacheNameWith(redisProperties.getKeyPrefix());
>         }
>         if (!redisProperties.isCacheNullValues()) {
>             config = config.disableCachingNullValues();
>         }
>         if (!redisProperties.isUseKeyPrefix()) {
>             config = config.disableKeyPrefix();
>         }
>         return config;
>     }
> }
> ```
>
> ### 总结
>
> 以上就是本讲的内容了，这一讲的目的是帮助你打开思路，了解 Spring Data 的生态体系。那么由于篇幅有限，我介绍的 Cache、Redis、JPA 只是这三个项目里的冰山一角，你在实际工作中可以根据实际的应用场景，想想它们各自的职责是什么，让它们发挥各自的特长，而不是依赖于 Hibernate 功能的强大，为了用而去用，这样会让代码的可读性和复杂度提高很多，就会遇到各种各样的问题，导致觉得 Hibernate 太难，或者不可控。
>
> 其实大多数时候是我们的思路不对，其实万事万物皆有优势和劣势，我们要抛弃其劣势，充分利用各个框架的优势，发挥各自的特长。如果你觉得本专栏对你有帮助，就动动手指分享吧，下一讲我们来聊聊 Spring Data Rest 的相关话题，到时见。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa