[300分钟搞懂 Spring Cloud](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=9&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=88)



> 你好，我是你的Spring Cloud讲师尹吉欢，资深Java技术专家一枚。作为一名Spring的忠实粉丝，我一直在使用Spring Boot、Spring Data等一系列框架来进行开发。在Spring Cloud和微服务方面有丰富的实战经验。我喜欢研究技术也喜欢技术分享，除了平时在个人技术站“猿天地”和一些技术大会上的分享外，我还出版了两本Spring Cloud微服务的书《Spring Cloud微服务：入门、实战与进阶》和《Spring Cloud微服务：全栈技术与案例解析》。
>
>  
>
> 但这些形式无论哪种都不方便做代码动态演示，对实操部分的讲解非常不友好。拉勾教育的视频专栏课支持知识点用PPT演示、项目代码录屏讲解，可以帮助学员更快更好地吸收知识并学以致用。这门课，我将从技术原理、工程实践、进阶提升3个维度详细讲解Spring Cloud，带你从零开始搭建一套基于Spring Cloud的微服务架构，进阶为Spring Cloud微服务技术达人。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p7qAeDgUAAIcBnqKoZM519.png)    
>
>  
>
> 在这个“唯快不破”的互联网时代，能否快速发布新功能、支持高并发和大数据，很大程度上决定了一个产品的生死。而传统的单体应用架构很难“快”起来，因为所谓的单体应用程序，简单来说，就是所有的功能全部堆积在一起。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p7uAYP0uAAF17jR2Q88758.png)    
>
>  
>
> 这就决定了它不够灵活、维护成本高、部署效率低、扩展能力差等特点。尽管模块化能解决一部分问题，但要实现快速交付还是很难的。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p7yAWF_TAAEUhL-bzw4676.png)    
>
>  
>
> 随着业务高速发展，微服务架构顺势而出。它将传统的单体应用划分为小型的服务单元，服务之间使用遵循 HTTP 的 API 进行资源访问与操作。每个小型服务可以独立部署，可以采用不同的语言进行开发，可以使用不同的数据库进行数据存储。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p72AE0ucAAI4KxPxI6A335.png)    
>
>  
>
> 以技术社区网站为例，可以看到在单体应用程序架构图中，文章、课程、用户等模块都是在一个项目中。从客户端请求到 Nginx， Nginx 再转发到后端服务，整个网站所有的功能模块都部署在一起。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p72AOfcpAAM9EXO56nQ340.png)    
>
>  
>
> 重构成微服务架构后，在 Nginx 后面不再是具体的业务服务，而是网关，通过网关进行路由调用其他的业务服务。将原先的整个项目按照业务领域拆分成多个微服务，各自独立部署，服务之间通过 HTTP 接口进行远程调用。每个服务都有各自独立的数据库，相互隔离、互不影响。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p76AMovPAAD2g9yJSYc465.png)    
>
>  
>
> 总之，微服务优点诸多。
>
>  
>
> 第一个优点是服务独立，每个服务都有独立的代码仓库、独立的数据库、独立进行部署的能力。
>
>  
>
> 第二点是开发体验好，开发体验好的关键点在于启动速度快，克隆代码速度快，编译部署速度快，技术选型更自由。即使用户服务用Spring Boot 1.X版本，积分服务用Spring Boot 2.X版本，服务之间也无任何影响。
>
>  
>
> 第三点是职责专一性，服务只负责本身的业务，不关心无关业务。这样有利于实现不同的团队维护不同的服务。
>
>  
>
> 第四点是按需扩容，也就是说某个服务特别耗内存，可以单独部署在内存比较大的机器上；如果特别耗 CPU , 那可以部署到 CPU 比较好的机器上。且只需要部署这个服务，不需要像单体应用那样部署整个应用。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p7-Ad5L5AAD9ZFJlRow140.png)    
>
>  
>
> 当然，也不是说微服务就是完美的，在解决问题的同时也引入了新的问题。例如，分布式带来的复杂性，服务拆分后，本地调用变成了远程调用，服务实例有多个，如何负载均衡；被调用的服务出问题的话，如何调用容错；服务之间的依赖关系如何等问题。
>
>  
>
> 再比如，运维的复杂性，拆分后的服务数量多，部署后的服务节点多，需要有日志的统一管理，才方便通过日志排查问题。服务需要有统一监控，才能在发生问题时及时告警。
>
>  
>
> 还有，服务拆分的复杂性，如何拆分出对应的服务很关键，需要结合自身的业务领域进行合理的拆分。拆分后每个服务都有自己的数据库，当一个业务操作涉及多个服务时，如何保证数据一致性，等等。
>
>  
>
> 从2017 年开始，Spring Cloud 在国内的普及度逐渐变高
>
> 2019年8月，Spring Cloud Alibaba 的发布助力Spring Cloud的高速发展
>
>  
>
> 不过这些缺点，并不影响它的发展，这一点从国内外大中型企业的微服务落地率就可以看出。只不过国外大多使用 Spring Cloud，而国内也有不少公司是基于阿里开源的 Dubbo 框架来构建微服务。不过从 2017 年开始，Spring Cloud 在国内的普及度逐渐变高，越来越多的企业也开始使用 Spring Cloud。今年8月，Spring Cloud Alibaba 的发布也助力了Spring Cloud的高速发展。相信在不久的将来，熟练掌握Spring Cloud 将会成为 Java 开发人员面试的门槛。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p7-ASXtPAACXuXr25Wg286.png)    
>
>  
>
> 相比之下，Dubbo更专注于服务治理这块，另一个优势就是性能非常高。而Spring Cloud的优势在于社区活跃，发布新功能的频率高，背靠Spring这个大家族，同时微服务场景需要的各种组件非常齐全，整合起来也非常简单。再加上2019年8月以来阿里巴巴的加持，相信未来几年国内互联网公司的分布式系统开发一定是 Spring Cloud 的天下。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p7-ALBWWAADwbaxz8-4255.png)    
>
>  
>
> 那么，如何才能快速掌握 Spring Cloud 呢？这里画了一张Spring Cloud的知识脑图，主要还是由Spring Cloud中的核心组件构成，包括：
>
> - Eureka 主要用于服务治理；
> - Ribbon 用于负载均衡；
> - Hystrix 用于服务之间远程调用时的容错保护；
> - Feign 可以让我们通过定义接口的方式直接调用其他服务的API；
> - Zuul是API网关，是客户端请求的入口，负责鉴权，路由等功能；
> - Gateway是新推出的基于Spring 5的响应式网关；
> - Config用于统一的配置管理；
> - Sleuth用于请求链路跟踪；
> - Stream用来为微服务应用构建消息驱动能力。
>
> 
>
> 同时介绍了每个组件下的一些核心知识点，可以帮助初学者在学习过程中有明确的方向。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p8CAWgRwAADWKxKipOc493.png)    
>
>  
>
> 以上知识点，我会用300分钟通过11课时一一详解。第1课时将带你重温Spring Boot重点知识，为后面的学习打下扎实的基础。2~5课时对Spring Cloud的几个核心组件进行使用和原理讲解。第6课时选择了携程的Apollo作为配置中心，之所以不讲Spring Cloud Config是因为Spring Cloud Config从使用角度来说，没有Apollo方便，具体会在后续的课程中进行说明。第7~9课时对链路跟踪、安全认证、灰度发布等非常实用的功能进行讲解，这些都是微服务架构下必用的功能点。10课时会对一些比较常见的问题进行讲解，这些也是在面试中经常被问到的知识点。11课时我们会讲解一个综合的案例，将本套课程所学习的组件全部结合起来，形成一个完整的微服务架构。
>
>  
>
> 最后的彩蛋部分会对第二代微服务架构Spring Cloud Alibaba进行简单的介绍，让大家了解Spring Cloud未来的发展方向。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/68/CgoB5l13p8GAVL-DAAPVftyn56U870.png)    
>
>  
>
> 希望通过这门课，帮你深刻理解微服务架构及设计原则，掌握 Eureka、Hystrix、API 网关等Spring Cloud核心组件的原理和使用方法，拥有解决微服务架构常见问题的能力、并能够从零搭建一套基于Spring Cloud的微服务架构。更快更好地完成工作，更轻松地搞定面试。
>
>  
>
> ​     ![img](http://s0.lgstatic.com/i/image2/M01/8A/87/CgotOV13p8KAZStoAAHK12i8krg593.png)    
>
>  
>
> 最后，小小透露一下综合案例的架构图，本课程所学的所有组件都在架构中，大家也可以在实际工作中使用这套架构。
>
>  
>
> 好，这节课就讲到这里，下节课将带你梳理学习Spring Cloud必备的基础知识，记得来听课哦，下节课见。
>
> 
>
> ![img](http://s0.lgstatic.com/i/image2/M01/8B/37/CgoB5l149x6AS9oZAAeLU-Q6uJo953.jpg)