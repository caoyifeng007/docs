[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4725)



> 在 JPA 的使用过程中，N+1 SQL 是很常见的问题，相信很多程序员都遇到过这一问题，我看见很多同事处理起来束手无策，那么它究竟真的有那么麻烦吗？这一讲我会帮助你梳理思路，看看到底如何解决这个经典问题。
>
> 注：由于内容较多，我将这部分内容拆分成两讲，方便你学习。
>
> ### 什么是 N+1 SQL 问题？
>
> 想要解决一个问题，必须要知道它是什么、如何产生的，这样才能有方法、有逻辑地去解决它。下面通过一个例子来看一下什么是 N+1 的 SQL 问题。
>
> 假设一个 UserInfo 实体对象和 Address 是一对多的关系，即一个用户有多个地址，我们首先看一下一般实体里面的关联关系会怎么写。两个实体对象如下述代码所示。
>
> 复制代码
>
> ```
> //UserInfo实体对象如下：
> @Entity
> @Data
> @SuperBuilder
> @AllArgsConstructor
> @NoArgsConstructor
> @Table
> @ToString(exclude = "addressList")//exclued防止 toString打印日志的时候死循环
> public class UserInfo extends BaseEntity {
>    private String name;
>    private String telephone;
>    // UserInfo实体对象的关联关系由Address对象里面的userInfo字段维护，默认是lazy加载模式，为了方便演示fetch取EAGER模式。此处是一对多关联关系
>    @OneToMany(mappedBy = "userInfo",fetch = FetchType.EAGER)
>    private List<Address> addressList;
> }
> //Address对象如下：
> @Entity
> @Table
> @Data
> @SuperBuilder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "userInfo")
> public class Address extends BaseEntity {
>    private String city;
>    //维护UserInfo和Address的外键关系，方便演示也采用EAGER模式；
>    @ManyToOne(fetch = FetchType.EAGER)
>    @JsonBackReference //此注解防止JSON死循环
>    private UserInfo userInfo;
> }
> ```
>
> 其次，我们假设数据库里面有三条 UserInfo 的数据，ID 分别为 3、6、9，如下图所示。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/78/8C/CgqCHl_KD8SABiKCAACKI1OHeQg789.png)
>
> 其中，每个 UserInfo 分别有两条 Address 数据，也就是一共 6 条 Address 的数据，如下图所示。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/78/8C/CgqCHl_KD8mAe5A7AAErBNjfxAg931.png)
>
> 然后，我们请求通过 UserInfoRepository 查询所有的 UserInfo 信息，方法如下面这行代码所示。
>
> 复制代码
>
> ```
> userInfoRepository.findAll()
> ```
>
> 现在，我们的控制台将会得到四个 SQL，如下所示。
>
> 复制代码
>
> ```
> org.hibernate.SQL                        :
> select userinfo0_.id                    as id1_1_,
>        userinfo0_.create_time           as create_t2_1_,
>        userinfo0_.create_user_id        as create_u3_1_,
>        userinfo0_.last_modified_time    as last_mod4_1_,
>        userinfo0_.last_modified_user_id as last_mod5_1_,
>        userinfo0_.version               as version6_1_,
>        userinfo0_.ages                  as ages7_1_,
>        userinfo0_.email_address         as email_ad8_1_,
>        userinfo0_.last_name             as last_nam9_1_,
>        userinfo0_.name                  as name10_1_,
>        userinfo0_.telephone             as telepho11_1_
> from user_info userinfo0_ org.hibernate.SQL                        :
> select addresslis0_.user_info_id          as user_inf8_0_0_,
>        addresslis0_.id                    as id1_0_0_,
>        addresslis0_.id                    as id1_0_1_,
>        addresslis0_.create_time           as create_t2_0_1_,
>        addresslis0_.create_user_id        as create_u3_0_1_,
>        addresslis0_.last_modified_time    as last_mod4_0_1_,
>        addresslis0_.last_modified_user_id as last_mod5_0_1_,
>        addresslis0_.version               as version6_0_1_,
>        addresslis0_.city                  as city7_0_1_,
>        addresslis0_.user_info_id          as user_inf8_0_1_
> from address addresslis0_
> where addresslis0_.user_info_id = ? org.hibernate.SQL                        :
> select addresslis0_.user_info_id          as user_inf8_0_0_,
>        addresslis0_.id                    as id1_0_0_,
>        addresslis0_.id                    as id1_0_1_,
>        addresslis0_.create_time           as create_t2_0_1_,
>        addresslis0_.create_user_id        as create_u3_0_1_,
>        addresslis0_.last_modified_time    as last_mod4_0_1_,
>        addresslis0_.last_modified_user_id as last_mod5_0_1_,
>        addresslis0_.version               as version6_0_1_,
>        addresslis0_.city                  as city7_0_1_,
>        addresslis0_.user_info_id          as user_inf8_0_1_
> from address addresslis0_
> where addresslis0_.user_info_id = ? org.hibernate.SQL                        :
> select addresslis0_.user_info_id          as user_inf8_0_0_,
>        addresslis0_.id                    as id1_0_0_,
>        addresslis0_.id                    as id1_0_1_,
>        addresslis0_.create_time           as create_t2_0_1_,
>        addresslis0_.create_user_id        as create_u3_0_1_,
>        addresslis0_.last_modified_time    as last_mod4_0_1_,
>        addresslis0_.last_modified_user_id as last_mod5_0_1_,
>        addresslis0_.version               as version6_0_1_,
>        addresslis0_.city                  as city7_0_1_,
>        addresslis0_.user_info_id          as user_inf8_0_1_
> from address addresslis0_
> where addresslis0_.user_info_id = ?
> ```
>
> 通过 SQL 我们可以看得出来，当取 UserInfo 的时候，有多少条 UserInfo 数据就会触发多少条查询 Address 的 SQL。
>
> 那么所谓的 N+1 的 SQL，此时 1 代表的是一条 SQL 查询 UserInfo 信息；N 条 SQL 查询 Address 的信息。你可以想象一下，如果有 100 条 UserInfo 信息，可能会触发 100 条查询 Address 的 SQL，性能多差呀。
>
> 很简单，这就是我们常说的 N+1 SQL 问题。我们这里使用的是 EAGER 模式，当使用 LAZY 的时候也是一样的道理，只是生成 N 条 SQL 的时机是不一样的。
>
> 上面我演示了 @OneToMany 的情况，那么我们再看一下 @ManyToOne 的情况。利用 AddressRepository 查询所有的 Address 信息，方法如下面这行代码所示。
>
> 复制代码
>
> ```
> addressRepository.findAll();
> ```
>
> 这个时候我们再看一下控制台，会产生如下 SQL。
>
> 复制代码
>
> ```
> org.hibernate.SQL                        :
> select address0_.id                    as id1_0_,
>        address0_.create_time           as create_t2_0_,
>        address0_.create_user_id        as create_u3_0_,
>        address0_.last_modified_time    as last_mod4_0_,
>        address0_.last_modified_user_id as last_mod5_0_,
>        address0_.version               as version6_0_,
>        address0_.city                  as city7_0_,
>        address0_.user_info_id          as user_inf8_0_
> from address address0_ 
> org.hibernate.SQL                        :
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id = ? 
> org.hibernate.SQL                        :
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id = ? 
> org.hibernate.SQL                        :
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id = ?
> ```
>
> 这里通过 SQL 我们可以看得出来，当取 Address 的时候，Address 里面有多少个 user_info_id，就会触发多少条查询 UserInfo 的 SQL。
>
> 那么所谓的 N+1 的 SQL，此时 1 就代表一条 SQL 查询 Address 信息；N 条 SQL 查询 UserInfo 的信息。同样，你可以想象一下，如果我们有 100 条 Address 信息，分别有不同的 user_info_id 可能会触发 100 条查询 UserInfo 的 SQL，性能依然很差。
>
> 这也是我们常说的 N+1 SQL 问题，我只是给你演示了 @OneToMany 和 @ManyToOne 的情况，@ManyToMany 和 @OneToOne 也是一样的道理，都是当我们查询主体信息时候，1 条 SQL 会衍生出来关联关系的 N 条 SQL。
>
> 现在你认识了这个问题，下一步该思考，怎么解决才更合理呢？有没有什么办法可以减少 SQL 条数呢？
>
> ### 减少 N+1 SQL 的条数
>
> 最容易想到，就是有没有什么机制可以减少 N 对应的 SQL 条数呢？从原理分析会知道，不管是 LAZY 还是 EAGER 都是没有用的，因为这两个只是决定了 N 条 SQL 的触发时机，而不能减少 SQL 的条数。
>
> 不知道你是否还记得在[第 20 讲（Spring JPA 中的 Hibernate 加载过程与配置项是怎么回事）](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490#/detail/pc?id=4720)中，我们介绍过的 Hibernate 的配置项有哪些，如果你回过头去看，会发现有个配置可以改变每次批量取数据的大小。
>
> #### hibernate.default_batch_fetch_size 配置
>
> hibernate.default_batch_fetch_size 配置在 AvailableSettings.class 里面，指的是批量获取数据的大小，默认是 -1，表示默认没有匹配取数据。那么我们把这个值改成 20 看一下效果，只需要在 application.properties 里面增加如下配置即可。
>
> 复制代码
>
> ```
> # 更改批量取数据的大小为20
> spring.jpa.properties.hibernate.default_batch_fetch_size= 20
> ```
>
> 在实体类不发生任何改变的前提下，我们再执行如下两个方法，分别看一下 SQL 的生成情况。
>
> 复制代码
>
> ```
> userInfoRepository.findAll();
> ```
>
> 还是先查询所有的 UserInfo 信息，看一下 SQL 的执行情况，代码如下所示。
>
> 复制代码
>
> ```
> org.hibernate.SQL                        :
> select userinfo0_.id                    as id1_1_,
>        userinfo0_.create_time           as create_t2_1_,
>        userinfo0_.create_user_id        as create_u3_1_,
>        userinfo0_.last_modified_time    as last_mod4_1_,
>        userinfo0_.last_modified_user_id as last_mod5_1_,
>        userinfo0_.version               as version6_1_,
>        userinfo0_.ages                  as ages7_1_,
>        userinfo0_.email_address         as email_ad8_1_,
>        userinfo0_.last_name             as last_nam9_1_,
>        userinfo0_.name                  as name10_1_,
>        userinfo0_.telephone             as telepho11_1_
> from user_info userinfo0_ org.hibernate.SQL                        :
> select addresslis0_.user_info_id          as user_inf8_0_1_,
>        addresslis0_.id                    as id1_0_1_,
>        addresslis0_.id                    as id1_0_0_,
>        addresslis0_.create_time           as create_t2_0_0_,
>        addresslis0_.create_user_id        as create_u3_0_0_,
>        addresslis0_.last_modified_time    as last_mod4_0_0_,
>        addresslis0_.last_modified_user_id as last_mod5_0_0_,
>        addresslis0_.version               as version6_0_0_,
>        addresslis0_.city                  as city7_0_0_,
>        addresslis0_.user_info_id          as user_inf8_0_0_
> from address addresslis0_
> where addresslis0_.user_info_id in (?, ?, ?)
> ```
>
> 我们可以看到 SQL 直接减少到两条了，其中查询 Address 的地方查询条件变成了 in(?,?,?)。
>
> 想象一下，如果我们有 20 条 UserInfo 信息，那么产生的 SQL 也是两条，此时要比 20+1 条 SQL 性能高太多了。
>
> 接着我们再执行另一个方法，看一下 @ManyToOne 的情况，代码如下所示。
>
> 复制代码
>
> ```
> addressRepository.findAll()
> ```
>
> 关于执行的 SQL 情况如下所示。
>
> 复制代码
>
> ```
> 2020-11-29 23:11:27.381 DEBUG 30870 --- [nio-8087-exec-5] org.hibernate.SQL                        : 
> select address0_.id                    as id1_0_,
>        address0_.create_time           as create_t2_0_,
>        address0_.create_user_id        as create_u3_0_,
>        address0_.last_modified_time    as last_mod4_0_,
>        address0_.last_modified_user_id as last_mod5_0_,
>        address0_.version               as version6_0_,
>        address0_.city                  as city7_0_,
>        address0_.user_info_id          as user_inf8_0_
> from address address0_
> 2020-11-29 23:11:27.383 DEBUG 30870 --- [nio-8087-exec-5] org.hibernate.SQL                        : 
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id in (?, ?, ?)
> ```
>
> 从代码中可以看到，我们查询所有的 Address 信息也只产生了 2 条 SQL；而当我们查询 UserInfo 的时候，SQL 最后的查询条件也变成了 in (？，？，？)，同样的道理这样也会提升不少性能。
>
> 而 hibernate.default_batch_fetch_size 的经验参考值，可以设置成 20、30、50、100 等，太高了也没有意义。一个请求执行一次，产生的 SQL 数量为 3-5 条基本上都算合理情况，这样通过设置 default_batch_fetch_size 就可以很好地避免大部分业务场景下的 N+1 条 SQL 的性能问题了。
>
> 此时你还需要注意一点就是，在实际工作中，一定要知道我们一次操作会产生多少 SQL，有没有预期之外的 SQL 参数，这是需要关注的重点，这种情况可以利用我们之前说过的如下配置来开启打印 SQL，请看代码。
>
> 复制代码
>
> ```
> ## 显示sql的执行日志，如果开了这个,show_sql就可以不用了，show_sql没有上下文，多线程情况下，分不清楚是谁打印的，所有我推荐如下配置项：
> logging.level.org.hibernate.SQL=debug
> ```
>
> 但是这种配置也有个缺陷，就是只能全局配置，没办法针对不通过的实体管理关系配置不同的 Fetch Size 的值。
>
> 而与之类似的 Hibernate 里面也提供了一个注解 @BatchSize 可以解决此问题。
>
> #### @BatchSize 注解
>
> @BatchSize 注解是 Hibernate 提供的用来解决查询关联关系的批量处理大小，默认无，可以配置在实体上，也可以配置在关联关系上面。此注解里面只有一个属性 size，用来指定关联关系 LAZY 或者是 EAGER 一次性取数据的大小。
>
> 我们还是将上面的例子中的 UserInfo 实体做一下改造，在里面增加两次 @BatchSize 注解，代码如下所示。
>
> 复制代码
>
> ```
> @Entity
> @Data
> @SuperBuilder
> @AllArgsConstructor
> @NoArgsConstructor
> @Table
> @ToString(exclude = "addressList")
> @BatchSize(size = 2)//实体类上加@BatchSize注解，用来设置当被关联关系的时候一次查询的大小，我们设置成2，方便演示Address关联UserInfo的时候的效果
> public class UserInfo extends BaseEntity {
>    private String name;
>    private String telephone;
>    @OneToMany(mappedBy = "userInfo",cascade = CascadeType.PERSIST,fetch = FetchType.EAGER)
>    @BatchSize(size = 20)//关联关系的属性上加@BatchSize注解，用来设置当通过UserInfo加载Address的时候一次取数据的大小
>    private List<Address> addressList;
> }
> ```
>
> 我们通过改造 UserInfo 实体，可以直接演示 @BatchSize 应用在实体类和属性字段上的效果，所以 Address 实体可以不做任何改变，hibernate.default_batch_fetch_size 还改成默认值 -1，我们再分别执行一下两个 findAll 方法，看一下效果。
>
> 第一种：查询所有 UserInfo，代码如下面这行所示。
>
> 复制代码
>
> ```
> userInfoRepository.findAll()
> ```
>
> 我们看一下 SQL 控制台。
>
> 复制代码
>
> ```
>  org.hibernate.SQL                        :
> select userinfo0_.id                    as id1_1_,
>        userinfo0_.create_time           as create_t2_1_,
>        userinfo0_.create_user_id        as create_u3_1_,
>        userinfo0_.last_modified_time    as last_mod4_1_,
>        userinfo0_.last_modified_user_id as last_mod5_1_,
>        userinfo0_.version               as version6_1_,
>        userinfo0_.ages                  as ages7_1_,
>        userinfo0_.email_address         as email_ad8_1_,
>        userinfo0_.last_name             as last_nam9_1_,
>        userinfo0_.name                  as name10_1_,
>        userinfo0_.telephone             as telepho11_1_
> from user_info userinfo0_ org.hibernate.SQL                        :
> select addresslis0_.user_info_id          as user_inf8_0_1_,
>        addresslis0_.id                    as id1_0_1_,
>        addresslis0_.id                    as id1_0_0_,
>        addresslis0_.create_time           as create_t2_0_0_,
>        addresslis0_.create_user_id        as create_u3_0_0_,
>        addresslis0_.last_modified_time    as last_mod4_0_0_,
>        addresslis0_.last_modified_user_id as last_mod5_0_0_,
>        addresslis0_.version               as version6_0_0_,
>        addresslis0_.city                  as city7_0_0_,
>        addresslis0_.user_info_id          as user_inf8_0_0_
> from address addresslis0_
> where addresslis0_.user_info_id in (?, ?, ?)
> ```
>
> 和刚才设置 hibernate.default_batch_fetch_size=20 的效果一模一样，所以我们可以利用 @BatchSize 这个注解针对不同的关联关系，配置不同的大小，从而提升 N+1 SQL 的性能。
>
> 第二种：查询一下所有 Address，如下面这行代码所示。
>
> 复制代码
>
> ```
> addressRepository.findAll();
> ```
>
> 我们看一下控制台的 SQL 情况，如下所示。
>
> 复制代码
>
> ```
> org.hibernate.SQL                        :
> select address0_.id                    as id1_0_,
>        address0_.create_time           as create_t2_0_,
>        address0_.create_user_id        as create_u3_0_,
>        address0_.last_modified_time    as last_mod4_0_,
>        address0_.last_modified_user_id as last_mod5_0_,
>        address0_.version               as version6_0_,
>        address0_.city                  as city7_0_,
>        address0_.user_info_id          as user_inf8_0_
> from address address0_ 
> org.hibernate.SQL                        :
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id in (?, ?) 
> org.hibernate.SQL                        :
> select userinfo0_.id                      as id1_1_0_,
>        userinfo0_.create_time             as create_t2_1_0_,
>        userinfo0_.create_user_id          as create_u3_1_0_,
>        userinfo0_.last_modified_time      as last_mod4_1_0_,
>        userinfo0_.last_modified_user_id   as last_mod5_1_0_,
>        userinfo0_.version                 as version6_1_0_,
>        userinfo0_.ages                    as ages7_1_0_,
>        userinfo0_.email_address           as email_ad8_1_0_,
>        userinfo0_.last_name               as last_nam9_1_0_,
>        userinfo0_.name                    as name10_1_0_,
>        userinfo0_.telephone               as telepho11_1_0_,
>        addresslis1_.user_info_id          as user_inf8_0_1_,
>        addresslis1_.id                    as id1_0_1_,
>        addresslis1_.id                    as id1_0_2_,
>        addresslis1_.create_time           as create_t2_0_2_,
>        addresslis1_.create_user_id        as create_u3_0_2_,
>        addresslis1_.last_modified_time    as last_mod4_0_2_,
>        addresslis1_.last_modified_user_id as last_mod5_0_2_,
>        addresslis1_.version               as version6_0_2_,
>        addresslis1_.city                  as city7_0_2_,
>        addresslis1_.user_info_id          as user_inf8_0_2_
> from user_info userinfo0_
>          left outer join address addresslis1_ on userinfo0_.id = addresslis1_.user_info_id
> where userinfo0_.id = ?
> ```
>
> 这里可以看到，由于我们在 UserInfo 的实体上设置了 @BatchSize(size = 2)，表示所有关联关系到 UserInfo 的时候一次取两条数据，所以就会发现这次我查询 Address 加载 UserInfo 的时候，产生了 3 条 SQL。
>
> 其中通过关联关系查询 UserInfo 产生了 2 条 SQL，由于我们 UserInfo 在数据库里面有三条数据，所以第一条 UserInfo 的 SQL 受 @BatchSize(size = 2) 控制，从而 in (?,?) 只支持了两个参数，同时也产生了第二条查 UserInfo 的 SQL。
>
> 从上面的例子中我们可以看到 @BatchSize 和 hibernate.default_batch_fetch_size 的效果是一样的，只不过一个是全局配置、一个是局部设置，这是可以减少 N+1 SQL 最直接、最方便的两种方式。
>
> **注意事项：**
>
> @BatchSize 的使用具有局限性，不能作用于 @ManyToOne 和 @OneToOne 的关联关系上，那样代码是不起作用的，如下所示。
>
> 复制代码
>
> ```
> public class Address extends BaseEntity {
>    private String city;
>    @ManyToOne(cascade = CascadeType.PERSIST,fetch = FetchType.EAGER)
>    @BatchSize(size = 30) //由于是@ManyToOne的关联关系所有没有作用
>    private UserInfo userInfo;
> }
> ```
>
> 因此，你要注意 @BatchSize 只能作用在 @ManyToMany、@OneToMany、实体类这三个地方。
>  此外，Hibernate 中还提供了一种 FetchMode 的策略，包含三种模式，分别为 FetchMode.SELECT、FetchMode.JOIN，以及 FetchMode.Subselect。由于内容较多，我怕你一次性不好消化，所以会在下一讲继续为你介绍。到时见。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa