[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4771)



> 在上一课时中，我们讨论了如何在微服务架构中实现认证和授权这两个基本的安全性控制手段，我们知道可以使用 OAuth2 协议来实现服务访问的授权，以及使用 JWT 来实现定制化的用户认证机制。同时，上一课时中也引出了 Spring Cloud 中专门用于提供服务访问安全性的 Spring Cloud Security 框架。今天，我们就将基于这一框架，讨论如何构建 OAuth2 授权服务器，并基于常用的密码模式生成对应的 Token，从而为下一节中的服务访问控制提供基础。
>
> ### 构建 OAuth2 授权服务器
>
> 在微服务架构的实现过程中，OAuth2 授权服务器和注册中心服务器、配置服务器一样也表现为一个独立的微服务，因此构建授权服务器的方法也是创建一个 Spring Boot 应用程序，我们需要引入合适的 Maven 依赖以及提供一个 Bootstrap 类作为访问的入口。
>
> 让我们回到 SpringHealth 案例，在前面各个服务的基础上，我们将在整个系统中创建一个新的代码工程并取名为 auth-server，同时引入与 OAuth2 协议相关的依赖，如下所示：
>
> 复制代码
>
> ```
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-security</artifactId>
> </dependency>
>  
> <dependency>
>     <groupId>org.springframework.security.oauth</groupId>
>     <artifactId>spring-security-oauth2</artifactId>
> </dependency>
> ```
>
> 其中第一个依赖 spring-cloud-security 属于 Spring Cloud 家族中的一员，而 spring-security-oauth2 则是来自于 Spring Security 中的 OAuth2 库。
>
> 现在 Maven 依赖已经添加完毕，下一步就是构建 Bootstrap 类，以下代码位于 auth-server 工程的 AuthApplication 类中。
>
> 复制代码
>
> ```
> @SpringCloudApplication
> @RestController
> @EnableResourceServer
> @EnableAuthorizationServer
> public class AuthApplication {
> 
>     public static void main(String[] args) {
>         SpringApplication.run(AuthApplication.class, args);
>     }
> }
> ```
>
> 注意到这里出现了一个新的注解 @EnableAuthorizationServer。顾名思义，@EnableAuthorizationServer 注解的作用在于为微服务运行环境提供一个基于 OAuth2 协议的授权服务，该授权服务会暴露一系列基于 RESTful 风格的端点（例如 /oauth/authorize 和 /oauth/token）供 OAuth2 授权流程进行使用。
>
> 构建 OAuth2 授权服务只是集成 OAuth2 协议的第一步。授权服务器是一种集中式系统，管理着所有与安全性流程相关的客户端和用户信息。因此，我们接下来需要在授权服务器中对这些基础信息进行初始化，而 Spring Cloud Security 框架为我们提供了各种配置类来实现这一目标。
>
> ### 基于密码模式生成 Token
>
> 在上一课时中，我们提到 OAuth2 协议存在四种授权模式。在本课程中，我们以简单但常用的密码模式为例进行展开。在密码模式下，用户向客户端提供用户名和密码，并将用户名和密码发给授权服务器从而请求 Token。授权服务器首先会对密码凭证信息进行认证，确认无误后，向客户端发放 Token。整个流程如下图所示：
>
> ![1.png](https://s0.lgstatic.com/i/image/M00/8B/01/CgqCHl_bCSOAXCQEAAF6rchmB6E579.png)
>
> 密码模式示意图
>
> 请注意，授权服务器在这里执行认证操作的目的，是验证所传入的用户名和密码是否正确。在密码模式下，这一步是必须的，而如果采用其他授权模式，则不一定会有用户认证这一环节。
>
> 确定了采用密码模式之后，我们来看为了实现这一授权模式，我们需要对授权服务器做哪些开发工作。首先我们需要设置一些基础数据，包括客户端信息和用户信息。然后基于这些基础数据，就可以通过 HTTP 请求获取所需的 Token。如下所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/89/6F/Ciqc1F_YZ5yAa21GAAA5jcj4Wlw391.png)
>
> 密码模式下的 OAuth2 协议集成开发流程
>
> #### 设置客户端信息
>
> 我们首先来看如何设置客户端信息。设置客户端时，用到的配置类是 ClientDetailsServiceConfigurer。显然，该配置类用来配置客户端详情服务 ClientDetailsService，而用于描述客户端详情的 ClientDetails 接口则包含了与安全性控制相关的多个重要方法，该接口中的部分方法定义如下：
>
> 复制代码
>
> ```
> public interface ClientDetails extends Serializable {
>  
>     //客户端唯一性 Id
>     String getClientId();
>     Set<String> getResourceIds();
>     boolean isSecretRequired();
>     //客户端安全码
>     String getClientSecret();
>     boolean isScoped();
>     //客户端的访问范围
>     Set<String> getScope();
>     //客户端可以使用的授权模式
>     Set<String> getAuthorizedGrantTypes();
> 	…
> }
> ```
>
> 我们无意对这些方法都详细进行展开，但有必要介绍与日常开发紧密相关的几个属性。首先，clientId 是一个必备属性，用来唯一标识客户的 Id，而 clientSecret 代表客户端安全码。
>
> 这里的 scope 用来限制客户端的访问范围，这个属性如果为空的话，客户端就拥有全部的访问范围。常见的设置方式可以是 webclient 或 mobileclient，分别代表 Web 端和移动端。
>
> 最后，authorizedGrantTypes 代表客户端可以使用的授权模式，可选的范围包括代表授权码模式的 authorization_code、代表隐式授权模式 implicit、代表密码模式的 password 以及代表客户端凭据模式的 client_credentials。这个属性在设置上也可以添加 refresh_token，用来通过刷新操作获取以上授权模式下所产生的新 Token。
>
> Spring Security 提供了 AuthorizationServerConfigurerAdapter 类来简化配置类的使用方式，我们可以通过继承该类并覆写其中的 configure() 方法来进行配置。使用 AuthorizationServerConfigurerAdapter 进行客户端信息配置的基本代码结构如下所示：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthAuthorizationServerConfigurer extends AuthorizationServerConfigurerAdapter {
>  
>     @Autowired
>     private AuthenticationManager authenticationManager;
>  
>     @Autowired
>     private UserDetailsService userDetailsService;
>  
>     @Override
>     public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
>         endpoints.authenticationManager(authenticationManager).userDetailsService(userDetailsService);
>     }
>  
>     @Override
>     public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
>  
>         clients.inMemory().withClient("springhealth").secret("{noop}springhealth_secret")
>                 .authorizedGrantTypes("refresh_token", "password", "client_credentials")
>                 .scopes("webclient", "mobileclient");
>     }
> }
> ```
>
> 可以看到这里我们设置了授权模式为密码模式。而在授权服务器中存储客户端信息有两种方式，一种就是如上述代码所示的基于内存级别的存储，另一种则是通过 JDBC 在数据库中存储详情信息。同时，我们注意到在设置客户端安全码时使用了"{noop}springhealth_secret"这种格式。这是因为在 Spring Security5 中统一使用 PasswordEncoder 来对密码进行编码，在设置密码时要求格式为“{id}password”。而这里的前缀“{noop}”就是代表具体 PasswordEncoder 的 id，表示我们使用的是 NoOpPasswordEncoder。
>
> 我们已经在前面的内容中提到，@EnableAuthorizationServer 注解会暴露一系列的端点，而授权是使用 AuthorizationEndpoint 这个端点来进行控制的。要想对该端点的行为进行配置，可以使用 AuthorizationServerEndpointsConfigurer 这个配置类。和ClientDetailsServiceConfigurer 配置类一样，我们也通过继承 AuthorizationServerConfigurerAdapter 并且覆写其中的 configure() 方法来进行配置。
>
> 因为我们指定了授权模式为密码模式，而密码模式包含认证环节。所以针对 AuthorizationServerEndpointsConfigurer 配置类需要指定一个认证管理器 AuthenticationManager，用于对用户名和密码进行认证。同样因为我们指定了基于密码的授权模式，所以需要指定一个自定义的 UserDetailsService 来替换全局的实现。关于 UserDetailsService 我们会放到下文中设置用户认证部分内容中进行介绍，这里只需要明确我们应该在 SpringHealthAuthorizationServerConfigurer 类中添加如下代码用来配置 AuthorizationServerEndpointsConfigurer：
>
> 复制代码
>
> ```
> @Autowired
> private AuthenticationManager authenticationManager;
>  
> @Autowired
> private UserDetailsService userDetailsService;
> 	 
> @Override
> public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
>  
>         endpoints.authenticationManager(authenticationManager)
>                 .userDetailsService(userDetailsService);
> }
> ```
>
> 至此，客户端设置工作全部完成，我们所做的事情就是实现了一个自定义的 SpringHealthAuthorizationServerConfigurer 配置类。
>
> #### 设置用户认证信息
>
> 设置用户认证信息所依赖的配置类是 WebSecurityConfigurer 类， Spring Security 同样提供了 WebSecurityConfigurerAdapter 类来简化该配置类的使用方式，我们可以继承 WebSecurityConfigurerAdapter 类并且覆写其中的 configure() 的方法来完成配置工作。
>
> 关于 WebSecurityConfigurer 配置类，我们首先需要明确配置的内容。实际上，设置用户信息非常简单，只需要指定用户名（User）、密码（Password）和角色（Role）这三项数据即可。这部分工作就是通过前文中提到的认证管理器 AuthenticationManager 来完成的，该接口非常简单，只包含一个用于认证的 authenticate 方法，如下所示：
>
> 复制代码
>
> ```
> public interface AuthenticationManager {
> 
>     Authentication authenticate(Authentication authentication)
>             throws AuthenticationException;
> }
> ```
>
> 在 Spring Security 中，我们可以使用 AuthenticationManagerBuilder 类轻松实现基于内存、LADP 和 JDBC 的认证机制。在 SpringHealth 案例中，我们使用的是该类中的 inMemoryAuthentication() 方法来实现基于内存的用户信息认证。完整的 SpringHealthWebSecurityConfigurer 类代码如下所示：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthWebSecurityConfigurer extends WebSecurityConfigurerAdapter {
>  
>     @Override
>     @Bean
>     public AuthenticationManager authenticationManagerBean() throws Exception {
>         return super.authenticationManagerBean();
>     }
>  
>     @Override
>     @Bean
>     public UserDetailsService userDetailsServiceBean() throws Exception {
>         return super.userDetailsServiceBean();
>     }
>  
>     @Override
>     protected void configure(AuthenticationManagerBuilder builder) throws Exception {
>      builder.inMemoryAuthentication().withUser("springhealth_user").password("{noop}password1").roles("USER").and()
>                 .withUser("springhealth_admin").password("{noop}password2").roles("USER", "ADMIN");
>     }
> }
> ```
>
> 从上面的代码中，我们看到构建了具有不同角色和密码的两个用户，请注意"springhealth_user"代表的角色是一个普通用户，而"springhealth_admin"则具有管理员角色。注意到我们在设置密码时，同样需要添加前缀“{noop}”。同时，我们还看到 authenticationManagerBean() 和 userDetailsServiceBean() 方法分别返回了父类的默认实现，而这里返回的 UserDetailsService 和 AuthenticationManager 在前面设置客户端时会用到。
>
> #### 生成 Token
>
> 当 OAuth2 授权服务器启动完毕，下一步就可以获取 Token。我们在构建 OAuth2 服务器时已经提到授权服务器中会暴露一批端点供HTTP请求进行访问。而获取Token的端点就是http://localhost:8080/oauth/token，在使用该端点时，我们需要提供前面所配置的客户端信息和用户信息。
>
> 这里使用 Postman 来模拟 HTTP 请求，客户端信息设置方式如下图所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/89/7A/CgqCHl_YZ76AOxeuAABMSQ09CHU759.png)
>
> 客户端信息设置示意图
>
> 我们在“Authorization”请求头中指定认证类型为“Basic Auth”，然后设置客户端名称和客户端安全码分别为“springhealth”和“springhealth_secret”。
>
> 接下去我们来指定针对授权模式的专用配置信息，首当其冲的是用于指定授权模式的 grant_type 属性，以及用于指定客户端访问范围的 scope 属性，这里分别设置为“password”和“webclient”。当然，既然设置了密码模式，所以也需要指定用户名和密码用于识别用户身份。这里，我们以“springhealth_user”这个用户为例进行设置，如下所示：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/89/6F/Ciqc1F_YZ8aAQ2VVAAA9E6V7PEk267.png)
>
> 用户信息设置示意图
>
> 在 Postman 中执行这个请求，会得到如下所示的返回结果：
>
> 复制代码
>
> ```
> {
>     "access_token": "868adf52-f524-4be8-a9e7-24c1c41aa7d6",
>     "token_type": "bearer",
>     "refresh_token": "96de5815-7935-4ca7-a24e-0d7441345696",
>     "expires_in": 43199,
>     "scope": "webclient"
> }
> ```
>
> 可以看到，除了作为请求参数的 scope 之外，这个返回结果中还包含了 access_token、token_type、refresh_token 和 expires_in 等属性。这些属性都很重要，我们一一进行解释。其中最重要的就是 access_token ，代表一个 OAuth2 Token；针对 token_type，在 OAuth2 协议中存在很多种 Token 类型可供选择，包括 bearer 类型、mac 类型等，这里返回的是最常见的一种类型，即 bearer 类型；refresh_token 的作用在于当 access_token 过期之后，用于重新下发一个新的 access_token；而 expires_in 属性用于指定 access_token 的有效时间，当超过这个有效时间时，access_token 将会自动失效。当然，因为每次请求都生成的 Token 都是唯一的，所以你在尝试时所获取的结果应该与我的不同。
>
> ### 小结与预告
>
> 对微服务访问进行安全性控制的首要条件是生成一个访问 Token。本课时从构建 OAuth2 服务器开始讲起，基于密码模式给出了如何设置客户端信息、用户认证信息以及如何最终生成 Token的实现过程。这个过程中需要开发人员熟悉 OAuth2 协议的相关概念以及 Spring Security 框架中所提供的各项配置功能。
>
> 这里给你留一道思考题：基于 OAuth2 协议所生成的一个合法的 Token 信息中应该包含哪些核心属性？
>
> 现在，我们已经成功获取了可用于访问各个服务的 Token 信息。在下一课时中，我们将在具体演示如何使用该 Token 来进行服务访问控制。