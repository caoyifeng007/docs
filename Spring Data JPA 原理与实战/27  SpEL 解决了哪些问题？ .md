[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4726)



> 实际工作中，我们经常会在一些注解中使用 SpEL 表达式，当然在 JPA 里也不例外，如果想知道它在 JPA 中的使用详情，必须要先从了解开始。那么这一讲，我们就来聊聊 SpEL 表达式相关知识。
>
> ### SpEL 基础语法
>
> #### SpEL 大纲
>
> SpEL 的全称为 Spring Expression Language，即 Spring 表达式语言，是 Spring framework 里面的核心项目。我们先来看一下 spring-expression 的 jar 包的引用关系，如下图所示。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/84/88/Ciqc1F_TZgOAeZinAAWQDYZICUE395.png)
>
> 从核心引用来看，SpEL 贯穿所有 Spring 的核心功能。当然了，SpEL 可以脱离 Spring 工程独立使用，其项目里有三个重要的接口：ExpressionParser、Expression、EvaluationContext，我从官方文档中找了一张图来说明它们之间的关系。
>  ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZg6AAkIJAADdShpwElA350.png)
>
> 注：图片来自网络
>
> **ExpressionParser**
>
> 它是 SpEL 的处理接口，默认实现类是 SpelExpressionParser，对外提供的只有两个方法，如下述代码所示。
>
> 复制代码
>
> ```
> public interface ExpressionParser {
>    // 根据传入的表达式生成Expression
>    Expression parseExpression(String expressionString) throws ParseException;
>    // 根据传入的表达式和ParserContext生成Expression对象
>    Expression parseExpression(String expressionString, ParserContext context) throws ParseException;
> }
> ```
>
> 我们可以看到，这两个方法的目的都是生成 Expression。
>
> **Expression**
>
> 它默认的实现是 SpELExpression，主要对外提供的接口就是根据表达式获得表达式响应的结果，如下图所示。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/84/88/Ciqc1F_TZhaAXrP7AAH3T5PfIR8675.png)
>
> 而它的这些方法中，最重的一个参数就是 EvaluationContext。
>
> **EvaluationContext**
>
> 表示解析 String 表达式所需要的上下文，例如寻找 ROOT 是谁，反射解析的 Method、Field、Constructor 的解析器和取值所需要的上下文。我们看一下其接口提供的方法，如下图所示。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZhyAfk9oAADDsWDvRJM660.png)
>
> 现在对这三个接口有了初步认识之后，我们通过实例来看一下基本用法。
>
> #### SpEL 的基本用法
>
> 下面是一个 SpEL 基本用法的例子，你可以结合注释来理解。
>
> 复制代码
>
> ```
> //ExpressionParser是操作SpEL的总入口，创建一个接口ExpressionParser对应的实例SpelExpressionParser
> ExpressionParser parser = new SpelExpressionParser();
> //通过上面我们讲的parser.parseExpression方法获得一个Expression的实例，里面实现的就是new一个SpelExpression对象；而parseExpression的参数就是SpEL的使用重点，各种表达式的字符串
> //1.简单的string类型用'' 引用
> Expression exp = parser.parseExpression("'Hello World'");
> //2.SpEL支持很多功能特性，如调用方法、访问属性、调用构造函数，我们可以直接调用String对象里面的concat方法进行字符串拼接
> Expression exp = parser.parseExpression("'Hello World'.concat('!')");
> //通过getValue方法可以得到经过Expresion计算parseExpression方法的字符串参数(符合SpEL语法的表达式)的结果
> String message = (String) exp.getValue();
> ```
>
> 而访问属性值如下所示。
>
> 复制代码
>
> ```
> //3.invokes getBytes()方法
> Expression exp = parser.parseExpression("'Hello World'.bytes");
> byte[] bytes = (byte[]) exp.getValue(); //得到 byte[]类型的结果
> ```
>
> SpEL 字符串表达式还支持使用“.”进行嵌套属性 prop1.prop2.prop3 访问，代码如下。
>
> 复制代码
>
> ```
> // invokes getBytes().length
> Expression exp = parser.parseExpression("'Hello World'.bytes.length");
> int length = (Integer) exp.getValue();
> ```
>
> 访问构造方法，例如字符串的构造方法，如下所示。
>
> 复制代码
>
> ```
> Expression exp = parser.parseExpression("new String('hello world').toUpperCase()");
> String message = exp.getValue(String.class);
> ```
>
> 我们也可以通过 EvaluationContext 来配置一些根元素，代码如下。
>
> 复制代码
>
> ```
> //我们通过一个Expression表达式想取name属性对应的值
> ExpressionParser parser = new SpelExpressionParser();
> Expression exp = parser.parseExpression("name");
> //我们通过EvaluationContext设置rootObject等于我们new的UserInfo对象
> UserInfo rootUserInfo = UserInfo.builder().name("jack").build();
> EvaluationContext context = new StandardEvaluationContext(rootUserInfo);
> //getValue根据我们设置context取值，可以得到jack字符串
> String name = (String) exp.getValue(context);
> //我们也可以利用SpEL的表达式进行运算，判断名字是否等于字符串Nikola
> Expression exp2 = parser.parseExpression("name == 'Nikola'");
> boolean result2 = exp2.getValue(context, Boolean.class); // 根据我们UserInfo的rootObject得到false
> ```
>
> 我们在看 SpelExpressionParser 的构造方法时，会发现其还支持一些配置，例如我们经常遇到空指针异常和下标越界的问题，就可以通过 SpelParserConfiguration 配置：当 Null 的时候自动初始化，当 Collection 越界的时候自动扩容增加。我们看一下例子，如下所示。
>
> 复制代码
>
> ```
> //构造一个Class，方便测试
> class MyUser {
>     public List<String> address;
> }
> //开启自动初始化null和自动扩容collection
> SpelParserConfiguration config = new SpelParserConfiguration(true,true);
> //利用config生成ExpressionParser的实例
> ExpressionParser parser = new SpelExpressionParser(config);
> //我们通过表达式取这个用户的第三个地址
> Expression expression = parser.parseExpression("address[3]");
> MyUser demo = new MyUser(); 
> //new一个对象，但是没有初始化MyUser里面的address，由于我们配置了自动初始化和扩容，所以通过下面的计算，没有得到异常，o可以得到一个空的字符串
> Object o = expression.getValue(demo);// 空字符串
> ```
>
> 通过上面的介绍，你大概就知道 SpEL 是什么意思了，也知道了该怎么单独使用它，其实不难理解。不过 SpEL 的功能远不止这么简单，我们通过在 Spring 中常见的应用场景，看一下它还有哪些功能。
>
> ### SpEL 在 Spring 中常见的使用场景
>
> SpEL 在 @Value 里面的用法最常见，我们通过 @Value 来了解一下。
>
> #### @Value 的应用场景
>
> 新建一个 DemoProperties 对象，用 Spring 装载，测试一下两个语法点：运算符和 Map、List。
>
> **第一个语法：通过 @Value 展示 SpEL 里面支持的各种运算符的写法。**如下面的表格所示。
>
> | **类型**     | **操作符**                                   |
> | ------------ | -------------------------------------------- |
> | 逻辑运算     | +, -, *, /, %, ^, div, mod                   |
> | 逻辑比较符号 | <, >, ==, !=, <=, >=, lt, gt, eq, ne, le, ge |
> | 逻辑关系     | and, or, not, &&, \|\|, !                    |
> | 三元表达式   | ?:                                           |
> | 正则表达式   | matches                                      |
>
> 我们通过四部分代码展示一下 SpEL 里面支持的各种运算符，用法如下所示。
>
> 复制代码
>
> ```
> @Data
> @ToString
> @Component //通过@Value使用SpEL的地方，一定要将此对象交由Spring进行管理
> public class DemoProperties {
> //第一部分：逻辑运算操作
>     @Value("#{19 + 1}") // 20
>     private double add;
>     @Value("#{'String1 ' + 'string2'}") // "String1 string2"
>     private String addString;
>     @Value("#{20 - 1}") // 19
>     private double subtract;
>     @Value("#{10 * 2}") // 20
>     private double multiply;
>     @Value("#{36 / 2}") // 19
>     private double divide;
>     @Value("#{36 div 2}") // 18, the same as for / operator
>     private double divideAlphabetic;
>     @Value("#{37 % 10}") // 7
>     private double modulo;
>     @Value("#{37 mod 10}") // 7, the same as for % operator
>     private double moduloAlphabetic;
> // 第二部分：逻辑比较符号
>     @Value("#{1 == 1}") // true
>     private boolean equal;
>     @Value("#{1 eq 1}") // true
>     private boolean equalAlphabetic;
>     @Value("#{1 != 1}") // false
>     private boolean notEqual;
>     @Value("#{1 ne 1}") // false
>     private boolean notEqualAlphabetic;
>     @Value("#{1 < 1}") // false
>     private boolean lessThan;
>     @Value("#{1 lt 1}") // false
>     private boolean lessThanAlphabetic;
>     @Value("#{1 <= 1}") // true
>     private boolean lessThanOrEqual;
>     @Value("#{1 le 1}") // true
>     private boolean lessThanOrEqualAlphabetic;
>     @Value("#{1 > 1}") // false
>     private boolean greaterThan;
>     @Value("#{1 gt 1}") // false
>     private boolean greaterThanAlphabetic;
>     @Value("#{1 >= 1}") // true
>     private boolean greaterThanOrEqual;
>     @Value("#{1 ge 1}") // true
>     private boolean greaterThanOrEqualAlphabetic;
> //第三部分：逻辑关系运算符    
>     @Value("#{250 > 200 && 200 < 4000}") // true
>     private boolean and;
>     @Value("#{250 > 200 and 200 < 4000}") // true
>     private boolean andAlphabetic;
>     @Value("#{400 > 300 || 150 < 100}") // true
>     private boolean or;
>     @Value("#{400 > 300 or 150 < 100}") // true
>     private boolean orAlphabetic;
>     @Value("#{!true}") // false
>     private boolean not;
>     @Value("#{not true}") // false
>     private boolean notAlphabetic;    
>     
> //第四部分：三元表达式 & Elvis运算符
>     @Value("#{2 > 1 ? 'a' : 'b'}") // "b"
>     private String ternary;
>     //demoProperties就是我们通过spring加载的当前对象，
>     //我们取spring容器里面的某个bean的属性，
>     //这里我们取的是demoProperties对象里面的someProperty属性，
>     //如果不为null就直接用，如果为null返回'default'字符串
>    @Value("#{demoProperties.someProperty != null ? demoProperties.someProperty : 'default'}")
>     private String ternaryProperty;
>     /**
>      * Elvis运算符是三元表达式简写的方式，和上面一样的结果。如果someProperty为null则返回default值。
>      */
>     @Value("#{demoProperties.someProperty ?: 'default'}")
>     private String elvis;
>     /**
>      * 取系统环境的属性，如果系统属性pop3.port已定义会直接注入，如果未定义，则返回默认值25。systemProperties是spring容器里面的systemProperties实体；
>      */
>     @Value("#{systemProperties['pop3.port'] ?: 25}")
>     private Integer port;
>     /**
>      * 还可以用于安全引用运算符主要为了避免空指针，源于Groovy语言。
>      * 很多时候你引用一个对象的方法或者属性时都需要做非空校验。
>      * 为了避免此类问题，使用安全引用运算符只会返回null而不是抛出一个异常。
>      */
>     //@Value("#{demoPropertiesx?:someProperty}") 
>     // 如果demoPropertiesx不为null，则返回someProperty值
>     private String someProperty;
>     
> //第五部分：正则表达式的支持
>     @Value("#{'100' matches '\\d+' }") // true
>     private boolean validNumericStringResult;
>     @Value("#{'100fghdjf' matches '\\d+' }") // false
>     private boolean invalidNumericStringResult;
>     // 利用matches匹配正则表达式，返回true
>     @Value("#{'valid alphabetic string' matches '[a-zA-Z\\s]+' }") 
>     private boolean validAlphabeticStringResult;
>     @Value("#{'invalid alphabetic string #$1' matches '[a-zA-Z\\s]+' }") // false
>     private boolean invalidAlphabeticStringResult;
>     //如果someValue只有数字
>     @Value("#{demoProperties.someValue matches '\\d+'}") // true 
>     private boolean validNumericValue;
>     //新增一个空的someValue属性方便测试
>     private String someValue="";
> }
> ```
>
> 我们可以通过 @Value 测试各种 SpEL 的表达式，这和放在 parser.parseExpression("SpEL 的表达式字符串"); 里面的效果是一样的。我们可以写一个测试用例来看一下，如下所示。
>
> 复制代码
>
> ```
> @ExtendWith(SpringExtension.class)
> @Import(TestConfiguration.class)
> @ComponentScan(value = "com.example.jpa.demo.config.DemoProperties")
> public class DemoPropertiesTest {
>     @Autowired(required = false)
>     private DemoProperties demoProperties;
>     @Test
>     public void testSpel() {
>         //通过测试用例就可以测试@Value里面不同表达式的值了
>         System.out.println(demoProperties.toString());
>     }
>     @TestConfiguration
>     static class TestConfig {
>         @Bean
>         public DemoProperties demoProperties () {
>             return new DemoProperties();
>         }
>     }
> }
> ```
>
> 或者你可以启动一下项目，也能看到结果。
>
> 下面我们通过源码来分析一下 @Value 的解析原理。Spring 项目启动的时候会根据 @Value 的注解，去加载 SpelExpressionResolver 及算出来需要的 StandardEvaluationContext，然后再调用 Expression 方法进行 getValue 操作，其中计算 StandardEvaluationContext 的关键源码如下面两张图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZjOAeGv_AATHsCZz1as768.png)
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZjmAW5suAARIid5TtLA365.png)
>
> **第二个语法：@Value 展示了 SpEL 可以直接读取 Map 和 List 里面的值**，代码如下所示。
>
> 复制代码
>
> ```
> //我们通过@Component加载一个类，并且给其中的List和Map附上值
> @Component("workersHolder")
> public class WorkersHolder {
>     private List<String> workers = new LinkedList<>();
>     private Map<String, Integer> salaryByWorkers = new HashMap<>();
>     public WorkersHolder() {
>         workers.add("John");
>         workers.add("Susie");
>         workers.add("Alex");
>         workers.add("George");
>         salaryByWorkers.put("John", 35000);
>         salaryByWorkers.put("Susie", 47000);
>         salaryByWorkers.put("Alex", 12000);
>         salaryByWorkers.put("George", 14000);
>     }
>     //Getters and setters ...
> }
> //SpEL直接读取Map和List里面的值
> @Value("#{workersHolder.salaryByWorkers['John']}") // 35000
> private Integer johnSalary;
> @Value("#{workersHolder.salaryByWorkers['George']}") // 14000
> private Integer georgeSalary;
> @Value("#{workersHolder.salaryByWorkers['Susie']}") // 47000
> private Integer susieSalary;
> @Value("#{workersHolder.workers[0]}") // John
> private String firstWorker;
> @Value("#{workersHolder.workers[3]}") // George
> private String lastWorker;
> @Value("#{workersHolder.workers.size()}") // 4
> private Integer numberOfWorkers;
> ```
>
> 以上就是 SpEL 的运算符和对 Map、List、SpringBeanFactory 里面的 Bean 的调用情况，不知道你是否掌握了？那么使用 @Value 都有哪些需要注意的呢？
>
> **@Value 使用的注意事项 # 与 $ 的区别**
>
> SpEL 表达式默认以 # 开始，以大括号进行包住，如 #{expression}。默认规则在 ParserContext 里面设置，我们也可以自定义，但是一般建议不要动。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZkSAcGUfAAIDnxtLZsQ409.png)
>
> 这里注意要与 Spring 中的 Properties 进行区别，Properties 相关的表达式是以 $ 开始的大括号进行包住的，如 ${property.name}。
>
> 也就是说 @Value 的值有两类：
>
> - ${ property**:**default_value }
> - \#{ obj.property**? :**default_value }
>
> 第一个注入的是外部参数对应的 Property，第二个则是 SpEL 表达式对应的内容。
>
> 而 Property placeholders 不能包含 SpEL 表达式，但是 SpEL 表达式可以包含 Property 的引用。如 #{${someProperty} + 2}，如果 someProperty=1，那么效果将是 #{ 1 + 2}，最终的结果将是 3。
>
> 上面我们通过 @Value 的应用场景讲解了一部分 SpEL 的语法，此外它同样适用于 @Query 注解，那么我们通过 @Query 再学习一些 SpEL 的其他语法。
>
> #### JPA 中 @Query 的应用场景
>
> SpEL 除了能在 @Value 里面使用外，也能在 @Query 里使用，而在 @Query 里还有一个特殊的地方，就是它可以用来取方法的参数。
>
> **通过 SpEL 取被 @Query 注解的方法参数**
>
> 在 @Query 注解中使用 SpEL 的主要目的是取方法的参数，主要有三种用法，如下所示。
>
> 复制代码
>
> ```
> //用法一：根据下标取方法里面的参数
> @Query("select u from User u where u.age = ?#{[0]}") 
> List<User> findUsersByAge(int age);
> //用法二：#customer取@Param("customer")里面的参数
> @Query("select u from User u where u.firstname = :#{#customer.firstname}")
> List<User> findUsersByCustomersFirstname(@Param("customer") Customer customer);
> //用法三：用JPA约定的变量entityName取得当前实体的实体名字
> @Query("from #{#entityName}")
> List<UserInfo> findAllByEntityName();
> ```
>
> 其中，
>
> - 方法一可以通过 [0] 的方式，根据下标取到方法的参数；
> - 方法二通过 #customer 可以根据 @Param 注解的参数的名字取到参数，必须通过 ?#{} 和 :#{} 来触发 SpEL 的表达式语法；
> - 方法三通过 #{#entityName} 取约定的实体的名字。
>
> 你要注意区别我们在“[05 | @Query 解决了什么问题？什么时候应该选择它？](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490#/detail/pc?id=4705)”中介绍的取 @Param 的用法`:lastname`这种方式。
>
> 下面我们再来看一个更复杂一点的例子，代码如下。
>
> 复制代码
>
> ```
> public interface UserInfoRepository extends JpaRepository<UserInfo, Long> {
>    // JPA约定的变量entityName取得当前实体的实体名字
>    @Query("from #{#entityName}")
>    List<UserInfo> findAllByEntityName();
>    
>    //一个查询中既可以支持SpEL也可以支持普通的:ParamName的方式
>    @Modifying
>    @Query("update #{#entityName} u set u.name = :name where u.id =:id")
>    void updateUserActiveState(@Param("name") String name, @Param("id") Long id);
>    
>    //演示SpEL根据数组下标取参数，和根据普通的Parma的名字:name取参数
>    @Query("select u from UserInfo u where u.lastName like %:#{[0]} and u.name like %:name%")
>    List<UserInfo> findContainingEscaped(@Param("name") String name);
>    
>    //SpEL取Parma的名字customer里面的属性
>    @Query("select u from UserInfo u where u.name = :#{#customer.name}")
>    List<UserInfo> findUsersByCustomersFirstname(@Param("customer") UserInfo customer);
>    
>    //利用SpEL根据一个写死的'jack'字符串作为参数
>    @Query("select u from UserInfo u where u.name = ?#{'jack'}")
>    List<UserInfo> findOliverBySpELExpressionWithoutArgumentsWithQuestionmark();
>    
>    //同时SpEL支持特殊函数escape和escapeCharacter
>    @Query("select u from UserInfo u where u.lastName like %?#{escape([0])}% escape ?#{escapeCharacter()}")
>    List<UserInfo> findByNameWithSpelExpression(String name);
>    
>    // #entityName和#[]同时使用
>    @Query("select u from #{#entityName} u where u.name = ?#{[0]} and u.lastName = ?#{[1]}")
>    List<UserInfo> findUsersByFirstnameForSpELExpressionWithParameterIndexOnlyWithEntityExpression(String name, String lastName);
>    //对于 native SQL同样适用，并且同样支持取pageable分页里面的属性值
>    @Query(value = "select * from (" //
>          + "select u.*, rownum() as RN from (" //
>          + "select * from user_info ORDER BY ucase(firstname)" //
>          + ") u" //
>          + ") where RN between ?#{ #pageable.offset +1 } and ?#{#pageable.offset + #pageable.pageSize}", //
>          countQuery = "select count(u.id) from user_info u", //
>          nativeQuery = true)
>    Page<UserInfo> findUsersInNativeQueryWithPagination(Pageable pageable);
> }
> ```
>
> 我个人比较推荐使用 @Param 的方式，这样语义清晰，参数换位置了也不影响执行结果。
>
> 关于源码的实现，你可以到 ExpressionBasedStringQuery.class 里面继续研究，关键代码如下图所示。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/84/88/Ciqc1F_TZlOAIupqAALzhPEi9nM327.png)
>
> 好了，以上就是 @Query 支持的 SpEL 的基本语法，其他场景我就不多列举了。那么其实 JPA 还支持自定义 rootObject，我们看一下。
>
> **spring-security-data 在 @Query 中的用法**
>
> 在实际工作中，我发现有些同事会用 spring-security 做鉴权，详细的 Spring Secrity 如何集成不是我们的重点，我就不多介绍了，具体怎么集成你可以查看官方文档：https://spring.io/projects/spring-security#learn。
>
> 我想说的是，当我们用 Spring Secrity 的时候，其实可以额外引入 jai 包 spring-security-data。如果我们使用了 JPA 和 Spring Secrity 的话，build.gradle 最终会变成如下形式，请看代码。
>
> 复制代码
>
> ```
> //引入spring data jpa
> implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
> //集成spring security
> implementation 'org.springframework.boot:spring-boot-starter-security'
> // 集成spring security data对JPA的支持
> implementation 'org.springframework.security:spring-security-data'
> ```
>
> 我们假设继承 Spring Security 之后，SecurityContextHolder 里面放置的 Authentication 是 UserInfo，代码如下。
>
> 复制代码
>
> ```
> //应用上下文中设置登录用户信息，此时Authentication类型为UserInfo
> SecurityContextHolder.getContext().setAuthentication(authentication);
> ```
>
> 这样 JPA 里面的 @Query 就可以取到当前的 SecurityContext 信息，其用法如下所示。
>
> 复制代码
>
> ```
> // 根据当前用户email取当前用户的信息
> @Query("select u from UserInfo u where u.emailAddress = ?#{principal.email}")
> List<UserInfo> findCurrentUserWithCustomQuery();
> //如果当前用户是admin，我们就返回某业务的所有对象；如果不是admin角色，就只给当前用户的某业务数据
> @Query("select o from BusinessObject o where o.owner.emailAddress like "+
>       "?#{hasRole('ROLE_ADMIN') ? '%' : principal.emailAddress}")
> List<BusinessObject> findBusinessObjectsForCurrentUser();
> ```
>
> 我们通过看源码会发现，spring-security-data 就帮我们做了一件事情：实现 EvaluationContextExtension，设置了 SpEL 所需要的 rootObject 为 SecurityExpressionRoot。关键代码如下图所示。
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZl2AIjYgAAYG1kqP1Zw860.png)
>
> 由于 SecurityExpressionRoot 是 rootObject，根据我们上面介绍的 SpEL 的基本用法，SecurityExpressionRoot 里面的各种属性和方法都可以在 SpEL 中使用，如下图所示。
>
> ![Drawing 9.png](https://s0.lgstatic.com/i/image/M00/84/88/Ciqc1F_TZmKAbFerAAHOBNbwd44831.png)
>
> 这其实也给了我们一些启发：当需要自动 rootObject 给 @Query 使用的时候，也可以采用这种方式，这样 @Query 的灵活性会增强很多。
>
> 最后我们再看看 SpEL 在 @Cacheable 里面做了哪些支持。
>
> #### SpEL 在 @Cacheable 中的应用场景
>
> 我们在实际工作中还有一个经常用到 SpEL 的场景，就是在 Cache 的时候，也就是 Spring Cache 的相关注解里面，如 @Cacheable、@CachePut、@CacheEvict 等。我们还是通过例子来体会一下，代码如下所示。
>
> 复制代码
>
> ```
> //缓存key取当前方法名，判断一下只有返回结果不为null或者非empty才进行缓存
> @Cacheable(value = "APP", key = "#root.methodName", cacheManager = "redis.cache", unless = "#result == null || #result.isEmpty()")
> @Override
> public Map<String, Map<String, String>> getAppGlobalSettings() {}
> //evict策略的key是当前参数customer里面的name属性
> @Caching(evict = {
> @CacheEvict(value="directory", key="#customer.name") })
> public String getAddress(Customer customer) {...}
> //在condition里面使用，当参数里面customer的name属性的值等于字符串Tom才放到缓存里面
> @CachePut(value="addresses", condition="#customer.name=='Tom'")
> public String getAddress(Customer customer) {...}
> //用在unless里面，利用SpEL的条件表达式判断，排除返回的结果地址长度小于64的请求
> @CachePut(value="addresses", unless="#result.length()<64")
> public String getAddress(Customer customer) {...}
> ```
>
> **Spring Cache 中 SpEL 支持的上下文语法**
>
> Spring Cache 提供了一些供我们使用的 SpEL 上下文数据，如下表所示（摘自 Spring 官方文档）。
>
> | 支持的属性    | 作用域     | 功能描述                                                     | 使用方法                         |
> | ------------- | ---------- | ------------------------------------------------------------ | -------------------------------- |
> | methodName    | root 对象  | 当前被调用的方法名                                           | #root.methodName                 |
> | method        | root 对象  | 当前被调用的方法                                             | #root.method.name                |
> | target        | root 对象  | 当前被调用的目标对象                                         | #root.target                     |
> | targetClass   | root 对象  | 当前被调用的目标对象类                                       | #root.targetClass                |
> | args          | root 对象  | 当前被调用的方法的参数列表                                   | #root.args[0]                    |
> | caches        | root 对象  | 当前方法调用使用的缓存列表（如@Cacheable(value={“cache1”, “cache2”})），则有两个 cache | #root.caches[0].name             |
> | argument name | 执行上下文 | 当前被调用的方法的参数，如 findById(Long id)，我们可以通过 #id 拿到参数 | #user.id 表示参数 user 里面的 id |
> | result        | 执行上下文 | 方法执行后的返回值（仅当方法执行之后的判断有效，如‘unless’，’cache evict’的 beforeInvocation=false） | #result                          |
>
> 有兴趣的话，你可以看一下 Spring Cache 中 SpEL 的 EvaluationContext 加载方式，关键源码如下图所示。
>
> ![Drawing 10.png](https://s0.lgstatic.com/i/image/M00/84/93/CgqCHl_TZm6AaOt5AAFe2hVbtrM473.png)
>
> ### 总结
>
> 本讲内容到这里就结束了。这一讲我们通过 SpEL 的基本语法介绍，分别介绍了其在 @Value、@Query、@Cache 注解里面的使用场景和方法，其中 # 和 $ 是容易在 @Value 里面犯错的地方；@Param 的用法 : 和 # 也是 @Query 里面容易犯错的地方，你要注意一下。
>
> 其实任何形式的 SpEL 的变化都离不开它基本的三个接口：ExpressionParser、Expression、EvaluationContext，只不过框架提供了不同形式的封装，你也可以根据实际场景自由扩展。
>
> 关于这一讲内容，希望你能认真去思考，有问题可以在下方留言，我们一起讨论。下一讲我们来聊聊 Hibernate 中一级缓存的概念，到时见。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa