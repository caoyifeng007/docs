[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4712)



> 今天我们来讲一下 JPA 的审计功能，即 Auditing，通过了解这一概念及其实现原理，分析这一功能可以帮我们解决哪些问题。
>
> 在学习的过程中，希望你可以跟着我的步骤去思考，并动动手自己去实践一下。希望通过本课时的学习，你可以掌握 JPA 审计功能的相关内容，在实操中运用起来会更加得心应手。
>
> ### Auditing 指的是什么？
>
> Auditing 是帮我们做审计用的，当我们操作一条记录的时候，需要知道这是谁创建的、什么时间创建的、最后修改人是谁、最后修改时间是什么时候，甚至需要修改记录……这些都是 Spring Data JPA 里面的 Auditing 支持的，它为我们提供了四个注解来完成上面说的一系列事情，如下：
>
> - @CreatedBy 是哪个用户创建的。
> - @CreatedDate 创建的时间。
> - @LastModifiedBy 最后修改实体的用户。
> - @LastModifiedDate 最后一次修改的时间。
>
> 这就是 Auditing 了，那么它具体怎么实现呢？
>
> ### Auditing 如何实现？
>
> 利用上面的四个注解实现方法，一共有三种方式实现 Auditing，我们分别看看。
>
> #### 第一种方式：直接在实例里面添加上述四个注解
>
> 我们还用之前的例子，把 User 实体添加四个字段，分别记录创建人、创建时间、最后修改人、最后修改时间。
>
> **第一步：在 @Entity：User 里面添加四个注解，并且新增 @EntityListeners(AuditingEntityListener.class) 注解。**
>
> 添加完之后，User 的实体代码如下：
>
> 复制代码
>
> ```
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "addresses")
> @EntityListeners(AuditingEntityListener.class)
> public class User implements Serializable {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String name;
>    private String email;
>    @Enumerated(EnumType.STRING)
>    private SexEnum sex;
>    private Integer age;
>    @OneToMany(mappedBy = "user")
>    @JsonIgnore
>    private List<UserAddress> addresses;
>    private Boolean deleted;
>    @CreatedBy
>    private Integer createUserId;
>    @CreatedDate
>    private Date createTime;
>    @LastModifiedBy
>    private Integer lastModifiedUserId;
>    @LastModifiedDate
>    private Date lastModifiedTime;
> }
> ```
>
> 在 @Entity 实体中我们需要做两点操作：
>
> 1.其中最主要的四个字段分别记录创建人、创建时间、最后修改人、最后修改时间，代码如下：
>
> 复制代码
>
> ```
>    @CreatedBy
>    private Integer createUserId;
>    @CreatedDate
>    private Date createTime;
>    @LastModifiedBy
>    private Integer lastModifiedUserId;
>    @LastModifiedDate
>    private Date lastModifiedTime;
> ```
>
> 2.其中 AuditingEntityListener 不能少，必须通过这段代码：
>
> 复制代码
>
> ```
> @EntityListeners(AuditingEntityListener.class)
> ```
>
> 在 Entity 的实体上面进行注解。
>
> **第二步：实现 AuditorAware 接口，告诉 JPA 当前的用户是谁。**
>
> 我们需要实现 AuditorAware 接口，以及 getCurrentAuditor 方法，并返回一个 Integer 的 user ID。
>
> 复制代码
>
> ```
> public class MyAuditorAware implements AuditorAware<Integer> {
>    //需要实现AuditorAware接口，返回当前的用户ID
>    @Override
>    public Optional<Integer> getCurrentAuditor() {
>       ServletRequestAttributes servletRequestAttributes =
>             (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
>       Integer userId = (Integer) servletRequestAttributes.getRequest().getSession().getAttribute("userId");
>       return Optional.ofNullable(userId);
>    }
> }
> ```
>
> 这里关键的一步，是实现 AuditorAware 接口的方法，如下所示：
>
> 复制代码
>
> ```
> public interface AuditorAware<T> {
>    T getCurrentAuditor();
> }
> ```
>
> 需要注意的是：这里获得用户 ID 的方法不止这一种，实际工作中，我们可能将当前的 user 信息放在 Session 中，可能把当前信息放在 Redis 中，也可能放在 Spring 的 security 里面管理。此外，这里的实现会有略微差异，我们以 security 为例：
>
> 复制代码
>
> ```
> Authentication authentication =  SecurityContextHolder.getContext().getAuthentication();
> if (authentication == null || !authentication.isAuthenticated()) {
>   return null;
> }
> Integer userId = ((LoginUserInfo) authentication.getPrincipal()).getUser().getId();
> ```
>
> 这时获取 userId 的代码可能会变成上面这样子，你了解一下就好。
>
> **第三步：通过 @EnableJpaAuditing 注解开启 JPA 的 Auditing 功能。**
>
> 第三步是最重要的一步，如果想使上面的配置生效，我们需要开启 JPA 的 Auditing 功能（默认没开启）。这里需要用到的注解是 @EnableJpaAuditing，代码如下：
>
> 复制代码
>
> ```
> @Inherited
> @Documented
> @Target(ElementType.TYPE)
> @Retention(RetentionPolicy.RUNTIME)
> @Import(JpaAuditingRegistrar.class)
> public @interface EnableJpaAuditing {
> //auditor用户的获取方法，默认是找AuditorAware的实现类；
> String auditorAwareRef() default "";
> //是否在创建修改的时候设置时间，默认是true
> boolean setDates() default true;
> //在创建的时候是否同时作为修改，默认是true
> boolean modifyOnCreate() default true;
> //时间的生成方法，默认是取当前时间(为什么提供这个功能呢？因为测试的时候有可能希望时间保持不变，它提供了一种自定义的方法)；
> String dateTimeProviderRef() default "";
> }
> ```
>
> 在了解了@EnableJpaAuditing注解之后，我们需要创建一个Configuration 文件，添加 @EnableJpaAuditing 注解，并且把我们的 MyAuditorAware 加载进去即可，如下所示：
>
> 复制代码
>
> ```
> @Configuration
> @EnableJpaAuditing
> public class JpaConfiguration {
>    @Bean
>    @ConditionalOnMissingBean(name = "myAuditorAware")
>    MyAuditorAware myAuditorAware() {
>       return new MyAuditorAware();
>    }
> }
> ```
>
> **经验之谈：**
>
> 1. 这里说一个 Congifuration 的最佳实践的写法。我们为什么要单独写一个JpaConfiguration的配置文件，而不是把@EnableJpaAuditing 放在 JpaApplication 的类里面呢？因为这样的话 JpaConfiguration 文件可以单独加载、单独测试，如果都放在 Appplication 类里面的话，岂不是每次测试都要启动整个应用吗？
> 2. MyAuditorAware 也可以通过 @Component 注解进行加载，我为什么推荐 @Bean 的方式呢？因为这种方式可以让使用的人直接通过我们的配置文件知道我们自定义了哪些组件，不会让用的人产生不必要的惊讶，这是一点写 framework 的经验，供你参考。
>
> **第四步：我们写个测试用例测试一下。**
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
>     @Test
>     public void testAuditing() {
>         //由于测试用例模拟web context环境不是我们的重点，我们这里利用@MockBean，mock掉我们的方法，期待返回13这个用户ID
>         Mockito.when(myAuditorAware.getCurrentAuditor()).thenReturn(Optional.of(13));
>         //我们没有显式的指定更新时间、创建时间、更新人、创建人
>         User user = User.builder()
>                 .name("jack")
>                 .email("123456@126.com")
>                 .sex(SexEnum.BOY)
>                 .age(20)
>                 .build();
>         userRepository.save(user);
>         //验证是否有创建时间、更新时间，UserID是否正确；
>         List<User> users = userRepository.findAll();
>         Assertions.assertEquals(13,users.get(0).getCreateUserId());
>         Assertions.assertNotNull(users.get(0).getLastModifiedTime());
>         System.out.println(users.get(0));
>     }
> }
> ```
>
> **需要注意的是：**
>
> 1. 我们利用 @MockBean 模拟 MyAuditorAware 返回结果 13 这个 UserID；
> 2. 我们测试并验证 create_user_id 是否是我们预期的。
>
> 测试结果如下：
>
> 复制代码
>
> ```
> User(id=1, name=jack, email=123456@126.com, sex=BOY, age=20, deleted=null, createUserId=13, createTime=Sat Oct 03 21:19:57 CST 2020, lastModifiedUserId=13, lastModifiedTime=Sat Oct 03 21:19:57 CST 2020)
> ```
>
> 结果完全符合我们的预期。
>
> 那么现在是不是学会了 Auditing 的第一种方式呢？此外，Spring Data JPA 还给我们提供了第二种方式：实体直接实现 Auditable 接口即可，我们来看一下。
>
> #### 第二种方式：实体里面实现Auditable 接口
>
> 我们改一下上面的 User 实体对象，如下：
>
> 复制代码
>
> ```
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "addresses")
> @EntityListeners(AuditingEntityListener.class)
> public class User implements Auditable<Integer,Long, Instant> {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String name;
>    private String email;
>    @Enumerated(EnumType.STRING)
>    private SexEnum sex;
>    private Integer age;
>    @OneToMany(mappedBy = "user")
>    @JsonIgnore
>    private List<UserAddress> addresses;
>    private Boolean deleted;
>    private Integer createUserId;
>    private Instant createTime;
>    private Integer lastModifiedUserId;
>    private Instant lastModifiedTime;
>    @Override
>    public Optional<Integer> getCreatedBy() {
>       return Optional.ofNullable(this.createUserId);
>    }
>    @Override
>    public void setCreatedBy(Integer createdBy) {
>       this.createUserId = createdBy;
>    }
>    @Override
>    public Optional<Instant> getCreatedDate() {
>       return Optional.ofNullable(this.createTime);
>    }
>    @Override
>    public void setCreatedDate(Instant creationDate) {
>       this.createTime = creationDate;
>    }
>    @Override
>    public Optional<Integer> getLastModifiedBy() {
>       return Optional.ofNullable(this.lastModifiedUserId);
>    }
>    @Override
>    public void setLastModifiedBy(Integer lastModifiedBy) {
>       this.lastModifiedUserId = lastModifiedBy;
>    }
>    @Override
>    public void setLastModifiedDate(Instant lastModifiedDate) {
>       this.lastModifiedTime = lastModifiedDate;
>    }
>    @Override
>    public Optional<Instant> getLastModifiedDate() {
>       return Optional.ofNullable(this.lastModifiedTime);
>    }
>    @Override
>    public boolean isNew() {
>       return id==null;
>    }
> }
> ```
>
> 与第一种方式的差异是，这里我们要去掉上面说的四个注解，并且要实现接口 Auditable 的方法，代码会变得很冗余和啰唆。
>
> 而其他都不变，我们再跑一次刚才的测试用例，发现效果是一样的。从代码的复杂程度来看，这种方式我不推荐使用。那么我们再看一下第三种方式。
>
> #### 第三种方式：利用 @MappedSuperclass 注解
>
> 我们在第 6 课时讲对象的多态的时候提到过这个注解，它主要是用来解决公共 BaseEntity 的问题，而且其代表的是继承它的每一个类都是一个独立的表。
>
> 我们先看一下 @MappedSuperclass 的语法。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/60/79/CgqCHl-NUkOAQEPqAABJcttRqYM933.png)
>
> 它注解里面什么都没有，其实就是代表了抽象关系，即所有子类的公共字段而已。那么接下来我们看一下实例。
>
> **第一步：创建一个 BaseEntity，里面放一些实体的公共字段和注解。**
>
> 复制代码
>
> ```
> package com.example.jpa.example1.base;
> import org.springframework.data.annotation.*;
> import javax.persistence.MappedSuperclass;
> import java.time.Instant;
> @Data
> @MappedSuperclass
> @EntityListeners(AuditingEntityListener.class)
> public class BaseEntity {
>    @CreatedBy
>    private Integer createUserId;
>    @CreatedDate
>    private Instant createTime;
>    @LastModifiedBy
>    private Integer lastModifiedUserId;
>    @LastModifiedDate
>    private Instant lastModifiedTime;
> }
> ```
>
> **注意：** BaseEntity里面需要用上面提到的四个注解，并且加上@EntityListeners(AuditingEntityListener.class)，这样所有的子类就不需要加了。
>
> **第二步：实体直接继承 BaseEntity 即可。**
>
> 我们修改一下上面的 User 实例继承 BaseEntity，代码如下：
>
> 复制代码
>
> ```
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "addresses")
> public class User extends BaseEntity {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String name;
>    private String email;
>    @Enumerated(EnumType.STRING)
>    private SexEnum sex;
>    private Integer age;
>    @OneToMany(mappedBy = "user")
>    @JsonIgnore
>    private List<UserAddress> addresses;
>    private Boolean deleted;
> }
> ```
>
> 这样的话，User 实体就不需要关心太多，我们只关注自己需要的逻辑即可，如下：
>
> 1. 去掉了 @EntityListeners(AuditingEntityListener.class)；
> 2. 去掉了 @CreatedBy、@CreatedDate、@LastModifiedBy、@LastModifiedDate 四个注解的公共字段。
>
> 接着我们再跑一下上面的测试用例，发现效果还是一样的。
>
> **这种方式，是我最推荐的，也是实际工作中使用最多的一种方式**。它的好处显而易见就是公用性强，代码简单，需要关心的少。
>
> 通过上面的实际案例，我们其实也能很容易发现 Auditing 帮我们解决了什么问题，下面总结一下。
>
> ### JPA 的审计功能解决了哪些问题？
>
> 1.可以很容易地让我们写自己的 BaseEntity，把一些公共的字段放在里面，不需要我们关心太多和业务无关的字段，更容易让我们公司的表更加统一和规范，就是统一加上 @CreatedBy、@CreatedDate、@LastModifiedBy、@LastModifiedDate 等。
>
> 实际工作中，BaseEntity 可能还更复杂一点，比如说把 ID 和 @Version 加进去，会变成如下形式：
>
> 复制代码
>
> ```
> @Data
> @MappedSuperclass
> @EntityListeners(AuditingEntityListener.class)
> public class BaseEntity {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    @CreatedBy
>    private Integer createUserId;
>    @CreatedDate
>    private Instant createTime;
>    @LastModifiedBy
>    private Integer lastModifiedUserId;
>    @LastModifiedDate
>    private Instant lastModifiedTime;
>    @Version
>    private Integer version;
> }
> ```
>
> 其中 @Version 的详细使用方法，我们在 14 课时讲乐观锁的机制时再详细讲解。
>
> 2.Auditing 在实战应用场景中，比较适合做后台管理项目，对应纯粹的 RestAPI 项目，提供给用户直接查询的 API 的话，可以考虑一个特殊的 UserID。
>
> 到这里，JPA 的审计功能解决了哪些问题，你都清楚了吗？
>
> ### Auditing 的实现原理
>
> 方法你应该已经掌握了，其实这个时候我们应该好奇一下，其原理是怎么实现的？我们来操作一下。
>
> **第一步：还是从 @EnableJpaAuditing 入手分析。**
>
> 我们前面讲了它的使用方法，这次我们分析一下其加载原理，看下面的图：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/60/79/CgqCHl-NUnKAeeBXAAQyKDxdJXI748.png)
>
> 我们可以知道，首先 Auditing 这套封装是 Spring Data JPA 实现的，而不是 Java Persistence API 规定的，其注解里面还有一项重要功能就是 @Import(JpaAuditingRegistrar.class) 这个类，它帮我们处理 Auditing 的逻辑。
>
> 我们看其源码，一步一步地 debug 下去可以发现如下所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/60/6E/Ciqc1F-NUn-ALSVrAAJFrZu0CbE230.png)
>
> 进一步进入到如下方法中：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/60/6E/Ciqc1F-NUoWASJBTAASvnhF4_WA538.png)
>
> 可以看到 Spring 容器给 AuditingEntityListener.class 注入了一个 AuditingHandler 的处理类。
>
> **第二步：打开 AuditingEntityListener.class 的源码分析 debug 一下。**
>
> 复制代码
>
> ```
> @Configurable
> public class AuditingEntityListener {
>    private @Nullable ObjectFactory<AuditingHandler> handler;
>    public void setAuditingHandler(ObjectFactory<AuditingHandler> auditingHandler) {
>       Assert.notNull(auditingHandler, "AuditingHandler must not be null!");
>       this.handler = auditingHandler;
>    }
>    @PrePersist
>    public void touchForCreate(Object target) {
>       Assert.notNull(target, "Entity must not be null!");
>       if (handler != null) {
>          AuditingHandler object = handler.getObject();
>          if (object != null) {
>             object.markCreated(target);
>          }
>       }
>    }
>    @PreUpdate
>    public void touchForUpdate(Object target) {
>       Assert.notNull(target, "Entity must not be null!");
>       if (handler != null) {
>          AuditingHandler object = handler.getObject();
>          if (object != null) {
>             object.markModified(target);
>          }
>       }
>    }
> }
> ```
>
> 从源码我们可以看到，AuditingEntityListener 的实现还是比较简单的，利用了 Java Persistence API 里面的@PrePersist、@PreUpdate 回调函数，在更新和创建之前通过AuditingHandler 添加了用户信息和时间信息。
>  那么通过原理，我们能得出什么结论呢？
>
> #### 原理分析结论
>
> 1. 查看 Auditing 的实现源码，其实给我们提供了一个思路，就是怎么利用 @PrePersist、@PreUpdate 等回调函数和 @EntityListeners 定义自己的框架代码。这是值得我们学习和参考的，比如说 Auditing 的操作日志场景等。
> 2. 想成功配置 Auditing 功能，必须将 @EnableJpaAuditing 和 @EntityListeners(AuditingEntityListener.class) 一起使用才有效。
> 3. 我们是不是可以不通过 Spring data JPA 给我们提供的 Auditing 功能，而是直接使用 @PrePersist、@PreUpdate 回调函数注解在实体上，也可以达到同样的效果呢？答案是肯定的，因为回调函数是实现的本质。
>
> ### 总结
>
> 到这里，关于 JPA 的审计功能我们就介绍完了，不知道你有没有理解透彻。
>
> 本课时我们详细讲解了 Auditing 的使用方法，以及最佳实践是什么，还分析了 Auditing 的实现原理。那么我在下一课时会为你讲解 Java Persistence API 给我们提供的回调函数有哪些，还有最佳实践及其原理，这样就会让我们对 JPA 的使用更加游刃有余。
>
> 最后如果你觉得有收获就动动手指分享吧，也欢迎在下方留言、讨论，多思考、多实践，相信你可以做得更好。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa