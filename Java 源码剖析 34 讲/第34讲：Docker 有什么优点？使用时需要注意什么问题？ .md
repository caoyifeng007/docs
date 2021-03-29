[Java 面试真题及源码 34 讲 - 前 360 技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=59&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=1794)



> Docker 从 2013 年发展到现在，它的普及率已经可以和最常用的 MySQL 和 Redis 并驾齐驱了，从最初偶尔出现在面试中，到现在几乎成为面试中必问的问题之一。如果再不了解 Docker 相关的知识点，可能就会与自己心仪的职位擦肩而过。所以本课时将会带领你对 Docker 相关的知识做一个全面的认识。
>
> 我们本课时的面试题是，Docker  是什么？它有什么优点？
>
> ### 典型回答
>
> Docker 是一个开源（开放源代码）的应用容器引擎，可以方便地对容器进行管理。可通过 Docker 打包各种环境应用配置，比如安装 JDK 环境、发布自己的 Java 程序等，然后再把它发布到任意 Linux 机器上。
>
> Docker 中有三个重要的概念，具体如下。
>
> - **镜像（Image）**：一个特殊的文件操作系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的配置参数（如匿名卷、环境变量、用户等）， 镜像不包含任何动态数据，其内容在构建之后也不会被改变。
> - **容器（Container）**：它是用来运行镜像的。例如，我们拉取了一个 MySQL 镜像之后，只有通过创建并启动 MySQL 容器才能正常的运行 MySQL，容器可以进行创建、启动、停止、删除、暂停等操作。
> - **仓库（Repository）**：用来存放镜像文件的地方，我们可以把自己制作的镜像上传到仓库中，Docker 官方维护了一个公共仓库 Docker Hub，[你也可以点击这里查询并下载所有的公共镜像](https://hub.docker.com)。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/2B/9F/CgqCHl7-xpKAFgHdAAEShADSk60188.png)
>
> 在 Docker 出现之前，我们如果要发布自己的 Java 程序，就需要在服务器上安装 JDK（或者 JRE）、Tomcat 容器，然后配置 Tomcat 参数，对 JVM 参数进行调优等操作。然而如果要在多台服务器上运行 Java 程序，则需要将同样繁杂的步骤在每台服务器都重复执行一遍，这样显然比较耗时且笨拙的。
>
> 后来有了虚拟机的技术，我们就可以将配置环境打包到一个虚拟机镜像中，然后在需要的服务器上装载这些虚拟机，从而实现了运行环境的复制，但虚拟机会占用很多的系统资源，比如内存资源和硬盘资源等，并且虚拟机的运行需要加载整个操作系统，这样就会浪费掉好几百兆的内存资源，最重要的是因为它需要加载整个操作系统所以它的运行速度就很慢，并且还包含了一些我们用不到的冗余功能。
>
> 因为虚拟机的这些缺点，所以在后来就有了 Linux 容器（Linux Containers，LXC），它是一种进程级别的 Linux 容器，用它可以模拟一个完整的操作系统。相比于虚拟机来说，Linux 容器所占用的系统资源更少，启动速度也更快，因为它本质上是一个进程而非真实的操作系统，因此它的启动速度就比较快。
>
> 而 Docker 则是对 Linux 容器的一种封装，并提供了更加方便地使用接口，所以 Docker 一经推出就迅速流行起来。Docker 和虚拟机（VM）区别如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/2B/9F/CgqCHl7-xqKAdySsAARl-ihH0Cc421.png)
>
> Docker 具备以下 6 个优点。
>
> - **轻量级**：Docker 容器主要利用并共享主机内核，它并不是完整的操作系统，因此它更加轻量化。
> - **灵活**：它可以将复杂的应用程序容器化，因此它非常灵活和方便。
> - **可移植**：可以在本地构建 Docker 容器，并把它部署到云服务器或任何地方进行使用。
> - **相互隔离，方便升级**：容器是高度自给自足并相互隔离的容器，这样就可以在不影响其他容器的情况下更换或升级你的 Docker 容器了。
> - **可扩展**：可以在数据中心内增加并自动分发容器副本。
> - **安全**：Docker 容器可以很好地约束和隔离应用程序，并且无须用户做任何配置。
>
> ### 考点分析
>
> 通过此面试题可以考察出面试者是否真的使用或了解过 Docker 技术，然而对于面试官来说，最关注的是面试者是否了解 Docker 和 Java 程序结合时会出现的一些问题，因此这部分的内容需要读者特别留意一下。
>
> 和此知识点相关的面试题还有以下这些：
>
> - Docker 的常用命令有哪些？
> - 在 Docker 中运行 Java 程序可能会存在什么问题？
>
> ### 知识扩展
>
> #### Docker 常用命令
>
> 我们在安装了 Docker Disktop（客户端）就可以用 docker --version 命令来查看 Docker 的版本号，使用示例如下：
>
> 复制代码
>
> ```
> $ docker --version
> Docker version 19.03.8, build afacb8b
> ```
>
> 然后可以到 Docker Hub 上查找我们需要的镜像，比如 Redis 镜像，如下图所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/2B/9F/CgqCHl7-xsCAC__xAATWzSiVbKU657.png)
>
> 接着我们选择并点击最多人下载的镜像，如下图所示：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/2B/93/Ciqc1F7-xseAVVs3AANxQpDC9RM481.png)
>
> 从描述中找到我们需要装载 Redis 的版本，然后使用 docker pull redis@ 版本号来拉取相关的镜像，或者使用 docker pull redis 直接拉取最新（版本）的 Redis 镜像，如下所示：
>
> 复制代码
>
> ```
> $ docker pull redis
> Using default tag: latest
> latest: Pulling from library/redis
> Digest: sha256:800f2587bf3376cb01e6307afe599ddce9439deafbd4fb8562829da96085c9c5
> Status: Image is up to date for redis:latest
> docker.io/library/redis:latest
> ```
>
> 紧接着就可以使用 docker images 命令来查看所有下载的镜像，如下所示：
>
> 复制代码
>
> ```
> $ docker images
> REPOSITORY TAG IMAGE ID CREATED SIZE
> redis latest 235592615444 13 days ago 104MB
> ```
>
> 有了镜像之后我们就可以使用 docker run 来**创建并运行容器**了，使用命令如下：
>
> 复制代码
>
> ```
> $ docker run --name myredis -d redis
> 22f560251e68b5afb5b7b52e202dcb3d47327f2136700d5a17bca7e37fc486bf
> ```
>
> 查看运行中的容器，命令如下：
>
> 复制代码
>
> ```
> ￥ docker ps
> CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
> 22f560251e68 redis "docker-entrypoint.s…" About a minute ago Up About a minute 6379/tcp myredis
> ```
>
> 其中，“myredis”为容器的名称，“6379/tcp”为 Redis 的端口号，容器的 ID 为“22f560251e68”。
>  最后我们使用如下命令来连接 Redis：
>
> 复制代码
>
> ```
> $ docker  exec -it myredis  redis-cli
> 127.0.0.1:6379> 
> ```
>
> 其他常用命令如下：
>
> - 容器停止：docker stop 容器名称
> - 启动容器：docker start 容器名称
> - 删除容器：docker rm 容器名称
> - 删除镜像：docker rmi 镜像名称
> - 查看运行的所有容器：docker ps
> - 查看所有容器：docker ps -a
> - 容器复制文件到物理机：docker cp 容器名称:容器目录 物理机目录
> - 物理机复制文件到容器：docker cp 物理机目录 容器名称:容器目录
>
> #### Docker 可能存在的问题
>
> Java 相对于 Docker 来说显然具有更悠久的历史，因此在早期的 Java 版本中（JDK 8u131）因为不能很好地识别 Docker 相关的配置信息，从而导致可能会出现 Java 程序意外被终止的情况或者是过度创建线程数而导致并发性能下降的问题。
>
> Java 程序意外终止的主要原因是因为，在 Docker 中运行的 Java 程序因为没有明确指定 JVM 堆和直接内存等参数，而 Java 程序也不能很好地识别 Docker 的相关容量配置，导致 Java 程序试图获取了超过 Docker 本身的容量，而被 Docker 容器强制结束进程的情况（这是 Docker 自身的防御保护机制）。
>
> 过度创建线程是因为早期的 Java 版本并不能很好地识别 Docker 容器的 CPU 资源，因此会错误地识别和创建过多的线程数。比如 ParallelStreams 和 ForkJoinPool 等，它们默认就是根据当前系统的 CPU 核心数来创建对应的线程数的，但因为在 Docker 中的 Java 程序并不能很好地识别 CPU 核心数，就会导致创建的线程数量大于 CPU 的核心数量，从而导致并发效率降低的情况。
>
> ParallelStreams 的基本用法如下：
>
> 复制代码
>
> ```
> List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);
> numbers.parallelStream().forEach(count -> {
>     System.out.println("val:" + count);
> });
> ```
>
> ParallelStreams 是将任务提交给 ForkJoinPool 来实现的，ForkJoinPool 获取本地 CPU 核心数的源码如下：
>
> 复制代码
>
> ```
> private static ForkJoinPool makeCommonPool() {
>     // 忽略其他代码
>     if (parallelism < 0 && // default 1 less than #cores
>         (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
>         parallelism = 1;
>     if (parallelism > MAX_CAP)
>         parallelism = MAX_CAP;
>     return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
>                             "ForkJoinPool.commonPool-worker-");
> }
> ```
>
> 其中，“Runtime.getRuntime().availableProcessors()”是用来获取本地线程数量的。
>
> 要解决以上这些问题的方法，最简单的解决方案就是升级 Java 版本，比如 Java 10 就可以很好地识别 Docker 容器的这些限制。但如果使用的是老版本的 Java，那么需要在启动 JVM 的时候合理的配置堆、元数据区等内存区域大小，并指定 ForkJoinPool 的最大线程数，如下所示：
>
> 复制代码
>
> ```
> -Djava.util.concurrent.ForkJoinPool.common.parallelism=8
> ```
>
> ### 小结
>
> 本课时我们介绍了 Docker 的概念以及 Docker 中最重要的三个组件：镜像、容器和仓库，并且介绍了 Docker 的 6 大特点：轻量级、灵活、可移植、相互隔离、可扩展和安全等特点；同时还介绍了 Docker 的常见使用命令；最后介绍了 Docker 可能在老版本（JDK 8u131 之前）的 Java 中可能会存在意外停止和线程创建过多的问题以及解决方案。
>
> 通过以上内容的学习相信你对 Docker 已经有了一个系统的认识，需要特别注意是 Docker 在 Java 老版本中可能出现的问题以及解决方案，这一点在面试中经常会被问到。
>
> OK，这一课时就讲到这里啦，恭喜你已经学习完了关于本系列的所有课程。如果你觉得课程不错，从中有所收获的话，不要忘了推荐给身边的朋友哦，最后希望大家都有所提高、不断成长，谢谢~