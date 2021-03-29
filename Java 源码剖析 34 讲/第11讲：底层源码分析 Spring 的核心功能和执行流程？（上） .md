[Java 面试真题及源码 34 讲 - 前 360 技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=59&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=1771)



> Spring Framework 已是公认的 Java 标配开发框架了，甚至还有人说 Java 编程就是面向 Spring 编程的，可见 Spring 在整个 Java 体系中的重要位置。
>
> Spring 中包含了众多的功能和相关模块，比如 spring-core、spring-beans、spring-aop、spring-context、spring-expression、spring-test 等，本课时先从面试中必问的问题出发，来帮你更好的 Spring 框架。
>
> 我们本课时的面试题是，Spring Bean 的作用域有哪些？它的注册方式有几种？
>
> ### 典型回答
>
> 在 Spring 容器中管理一个或多个 Bean，这些 Bean 的定义表示为 BeanDefinition 对象，这些对象包含以下重要信息：
>
> - Bean 的实际实现类
> - Bean 的作用范围
> - Bean 的引用或者依赖项
>
> Bean 的注册方式有三种：
>
> - XML 配置文件的注册方式
> - Java 注解的注册方式
> - Java API 的注册方式
>
> #### 1. XML 配置文件注册方式
>
> 复制代码
>
> ```html
> <bean id="person" class="org.springframework.beans.Person">
>    <property name="id" value="1"/>
>    <property name="name" value="Java"/>
> </bean>
> ```
>
> #### 2. Java 注解注册方式
>
> 可以使用 @Component 注解方式来注册 Bean，代码如下：
>
> 复制代码
>
> ```java
> @Component
> public class Person {
>    private Integer id;
>    private String name
>    // 忽略其他方法
> }
> ```
>
> 也可以使用 @Bean 注解方式来注册 Bean，代码如下：
>
> 复制代码
>
> ```java
> @Configuration
> public class Person {
>    @Bean
>    public Person  person(){
>       return new Person();
>    }
>    // 忽略其他方法
> }
> ```
>
> 其中 @Configuration 可理解为 XML 配置里的 <beans> 标签，而 @Bean 可理解为用 XML 配置里面的 <bean> 标签。
>
> #### 3. Java API 注册方式
>
> 使用 BeanDefinitionRegistry.registerBeanDefinition() 方法的方式注册 Bean，代码如下：
>
> 复制代码
>
> ```java
> public class CustomBeanDefinitionRegistry implements BeanDefinitionRegistryPostProcessor {
> 	@Override
> 	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
> 	}
> 	@Override
> 	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
> 		RootBeanDefinition personBean = new RootBeanDefinition(Person.class);
> 		// 新增 Bean
> 		registry.registerBeanDefinition("person", personBean);
> 	}
> }
> ```
>
> Bean 的作用域一共有 5 个。
>
> （1）**singleton 作用域**：表示在 Spring 容器中只有一个 Bean 实例，以单例的形式存在，是默认的 Bean 作用域。
>
> 配置方式，缺省即可，XML 的配置方式如下：
>
> 复制代码
>
> ```html
> <bean class="..."></bean>
> ```
>
> （2）**prototype 作用域**：原型作用域，每次调用 Bean 时都会创建一个新实例，也就是说每次调用 getBean() 方法时，相当于执行了 new Bean()。
>
> XML 的配置方式如下：
>
> 复制代码
>
> ```html
> <bean class="..." scope="prototype"></bean>
> ```
>
> （3）**request 作用域**：每次 Http 请求时都会创建一个新的 Bean，该作用域仅适应于 WebApplicationContext 环境。
>
> XML 的配置方式如下：
>
> 复制代码
>
> ```html
> <bean class="..." scope="request"></bean>
> ```
>
> Java 注解的配置方式如下：
>
> 复制代码
>
> ```java
> @Scope(WebApplicationContext.SCOPE_REQUEST)
> ```
>
> 或是：
>
> 复制代码
>
> ```java
> @RequestScope(WebApplicationContext.SCOPE_REQUEST)
> ```
>
> （4）**session 作用域**：同一个 Http Session 共享一个 Bean 对象，不同的 Session 拥有不同的 Bean 对象，仅适用于 WebApplicationContext 环境。
>
> XML 的配置方式如下：
>
> 复制代码
>
> ```html
> <bean class="..." scope="session"></bean>
> ```
>
> Java 注解的配置方式如下：
>
> 复制代码
>
> ```java
> @Scope(WebApplicationContext.SCOPE_SESSION)
> ```
>
> 或是：
>
> 复制代码
>
> ```java
> @RequestScope(WebApplicationContext.SCOPE_SESSION)
> ```
>
> （5）**application 作用域**：全局的 Web 作用域，类似于 Servlet 中的 Application。
>
> XML 的配置方式如下：
>
> 复制代码
>
> ```html
> <bean class="..." scope="application"></bean>
> ```
>
> Java 注解的配置方式如下：
>
> 复制代码
>
> ```java
> @Scope(WebApplicationContext.SCOPE_APPLICATION)
> ```
>
> 或是：
>
> 复制代码
>
> ```java
> @RequestScope(WebApplicationContext.SCOPE_APPLICATION)
> ```
>
> ### 考点分析
>
> 在 Spring 中最核心的概念是 AOP（面向切面编程）、IoC（控制反转）、DI（依赖注入）等（此内容将会在下一课时中讲到），而最实用的功能则是 Bean，他们是概念和具体实现的关系。和 Bean 相关的面试题，还有以下几个：
>
> - 什么是同名 Bean？它是如何产生的？应该如何避免？
> - 聊一聊 Bean 的生命周期。
>
> ### 知识扩展
>
> #### 1.同名 Bean 问题
>
> 每个 Bean 拥有一个或多个标识符，在基于 XML 的配置中，我们可以使用 id 或者 name 来作为 Bean 的标识符。通常 Bean 的标识符由字母组成，允许使用特殊字符。
>
> 同一个 Spring 配置文件中 Bean 的 id 和 name 是不能够重复的，否则 Spring 容器启动时会报错。但如果 Spring 加载了多个配置文件的话，可能会出现同名 Bean 的问题。**同名 Bean** 指的是多个 Bean 有相同的 name 或者 id。
>
> Spring 对待同名 Bean 的处理规则是使用最后面的 Bean 覆盖前面的 Bean，所以我们在定义 Bean 时，尽量使用长命名非重复的方式来定义，避免产生同名 Bean 的问题。
>
> Bean 的 id 或 name 属性并非必须指定，如果留空的话，容器会为 Bean 自动生成一个唯一的
>
> 名称，这样也不会出现同名 Bean 的问题。
>
> #### 2.Bean 生命周期
>
> 对于 Spring Bean 来说，并不是启动阶段就会触发 Bean 的实例化，只有当客户端通过显式或者隐式的方式调用 BeanFactory 的 getBean() 方法时，它才会触发该类的实例化方法。当然对于 BeanFactory 来说，也不是所有的 getBean() 方法都会实例化 Bean 对象，例如作用域为 singleton 时，只会在第一次，实例化该 Bean 对象，之后会直接返回该对象。但如果使用的是 ApplicationContext 容器，则会在该容器启动的时候，立即调用注册到该容器所有 Bean 的实例化方法。
>
> getBean() 既然是 Bean 对象的入口，我们就先从这个方法说起，getBean() 方法是属于 BeanFactory 接口的，它的真正实现是 AbstractAutowireCapableBeanFactory 的 createBean() 方法，而 createBean() 是通过 doCreateBean() 来实现的，具体源码实现如下：
>
> 复制代码
>
> ```java
> @Override
> protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
>         throws BeanCreationException {
>     if (logger.isTraceEnabled()) {
>         logger.trace("Creating instance of bean '" + beanName + "'");
>     }
>     RootBeanDefinition mbdToUse = mbd;
>     // 确定并加载 Bean 的 class
>     Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
>     if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
>         mbdToUse = new RootBeanDefinition(mbd);
>         mbdToUse.setBeanClass(resolvedClass);
>     }
>     // 验证以及准备需要覆盖的方法
>     try {
>         mbdToUse.prepareMethodOverrides();
>     }
>     catch (BeanDefinitionValidationException ex) {
>         throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
>                 beanName, "Validation of method overrides failed", ex);
>     }
>     try {
>         // 给BeanPostProcessors 一个机会来返回代理对象来代替真正的 Bean 实例，在这里实现创建代理对象功能
>         Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
>         if (bean != null) {
>             return bean;
>         }
>     }
>     catch (Throwable ex) {
>         throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
>                 "BeanPostProcessor before instantiation of bean failed", ex);
>     }
>     try {
>         // 创建 Bean
>         Object beanInstance = doCreateBean(beanName, mbdToUse, args);
>         if (logger.isTraceEnabled()) {
>             logger.trace("Finished creating instance of bean '" + beanName + "'");
>         }
>         return beanInstance;
>     }
>     catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
>         throw ex;
>     }
>     catch (Throwable ex) {
>         throw new BeanCreationException(
>                 mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
>     }
> }
> ```
>
> doCreateBean 源码如下：
>
> 复制代码
>
> ```java
> protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
>         throws BeanCreationException {
>     // 实例化 bean，BeanWrapper 对象提供了设置和获取属性值的功能
>     BeanWrapper instanceWrapper = null;
>     // 如果 RootBeanDefinition 是单例，则移除未完成的 FactoryBean 实例的缓存
>     if (mbd.isSingleton()) {
>         instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
>     }
>     if (instanceWrapper == null) {
>         // 创建 bean 实例
>         instanceWrapper = createBeanInstance(beanName, mbd, args);
>     }
>     // 获取 BeanWrapper 中封装的 Object 对象，其实就是 bean 对象的实例
>     final Object bean = instanceWrapper.getWrappedInstance();
>     // 获取 BeanWrapper 中封装 bean 的 Class
>     Class<?> beanType = instanceWrapper.getWrappedClass();
>     if (beanType != NullBean.class) {
>         mbd.resolvedTargetType = beanType;
>     }
>     // 应用 MergedBeanDefinitionPostProcessor 后处理器，合并 bean 的定义信息
>     // Autowire 等注解信息就是在这一步完成预解析，并且将注解需要的信息放入缓存
>     synchronized (mbd.postProcessingLock) {
>         if (!mbd.postProcessed) {
>             try {
>                 applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
>             } catch (Throwable ex) {
>                 throw new BeanCreationException(mbd.getResourceDescription(), beanName,
>                         "Post-processing of merged bean definition failed", ex);
>             }
>             mbd.postProcessed = true;
>         }
>     }
>     boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
>             isSingletonCurrentlyInCreation(beanName));
>     if (earlySingletonExposure) {
>         if (logger.isTraceEnabled()) {
>             logger.trace("Eagerly caching bean '" + beanName +
>                     "' to allow for resolving potential circular references");
>         }
>         // 为了避免循环依赖，在 bean 初始化完成前，就将创建 bean 实例的 ObjectFactory 放入工厂缓存（singletonFactories）
>         addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
>     }
>     // 对 bean 属性进行填充
>     Object exposedObject = bean;
>     try {
>         populateBean(beanName, mbd, instanceWrapper);
>         // 调用初始化方法，如 init-method 注入 Aware 对象
>         exposedObject = initializeBean(beanName, exposedObject, mbd);
>     } catch (Throwable ex) {
>         if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
>             throw (BeanCreationException) ex;
>         } else {
>             throw new BeanCreationException(
>                     mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
>         }
>     }
>     if (earlySingletonExposure) {
>         // 如果存在循环依赖，也就是说该 bean 已经被其他 bean 递归加载过，放入了提早公布的 bean 缓存中
>         Object earlySingletonReference = getSingleton(beanName, false);
>         if (earlySingletonReference != null) {
>             // 如果 exposedObject 没有在 initializeBean 初始化方法中被增强
>             if (exposedObject == bean) {
>                 exposedObject = earlySingletonReference;
>             } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
>                 // 依赖检测
>                 String[] dependentBeans = getDependentBeans(beanName);
>                 Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
>                 for (String dependentBean : dependentBeans) {
>                     if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
>                         actualDependentBeans.add(dependentBean);
>                     }
>                 }
>                 // 如果 actualDependentBeans 不为空，则表示依赖的 bean 并没有被创建完，即存在循环依赖
>                 if (!actualDependentBeans.isEmpty()) {
>                     throw new BeanCurrentlyInCreationException(beanName,
>                             "Bean with name '" + beanName + "' has been injected into other beans [" +
>                                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
>                                     "] in its raw version as part of a circular reference, but has eventually been " +
>                                     "wrapped. This means that said other beans do not use the final version of the " +
>                                     "bean. This is often the result of over-eager type matching - consider using " +
>                                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
>                 }
>             }
>         }
>     }
>     try {
>         // 注册 DisposableBean 以便在销毁时调用
>         registerDisposableBeanIfNecessary(beanName, bean, mbd);
>     } catch (BeanDefinitionValidationException ex) {
>         throw new BeanCreationException(
>                 mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
>     }
>     return exposedObject;
> }
> ```
>
> 从上述源码中可以看出，在 doCreateBean() 方法中，首先对 Bean 进行了实例化工作，它是通过调用 createBeanInstance() 方法来实现的，该方法返回一个 BeanWrapper 对象。BeanWrapper 对象是 Spring 中一个基础的 Bean 结构接口，说它是基础接口是因为它连基本的属性都没有。
>
> BeanWrapper 接口有一个默认实现类 BeanWrapperImpl，其主要作用是对 Bean 进行填充，比如填充和注入 Bean 的属性等。
>
> 当 Spring 完成 Bean 对象实例化并且设置完相关属性和依赖后，则会调用 Bean 的初始化方法 initializeBean()，初始化第一个阶段是检查当前 Bean 对象是否实现了 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware 等接口，源码如下：
>
> 复制代码
>
> ```java
> private void invokeAwareMethods(final String beanName, final Object bean) {
>     if (bean instanceof Aware) {
>         if (bean instanceof BeanNameAware) {
>             ((BeanNameAware) bean).setBeanName(beanName);
>         }
>         if (bean instanceof BeanClassLoaderAware) {
>             ClassLoader bcl = getBeanClassLoader();
>             if (bcl != null) {
>                 ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
>             }
>         }
>         if (bean instanceof BeanFactoryAware) {
>             ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
>         }
>     }
> }
> ```
>
> 其中，BeanNameAware 是把 Bean 对象定义的 beanName 设置到当前对象实例中；BeanClassLoaderAware 是将当前 Bean 对象相应的 ClassLoader 注入到当前对象实例中；BeanFactoryAware 是 BeanFactory 容器会将自身注入到当前对象实例中，这样当前对象就会拥有一个 BeanFactory 容器的引用。
>
> 初始化第二个阶段则是 BeanPostProcessor 增强处理，它主要是对 Spring 容器提供的 Bean 实例对象进行有效的扩展，允许 Spring 在初始化 Bean 阶段对其进行定制化修改，比如处理标记接口或者为其提供代理实现。
>
> 在初始化的前置处理完成之后就会检查和执行 InitializingBean 和 init-method 方法。
>
> InitializingBean 是一个接口，它有一个 afterPropertiesSet() 方法，在 Bean 初始化时会判断当前 Bean 是否实现了 InitializingBean，如果实现了则调用 afterPropertiesSet() 方法，进行初始化工作；然后再检查是否也指定了 init-method，如果指定了则通过反射机制调用指定的 init-method 方法，它的实现源码如下：
>
> 复制代码
>
> ```java
> protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
>         throws Throwable {
>     // 判断当前 Bean 是否实现了 InitializingBean，如果是的话需要调用 afterPropertiesSet()
>     boolean isInitializingBean = (bean instanceof InitializingBean);
>     if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
>         if (logger.isTraceEnabled()) {
>             logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
>         }
>         if (System.getSecurityManager() != null) { // 安全模式
>             try {
>                 AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
>                     ((InitializingBean) bean).afterPropertiesSet(); // 属性初始化
>                     return null;
>                 }, getAccessControlContext());
>             } catch (PrivilegedActionException pae) {
>                 throw pae.getException();
>             }
>         } else {
>             ((InitializingBean) bean).afterPropertiesSet(); // 属性初始化
>         }
>     }
>     // 判断是否指定了 init-method()
>     if (mbd != null && bean.getClass() != NullBean.class) {
>         String initMethodName = mbd.getInitMethodName();
>         if (StringUtils.hasLength(initMethodName) &&
>                 !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
>                 !mbd.isExternallyManagedInitMethod(initMethodName)) {
>             // 利用反射机制执行指定方法
>             invokeCustomInitMethod(beanName, bean, mbd);
>         }
>     }
> }
> ```
>
> 初始化完成之后就可以正常的使用 Bean 对象了，在 Spring 容器关闭时会执行销毁方法，但是 Spring 容器不会自动去调用销毁方法，而是需要我们主动的调用。
>
> 如果是 BeanFactory 容器，那么我们需要主动调用 destroySingletons() 方法，通知 BeanFactory 容器去执行相应的销毁方法；如果是 ApplicationContext 容器，那么我们需要主动调用 registerShutdownHook() 方法，告知 ApplicationContext 容器执行相应的销毁方法。
>
> > 注：本课时源码基于 Spring 5.2.2.RELEASE。
>
> ### 小结
>
> 本课时我们讲了 Bean 的三种注册方式：XML、Java 注解和 JavaAPI，以及 Bean 的五个作用域：singleton、prototype、request、session 和 application；还讲了读取多个配置文件可能会出现同名 Bean 的问题，以及通过源码讲了 Bean 执行的生命周期，它的生命周期如下图所示：
>
> ![img](https://s0.lgstatic.com/i/image3/M01/89/0C/Cgq2xl6WvHqAdmt4AABGAn2eSiI631.png)         