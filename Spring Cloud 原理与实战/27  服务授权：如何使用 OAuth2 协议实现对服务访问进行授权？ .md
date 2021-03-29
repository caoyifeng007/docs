[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4772)



> 上一课时，我们构建了 OAuth2 授权服务器，并掌握了如何生成 Token 的系统方法。今天，我们关注如何使用 Token 来实现对服务访问的具体授权。在日常开发过程中，我们需要对每个服务的不同功能进行不同粒度的权限控制，并且希望这种控制方法足够灵活。同时，在微服务架构中，我们还需要考虑如何在多个服务中对 Token 进行有效的传播，确保整个服务访问的链路都能够得到授权管理。借助于 Spring Cloud 框架，实现这些需求都很简单，让我们一起来看一下。
>
> ### 在微服务中集成 OAuth2 授权机制
>
> 现在让我们回到 SpringHealth 案例，看看如何基于上一课时构建的 OAuth2 授权服务来完成对单个微服务访问的有效授权。同样，我们还是先关注于 user-service 这个微服务。
>
> 我们知道在 OAuth2 协议中，单个微服务的定位就是资源服务器。Spring Cloud Security 框架为此提供了专门的 @EnableResourceServer 注解。通过在 Bootstrap 类中添加 @EnableResourceServer 注解，相当于就是声明了该服务中的所有内容都是受保护的资源。以 user-service 类为例，示例代码如下所示：
>
> 复制代码
>
> ```
> @SpringCloudApplication
> @EnableResourceServer
> public class UserApplication {
>  
>     public static void main(String[] args) {
>         SpringApplication.run(UserApplication.class, args);
>     }
> }
> ```
>
> 一旦我们在 user-service 中添加了 @EnableResourceServer 注解之后，user-service 会对所有的 HTTP 请求进行验证以确定 Header 部分中是否包含 Token 信息，如果没有 Token 信息，则会直接限制访问。如果有 Token 信息，就会通过访问 OAuth2 服务器并进行 Token 的验证。那么问题就来了，user-service 是如何与 OAuth2 服务器进行通信并获取所传入 Token 的验证结果呢？
>
> 要想回答这个问题，我们要明确将 Token 传递给 OAuth2 授权服务器的目的就是获取该 Token 中包含的用户和授权信息。这样，势必需要在 user-service 和 OAuth2 授权服务器之间建立起一种交互关系，我们可以在 user-service 中添加如下所示的 security.oauth2.resource.userInfoUri 配置项来实现这一目标：
>
> 复制代码
>
> ```
> security:
>   oauth2:
>     resource:
> 	  userInfoUri: http://localhost:8080/userinfo
> ```
>
> 这里的[http://localhost:8080/user 指向 OAuth2](http://localhost:8080/user指向OAuth2)服务中的一个端点，我们需要进行构建。相关代码如下所示：
>
> 复制代码
>
> ```
> @RequestMapping(value = "/userinfo", produces = "application/json")
> public Map<String, Object> user(OAuth2Authentication user) {
>         Map<String, Object> userInfo = new HashMap<>();
>         userInfo.put("user", user.getUserAuthentication().getPrincipal());
>         userInfo.put("authorities", AuthorityUtils.authorityListToSet(
> user.getUserAuthentication().getAuthorities()));
>         return userInfo;
> }
> ```
>
> 这个端点的作用就是为了获取可访问那些受保护服务的用户信息。这里用到了 OAuth2Authentication 类，该类保存着用户的身份（Principal）和权限（Authority）信息。
>
> 当使用 Postman 访问http://localhost:8080/userinfo 端点时，我们就需要传入一个有效的 Token。这里以上一课时生成的 Token“868adf52-f524-4be8-a9e7-24c1c41aa7d6”为例，在 HTTP 请求中添加一个“Authorization”请求头。请注意，因为我们使用的是 bearer 类型的 Token，所以需要在 access_token 的具体值之前加上“bearer”前缀。当然，我们也可以直接在“Authorization”业中选择协议类型为 OAuth 2.0，然后输入 Access Token，这样相当于就是添加了请求头信息，如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/89/6F/Ciqc1F_YaASASJl8AABW-5attyY705.png)
>
> 通过 Token 发起 HTTP 请求示意图
>
> 在后续的 HTTP 请求中，我们都将以这种方式发起对微服务的调用。该请求的结果如下所示：
>
> 复制代码
>
> ```
> {
>      "user":{
>          "password":null,
>          "username":"springhealth_user",
>          "authorities":[
>              {
>                  "autority":"ROLE_USER"
>              }
>          ],
>          "accountNonExpired":true,
>          "accountNonLocker":true,
>          "credentialsNonExpired":true,
>          "enabled":true
>      },
>      "authorities":[
>          "ROLE_USER"
>      ]
>  }
> ```
>
> 我们知道“868adf52-f524-4be8-a9e7-24c1c41aa7d6”这个 Token 是由“springhealth_user”这个用户生成的，可以看到该结果中包含了用户的用户名、密码以及该用户名所拥有的角色，这些信息与我们在上一课时中所初始化的“springhealth_user”用户信息保持一致。我们也可以尝试使用“springhealth_admin”这个用户来重复上述过程。
>
> ### 在微服务中嵌入访问授权控制
>
> 在《服务安全：如何理解微服务访问的安全需求和实现方案？》课时中，我们讨论了作为资源服务器，每个微服务对于自身资源的保护粒度并不是固定的，而是可以根据需求对访问权限进行精细化控制。在 Spring Cloud Security 中对访问的不同控制层级进行了抽象，形成了用户、角色和请求方法这三种粒度，如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/89/7A/CgqCHl_YaBCAAfgqAAAmFvCTcpI745.png)
>
> 用户、角色和请求方法三种控制粒度示意图
>
> 基于上图，我们可以对这三种粒度进行排列组合，形成用户、用户+角色以及用户+角色+请求方法这三种层级，这三种层级所能访问的资源范围逐一递减。所谓的用户层级是指只要是认证用户就可能访问服务内的各种资源。而用户+角色层级在用户层级的基础上，还要求用户属于某一个或多个特定角色。最后的用户+角色+请求方法层级要求最高，能够对某些HTTP操作进行访问限制。接下来我们分别对这三种层级展开讨论。
>
> #### 用户层级的权限访问控制
>
> 在上一课时中，我们已经熟悉了通过扩展各种 ConfigurerAdapter 类来实现自定义配置信息的方法。对于资源服务器而言，也存在一个 ResourceServerConfigurerAdapter 类。在 SpringHealth 案例系统中，为了实现用户层级的控制，我们的做法同样是在 user-service 中创建一个继承了该类的 SpringHealthResourceServerConfiguration 类并覆写它的 configure 方法，如下所示：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthResourceServerConfiguration extends ResourceServerConfigurerAdapter {
>  
>     @Override
>     public void configure(HttpSecurity httpSecurity) throws Exception{
>         httpSecurity.authorizeRequests()
>              .anyRequest()
>              .authenticated();
>     }
> }
> ```
>
> 注意到这个方法的入参是一个 HttpSecurity 对象，而上述配置中的 anyRequest().authenticated() 方法指定了访问该服务的任何请求都需要进行验证。因此，当我们使用普通的 HTTP 请求来访问 user-service 中的任何 URL 时，将会得到一个“unauthorized”的 401 错误信息。解决办法就是在 HTTP 请求中设置“Authorization”请求头并传入一个有效的 Token 信息，你可以模仿前面的示例做一些练习。
>
> #### 用户+角色层级的权限访问控制
>
> 对于某些安全性要求比较高的资源，我们不应该开放资源访问入口给所有的认证用户，而是需要限定访问资源的角色。就 SpringHealth 案例系统而言，显然，我们认为 intervention-service 服务涉及健康干预这一核心业务流程，会对用户的健康管理产生直接的影响，所以不应该开放给普通用户，而是应该限定只有角色为“ADMIN”的管理员才能访问该服务。要想达到这种效果，实现方式也比较简单，就是在 HttpSecurity 中通过 antMatchers() 和 hasRole() 方法指定想要限制的资源和角色。我们在 intervention-service 中创建一个新的 SpringHealthResourceServerConfiguration 类并覆写它的 configure 方法，如下所示：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthResourceServerConfiguration extends 
> 	ResourceServerConfigurerAdapter{
>  
>     @Override
> 	public void configure(HttpSecurity httpSecurity) throws Exception {
> 	 
>         httpSecurity.authorizeRequests()
>                 .antMatchers("/interventions/**")
>                 .hasRole("ADMIN")
>                 .anyRequest()
>                 .authenticated();
>     }
> }
> ```
>
> 现在，如果我们使用角色为“User”的 Token 访问 invervention-service，就会得到一个“access_denied”的错误信息。然后，我们使用在上一课时中初始化的一个具有“ADMIN”角色的用户“springhealth_admin”来创建新的 Token，并再次访问 intervention-service 服务就能得到正常的返回结果。
>
> #### 用户+角色+操作层级的权限访问控制
>
> 更进一步，我们还可以针对某个端点的某个具体 HTTP 方法进行控制。假设在 SpringHealth 案例系统中，我们认为对 device-service 中的"/devices/"端点下的资源进行更新的风险很高，那么就可以在 HttpSecurity 的 antMatchers() 中添加 HttpMethod.PUT 限定。
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthResourceServerConfiguration extends ResourceServerConfigurerAdapter {
>  
>     @Override
>     public void configure(HttpSecurity httpSecurity) throws Exception{
>         httpSecurity.authorizeRequests()
>                 .antMatchers(HttpMethod.PUT, "/devices/**")
>                 .hasRole("ADMIN")
>                 .anyRequest()
>                 .authenticated();
>     }
> }
> ```
>
> 现在，我们使用普通“USER”角色生成的 Token，并调用 device-service 中"/devices/"端点中的 Update 操作，同样会得到“access_denied”错误信息。而尝试使用“ADMIN”角色生成的 Token 进行访问，就可以得到正常响应。
>
> ### 在微服务中传播 Token
>
> 让我们再次回到 SpringHealth 案例系统，以添加健康干预这一业务场景为例，就涉及 intervention-service 同时调用 user-service 和 device-service 的实现过程，我们来回顾一下这一场景下的代码结构，如下所示：
>
> 复制代码
>
> ```
> public Intervention generateIntervention(String userName, String deviceCode) {
>         Intervention intervention = new Intervention();
>  
>         //获取远程 User 信息
>         UserMapper user = getUser(userName);
>         …
>         
>         //获取远程 Device 信息
>         DeviceMapper device = getDevice(deviceCode);
>         …
>  
>         //创建并保存 Intervention 信息      
>  
>         interventionRepository.save(intervention);
>  
>         return intervention;
> }
> ```
>
> 这样在控制单个微服务访问授权的基础上，就需要确保 Token 在这三个微服务之间进行有效的传播，如下图所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/89/6F/Ciqc1F_YaB2AIia3AABOnfqhLUo283.png)
>
> 微服务中 Token 传播示意图
>
> 持有 Token 的客户端访问 intervention-service 提供的 HTTP 端点进行下单操作，该服务会验证所传入 Token 的有效性。intervention-service 会再通过网关访问 user-service 和 device-service，这两个服务同样分别对所传入 Token 进行验证并返回相应的结果。
>
> 如何实现上图中的 Token 传播效果？Spring Security 基于 RestTemplate 进行了封装，专门提供了一个用于在 HTTP 请求中传播 Token 的 OAuth2RestTemplate 工具类。想要在业务代码中构建一个 OAuth2RestTemplate 对象，可以使用如下所示的示例代码：
>
> 复制代码
>
> ```
> @Bean
> public OAuth2RestTemplate oauth2RestTemplate(
> 	OAuth2ClientContext oauth2ClientContext,
>         OAuth2ProtectedResourceDetails details) {
>  
>         return new OAuth2RestTemplate(details, oauth2ClientContext);
> }
> ```
>
> 可以看到，通过传入 OAuth2ClientContext 和 OAuth2ProtectedResourceDetails，我们就可以创建一下 OAuth2RestTemplate 类。OAuth2RestTemplate 会把从 HTTP 请求头中获取的 Token 保存到一个 OAuth2ClientContext 上下文对象中，而 OAuth2ClientContext 会把每个用户的请求信息控制在会话范围内，以确保不同用户的状态分离。另一方面，OAuth2RestTemplate 还依赖于 OAuth2ProtectedResourceDetails 类，该类封装了上一课时中介绍过的 clientId、客户端安全码 clientSecret、访问范围 scope 等属性。
>
> 一旦 OAuth2RestTemplate 创建成功，我们就可以使用它来对 SpringHealth 原有的服务交互流程进行重构。我们来到 intervention-service 中的 UserServiceClient 类中，重构之后的代码如下所示：
>
> 复制代码
>
> ```
> @Component
> public class UserServiceClient {
>  
>      @Autowired
> 	OAuth2RestTemplate restTemplate;
>  
>     public UserMapper getUserByUserName(String userName){
>      
>         ResponseEntity<UserMapper> restExchange =
>                 restTemplate.exchange(
>                         "http://userservice/users/{userName}",
>                         HttpMethod.GET,
>                         null, UserMapper.class, userName);
>                 
>         UserMapper user = restExchange.getBody();
>         
>         return user;
>     }
> }
> ```
>
> 显然，对于原有的实现方式而言，我们唯一要做的就是使用 OAuth2RestTemplate 来替换原有的 RestTemplate，所有关于 Token 传播的细节已经被完整得封装在每次请求中。对于 DeviceServiceClient 类而言，重构方式完全一样。
>
> 最后，我们通过 Postman 来验证以上流程的正确性。通过访问 Zuul 中配置的 intervention-service 端点，并传入角色为“ADMIN”的用户对应的 Token 信息，可以看到健康干预记录已经被成功创建。你可以尝试通过生成不同的 Token 来执行这一流程，并验证授权效果。
>
> ### 小结与预告
>
> 本课时关注于对服务访问进行授权。通过今天课程的学习，我们明确了基于 Token 在微服务中嵌入访问授权控制的三种粒度，并基于 SpringHealth 案例给出了这三种粒度之下的控制实现方式。同时，在微服务系统中，因为涉及多个服务之间进行交互，所以也需要将 Token 在这些服务之间进行有效的传播。借助于 Spring Cloud Security 为我们提供的工具类，我们可以很轻松地实现这些需求。
>
> 这里给你留一道思考题：你能描述对服务访问进行授权的三种层级，以及每个层级对应的实现方法吗？
>
> 介绍完授权机制之后，接下来要讨论的是认证问题。在下一课时中，我们将详细介绍 JWT 机制的实现过程以及它提供的扩展性。