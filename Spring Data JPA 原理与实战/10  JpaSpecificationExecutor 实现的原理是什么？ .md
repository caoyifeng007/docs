[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4710)



> 通过上一课时，我们了解到 JpaSpecificationExecutor 给我们提供了动态查询或者写框架的一种思路，那么这节课我们来看一下 JpaSpecificationExecutor 的详细用法和原理，及其实战应用场景中如何实现自己的框架。
>
> 在开始讲解之前，请先思考几个问题：
>
> 1. JpaSpecificationExecutor 如何创建？
> 2. 它的使用方法有哪些？
> 3. toPredicate 方法如何实现？
>
> 带着这些问题，我们开始探索。先看一个例子感受一下 JpaSpecificationExecutor 具体的用法。
>
> ### JpaSpecificationExecutor 使用案例
>
> 我们假设一个后台管理页面根据 name 模糊查询、sex 精准查询、age 范围查询、时间区间查询、address 的 in 查询这样一个场景，来查询 user 信息，我们看看这个例子应该怎么写。
>
> **第一步：创建 User 和 UserAddress 两个实体。** 代码如下：
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import com.fasterxml.jackson.annotation.JsonIgnore;
> import lombok.*;
> import javax.persistence.*;
> import java.io.Serializable;
> import java.time.Instant;
> import java.util.Date;
> import java.util.List;
> /**
> * 用户基本信息表
> **/
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "addresses")
> public class User implements Serializable {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String name;
>    private String email;
>    @Enumerated(EnumType.STRING)
>    private SexEnum sex;
>    private Integer age;
>    private Instant createDate;
>    private Date updateDate;
>    @OneToMany(mappedBy = "user")
>    @JsonIgnore
>    private List<UserAddress> addresses;
> }
> enum SexEnum {
>    BOY,GIRL
> }
> package com.example.jpa.example1;
> import lombok.*;
> import javax.persistence.*;
> /**
>  * 用户地址表
>  */
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> @ToString(exclude = "user")
> public class UserAddress {
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String address;
>    @ManyToOne(cascade = CascadeType.ALL)
>    private User user;
> }
> ```
>
> **第二步：创建 UserRepository 继承 JpaSpecificationExecutor 接口。**
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import org.springframework.data.jpa.repository.JpaRepository;
> import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
> public interface UserRepository extends JpaRepository<User,Long>, JpaSpecificationExecutor<User> {
> }
> ```
>
> **第三步：创建一个测试用例进行测试。**
>
> 复制代码
>
> ```
> @DataJpaTest
> @TestInstance(TestInstance.Lifecycle.PER_CLASS)
> public class UserJpeTest {
>     @Autowired
>     private UserRepository userRepository;
>     @Autowired
>     private UserAddressRepository userAddressRepository;
>     private Date now = new Date();
>     /**
>      * 提前创建一些数据
>      */
>     @BeforeAll
>     @Rollback(false)
>     @Transactional
>     void init() {
>         User user = User.builder()
>                 .name("jack")
>                 .email("123456@126.com")
>                 .sex(SexEnum.BOY)
>                 .age(20)
>                 .createDate(Instant.now())
>                 .updateDate(now)
>                 .build();
>         userAddressRepository.saveAll(Lists.newArrayList(UserAddress.builder().user(user).address("shanghai").build(),
>                 UserAddress.builder().user(user).address("beijing").build()));
>     }
>     @Test
>     public void testSPE() {
>         //模拟请求参数
>         User userQuery = User.builder()
>                 .name("jack")
>                 .email("123456@126.com")
>                 .sex(SexEnum.BOY)
>                 .age(20)
>                 .addresses(Lists.newArrayList(UserAddress.builder().address("shanghai").build()))
>                 .build();
>                 //假设的时间范围参数
>         Instant beginCreateDate = Instant.now().plus(-2, ChronoUnit.HOURS);
>         Instant endCreateDate = Instant.now().plus(1, ChronoUnit.HOURS);
>         //利用Specification进行查询
>         Page<User> users = userRepository.findAll(new Specification<User>() {
>             @Override
>             public Predicate toPredicate(Root<User> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
>                 List<Predicate> ps = new ArrayList<Predicate>();
>                 if (StringUtils.isNotBlank(userQuery.getName())) {
>                     //我们模仿一下like查询，根据name模糊查询
>                     ps.add(cb.like(root.get("name"),"%" +userQuery.getName()+"%"));
>                 }
>                 if (userQuery.getSex()!=null){
>                     //equal查询条件，这里需要注意，直接传递的是枚举
>                     ps.add(cb.equal(root.get("sex"),userQuery.getSex()));
>                 }
>                 if (userQuery.getAge()!=null){
>                     //greaterThan大于等于查询条件
>                     ps.add(cb.greaterThan(root.get("age"),userQuery.getAge()));
>                 }
>                 if (beginCreateDate!=null&&endCreateDate!=null){
>                     //根据时间区间去查询创建
>                     ps.add(cb.between(root.get("createDate"),beginCreateDate,endCreateDate));
>                 }
>                 if (!ObjectUtils.isEmpty(userQuery.getAddresses())) {
>                     //联表查询，利用root的join方法，根据关联关系表里面的字段进行查询。
>                     ps.add(cb.in(root.join("addresses").get("address")).value(userQuery.getAddresses().stream().map(a->a.getAddress()).collect(Collectors.toList())));
>                 }
>                 return query.where(ps.toArray(new Predicate[ps.size()])).getRestriction();
>             }
>         }, PageRequest.of(0, 2));
>         System.out.println(users);
>     }
> }
> ```
>
> 我们看一下执行结果。
>
> 复制代码
>
> ```
> Hibernate: select user0_.id as id1_1_, user0_.age as age2_1_, user0_.create_date as create_d3_1_, user0_.email as email4_1_, user0_.name as name5_1_, user0_.sex as sex6_1_, user0_.update_date as update_d7_1_ from user user0_ inner join user_address addresses1_ on user0_.id=addresses1_.user_id where (user0_.name like ?) and user0_.sex=? and user0_.age>20 and (user0_.create_date between ? and ?) and (addresses1_.address in (?)) limit ?
> ```
>
> 此 SQL 的参数如下：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw4GAVwjvAACBlHuTFN0497.png)
>
> 此 SQL 就是查询 User inner Join user_address 之后组合成的查询 SQL，基本符合我们的预期，即不同的查询条件。我们通过这个例子大概知道了 JpaSpecificationExecutor 的用法，那么它具体是什么呢？
>
> ### JpaSpecificationExecutor 语法详解
>
> 我们依然通过看 JpaSpecificationExecutor 的源码，来了解一下它的几个使用方法，如下所示：
>
> 复制代码
>
> ```
> public interface JpaSpecificationExecutor<T> {
>    //根据 Specification 条件查询单个对象，要注意的是，如果条件能查出来多个会报错
>    T findOne(@Nullable Specification<T> spec);
>    //根据 Specification 条件，查询 List 结果
>    List<T> findAll(@Nullable Specification<T> spec);
>    //根据 Specification 条件，分页查询
>    Page<T> findAll(@Nullable Specification<T> spec, Pageable pageable);
>    //根据 Specification 条件，带排序的查询结果
>    List<T> findAll(@Nullable Specification<T> spec, Sort sort);
>    //根据 Specification 条件，查询数量
>    long count(@Nullable Specification<T> spec);
> }
> ```
>
> 其返回结果和 Pageable、Sort，我们在前面课时都有介绍过，这里我们重点关注一下 Specification。看一下 Specification 接口的代码。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw4uAGaD2AADHMlVN_q0440.png)
>
> 通过看其源码就会发现里面提供的方法很简单。其中，下面一段代码表示组合的 and 关系的查询条件。
>
> 复制代码
>
> ```
> default Specification<T> and(@Nullable Specification<T> other) {
>    return composed(this, other, (builder, left, rhs) -> builder.and(left, rhs));
> }
> ```
>
> 下面是静态方法，创建 where 后面的 Predicate 集合。
>
> 复制代码
>
> ```
> static <T> Specification<T> where(@Nullable Specification<T> spec)
> ```
>
> 下面是默认方法，创建 or 条件的查询参数。
>
> 复制代码
>
> ```
> default Specification<T> or(@Nullable Specification<T> other)
> ```
>
> 这是静态方法，创建 Not 的查询条件。
>
> 复制代码
>
> ```
> static <T> Specification<T> not(@Nullable Specification<T> spec)
> ```
>
> 上面这几个方法比较简单，我就不一一细说了，我们主要看一下需要实现的方法：toPredicate。
>
> 复制代码
>
> ```
> Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
> ```
>
> toPredicate 这个方法是我们用到的时候需要自己去实现的，接下来我们详细介绍一下。
>
> 首先我们在刚才的 Demo 里面设置一个断点，看到如下界面。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw5SAeoXKAAMzN3mF_8I181.png)
>
> 这里可以分别看到 Root 的实现类是 RootImpl，CriteriaQuery 的实现类是 CriteriaQueryImpl，CriteriaBuilder 的实现类是 CriteriaBuilderImpl。
>
> 复制代码
>
> ```
> javax.persistence.criteria.Root
> javax.persistence.criteria.CriteriaQuery
> javax.persistence.criteria.CriteriaBuilder
> ```
>
> 其中，上面三个接口是 Java Persistence API 定义的接口。
>
> 复制代码
>
> ```
> org.hibernate.query.criteria.internal.path.RootImpl
> rg.hibernate.query.criteria.internal.CriteriaQueryImpl
> org.hibernate.query.criteria.internal.CriteriaBuilderImpl
> ```
>
> 而这个三个实现类都是由 Hibernate 进行实现的，也就是说 JpaSpecificationExecutor 封装了原本需要我们直接操作 Hibernate 中 Criteria 的 API 方法。
>
> 下面分别解释上述三个参数。
>
> #### Root`<User>` root
>
> 代表了可以查询和操作的实体对象的根，如果将实体对象比喻成表名，那 root 里面就是这张表里面的字段，而这些字段只是 JPQL 的实体字段而已。我们可以通过里面的 Path get（String attributeName），来获得我们想要操作的字段。
>
> 复制代码
>
> ```
> 类似于我们上面的：root.get("createDate")等操作
> ```
>
> #### CriteriaQuery<?> query
>
> 代表一个 specific 的顶层查询对象，它包含着查询的各个部分，比如 select 、from、where、group by、order by 等。CriteriaQuery 对象只对实体类型或嵌入式类型的 Criteria 查询起作用。简单理解为，它提供了查询 ROOT 的方法。常用的方法有如下几种：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw6uAJmdyAALVLoSlnLQ418.png)
>
> 复制代码
>
> ```
> 正如我们上面where的用法：query.where(.....）一样
> ```
>
> 这个语法比较简单，我们在其方法后面加上相应的参数即可。下面看一个 group by 的例子。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/5E/D2/CgqCHl-Hw7SAYWLnAAEIg0u_SOc872.png)
>
> 如上图所示，我们加入 groupyBy 某个字段，SQL 也会有相应的变化。那么我们再来看第三个参数。
>
> #### CriteriaBuilder cb
>
> CriteriaBuilder 是用来构建 CritiaQuery 的构建器对象，其实就相当于条件或者条件组合，并以 Predicate 的形式返回。它基本上提供了所有常用的方法，如下所示：
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw7yAZgOYAASXGLn8fS0352.png)
>
> 我们直接通过此类的 Structure 视图就可以看到都有哪些方法。其中，and、any 等用来做查询条件的组合；类似 between、equal、exist、ge、gt、isEmpty、isTrue、in 等用来做查询条件的查询，类似下图的一些方法。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/5E/D3/CgqCHl-Hw8KAS41dAAKQ7MNjups722.png)
>
> 而其中 Expression 很简单，都是通过 root.get(...) 某些字段即可返回，正如下面的用法。
>
> 复制代码
>
> ```
> Predicate p1=cb.like(root.get(“name”).as(String.class), “%”+uqm.getName()+“%”);
> Predicate p2=cb.equal(root.get("uuid").as(Integer.class), uqm.getUuid());
> Predicate p3=cb.gt(root.get("age").as(Integer.class), uqm.getAge());
> ```
>
> 我们利用 like、equal、gt 可以生产 Predicate，而 Predicate 可以组合查询。比如我们预定它们之间是 and 或 or 的关系：`Predicate p = cb.and(p3,cb.or(p1,p2));`
>
> 我们让 p1 和 p2 是 or 的关系，并且得到的 Predicate 和 p3 又构成了 and 的关系。你可以发现它的用法还是比较简单的，正如我们开篇所说的 Junit 中 test 里面一样的写法。
>
> 关于 JpaSpecificationExecutor 的语法我们就介绍完了，其实它里面的功能相当强大，只是我们发现 Spring Data JPA 介绍得并不详细，只是一笔带过，可能他们认为写框架的人才能用到，所以介绍得不多。
>
> 如果你想了解更多语法的话，可以参考 Hibernate 的文档：https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#criteria。我们再来看看 JpaSpecificationExecutor 的实现原理。
>
> ### JpaSpecificationExecutor 原理分析
>
> 我们先看一下 JpaSpecificationExecutor 的类图。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw9KAPLUnAAEGAy5X1Y8192.png)
>
> 从图中我们可以看得出来：
>
> 1. JpaSpecificationExecutor 和 JpaRepository 是平级接口，而它们对应的实现类都是 SimpleJpaRepository；
> 2. Specification 被 ExampleSpecification 和 JpaSpecificationExector 使用，用来创建查询；
> 3. Predicate 是 JPA 协议里面提供的查询条件的根基；
> 4. SimpleJpaRepository 利用 EntityManager 和 criteria 来实现由 JpaSpecificationExector 组合的 query。
>
> 那么我们再直观地看一下 JpaSpecificationExecutor 接口里面的方法 findAll 对应的 SimpleJpaRepository 里面的实现方法 findAl，我们通过工具可以很容易地看到相应的实现方法，如下所示：
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/5E/D3/CgqCHl-Hw9uAHAiHAAFZTdcBw2Y564.png)
>
> 你要知道，得到 TypeQuery 就可以直接操作JPA协议里面相应的方法了，那么我们看下 getQuery（spec，pageable）的实现过程。
>
> ![Drawing 9.png](https://s0.lgstatic.com/i/image/M00/5E/C7/Ciqc1F-Hw-qAcTArAACqAeHRq3M302.png)
>
> 之后一步一步 debug 就可以了。
>
> ![Drawing 10.png](https://s0.lgstatic.com/i/image/M00/5E/D3/CgqCHl-Hw_KAU1wCAAW1BokuNt8728.png)
>
> 到了上图所示这里，就可以看到：
>
> 1. Specification`<S>` spec 是我们测试用例写的 specification 的匿名实现类；
> 2. 由于是方法传递，所以到第 693 行断点的时候，才会执行我们在测试用里面写的 Specification；
> 3. 我们可以看到这个方法最后是调用的 EntityManager，而 EntitytManger 是 JPA 操作实体的核心原理，我在下一课时讲自定义 Repsitory 的时候再详细介绍；
> 4. 从上面的方法实现过程中我们可以看得出来，所谓的JpaSpecificationExecutor原理，用一句话概况，就是利用Java Persistence API定义的接口和Hibernate的实现，做了一个简单的封装，方便我们操作JPA协议中 criteria 的相关方法。
>
> 到这里，原理和使用方法，我们基本介绍完了。你可能会有疑问：这个感觉有点重要，但是一般用不到吧？那么接下来我们看看 JpaSpecificationExecutor 的实战应用场景是什么样的。
>
> ### JpaSpecificationExecutor 实战应用场景
>
> 其实JpaSpecificationExecutor 的目的不是让我们做日常的业务查询，而是给我们提供了一种自定义 Query for rest 的架构思路，如果做日常的增删改查，肯定不如我们前面介绍的 Defining Query Methods 和 @Query 方便。
>
> 那么来看下，实战过程中如何利用 JpaSpecificationExecutor 写一个框架。
>
> #### MySpecification 自定义
>
> 我们可以自定义一个Specification 的实现类，它可以实现任何实体的动态查询和各种条件的组合。
>
> 复制代码
>
> ```
> package com.example.jpa.example1.spe;
> import org.springframework.data.jpa.domain.Specification;
> import javax.persistence.criteria.*;
> public class MySpecification<Entity> implements Specification<Entity> {
>    private SearchCriteria criteria;
>    public MySpecification (SearchCriteria criteria) {
>       this.criteria = criteria;
>    }
>    /**
>     * 实现实体根据不同的字段、不同的Operator组合成不同的Predicate条件
>     *
>     * @param root            must not be {@literal null}.
>     * @param query           must not be {@literal null}.
>     * @param builder  must not be {@literal null}.
>     * @return a {@link Predicate}, may be {@literal null}.
>     */
>    @Override
>    public Predicate toPredicate(Root<Entity> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
>       if (criteria.getOperation().compareTo(Operator.GT)==0) {
>          return builder.greaterThanOrEqualTo(
>                root.<String> get(criteria.getKey()), criteria.getValue().toString());
>       }
>       else if (criteria.getOperation().compareTo(Operator.LT)==0) {
>          return builder.lessThanOrEqualTo(
>                root.<String> get(criteria.getKey()), criteria.getValue().toString());
>       }
>       else if (criteria.getOperation().compareTo(Operator.LK)==0) {
>          if (root.get(criteria.getKey()).getJavaType() == String.class) {
>             return builder.like(
>                   root.<String>get(criteria.getKey()), "%" + criteria.getValue() + "%");
>          } else {
>             return builder.equal(root.get(criteria.getKey()), criteria.getValue());
>          }
>       }
>       return null;
>    }
> }
> ```
>
> 我们通过 `<Entity>` 泛型，解决不同实体的动态查询（当然了，我只是举个例子，这个方法可以进行无限扩展）。我们通过 SearchCriteria 可以知道不同的字段是什么、值是什么、如何操作的，看一下代码：
>
> 复制代码
>
> ```
> package com.example.jpa.example1.spe;
> import lombok.*;
> /**
>  * @author jack，实现不同的查询条件，不同的操作，针对Value;
>  */
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> public class SearchCriteria {
>    private String key;
>    private Operator operation;
>    private Object value;
> }
> ```
>
> 其中的 Operator 也是我们自定义的。
>
> 复制代码
>
> ```
> package com.example.jpa.example1.spe;
> public enum Operator {
>    /**
>     * 等于
>     */
>    EQ("="),
>    /**
>     * 等于
>     */
>    LK(":"),
>    /**
>     * 不等于
>     */
>    NE("!="),
>    /**
>     * 大于
>     */
>    GT(">"),
>    /**
>     * 小于
>     */
>    LT("<"),
>    /**
>     * 大于等于
>     */
>    GE(">=");
>    Operator(String operator) {
>       this.operator = operator;
>    }
>    private String operator;
> }
> ```
>
> 在 Operator 枚举里面定义了逻辑操作符（大于、小于、不等于、等于、大于等于……也可以自己扩展），并在 MySpecification 里面进行实现。那么我们来看看它是怎么用的，写一个测试用例试一下。
>
> 复制代码
>
> ```
> /**
>  * 测试自定义的Specification语法
>  */
> @Test
> public void givenLast_whenGettingListOfUsers_thenCorrect() {
>     MySpecification<User> name =
>         new MySpecification<User>(new SearchCriteria("name", Operator.LK, "jack"));
> MySpecification<User> age =
>         new MySpecification<User>(new SearchCriteria("age", Operator.GT, 2));
> List<User> results = userRepository.findAll(Specification.where(name).and(age));
>     System.out.println(results.get(0).getName());
> }
> ```
>
> 你就会发现，我们在调用findAll 组合 Predicate 的时候就会非常简单，省去了各种条件的判断和组合，而省去的这些逻辑可以全部在我们的框架代码 MySpecification 里面实现。
>
> 那么如果把这个扩展到 API 接口层面会是什么样的结果呢？我们来看下。
>
> #### 利用 Specification 创建 search 为查询条件的 Rest API 接口
>
> 先创建一个 Controller，用来接收 search 这样的查询条件：类似 userssearch=lastName:doe,age>25 的参数。
>
> 复制代码
>
> ```
> package com.example.jpa.example1.web;
> import com.example.jpa.example1.User;
> import com.example.jpa.example1.UserRepository;
> import com.example.jpa.example1.spe.SpecificationsBuilder;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.data.jpa.domain.Specification;
> import org.springframework.web.bind.annotation.*;
> import java.util.List;
> @RestController
> public class UserController {
>     @Autowired
>     private UserRepository repo;
>     @RequestMapping(method = RequestMethod.GET, value = "/users")
>     @ResponseBody
>     public List<User> search(@RequestParam(value = "search") String search) {
>         Specification<User> spec = new SpecificationsBuilder<User>().buildSpecification(search);
>         return repo.findAll(spec);
>     }
> }
> ```
>
> Controller 里面非常简单，利用 SpecificationsBuilder 生成我们需要的 Specification 即可。那么我们看看 SpecificationsBuilder 里面是怎么写的。
>
> 复制代码
>
> ```
> package com.example.jpa.example1.spe;
> import com.example.jpa.example1.User;
> import org.springframework.data.jpa.domain.Specification;
> import java.util.ArrayList;
> import java.util.List;
> import java.util.regex.Matcher;
> import java.util.regex.Pattern;
> import java.util.stream.Collectors;
> /**
>  * 处理请求参数
>  * @param <Entity>
>  */
> public class SpecificationsBuilder<Entity> {
>    private final List<SearchCriteria> params;
> 
>    //初始化params，保证每次实例都是一个新的ArrayList
>    public SpecificationsBuilder() {
>       params = new ArrayList<SearchCriteria>();
>    }
> 
>    //利用正则表达式取我们search参数里面的值，解析成SearchCriteria对象
>    public Specification<Entity> buildSpecification(String search) {
>       Pattern pattern = Pattern.compile("(\\w+?)(:|<|>)(\\w+?),");
>       Matcher matcher = pattern.matcher(search + ",");
>       while (matcher.find()) {
>          this.with(matcher.group(1), Operator.fromOperator(matcher.group(2)), matcher.group(3));
>       }
>       return this.build();
>    }
>    //根据参数返回我们刚才创建的SearchCriteria
>    private SpecificationsBuilder with(String key, Operator operation, Object value) {
>       params.add(new SearchCriteria(key, operation, value));
>       return this;
>    }
>    //根据我们刚才创建的MySpecification返回所需要的Specification
>    private Specification<Entity> build() {
>       if (params.size() == 0) {
>          return null;
>       }
>       List<Specification> specs = params.stream()
>             .map(MySpecification<User>::new)
>             .collect(Collectors.toList());
>       Specification result = specs.get(0);
>       for (int i = 1; i < params.size(); i++) {
>          result = Specification.where(result)
>                .and(specs.get(i));
>       }
>       return result;
>    }
> }
> ```
>
> 通过上面的代码，我们可以看到通过自定义的 SpecificationsBuilder，来处理请求参数 search 里面的值，然后转化成我们上面写的 SearchCriteria 对象，再调用 MySpecification 生成我们需要的 Specification，从而利用 JpaSpecificationExecutor 实现查询效果。是不是学起来并不困难了，你学会了吗？
>
> ### 总结
>
> 本课时，我们通过实例学习了 JpaSpecificationExecutor 的用法，并且通过源码了解了 JpaSpecificationExecutor 的实现原理，最后我举了一个实战场景的例子，使我们可以利用 Spring Data JPA and Specifications 很轻松地创建一个基于 Search 的 Rest API。虽然我介绍的这个例子还有很多可以扩展的地方，但是希望你根据实际情况再进行相应的扩展。
>
> 这里顺带留一个思考题：怎么查询 UserAddress？提示你可以利用我上面提到的 SpecificationsBuilder 进行解决。
>
> 本课时到这就结束了，欢迎你在下面留言讨论或分享。下节课，我会为你讲解如何自定义 Repository。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa