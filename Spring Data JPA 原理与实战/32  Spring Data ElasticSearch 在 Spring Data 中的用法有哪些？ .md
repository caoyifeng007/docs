[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4731)



> 这一讲是这门专栏的最后一讲了，恭喜你一直坚持到现在。
>
> 相信到这里，你已经对 Spring Data JPA 有一定的认识了，那么这一讲我会为你演示 Spring Data ElasticSearch 如何使用，帮助你打开思路，感受 Spring Data 的抽象封装。
>
> 我们还是从一个案例入手。
>
> ### Spring Data ElasticSearch 入门案例
>
> Spring Data 和 Elasticsearch 结合的时候，唯一需要注意的是版本之间的兼容性问题，Elasticsearch 和 Spring Boot 是同时向前发展的，而 Elasticsearch 的大版本之间还存在一定的 API 兼容性问题，所以我们必须要知道这些版本之间的关系，我整理了一个表格，如下。
>
> | **Spring Data Release Train**                                | **Spring Data Elasticsearch**                                | **Elasticsearch** | **Spring Boot**                                              |
> | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------- | ------------------------------------------------------------ |
> | 2020.0.0[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] | 4.1.x[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] | 7.9.3             | 2.4.x[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] |
> | Neumann                                                      | 4.0.x                                                        | 7.6.2             | 2.3.x                                                        |
> | Moore                                                        | 3.2.x                                                        | 6.8.12            | 2.2.x                                                        |
> | Lovelace                                                     | 3.1.x                                                        | 6.2.2             | 2.1.x                                                        |
> | Kay[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] | 3.0.x[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] | 5.5.0             | 2.0.x[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] |
> | Ingalls[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] | 2.1.x[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] | 2.4.0             | 1.5.x[[2](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_2)] |
>
> 现在你对这些版本之间的关联关系有了一定印象，由于版本越新越便利，所以一般情况下我们直接采用最新的版本。
>
> 接下来看看这个版本是怎么完成 Demo 演示的。
>
> **第一步：利用 Helm Chart 安装一个 Elasticsearch 集群 7.9.3 版本**，执行命令如下。
>
> 复制代码
>
> ```
> 1. helm2 repo add elastic https://helm.elastic.co
> 2. helm2 install --name myelasticsearch elastic/elasticsearch  --set imageTag=7.9.3
> ```
>
> 安装完之后，我们就可以看到如下信息。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/04/56/CgpVE1_tdLyAY0UfAAH31rOGV0o472.png)
>
> 这代表我们安装成功。
>
> 由于 ElasticSearch 是发展变化的，所以它的安装方式你可以参考官方文档：https://github.com/elastic/helm-charts/tree/master/elasticsearch
>
> 然后我们利用 k8s 集群端口映射到本地，就可以开始测试了。
>
> 复制代码
>
> ```
> ~ ❯❯❯ kubectl port-forward svc/elasticsearch-master 9200:9200 -n my-namespace
> Forwarding from 127.0.0.1:9200 -> 9200
> Forwarding from [::1]:9200 -> 9200
> ```
>
> **第二步：在 gradle.build 里面配置 Spring Data ElasticSearch 依赖的 Jar 包**。
>
> 我们依赖 Spring Boot 2.4.1 版本，完整的 gradle.build 文件如下所示。
>
> 复制代码
>
> ```
> plugins {
>    id 'org.springframework.boot' version '2.4.1'
>    id 'io.spring.dependency-management' version '1.0.10.RELEASE'
>    id 'java'
> }
> group = 'com.example.data.es'
> version = '0.0.1-SNAPSHOT'
> sourceCompatibility = '1.8'
> configurations {
>    compileOnly {
>       extendsFrom annotationProcessor
>    }
> }
> repositories {
>    mavenCentral()
> }
> dependencies {
>    implementation 'org.springframework.boot:spring-boot-starter-actuator'
>    implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
>    implementation 'org.springframework.boot:spring-boot-starter-web'
>    compileOnly 'org.projectlombok:lombok'
>    developmentOnly 'org.springframework.boot:spring-boot-devtools'
>    runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
>    annotationProcessor 'org.projectlombok:lombok'
>    testImplementation 'org.springframework.boot:spring-boot-starter-test'
> }
> test {
>    useJUnitPlatform()
> }
> ```
>
> **第三步：新建一个目录，结构如下图所示，方便我们测试**。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/04/55/Cip5yF_tdMaAbP03AAD7ix9soGU430.png)
>
> **第四步：在 application.properties 里面新增 es 的连接地址，连接本地的 Elasticsearch**。
>
> 复制代码
>
> ```
> spring.data.elasticsearch.client.reactive.endpoints=127.0.0.1:9200
> ```
>
> **第五步：新增一个 ElasticSearchConfiguration 的配置文件，主要是为了开启扫描的包**。
>
> 复制代码
>
> ```
> package com.example.data.es.demo.es;
> import org.springframework.context.annotation.Configuration;
> import org.springframework.data.elasticsearch.repository.config.EnableElasticsearchRepositories;
> //利用@EnableElasticsearchRepositories注解指定Elasticsearch相关的Repository的包路径在哪里
> @EnableElasticsearchRepositories(basePackages = "com.example.data.es.demo.es")
> @Configuration
> public class ElasticSearchConfiguration {
> }
> ```
>
> **第六步：我们新增一个 Topic 的 Document，它类似 JPA 里面的实体，用来保存和读取 Topic 的数据**，代码如下所示。
>
> 复制代码
>
> ```
> package com.example.data.es.demo.es;
> import lombok.Builder;
> import lombok.Data;
> import lombok.ToString;
> import org.springframework.data.annotation.Id;
> import org.springframework.data.elasticsearch.annotations.Document;
> import org.springframework.data.elasticsearch.annotations.Field;
> import org.springframework.data.elasticsearch.annotations.FieldType;
> import java.util.List;
> @Data
> @Builder
> @Document(indexName = "topic")
> @ToString(callSuper = true)
> //论坛主题信息
> public class Topic {
>     @Id
>     private Long id;
>     private String title;
>     @Field(type = FieldType.Nested, includeInParent = true)
>     private List<Author> authors;
> }
> package com.example.data.es.demo.es;
> import lombok.Builder;
> import lombok.Data;
> @Data
> @Builder
> //作者信息
> public class Author {
>     private String name;
> }
> ```
>
> **第七步：新建一个 Elasticsearch 的 Repository，用来对 Elasticsearch 索引的增删改查**，代码如下所示。
>
> 复制代码
>
> ```
> package com.example.data.es.demo.es;
> import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
> import java.util.List;
> //类似JPA一样直接操作Topic类型的索引
> public interface TopicRepository extends ElasticsearchRepository<Topic,Long> {
>     List<Topic> findByTitle(String title);
> }
> ```
>
> **第八步: 新建一个 Controller，对 Topic 索引进行查询和添加。**
>
> 复制代码
>
> ```
> @RestController
> public class TopicController {
>     @Autowired
>     private TopicRepository topicRepository;
>     //查询topic的所有索引
>     @GetMapping("topics")
>     public List<Topic> query(@Param("title") String title) {
>         return topicRepository.findByTitle(title);
>     }
>     //保存 topic索引
>     @PostMapping("topics")
>     public Topic create(@RequestBody Topic topic) {
>         return topicRepository.save(topic);
>     }
> }
> ```
>
> **第九步：发送一个添加和查询的请求测试一下**。
>
> 我们发送三个 POST 请求，添加三条索引，代码如下所示。
>
> 复制代码
>
> ```
> POST /topics HTTP/1.1
> Host: 127.0.0.1:8080
> Content-Type: application/json
> Cache-Control: no-cache
> Postman-Token: d9cc1f6c-24dd-17ff-f2e8-3063fa6b86fc
> {
>     "title":"jack",
>     "id":2,
>     "authors":[{
>         "name":"jk1"
>         },{
>         "name":"jk2"
>         }]
> }
> ```
>
> 然后发送一个 get 请求，获得标题是 jack 的索引，如下面这行代码所示。
>
> 复制代码
>
> ```
> GET http://127.0.0.1:8080/topics?title=jack
> ```
>
> 得到如下结果。
>
> 复制代码
>
> ```
> GET http://127.0.0.1:8080/topics?title=jack
> HTTP/1.1 200 
> Content-Type: application/json
> Transfer-Encoding: chunked
> Date: Wed, 30 Dec 2020 15:12:16 GMT
> Keep-Alive: timeout=60
> Connection: keep-alive
> [
>   {
>     "id": 1,
>     "title": "jack",
>     "authors": [
>       {
>         "name": "jk1"
>       },
>       {
>         "name": "jk2"
>       }
>     ]
>   },
>   {
>     "id": 3,
>     "title": "jack",
>     "authors": [
>       {
>         "name": "jk1"
>       },
>       {
>         "name": "jk2"
>       }
>     ]
>   },
>   {
>     "id": 2,
>     "title": "jack",
>     "authors": [
>       {
>         "name": "jk1"
>       },
>       {
>         "name": "jk2"
>       }
>     ]
>   }
> ]
> Response code: 200; Time: 348ms; Content length: 199 bytes
> Cannot preserve cookies, cookie storage file is included in ignored list:
> > /Users/jack/Company/git_hub/spring-data-jpa-guide/2.3/elasticsearch-data/.idea/httpRequests/http-client.cookies
> ```
>
> 这时，一个完整的 Spring Data Elasticsearch 的例子就演示完了。其实你会发现，我们使用 Spring Data Elasticsearch 来操作 ES 相关的 API 的话，比我们直接写 Http 的 client 要简单很多，因为这里面帮我们封装了很多基础逻辑，省去了很多重复造轮子的过程。
>
> 其实测试用例也是很简单的，我们接着来看一下写法。
>
> **第十步：Elasticsearch Repository 的测试用例写法**，如下面的代码和注释所示。
>
> 复制代码
>
> ```
> package com.example.data.es.demo;
> import com.example.data.es.demo.es.Author;
> import com.example.data.es.demo.es.Topic;
> import com.example.data.es.demo.es.TopicRepository;
> import org.assertj.core.util.Lists;
> import org.junit.jupiter.api.BeforeEach;
> import org.junit.jupiter.api.Test;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.boot.test.context.SpringBootTest;
> import org.springframework.test.context.TestPropertySource;
> import java.util.List;
> @SpringBootTest
> @TestPropertySource(properties = {"logging.level.org.springframework.data.elasticsearch.core=TRACE", "logging.level.org.springframework.data.elasticsearch.client=trace", "logging.level.org.elasticsearch.client=TRACE", "logging.level.org.apache.http=TRACE"})//新增一些配置， 开启spring data elastic search的http的调用过程，我们可以查看一下日志
> public class ElasticSearchRepositoryTest {
>     @Autowired
>     private TopicRepository topicRepository;
>     @BeforeEach
>     public void init() {
> //        topicRepository.deleteAll(); //可以直接删除所有索引
>         Topic topic = Topic.builder().id(11L).title("jacktest").authors(Lists.newArrayList(Author.builder().name("jk1").build())).build();
>         topicRepository.save(topic);//集成测试保存索引
>         Topic topic1 = Topic.builder().id(14L).title("jacktest").authors(Lists.newArrayList(Author.builder().name("jk1").build())).build();
>         topicRepository.save(topic1);
>         Topic topic2 = Topic.builder().id(15L).title("jacktest").authors(Lists.newArrayList(Author.builder().name("jk1").build())).build();
>         topicRepository.save(topic2);//保存索引
>     }
>     @Test
>     public void testTopic() {
>         Iterable<Topic> topics = topicRepository.findAll();
>         topics.forEach(topic1 -> {
>             System.out.println(topic1);
>         });
>         List<Topic> topicList = topicRepository.findByTitle("jacktest");
>         topicList.forEach(t -> {
>             System.out.println(t);//获得索引的查询结果
>         });
>         List<Topic> topicList2 = topicRepository.findByTitle("xxx");
>         topicList2.forEach(t -> {
>             System.out.println(t);//我们也可以用上一讲介绍的断言测试
>         });
>     }
> }
> ```
>
> 接着我们看一下测试用例的调用日志，从日志可以看出，调用的时候发生的 Http 的 PUT 请求，是用来创建和修改一个索引的文档的。请看下面的图片。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image2/M01/04/56/CgpVE1_tdNiAXq0WAAPx9WYUcvE585.png)
>
> 从中也可以看得出来，转化成 es 的 api 查询语法之后，发送的 post 请求又变成下图显示的样子。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/04/55/Cip5yF_tdN6AQ3l9AAPPn8brHa8263.png)
>
> 日志比较长，你有兴趣的话，可以按照我的 DEMO 和开启日志的方法，自己去分析体会一下。
>
> 下面来说说 Spring Data ElasticSearch 中关键的几个类。
>
> ### Spring Data ElasticSearch 关键的类
>
> 通过上面的案例我们可以知道，Spring Data ElasticSearch 的用法其实非常简单，并且我们通过日志也可以看到，底层实现是基于 http 请求，来操作 Elasticsearch 的 server 中的 api 进行的。
>
> 那么我们简单看一下这一框架还给我们提供了哪些 ElasticSearch 的操作方法。和分析 Spring Data JPA 一样，看一下 Repository 的所有子类，如下图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image2/M01/04/55/Cip5yF_tdOWAN1p8AAKW4zuYBgc483.png)
>
> 从图中可以看得出来，ElasticsearchRepository 是默认的 Repository 的实现类，我们如果继续往下面看源码的话，就可以看到里面进行了很多 ES 的 Http Client 操作。
>
> 同时再看一下 Structure 视图，如下所示。
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image2/M01/04/57/CgpVE1_tdOyAa_qxAARM3eWQpnQ793.png)
>
> 从这张图可以知道，ElasticsearchRepository 默认给我们提供了 search 和 index 相关的一些操作方法，并且 Spring Data Common 里面的一些公共方法同样适用，这和我们刚才演示的 Defining Method Query 的 JPA 语法同样适用，可以大大减轻操作 ES 的难度，提高了开发的效率，甚至像我们没有演示到的分页、排序、limit 等同样适用。
>
> 所以你现在学到了一个“套路”：和 Spring Data JPA 用相同的思路，就可以很快掌握 Spring Data Elasticsearch 的基本用法，及其大概的实现思路。
>
> 那么很多时候同一个工程里面既有 JPA 又有 Elasticsearch，又该怎么写呢？
>
> ### ESRepository 和 JPARepository 同时存在
>
> 这个时候应该怎么区分不同的 Repository 用什么呢？
>
> 我们假设刚才测试的样例里面，同时有关于 User 信息的 DB 操作，那么看一下我们的项目应该怎么写。
>
> **第一步：我们将对 Elasticsearch 的实体、Repository 和对 JPA 操作的实体、Repository 放到不同的文件里面**，如下图所示。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image2/M01/04/55/Cip5yF_tdPOAVRMHAACTufgK21A436.png)
>
> **第二步：新增 JpaConfiguration，用来指定 Jpa 相关的 Repository 目录**，完整代码如下。
>
> 复制代码
>
> ```
> package com.example.data.es.demo.jpa;
> import org.springframework.context.annotation.Configuration;
> import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
> //利用@EnableJpaRepositories指定JPA的目录是"com.example.data.es.demo.jpa"
> @EnableJpaRepositories(basePackages = "com.example.data.es.demo.jpa")
> @Configuration
> public class JpaConfiguration {
> }
> ```
>
> **第三步：新增 User 实体，用来操作用户基本信息**。
>
> 复制代码
>
> ```
> @Data
> @Builder
> @Entity
> @Table
> @NoArgsConstructor
> @AllArgsConstructor
> @ToString
> public class User {
>     @Id
>     @GeneratedValue(strategy= GenerationType.AUTO)
>     private Long id;
>     private String name;
>     private String email;
> }
> ```
>
> **第四步：新增 UserRepository，用来进行 DB 操作**。
>
> 复制代码
>
> ```
> package com.example.data.es.demo.jpa;
> import org.springframework.data.jpa.repository.JpaRepository;
> //对User的DB操作，我们直接继承JpaRepository
> public interface UserRepository extends JpaRepository<User,Long> {
> }
> ```
>
> **第五步：写测试用例进行测试**。
>
> 复制代码
>
> ```
> package com.example.data.es.demo;
> import com.example.data.es.demo.jpa.User;
> import com.example.data.es.demo.jpa.UserRepository;
> import org.junit.jupiter.api.Test;
> import org.springframework.beans.factory.annotation.Autowired;
> import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
> import java.util.List;
> //利用@DataJpaTest完成集成测试
> @DataJpaTest
> public class UserRepositoryTest {
>     @Autowired
>     private UserRepository userRepository;
>     @Test
>     public void testJpa() {
> //往数据库里面保存一条数据，并且打印一下
>                 userRepository.save(User.builder().id(1L).name("jkdb").email("jack@email.com").build());
>         List<User> users = userRepository.findAll();
>         users.forEach(user -> {
>             System.out.println(user);
>         });
>     }
> }
> ```
>
> 这个时候，我们的测试用例就变成了如下图所示的结构。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image2/M01/04/57/CgpVE1_tdQKAAa7zAABNF77hZ_A879.png)
>
> 那么现在我们知道了，JPA 和 Elasticsearch 同时存在，和启动项目是一样的效果，这里就不写 Controller 了。
>
> 我们再整体运行一下这三个测试用例，进行完整的测试，就可以看到如下结果。
>
> 1.ElasticSearchRepositoryTest 执行的时候，通过日志可以看到这是对 ES 进行的操作，如下图所示。
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/8C/74/Ciqc1F_tdRWABN3HAASErifQeiw553.png)
>
> 2.UserRepositoryTest 执行的时候，通过日志我们可以看出来这是对 DB 进行的操作，所以谁也不影响谁，如下图所示。
>
> ![Drawing 9.png](https://s0.lgstatic.com/i/image/M00/8C/74/Ciqc1F_tdRyAe_oeAAMw4yV6H4o471.png)
>
> 通过上面的例子我们可以知道，Spring Data 对 JPA 等 SQL 型的传统数据库的支持是非常好的，同时对 NoSQL 型的非关系类数据库的支持也比较友好，大大降低了操作不同数据源的难度，可以有效提升我们的开发效率。
>
> ### 总结
>
> 这一讲内容到这里就结束了，我通过“入门型”的 Spring Data Elasticsearch 样例展示，让你体会了 Spring Data 对数据操作的抽象封装的强大之处。
>
> 如果你研究好了这部分内容，其实 Spring Data 中的其他系列也是可以通用的。这里我只是期望起到抛砖引玉的效果，希望你能更好地掌握 Spring Data 的精髓，并且能深入理解 JPA。
>
> 至此，我们的专栏也将告一段落，不知道这 32 讲的内容对你是否有帮助，我希望你可以回过头好好回顾，更好地掌握 Spring Data JPA。
>
> 如果对此有不懂的地方，也欢迎你在评论区留言，我们一起探讨，一起在这条路上不断精进。再见！
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa