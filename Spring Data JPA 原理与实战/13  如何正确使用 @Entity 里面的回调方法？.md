[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4713)



> 本课时我要介绍的是 @Entity 的回调方法。
>
> 为什么要讲回调函数呢？因为在工作中，我发现有些同事会把这个回调方法用得非常复杂，不得要领，所以我专门拿出一个课时来为你详细说明，并分享我的经验供你参考。我将通过“语法 + 实践”的方式讲解如何使用 @Entity 的回调方法，从而达到提高开发效率的目的。下面开始本课时的学习。
>
> ### Java Persistence API 里面规定的回调方法有哪些？
>
> JPA 协议里面规定，可以通过一些注解，为其监听回调事件、指定回调方法。下面我整理了一个回调事件注解表，分别列举了 @PrePersist、@PostPersist、@PreRemove、@PostRemove、@PreUpdate、@PostUpdate、@PostLoad注解及其概念。
>
> #### 回调事件注解表
>
> ![image (5).png](https://s0.lgstatic.com/i/image/M00/62/8F/Ciqc1F-SoLyAODuaAADhS0Urg_0032.png)
>
> #### 语法注意事项
>
> 关于上表所述的几个方法有一些需要注意的地方，如下：
>
> 1. 回调函数都是和 EntityManager.flush 或 EntityManager.commit 在同一个线程里面执行的，只不过调用方法有先后之分，都是同步调用，所以当任何一个回调方法里面发生异常，都会触发事务进行回滚，而不会触发事务提交。
> 2. Callbacks 注解可以放在实体里面，可以放在 super-class 里面，也可以定义在 entity 的 listener 里面，但需要注意的是：放在实体（或者 super-class）里面的方法，签名格式为“void ()”，即没有参数，方法里面操作的是 this 对象自己；放在实体的 EntityListener 里面的方法签名格式为“void (Object)”，也就是方法可以有参数，参数是代表用来接收回调方法的实体。
> 3. 使上述注解生效的回调方法可以是 public、private、protected、friendly 类型的，但是不能是 static 和 finnal 类型的方法。
>
> JPA 里面规定的回调方法还有一些，但不常用，我就不过多介绍了。接下来，我们看一下回调注解在实体里面是如何使用的。
>
> ### JPA Callbacks 的使用方法
>
> 这里我介绍两种方法，是你可能会在实际工作中用到的。
>
> #### 第一种用法：在实体和 super-class 中使用
>
> **第一步：修改 BaseEntity，在里面新增回调函数和注解，代码如下:**
>
> 复制代码
>
> ```
> package com.example.jpa.example1.base;
> import lombok.Data;
> import org.springframework.data.annotation.*;
> import org.springframework.data.jpa.domain.support.AuditingEntityListener;
> import javax.persistence.*;
> import java.time.Instant;
> @Data
> @MappedSuperclass
> @EntityListeners(AuditingEntityListener.class)
> public class BaseEntity {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
> // @CreatedBy 这个可能会被 AuditingEntityListener覆盖，为了方便测试，我们先注释掉
>    private Integer createUserId;
>    @CreatedDate
>    private Instant createTime;
>    @LastModifiedBy
>    private Integer lastModifiedUserId;
>    @LastModifiedDate
>    private Instant lastModifiedTime;
> //  @Version 由于本身有乐观锁机制，这个我们测试的时候先注释掉，改用手动设置的值；
>    private Integer version;
>    @PreUpdate
>    public void preUpdate() {
>       System.out.println("preUpdate::"+this.toString());
>       this.setCreateUserId(200);
>    }
>    @PostUpdate
>    public void postUpdate() {
>       System.out.println("postUpdate::"+this.toString());
>    }
>    @PreRemove
>    public void preRemove() {
>       System.out.println("preRemove::"+this.toString());
>    }
>    @PostRemove
>    public void postRemove() {
>       System.out.println("postRemove::"+this.toString());
>    }
>    @PostLoad
>    public void postLoad() {
>       System.out.println("postLoad::"+this.toString());
>    }
> }
> ```
>
> 上述代码中，我在类里面使用了@PreUpdate、@PostUpdate、@PreRemove、@PostRemove、@PostLoad 几个注解，并在相应的回调方法里面加了相应的日志。并且在 @PreUpdate 方法里面修改了 create_user_id 的值为 200，这样做是为了方便我们后续测试。
>
> **第二步：修改一下 User 类，也新增两个回调函数，并且和 BaseEntity 做法一样，代码如下：**
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import com.example.jpa.example1.base.BaseEntity;
> import com.fasterxml.jackson.annotation.JsonIgnore;
> import lombok.*;
> import javax.persistence.*;
> import java.util.List;
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "addresses",callSuper = true)
> @EqualsAndHashCode(callSuper=false)
> public class User extends BaseEntity {// implements Auditable<Integer,Long, Instant> {
>    private String name;
>    private String email;
>    @Enumerated(EnumType.STRING)
>    private SexEnum sex;
>    private Integer age;
>    @OneToMany(mappedBy = "user")
>    @JsonIgnore
>    private List<UserAddress> addresses;
>    private Boolean deleted;
>    @PrePersist
>    private void prePersist() {
>       System.out.println("prePersist::"+this.toString());
>       this.setVersion(1);
>    }
>    @PostPersist
>    public void postPersist() {
>       System.out.println("postPersist::"+this.toString());
>    }
> }
> ```
>
> 我在其中使用了 @PrePersist、@PostPersist 回调事件，为了方便我们测试，我在 @PrePersist 里面将 version 修改为 1。
>
> **第三步：写一个测试用例测试一下。**
>
> 复制代码
>
> ```
> @DataJpaTest
> @TestInstance(TestInstance.Lifecycle.PER_CLASS)
> @Import(JpaConfiguration.class)
> public class UserRepositoryTest {
>     @Autowired
>     private UserRepository userRepository;
>     @MockBean
>     MyAuditorAware myAuditorAware;
>     /**
>      * 为了和测试方法的事务分开，我们在 init 里面初始化数据做新增操作
>      */
>     @BeforeAll
>     @Rollback(false)
>     @Transactional
>     public void init() {
>         //由于测试用例模拟 web context 环境不是我们的重点，这里利用@MockBean，mock掉我们的方法，期待返回13这个用户ID
>        Mockito.when(myAuditorAware.getCurrentAuditor()).thenReturn(Optional.of(13));
>         User u1 = User.builder()
>                 .name("jack")
>                 .email("123456@126.com")
>                 .sex(SexEnum.BOY)
>                 .age(20)
>                 .build();
>         //没有save之前 version是null
>         Assertions.assertNull(u1.getVersion());
>         userRepository.save(u1);
>         //这里面触发保存方法，这个时候我们将version设置成了1，然后验证一下
>         Assertions.assertEquals(1,u1.getVersion());
>     }
>     /**
>      * 测试一下更新和查询
>      */
>     @Test
>     @Rollback(false)
>     @Transactional
>     public void testCallBackUpdate() {
>         //此时会触发@PostLoad事件
>         User u1 = userRepository.getOne(1L);
>         //我们从db里面重新查询出来，验证一下version是不是1
>         Assertions.assertEquals(1,u1.getVersion());
>         u1.setSex(SexEnum.GIRL);
>         //此时会触发@PreUpdate事件
>         userRepository.save(u1);
>         List<User> u3 = userRepository.findAll();
>         u3.stream().forEach(u->{
>             //我们从db查询出来，验证一下CcreateUserId是否为我们刚才修改的200
>            Assertions.assertEquals(200,u.getCreateUserId());
>         });
>     }
>     /**
>      * 测试一下删除事件
>      */
>     @Test
>     @Rollback(false)
>     @Transactional
>     public void testCallBackDelete() {
>         //此时会触发@PostLoad事件
>         User u1 = userRepository.getOne(1L);
>         Assertions.assertEquals(200,u1.getCreateUserId());
>         userRepository.delete(u1);
>         //此时会触发@PreRemove、@PostRemove事件
>         System.out.println("delete_after::");
>     }
> }
> ```
>
> 我们通过测试用例验证了回调函数的事件后，看一下输出的 SQL 和日志：
>
> ![image (6).png](https://s0.lgstatic.com/i/image/M00/62/9A/CgqCHl-SoPOAN80hAAMNJSyVLgc502.png)
>
> 我们通过上图的日志也可以看到响应的回调函数被触发了，并且可以看到我们在insert之前执行 prePersist 日志、在 insert 之后执行 postPersist 日志、在 select 之后执行 postLoad 方法的日志，以及在 update 的 sql 前后执行的 preUpdate 和 postUpdate 日志。
>
> 如果我们执行上面 remove 的测试用例，也会得到一样的效果：在 delete sql 之前会执行 preRemove 的方法并且打印日志，在 delete sql 之后会执行 postRemove 方法并打印日志。
>
> 那么使用这种方法，回调函数里面发生异常会怎么样呢？这也是你可能会遇到的问题，我来告诉你解决办法。
>
> 我们稍微修改一下上面的 @PostPersist 方法，手动抛一个异常出来，看看会发生什么。
>
> 复制代码
>
> ```
> @PostPersist
> public void postPersist() {
>    System.out.println("postPersist::"+this.toString());
>    throw new RuntimeException("jack test exception transactional roll back");
> }
> ```
>
> 我们再跑测试用例就会发现，其中发生了 RollbackException 异常，这样的话数据是不会提交到 DB 里面的，也就会导致数据进行回滚，后面的业务流程无法执行下去。
>
> 复制代码
>
> ```
> Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction
> org.springframework.transaction.TransactionSystemException: Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction
> ```
>
> 所以在使用此方法时，你要注意考虑异常情况，避免不必要的麻烦。
>
> #### 第二种用法：自定义 EntityListener
>
> **第一步：自定义一个 EntityLoggingListenner 用来记录操作日志，通过 listener 的方式配置回调函数注解，代码如下:**
>
> 复制代码
>
> ```
> package com.example.jpa.example1.base;
> import com.example.jpa.example1.User;
> import lombok.extern.log4j.Log4j2;
> import javax.persistence.*;
> @Log4j2
> public class EntityLoggingListener {
>     @PrePersist
>     private void prePersist(BaseEntity entity) {
>     //entity.setVersion(1); 如果注释了，测试用例这个地方的验证也需要去掉
>         log.info("prePersist::{}",entity.toString());
>     }
>     @PostPersist
>     public void postPersist(Object entity) {
>         log.info("postPersist::{}",entity.toString());
>     }
>     @PreUpdate
>     public void preUpdate(BaseEntity entity) {
>     //entity.setCreateUserId(200); 如果注释了，测试用例这个地方的验证也需要去掉
>         log.info("preUpdate::{}",entity.toString());
>     }
>     @PostUpdate
>     public void postUpdate(Object entity) {
>         log.info("postUpdate::{}",entity.toString());
>     }
>     @PreRemove
>     public void preRemove(Object entity) {
>         log.info("preRemove::{}",entity.toString());
>     }
>     @PostRemove
>     public void postRemove(Object entity) {
>         log.info("postRemove::{}",entity.toString());
>     }
>     @PostLoad
>     public void postLoad(Object entity) {
>     //查询方法里面可以对一些敏感信息做一些日志
>         if (User.class.isInstance(entity)) {
>             log.info("postLoad::{}",entity.toString());
>         }
>     }
> }
> ```
>
> 在这一步骤中需要注意的是：
>
> 1. 我们上面注释的代码，也可以改变 entity 里面的值，但是在这个 Listener 的里面我们不做修改，所以把 setVersion 和 setCreateUserId 注释掉了，要注意测试用例里面这两处也需要修改。
> 2. 如果在 @PostLoad 里面记录日志，不一定每个实体、每次查询都需要记录日志，只需要对一些敏感的实体或者字段做日志记录即可。
> 3. 回调函数时我们可以加上参数，这个参数可以是父类 Object，可以是 BaseEntity，也可以是具体的某一个实体；我推荐用 BaseEntity，因为这样的方法是类型安全的，它可以约定一些框架逻辑，比如 getCreateUserId、getLastModifiedUserId 等。
>
> **第二步：还是一样的道理，写一个测试用例跑一下。**
>
> 这次我们执行 testCallBackDelete()，看看会得到什么样的效果。
>
> 复制代码
>
> ```
> 2020-10-05 13:55:19.332  INFO 62541 --- [    Test worker] c.e.j.e.base.EntityLoggingListener       : prePersist::User(super=BaseEntity(id=null, createUserId=13, createTime=2020-10-05T05:55:19.246Z, lastModifiedUserId=13, lastModifiedTime=2020-10-05T05:55:19.246Z, version=null), name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null)
> 2020-10-05 13:55:19.449  INFO 62541 --- [    Test worker] c.e.j.e.base.EntityLoggingListener       : postPersist::User(super=BaseEntity(id=1, createUserId=13, createTime=2020-10-05T05:55:19.246Z, lastModifiedUserId=13, lastModifiedTime=2020-10-05T05:55:19.246Z, version=0), name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null)
> 2020-10-05 13:55:19.698  INFO 62541 --- [    Test worker] c.e.j.e.base.EntityLoggingListener       : postLoad::User(super=BaseEntity(id=1, createUserId=13, createTime=2020-10-05T05:55:19.246Z, lastModifiedUserId=13, lastModifiedTime=2020-10-05T05:55:19.246Z, version=0), name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null)
> 2020-10-05 13:55:19.719  INFO 62541 --- [    Test worker] c.e.j.e.base.EntityLoggingListener       : preRemove::User(super=BaseEntity(id=1, createUserId=13, createTime=2020-10-05T05:55:19.246Z, lastModifiedUserId=13, lastModifiedTime=2020-10-05T05:55:19.246Z, version=0), name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null)
> 2020-10-05 13:55:19.798  INFO 62541 --- [    Test worker] c.e.j.e.base.EntityLoggingListener       : postRemove::User(super=BaseEntity(id=1, createUserId=13, createTime=2020-10-05T05:55:19.246Z, lastModifiedUserId=13, lastModifiedTime=2020-10-05T05:55:19.246Z, version=0), name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null)
> ```
>
> 通过日志我们可以很清晰地看到 callback 注解标注的方法的执行过程，及其实体参数的值。你就会发现，原来自定义 EntityListener 回调函数的方法也是如此简单。
>
> 细心的你这个时候可能也会发现，我们上面其实应用了两个 EntityListener，所以这个时候 @EntityListeners 有个加载顺序的问题，你需要重点注意一下。
>
> #### 关于 @EntityListeners 加载顺序的说明
>
> 1. 默认如果子类和父类都有 EntityListeners，那么 listeners 会按照加载的顺序执行所有 EntityListeners；
> 2. EntityListeners 和实体里面的回调函数注解可以同时使用，但需要注意顺序问题；
> 3. 如果我们不想加载super-class里面的EntityListeners，那么我们可以通过注解 @ExcludeSuperclassListeners，排除所有父类里面的实体监听者，需要用到的时候，我们再在子类实体里面重新引入即可，代码如下：
>
> 复制代码
>
> ```
> @ExcludeSuperclassListeners
> public class User extends BaseEntity {
> ......}
> ```
>
> 看完了上面介绍的两种方式，关于 Callbacks 注解的用法你是不是已经掌握了呢？我强调需要注意的地方你要重点看一下，并切记在应用时不要搞错了。
>
> 上面说了这么多回调函数的注解使用方法，那么它的最佳实践是什么呢？
>
> ### JPA Callbacks 的最佳实践
>
> **我以个人经验总结了几个最佳实践。**
>
> 1.回调函数里面应尽量避免直接操作业务代码，最好用一些具有框架性的公用代码，如上一课时我们讲的 Auditing，以及本课时前面提到的实体操作日志等；
>
> 2.注意回调函数方法要在同一个事务中进行，异常要可预期，非可预期的异常要进行捕获，以免出现意想不到的线上 Bug；
>
> 3.回调函数方法是同步的，如果一些计算量大的和一些耗时的操作，可以通过发消息等机制异步处理，以免阻塞主流程，影响接口的性能。比如上面说的日志，如果我们要将其记录到数据库里面，可以在回调方法里面发个消息，改进之后将变成如下格式：
>
> 复制代码
>
> ```
> public class AuditLoggingListener {
>    @PostLoad
>    private void postLoad(Object entity) {
>       this.notice(entity, OperateType.load);
>    }
>    @PostPersist
>    private void postPersist(Object entity) {
>       this.notice(entity, OperateType.create);
>    }
>    @PostRemove
>    private void PostRemove(Object entity) {
>       this.notice(entity, OperateType.remove);
>    }
>    @PostUpdate
>    private void PostUpdate(Object entity) {
>       this.notice(entity, OperateType.update);
>    }
>    private void notice(Object entity, OperateType type) {
>       //我们通过active mq 异步发出消息处理事件
>       ActiveMqEventManager.notice(new ActiveMqEvent(type, entity));
>    }
>    @Getter
>    enum OperateType {
>       create("创建"), remove("删除"),update("修改"),load("查询");
>       private final String description;
>       OperateType(String description) {
>          this.description=description;
>       }
>    }
> }
> ```
>
> 4.在回调函数里面，尽量不要直接在操作 EntityManager 后再做 session 的整个生命周期的其他持久化操作，以免破坏事务的处理流程；也不要进行其他额外的关联关系更新动作，业务性的代码一定要放在 service 层面，否则太过复杂，时间长了代码很难维护；（ps：我曾经看到有人把回调函数用得十分复杂，做各种状态流转逻辑，时间长了连他自己也不知道是干什么的，耦合度太高了，你一定要谨慎。）
>
> 5.回调函数里面比较适合用一些计算型的transient方法，如下面这个操作：
>
> 复制代码
>
> ```
> public class UserListener {
>     @PrePersist
>     public void prePersist(User user) {
>         //通过一些逻辑计算年龄；
>         user.calculationAge();
>     }
> }
> ```
>
> 6.JPA 官方比较建议放一些默认值，但是我不是特别赞同，因为觉得那样不够直观，我们直接用字段初始化就可以了，没必要在回调函数里面放置默认值。
>
> **那么除了日志，还有没有其他实战应用场景呢？**
>
> 确实目前除了日志，Auditing 稍微公用一点，其他公用的场景不多。当遇到其他场景，你可以根据不同的实体实际情况制定自己独有的 EntityListener 方法，如下：
>
> 复制代码
>
> ```
> @Entity
> @EntityListeners(UserListener.class)
> public class User extends BaseEntity {// implements Auditable<Integer,Long, Instant> {
>    @Transient
>    public void calculationAge() {
>       //通过一些逻辑计算年龄；
>       this.age=10;
>    }
>    ......//其他不重要的省略
> }
> ```
>
> 例如，User 中我们有个计算年龄的逻辑要独立调用，就可以在持久化之前调用此方法，新建一个自己的 UserListener 即可，代码如下：
>
> 复制代码
>
> ```
> public class UserListener {
>     @PrePersist
>     public void prePersist(User user) {
>         //通过一些逻辑计算年龄；
>         user.calculationAge();
>     }
> }
> ```
>
> 以上，关于 JPA Callbacks 在一些实际场景中的最佳实践就介绍这些，希望你在应用的时候多注意找方法，避免不必要的操作，也希望我的经验可以帮助到你。
>
> ### JPA Callbacks 的实现原理，事件机制
>
> 那么 callbacks 的实现原理是什么呢？其实很简单，Java Persistence API规定：JPA 的实现方需要实现功能，需要支持回调事件注解；而 Hibernate 内部负责实现，Hibernate 内部维护了一套实体的 EventType，其内部包含了各种回调事件，下面列举一下：
>
> 复制代码
>
> ```
> public static final EventType<PreLoadEventListener> PRE_LOAD = create( "pre-load", PreLoadEventListener.class );
> public static final EventType<PreDeleteEventListener> PRE_DELETE = create( "pre-delete", PreDeleteEventListener.class );
> public static final EventType<PreUpdateEventListener> PRE_UPDATE = create( "pre-update", PreUpdateEventListener.class );
> public static final EventType<PreInsertEventListener> PRE_INSERT = create( "pre-insert", PreInsertEventListener.class );
> public static final EventType<PostLoadEventListener> POST_LOAD = create( "post-load", PostLoadEventListener.class );
> public static final EventType<PostDeleteEventListener> POST_DELETE = create( "post-delete", PostDeleteEventListener.class );
> public static final EventType<PostUpdateEventListener> POST_UPDATE = create( "post-update", PostUpdateEventListener.class );
> public static final EventType<PostInsertEventListener> POST_INSERT = create( "post-insert", PostInsertEventListener.class );
> ```
>
> 更多的事件类型，你可以通过查看 org.hibernate.event.spi.EventType 类，了解更多；在 session factory 构建的时候，EventListenerRegistryImpl 负责注册这些事件，我们看一下 debug 的关键节点：
>
> ![image (7).png](https://s0.lgstatic.com/i/image/M00/62/9B/CgqCHl-SoWSAXvSiAARU062JRFI108.png)
>
> 通过一步一步断点，再结合 Hibernate 的官方文档，可以了解内部 EventType 事件的创建机制，由于我们不常用这部分原理，知道有这么回事即可，你有兴趣也可以深入 debug 研究一下。
>
> ### 总结
>
> 到这里，本课时内容就介绍这么多。这一节，我们分析了语法，列举了实战使用场景及最佳实践，相信通过上面提到的异常、异步、避免死循环等处理方法，你已经知道回调函数的正确使用方法了。其中最佳实践场景也欢迎你补充，我们可以一起探讨。
>
> 下一课时，我们将迎来很多人都感兴趣的“乐观锁机制和重试机制”相关内容，到时候我会告诉你它们在实战中都是怎么使用的。
>
> > 点击下方链接查看源码：（不定时更新）