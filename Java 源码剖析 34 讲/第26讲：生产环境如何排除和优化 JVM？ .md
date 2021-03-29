[Java 面试真题及源码 34 讲 - 前 360 技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=59&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=1786)



> 通过前面几个课时的学习，相信你对 JVM 的理论及实践等相关知识有了一个大体的印象。而本课时将重点讲解 JVM 的排查与优化，这样就会对 JVM 的知识点有一个完整的认识，从而可以更好地应用于实际工作或者面试了。
>
> 我们本课时的面试题是，生产环境如何排查问题？
>
> ### 典型回答
>
> 如果是在生产环境中直接排查 JVM 的话，最简单的做法就是使用 JDK 自带的 6 个非常实用的命令行工具来排查。它们分别是：jps、jstat、jinfo、jmap、jhat 和 jstack，它们都位于 JDK 的 bin 目录下，可以使用命令行工具直接运行，其目录如下图所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/19/15/CgqCHl7Z6XeAXH4mAAQG2yKoYrQ797.png)
>
> 接下来我们来看看这些工具的具体使用。
>
> #### 1. jps（虚拟机进程状况工具）
>
> jps（JVM Process Status tool，虚拟机进程状况工具）它的功能和 Linux 中的 ps 命令比较类似，用于列出正在运行的 JVM 的 LVMID（Local Virtual Machine IDentifier，本地虚拟机唯一 ID），以及 JVM 的执行主类、JVM 启动参数等信息。语法如下：
>
> 复制代码
>
> ```java
> jps [options] [hostid]
> ```
>
> 常用的 options 选项：
>
> - -l：用于输出运行主类的全名，如果是 jar 包，则输出 jar 包的路径；
> - -q：用于输出 LVMID（Local Virtual Machine Identifier，虚拟机唯一 ID）；
> - -m：用于输出虚拟机启动时传递给主类 main() 方法的参数；
> - -v：用于输出启动时的 JVM 参数。
>
> 使用实例：
>
> 复制代码
>
> ```powershell
> ➜  jps -l
> 68848
> 40085 org.jetbrains.jps.cmdline.Launcher
> 40086 com.example.optimize.NativeOptimize
> 40109 jdk.jcmd/sun.tools.jps.Jps
> 68879 org.jetbrains.idea.maven.server.RemoteMavenServer36
> ➜  jps -q
> 40368
> 68848
> 40085
> 40086
> 68879
> ➜  jps -m
> 40400 Jps -m
> 68848
> 40085 Launcher /Applications/IntelliJ IDEA2.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/oro-2.0.8.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/resources_en.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/maven-model-3.6.1.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/qdox-2.0-M10.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/plexus-component-annotations-1.7.1.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/httpcore-4.4.13.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/maven-resolver-api-1.3.3.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/netty-common-4.1.47.Final.jar:/Applications/IntelliJ IDEA2.app/Contents/plugins/java/lib/maven-resolver-connector-basic-1.3.3.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/maven-artifact-3.6.1.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/plexus-utils-3.2.0.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/netty-resolver-4.1.47.Final.jar:/Applications/IntelliJ IDEA2.app/Contents/lib/guava-28.2-
> 40086 NativeOptimize
> 68879 RemoteMavenServer36
> ➜  jps -v
> 68848  -Xms128m -Xmx2048m -XX:ReservedCodeCacheSize=240m -XX:+UseCompressedOops -Dfile.encoding=UTF-8 -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -XX:CICompilerCount=2 -Dsun.io.useCanonPrefixCache=false -Djava.net.preferIPv4Stack=true -Djdk.http.auth.tunneling.disabledSchemes="" -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Djdk.attach.allowAttachSelf -Dkotlinx.coroutines.debug=off -Djdk.module.illegalAccess.silent=true -Xverify:none -XX:ErrorFile=/Users/admin/java_error_in_idea_%p.log -XX:HeapDumpPath=/Users/admin/java_error_in_idea.hprof -javaagent:/Users/admin/.jetbrains/jetbrains-agent-v3.2.0.de72.619 -Djb.vmOptionsFile=/Users/admin/Library/Application Support/JetBrains/IntelliJIdea2020.1/idea.vmoptions -Didea.paths.selector=IntelliJIdea2020.1 -Didea.executable=idea -Didea.home.path=/Applications/IntelliJ IDEA2.app/Contents -Didea.vendor.name=JetBrains
> 40085 Launcher -Xmx700m -Djava.awt.headless=true -Djava.endorsed.dirs="" -Djdt.compiler.useSingleThread=true -Dpreload.project.path=/Users/admin/github/blog-example/blog-example -Dpreload.config.path=/Users/admin/Library/Application Support/JetBrains/IntelliJIdea2020.1/options -Dcompile.parallel=false -Drebuild.on.dependency.change=true -Djava.net.preferIPv4Stack=true -Dio.netty.initialSeedUniquifier=1366842080359982660 -Dfile.encoding=UTF-8 -Duser.language=zh -Duser.country=CN -Didea.paths.selector=IntelliJIdea2020.1 -Didea.home.path=/Applications/IntelliJ IDEA2.app/Contents -Didea.config.path=/Users/admin/Library/Application Support/JetBrains/IntelliJIdea2020.1 -Didea.plugins.path=/Users/admin/Library/Application Support/JetBrains/IntelliJIdea2020.1/plugins -Djps.log.dir=/Users/admin/Library/Logs/JetBrains/IntelliJIdea2020.1/build-log -Djps.fallback.jdk.home=/Applications/IntelliJ IDEA2.app/Contents/jbr/Contents/Home -Djps.fallback.jdk.version=11.0.6 -Dio.netty.noUnsafe=true -Djava.io.tmpdir=/Users/admin/Library/Caches/Je
> 40086 NativeOptimize -Dfile.encoding=UTF-8
> 40425 Jps -Dapplication.home=/Users/admin/Library/Java/JavaVirtualMachines/openjdk-14/Contents/Home -Xms8m -Djdk.module.main=jdk.jcmd
> 68879 RemoteMavenServer36 -Djava.awt.headless=true -Dmaven.defaultProjectBuilder.disableGlobalModelCache=true -Xmx768m -Didea.maven.embedder.version=3.6.1 -Dmaven.ext.class.path=/Applications/IntelliJ IDEA2.app/Contents/plugins/maven/lib/maven-event-listener.jar -Dfile.encoding=UTF-8
> ```
>
> #### 2. jstat（虚拟机统计信息监视工具）
>
> jstat（JVM Statistics Monitoring Tool，虚拟机统计信息监视工具）用于监控虚拟机的运行状态信息。
>
> 例如，我们用它来查询某个 Java 进程的垃圾收集情况，示例如下：
>
> 复制代码
>
> ```java
> ➜  jstat -gc 43704
>  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT
> 10752.0 10752.0  0.0    0.0   65536.0   5243.4   175104.0     0.0     4480.0 774.0  384.0   75.8       0    0.000   0      0.000   -          -    0.000
> ```
>
> 参数说明如下表所示：
>
> | **参数** | **说明**                                          |
> | -------- | ------------------------------------------------- |
> | S0C      | 年轻代中第一个存活区的大小                        |
> | S1C      | 年轻代中第二个存活区的大小                        |
> | S0U      | 年轻代中第一个存活区已使用的空间（字节）          |
> | S1U      | 年轻代中第二个存活区已使用的空间（字节）          |
> | EC       | Edem 区大小                                       |
> | EU       | 年轻代中 Edem 区已使用的空间（字节）              |
> | OC       | 老年代大小                                        |
> | OU       | 老年代已使用的空间（字节）                        |
> | YGC      | 从应用程序启动到采样时 young gc 的次数            |
> | YGCT     | 从应用程序启动到采样时 young gc 的所用的时间（s） |
> | FGC      | 从应用程序启动到采样时 full gc 的次数             |
> | FGCT     | 从应用程序启动到采样时 full gc 的所用的时间       |
> | GCT      | 从应用程序启动到采样时整个 gc 所用的时间          |
>
> > 注意：年轻代的 Edem 区满了会触发 young gc，老年代满了会触发 old gc。full gc 指的是清除整个堆，包括 young 区 和 old 区。
>
> jstat 常用的查询参数有：
>
> - -class，查询类加载器信息；
> - -compiler，JIT 相关信息；
> - -gc，GC 堆状态；
> - -gcnew，新生代统计信息；
> - -gcutil，GC 堆统计汇总信息。
>
> #### 3. jinfo（查询虚拟机参数配置工具）
>
> jinfo（Configuration Info for Java）用于查看和调整虚拟机各项参数。语法如下：
>
> 复制代码
>
> ```java
> jinfo <option> <pid>
> ```
>
> 查看 JVM 参数示例如下：
>
> 复制代码
>
> ```powershell
> ➜  jinfo -flags 45129
> VM Flags:
> -XX:CICompilerCount=3 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
> ```
>
> 其中 45129 是使用 jps 查询的 LVMID。
>  我们可以通过 jinfo -flag [+/-]name 来修改虚拟机的参数值，比如下面的示例：
>
> 复制代码
>
> ```powershell
> ➜  jinfo -flag PrintGC 45129 # 查询是否开启 GC 打印
> -XX:-PrintGC
> ➜  jinfo -flag +PrintGC 45129 # 开启 GC 打印
> ➜  jinfo -flag PrintGC 45129 # 查询是否开启 GC 打印
> -XX:+PrintGC
> ➜  jinfo -flag -PrintGC 45129 # 关闭 GC 打印
> ➜  jinfo -flag PrintGC 45129 # 查询是否开启 GC 打印
> -XX:-PrintGC
> ```
>
> #### 4. jmap（堆快照生成工具）
>
> jmap（Memory Map for Java）用于查询堆的快照信息。
>
> 查询堆信息示例如下：
>
> 复制代码
>
> ```powershell
> ➜  jmap -heap 45129
> Attaching to process ID 45129, please wait...
> Debugger attached successfully.
> Server compiler detected.
> JVM version is 25.101-b13
> using thread-local object allocation.
> Parallel GC with 6 thread(s)
> Heap Configuration:
>    MinHeapFreeRatio         = 0
>    MaxHeapFreeRatio         = 100
>    MaxHeapSize              = 4294967296 (4096.0MB)
>    NewSize                  = 89128960 (85.0MB)
>    MaxNewSize               = 1431306240 (1365.0MB)
>    OldSize                  = 179306496 (171.0MB)
>    NewRatio                 = 2
>    SurvivorRatio            = 8
>    MetaspaceSize            = 21807104 (20.796875MB)
>    CompressedClassSpaceSize = 1073741824 (1024.0MB)
>    MaxMetaspaceSize         = 17592186044415 MB
>    G1HeapRegionSize         = 0 (0.0MB)
> Heap Usage:
> PS Young Generation
> Eden Space:
>    capacity = 67108864 (64.0MB)
>    used     = 5369232 (5.1204986572265625MB)
>    free     = 61739632 (58.87950134277344MB)
>    8.000779151916504% used
> From Space:
>    capacity = 11010048 (10.5MB)
>    used     = 0 (0.0MB)
>    free     = 11010048 (10.5MB)
>    0.0% used
> To Space:
>    capacity = 11010048 (10.5MB)
>    used     = 0 (0.0MB)
>    free     = 11010048 (10.5MB)
>    0.0% used
> PS Old Generation
>    capacity = 179306496 (171.0MB)
>    used     = 0 (0.0MB)
>    free     = 179306496 (171.0MB)
>    0.0% used
> 
> 2158 interned Strings occupying 152472 bytes.
> ```
>
> 我们也可以直接生成堆快照文件，示例如下：
>
> 复制代码
>
> ```powershell
> ➜  jmap -dump:format=b,file=/Users/admin/Documents/2020.dump 47380
> Dumping heap to /Users/admin/Documents/2020.dump ...
> Heap dump file created
> ```
>
> #### 5. jhat（堆快照分析功能）
>
> jhat（JVM Heap Analysis Tool，堆快照分析工具）和 jmap 搭配使用，用于启动一个 web 站点来分析 jmap 生成的快照文件。
>
> 执行示例如下：
>
> 复制代码
>
> ```java
> jhat /Users/admin/Documents/2020.dump
> Reading from /Users/admin/Documents/2020.dump...
> Dump file created Tue May 26 16:12:41 CST 2020
> Snapshot read, resolving...
> Resolving 17797 objects...
> Chasing references, expect 3 dots...
> Eliminating duplicate references...
> Snapshot resolved.
> Started HTTP server on port 7000
> Server is ready.
> ```
>
> 上述信息表示 jhat 启动了一个 http 的服务器端口为 7000 的站点来展示信息，此时我们在浏览器中输入：http://localhost:7000/，会看到如下图所示的信息：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/19/0A/Ciqc1F7Z6YyAZxEUAAHRgiBjrK8704.png)
>
> #### 6. jstack（查询虚拟机当前的线程快照信息）
>
> jstack（Stack Trace for Java）用于查看当前虚拟机的线程快照，用它可以排查线程的执行状况，例如排查死锁、死循环等问题。
>
> 比如，我们先写一段死锁的代码：
>
> 复制代码
>
> ```java
> public class NativeOptimize {
>     private static Object obj1 = new Object();
>     private static Object obj2 = new Object();
>     public static void main(String[] args) {
>         new Thread(new Runnable() {
>             @Override
>             public void run() {
>                 synchronized (obj2) {
>                     System.out.println(Thread.currentThread().getName() + "锁住 obj2");
>                     try {
>                         Thread.sleep(1000);
>                     } catch (InterruptedException e) {
>                         e.printStackTrace();
>                     }
>                     synchronized (obj1) {
>                         // 执行不到这里
>                         System.out.println("1秒钟后，" + Thread.currentThread().getName()
>                                 + "锁住 obj1");
>                     }
>                 }
>             }
>         }).start();
>         synchronized (obj1) {
>             System.out.println(Thread.currentThread().getName() + "锁住 obj1");
>             try {
>                 Thread.sleep(1000);
>             } catch (InterruptedException e) {
>                 e.printStackTrace();
>             }
>             synchronized (obj2) {
>                 // 执行不到这里
>                 System.out.println("1秒钟后，" + Thread.currentThread().getName()
>                         + "锁住 obj2");
>             }
>         }
>     }
> }
> ```
>
> 以上程序的执行结果如下：
>
> 复制代码
>
> ```java
> main：锁住 obj1
> Thread-0：锁住 obj2
> ```
>
> 此时我们使用 jstack 工具打印一下当前线程的快照信息，结果如下：
>
> 复制代码
>
> ```powershell
> ➜  bin jstack -l 50016
> 2020-05-26 18:01:41
> Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.101-b13 mixed mode):
> "Attach Listener" #10 daemon prio=9 os_prio=31 tid=0x00007f8c00840800 nid=0x3c03 waiting on condition [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "Thread-0" #9 prio=5 os_prio=31 tid=0x00007f8c00840000 nid=0x3e03 waiting for monitor entry [0x00007000100c8000]
>    java.lang.Thread.State: BLOCKED (on object monitor)
> 	at com.example.optimize.NativeOptimize$1.run(NativeOptimize.java:25)
> 	- waiting to lock <0x000000076abb62d0> (a java.lang.Object)
> 	- locked <0x000000076abb62e0> (a java.lang.Object)
> 	at java.lang.Thread.run(Thread.java:745)
>    Locked ownable synchronizers:
> 	- None
> "Service Thread" #8 daemon prio=9 os_prio=31 tid=0x00007f8c01814800 nid=0x4103 runnable [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "C1 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007f8c0283c800 nid=0x4303 waiting on condition [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007f8c0300a800 nid=0x4403 waiting on condition [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007f8c0283c000 nid=0x3603 waiting on condition [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007f8c0283b000 nid=0x4603 runnable [0x0000000000000000]
>    java.lang.Thread.State: RUNNABLE
>    Locked ownable synchronizers:
> 	- None
> "Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007f8c03001000 nid=0x5003 in Object.wait() [0x000070000f8ad000]
>    java.lang.Thread.State: WAITING (on object monitor)
> 	at java.lang.Object.wait(Native Method)
> 	- waiting on <0x000000076ab08ee0> (a java.lang.ref.ReferenceQueue$Lock)
> 	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
> 	- locked <0x000000076ab08ee0> (a java.lang.ref.ReferenceQueue$Lock)
> 	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
> 	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
>    Locked ownable synchronizers:
> 	- None
> "Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007f8c03000000 nid=0x2f03 in Object.wait() [0x000070000f7aa000]
>    java.lang.Thread.State: WAITING (on object monitor)
> 	at java.lang.Object.wait(Native Method)
> 	- waiting on <0x000000076ab06b50> (a java.lang.ref.Reference$Lock)
> 	at java.lang.Object.wait(Object.java:502)
> 	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
> 	- locked <0x000000076ab06b50> (a java.lang.ref.Reference$Lock)
> 	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)
>    Locked ownable synchronizers:
> 	- None
> "main" #1 prio=5 os_prio=31 tid=0x00007f8c00802800 nid=0x1003 waiting for monitor entry [0x000070000ef92000]
>    java.lang.Thread.State: BLOCKED (on object monitor)
> 	at com.example.optimize.NativeOptimize.main(NativeOptimize.java:41)
> 	- waiting to lock <0x000000076abb62e0> (a java.lang.Object)
> 	- locked <0x000000076abb62d0> (a java.lang.Object)
>    Locked ownable synchronizers:
> 	- None
> "VM Thread" os_prio=31 tid=0x00007f8c01008800 nid=0x2e03 runnable
> "GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007f8c00803000 nid=0x2007 runnable
> 
> "GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007f8c00006800 nid=0x2403 runnable
> 
> "GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007f8c01800800 nid=0x2303 runnable
> "GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007f8c01801800 nid=0x2a03 runnable
> "GC task thread#4 (ParallelGC)" os_prio=31 tid=0x00007f8c01802000 nid=0x5403 runnable
> "GC task thread#5 (ParallelGC)" os_prio=31 tid=0x00007f8c01006800 nid=0x2d03 runnable
> "VM Periodic Task Thread" os_prio=31 tid=0x00007f8c00010800 nid=0x3803 waiting on condition
> JNI global references: 6
> Found one Java-level deadlock:
> =============================
> "Thread-0":
>   waiting to lock monitor 0x00007f8c000102a8 (object 0x000000076abb62d0, a java.lang.Object),
>   which is held by "main"
> "main":
>   waiting to lock monitor 0x00007f8c0000ed58 (object 0x000000076abb62e0, a java.lang.Object),
>   which is held by "Thread-0"
> 
> Java stack information for the threads listed above:
> ===================================================
> "Thread-0":
> 	at com.example.optimize.NativeOptimize$1.run(NativeOptimize.java:25)
> 	- waiting to lock <0x000000076abb62d0> (a java.lang.Object)
> 	- locked <0x000000076abb62e0> (a java.lang.Object)
> 	at java.lang.Thread.run(Thread.java:745)
> "main":
> 	at com.example.optimize.NativeOptimize.main(NativeOptimize.java:41)
> 	- waiting to lock <0x000000076abb62e0> (a java.lang.Object)
> 	- locked <0x000000076abb62d0> (a java.lang.Object)
> 
> Found 1 deadlock.
> ```
>
> 从上述信息可以看出使用 jstack ，可以很方便地排查出代码中出现“deadlock”（死锁）的问题。
>
> ### 考点分析
>
> Java 虚拟机的排查工具是一个合格程序员必备的技能，使用它我们可以很方便地定位出问题的所在，尤其在团队合作的今天，每个人各守一摊很容易出现隐藏的 bug（缺陷）。因此使用这些排查功能可以帮我们快速地定位并解决问题，所以它也是面试中常问的问题之一。
>
> 和此知识点相关的面试题还有以下这些：
>
> - 除了比较实用的命令行工具之外，有没有方便一点的排查工具？
> - JVM 常见的调优手段有哪些？
>
> ### 知识扩展
>
> #### 可视化排查工具
>
> JVM 除了上面的 6 个基础命令行工具之外，还有两个重要的视图调试工具，即 JConsole 和 JVisualVM，它们相比于命令行工具使用更方便、操作更简单、结果展现也更直观。
>
> JConsole 和 JVisualVM 都位于 JDK 的 bin 目录下，JConsole（Java Monitoring and Management Console）是最早期的视图调试工具，其启动页面如下图所示：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/19/0A/Ciqc1F7Z6ZuAZOoBAAIE-PlP8mY751.png)
>
> 可以看出我们可以用它来连接远程的服务器，或者是直接调试本机，这样就可以在不消耗生产环境的性能下，从本机启动 JConsole 来连接服务器。选择了调试的进程之后，运行界面如下图所示：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/19/16/CgqCHl7Z6aKAZfp0AAKE9E3ar6c820.png)
>
> 从上图可以看出，使用 JConsole 可以监控线程、CPU、类、堆以及 VM 的相关信息，同样我们可以通过线程这一页的信息，发现之前我们故意写的死锁问题，如下图所示：
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/19/16/CgqCHl7Z6amAF-LOAAKoeNJzszw795.png)
>
> 可以看到 main（主线程）和 Thread-0 线程处于死锁状态。
>
> JVisualVM 的启动图如下图所示：
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/19/16/CgqCHl7Z6a-AK0DMAAHTxm7JgYI402.png)
>
> 由上图可知，JVisualVM 既可以调试本地也可以调试远程服务器，当我们选择了相关的进程之后，运行如下图所示：
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/19/16/CgqCHl7Z6beAUJ6sAAMkaaTyA9U352.png)
>
> 可以看出 JVisualVM 除了包含了 JConsole 的信息之外，还有更多的详细信息，并且更加智能。例如，线程死锁检查的这页内容如下图所示：
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/19/0A/Ciqc1F7Z6b2Aa8CCAANpsufYncw124.png)
>
> 可以看出 JVisualVM 会直接给你一个死锁的提示，而 JConsole 则需要程序员自己分析。
>
> #### JVM 调优
>
> JVM 调优主要是根据实际的硬件配置信息重新设置 JVM 参数来进行调优的，例如，硬件的内存配置很高，但 JVM 因为是默认参数，所以最大内存和初始化堆内存很小，这样就不能更好地利用本地的硬件优势了。因此，需要调整这些参数，让 JVM 在固定的配置下发挥最大的价值。
>
> JVM 常见调优参数包含以下这些：
>
> - -Xmx，设置最大堆内存大小；
> - -Xms，设置初始堆内存大小；
> - -XX:MaxNewSize，设置新生代的最大内存；
> - -XX:MaxTenuringThreshold，设置新生代对象经过一定的次数晋升到老生代；
> - -XX:PretrnureSizeThreshold，设置大对象的值，超过这个值的对象会直接进入老生代；
> - -XX:NewRatio，设置分代垃圾回收器新生代和老生代内存占比；
> - -XX:SurvivorRatio，设置新生代 Eden、Form Survivor、To Survivor 占比。
>
> 我们要根据自己的业务场景和硬件配置来设置这些值。例如，当我们的业务场景会有很多大的临时对象产生时，因为这些大对象只有很短的生命周期，因此需要把“-XX:MaxNewSize”的值设置的尽量大一些，否则就会造成大量短生命周期的大对象进入老生代，从而很快消耗掉了老生代的内存，这样就会频繁地触发 full gc，从而影响了业务的正常运行。
>
> ### 小结
>
> 本课时我们讲了 JVM 排查的 6 个基本命令行工具：jps、jstat、jinfo、jmap、jhat、jstack，以及 2 个视图排查工具：JConsole 和 JVisualVM；同时还讲了 JVM 的常见调优参数，希望本课时的内容可以切实的帮助到你。