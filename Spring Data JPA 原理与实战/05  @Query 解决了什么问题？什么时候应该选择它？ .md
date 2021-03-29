[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4705)



> 上个课时我们介绍了 Query Define Method 的语法，这一课时来介绍一下 @Query 注解的语法是什么样的。我们通过快速体验 @Query 的方法、JpaQueryLookupStrategy 关键源码剖析、@Query 的基本用法、@Query 之 Projections 应用返回指定 DTO、@Query 动态查询解决方法，这几个部分来掌握 @Query 的用法。在学会了之后，你就可以应对工作中常见的 CURD 的写法问题。
>
> ### 快速体验 @Query 的方法
>
> 开始之前，首先来看一个 Demo，沿用我们之前的例子，新增一个 @Query 的方法，快速体验一下 @Query 的使用方法，如下所示：
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import org.springframework.data.jpa.repository.JpaRepository;
> import org.springframework.data.jpa.repository.Query;
> import org.springframework.data.repository.query.Param;
> public interface UserDtoRepository extends JpaRepository<User,Long> {
>    //通过query注解根据name查询user信息
>    @Query("From User where name=:name")
>    User findByQuery(@Param("name") String nameParam);
> }
> ```
>
> 然后，我们新增一个测试类：
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import org.junit.jupiter.api.Test;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
> @DataJpaTest
> public class UserRepositoryQueryTest {
>    @Autowired
>    private UserDtoRepository userDtoRepository;
>    @Test
>    public void testQueryAnnotation() {
> //新增一条数据方便测试      userDtoRepository.save(User.builder().name("jackxx").email("123456@126.com").sex("man").address("shanghai").build());
>       //调用上面的方法查看结果
>       User user2 = userDtoRepository.findByQuery("jack");
>       System.out.println(user2);
>    }
> }
> ```
>
> 最后，看到运行的结果如下：
>
> 复制代码
>
> ```
> Hibernate: insert into user (address, email, name, sex, version, id) values (?, ?, ?, ?, ?, ?)
> Hibernate: select user0_.id as id1_0_, user0_.address as address2_0_, user0_.email as email3_0_, user0_.name as name4_0_, user0_.sex as sex5_0_, user0_.version as version6_0_ from user user0_ where user0_.name=?
> User(id=1, name=jack, email=123456@126.com, version=0, sex=man, address=shanghai)
> ```
>
> 通过上面的例子我们发现，这次不是通过方法名来生成查询语法，而是 @Query 注解在其中起了作用，使 "From User where name=:name"JPQL 生效了。那么它的实现原理是什么呢？我们通过源码来看一下。
>
> ### JpaQueryLookupStrategy 关键源码剖析
>
> 我们在 03 课时已经介绍过 QueryLookupStrategy 的策略值有哪些，那么我们来看下源码是如何起作用的。
>
> 我们先打开 QueryExecutorMethodInterceptor 类，找到如下代码：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/56/57/CgqCHl9rKUmAaHv0AAHfhSZLnCM314.png)
>
> 再运行上面的测试用例，这时候在这里设置一个断点，可以看到默认的策略是CreateIfNotFound，也就是如果有@Query注解，那么以@Query的注解内容为准，可以忽略方法名。
>
> 我们继续往后面看，进入到 lookupStrategy.resolveQuery 里面，如下所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/56/4C/Ciqc1F9rKU-AKQ5bAAWmeacWTAQ768.png)
>
> 通过上图的断点和红框之处，我们也发现了，Spring Data JPA 这个地方使用了策略、模式，当我们自己写策略模式的时候也可以进行参考。
>
> 那么接着往下 debug，进入到 resolveQuery 方法里面，如下图所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/56/4C/Ciqc1F9rKVaARZk-AAYrhMro5y4100.png)
>
> 我们可以看到图中 ①处，如果 Query 注解找到了，就不会走到 ② 处了（即我们第 03 课时中讲的 Defined Query Method 语法）。
>
> 这时我们点开 Query 里面的 Query 属性的值看一下，你会发现这里同时生成了两个 SQL：一个是查询总数的 Query 定义，另一个是查询结果 Query 定义。
>
> 到这里我们已经基本明白了，如果想看看 Query 具体是怎么生成的、上面的 @Param 注解是怎么生效的，可以在上面的图 ① 处 debug 继续往里面看，如下所示：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/56/4C/Ciqc1F9rKV2AMpasAAE0Zjc3fRw672.png)
>
> 我们继续一路 debug 就可以看到怎么通过 @Query 去生成 SQL 了，这个不是本节的重点，我在这里就简单带过了，你有兴趣可以自己去 debug 看一下。
>
> 那么原理我们掌握了，接下来看看 @Query 给我们提供了哪些语法吧，先看下基本用法。
>
> ### @Query 的基本用法
>
> 在讲解它的语法之前，我们看一下它的注解源码，了解一下基本用法。
>
> 复制代码
>
> ```
> package org.springframework.data.jpa.repository;
> public @interface Query {
>    /**
>     * 指定JPQL的查询语句。（nativeQuery=true的时候，是原生的Sql语句）
> 	*/
>    String value() default "";
>    /**
> 	* 指定count的JPQL语句，如果不指定将根据query自动生成。
>     * （如果当nativeQuery=true的时候，指的是原生的Sql语句）
>     */
>    String countQuery() default "";
>    /**
>     * 根据哪个字段来count，一般默认即可。
> 	*/
>    String countProjection() default "";
>    /**
>     * 默认是false，表示value里面是不是原生的sql语句
> 	*/
>    boolean nativeQuery() default false;
>    /**
>     * 可以指定一个query的名字，必须唯一的。
> 	* 如果不指定，默认的生成规则是：
>     * {$domainClass}.${queryMethodName}
>     */
>    String name() default "";
>    /*
>     * 可以指定一个count的query的名字，必须唯一的。
> 	* 如果不指定，默认的生成规则是：
>     * {$domainClass}.${queryMethodName}.count
>     */
>    String countName() default "";
> }
> ```
>
> 所以到这里你会发现， @Query 用法是使用 JPQL 为实体创建声明式查询方法。我们一般只需要关心 @Query 里面的 value 和 nativeQuery、countQuery 的值即可，因为其他的不常用。
>
> 使用声明式 JPQL 查询有个好处，就是启动的时候就知道你的语法正确不正确。那么我们简单介绍一下 JPQL 语法。
>
> #### JPQL 的语法
>
> 我们先看一下查询的语法结构，代码如下：
>
> 复制代码
>
> ```
> SELECT ... FROM ...
> [WHERE ...]
> [GROUP BY ... [HAVING ...]]
> [ORDER BY ...]
> ```
>
> 你会发现它的语法结构有点类似我们 SQL，唯一的区别就是 JPQL FROM 后面跟的是对象，而 SQL 里面的字段对应的是对象里面的属性字段。
>
> 同理我们看一下 update 和 delete 的语法结构：
>
> 复制代码
>
> ```
> DELETE FROM ... [WHERE ...]
>  
> UPDATE ... SET ... [WHERE ...]
> ```
>
> 其中“...”省略的部分是实体对象名字和实体对象里面的字段名字，而其中类似 SQL 一样包含的语法关键字有：SELECT FROM WHERE UPDATE DELETE JOIN OUTER INNER LEFT GROUP BY HAVING FETCH DISTINCT OBJECT NULL TRUE FALSE NOT AND OR BETWEEN LIKE IN AS UNKNOWN EMPTY MEMBER OF IS AVG MAX MIN SUM COUNT ORDER BY ASC DESC MOD UPPER LOWER TRIM POSITION CHARACTER_LENGTH CHAR_LENGTH BIT_LENGTH CURRENT_TIME CURRENT_DATE CURRENT_TIMESTAMP NEW EXISTS ALL ANY SOME 这么多，我们就不一一介绍了。
>  这个语法用起来不复杂，遇到问题时，简单想一下 SQL 就可以知道了，我推荐一个 Oracle 的文档地址：https://docs.oracle.com/html/E13946_04/ejb3_langref.html，你也可以通过查看这个文档，找到解决问题的办法。
>
> 那么 JPQL 的语法你已经大概了解了，我们再来看下 @Query 怎么使用。
>
> #### @Query 用法案例
>
> 我们通过几个案例来了解一下 @Query 的用法，你就可以知道 @Query 怎么使用、怎么传递参数、怎么分页等。
>
> **案例 1：** 要在 Repository 的查询方法上声明一个注解，这里就是 @Query 注解标注的地方。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long>{
>   @Query("select u from User u where u.emailAddress = ?1")
>   User findByEmailAddress(String emailAddress);
> }
> ```
>
> **案例 2：** LIKE 查询，注意 firstname 不会自动加上“%”关键字。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query("select u from User u where u.firstname like %?1")
>   List<User> findByFirstnameEndsWith(String firstname);
> }
> ```
>
> **案例 3：** 直接用原始 SQL，nativeQuery = true 即可。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query(value = "SELECT * FROM USERS WHERE EMAIL_ADDRESS = ?1", nativeQuery = true)
>   User findByEmailAddress(String emailAddress);
> }
> ```
>
> > 注意：nativeQuery 不支持直接 Sort 的参数查询。
>
> **案例 4：** 下面是****nativeQuery 的排序错误的写法，会导致无法启动。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
> @Query(value = "select * from user_info where first_name=?1",nativeQuery = true)
> List<UserInfoEntity> findByFirstName(String firstName,Sort sort);
> }
> ```
>
> **案例 5：** nativeQuery 排序的正确写法。
>
> 复制代码
>
> ```
> @Query(value = "select * from user_info where first_name=?1 order by ?2",nativeQuery = true)
> List<UserInfoEntity> findByFirstName(String firstName,String sort);
> //调用的地方写法last_name是数据里面的字段名，不是对象的字段名
> repository.findByFirstName("jackzhang","last_name");
> ```
>
> 通过上面几个案例，我们看到了 @Query 的几种用法，你就会明白排序、参数、使用方法、LIKE、原始 SQL 怎么写。下面继续通过案例来看下 @Query 的排序。
>
> #### @Query 的排序
>
> @Query中在用JPQL的时候，想要实现排序，方法上直接用 PageRequest 或者 Sort 参数都可以做到。
>
> 在排序实例中，实际使用的属性需要与实体模型里面的字段相匹配，这意味着它们需要解析为查询中使用的属性或别名。我们看一下例子，这是一个`state_field_path_expression JPQL`的定义，并且 Sort 的对象支持一些特定的函数。
>
> **案例 6：** Sort and JpaSort 的使用，它可以进行排序。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query("select u from User u where u.lastname like ?1%")
>   List<User> findByAndSort(String lastname, Sort sort);
>   @Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
>   List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);
> }
> //调用方的写法，如下：
> repo.findByAndSort("lannister", new Sort("firstname"));
> repo.findByAndSort("stark", new Sort("LENGTH(firstname)"));
> repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)"));
> repo.findByAsArrayAndSort("bolton", new Sort("fn_len"));
> ```
>
> 上面这个案例讲述的是排序用法，再来看下 @Query 的分页用法。
>
> #### @Query 的分页
>
> @Query 的分页分为两种情况，分别为 JPQL 的排序和 nativeQuery 的排序。看下面的案例。
>
> **案例 7**：直接用 Page 对象接受接口，参数直接用 Pageable 的实现类即可。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query(value = "select u from User u where u.lastname = ?1")
>   Page<User> findByLastname(String lastname, Pageable pageable);
> }
> //调用者的写法
> repository.findByFirstName("jackzhang",new PageRequest(1,10));
> ```
>
> **案例 8：**@Query 对原生 SQL 的分页支持，并不是特别友好，因为这种写法比较“骇客”，可能随着版本的不同会有所变化。我们以 MySQL 为例。
>
> 复制代码
>
> ```
>  public interface UserRepository extends JpaRepository<UserInfoEntity, Integer>, JpaSpecificationExecutor<UserInfoEntity> {
>    @Query(value = "select * from user_info where first_name=?1 /* #pageable# */",
>          countQuery = "select count(*) from user_info where first_name=?1",
>          nativeQuery = true)
>    Page<UserInfoEntity> findByFirstName(String firstName, Pageable pageable);
> }
> //调用者的写法
> return userRepository.findByFirstName("jackzhang",new PageRequest(1,10, Sort.Direction.DESC,"last_name"));
> //打印出来的sql
> select  *   from  user_info  where  first_name=? /* #pageable# */  order by  last_name desc limit ?, ?
> ```
>
> **这里需要注意：这个注释 /\* #pageable# \*/ 必须有。**
>  另外，随着版本的变化，这个方法有可能会进行优化。此外还有一种实现方法，就是自己写两个查询方法，自己手动分页。
>
> 关于 @Query 的用法，还有一个需要了解的内容，就是 @ Param 用法。
>
> #### @Param 用法
>
> @Param 注解指定方法参数的具体名称，通过绑定的参数名字指定查询条件，这样不需要关心参数的顺序。**我比较推荐这种做法，因为它比较利于代码重构**。如果不用 @Param 也是可以的，参数是有序的，这使得查询方法对参数位置的重构容易出错。我们看个案例。
>
> **案例 9**：根据 firstname 和 lastname 参数查询 user 对象。
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
>   User findByLastnameOrFirstname(@Param("lastname") String lastname,
>                                  @Param("firstname") String firstname);
> }
> ```
>
> **案例 10：** 根据参数进行查询，top 10 前面说的“query method”关键字照样有用，如下所示：
>
> 复制代码
>
> ```
> public interface UserRepository extends JpaRepository<User, Long> {
>   @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
>   User findTop10ByLastnameOrFirstname(@Param("lastname") String lastname,
>                                  @Param("firstname") String firstname);
> }
> ```
>
> **这里说下我的经验之谈：你在通过 @Query 定义自己的查询方法时，我建议也用 Spring Data JPA 的 name query 的命名方法，这样下来风格就比较统一了。**
>  上面我介绍了 @Query 的基本用法，下面介绍一下 @Query 在我们的实际应用中最受欢迎的两处场景。
>
> ### @Query 之 Projections 应用返回指定 DTO
>
> 我们在之前的例子的基础上新增一张表 UserExtend，里面包含身份证、学号、年龄等信息，最终我们的实体变成如下模样：
>
> 复制代码
>
> ```
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> public class UserExtend { //用户扩展信息表
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private Long userId;
>    private String idCard;
>    private Integer ages;
>    private String studentNumber;
> }
> @Entity
> @Data
> @Builder
> @AllArgsConstructor
> @NoArgsConstructor
> public class User { //用户基本信息表
>    @Id
>    @GeneratedValue(strategy= GenerationType.AUTO)
>    private Long id;
>    private String name;
>    private String email;
>    @Version
>    private Long version;
>    private String sex;
>    private String address;
> }
> ```
>
> 如果我们想定义一个 DTO 对象，里面只要 name、email、idCard，这个时候我们怎么办呢？这种场景非常常见，但好多人使用的都不是最佳实践，我在这里介绍几种方式做一下对比。
>
> 我们先看一下，刚学 JPA 的时候别手别脚的写法：
>
> 复制代码
>
> ```
> public interface UserDtoRepository extends JpaRepository<User,Long> {
>    /**
>     * 查询用户表里面的name、email和UserExtend表里面的idCard
>     * @param id
>     * @return
>     */
>    @Query("select u.name,u.email,e.idCard from User u,UserExtend e where u.id= e.userId and u.id=:id")
>    List<Object[]> findByUserId(@Param("id") Long id);
> }
> ```
>
> 我们通过下面的测试用例来取上面 findByUserId 方法返回的数据组结果值，再塞到 DTO 里面，代码如下：
>
> 复制代码
>
> ```
> @Test
> public void testQueryAnnotation() {
> //新增一条用户数据  userDtoRepository.save(User.builder().name("jack").email("123456@126.com").sex("man").address("shanghai").build());
> //再新增一条和用户一对一的UserExtend数据  userExtendRepository.save(UserExtend.builder().userId(1L).idCard("shengfengzhenghao").ages(18).studentNumber("xuehao001").build());
> //查询我们想要的结果
>    List<Object[]> userArray = userDtoRepository.findByUserId(1L);
>    System.out.println(String.valueOf(userArray.get(0)[0])+String.valueOf(userArray.get(0)[1]));
>    UserDto userDto = UserDto.builder().name(String.valueOf(userArray.get(0)[0])).build();
>    System.out.println(userDto);
> }
> ```
>
> 其实经验的丰富的“老司机”一看就知道这肯定不是最佳实践，这多麻烦呀，肯定会有更优解。那么我们再对此稍加改造，用 UserDto 接收返回结果。
>
> #### 利用 class UserDto 获取我们想要的结果
>
> 首先，我们新建一个 UserDto 类的内容。
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import lombok.AllArgsConstructor;
> import lombok.Builder;
> import lombok.Data;
> @Data
> @Builder
> @AllArgsConstructor
> public class UserDto {
>     private String name,email,idCard;
> }
> ```
>
> 其次，我们看下利用 @Query 在 Repository 里面怎么写。
>
> 复制代码
>
> ```
> public interface UserDtoRepository extends JpaRepository<User, Long> {
>    @Query("select new com.example.jpa.example1.UserDto(CONCAT(u.name,'JK123'),u.email,e.idCard) from User u,UserExtend e where u.id= e.userId and u.id=:id")
>    UserDto findByUserDtoId(@Param("id") Long id);
> }
> ```
>
> 我们利用 JPQL，new 了一个 UserDto；再通过构造方法，接收查询结果。其中你会发现，我们用 CONCAT 的关键字做了一个字符串拼接，这时有的同学就会问了，这种方法支持的关键字有哪些呢？
>
> 你可以查看JPQL的 Oracal 文档，也可以通过源码来看支持的关键字有哪些。
>
> 首先，我们打开 ParameterizedFunctionExpression 会发现 Hibernate 支持的关键字有这么多，都是 MySQL 数据库的查询关键字，这里就不一一解释了。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/56/57/CgqCHl9rKZeAHUEyAAGh83E9Xs4928.png)
>
> 然后，我们写一个测试方法，调用上面的方法测试一下。
>
> 复制代码
>
> ```
> @Test
> public void testQueryAnnotationDto() {
>    userDtoRepository.save(User.builder().name("jack").email("123456@126.com").sex("man").address("shanghai").build());
>    userExtendRepository.save(UserExtend.builder().userId(1L).idCard("shengfengzhenghao").ages(18).studentNumber("xuehao001").build());
>    UserDto userDto = userDtoRepository.findByUserDtoId(1L);
>    System.out.println(userDto);
> }
> ```
>
> 最后，我们运行一下测试用例，结果如下。这时你会发现，我们按照预期操作得到了 UserDto 的结果。
>
> 复制代码
>
> ```
> Hibernate: insert into user (address, email, name, sex, version, id) values (?, ?, ?, ?, ?, ?)
> Hibernate: insert into user_extend (ages, id_card, student_number, user_id, id) values (?, ?, ?, ?, ?)
> Hibernate: select (user0_.name||'JK123') as col_0_0_, user0_.email as col_1_0_, userextend1_.id_card as col_2_0_ from user user0_ cross join user_extend userextend1_ where user0_.id=userextend1_.user_id and user0_.id=?
> UserDto(name=jackJK123, email=123456@126.com, idCard=shengfengzhenghao)
> ```
>
> 那么还有更简单的方法吗？答案是有，下面我们利用 UserDto 接口来实现一下。
>
> #### 利用 UserDto 接口获得我们想要的结果
>
> 首先，新增一个 UserSimpleDto 接口来得到我们想要的 name、email、idCard 信息。
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> public interface UserSimpleDto {
>    String getName();
>    String getEmail();
>    String getIdCard();
> }
> ```
>
> 其次，在 UserDtoRepository 里面新增一个方法，返回结果是 UserSimpleDto 接口。
>
> 复制代码
>
> ```
> public interface UserDtoRepository extends JpaRepository<User, Long> {
> //利用接口DTO获得返回结果，需要注意的是每个字段需要as和接口里面的get方法名字保持一样
> @Query("select CONCAT(u.name,'JK123') as name,UPPER(u.email) as email ,e.idCard as idCard from User u,UserExtend e where u.id= e.userId and u.id=:id")
> UserSimpleDto findByUserSimpleDtoId(@Param("id") Long id);
> }
> ```
>
> 然后，测试用例写法如下。
>
> 复制代码
>
> ```
> @Test
> public void testQueryAnnotationDto() {
>    userDtoRepository.save(User.builder().name("jack").email("123456@126.com").sex("man").address("shanghai").build());
>   userExtendRepository.save(UserExtend.builder().userId(1L).idCard("shengfengzhenghao").ages(18).studentNumber("xuehao001").build());
>    UserSimpleDto userDto = userDtoRepository.findByUserSimpleDtoId(1L);
>    System.out.println(userDto);  System.out.println(userDto.getName()+":"+userDto.getEmail()+":"+userDto.getIdCard());
> }
> ```
>
> 最后，我们执行可以得到如下结果。
>
> 复制代码
>
> ```
> org.springframework.data.jpa.repository.query.AbstractJpaQuery$TupleConverter$TupleBackedMap@373c28e5
> jackJK123:123456@126.COM:shengfengzhenghao
> ```
>
> 我们发现，比起 DTO 我们不需要 new 了，并且接口只能读，那么我们返回的结果 DTO 的职责就更单一了，只用来查询。
>
> **接口的方式是我比较推荐的做法，因为它是只读的，对构造方法没有要求，返回的实际是 HashMap。**
>
> 返回结果介绍完了，那么我们来看下一个最常见的问题：如何用 @Query 注解实现动态查询？
>
> ### @Query 动态查询解决方法
>
> 我们看一个例子，来了解一下如何实现 @Query 的动态参数查询。
>
> 首先，新增一个 UserOnlyName 接口，只查询 User 里面的 name 和 email 字段。
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> //获得返回结果
> public interface UserOnlyName {
>     String getName();
>     String getEmail();
> }
> ```
>
> 其次，在我们的 UserDtoRepository 里面新增两个方法：一个是利用 JPQL 实现动态查询，一个是利用原始 SQL 实现动态查询。
>
> 复制代码
>
> ```
> package com.example.jpa.example1;
> import org.springframework.data.jpa.repository.JpaRepository;
> import org.springframework.data.jpa.repository.Query;
> import org.springframework.data.repository.query.Param;
> import java.util.List;
> public interface UserDtoRepository extends JpaRepository<User, Long> {
>    /**
>     * 利用JQPl动态查询用户信息
>     * @param name
>     * @param email
>     * @return UserSimpleDto接口
>     */
>    @Query("select u.name as name,u.email as email from User u where (:name is null or u.name =:name) and (:email is null or u.email =:email)")
>    UserOnlyName findByUser(@Param("name") String name,@Param("email") String email);
>    /**
>     * 利用原始sql动态查询用户信息
>     * @param user
>     * @return
>     */
>    @Query(value = "select u.name as name,u.email as email from user u where (:#{#user.name} is null or u.name =:#{#user.name}) and (:#{#user.email} is null or u.email =:#{#user.email})",nativeQuery = true)
>    UserOnlyName findByUser(@Param("user") User user);
> ```
>
> 然后，我们新增一个测试类，测试一下上面方法的结果。
>
> 复制代码
>
> ```
> @Test
> public void testQueryDinamicDto() {
>    userDtoRepository.save(User.builder().name("jack").email("123456@126.com").sex("man").address("shanghai").build());
>    UserOnlyName userDto = userDtoRepository.findByUser("jack", null);
>    System.out.println(userDto.getName() + ":" + userDto.getEmail());
>    UserOnlyName userDto2 = userDtoRepository.findByUser(User.builder().email("123456@126.com").build());
>    System.out.println(userDto2.getName() + ":" + userDto2.getEmail());
> }
> ```
>
> 最后，运行结果如下。
>
> 复制代码
>
> ```
> Hibernate: insert into user (address, email, name, sex, version, id) values (?, ?, ?, ?, ?, ?)
>  : binding parameter [1] as [VARCHAR] - [shanghai]
>  : binding parameter [2] as [VARCHAR] - [123456@126.com]
>  : binding parameter [3] as [VARCHAR] - [jack]
>  : binding parameter [4] as [VARCHAR] - [man]
>  : binding parameter [5] as [BIGINT] - [0]
>  : binding parameter [6] as [BIGINT] - [1]
> Hibernate: select user0_.name as col_0_0_, user0_.email as col_1_0_ from user user0_ where (? is null or user0_.name=?) and (? is null or user0_.email=?)
>  : binding parameter [1] as [VARCHAR] - [jack]
>  : binding parameter [2] as [VARCHAR] - [jack]
>  : binding parameter [3] as [VARCHAR] - [null]
>  : binding parameter [4] as [VARCHAR] - [null]
> jack:123456@126.com
> Hibernate: select u.name as name,u.email as email from user u where (? is null or u.name =?) and (? is null or u.email =?)
>  : binding parameter [1] as [VARBINARY] - [null]
>  : binding parameter [2] as [VARBINARY] - [null]
>  : binding parameter [3] as [VARCHAR] - [123456@126.com]
>  : binding parameter [4] as [VARCHAR] - [123456@126.com]
> jack:123456@126.com
> ```
>
> 注意：其中我们打印了一下 SQL 传入的参数，是为了让我们更清楚参数都传入了什么值。
>  上面的两个方法，我们分别采用了 JPQL 的动态参数和 SPEL 的表达式方式获取参数（这个我们在第 26 课时“SpEL 解决了哪些问题”中再详细介绍）。
>
> 通过上面的实例可以看得出来，我们采用了 :email isnullor s.email = :email 这种方式来实现动态查询的效果，实际工作中也可以演变得很复杂。所以，我们再看一个实际工作中复杂一点的例子。
>
> **通过原始 sql，根据动态条件 room 关联 room_record 来获取 room_record 的结果。**
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/56/57/CgqCHl9rKauAfQmmAAHMquqHBVg909.png)
>
> **通过 JPQL 动态参数查询 RoomRecord，如下图：**
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/56/4C/Ciqc1F9rKbGAT_6FAAL_YUkV6AQ306.png)
>
> 这两个例子就是比较复杂的动态查询的案例，和上面的简单动态查询语法是一样的，你仔细看一下就会明白，并且可以轻松应用到工作中进行动态查询。
>
> ### 总结
>
> 到此，@Query 的常见用法，就已经讲完了。我们通过基本语法，分析了一些原理，讲解了常见的 DTO 和动态参数的实现方法，详细掌握了 @Query 的用法。
>
> 那么这个时候你可能会问了，我们知道定义方法名可以获得想要的结果，@Query 注解亦可以获得想要的结果，nativeQuery 也可以获得想要的结果，那么我们该如何做选择呢？**下面我从个人经验中总结了一些观点，分享给你。**
>
> 1. 能用方法名表示的，尽量用方法名表示，因为这样语义清晰、简单快速，基本上只要编译通过，一定不会有问题；
> 2. 能用 @Query 里面的 JPQL 表示的，就用 JPQL，这样与 SQL 无关，万一哪天换数据库了，基本上代码不用改变；
> 3. 最后实在没有办法了，可以选择 nativeQuery 写原始 SQL，特别是一开始从 MyBatis 转过来的同学，选择写 SQL 会更容易一些。
>
> > 好的架构师写代码时报错的顺序是：编译 < 启动 < 运行，即越早发现错误越好。
> >  点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa