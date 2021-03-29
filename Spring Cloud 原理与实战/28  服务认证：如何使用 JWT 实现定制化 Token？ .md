[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4773)



> 在上一课时中，我们详细介绍了在微服务架构中，如何使用 Token 对服务的访问过程进行权限控制，这里的 Token 是类似“b7c2c7e0-0223-40e2-911d-eff82d125b80”的一种字符串结构。可能你会问，这个字符串中能包含哪些内容呢？是不是所有的 Token 都是这样的结构吗？事实上，在 OAuth2 协议中，并没有对 Token 具体的组成结构有明确的规定。为了解决 Token 的标准化问题，就诞生了今天我们要介绍的 JWT。
>
> ### 什么是 JWT？
>
> JWT 的全称是 JSON Web Token，所以它本质上就是一种基于 JSON 表示的 Token。JWT 的设计目标就是为 OAuth2 中所使用的 Token 提供一种标准结构，所以它经常与 OAuth2 协议集成在一起使用。
>
> 从结构上讲，JWT 本身是由三段信息构成的，第一段为头部（Header），第二段为有效负载（Payload），第三段为签名（Signature），如下所示：
>
> 复制代码
>
> ```
> header. payload. signature
> ```
>
> 以上三个部分的内容从数据格式上讲都是一个 JSON 对象。在JWT中，每一段 JSON 对象都被 Base64 进行编码，然后编码后的内容用“.”号链接一起。所以本质上 JWT 就是一个字符串，如下所示的就是一个 JWT 字符串的示例：
>
> 复制代码
>
> ```
> eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL3NwcmluZ2hlYWx0aC5leGFtcGxlLmNvbSIsInN1YiI6Im1haWx0bzpzcHJpbmdoZWFsdGhAZXhhbXBsZS5jb20iLCJuYmYiOjE1OTkwNTY4NjIsImV4cCI6MTU5OTA2MDQ2MiwiaWF0IjoxNTk5MDU2ODYyLCJqdGkiOiJpZDEyMzQ1NiIsInR5cCI6Imh0dHBzOi8vc3ByaW5naGVhbHRoLmV4YW1wbGUuY29tL3JlZ2lzdGVyIn0.rlg2i8mWwV-gFjHUSCutX-UBMYrqxL0th1xtyGq7UdE
> ```
>
> 显然，我们无法从这个经过 Base64 编码的字符串中获取任何有用的信息。业界也存在一些在线生成和解析 JWT 的工具，这里以[jsjws](http://kjur.github.io/jsjws/tool_jwt.html)[上所提供的在线工具为例来演示 JWT](http://kjur.github.io/jsjws/tool_jwt.html上所提供的在线工具为例来演示JWT)的内部结构和签名方式。在这个在线工具中，我们首先需要设置一系列的声明（Claim），然后指定签名的算法和键值，如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/01/53/Cip5yF_Yc4WAH7kqAABfUNZNgrk543.png)
>
> 生成 JWT 的步骤
>
> 一旦我们指定了这些内容之后，就可以获取前面所给出的 JWT 字符串。反之，我们可以使用http://jwt.calebb.net/所提供的反向转换原始数据的功能。针对前面的 JWT 字符串，我们可以看到其中所包含的原始 JSON 数据，如下所示：
>
> 复制代码
>
> ```
> {
>  alg: "HS256",
>  typ: "JWT"
> }.
> {
>  iss: "https://springhealth.example.com",
>  sub: "mailto:springhealth@example.com",
>  nbf: 1599056862,
>  exp: 1599060462,
>  iat: 1599056862,
>  jti: "id123456",
>  typ: "https://springhealth.example.com/register"
> }.
> [signature]
> ```
>
> 现在，我们可以清晰地看到一个 JWT 中所包含的 Header 部分和 Payload 部分的数据，而出于安全考虑，Signature 部分数据并没有进行展示。
>
> Spring Cloud Security 为 JWT 的生成和验证提供了开箱即用的支持。当然，要发送和消费 JWT，OAuth2 授权服务和各个受保护的微服务必须以不同的方式进行配置。整个开发流程与《服务授权：如何基于 Spring Cloud Security 集成 OAuth2 协议？》中介绍的普通 Token 是一致的，不同之处在于配置的内容和方式。接下来，我们先来看如何在 OAuth2 授权服务器中配置 JWT。
>
> ### 如何集成 OAuth2 与 JWT？
>
> 对于所有需要用到 JWT 的独立服务，我们都首先需要在 Maven 的 pom 文件中添加对应的依赖包，如下所示：
>
> 复制代码
>
> ```
> <dependency>
> 	<groupId>org.springframework.security</groupId>
> 	<artifactId>spring-security-jwt</artifactId>
> </dependency>
> ```
>
> 可以想象，接下来的一步就是提供一个配置类用于完成 JWT 的生成和转换。事实上，在 OAuth2 协议中专门提供了一个接口用于管理 Token 的存储，这个接口就是 TokenStore，该接口的实现类如下所示：
>
> ![Lark20201224-184609.png](https://s0.lgstatic.com/i/image/M00/8C/12/CgqCHl_kcZeANBHuAAKZw9BsRRo761.png)
>
> TokenStore 接口的实现类
>
> 注意到这里有一个 JwtTokenStore 专门用来存储 JWT Token。对应的，我们也将创建一个用于配置 JwtTokenStore 的配置类。让我们回到 SpringHealth 案例系统中的 auth-server 服务，添加一个 SpringHealthJWTTokenStoreConfig 配置类，如下所示：
>
> 复制代码
>
> ```
> @Configuration
> public class SpringHealthJWTTokenStoreConfig {
>  
>     @Bean
>     public TokenStore tokenStore() {
>         return new JwtTokenStore(jwtAccessTokenConverter());
>     }
>  
>     @Bean
>     public JwtAccessTokenConverter jwtAccessTokenConverter() {
>         JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
>         converter.setSigningKey("123456");
>         return converter;
> 	}
>  
>     @Bean
>     public DefaultTokenServices tokenServices() {
>         DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
>         defaultTokenServices.setTokenStore(tokenStore());
>         defaultTokenServices.setSupportRefreshToken(true);
>         return defaultTokenServices;
>     }
> }
> ```
>
> 可以看到，这里构建了 JwtTokenStore 对象，而在它的构造函数中传入了一个 JwtAccessTokenConverter。JwtAccessTokenConverters 是一个用来转换 JWT 的转换器，而转换的过程需要签名键。在创建完了 JwtTokenStore 之后，我们通过 tokenServices 方法返回了已经设置 JwtTokenStore 对象的 DefaultTokenServices。
>
> SpringHealthJWTTokenStoreConfig 的作用就是创建了一系列对象供 Spring 容器进行使用，那么我们在什么时候会用到这些对象呢？答案就是在将 JWT 集成到 OAuth2 授权服务的过程中，而这个过程似曾相识。基于《服务授权：如何基于 Spring Cloud Security 集成 OAuth2 协议？》课时中的讨论，我们可以构建一个 SpringHealthAuthorizationServerConfigurer 类来覆写 AuthorizationServerConfigurerAdapter 中的 configure 方法。回想原先的这个 configure 方法实现如下：
>
> 复制代码
>
> ```
> @Override
> public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
>  
>         endpoints.authenticationManager(authenticationManager)
>                 .userDetailsService(userDetailsService);
> }
> ```
>
> 而集成了 JWT 之后，该方法的实现过程如下所示：
>
> 复制代码
>
> ```
> @Override
> public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
>         TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
>         tokenEnhancerChain.setTokenEnhancers(Arrays.asList(jwtTokenEnhancer, jwtAccessTokenConverter));
>         endpoints.tokenStore(tokenStore).accessTokenConverter(jwtAccessTokenConverter) 
>                 .tokenEnhancer(tokenEnhancerChain) 
>                 .authenticationManager(authenticationManager)
>                 .userDetailsService(userDetailsService);
> }
> ```
>
> 可以看到，这里构建了一个对 Token 的增强链 TokenEnhancerChain，并用到了在 SpringHealthJWTTokenStoreConfig 中创建的 tokenStore、jwtAccessTokenConverter 对象。至此，在 OAuth2 协议中集成 JWT 的过程介绍完成，也就是说现在我们访问 OAuth2 授权服务器时获取的 Token 应该就是 JWT Token。让我们来尝试一下，通过 Postman，我们发起了如下所示的请求并得到了相应的 Token：
>
> 复制代码
>
> ```
> {
>     "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzeXN0ZW0iOiJTcHJpbmdIZWFsdGgiLCJ1c2VyX25hbWUiOiJzcHJpbmdoZWFsdGhfYWRtaW4iLCJzY29wZSI6WyJ3ZWJjbGllbnQiXSwiZXhwIjoxNjA2MzYyMTM3LCJhdXRob3JpdGllcyI6WyJST0xFX0FETUlOIiwiUk9MRV9VU0VSIl0sImp0aSI6IjU3YjhjYjM5LThkMGYtNGE4Ny1hZDU2LTQyZGExZTIxNmRjYyIsImNsaWVudF9pZCI6InNwcmluZ2hlYWx0aCJ9.kEjhuZtSYrj7HJlQhowfBQDzH9qJiqCQm8p7gHUhhcU",
>     "token_type": "bearer",
>     "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzeXN0ZW0iOiJTcHJpbmdIZWFsdGgiLCJ1c2VyX25hbWUiOiJzcHJpbmdoZWFsdGhfYWRtaW4iLCJzY29wZSI6WyJ3ZWJjbGllbnQiXSwiYXRpIjoiNTdiOGNiMzktOGQwZi00YTg3LWFkNTYtNDJkYTFlMjE2ZGNjIiwiZXhwIjoxNjA4OTEwOTM3LCJhdXRob3JpdGllcyI6WyJST0xFX0FETUlOIiwiUk9MRV9VU0VSIl0sImp0aSI6IjVmOGZkNDFjLTdlMTEtNDk1OC05ZDVmLWFlY2MzNGRmYThiNSIsImNsaWVudF9pZCI6InNwcmluZ2hlYWx0aCJ9.Rq8pLRHSvZgda_0DgSFQ8eI5dctAGE4Jqlb_qabRDvE",
>     "expires_in": 43199,
>     "scope": "webclient",
>     "system": "SpringHealth",
>     "jti": "57b8cb39-8d0f-4a87-ad56-42da1e216dcc"
> }
> ```
>
> 显然，这里的 access_token 和 refresh_token 都已经是经过 Base64 编码的字符串。同样，我们可以通过前面介绍的在线工具来获取其 JSON 数据格式的内容，如下所示的就是 access_token 的原始内容：
>
> 复制代码
>
> ```
> {
>  alg: "HS256",
>  typ: "JWT"
> }.
> {
>  system: "SpringHealth",
>  user_name: "springhealth_admin",
>  scope: [
>   "webclient"
>  ],
>  exp: 1606362137,
>  authorities: [
>   "ROLE_ADMIN",
>   "ROLE_USER"
>  ],
>  jti: "57b8cb39-8d0f-4a87-ad56-42da1e216dcc",
>  client_id: "springhealth"
> }.
> [signature]
> ```
>
> ### 如何在微服务中使用 JWT？
>
> 在各个微服务中使用 JWT 的第一步也是配置工作。我们需要在 SpringHealth 案例系统中的三个业务微服务中分别添加一个 SpringHealthJWTTokenStoreConfig 配置类，这个配置类的内容就是创建一个 JwtTokenStore 并构建 tokenServices，具体代码在前面已经做了介绍，这里不再展开。
>
> 配置工作完成之后，剩下的问题就是在服务调用链中传播 JWT。在上一课时中，我们给出了 OAuth2RestTemplate 这个工具类，该类可以传播普通的 Token。但可惜的是，它并不能传播基于 JWT 的 Token。从实现原理上，OAuth2RestTemplate 也是在 RestTemplate 的基础上做了一层封装，所以我们的思路也是尝试在 RestTemplate 请求中添加对 JWT 的支持。
>
> 我们知道 HTTP 请求是通过在 Header 部分中添加一个“Authorization”消息头来完成对 Token 的传递，所以第一步需要能够从 HTTP 请求中获取这个 JWT Token。然后，我们需要将这个 Token 存储在一个线程安全的地方以便在后续的服务链中进行使用，这是第二步。第三步，也是最关键的一步，就是在通过 RestTemplate 发起请求时，能够把这个 Token 自动嵌入到所发起的每一个 HTTP 请求中。整个实现思路如下图所示：
>
> ![Lark20201224-184612.png](https://s0.lgstatic.com/i/image/M00/8C/07/Ciqc1F_kcaSAR0NjAAEcuT033u0868.png)
>
> 在服务调用链中传播 JWT Token 的三个实现步骤
>
> 实现这一思路需要你对 HTTP 请求的过程和原理有一定的理解，在代码实现上也需要有一些技巧，我来一一进行展开。
>
> 首先，在 HTTP 请求过程中，我们可以通过过滤器 Filter 对所有请求进行过滤。Filter 是 servlet 中的一个核心组件，其基本原理就是构建一个过滤器链并对经过该过滤器链的请求和响应添加定制化的处理机制。Filter 接口的定义如下所示：
>
> 复制代码
>
> ```
> public interface Filter {
>     public void init(FilterConfig filterConfig) throws ServletException;
>     public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException;
>     public void destroy();
> }
> ```
>
> 通常，我们会实现 Filter 接口中的 doFilter 方法。例如，在 SpringHealth 案例系统中，我们可从将 ServletRequest 转化为一个 HttpServletRequest 对象，并从该对象中获取“Authorization”消息头，示例代码如下所示：
>
> 复制代码
>
> ```
> @Component
> public class AuthorizationHeaderFilter implements Filter {
> 
>     @Override
>     public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
>             throws IOException, ServletException {
>  
>         HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
>  
>         AuthorizationHeaderHolder.getAuthorizationHeader().setAuthorizationHeader(httpServletRequest.getHeader(AuthorizationHeader.AUTHORIZATION_HEADER));
> 
>         filterChain.doFilter(httpServletRequest, servletResponse);
>     }
>  
>     @Override
>     public void init(FilterConfig filterConfig) throws ServletException {}
>  
>     @Override
>     public void destroy() {}
> }
> ```
>
> 注意到，这里我们把从 HTTP 请求中获取的“Authorization”消息头保存到了一个 AuthorizationHeaderHolder 对象中。从命名上看，AuthorizationHeader 对象代表的就是 HTTP 中“Authorization” 消息头，而 AuthorizationHeaderHolder 则是该消息头对象的持有者。这种命名方式在 Spring 等主流开源框架中非常常见。一般而言，以 -Holder 结尾的多是一种封装类，用于对原有对象添加线程安全等附加特性。这里的 AuthorizationHeaderHolder 就是这样一个封装类，如下所示：
>
> 复制代码
>
> ```
> public class AuthorizationHeaderHolder {
>     private static final ThreadLocal<AuthorizationHeader> authorizationHeaderContext = new ThreadLocal<AuthorizationHeader>();
>  
>     public static final AuthorizationHeader getAuthorizationHeader(){
>         AuthorizationHeader header = authorizationHeaderContext.get();
>  
>         if (header == null) {
>          header = new AuthorizationHeader();
>             authorizationHeaderContext.set(header);
>  
>         }
>         return authorizationHeaderContext.get();
>     }
>  
>     public static final void setAuthorizationHeader(AuthorizationHeader header) {
>         authorizationHeaderContext.set(header);
>     }
> }
> ```
>
> 可以看到，这里使用了 ThreadLocal 来确保对 AuthorizationHeader 对象访问的线程安全性，AuthorizationHeader 定义如下，用于保存来自 HTTP 请求头的 JWT Token：
>
> 复制代码
>
> ```
> @Component
> public class AuthorizationHeader {
>     public static final String AUTHORIZATION_HEADER = "Authorization";
> 
>     private String authorizationHeader = new String();
>  
>     public String getAuthorizationHeader() {
>         return authorizationHeader;
>     }
>  
>     public void setAuthorizationHeader(String authorizationHeader) {
>         this.authorizationHeader = authorizationHeader;
>     }
> }
> ```
>
> 现在，对于每一个 HTTP 请求，我们都能获取其中的 Token 并将其保存在上下文对象中。剩下的唯一问题就是如何通过 RestTemplate 将这个 Token 继续传递到下一个服务中，以便下一个服务也能从 HTTP 请求中获取 Token 并继续向后传递，从而确保 Token 在整个调用链中持续传播。要想实现这一目标，我们需要对 RestTemplate 进行一些设置，如下所示：
>
> 复制代码
>
> ```
> @Bean
> public RestTemplate getCustomRestTemplate() {
>         RestTemplate template = new RestTemplate();
>         List<ClientHttpRequestInterceptor> interceptors = template.getInterceptors();
>         if (interceptors == null) {
>             template.setInterceptors(Collections.singletonList(new AuthorizationHeaderInterceptor()));
>         } else {
>             interceptors.add(new AuthorizationHeaderInterceptor());
>             template.setInterceptors(interceptors);
>         }
>  
>         return template;
> }
> ```
>
> RestTemplate 允许开发人员添加自定义的拦截器 Interceptor，拦截器本质上与过滤器的功能类似，用于对传入的 HTTP 请求进行定制化处理。例如，上述代码中的 AuthorizationHeaderInterceptor 的作用就是在 HTTP 请求的消息头中嵌入保存在 AuthorizationHeaderHolder 中的 JWT Token，如下所示：
>
> 复制代码
>
> ```
> public class AuthorizationHeaderInterceptor implements ClientHttpRequestInterceptor {
>  
>     @Override
>     public ClientHttpResponse intercept(
>             HttpRequest request, byte[] body, ClientHttpRequestExecution execution)
>             throws IOException {
>  
>         HttpHeaders headers = request.getHeaders();
>         headers.add(AuthorizationHeader.AUTHORIZATION_HEADER, AuthorizationHeaderHolder.getAuthorizationHeader().getAuthorizationHeader());
>  
>         return execution.execute(request, body);
>     }
> }
> ```
>
> 至此，我们已经完成了在微服务中嵌入 JWT 的完整过程。现在，让我们回到 intervention-service 中的 UserServiceClient 类，会发现它重新使用了前面所构建的 RestTemplate 对象来发起远程调用，代码如下所示：
>
> 复制代码
>
> ```
> @Component
> public class UserServiceClient {
> 	 
>  
>      @Autowired
> 	RestTemplate restTemplate;
>  
>     public UserMapper getUserByUserName(String userName){
> 
>         ResponseEntity<UserMapper> restExchange =
>                 restTemplate.exchange(
>                          "http://zuulservice:5555/springhealth/user/users/username/{userName}",
>                        HttpMethod.GET,
>                         null, UserMapper.class, userName);
> 
>         UserMapper user = restExchange.getBody();
> 
>         return user;
>     }
> }
> ```
>
> ### 如何扩展 JWT？
>
> 在本课时的最后，我们来讨论如何扩展 JWT。JWT具有良好的可扩展性，开发人员可以根据需要在 JWT Token 中添加自己想要添加的各种附加信息。
>
> 针对 JWT 的扩展性场景，Spring Security 专门提供了一个 TokenEnhancer 接口来对 Token 进行增强（Enhance），该接口定义如下：
>
> 复制代码
>
> ```
> public interface TokenEnhancer {
>     OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication);
> }
> ```
>
> 可以看到这里处理的是一个 OAuth2AccessToken 接口，而该接口有一个默认的实现类 DefaultOAuth2AccessToken。我们可以通过该实现类的 setAdditionalInformation 方法以键值对的方式将附加信息添加到 OAuth2AccessToken 中，示例代码如下所示：
>
> 复制代码
>
> ```
> public class SpringHealthJWTTokenEnhancer implements TokenEnhancer {
> 
>     @Override
>     public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
>         Map<String, Object> systemInfo= new HashMap<>();
>  
>         systemInfo.put("system", "springhealth");
>  
>         ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(systemInfo);
>         return accessToken;
>     }
> }
> ```
>
> 这里我们以硬编码的方式添加了一个“system”属性，你也可以根据需要进行相应的调整。
>
> 要想使得上述 SpringHealthJWTTokenEnhancer 类能够生效，我们需要对 SpringHealthAuthorizationServerConfigurer 类中的 configure 方法进行重新配置，并将 SpringHealthJWTTokenEnhancer 嵌入到 TokenEnhancerChain 中。事实上，我们在前面的代码中已经演示了这部分内容。
>
> 现在，我们已经扩展了 JWT Token。那么，如何从这个 JWT Token 中获取所扩展的属性呢？方法也比较简单和固定，如下所示：
>
> 复制代码
>
> ```
> //获取 JWTToken
> RequestContext ctx = RequestContext.getCurrentContext();
> String authorizationHeader = ctx.getRequest().getHeader(AUTHORIZATION_HEADER);
> String jwtToken = authorizationHeader.replace("Bearer ","");
> 
> //解析 JWTToken
> String[] split_string = jwtToken.split("\\.");
> String base64EncodedBody = split_string[1];
> Base64 base64Url = new Base64(true);
> String body = new String(base64Url.decode(base64EncodedBody));
> JSONObject jsonObj = new JSONObject(body);
> 
> //获取定制化属性值
> String systemName = jsonObj.getString("system");
> ```
>
> 我们可以把这段代码嵌入到需要使用到自定义“system”属性的任何场景中。
>
> ### 小结与预告
>
> 这是微服务安全性的最后一个课时，关注的是认证问题而不是授权问题，为此我们引入了 JWT 机制。JWT 本质上也是一种 Token，只不过提供了标准化的规范定义，可以与 OAuth2 协议进行集成。而我们使用 JWT 时，也可以将各种信息添加到这种 Token 中并在微服务访问链路中进行传播。同时，作为一种具有高扩展性的 Token 解决方案，我们也可以轻松为 JWT 提交各种定制化的认证信息。
>
> 这里给你留一道思考题：如果想要对 JWT 中的数据进行扩展，你有什么办法？
>
> 介绍完安全性问题之后，下一课时我们将进入到新的一个主题，即服务监控。我们将首先介绍服务监控基本原理以及引入 Spring Cloud Sleuth 框架。