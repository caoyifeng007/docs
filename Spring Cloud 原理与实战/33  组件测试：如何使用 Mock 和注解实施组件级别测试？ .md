[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4778)



> 在上一课时中，我们全面介绍了针对微服务架构的测试方案。我们提出在测试微服务架构中需要直接面对的两个核心问题，即如何验证组件级别的正确性以及如何验证服务级别的正确性。本课时和下一课时的内容将分别围绕这两个核心问题进行展开，今天让我们先来看一下组件级别的测试方法和工程实践。
>
> ### 组件级别的测试方案
>
> 在上一课时中，我们已经讨论到使用 Mock 来对组件进行测试。Mock 是一种策略而不是技术，今天我们就需要给出如何实现 Mock 的技术体系。假设在 intervention-service 中存在这样一个 InterventionService 类，其中包含一个 getInterventionById 方法，如下所示：
>
> 复制代码
>
> ```
> @Service
> public class InterventionService {
> 
>     public Intervention getInterventionById(Long id) {
>         …
> 	}
> }
> ```
>
> 那么，如何对这个方法进行 Mock 呢？通常，我们可以使用 easymock、jmockMock 等工具包来隐式实现这个方法。对于某一个或一些被测试对象所依赖的方法而言，编写 Mock 相对简单，只需要模拟被使用的方法即可。在这个例子中，如果依赖于 InterventionService，我们只需要给出 getInterventionById 方法的实现。
>
> 让我们回到单个微服务的内部，涉及组件级别测试的维度有很多，包括数据访问 Repository 层、服务构建 Service 层和提供外部端点的 Controller 层。同时，基于常见的代码组织结构，组件测试也体现为一种层次关系，即我们需要测试从 Repository 层到 Service 层再到 Controller 层的完整业务链路。
>
> 另一方面，Spring Boot 也内置了一个测试模块可以用于组件级别的测试场景。在该模块中，提供了一批非常有用的注解来简化测试过程，要想使用这些注解，我们需要引入 spring-boot-starter-test 依赖，如下所示：
>
> 复制代码
>
> ```
> <dependency>
>       <groupId>org.springframework.boot</groupId>
>       <artifactId>spring-boot-starter-test</artifactId>
>       <scope>test</scope>
> </dependency>
> ```
>
> 首先，因为 Spring Boot 程序的入口是 Bootstrap 类，Spring Boot 专门提供了一个 @SpringBootTest 注解来测试你的 Bootstrap 类，使用方法如下所示：
>
> 复制代码
>
> ```
> @SpringBootTest(classes = UserApplication.class, 
> 	webEnvironment = SpringBootTest.WebEnvironment.MOCK)
> ```
>
> 在 Spring Boot 中，@SpringBootTest 注解主要用于测试基于自动配置的 ApplicationContext，它允许你来设置测试上下文中的 Servlet 环境。在多数场景下，一个真实的 Servlet 环境对于测试而言过于重量级，所以我们一般通过 WebEnvironment.MOCK 环境来模拟测试环境。
>
> 我们知道对于一个 Spring Boot 应用程序而言，Bootstrap 类中的 main() 入口通过 SpringApplication.run() 方法将启动 Spring 容器。如下所示的 intervention-service 中的启动类 InterventionApplication：
>
> 复制代码
>
> ```
> @SpringBootApplication
> public class InterventionApplication {
>  
>     public static void main(String[] args) {
>         SpringApplication.run(InterventionApplication.class, args);
>     }
> }
> ```
>
> 针对这个 BootStrap 类，我们可以通过编写测试用例的方式来验证 Spring 容器是否能够正常启动，该测试用例如下所示：
>
> 复制代码
>
> ```
> package com.tianyalan.testing.orders;
> 	 
> import org.junit.Assert;
> import org.junit.Test;
> import org.junit.runner.RunWith;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.boot.test.context.SpringBootTest;
> import org.springframework.context.ApplicationContext;
> import org.springframework.test.context.junit4.SpringRunner;
>  
> @SpringBootTest
> @RunWith(SpringRunner.class)
> public class ApplicationContextTests {
>  
>   @Autowired
>   private ApplicationContext applicationContext;
>  
>   @Test
>   public void testContextLoaded() throws Throwable {
>       Assert.assertNotNull(this.applicationContext);
>   }
> }
> ```
>
> 我们看到这里用到了 @SpringBootTest 注解和 @RunWith 注解。前面已经介绍了 @SpringBootTest 注解，而 @RunWith 注解由 JUnit 框架提供，用于设置测试运行器，例如我们可以通过 @RunWith(SpringRunner.class) 让测试运行于 Spring 测试环境中。
>
> 同时，我们在 testContextLoads 方法上添加了一个 @Test 注解，该注解来自 JUnit 框架，代表该方法为一个有效的测试用例。这里测试的场景是指对 Spring 中的 ApplicationContext 作了非空验证。执行该测试用例，我们从输出的控制台信息中看到 Spring Boot 应用程序被正常启动，同时测试用例本身也会给出执行成功的提示。
>
> 在验证完容器可以正常启动之后，我们继续来看一个 Spring Boot 应用程序的其他组件层。对于 Repository 层而言，主要的交互媒介是数据库，所以 Spring Boot 专门提供了一个 @DataJpaTest 注解来模拟基于 JPA 规范的数据访问过程。同样，对于 Controller 层而言，Spring Boot 也提供了一个 @WebMvcTest 注解来模拟 Web 交互的测试场景。
>
> 讲到这里，你可能会奇怪，为什么 Service 层没有专门的测试注解呢？实际上原因也很简单，因为对于 Repository 层和 Controller 层组件而言，它们都涉及与某一种特定技术体系的交互，Repository 层的交互对象是数据库，而 Controller 层的交互对象是 Web 请求，所以需要专门的测试注解。而 Service 层因为主要是业务代码，并没有跟具体某一项技术体系有直接的关联，所以我们在测试过程中只需要充分使用 Mock 机制就可以了。下图展示了一个业务微服务中各层的测试方法：
>
> ![Lark20210112-140800.png](https://s0.lgstatic.com/i/image/M00/8D/54/CgqCHl_9PPWAEejIAAHVEFm_lKo487.png)
>
> 组件测试的层次和实现方式
>
> 接下来，我们就将对上图中的三个层次和对应的实现方法分别展开讨论。
>
> ### Repository 层：@DataJpaTest 注解
>
> 对于业务微服务而言，一般都涉及数据持久化，我们将首先从数据持久化的角度出发讨论如何对 Repository 层进行测试，并引入 @DataJpaTest 注解。@DataJpaTest 注解会自动注入各种 Repository 类，并会初始化一个内存数据库及访问该数据库的数据源。为了演示方便，我们使用 h2 作为内存数据库，并通过 Mysql 实现数据持久化，因此需要引入以下 Maven 依赖。
>
> 复制代码
>
> ```
> <dependency>
>        <groupId>com.h2database</groupId>
>        <artifactId>h2</artifactId>
> </dependency>
>  
> <dependency>
>        <groupId>mysql</groupId>
>        <artifactId>mysql-connector-java</artifactId>
>        <scope>runtime</scope>
> </dependency>
> ```
>
> 让我们回顾 SpringHealth 案例系统中的 intervention-service 中的 InterventionRepository 接口，如下所示：
>
> 复制代码
>
> ```
> public interface InterventionRepository extends JpaRepository<Intervention, Long> {
>  
>     List<Intervention> findInterventionsByUserId(@Param("userId") String userId);
> 
>     List<Intervention> findInterventionsByDeviceId(@Param("deviceId") String deviceId);
> }
> ```
>
> 注意到这里 InterventionRepository 扩展了 Spring Data 中的 JpaRepository 接口。针对该 InterventionRepository 接口的测试用例如下所示：
>
> 复制代码
>
> ```
> @RunWith(SpringRunner.class)
> @DataJpaTest
> public class InterventionRepositoryTest {
> 
>     @Autowired
>     private TestEntityManager entityManager;
>  
>     @Autowired
>     private InterventionRepository interventionRepository;
>  
>     @Test
>     public void testFindInterventionByUserId() throws Exception {
>         this.entityManager.persist(new Intervention(1L, 1L, 100F, "Intervention1", new Date()));
>         this.entityManager.persist(new Intervention(1L, 2L, 200F, "Intervention2", new Date()));
> 
>         Long userId = 1L;
>         List<Intervention> interventions = this.interventionRepository.findInterventionsByUserId(userId);
>         assertThat(interventions).size().isEqualTo(2);
>         Intervention actual = interventions.get(0);
>         assertThat(actual.getUserId()).isEqualTo(userId);
>     }
>  
>     @Test
>     public void testFindInterventionByNonExistedUserId() throws Exception {
>         this.entityManager.persist(new Intervention(1L, 1L, 100F, "Intervention1", new Date()));
>         this.entityManager.persist(new Intervention(1L, 2L, 200F, "Intervention2", new Date()));
> 
>         Long userId = 3L;
>         List<Intervention> interventions = this.interventionRepository.findInterventionsByUserId(userId);
>         assertThat(interventions).size().isEqualTo(0);
>     }
> }
> ```
>
> 可以看到这里使用了 @DataJpaTest 以完成 InterventionRepository 的注入。同时，我们还注意到另一个核心测试组件 TestEntityManager，该类内部定义了一个 EntityManagerFactory 变量，而 EntityManagerFactory 能够构建数据持久化操作所需要的 EntityManager 对象。所以，TestEntityManager 的效果相当于不使用真正的 InterventionRepository 来完成数据的持久化，从而提供了一种数据与环境之间的隔离机制。TestEntityManager 中所包含的方法如下所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/04/D7/Cip5yF_2vaWAH0x0AAArKL9ORpE536.png)
>
> TestEntityManager 中的方法定义列表
>
> 基于 InterventionRepository 中的方法定义以及我们初始化的数据，以上测试用例的结果显而易见。你可以尝试执行这些单元测试，并观察控制台的日志输出，从这些日志中可以看出各种 SQL 语句的效果。
>
> ### Service 层：Mock
>
> [前面我们已经介绍了 @SpringBootTest 注解中的 SpringBootTest.WebEnvironment.MOCK](mailto:前面我们已经介绍了@springboottest注解中的SpringBootTest.WebEnvironment.MOCK)选项，该选项用于加载 WebApplicationContext 并提供一个 Mock 的 Servlet 环境，内置的 Servlet 容器并没有真实的启动。现在，我们就针对 Service 层来演示这种测试方式。
>
> InterventionService 类的 generateIntervention 方法是其最核心的方法，涉及对 user-service 和 device-service 的远程调用，让我们做一些回顾：
>
> 复制代码
>
> ```
> public Intervention generateIntervention(String userName, String deviceCode) {
> 
>         logger.debug("Generate intervention record with user: {} from device: {}", userName, deviceCode);
> 
>         Intervention intervention = new Intervention();
>  
>         //获取远程 User 信息
>         UserMapper user = getUser(userName);
>         if (user == null) {
>             return intervention;
>         }
>         logger.debug("Get remote user: {} is successful", userName);
> 
>         //获取远程 Device 信息
>         DeviceMapper device = getDevice(deviceCode);
>         if (device == null) {
>             return intervention;
>         }
>         logger.debug("Get remote device: {} is successful", deviceCode);
>  
>         //创建并保存 Intervention 信息
>         intervention.setUserId(user.getId());
>         intervention.setDeviceId(device.getId());
>         intervention.setHealthData(device.getHealthData());
>         intervention.setIntervention("InterventionForDemo");
>         intervention.setCreateTime(new Date());
>  
>         interventionRepository.save(intervention);
>  
>         return intervention;
> }
> ```
>
> 请注意以上代码中的 getUser 方法和 getDevice 方法中涉及了远程访问。以 getUser 方法为例，就会基于 UserServiceClient 发送HTTP请求，我们在前面的课程中都已经介绍过这个类，这里也做一下回顾：
>
> 复制代码
>
> ```
> @Component
> public class UserServiceClient {
>  
>     @Autowired
>     RestTemplate restTemplate;
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
> 对于测试而言，InterventionService 类实际上不需要关注这个 UserServiceClient 中如何实现远程访问的具体过程，因为对于测试过程而言只需要关注方法调用返回的结果。所以，我们对于 UserServiceClient 以及 DeviceServiceClient 同样将采用 Mock 机制完成隔离。针对 InterventionService 的测试用例代码如下所示，可以看到我们采用的是同样的测试方式：
>
> 复制代码
>
> ```
> @RunWith(SpringRunner.class)
> @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
> public class InterventionServiceTests {
>  
>     @MockBean
>     private UserServiceClient userClient;
>  
>     @MockBean
>     private DeviceServiceClient deviceClient;
>  
>     @MockBean
>     private InterventionRepository interventionRepository;
>  
>     @Autowired
>     private InterventionService interventionService;
> 
>     @Test
>     public void testGenerateIntervention() throws Exception {
>         String userName = "springhealth_user1";
>         String deviceCode = "device1";
> 
>         given(this.userClient.getUserByUserName(userName))
>             .willReturn(new UserMapper(1L, "user1", userName));
>         given(this.deviceClient.getDevice(deviceCode))
>             .willReturn(new DeviceMapper(1L, "便携式血压计", "device1", "Sphygmomanometer", 100F));
>  
>         Intervention actual = interventionService.generateIntervention(userName, deviceCode);
>  
>         assertThat(actual.getHealthData()).isEqualTo(100L);
>     }
> }
> ```
>
> 这里同样基于 mockito 对 UserServiceClient 和 DeviceServiceClient 这两个远程访问类的返回结果做了模拟。上述测试用例演示了在 Service 层中进行集成测试的各种手段，这些手段已经能够满足一般场景的需要。
>
> ### Controller 层：@WebMvcTest 注解
>
> 我们再回到 intervention-service 来看看如何对 InterventionController 进行测试。InterventionController 类的功能非常简单，基本都是对 InterventionService 的直接封装，代码如下所示：
>
> 复制代码
>
> ```
> @RestController
> @RequestMapping(value="interventions")
> public class InterventionController {
> 
>     @Autowired
>     private InterventionService interventionService;
> 
>     @RequestMapping(value = "/{userName}/{deviceCode}", method = RequestMethod.POST)
>     public Intervention generateIntervention( @PathVariable("userName") String userName,
>             @PathVariable("deviceCode") String deviceCode) {
>         Intervention intervention = interventionService.generateIntervention(userName, deviceCode);
> 
>         return intervention;
>     }
> 
>     @RequestMapping(value = "/{id}", method = RequestMethod.GET)
>     public Intervention getIntervention(@PathVariable Long id) {
>         Intervention intervention = interventionService.getInterventionById(id);
> 
>      return intervention;
>     }
> }
> ```
>
> 在测试 Controller 类之前，我们先介绍一个新的注解 @WebMvcTest，该注解将初始化测试 Controller 所必需的 Spring MVC 基础设施。InterventionController 类的测试用例如下所示：
>
> 复制代码
>
> ```
> import org.junit.Test;
> import org.junit.runner.RunWith;
> import static org.mockito.BDDMockito.given;
> import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
> import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
>  
> @RunWith(SpringRunner.class)
> @WebMvcTest(InterventionController.class)
> public class InterventionControllerTests {
>  
>     @Autowired
>     private MockMvc mvc;
>  
>     @MockBean
>     private InterventionService interventionService;
>  
>     @Test
>     public void testGenerateIntervention() throws Exception {
>         String userName = "springhealth_user1";
>         String deviceCode = "device1";
>         Intervention intervention = new Intervention(100L, 1L, 1L, 100F, "Intervention1", new Date());
>  
>         given(this.interventionService.generateIntervention(userName, deviceCode))
>                 .willReturn(intervention);
>  
>         this.mvc.perform(post("/interventions/" + userName+ "/" + deviceCode).accept(MediaType.APPLICATION_JSON)).andExpect(status().isOk());
>     }
> }
> ```
>
> 以上代码的关键是 MockMvc 工具类，对这个工具类我们有必要展开一下。MockMvc 类提供的一系列基础方法来满足对 Controller 层组件的测试需求。首先，我们首先需要声明发送 HTTP 请求的方式，MockMvc 类中的一组 get/post/put/delete 方法用来初始化一个 HTTP 请求。然后我们可以使用 param 方法来为该请求添加参数。一旦请求构建完成，perform 方法负责执行请求，并自动将请求映射到相应的 Controller 进行处理。执行完请求之后就是验证结果，这时候可以使用 andExpect、andDo 和 andReturn 等方法对返回的数据进行判断来验证 HTTP 请求执行结果是否正确。
>
> 执行该测试用例，我们从输出的控制台日志中不难发现整个流程相当于启动了 InterventionController 并执行远程访问，而 InterventionController 中所用到的 InterventionService 则做了 Mock。显然测试 InterventionController 的目的在于验证请求是否成功发送和返回，所以我们通过 perform、accept 和 andExpect 方法最终模拟 HTTP 请求的整个过程并验证结果的正确性。
>
> ### 小结与预告
>
> 今天的课程讨论了如何对单个微服务中的各个组件进行测试，我们大量使用到了 Spring 框架中的测试注解。作为小结，这里通过一张表格来对这些注解做一个梳理，如下所示：
>
> ![Lark20210112-140808.png](https://s0.lgstatic.com/i/image2/M01/05/2D/Cip5yF_9POSAVgg2AAJcUWeqOuc506.png)
>
> 这里给你留一道思考题：如果我们想要对所依赖的组件的行为进行模拟，可以使用什么方法？
>
> 讲完组件级别的测试方法之后，下一课时，我们将关注于基于服务级别测试用例的设计，并将引入 Spring Cloud Contract 框架来实施这一过程。