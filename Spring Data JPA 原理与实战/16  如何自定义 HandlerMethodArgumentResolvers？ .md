[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4716)



> 上一讲我们介绍了 SpringDataWebConfiguration 类的用法，那么这次我们来看一下这个类是如何被加载的，PageableHandlerMethodArgumentResolver 和 SortHandlerMethodArgumentResolver 又是如何生效的，以及如何定义自己的 HandlerMethodArgumentResolvers 类，还有没有其他 Web 场景需要我们自定义呢？
>
> 关于上述几个类，你要先在心里有点印象，我们接下来一个一个详细讲解。
>
> ### Page 和 Sort 参数原理
>
> 想要知道分页和排序参数的加载原理，我们可以通过源码发现是 @EnableSpringDataWebSupport 将这个类加载进去的，其关键代码如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/67/F1/CgqCHl-ifKuAZqvLAAGaihgL6z0625.png)
>
> 其中，@EnableSpringDataWebSupport 注解是上一讲讲解的核心，即 Spring Data JPA 对 Web 支持需要开启的入口，由于我们使用的是 Spring Boot，所以 @EnableSpringDataWebSupport 不需要我们手动去指定。
>
> 这是由于 Spring Boot 有自动加载的机制，我们会发现 org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration 类里面引用了 @EnableSpringDataWebSupport 的注解，所以也不需要我们手动去引用了。这里面的关键代码如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/67/E5/Ciqc1F-ifLGAXeScAACYeXQaXt0313.png)
>
> 而 Spring Boot 的自动加载的核心文件就是 spring.factories 文件，那么我们打开 spring-boot-autoconfigure-2.3.3.jar 包，看一下 spring.factories 文件内容，可以找到 SpringDataWebAutoConfiguration 这个配置类，如下：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/67/F1/CgqCHl-ifLiATqQNAADQjUYmp3o182.png)
>
> 所以可以得出结论：只要是 Spring Boot 项目，我们什么都不需要做，它就会天然地让 Spring Data JPA 支持 Web 相关的操作。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/67/F1/CgqCHl-ifL2AJfQTAAA5uE86eqs914.png)
>
> 而 PageableHandlerMethodArgumentResolver 和 SortHandlerMethodArgumentResolver 两个类是通过 SpringDataWebConfiguration 加载进去的，所以我们基本可以知道 Spring Data JPA 的 Page 和 Sort 参数是因为 SpringDataWebConfiguration 里面 @Bean 的注入才生效的。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/67/E6/Ciqc1F-ifMKAEkZ7AAD0bB-3aYU721.png)
>
> 通过 PageableHandlerMethodArgumentResolver 和 SortHandlerMethodArgumentResolver 这两个类的源码，我们可以分析出它们分别实现了 Spring MVC Web 框架里面的 org.springframework.web.method.support.HandlerMethodArgumentResolver 这个接口，从而对 Request 里面的 Page 和 Sort 的参数做了处理逻辑和解析逻辑。
>
> 那么在实际工作中，可能存在特殊情况需要对其进行扩展，比如 Page 的参数可能需要支持多种 Key 的情况，那么我们应该怎么做呢？下面通过 HandlerMethodArgumentResolver 的用法来学习一下。
>
> ### HandlerMethodArgumentResolver 用法
>
> #### HandlerMethodArgumentResolvers 详解
>
> 熟悉 MVC 的人都知道，HandlerMethodArgumentResolvers 在 Spring MVC 中的主要作用是对 Controller 里面的方法参数做解析，即可以把 Request 里面的值映射到方法的参数中。我们打开此类的源码会发现只有两个方法，如下所示：
>
> 复制代码
>
> ```
> public interface HandlerMethodArgumentResolver {
>    //检查方法的参数是否支持处理和转化
>    boolean supportsParameter(MethodParameter parameter);
>    //根据reqest上下文，解析方法的参数
>    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
>          NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
> }
> ```
>
> 此接口的应用场景非常广泛，我们可以看到其子类非常多，如下图所示：
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/67/F1/CgqCHl-ifNKAUr5ZAAKB24GVNXo607.png)
>
> 其中几个类的作用如下：
>
> - PathVariableMapMethodArgumentResolver 专门解析 @PathVariable 里面的值；
> - RequestResponseBodyMethodProcessor 专门解析带 @RequestBody 注解的方法参数的值；
> - RequestParamMethodArgumentResolver 专门解析 @RequestParam 的注解参数的值，当方法的参数中没有任何注解的时候，默认是 @RequestParam；
> - 以及我们上一讲提到的 PageableHandlerMethodArgumentResolver 和 SortHandlerMethodArgumentResolver。
>
> 到这里你会发现，我们上一讲还讲解了 HttpMessageConverter，那么它和 HandlerMethodArgumentResolvers 是什么关系呢？我们接着看。
>
> #### HandlerMethodArgumentResolvers 与 HttpMessageConverter 的关系
>
> 我们打开 RequestResponseBodyMethodProcessor 就会发现，这个类中主要处理的是，方法里面带 @RequestBody 注解的参数，如下图所示：
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/67/E6/Ciqc1F-ifNqALsgiAAPVRBCs4Q4327.png)
>
> 而其中的 readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType()) 方法，如果我们点进去继续观察，发现里面会根据 Http 请求的 MediaType，来选择不同的 HttpMessageConverter 进行转化。
>
> 所以到这里你可以很清楚 HandlerMethodArgumentResolvers 与 HttpMessageConverter 的关系了，即不同的 HttpMessageConverter 都是由 RequestResponseBodyMethodProcessor 进行调用的。
>
> 那么调用关系我们知道了，如此多的 HttpMessageConverter 之间是通过什么顺序执行的呢？
>
> #### HttpMessageConverter 的执行顺序
>
> 当我们自定义 HandlerMethodArgumentResolver 时，通过下面的方法加载进去。
>
> 复制代码
>
> ```
> @Override
> public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
>    resolvers.add(myPageableHandlerMethodArgumentResolver);
> }
> ```
>
> 在 List`<HandlerMethodArgumentResolver>` 里面自定义的 resolver 的优先级是最高的，也就是会优先执行 HandlerMethodArgumentResolver 之后，才会按照顺序执行系统里面自带的那一批 HttpMessageConverter，按照 List 的循环顺序一个一个执行。
>
> Spring 里面有个执行效率问题，就是一旦一次执行找到了需要的 HandlerMethodArgumentResolver 的时候，利用 Spring 中的缓存机制，执行过程中就不会再遍历 List`<HandlerMethodArgumentResolver>` 了，而是直接用上次找到的 HandlerMethodArgumentResolver，这样提升了执行效率。
>
> 如果想要了解更多的 Resolver，你可以看下图这个类，我不一一细说了。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/67/F1/CgqCHl-ifQaAX5YQAAN2flcVp0c284.png)
>
> 那么了解了这么多，能否举个实战的例子呢？
>
> ### 自定义 HandlerMethodArgumentResolver 实战
>
> 在实际的工作中，你可能会遇到对老项目进行改版的工作，如果要我们把旧的 API 接口改造成 JPA 的技术实现，那么可能会出现需要新、老参数的问题。假设在实际场景中，我们 Page 的参数是 page[number]，而 page size 的参数是 page[size]，看看应该怎么做。
>
> **第一步：新建 MyPageableHandlerMethodArgumentResolver。**
>
> 这个类的作用有两个：
>
> 1. 用来兼容 ?page[size]=2&page[number]=0 的参数情况；
> 2. 支持 JPA 新的参数形式 ?size=2&page=0。
>
> 我们通过自定义的 MyPageableHandlerMethodArgumentResolver 来实现这个需求，请看下面这段代码。
>
> 复制代码
>
> ```
> /**
>  * 通过@Component把此类加载到Spring的容器里面去 
>  */
> @Component
> public class MyPageableHandlerMethodArgumentResolver extends PageableHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
>    //我们假设sort的参数没有发生变化，采用PageableHandlerMethodArgumentResolver里面的写法
>    private static final SortHandlerMethodArgumentResolver DEFAULT_SORT_RESOLVER = new SortHandlerMethodArgumentResolver();
>    //给定两个默认值
>    private static final Integer DEFAULT_PAGE = 0;
>    private static final Integer DEFAULT_SIZE = 10;
>    //兼容新版，引入JPA的分页参数
>    private static final String JPA_PAGE_PARAMETER = "page";
>    private static final String JPA_SIZE_PARAMETER = "size";
>    //兼容原来老的分页参数
>    private static final String DEFAULT_PAGE_PARAMETER = "page[number]";
>    private static final String DEFAULT_SIZE_PARAMETER = "page[size]";
>    private SortArgumentResolver sortResolver;
>    //模仿PageableHandlerMethodArgumentResolver里面的构造方法
>    public MyPageableHandlerMethodArgumentResolver(@Nullable SortArgumentResolver sortResolver) {
>       this.sortResolver = sortResolver == null ? DEFAULT_SORT_RESOLVER : sortResolver;
>    }
>    
>    @Override
>    public boolean supportsParameter(MethodParameter parameter) {
> //    假设用我们自己的类MyPageRequest接收参数
>       return MyPageRequest.class.equals(parameter.getParameterType());
>       //同时我们也可以支持通过Spring Data JPA里面的Pageable参数进行接收，两种效果是一样的
> //    return Pageable.class.equals(parameter.getParameterType());
>    }
>    /**
>     * 参数封装逻辑page和sort，JPA参数的优先级高于page[number]和page[size]参数
>     */
>     //public Pageable resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) { //这种是Pageable的方式
>    @Override
>    public MyPageRequest resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
>       String jpaPageString = webRequest.getParameter(JPA_PAGE_PARAMETER);
>       String jpaSizeString = webRequest.getParameter(JPA_SIZE_PARAMETER);
>       //我们分别取参数里面page、sort和 page[number]、page[size]的值
>       String pageString = webRequest.getParameter(DEFAULT_PAGE_PARAMETER);
>       String sizeString = webRequest.getParameter(DEFAULT_SIZE_PARAMETER);
>       //当两个都有值时候的优先级，及其默认值的逻辑
>       Integer page = jpaPageString != null ? Integer.valueOf(jpaPageString) : pageString != null ? Integer.valueOf(pageString) : DEFAULT_PAGE;
>       //在这里同时可以计算 page+1的逻辑;如：page=page+1;
>       Integer size = jpaSizeString != null ? Integer.valueOf(jpaSizeString) : sizeString != null ? Integer.valueOf(sizeString) : DEFAULT_SIZE;
>        //我们假设，sort排序的取值方法先不发生改变
>       Sort sort = sortResolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
> //    如果使用Pageable参数接收值，我们也可以不用自定义MyPageRequest对象，直接返回PageRequest;
> //    return PageRequest.of(page,size,sort);
>       //将page和size计算出来的记过封装到我们自定义的MyPageRequest类里面去
>       MyPageRequest myPageRequest = new MyPageRequest(page, size,sort);
>       //返回controller里面的参数需要的对象；
>       return myPageRequest;
>    }
> }
> ```
>
> 你可以通过代码里面的注释仔细看一下其中的逻辑，其实这个类并不复杂，就是取 Request 的 Page 相关的参数，封装到对象中返回给 Controller 的方法参数里面。其中 MyPageRequest 不是必需的，我只是为了给你演示不同的做法。
>
> **第二步：新建 MyPageRequest。**
>
> 复制代码
>
> ```
> /**
>  * 继承父类，可以省掉很多计算page和index的逻辑
>  */
> public class MyPageRequest extends PageRequest {
>    protected MyPageRequest(int page, int size, Sort sort) {
>       super(page, size, sort);
>    }
> }
> ```
>
> 此类，我们用来接收 Page 相关的参数值，也不是必需的。
>
> **第三步：implements WebMvcConfigurer 加载 myPageableHandlerMethodArgumentResolver。**
>
> 复制代码
>
> ```
> /**
>  * 实现WebMvcConfigurer
>  */
> @Configuration
> public class MyWebMvcConfigurer implements WebMvcConfigurer {
>    @Autowired
>    private MyPageableHandlerMethodArgumentResolver myPageableHandlerMethodArgumentResolver;
>    /**
>     * 覆盖这个方法，把我们自定义的myPageableHandlerMethodArgumentResolver加载到原始的mvc的resolvers里面去
>     * @param resolvers
>     */
>    @Override
>    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
>       resolvers.add(myPageableHandlerMethodArgumentResolver);
>    }
> }
> ```
>
> 这里我利用 Spring MVC 的机制加载我们自定义的 myPageableHandlerMethodArgumentResolver，由于自定义的优先级是最高的，所以用 MyPageRequest.class
>
> 和 Pageable.class 都是可以的。
>
> **第四步：我们看下 Controller 里面的写法。**
>
> 复制代码
>
> ```
> //用Pageable这种方式也是可以的
> @GetMapping("/users")
> public Page<UserInfo> queryByPage(Pageable pageable, UserInfo userInfo) {
>    return userInfoRepository.findAll(Example.of(userInfo),pageable);
> }
> //用MyPageRequest进行接收
> @GetMapping("/users/mypage")
> public Page<UserInfo> queryByMyPage(MyPageRequest pageable, UserInfo userInfo) {
>    return userInfoRepository.findAll(Example.of(userInfo),pageable);
> }
> ```
>
> 你可以看到，这里利用 Pageable 和 MyPageRequest 两种方式都是可以的。
>  **第五步：启动项目测试一下。**
>
> 我们依次可以测试下面两种情况，发现都是可以正常工作的。
>
> 复制代码
>
> ```
> GET http://127.0.0.1:8089/users?page[size]=2&page[number]=0&ages=10&sort=id,desc
> ###
> GET http://127.0.0.1:8089/users?size=2&page=0&ages=10&sort=id,desc
> ###
> GET http://127.0.0.1:8089/users/mypage?page[size]=2&page[number]=0&ages=10&sort=id,desc
> ###
> GET http://127.0.0.1:8089/users/mypage?size=2&page=0&ages=10&sort=id,desc
> ```
>
> 其中，你应该可以注意到，我演示的 Controller 方法里面有多个参数的，每个参数都各司其职，找到自己对应的 HandlerMethodArgumentResolver，这正是 Spring MVC 框架的优雅之处。
>
> 那么除了上面的 Demo，自定义 HandlerMethodArgumentResolver 对我们的实际工作还有什么建议呢？
>
> ### 实际工作的建议
>
> 自定义 HandlerMethodArgumentResolver 到底对我们的实际工作起到哪些作用呢？分为下述几个场景。
>
> #### 场景一
>
> 当我们在 Controller 里面处理某些参数时，重复的步骤非常多，那么我们就可以考虑写一下自己的框架，来处理请求里面的参数，而 Controller 里面的代码就会变得非常优雅，不需要关心其他框架代码，只要知道方法的参数有值就可以了。
>
> #### 场景二
>
> 再举个例子，在实际工作中需要注意的是，默认 JPA 里面的 Page 是从 0 开始，而我们可能有些老的代码也要维护，因为老的代码大多数的 Page 都会从 1 开始。如果我们不自定义 HandlerMethodArgumentResolver，那么在用到分页时，每个 Controller 的方法里面都需要关心这个逻辑。那么这个时候你就应该想到上面列举的自定义 MyPageableHandlerMethodArgumentResolver 的 resolveArgument 方法的实现，使用这种方法我们只需要在里面修改 Page 的计算逻辑即可。
>
> #### 场景三
>
> 再举个例子，在实际的工作中，还经常会遇到“取当前用户”的应用场景。此时，普通做法是，当使用到当前用户的 UserInfo 时，每次都需要根据请求 header 的 token 取到用户信息，伪代码如下所示：
>
> 复制代码
>
> ```
> @PostMapping("user/info")
> public UserInfo getUserInfo(@RequestHeader String token) {
>     // 伪代码
>     Long userId = redisTemplate.get(token);
>     UserInfo useInfo = userInfoRepository.getById(userId);
>     return userInfo;
> }
> ```
>
> 如果我们使用`HandlerMethodArgumentResolver`接口来实现，代码就会变得优雅许多。伪代码如下：
>
> 复制代码
>
> ```
> // 1. 实现HandlerMethodArgumentResolver接口
> @Component
> public class UserInfoArgumentResolver implements HandlerMethodArgumentResolver {
>    private final RedisTemplate redisTemplate;//伪代码，假设我们token是放在redis里面的
>    private final UserInfoRepository userInfoRepository;
>    public UserInfoArgumentResolver(RedisTemplate redisTemplate, UserInfoRepository userInfoRepository) {
>       this.redisTemplate = redisTemplate;//伪代码，假设我们token是放在redis里面的
>       this.userInfoRepository = userInfoRepository;
>    }
>    @Override
>    public boolean supportsParameter(MethodParameter parameter) {
>       return UserInfo.class.isAssignableFrom(parameter.getParameterType());
>    }
>    @Override
>    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
>                           NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
>       HttpServletRequest nativeRequest = (HttpServletRequest) webRequest.getNativeRequest();
>       String token = nativeRequest.getHeader("token");
>       Long userId = (Long) redisTemplate.opsForValue().get(token);//伪代码，假设我们token是放在redis里面的
>       UserInfo useInfo = userInfoRepository.getOne(userId);
>       return useInfo;
>    }
> }
> //2. 我们只需要在MyWebMvcConfigurer里面把userInfoArgumentResolver添加进去即可，关键代码如下：
> @Configuration
> public class MyWebMvcConfigurer implements WebMvcConfigurer {
>    @Autowired
>    private MyPageableHandlerMethodArgumentResolver myPageableHandlerMethodArgumentResolver;
> @Autowired
> private UserInfoArgumentResolver userInfoArgumentResolver;
> @Override
> public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
>    resolvers.add(myPageableHandlerMethodArgumentResolver);
>    //我们只需要把userInfoArgumentResolver加入resolvers中即可
>    resolvers.add(userInfoArgumentResolver);
> }
> }
> // 3. 在Controller中使用
> @RestController
> public class UserInfoController {
>   //获得当前用户的信息
>   @GetMapping("user/info")
>   public UserInfo getUserInfo(UserInfo userInfo) {
>      return userInfo;
>   }
>   //给当前用户 say hello
>   @PostMapping("sayHello")
>   public String sayHello(UserInfo userInfo) {
>     return "hello " + userInfo.getTelephone();
>   }
> }
> ```
>
> 上述代码可以看到，在 Contoller 里面可以完全省掉根据 token 从 redis 取当前用户信息的过程，优化了操作流程。
>
> #### 场景四
>
> 有的时候我们也会更改 Pageable 的默认值和参数的名字，也可以在 application.properties 的文件里面通过如下的 Key 值对自定义进行配置，如下图所示：
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/67/E6/Ciqc1F-ifTSAY0xeAAIfdBh0SkQ798.png)
>
> 关于 Spring MVC 和 Spring Data 相关的参数处理，你通过了解上面的内容并动手操作一下，基本上就可以掌握了。但是实际工作肯定不会这么简单，还会遇到 WebMvcConfigurer 里面其他方法的需求，我顺带给你介绍一下。
>
> ### 思路拓展
>
> #### WebMvcConfigurer 介绍
>
> 当我们做 Spring 的 MVC 开发的时候，可能会通过实现 WebMvcConfigurer 去做一些公用的业务逻辑，下面我列举几个常见的方法，方便你了解。
>
> 复制代码
>
> ```
>  /* 拦截器配置 */
> void addInterceptors(InterceptorRegistry var1);
> /* 视图跳转控制器 */
> void addViewControllers(ViewControllerRegistry registry);
> /**
>   *静态资源处理
> **/
> void addResourceHandlers(ResourceHandlerRegistry registry);
> /* 默认静态资源处理器 */
> void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer);
> /**
>   *这里配置视图解析器
>  **/
> void configureViewResolvers(ViewResolverRegistry registry);
> /* 配置内容裁决的一些选项*/
> void configureContentNegotiation(ContentNegotiationConfigurer configurer);
> /** 解决跨域问题 **/
> void addCorsMappings(CorsRegistry registry) ;
> /** 添加都会contoller的Return的结果的处理 **/
> void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers)；
> ```
>
> 当我们实现 Restful 风格的 API 协议时，会经常看到其对 json 响应结果进行了统一的封装，我们也可以采用 HandlerMethodReturnValueHandler 来实现，再来看一个例子。
>
> #### 用 Result 对 JSON 的返回结果进行统一封装
>
> 下面通过五个步骤来实现一个通过自定义注解，利用**HandlerMethodReturnValueHandler 实现 JSON 结果封装**的例子。
>
> **第一步：我们自定义一个注解 @WarpWithData**，表示此注解包装的返回结果用 Data 进行包装，代码如下：
>
> 复制代码
>
> ```
> @Target({ElementType.TYPE, ElementType.METHOD})
> @Retention(RetentionPolicy.RUNTIME)
> @Documented
> /**
>  * 自定义一个注解对返回结果进行包装
>  */
> public @interface WarpWithData {
> }
> ```
>
> **第二步：自定义 MyWarpWithDataHandlerMethodReturnValueHandler，并继承 RequestResponseBodyMethodProcessor 来实现 HandlerMethodReturnValueHandler 接口**，用来处理 Data 包装的结果，代码如下：
>
> 复制代码
>
> ```
> //自定义自己的return的处理类，我们直接继承RequestResponseBodyMethodProcessor，这样父类里面的方法我们直接使用就可以了
> @Component
> public class MyWarpWithDataHandlerMethodReturnValueHandler extends RequestResponseBodyMethodProcessor implements HandlerMethodReturnValueHandler {
>    //参考父类RequestResponseBodyMethodProcessor的做法
>    @Autowired
>    public MyWarpWithDataHandlerMethodReturnValueHandler(List<HttpMessageConverter<?>> converters) {
>       super(converters);
>    }
>    //只处理需要包装的注解的方法
>    @Override
>    public boolean supportsReturnType(MethodParameter returnType) {
>       return returnType.hasMethodAnnotation(WarpWithData.class);
>    }
>    //将返回结果包装一层Data
>    @Override
>    public void handleReturnValue(Object returnValue, MethodParameter methodParameter, ModelAndViewContainer modelAndViewContainer, NativeWebRequest nativeWebRequest) throws IOException, HttpMediaTypeNotAcceptableException {
>       Map<String,Object> res = new HashMap<>();
>       res.put("data",returnValue);
>       super.handleReturnValue(res,methodParameter,modelAndViewContainer,nativeWebRequest);
>    }
> }
> ```
>
> **第三步：在 MyWebMvcConfigurer 里面直接把 myWarpWithDataHandlerMethodReturnValueHandler 加入 handlers 里面即可**，也是通过覆盖父类 WebMvcConfigurer 里面的 addReturnValueHandlers 方法完成的，关键代码如下：
>
> 复制代码
>
> ```
> @Configuration
> public class MyWebMvcConfigurer implements WebMvcConfigurer {
>    @Autowired
>    private MyWarpWithDataHandlerMethodReturnValueHandler myWarpWithDataHandlerMethodReturnValueHandler;
>    //把我们自定义的myWarpWithDataHandlerMethodReturnValueHandler加入handlers里面即可
>    @Override
>    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
>       handlers.add(myWarpWithDataHandlerMethodReturnValueHandler);
>    }
>    
>   @Autowired
>   private RequestMappingHandlerAdapter requestMappingHandlerAdapter;
>   //由于HandlerMethodReturnValueHandler处理的优先级问题，我们通过如下方法，把我们自定义的myWarpWithDataHandlerMethodReturnValueHandler放到第一个；
>   @PostConstruct
>   public void init() {
>      List<HandlerMethodReturnValueHandler> returnValueHandlers = Lists.newArrayList(myWarpWithDataHandlerMethodReturnValueHandler);
> //取出原始列表，重新覆盖进去；
>         returnValueHandlers.addAll(requestMappingHandlerAdapter.getReturnValueHandlers());
>      requestMappingHandlerAdapter.setReturnValueHandlers(returnValueHandlers);
>   }
> }
> ```
>
> 这里需要注意的是，我们利用 @PostConstruct 调整了一下 HandlerMethodReturnValueHandler 加载的优先级，使其生效。
>
> **第四步：Controller 方法中直接加上 @WarpWithData 注解**，关键代码如下：
>
> 复制代码
>
> ```
> @GetMapping("/user/{id}")
> @WarpWithData
> public UserInfo getUserInfoFromPath(@PathVariable("id") Long id) {
>    return userInfoRepository.getOne(id);
> }
> ```
>
> **第五步：我们测试一下。**
>
> 复制代码
>
> ```
> GET http://127.0.0.1:8089/user/1
> ```
>
> 就会得到如下结果，你会发现我们的 JSON 结果多了一个 Data 的包装。
>
> 复制代码
>
> ```
> {
>   "data": {
>     "id": 1,
>     "version": 0,
>     "createUserId": null,
>     "createTime": "2020-10-23T00:23:10.185Z",
>     "lastModifiedUserId": null,
>     "lastModifiedTime": "2020-10-23T00:23:10.185Z",
>     "ages": 10,
>     "telephone": null,
>     "hibernateLazyInitializer": {}
>   }
> }
> ```
>
> 我们通过五个步骤，利用 Spring MVC 的扩展机制，实现了对返回结果的格式统一处理。不知道你是否掌握了这种方法，希望你可以多多实践，将它运用得更好。
>
> ### 总结
>
> 以上就是这一讲的内容了。在这一讲中，我通过原理分析、语法讲解、实战经验分享，帮助你掌握了 HandlerMethodArgumentResolvers 的详细用法，并为你扩展了学习思路，了解了 HandlerMethodReturnValueHandler 的用法。
>
> 其实 Spring MVC 肯定远不止这些，这里我只介绍了一些和 Spring Data 相关的知识点。你在工作和学习中，要时刻保持好奇心和挖掘精神，不断地探究不理解的知识点。
>
> 最后，如果你觉得有帮助就动动手指分享吧，也欢迎你在评论区留言，一起讨论、进步。下一讲我们将学习数据源相关的知识。到时见~
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa