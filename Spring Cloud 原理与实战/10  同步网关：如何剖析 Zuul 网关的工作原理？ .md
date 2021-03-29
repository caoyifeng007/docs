[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4755)



> 上一课时中，我们详细介绍了 Spring Cloud 中的 Zuul 网关的使用方式。作为 API 网关的一种实现方式，Spring Cloud Netflix Zuul 为我们提供了丰富的网关控制功能。今天我们将对它的内部原理进行深入剖析。Spring Cloud Netflix Zuul 同样是基于 Netflix 旗下的 Zuul 组件，本课时内容我们也先从 Netflix Zuul 开始讲起。
>
> ### ZuulFilter 组件架构
>
> Zuul 响应 HTTP 请求的过程是一种典型的过滤器结构，内部提供了 ZuulFilter 组件来实现这一机制。作为切入点，我们先从 ZuulFilter 展开讨论。
>
> #### ZuulFilter 的定义与 ZuulRegistry
>
> 在 Zuul 中，ZuulFilter 是 Zuul 中的关键组件，我们来看一下它在设计上的抽象过程。在 Netflix Zuul 中存在一个 IZuulFilter 接口，定义如下：
>
> 复制代码
>
> ```
> public interface IZuulFilter {
>     boolean shouldFilter();
>     Object run() throws ZuulException;
> }
> ```
>
> 很明显，IZuulFilter 接口的 shouldFilter() 方法决定是否需要执行该过滤器，一般情况该方法都会返回 true，但我们可以使用该方法来根据场景动态设置过滤器是否生效。而 run() 方法显然代表着该 Filter 具体需要实现的业务逻辑。
>
> IZuulFilter的直接实现类是 ZuulFilter，ZuulFilter 是一个抽象类，额外提供了如下所示的两个抽象方法：
>
> 复制代码
>
> ```
> //过滤器类型
> abstract public String filterType();
> //过滤器顺序	 
> abstract public int filterOrder();
> ```
>
> 其中 filterType() 代表过滤器的类型，内置过滤器分为 PRE、ROUTING、POST 和 ERROR 四种，本课时后续内容会对这些过滤器进行具体展开；而 filterOrder() 方法用于设置过滤器的执行顺序，这个顺序用数字进行表示，数字越小则越先执行。
>
> 然后在 ZuulFilter 类中最核心的就是负责执行 Filter 的 runFilter() 方法，该方法的执行流程并不复杂，如下所示：
>
> 复制代码
>
> ```
> public ZuulFilterResult runFilter() {
>         ZuulFilterResult zr = new ZuulFilterResult();
>         if (!isFilterDisabled()) {
>             if (shouldFilter()) {
>                 Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
>                 try {
>                     Object res = run();
>                     zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
>                 } catch (Throwable e) {
>                     t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
>                     zr = new ZuulFilterResult(ExecutionStatus.FAILED);
>                     zr.setException(e);
>                 } finally {
>                     t.stopAndLog();
>                 }
>             } else {
>                 zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
>             }
>         }
>         return zr;
> }
> ```
>
> 既然有了过滤器，系统中就应该存在一个管理过滤器的组件。有时候，我们称这个组件为过滤器链。而在 Netflix Zuul 中，则使用了一个称为过滤器注册表 FilterRegistry 的组件来保存和维护所有 ZuulFilter。FilterRegistry 类定义如下：
>
> 复制代码
>
> ```
> public class FilterRegistry {
>  
>     private static final FilterRegistry INSTANCE = new FilterRegistry();
>  
>     public static final FilterRegistry instance() {
>         return INSTANCE;
>     }
>  
>     private final ConcurrentHashMap<String, ZuulFilter> filters = new ConcurrentHashMap<String, ZuulFilter>();
>  
>     private FilterRegistry() {
>     }
>  
>     public ZuulFilter remove(String key) {
>         return this.filters.remove(key);
>     }
>  
>     public ZuulFilter get(String key) {
>         return this.filters.get(key);
>     }
>  
>     public void put(String key, ZuulFilter filter) {
>         this.filters.putIfAbsent(key, filter);
>     }
>  
>     public int size() {
>         return this.filters.size();
>     }
>  
>     public Collection<ZuulFilter> getAllFilters() {
>         return this.filters.values();
>     } 
> }
> ```
>
> FilterRegistry 的实现意外得简单，这里采用了经典的单例模式来创建 FilterRegistry 实例。在 FilterRegistry 内部，可以看到就是直接使用了线程安全的 ConcurrentHashMap 来缓存 ZuulFilter。
>
> 有了 ZuulFilter 的定义以及保存它的媒介，接下来我们重点讨论 Netflix Zuul 中加载各种 ZuulFilter 的过程。
>
> #### ZuulFilter 的加载与 FilterLoader
>
> 在 Zuul 中，FilterLoader 负责 ZuulFilter 的加载。FilterLoader 类是 Zuul 的一个核心类，这点从它的注释说明中就能看出：它用来在源码变化时编译、载入和校验过滤器。针对这个核心类，我们将从它的变量入手，对它进行分析。
>
> 复制代码
>
> ```
> //Filter 文件名与修改时间的映射
> private final ConcurrentHashMap<String, Long> filterClassLastModified = new ConcurrentHashMap<String, Long>();
> //Filter 名称与代码的映射
> private final ConcurrentHashMap<String, String> filterClassCode = new ConcurrentHashMap<String, String>();
> //Filter 名称与名称的映射，作用是判断该 Filter 是否存在
> private final ConcurrentHashMap<String, String> filterCheck = new ConcurrentHashMap<String, String>();
> //Filter 类型与 List<ZuulFilter> 的映射
> private final ConcurrentHashMap<String, List<ZuulFilter>> hashFiltersByType = new ConcurrentHashMap<String, List<ZuulFilter>>();
> //前面提到的 FilterRegistry 单例
> private FilterRegistry filterRegistry = FilterRegistry.instance();
> //动态代码编译器实例，Zuul 提供的默认实现是 GroovyCompiler
> static DynamicCodeCompiler COMPILER;
> //ZuulFilter 工厂类
> static FilterFactory FILTER_FACTORY = new DefaultFilterFactory();
> ```
>
> 这些变量中唯一值得注意的就是 DynamicCodeCompiler。顾名思义，它一种动态代码编译器。DynamicCodeCompiler 接口定义如下：
>
> 复制代码
>
> ```
> public interface DynamicCodeCompiler {
>     Class compile(String sCode, String sName) throws Exception;
>  
>     Class compile(File file) throws Exception;
> }
> ```
>
> 可以看到 DynamicCodeCompiler 提供了两个接口，一个是从代码编译到类，另一个是从文件编译到类。GroovyCompiler 是该接口的实现类，用于把 Groovy 代码编译为 Java 类。GroovyCompile 的实现如下所示：
>
> 复制代码
>
> ```
> public class GroovyCompiler implements DynamicCodeCompiler {
>  
>     GroovyClassLoader getGroovyClassLoader() {
>         return new GroovyClassLoader();
>     }
>  
>     @Override
>     public Class compile(String sCode, String sName) {
>         GroovyClassLoader loader = getGroovyClassLoader();
>         Class groovyClass = loader.parseClass(sCode, sName);
>         return groovyClass;
>     }
>  
>     @Override
>     public Class compile(File file) throws IOException {
>         GroovyClassLoader loader = getGroovyClassLoader();
>         Class groovyClass = loader.parseClass(file);
>         return groovyClass;
> 	}
> }
> ```
>
> 注意， GroovyCompile 的核心代码同样比较简单。我们通过 groovy.lang.GroovyClassLoader 构建一个类加载器，然后调用它的 parseClass 方法即可将一个 Groovy 文件加载为一个 Java 类。
>
> 有了 GroovyCompile 提供的动态加载和编译代码的能力，我们再回到 FilterLoader。FilterLoader 分别提供了 getFilter、putFilter 和 getFiltersByType 这三个工具方法，其中实际涉及加载和存储 Filter 的方法只有 putFilter，该方法如下所示：
>
> 复制代码
>
> ```
> public boolean putFilter(File file) throws Exception {
>         String sName = file.getAbsolutePath() + file.getName();
>         if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
>             LOG.debug("reloading filter " + sName);
>             filterRegistry.remove(sName);
>         }
>         ZuulFilter filter = filterRegistry.get(sName);
>         if (filter == null) {
>             Class clazz = COMPILER.compile(file);
>             if (!Modifier.isAbstract(clazz.getModifiers())) {
>                 filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
>                 List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
>                 if (list != null) {
>                     hashFiltersByType.remove(filter.filterType()); 
>                 }
>                 filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
>                 filterClassLastModified.put(sName, file.lastModified());
>                 return true;
>             }
>         }
>  
>         return false;
> }
> ```
>
> 上述 putFilter 方法通过文件加载 ZuulFilter，整体执行流程也比较简洁明了。首先通过修改时间判断文件是否被修改过，如果被修改过则从 FilterRegistry 移除这个 Filter。然后根据 FilterRegistry 是否缓存了这个 Filter 来决定是否使用前面介绍 DynamicCodeCompiler 动态加载代码，而根据加载的代码再使用 FilterFactory 工厂类来实例化 ZuulFilter。
>
> FilterFactory 的实现类是 DefaultFilterFactory，它的实例创建过程也很直接，就是利用了反射机制，如下所示：
>
> 复制代码
>
> ```
> @Override
> public ZuulFilter newInstance(Class clazz) throws InstantiationException, IllegalAccessException {
>         return (ZuulFilter) clazz.newInstance();
> }
> ```
>
> 顺着 FilterLoader 类的 putFilter 方法继续往前走，我们需要明确该方法中传入的 File 对象从何而来。实际上这个 File 对象是由 FilterFileManager 类提供的。顾名思义，这个类的作用是管理这些 File 对象。FilterFileManager 中包含了如下所示的 manageFiles 和 processGroovyFiles 方法，后者调用了 FilterLoader 的 putFilter 方法以进行 Groovy 文件的处理。
>
> 复制代码
>
> ```
> void manageFiles() throws Exception, IllegalAccessException, InstantiationException {
>         List<File> aFiles = getFiles();
>         processGroovyFiles(aFiles);
> }
> 	 
> void processGroovyFiles(List<File> aFiles) throws Exception, InstantiationException, IllegalAccessException {
>         for (File file : aFiles) {
>             FilterLoader.getInstance().putFilter(file);
>         }
> }
> ```
>
> 那么这个 manageFiles 方法是什么时候被调用的呢？注意到在 FilterFileManager 中存在一个每隔 pollingIntervalSeconds 秒就会轮询一次的后台守护线程，该线程就会调用 manageFiles 方法来重新加载 Filter，如下所示：
>
> 复制代码
>
> ```
> void startPoller() {
>         poller = new Thread("GroovyFilterFileManagerPoller") {
>             public void run() {
>                 while (bRunning) {
>                     try {
>                         sleep(pollingIntervalSeconds * 1000);
>                         manageFiles();
>                     } catch (Exception e) {
>                         e.printStackTrace();
>                     }
>                 }
>             }
>         };
>         poller.setDaemon(true);
>         poller.start();
> }
> ```
>
> 分析到这里，我们对 ZuulFilter 的加载和管理流程已经非常清晰了。Zuul 通过文件来存储各种 ZuulFilter 的定义和实现逻辑，然后启动一个守护线程定时轮询这些文件，确保变更之后的文件能够动态加载到 FilterRegistry 中。
>
> #### RequestContext 与上下文
>
> 使用过 Zuul 的同学都知道其中的 RequestContext 对象。我们可以通过该对象将业务信息放到请求上下文（Context）中，并使其在各个 ZuulFilter 中进行传递。上下文机制是众多开源软件中所必备的一种需求和实现方案。我们将通过学习 Zuul 中的 RequestContext 来掌握针对 Context 的处理技巧。
>
> 为了掌握 Context 处理技巧，我们必须有一种共识，即每个新的请求都是由一个独立的线程进行处理，诸如 Tomcat 之类的服务器会为我们启动了这个线程。也就是说，HTTP 请求的所有参数总是绑定在处理请求的线程中。基于这一点，我们不难想象，对 RequestContext 的访问必须设计成线程安全，Zuul 使用了非常常见和实用的 ThreadLocal，如下所示：
>
> 复制代码
>
> ```
> protected static final ThreadLocal<? extends RequestContext> threadLocal = new ThreadLocal<RequestContext>() {
>         @Override
>         protected RequestContext initialValue() {
>             try {
>                 return contextClass.newInstance();
>             } catch (Throwable e) {
>                 throw new RuntimeException(e);
>             }
>         }
> };
> ```
>
> 可以看到，这里使用了线程安全的 ThreadLocal 来存放所有的 RequestContext。作为一个实现上的技巧，请注意，这里重写了 ThreadLocal 的 initialValue() 方法。重写目的是确保ThreadLocal 的 get() 方法始终会获取一个 RequestContext 实例。这样做的原因是因为默认情况下 ThreadLocal 的 get() 方法可能会返回 null，而通过重写 initialValue() 方法可以在返回 null 时自动初始化一个 RequestContext 实例。
>
> 用于获取 RequestContext 的 getCurrentContext 方法如下所示：
>
> 复制代码
>
> ```
> public static RequestContext getCurrentContext() {
> 	    RequestContext context = threadLocal.get();
>         return context;
> }
> ```
>
> 我们在开发业务代码时可以先获取这个 RequestContext 对象，进而获取该对象中所存储的各种业务数据。RequestContext 为我们提供了大量的 getter/setter 方法对来完成这一操作。与 Zuul 中的其他组件一样，通过分析代码，我们发现 RequestContext 的实现也是直接对 ConcurrentHashMap 做了一层封装，其中 Key 是 String 类型，而 Value 是 Object 类型，因此我们可以将任何想要传递的数据放到 RequestContext 中。
>
> #### HTTP 请求与过滤器执行
>
> 接下去我们分析如何使用加载好的 ZuulFilter。Zuul 提供了 FilterProcessor 类来执行 Filter，参考如下所示的 runFilters 方法：
>
> 复制代码
>
> ```
> public Object runFilters(String sType) throws Throwable {
> 
>         boolean bResult = false;
>         List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
>         if (list != null) {
>             for (int i = 0; i < list.size(); i++) {
>                 ZuulFilter zuulFilter = list.get(i);
>                 Object result = processZuulFilter(zuulFilter);
>                 if (result != null && result instanceof Boolean) {
>                     bResult |= ((Boolean) result);
>                 }
>             }
>         }
>         return bResult;
> }
> ```
>
> 注意这个方法传入了 Filter 的类型，然后通过 FilterLoader 获取所属类型的 ZuulFilter 列表。通过遍历列表并调用 processZuulFilter 方法来完成对每个 Filter 的执行。processZuulFilter 方法的结构也比较简单，核心就是通过调用 ZuulFilter 自身的 runFilter 方法，然后获取执行结果状态。
>
> 我们同时看到 FilterProcessor 基于 runFilters 方法衍生出 postRoute()、preRoute()、error() 和 route() 这四个方法，分别对应 Zuul 中的四种过滤器类型。其中 PRE 过滤器在请求到达目标服务器之前调用，ROUTING 过滤器把请求发送给目标服务，POST 过滤器在请求从目标服务返回之后执行，而 ERROR 过滤器则在发生错误时执行。同时，这四种过滤器有一定的执行顺序，如下所示：
>
> ![Lark20201022-135138.png](https://s0.lgstatic.com/i/image/M00/61/FC/CgqCHl-RHgmAPr6fAABM8NTETsI789.png)
>
> Zuul 过滤器生命周期（翻译自 Zuul 官网）
>
> 在上图中，可以看到每一种过滤器对应了一个 HTTP 请求的执行生命周期。因此，我们可以在不同的过滤器中添加针对该生命周期的特定执行逻辑。例如，在日常开发过程中，常见的做法是在 PRE 过滤器中执行身份验证和记录请求日志等操作；而在 POST 过滤器返回的响应中添加消息头或者各种统计信息等。
>
> 基于对 Zuul 请求生命周期的理解，我们最后来看一下 ZuulServlet。在 Spring Cloud 中所有请求都是 HTTP 请求，Zuul 作为一个服务网关同样也需要完成对 HTTP 请求的响应。ZuulServlet 是 Zuul 中对 HttpServlet 接口的一个实现类。我们知道 Servlet 中最核心的就是 service 方法，ZuulServlet 的 service 方法如下所示：
>
> 复制代码
>
> ```
> @Override
> public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
>         try {
>             init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
>  
>             RequestContext context = RequestContext.getCurrentContext();
>             context.setZuulEngineRan();
>  
>             try {
>                 preRoute();
>             } catch (ZuulException e) {
>                 error(e);
>                 postRoute();
>                 return;
>             }
>             try {
>                 route();
>             } catch (ZuulException e) {
>                 error(e);
>                 postRoute();
>                 return;
>             }
>             try {
>                 postRoute();
>             } catch (ZuulException e) {
>                 error(e);
>                 return;
>             }
>  
>         } catch (Throwable e) {
>             error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
>         } finally {
>             RequestContext.getCurrentContext().unset();
>         }
> }
> ```
>
> 注意这里的 postRoute()、preRoute()、error() 和 route() 这四个方法同样只是对 ZuulRunner 中同名方法的直接封装。你可以尝试通过这段代码梳理 Zuul 中四种过滤器的执行顺序。
>
> ### Spring Cloud 集成 ZuulFilter
>
> 针对 Zuul 的整体架构，在设计上的初衷是把所有 ZuulFilter 通过 Groovy 文件的形式进行管理和加载。但是，很遗憾，在 Spring Cloud 体系中并没有使用此策略来加载 ZuulFilter。那么 Spring Cloud 的 ZuulFilter是如何实现加载的呢？
>
> spring-cloud-netflix-zuul 同样是一个 Spring Boot 应用程序，所以我们还是直接进入 ZuulServerAutoConfiguration 类，并找到了如下所示的 ZuulFilterConfiguration 类：
>
> 复制代码
>
> ```
> @Configuration
> protected static class ZuulFilterConfiguration {
>         @Autowired
>         private Map<String, ZuulFilter> filters;
>  
>         @Bean
>         public ZuulFilterInitializer zuulFilterInitializer(
>                 CounterFactory counterFactory, TracerFactory tracerFactory) {
>             FilterLoader filterLoader = FilterLoader.getInstance();
>             FilterRegistry filterRegistry = FilterRegistry.instance();
>             return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory, filterLoader, filterRegistry);
>         }
> }
> ```
>
> 这里我们看到了熟悉的 FilterLoader 和 FilterRegistry，通过 ZuulFilterInitializer 的构造函数引入了过滤器加载的流程。同时，我们发现 filters 变量是一个 Key 为 String、Value 为 ZuulFilter 的 Map，而该变量上添加了 @Autowired 注解。这就意味着 Spring 会把 ZuulFilter 的 bean 自动装载到 Map 对象中。
>
> 我们在 ZuulServerAutoConfiguration 以及它的子类 ZuulProxyAutoConfiguration 中发现了很多添加了 @Bean 注解并以“Filter”结尾的类，其形式如下所示：
>
> 复制代码
>
> ```
> @Bean
> public ServletDetectionFilter servletDetectionFilter() {
>         return new ServletDetectionFilter();
> }
> ```
>
> 这个 ServletDetectionFilter 类就是 ZuulFilter 的子类，在 Spring 容器中会被自动注入ZuulFilterConfiguration 的 filters 对象中。
>
> 获取了容器中自动注入的 ZuulFilter 之后，我们继续跟进 ZuulFilterInitializer，发现了如下所示的 contextInitialized 方法：
>
> 复制代码
>
> ```
> @PostConstruct
> public void contextInitialized() {
>         log.info("Starting filter initializer");
>  
>         TracerFactory.initialize(tracerFactory);
>         CounterFactory.initialize(counterFactory);
>  
>         for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
>             filterRegistry.put(entry.getKey(), entry.getValue());
>         }
> }
> ```
>
> 注意到该方法开启了 @PostConstruct 注解，因此在 Spring 容器中，一旦 ZuulFilterInitializer 的构造函数执行完成，就会执行这个 contextInitialized 方法。而在该方法中，我们通过 FilterRegistry 的 put 方法把从容器中自动注入的各个 Filter 添加到 FilterRegistry 中。同样，在 ZuulFilterInitializer 也存在一个被添加了 @PostConstruct 注解的 contextDestroyed 方法，该方法中调用了 filterRegistry 的 remove 方法删除 Filter。
>
> 至此，Spring Cloud 通过 Netflix Zuul 提供的 FilterLoader 和 FilterRegistry 等工具类完成了两者之间的集成。
>
> ### 小结与预告
>
> 在上一课时的基础上，本课时详细解析了 Zuul 网关的内部构造和实现原理。在 Zuul 中，最核心的组件就是过滤器组件 ZuulFilter，我们围绕 ZuulFilter 的定义、加载、存储和管理给出源码级的分析过程。同时，我们也基于一个 HTTP 请求的处理过程，描绘了 Zuul 中的过滤器类型以及处理机制。
>
> 这里给你留一道思考题：在 Zuul 中，处理请求上下文时使用了哪种实现方式类确保线程安全性？
>
> 在介绍完 Zuul 之后，下一课时我们将介绍另一款主流的 API 网关，这就是 Spring Cloud 中自研的 Spring Cloud Gateway。