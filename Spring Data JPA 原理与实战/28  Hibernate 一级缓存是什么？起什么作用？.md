[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4727)



> 如果你已经看完了之前的课时，相信你对 Hibernate 和 JPA 已经有一些深入的认识了，那么这一讲开始，我再对大家平时感到迷惑的概念做一下解释，帮助你更好地掌握 JPA。
>
> 这一讲我们来聊聊经常说的 Hibernate 的一级缓存是什么意思，Query Plan Cache 又和一级缓存是什么关系呢？
>
> ### 一级缓存
>
> 什么是一级缓存？这个大家最容易存在疑惑，不知道你是否也在工作中遇见过这些问题：没有办法取到最新的数据、不知道一级缓存该如何释放、怎样关闭一级缓存？我们又为什么要用一级缓存呢？
>
> #### 什么是一级缓存？
>
> 按照 Hibernate 和 JPA 协议里面的解释，我们经常说的 First Level Cache（一级缓存）也就是我在之前的课时中说过的 PersistenceContext，既然如此，那么就意味着一级缓存的载体是 Session 或者 EntityManager；而一级缓存的实体也就是数据库里面对应的实体。
>
> 在 SessionImpl 的实现过程中，我们会发现 PersistenceContext 的实现类 StatefulPersistenceContext 是通过 HashMap 来存储实体信息的，其关键源码如下所示。
>
> 复制代码
>
> ```
> public class StatefulPersistenceContext implements PersistenceContext {
>   //根据EntityUniqueKey作为key来储存Entity
>   private HashMap<EntityUniqueKey, Object> entitiesByUniqueKey;
>   //根据EntityUniqueKey作为key取当前实体
>   @Override
>   public Object getEntity(EntityUniqueKey euk) {
>      return entitiesByUniqueKey == null ? null : entitiesByUniqueKey.get( euk );
>   }
>   //储存实体，如果是第一次，那么创建HashMap<>
>   @Override
>   public void addEntity(EntityUniqueKey euk, Object entity) {
>      if ( entitiesByUniqueKey == null ) {
>         entitiesByUniqueKey = new HashMap<>( INIT_COLL_SIZE );
>      }
>      entitiesByUniqueKey.put( euk, entity );
>   }
> ......}
> ```
>
> 其中 EntityUniqueKey 的核心源码如下所示。
>
> 复制代码
>
> ```
> public class EntityUniqueKey implements Serializable {
>    private final String uniqueKeyName;
>    private final String entityName;
>    private final Object key;
>    private final Type keyType;
>    private final EntityMode entityMode;
>    private final int hashCode;
>   @Override
>   public boolean equals(Object other) {
>      EntityUniqueKey that = (EntityUniqueKey) other;
>      return that != null && that.entityName.equals( entityName )
>            && that.uniqueKeyName.equals( uniqueKeyName )
>            && keyType.isEqual( that.key, key );
>   }
> ...
> }
> ```
>
> 通过源码可以看到，用 PersistenceContext 来判断实体是不是同一个，可以直接根据实体里面的主键进行。那么一级缓存的作用是什么呢？
>
> #### 一级缓存的作用
>
> 由于一级缓存就是 PersistenceContext，那么一级缓存的最大作用就是管理 Entity 的生命周期，详细的内容我已经在“[21 | Persistence Context 所表达的核心概念是什么？](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490#/detail/pc?id=4721)”介绍过了，这里我就稍加总结。
>
> 1. New（Transient）状态的，不在一级缓存管理之列，这是新创建的；
> 2. Detached 游离状态的，不在一级缓存里面，和 New 的唯一区别是它带有主键和 Version 信息；
> 3. Manager、Removed 状态的实体在一级缓存管理之列，所有对这两种状态的实体进行的更新操作，都不会立即更新到数据库里面，只有执行了 flush 之后才会同步到数据库里面。
>
> 我们用一张图来表示，如下所示。
>
> ![image (2).png](https://s0.lgstatic.com/i/image/M00/8B/04/Ciqc1F_bHCyAUu5UAACWL1LulDM631.png)
>
> 注：图片来源于网络
>
> 对于实体 1 来说，新增和更新操作都是先进行一级缓存，只有 flush 的时候才会同步到数据库里面。而当我们执行了 entityManager.clean() 或者是 entityManager.detach(entity1)，那么实体 1 就会变成游离状态，这时再对实体 1 进行修改，如果再执行 flush 的话，就不会同步到 DB 里面了。我们用代码来说明一下，如下所示。
>
> 复制代码
>
> ```
> public class UserInfoRepositoryTest {
>     @Autowired
>     private UserInfoRepository userInfoRepository;
>     @PersistenceContext(properties = {@PersistenceProperty(
>             name = "org.hibernate.flushMode",
>             value = "MANUAL"//手动flush
>     )})
>     private EntityManager entityManager;
>     @Test
>     @Transactional
>     public void testLife() {
>         UserInfo userInfo = UserInfo.builder().name("new name").build();
>         //新增一个对象userInfo交给PersistenceContext管理，即一级缓存
>         entityManager.persist(userInfo);
>         //此时没有detach和clear之前，flush的时候还会产生更新SQL
>         userInfo.setName("old name");
>         entityManager.flush();
>         entityManager.clear();
> //        entityManager.detach(userInfo);
>         // entityManager已经clear，此时已经不会对UserInfo进行更新了
>         userInfo.setName("new name 11");
>         entityManager.flush();
>         //由于有cache机制，相同的对象查询只会触发一次查询SQL
>         UserInfo u1 = userInfoRepository.findById(1L).get();
>         //to do some thing
>         UserInfo u2 = userInfoRepository.findById(1L).get();
>     }
> }
> ```
>
> 利用我们之前讲过的打印日志的方法，把 SQL 打印一下，输出到控制台的 SQL 如下所示。
>
> 复制代码
>
> ```
> Hibernate: insert into user_info (create_time, create_user_id, last_modified_time, last_modified_user_id, version, ages, email_address, last_name, name, telephone, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
> Hibernate: update user_info set create_time=?, create_user_id=?, last_modified_time=?, last_modified_user_id=?, version=?, ages=?, email_address=?, last_name=?, name=?, telephone=? where id=? and version=?
> Hibernate: select userinfo0_.id as id1_2_0_, userinfo0_.create_time as create_t2_2_0_, userinfo0_.create_user_id as create_u3_2_0_, userinfo0_.last_modified_time as last_mod4_2_0_, userinfo0_.last_modified_user_id as last_mod5_2_0_, userinfo0_.version as version6_2_0_, userinfo0_.ages as ages7_2_0_, userinfo0_.email_address as email_ad8_2_0_, userinfo0_.last_name as last_nam9_2_0_, userinfo0_.name as name10_2_0_, userinfo0_.telephone as telepho11_2_0_, rooms1_.user_info_id as user_inf1_3_1_, room2_.id as rooms_id2_3_1_, room2_.id as id1_1_2_, room2_.create_time as create_t2_1_2_, room2_.create_user_id as create_u3_1_2_, room2_.last_modified_time as last_mod4_1_2_, room2_.last_modified_user_id as last_mod5_1_2_, room2_.version as version6_1_2_, room2_.title as title7_1_2_ from user_info userinfo0_ left outer join user_info_rooms rooms1_ on userinfo0_.id=rooms1_.user_info_id left outer join room room2_ on rooms1_.rooms_id=room2_.id where userinfo0_.id=?
> ```
>
> 通过日志可以看到没有第二次更新。
>
> 除此之外，关于一级缓存还有其他问题你应该了解一下。
>
> 它的生命周期是怎么样的呢？可想而知，肯定和 Session 一样，这个问题你可以回过头仔细看看“[22 | Session 的 open-in-view 对事务的影响是什么？](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490#/detail/pc?id=4722)”。但同时实体在一级 Cache 里面的生命周期还受到的 entityManager.clear() 和 entityManger.detach() 两个方法的影响。
>
> 一级缓存的大小可以设置吗？这个肯定是不能的，我们从底层原理可以分析出：一级缓存依赖Java 内存堆的大小，所以受到最大堆和最小堆的限制，即清除一级缓存的机制就是利用 JVM 的 GC 机制，清理掉 GC 就会清理掉一级缓存。
>
> 所以当我们请求并发量大的时候，Session 的对象就会变得很多，此时就会需要更多内存。当请求结束之后，随着 GC 的回收，里面就会清除一级缓存留下来的对象。
>
> 一级缓存可以关闭吗？答案肯定是不能的，除非我们不用 Hibernate 或 JPA，改用 Mybatis，因为一级缓存是 JPA 的最大优势之一。
>
> 而在实际工作中，最容易被我们忽略的是和一级缓存差不多的 Query Plan Cache，我们来了解一下。
>
> ### Query Plan Cache
>
> 我们都知道 JPA 里面大部分的查询都是基于 JPQL 查询语法，从而会有一个过程把 JPQL 转化成真正的 SQL，而后到数据库里执行。而 JPQL 转化成原始的 SQL 时，就会消耗一定的性能，所以 Hibernate 设计了一个 QueryPlanCache 的机制，用来存储 JPQL 或者 Criteria Query 到 Native SQL 中转化的结果，也就是说 QueryPlanCache 里面存储了最终要执行的 SQL，以及参数和返回结果的类型。
>
> #### QueryPlanCache 是什么？
>
> 在 Hibernate 中，QueryPlanCache 就是指具体的某一个类。我们通过核心源码看一下它是什么，如下所示。
>
> 复制代码
>
> ```
> package org.hibernate.engine.query.spi;
> //存储query plan 和 query parameter metdata
> public class QueryPlanCache implements Serializable {
>     //queryPlanCache的存储结构为自定义的HashMap结构，用来存储JPQL到SQL的转化过程及其SQL的执行语句和参数，返回结果的metadata;
>     private final BoundedConcurrentHashMap queryPlanCache;
>     //这个用来存储@Query的nativeQuery = true的query plan，即原始SQL的meta,包含参数和return type的 meta;
>     private final BoundedConcurrentHashMap<ParameterMetadataKey,ParameterMetadataImpl> parameterMetadataCache;
>     //QueryPlanCache的构造方法
>     public QueryPlanCache(final SessionFactoryImplementor factory, QueryPlanCreator queryPlanCreator) {
>        this.factory = factory;
>        this.queryPlanCreator = queryPlanCreator;
>        //maxParameterMetadata的个数，计算逻辑，可以自定义配置，或者采用默认值
>        Integer maxParameterMetadataCount = ConfigurationHelper.getInteger(
>              Environment.QUERY_PLAN_CACHE_PARAMETER_METADATA_MAX_SIZE,
>              factory.getProperties()
>        );
>        if ( maxParameterMetadataCount == null ) {
>           maxParameterMetadataCount = ConfigurationHelper.getInt(
>                 Environment.QUERY_PLAN_CACHE_MAX_STRONG_REFERENCES,
>                 factory.getProperties(),
>                 DEFAULT_PARAMETER_METADATA_MAX_COUNT
>           );
>        }
>        //maxQueryPlan的个数，计算逻辑，可以自定义配置大小，或者采用默认值
>        Integer maxQueryPlanCount = ConfigurationHelper.getInteger(
>              Environment.QUERY_PLAN_CACHE_MAX_SIZE,
>              factory.getProperties()
>        );
>        if ( maxQueryPlanCount == null ) {
>           maxQueryPlanCount = ConfigurationHelper.getInt(
>                 Environment.QUERY_PLAN_CACHE_MAX_SOFT_REFERENCES,
>                 factory.getProperties(),
>                 DEFAULT_QUERY_PLAN_MAX_COUNT
>           );
>        }
>        //新建一个 BoundedConcurrentHashMap的queryPlanCache，用来存储JPQL和Criteria Query到SQL的转化过程
>        queryPlanCache = new BoundedConcurrentHashMap( maxQueryPlanCount, 20, BoundedConcurrentHashMap.Eviction.LIRS );
>     //新建一个 BoundedConcurrentHashMap的parameterMetadataCache，用来存储Native SQL的转化过程
>        parameterMetadataCache = new BoundedConcurrentHashMap<>(
>              maxParameterMetadataCount,
>              20,
>              BoundedConcurrentHashMap.Eviction.LIRS
>        );
>        nativeQueryInterpreter = factory.getServiceRegistry().getService( NativeQueryInterpreter.class );
>     }
>     // 默认的parameterMetadataCache的HashMap的存储空间大小，默认128条
>     public static final int DEFAULT_PARAMETER_METADATA_MAX_COUNT = 128;
>     //默认的queryPlanCache的HashMap存储空间大小，默认2048条
>     public static final int DEFAULT_QUERY_PLAN_MAX_COUNT = 2048;
> ......不重要的代码先省略
> }
> ```
>
> 很好理解，通过源码和概念的分析你就大概知道 QueryPlanCache 是什么了，那么我们再来看一下它的里面具体会存储什么内容呢？
>
> #### QueryPlanCache 存储的内容
>
> 我们新建一个 UserInfoRepository，来测试一下。假设 UserInfoRepository 里面有如下几个方法。
>
> 复制代码
>
> ```
> public interface UserInfoRepository extends JpaRepository<UserInfo, Long> {
>    //没有用@Query，直接使用method name defining query
>    List<UserInfo> findByNameAndCreateTimeBetween(String name, Instant begin, Instant endTime);
>    //演示SpEL根据数组下标取参数，和根据普通的Parma的名字:name取参数
>    @Query("select u from UserInfo u where u.lastName like %:#{[0]} and u.name like %:name%")
>    List<UserInfo> findContainingEscaped(@Param("name") String name);
>    //SpEL取Parma的名字customer里面的属性
>    @Query("select u from UserInfo u where u.name = :#{#customer.name}")
>    List<UserInfo> findUsersByCustomersFirstname(@Param("customer") UserInfo customer);
>    //利用SpEL根据一个写死的'jack'字符串作为参数
>    @Query("select u from UserInfo u where u.name = ?#{'jack'}")
>    List<UserInfo> findOliverBySpELExpressionWithoutArgumentsWithQuestionmark();
>    @Query(value = "select * from user_info where name=:name",nativeQuery = true)
>    List<UserInfo> findByName(@Param(value = "name") String name);
> }
> ```
>
> 当项目启动成功之后你会发现，通过 @Query 定义的 nativeQuery=false 的 JPQL，会在启动成功之后预先放在 QueryPlanCache 里面，我们设置一个断点就可以看到如下内容。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3dOAL3LeAAQ5bROVfnw790.png)
>
> 发现里面 parameterMetadataCache 是空的，也就是没有放置 nativeQuery=true 的 Query SQL，并且可以看到我们在方法里面定义的其他三个 @Query 的 JPQL 解析过程。那么我们打开第一个详细看一下，如下图所示。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3dmAPB5bAAKvPbM8_1U786.png)
>
> 你会发现一个 QueryPlanCache 还是能存挺多东西的：navtive sql、参数、return 等各种 metadata。也可以看出一个简单的 JPQL 查询会有些占用堆内存，所以如果是复杂点的项目，各种查询的 JPQL 多一点的话，启动所需要的最小堆内存会占用 300M、400M 的空间，这是正常现象。
>
> 在 UserInfoRepository 的五个方法中，剩下的两个方法分别是 name defining query 和 nativeQuery=true。这两种情况是，当调用的时候发现 QueryPlanCache 里面没有它们，于是就会被增加进去，下次就可以直接从 QueryPlanCache 里面取了。那么我们在 Controller 里面执行这两个方法，如下所示。
>
> 复制代码
>
> ```
> userInfoRepository.findByNameAndCreateTimeBetween("JK", Instant.now(),Instant.now());
> userInfoRepository.findByName("jack");
> ```
>
> 然后通过断点就会发现 QueryPlanCache 里面多了两个 Cache，如下图所示。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/02/2A/Cip5yF_Z3eCAeweyAAFrt_Cy4Fw449.png)
>
> 同时，parameterMetadataCache 里面就会多一条 key/value的nativeQuery=true 的解析记录，如下图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3eaAOrDUAAJ8yCqnbQI709.png)
>
> 通过上面的案例讲解，相信你已经清楚了 QueryPlanCache 的概念，总结起来就是，QueryPlanCache 用来存储的 JQPL 或者 SQL 的 Metadata 信息，从而提升了 Hibernate 执行 JPQL 的性能，因为只有第一次需要把 JPQL 转化成 SQL，后面的每次操作就可以直接从 HashMap 中找到对应的 SQL，直接执行就可以了。
>
> 那么它和 Session 到底是什么关系呢？它是否在一级缓存里面？
>
> #### QueryPlanCache 和 Session 是什么关系？
>
> 我们通过查看源码会发现，在 SessionFactoryImpl 的构造方法里面会 new QueryPlanCache(...)，关键源码如下。
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3e6APucVAAH8SpvcdMQ168.png)
>
> 说明这个 application 只需要创建一次 QueryPlanCache，整个项目周期是单例的，也就是可以被不同的 Session 共享，那么我们可以查看 Session 的关键源码，如下图所示。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image2/M01/02/2A/Cip5yF_Z3feAeEqeAAMBOQfm2s8156.png)
>
> 也就是说，每一个 SessionImpl 的实例在获得 query plan 之前，都会去同一个 QueryPlanCache 里面查询一下 JPQL 对应的执行计划。所以我们可以看得出来 QueryPlanCache 和 Session 的关系有如下几点。
>
> 1. QueryPlanCache 在整个 Spring Application 周期内就是一个实例；
> 2. 不同的 Session 作用域，可以代表不同的 SessionImpl 实例共享 QueryPlanCache；
> 3. QueryPlanCache 和我们所说的一级缓存完全不是一个概念，这点你要分清楚。
>
> 而实际工作中大部分场景 QueryPlanCache 都是没有问题的，只有在 In 的 SQL 查询的场景会引发内存泄漏的问题，我们看一下。
>
> ### QueryPlanCache 中 In 查询引发的内存泄漏问题
>
> 我们在实际的工作中使用 JPA 的时候，会发现其内存越来越大，而不会被垃圾回收机制给回收掉，现象就是堆内存随着时间的推移使用量越来越大，如下图所示，很明显是内存泄漏的问题。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3f6AJ3zkAAA0cwikRBI522.png)
>
> 而我们把堆栈拿出来分析的话会发现，其实是 Hibernate 的 QueryPlanCache 占用了大量的内存，如下图所示。
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image2/M01/02/2A/Cip5yF_Z3gOAAypLAAH1VLcPqOU952.png)
>
> 我们点开仔细看的话，发现大部分都是某些 In 相关的 SQL 语句。这就是我们常见的 In 查询引起的内存泄漏，那么为什么会发生这种现象呢？
>
> #### In 查询条件引发内存泄漏的原因
>
> 我们在 UserInfoRepository 里面新增一个 In 条件的查询方法，模拟一下实际工作中的 In 查询条件的场景，如下所示。
>
> 复制代码
>
> ```
> public interface UserInfoRepository extends JpaRepository<UserInfo, Long> {
> //测试In查询条件的情况
> List<UserInfo> findByNameAndUrlIn(String name, Collection<String> urls);
> }
> ```
>
> 假设有个需求，查询拥有个人博客地址的用户有哪些？那么我们的 Controller 里面有如下方法。
>
> 复制代码
>
> ```
> @GetMapping("/users")
> public List<UserInfo> getUserInfos(List<String> urls) {
>   //根据urls批量查询，我们模拟实际工作中的批量查询情况，实际工作中可能会有大量的根据不同的IDS批量查询的场景；
>   return userInfoRepository.findByNameAndUrlIn("jack",urls);
> }
> ```
>
> 我们 debug 看一下 QueryPlanCache 里面的情况，会发现随着 In 查询条件的个数增加，会生成不同的 QueryPlanCache，如下图所示，分别是 1 个参数、3 个参数、6个参数的情况。
>
> ![Drawing 9.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3guAcgqyAAILGhMwzQg748.png)
>
> 从图中我们可以想象一下，如果业务代码中有各种 In 的查询操作，不同的查询条件的个数肯定在大部分场景中也是不一样的，甚至有些场景我们能一下查询到几百个 ID 对应的数据，可想而知，那得生成多少个 In 相关的 QueryPlanCache 呀。
>
> 而依据 QueryPlanCache 的原理，整个工程都是单例的，放进去之后肯定不会进行内存垃圾回收，那么程序运行时间久了之后就会发生内存泄漏，甚至一段时间之后还会导致内存溢出的现象发生。那么该如何解决此类问题呢？
>
> #### 解决 In 查询条件内存泄漏的方法
>
> **第一种方法：修改缓存的最大条数限制**
>
> 正如我们上面介绍的，默认 DEFAULT_QUERY_PLAN_MAX_COUNT = 2048，也就是 query plan 的最大条数限制是 2048。这样默认值可能有点大了，我们可以通过如下方式修改默认值，请看代码。
>
> 复制代码
>
> ```
> #修改 默认的plan_cache_max_size，太小会影响JPQL的执行性能，所以根据实际情况可以自由调整，不宜太小，也不宜太大，太大可能会引发内存溢出
> spring.jpa.properties.hibernate.query.plan_cache_max_size=512
> #修改 默认的native query的cache大小
> spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size=128
> ```
>
> **第二种方法：根据 max plan count 适当增加堆内存大小**
>
> 因为 QueryPlanMaxCount 是有限制的，那么肯定最大堆内存的使用也是有封顶限制的，我们找到临界值修改最小、最大堆内存即可。
>
> **第三种方法：减少 In 的查询 SQL 生成条数**，配置如下所示。
>
> 复制代码
>
> ```
> ### 默认情况下，不同的in查询条件的个数会生成不同的plan query cache，我们开启了in_clause_parameter_padding之后会减少in生成cache的个数，会根据参数的格式运用几何的算法生成QueryCache；
> spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
> ```
>
> 也就是说，当 In 的时候，参数个数会对应归并 QueryPlanCache 变成 1、2、4、8、16、32、64、128 个参数的 QueryPlanCache。那么我们再看一下刚才参数个数分别在 1、3、4、5、6、7、8 个的时候生成 QueryPlanCache 的情况，如下图所示。
>
> ![Drawing 10.png](https://s0.lgstatic.com/i/image2/M01/02/2B/CgpVE1_Z3hWAGkLbAAH30LPJHmA137.png)
>
> 我们会发现，In 产生个数是 1 个的时候，它会共享参数为 1 个的 QueryPlanCache；而当参数是 3、4 个 In 参数的时候，它就会使用 4 个参数的 QueryPlanCache；以此类推，当参数是 5、6、7、8 个的时候，会使用 8 个参数的 QueryPlanCache……这种算法可以大大地减少 In 的不同查询参数生成的 QueryPlanCache 个数，占用的内存自然会减少很多。
>
> ### 总结
>
> 以上就是本讲介绍的全部内容，主要是帮助你理清工作中关于缓存的一些概念，其实一级缓存的原理我们在前面几讲都有详细介绍。其中你要重点了解一下 Query Plan Cache，因为实际工作中很多人会把它和一级缓存的概念混为一谈。
>
> 学习就是不断思考的过程，希望你能踊跃留言讨论。下个课时我会重点介绍二级缓存以及它的最佳实践，到时见。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa