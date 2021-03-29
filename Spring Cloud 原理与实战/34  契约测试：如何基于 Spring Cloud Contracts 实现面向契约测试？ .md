[Spring Cloud 原理与实战 - 资深架构师 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=492#/detail/pc?id=4779)



> 在上一课时中，我们介绍了组件级别的测试方案和实现方法。组件级别的测试关注于单个微服务的内部，而今天要介绍的面向契约测试则是一种服务级别的测试方法，关注于整个微服务系统中的数据和状态传递过程。Spring Cloud Contract 是 Spring Cloud 中专门用于实现面向契约测试的开发框架，对面向契约的端到端测试过程进行了抽象和提炼，并梳理出一套完整的解决方案。让我们一起来看一下。
>
> ### 什么是 Spring Cloud Contract？
>
> 在引入 Spring Cloud Contract 之前，我们需要先明确在测试领域中另一个非常重要的概念，即 Stub，也就是打桩。Stub 与 Mock 经常被混淆，因为他们都可以用来替代真实的依赖对象，从而实现对测试对象的隔离效果。然而，Stub 和 Mock 的区别也非常明显，从类的实现方式上看，Stub 必须有一个显式的类实现，这个实现类需要提供被替代对象的所有逻辑，即使是不需要关注的方法也至少要给出空实现。而 Mock 则不同，它只需要实现自己感兴趣的方法即可，这点在上一课时中已经得到了体现。
>
> 回到 SpringHealth 案例系统，我们来看基于 Stub 的测试场景，如下图所示：
>  ![图片1.png](https://s0.lgstatic.com/i/image2/M01/05/5A/Cip5yF__uIuAbLcFAAHK4CP0YMY943.png)
>
> 基于 Stub 的 SpringHealth 案例系统测试场景
>
> 
>
> 
>
> 在上图中，对于 intervention-service 而言，我们希望不需要真正启动所依赖的 user-service 和 device-service 就能完成服务契约的正确性验证。要想实现这一目标，这里的 user-service 和 device-service 就需要提供对应的 Stub 供 intervention-service 进行使用。
>
> 当使用 Spring Cloud 开发微服务系统时，集成 Spring Cloud Contract 来作为面向契约的测试工具是最佳选择。Spring Cloud Contract 中提供了契约验证器 Contract Verifier 和 Stub 执行器 Stub Runner 等核心组件，这些组件可以确保能够正确模拟服务端的接口，并在契约发生变化时，让服务端和消费端立即能够发现这种变化。
>
> 在接下来的内容中，我们将基于 intervention-service 与 user-service 之间的调用关系来讨论如何基于 Spring Cloud Contract 实现面向契约的端对端测试。我们知道，从业务场景上讲，user-service 相当于服务的提供者，而 intervention-service 是 user-service 的消费者。无论是服务的提供者还是消费者，都需要导入关于 Spring Cloud Contract 的 Maven 依赖，如下所示：
>
> 复制代码
>
> ```
> <dependency>
> 	    <groupId>org.springframework.cloud</groupId>
> 	    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
>         <scope>test</scope>
> </dependency>
>  
> <dependency>
>         <groupId>org.springframework.cloud</groupId>
>         <artifactId>spring-cloud-starter-contract-verifier</artifactId>
>         <scope>test</scope>
> </dependency>
>  
> <dependency>
>         <groupId>org.springframework.cloud</groupId>
>         <artifactId>spring-cloud-contract-wiremock</artifactId>
>         <scope>test</scope>
> </dependency>
> ```
>
> 基于 Spring Cloud Contract 实现面向契约测试的开发流程比较特殊，也有一定的复杂性，在具体编写案例代码之前，我们有必要先对这个流程做一个梳理，如下所示：
>
> ![图片2.png](https://s0.lgstatic.com/i/image2/M01/05/5C/CgpVE1__uKyACLiNAAJkLYEm61s912.png)
>
> 基于 Spring Cloud Contract 实现面向契约测试的开发流程
>
> 针对上图，我们先站在服务提供者的角度来看这个流程。显然，服务提供者需要编写契约文件。请注意，Spring Cloud Contract 中的契约文件并不是一个普通的 Java 文件，而是一个支持动态语言的 groovy 文件。有了契约文件之后，基于 Spring Cloud Contract 内置的 Stub 处理机制，我们自动生成一个 Stub 文件，而这个 Stub 文件实际上就是一个 jar 包。然后，我们需要把这个 Stub 文件上传到 Maven 仓库，供服务的消费者进行使用。显然，这里的 Maven 仓库一般指的是我们自己搭建的 nexus 私服。
>
> 我们接着讨论服务消费者。对于消费者而言，我们会编写并执行针对契约的测试用例。在执行过程中，Spring Cloud Contract 中的 Stub Runner 组件就会从 Maven 仓库中下载 Stub 文件并使用一个内嵌的 Tomcat 服务器来启动 Stub 服务，这样服务消费者就可以基于既定的测试用例来开展端到端测试。
>
> ### 如何使用 Spring Cloud Contracts 实现面向契约测试？
>
> 在接下来的内容中，我们即将围绕 SpringHealth 案例系统给出实现这些步骤的详细过程以及示例代码。
>
> #### 服务提供者制定服务契约
>
> 对于 user-service 而言，我们首先要提供了一个 HTTP 端点，所以我们实现了如下所示的 UserController 类：
>
> 复制代码
>
> ```
> @RestController
> @RequestMapping(value = "users")
> public class UserController {
>  
>     @Autowired
>     private UserRepository repository;
> 
>     @RequestMapping(path = "/userlist")
>     public UserList getUserList() {
>         UserList userList = new UserList();
>         userList.setData(repository.findAll());
>         return userList;
>     }
> }
> ```
>
> 为了演示的简单性，这里省略了 Service 层实现类，而是直接在 Controller 层中调用 Repository 层组件并返回一些数据。在面向契约的测试过程中，这个 UserController 的具体细节其实并不重要，因为我们关注的是服务的对外契约而不是内部实现。
>
> 然后，我们在引入 Spring Cloud Contract Verifier 组件之后我们就可以使用该组件来定义契约。前面提到 Spring Cloud 中的契约文件的表现形式是 groovy 文件，这里我们就定义一个 UserContract.groovy 契约文件，如下所示：
>
> 复制代码
>
> ```
> import org.springframework.cloud.contract.spec.Contract
> import org.springframework.http.HttpHeaders
> import org.springframework.http.MediaType
>  
> Contract.make {
>     description "return all users"
>  
>     request {
>         url "/users/userlist"
>         method GET()
>     }
>  
>     response {
>         status 200
>         headers {
>             header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_UTF8_VALUE)
>         }
>         body("data": [
> 	[id: 1L, userCode: "user1", userName: "springhealth_user1"], 
>                   [id: 2L, userCode: "user2", userName: "springhealth_user2"],
> [id: 3L, userCode: "user3", userName: "springhealth_user3"]])
>     }
> }
> ```
>
> 我们看到以上契约文件中包含三个部分，即 description、request 和 response。其中 description 是对该契约提供的描述信息；request 则定义了请求时的 url 和 method，而 response 显然对返回的 headers 和 body 信息进行了约定。该契约描述的语义也一目了然，就是通过 /users/userlist 这个 URL 来获取一个 JSON 格式的 User 对象列表，该列表将返回三个用户信息。
>
> #### 服务提供者生成 Stub 文件
>
> 定义完契约文件之后，接下来我们就可以生成 Stub 文件。Stub 文件在表现形式上也是一个 jar 包，这个 jar 包的目的就是可以被消费者拿来当作一个模拟服务进行启动并在本地运行测试用例，而不需要服务提供者真正启动服务。
>
> 我们首先需要在 user-service 中引入 spring-cloud-contract-maven-plugin 插件，spring-cloud-contract-maven-plugin 插件的使用方式如下，该插件在 Maven 打包过程中会自动创建 Stub 文件。
>
> 复制代码
>
> ```
> <build>
>       <plugins>
>          <plugin>
>             <groupId>org.springframework.boot</groupId>
>             <artifactId>spring-boot-maven-plugin</artifactId>
>          </plugin>
>          <plugin>
>             <groupId>org.springframework.cloud</groupId>
>               <artifactId>spring-cloud-contract-maven-plugin
> 	</artifactId>
>             <extensions>true</extensions>
>             <configuration>
>                 <packageForbaseClasses>
> 	       com.springhealth.user
> 	</packageForbaseClasses>
>             </configuration>
>           </plugin>
>        </plugins>
> </build>
> ```
>
> 现在我们通过 mvn install –DskipTests=true 命令打包 user-service，除了普通的日志输出之外，控制台还会生成如下信息（为了显示简单做了裁剪）：
>
> 复制代码
>
> ```
> 	[INFO] Copying file UserContract.groovyy
> 	[INFO] Converting from Spring Cloud Contract Verifier contracts to WireMock stubs mappings
> 	[INFO]      Spring Cloud Contract Verifier contracts directory: …\user-testing-service\src\test\resources\contracts
> 	[INFO] WireMock stubs mappings directory: …\user-testing-service \target\stubs\META-INF\com.springhealth\user-testing-service\0.0.1-SNAPSHOT\mappings
> 	[INFO] Creating new stub […\user-testing-service\target\stubs\META-INF\ com.springhealth\user-testing-service\0.0.1-SNAPSHOT\mappings\UserContract.json]
> 	 
> 	Installing …\user-testing-service\target\ user-testing-service-0.0.1-SNAPSHOT-stubs.jar to C:\Users\user\.m2\repository\com\springhealth\user-testing-service \0.0.1-SNAPSHOT\user-testing-service-0.0.1-SNAPSHOT-stubs.jar
> ```
>
> 根据这些日志信息，我们看到打包过程对 UserContract.groovy 契约文件做了处理。打包完成之后，在 target 目录下会生成两个 jar 包，一个是正常的 user-testing-service-0.0.1-SNAPSHOT.jar 文件，另一个就是新的 Stub 文件。Stub 文件的名称为 user-testing-service-0.0.1-SNAPSHOT-stubs.jar，打开该文件会发现两个文件夹，一个是 contracts 文件夹，内部存放着 UserContract.groovy 契约文件，另一个是 mappings 文件夹，内部存放着 UserContract.json 文件，UserContract.json 文件是用 JSON 格式对 UserContract.groovy 契约文件的一种数据转换。
>
> 生成 Stub 文件之后，我们还需要做的事情是通过 install 命令将该 Stub 文件上传到 Maven 仓库，以便消费者通过 pom 中定义的 group-id 和 artifact-id 加载该 jar 包。至此，服务提供者的开发工作告一段落。
>
> #### 服务消费者编写测试用例
>
> 现在回到服务消费者端编写测试用例 InterventionApplicationTests 类，该类展示了典型的 Spring Cloud Contract 端到端测试用例代码风格。其中的 testGetUsers() 测试用例使用 RestTemplate 访问远程 HTTP 端点并验证返回数据的正确性，如下所示：
>
> 复制代码
>
> ```
> import org.junit.Assert;
> import org.junit.Test;
> import org.junit.runner.RunWith;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.boot.test.context.SpringBootTest;
> import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
> import org.springframework.core.ParameterizedTypeReference;
> import org.springframework.http.HttpMethod;
> import org.springframework.http.ResponseEntity;
> import org.springframework.test.context.junit4.SpringRunner;
> import org.springframework.web.client.RestTemplate;
>  
> @RunWith(SpringRunner.class)
> @SpringBootTest(classes = InterventionApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
> @AutoConfigureStubRunner(ids = { "com.springhealth:user-testing-service:+:8080" }, workOffline = true)
> public class InterventionApplicationTests {
>  
>     @Autowired
>     private RestTemplate restTemplate;
>  
>     @Test
>     public void testGetUsers() {
>         ParameterizedTypeReference<UserList> ptf = new ParameterizedTypeReference<UserList>() {
>         };
> 
>         ResponseEntity<UserList> responseEntity = restTemplate.exchange("http://localhost:8080/user/userlist",
>                 HttpMethod.GET, null, ptf);
> 
>         Assert.assertEquals(3, responseEntity.getBody().getData().size());
>     }
> }
> ```
>
> 以上代码中引入了一个新的注解 @AutoConfigureStubRunner，通过该注解来自动注入一个 StubRunner。@AutoConfigureStubRunner 注解包含两个参数，即 ids 和 workOffline。其中最重要的就是需要指定 ids 参数。
>
> ids 参数用于定位存放在 Maven 仓库中的 Stub 包，然后在指定端口启动该 Stub 包中的服务。ids 的格式为 groupId:artifactId:version:classifier:port。这里"com.springhealth: user-testing-service:+:8080"表示去 Maven 仓库定位上一个步骤中上传的 user-testing-service-0.0.1-SNAPSHOT-stubs.jar 包并在 8080 端口中启动服务。
>
> #### 服务消费者执行测试用例
>
> 执行 InterventionApplicationTests，我们会在控制台中看到很多日志输出，其中重要的信息如下所示。可以看到在执行过程中 StubRunner 会下载 Stub 文件，将该 jar 文件解压到临时目录并启动了内嵌的 Tomcat 监听 8080 端口，然后注册相应的 servlet 并最终运行所有的 Stubs，部分核心日志信息如下所示：
>
> 复制代码
>
> ```
> 	o.s.c.c.s.StubDownloaderBuilderProvider  : Will download stubs using Aether
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Remote repos not passed but the switch to work offline was set. Stubs will be used from your local Maven repository.
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Desired version is [+] - will try to resolve the latest version
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Resolved version is [0.0.1-SNAPSHOT]
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Resolving artifact [com.springhealth: user-testing-service:jar:stubs:0.0.1-SNAPSHOT] using remote repositories []
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Resolved artifact [com.springhealth: user-testing-service:jar:stubs:0.0.1-SNAPSHOT] to C:\Users\user\.m2\repository\com\ springhealth\user-testing-service\0.0.1-SNAPSHOT\user-testing-service-0.0.1-SNAPSHOT-stubs.jar
> 	o.s.c.c.stubrunner.AetherStubDownloader  : Unpacking stub from JAR [URI: file:/C:/Users/user/.m2/repository/com/springhealth/user-testing-service/0.0.1-SNAPSHOT/user-testing-service-0.0.1-SNAPSHOT-stubs.jar]
> 	…
> 	s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
> 	o.apache.catalina.core.StandardService   : Starting service [Tomcat]
> 	org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.23
> 	o.a.c.c.C.[Tomcat-1].[localhost].[/]     : Initializing Spring embedded WebApplicationContext
> 	o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 512 ms
> 	o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'stub' to [/]
> 	o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'admin' to [/__admin/*]
> 	s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
> 	o.s.c.contract.stubrunner.StubServer     : Started stub server for project [com.springhealth: user-testing-service:0.0.1-SNAPSHOT:stubs] on port 8080
> 	o.a.c.c.C.[Tomcat-1].[localhost].[/]     : RequestHandlerClass from context returned com.github.tomakehurst.wiremock.http.AdminRequestHandler. Normalized mapped under returned 'null'
> 	o.s.c.c.stubrunner.StubRunnerExecutor    : All stubs are now running RunningStubs [namesAndPorts={com.springhealth: user-testing-service:0.0.1-SNAPSHOT:stubs=8080}]
> ```
>
> 在上面的日志中，我们看到 servlet 生成了 /__admin/* 映射。该端点在测试用例执行完成之后会自动消失，所以可以在 testGetUsers() 方法中打一个断点，然后执行测试用例就可以访问这些自动生成的 HTTP 端点信息。当访问 http://localhost:8080/__admin/ 端点，我们可以获取根据 UserContract 契约生成的用于 Stub 的 JSON 数据，如下所示：
>
> 复制代码
>
> ```
> {
> 	    "mappings":[
> 	        {
> 	            "id":"b134a01f-d983-4a05-8889-b1d5aa2e8781",
> 	            "request":{
> 	                "url":"/userlist/",
> 	                "method":"GET"
> 	            },
> 	            "response":{
>                 "status":200, "body":"{"data":[{"id":1," userCode":"user1",
> "userName":"springhealth_user1"},{"id":2," userCode":"user2","userName":"springhealth_user2"},{"id":3," userCode":"user3","userName":"springhealth_user3"}]}",
> 	                "headers":{
> 	                    "Content-Type":"application/json;charset=UTF-8"
> 	                },
> 	                "transformers":[
> 	                    "response-template"
> 	                ]
> 	            },
> 	            "uuid":"b134a01f-d983-4a05-8889-b1d5aa2e8781"
> 	        },
> 	        {
> 	            "id":"02f3f379-ce66-4136-8b35-7b2fd1aafec9",
> 	            "request":{
> 	                "url":"/user",
> 	                "method":"GET"
> 	            },
> 	            "response":{
> 	                "status":200,
> 	                "body":"OK"
> 	            },
> 	            "uuid":"02f3f379-ce66-4136-8b35-7b2fd1aafec9"
> 	        },
> 	        {
> 	            "id":"d5ae77de-dddd-43b3-b1a3-19145ee5582d",
> 	            "request":{
> 	                "url":"/ping",
> 	                "method":"GET"
> 	            },
> 	            "response":{
> 	                "status":200,
> 	                "body":"OK"
> 	            },
> 	            "uuid":"d5ae77de-dddd-43b3-b1a3-19145ee5582d"
> 	        }
> 	    ],
> 	    "meta":{
> 	        "total":3
> 	    }
> }
> ```
>
> Spring Cloud Contract 在内部集成了 WireMock 工具，该工具通过这些 JSON 数据来模拟定义的接口。同时，在执行测试用例的过程中，我们可以访http://localhost:8080/users/userlist 端点来获取所生成的接口数据，正如我们所预想的一样，返回的数据如下所示：
>
> 复制代码
>
> ```
> {
> 	    "data":[
> 	        {
> 	            "id":1,
> 	            "userCode":"user1",
> 	            "userName":"springhealth_user1"
> 	        },
> 	        {
> 	            "id":2,
> 	            "userCode":"user2",
> 	            "userName":"springhealth_user2"
> 	        },
> 	{
> 	            "id":3,
> 	            "userCode":"user3",
> 	            "userName":"springhealth_user3"
> 	        }
> 	    ]
> }
> ```
>
> 通过整个流程，我们注意到服务提供者是基于消费者的契约来开发接口，而测试用例则是由 Spring Cloud Contract Verifier 根据契约所生成，因此就形成了对契约的一种约束，也就是消费者对服务提供者的约束。如果服务提供者不能满足测试用例则意味着契约已经发生了变化，这正是面向契约的端对端测试的本质所在。
>
> ### 小结与预告
>
> 今天的课程讨论了面向契约的端到端测试，我们基于 Spring Cloud 家族中的 Spring Cloud Contract 框架全面介绍了如何制定服务契约、如何生成 Stub 文件、如何编写和执行测试用例等一系列核心的开发步骤。
>
> 这里给你留一道思考题：如果使用 Spring Cloud Contract 来实现面向契约测试，开发流程上需要实施哪些步骤？
>
> 讲完 Spring Cloud Contract 之后，下一课时是整个课程的最后一讲，我们将对微服务架构和 Spring Cloud 进行总结，并对它的后续发展进行展望。