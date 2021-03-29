[Spring Data JPA 原理与实战](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=490&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=4720)



> 你好，欢迎来到第 20 讲。前面我们已经学习完了两个模块：基础知识以及高阶用法与实战的内容，不知道你掌握得如何，有疑问的地方一定要留言提问，或者和大家一起讨论，请记住学习的路上你不是一个人在战斗。
>
> 那么从这一讲开始，我们进入“模块三：原理与问题排查”知识的学习。这一模块，我将带你了解Hibernate 的加载过程、Session 和事务之间的关系，帮助你知道在遇到 LazyException 以及经典的 N+1 SQL 问题时该如何解决，希望你在工作中可以灵活运用所学知识。
>
> 这一讲，我们来分析一下在 Spring Data JPA 的项目下面 Hibernate 的配置参数有哪些，先从 Hibernate 的整体架构进行分析。
>
> ### Hibernate 架构分析
>
> 首先看一下 Hibernate 5.2 版本中，官方提供的架构图。
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08OiAT8tUAAAtx02IC70594.png)
>
> 从架构图上，我们可以知道 Hiberante 实现的 ORM 的接口有两种，一种是 Hiberante 自己的 API 接口；一种是 Java Persistence API 的接口实现。
>
> 因为 Hibernate 其实是比 Java Persistence API 早几年发展的，后来才有了 Java 的持久化协议。以我个人的观点来看，随着时间的推移，Hiberante 的实现逻辑可能会逐渐被弱化，由 Java Persistence API 统一对外提供服务。
>
> 那么有了这个基础，我们研究 Hibernate 在 Spring Data JPA 里面的作用，得出的结论就是：Hibernate 5.2 是 Spring Data JPA 持久化操作的核心。我们再从类上面具体看一下，关键类的图如下所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08PGACMAWAABkBYWN3EQ292.png)
>
> 结合类的关系图来看，Session 接口和 SessionFactory 接口都是 Hibernate 的概念，而 EntityManger 和 EntityManagerFactory 都是 Java Persistence API 协议规定的接口。
>
> 不过 HibernateEntityManger 从 Hibernate 5.2 之后就开始不推荐使用了，而是建议直接使用 EntityManager 接口即可。那么我们看看 Hibernate 在 Spring BOOT 里面是如何被加载进去的。
>
> ### Hibernate 5 在 Spring Boot 2 里面的加载过程
>
> 不同的 Spring Boot 版本，可能加载类的实现逻辑是不一样的，但是分析过程都是相同的。我们先打开 spring.factories 文件，如下图所示，其中可以自动加载 Hibernate 的只有一个类，那就是 HibernateJpaAutoConfiguration。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/6F/3A/Ciqc1F-08PqAOGq1AAPvvo_ZD7w314.png)
>
> HibernateJpaAutoConfiguration 就是 Spring Boot 加载 Hibernate 的主要入口，所以我们可以直接打开这个类看一下。
>
> 复制代码
>
> ```
> @Configuration(proxyBeanMethods = false)
> @ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class, SessionImplementor.class })
> @EnableConfigurationProperties(JpaProperties.class)//JPAProperties的配置
> @AutoConfigureAfter({ DataSourceAutoConfiguration.class })
> @Import(HibernateJpaConfiguration.class) //hibernate加载的关键类
> public class HibernateJpaAutoConfiguration {
> }
> ```
>
> 其中，我们第一个需要关注的就是 JpaProperties 类，因为通过这个类我们可以间接知道，application.properties 里面可以配置的 spring.jpa 的属性有哪些。
>
> #### JpaProperties 属性
>
> 我们打开 JpaProperties 类看一下，如下图所示。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/6F/3A/Ciqc1F-08Q6AR-PoAAGybbuAJ7g272.png)
>
> 通过这个类，我们可以在 application.properties 里面得到如下配置项。
>
> 复制代码
>
> ```
> # 可以配置JPA的实现者的原始属性的配置，如：这里我们用的JPA的实现者是hibernate
> # 那么hibernate里面的一些属性设置就可以通过如下方式实现，具体properties里面有哪些，本讲会详细介绍，我们先知道这里可以设置即可
> spring.jpa.properties.hibernate.hbm2ddl.auto=none
> #hibernate的persistence.xml文件有哪些，目前已经不推荐使用
> #spring.jpa.mapping-resources=
> # 指定数据源的类型，如果不指定，Spring Boot加载Datasource的时候会根据URL的协议自己判断
> # 如：spring.datasource.url=jdbc:mysql://localhost:3306/test 上面可以明确知道是mysql数据源，所以这个可以不需要指定；
> # 应用场景，当我们通过代理的方式，可能通过datasource.url没办法判断数据源类型的时候，可以通过如下方式指定，可选的值有：DB2,H2,HSQL,INFORMIX,MYSQL,ORACLE,POSTGRESQL,SQL_SERVER,SYBASE)
> spring.jpa.database=mysql
> # 是否在启动阶段根据实体初始化数据库的schema，默认false，当我们用内存数据库做测试的时候可以打开，很有用
> spring.jpa.generate-ddl=false
> # 和spring.jpa.database用法差不多，指定数据库的平台，默认会自己发现；一般不需要指定，database-platform指定的必须是org.hibernate.dialect.Dialect的子类，如mysql默认是用下面的platform
> spring.jpa.database-platform=org.hibernate.dialect.MySQLInnoDBDialect
> # 是否在view层打开session，默认是true，其实大部分场景不需要打开，我们可以设置成false，
> # 22课时我们再详细讲解
> spring.jpa.open-in-view=false
> # 是否显示sql，当执行JPA的数据库操作的时候，默认是false，在本地开发的时候我们可以把这个打开，有助于分析sql是不是我们预期的
> # 在生产环境的时候建议给这个设置成false，改由logging.level.org.hibernate.SQL=DEBUG代替，这样的话日志默认是基于logback输出的
> # 而不是直接打印到控制台的，有利于增加traceid和线程ID等信息，便于分析
> spring.jpa.show-sql=true
> ```
>
> 其中，spring.jpa.show-sql=true 输出的 sql 效果如下所示。
>
> 复制代码
>
> ```
> Hibernate: insert into user_info (create_time, create_user_id, last_modified_time, last_modified_user_id, version, ages, email_address, last_name, telephone, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
> ```
>
> 上面是孤立无援的 System.out.println 的效果，如果是在线上环境，多线程的情况下就不知道是哪个线程输出来的，而 logging.level.org.hibernate.SQL=DEBUG 输出的 sql 效果如下所示。
>
> 复制代码
>
> ```
> 2020-11-08 16:54:22.275 DEBUG 6589 --- [nio-8087-exec-1] org.hibernate.SQL                        : insert into user_info (create_time, create_user_id, last_modified_time, last_modified_user_id, version, ages, email_address, last_name, telephone, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
> ```
>
> 这样我们可以轻易知道线程 ID 和执行时间，甚至可以有 tranceID 和 spanID 进行日志跟踪，方便分析是哪个线程打印的。
>
> 我们了解完了 JpaProperties，下面再看另外一个关键类 HibernateJpaConfiguration，它也是 HibernateJpaAutoConfiguration 导入进来加载的。
>
> #### HibernateJpaConfiguration 分析
>
> 我们通过上述 HibernateJpaAutoConfiguration 里面的 @Import(HibernateJpaConfiguration.class)，打开 HibernateJpaConfiguration.class 看看是什么情况。
>
> 复制代码
>
> ```
> @Configuration(proxyBeanMethods = false)
> @EnableConfigurationProperties(HibernateProperties.class)
> @ConditionalOnSingleCandidate(DataSource.class)
> class HibernateJpaConfiguration extends JpaBaseConfiguration {
> .......//其他我们暂不关心的代码我们可以先省略}
> ```
>
> 通过源码我们可以得到 Hibernate 在 JPA 中配置的三个重要线索，下面详细说明。
>
> **第一个线索：HibernatePropertes 这个配置类对应的是 spring.jpa.hibernate 的配置。**
>
> 我们通过源码可以看得出来，@EnableConfigurationProperties(HibernateProperties.class) 启用了 HibernatePropertes 的配置类，如下图所示。
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08RuALMNSAAKTACKebbE349.png)
>
> 其中可以看到 application.properties 的配置项，如下所示。
>
> 复制代码
>
> ```
> # 正如我们之前课时讲到的nameing的物理策略值有：org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy(默认)和org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
> spring.jpa.hibernate.naming.physical-strategy=
> # ddl的生成策略，默认none；如果我们没有指定任何数据源的url，采用的是spring的集成数据源，也就是内存数据源H2的时候，默认值是create-drop；
> # 所以你会发现当我们每次用H2的时候什么都没做，它就会自动帮我们创建表等，内存数据库和写测试用的时候，create-drop就非常方便了；不过，当我们生产数据库的时候一定要设置成none;
> spring.jpa.hibernate.ddl-auto=none
> # 当我们的@Id配置成@GeneratedValue(strategy= GenerationType.AUTO)的时候是否采用hibernate的Id-generator-mappings(即会默认帮我们创建一张表hibernate_sequence来存储和生成ID)，默认是true
> spring.jpa.hibernate.use-new-id-generator-mappings=true
> ```
>
> **第二个线索：通过源码我们还可以看得出来，HibernateJpaConfiguration 的父类 JpaBaseConfiguration 也会优先加载，此类就是 Spring Boot 加载 JPA 的核心逻辑。**
>
> 那么我们打开 JpaBaseConfiguration 类看一下源码。
>
> 复制代码
>
> ```
> @Configuration(proxyBeanMethods = false)
> @EnableConfigurationProperties(JpaProperties.class)
> //DataSourceInitializedPublisher用来进行数据源的初始化操作
> @Import(DataSourceInitializedPublisher.Registrar.class)
> public abstract class JpaBaseConfiguration implements BeanFactoryAware {
> protected JpaBaseConfiguration(DataSource dataSource, JpaProperties properties,
>       ObjectProvider<JtaTransactionManager> jtaTransactionManager) {
>    this.dataSource = dataSource;
>    this.properties = properties;
>    //jtaTransactionManager赋值，正常情况下我们用不到，一般用来解决分布式事务的场景才会用到。
>    this.jtaTransactionManager = jtaTransactionManager.getIfAvailable(); 
> }
> //加载JPA的实现方式
> @Bean
> @ConditionalOnMissingBean
> public JpaVendorAdapter jpaVendorAdapter() {
>    //createJpaVendorAdapter是由子类HibernateJpaConfiguration实现的，创建JPA的实现类
>    AbstractJpaVendorAdapter adapter = createJpaVendorAdapter();
>    adapter.setShowSql(this.properties.isShowSql());
>    if (this.properties.getDatabase() != null) {
>       adapter.setDatabase(this.properties.getDatabase());
>    }
>    if (this.properties.getDatabasePlatform() != null) {
>    adapter.setDatabasePlatform(this.properties.getDatabasePlatform());
>    }
>    adapter.setGenerateDdl(this.properties.isGenerateDdl());
>    return adapter;
> }
> .......其他我们暂时不关心的代码先省略}
> ```
>
> 我们从上面的源码中可以看到，@Import(DataSourceInitializedPublisher.Registrar.class) 是用来初始化数据的；从构造函数中我们也可以看到其是否有用到 jtaTransactionManager（这个是分布式事务才会用到）；而 createJpaVendorAdapter() 是在 HibernateJpaConfiguration 里面实现的，这个要重点说一下，关键代码如下。
>
> 复制代码
>
> ```
> class HibernateJpaConfiguration extends JpaBaseConfiguration {
> //这里是hibernate和Jpa的结合，可以看到使用的HibernateJpaVendorAdapter作为JPA的实现者，感兴趣的话你可以打开HibernateJpaVendorAdapter里面设置一些断点，就会知道Spring boot是如何一步一步加载Hibernate的了；
> @Override
> protected AbstractJpaVendorAdapter createJpaVendorAdapter() {
>    return new HibernateJpaVendorAdapter();
> }
> .......}
> ```
>
> 现在我们知道了 HibernateJpaVendorAdapter 的加载逻辑，而 HibernateJpaVendorAdapter 里面实现了 Hibernate 的初始化逻辑，我在这里不多说了，你过后可以仔细 debug 看一下，基本上就是 Hibernate 5.2 官方的加载逻辑。那么 Hibernate Jpa 对应的原始配置有哪些呢？
>
> **第三个线索：spring.jpa.properties 配置项有哪些？**
>
> 我们如果接着在 HibernateJpaConfiguration 类里面 debug 查看关键代码的话，可以找到如下代码。
>
> ![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08SmAc9J8AARC3m5NP2o468.png)
>
> 上图中的代码显示，JpaProperties 类里面的 properties 属性，也就是 spring.jpa.properties 的配置加载到了 vendorProperties（即 Hibernate 5.2）里面。而 properties 里面是 HashMap 结构，那么它都可以支持哪些配置呢？
>
> 我们打开 org.hibernate.cfg.AvailableSettings 可以看到 Hibernate 支持的配置项大概有 100 多个配置信息，如下所示。
>
> 复制代码
>
> ```
> String JPA_PERSISTENCE_PROVIDER = "javax.persistence.provider";
> String JPA_TRANSACTION_TYPE = "javax.persistence.transactionType";
> String JPA_JTA_DATASOURCE = "javax.persistence.jtaDataSource";
> String JPA_NON_JTA_DATASOURCE = "javax.persistence.nonJtaDataSource";
> String JPA_JDBC_DRIVER = "javax.persistence.jdbc.driver";
> String JPA_JDBC_URL = "javax.persistence.jdbc.url";
> String JPA_JDBC_USER = "javax.persistence.jdbc.user";
> String JPA_JDBC_PASSWORD = "javax.persistence.jdbc.password";
> String JPA_SHARED_CACHE_MODE = "javax.persistence.sharedCache.mode";
> String JPA_SHARED_CACHE_RETRIEVE_MODE ="javax.persistence.cache.retrieveMode";
> String JPA_SHARED_CACHE_STORE_MODE ="javax.persistence.cache.storeMode";
> String JPA_VALIDATION_MODE = "javax.persistence.validation.mode";
> String JPA_VALIDATION_FACTORY = "javax.persistence.validation.factory";
> String JPA_PERSIST_VALIDATION_GROUP = "javax.persistence.validation.group.pre-persist";
> String JPA_UPDATE_VALIDATION_GROUP = "javax.persistence.validation.group.pre-update";
> String JPA_REMOVE_VALIDATION_GROUP = "javax.persistence.validation.group.pre-remove";
> String JPA_LOCK_SCOPE = "javax.persistence.lock.scope";
> String JPA_LOCK_TIMEOUT = "javax.persistence.lock.timeout";
> String CDI_BEAN_MANAGER = "javax.persistence.bean.manager";
> String CLASSLOADERS = "hibernate.classLoaders";
> String TC_CLASSLOADER = "hibernate.classLoader.tccl_lookup_precedence";
> String APP_CLASSLOADER = "hibernate.classLoader.application";
> String RESOURCES_CLASSLOADER = "hibernate.classLoader.resources";
> String HIBERNATE_CLASSLOADER = "hibernate.classLoader.hibernate";
> String ENVIRONMENT_CLASSLOADER = "hibernate.classLoader.environment";
> String JPA_METAMODEL_GENERATION = "hibernate.ejb.metamodel.generation";
> String JPA_METAMODEL_POPULATION = "hibernate.ejb.metamodel.population";
> String STATIC_METAMODEL_POPULATION = "hibernate.jpa.static_metamodel.population";
> String CONNECTION_PROVIDER ="hibernate.connection.provider_class";
> String DRIVER ="hibernate.connection.driver_class";
> String URL ="hibernate.connection.url";
> String USER ="hibernate.connection.username";
> String PASS ="hibernate.connection.password";
> String ISOLATION ="hibernate.connection.isolation";
> String AUTOCOMMIT = "hibernate.connection.autocommit";
> String POOL_SIZE ="hibernate.connection.pool_size";
> String DATASOURCE ="hibernate.connection.datasource";
> String CONNECTION_PROVIDER_DISABLES_AUTOCOMMIT= "hibernate.connection.provider_disables_autocommit";
> String CONNECTION_PREFIX = "hibernate.connection";
> String JNDI_CLASS ="hibernate.jndi.class";
> String JNDI_URL ="hibernate.jndi.url";
> String JNDI_PREFIX = "hibernate.jndi";
> String DIALECT ="hibernate.dialect";
> String DIALECT_RESOLVERS = "hibernate.dialect_resolvers";
> String STORAGE_ENGINE = "hibernate.dialect.storage_engine";
> String SCHEMA_MANAGEMENT_TOOL = "hibernate.schema_management_tool";
> String TRANSACTION_COORDINATOR_STRATEGY = "hibernate.transaction.coordinator_class";
> String JTA_PLATFORM = "hibernate.transaction.jta.platform";
> String PREFER_USER_TRANSACTION = "hibernate.jta.prefer_user_transaction";
> String JTA_PLATFORM_RESOLVER = "hibernate.transaction.jta.platform_resolver";
> String JTA_CACHE_TM = "hibernate.jta.cacheTransactionManager";
> String JTA_CACHE_UT = "hibernate.jta.cacheUserTransaction";
> String JDBC_TYLE_PARAMS_ZERO_BASE = "hibernate.query.sql.jdbc_style_params_base";
> String DEFAULT_CATALOG = "hibernate.default_catalog";
> String DEFAULT_SCHEMA = "hibernate.default_schema";
> String DEFAULT_CACHE_CONCURRENCY_STRATEGY = "hibernate.cache.default_cache_concurrency_strategy";
> String USE_NEW_ID_GENERATOR_MAPPINGS = "hibernate.id.new_generator_mappings";
> String FORCE_DISCRIMINATOR_IN_SELECTS_BY_DEFAULT = "hibernate.discriminator.force_in_select";
> String IMPLICIT_DISCRIMINATOR_COLUMNS_FOR_JOINED_SUBCLASS = "hibernate.discriminator.implicit_for_joined";
> String IGNORE_EXPLICIT_DISCRIMINATOR_COLUMNS_FOR_JOINED_SUBCLASS = "hibernate.discriminator.ignore_explicit_for_joined";
> String USE_NATIONALIZED_CHARACTER_DATA = "hibernate.use_nationalized_character_data";
> String SCANNER_DEPRECATED = "hibernate.ejb.resource_scanner";
> String SCANNER = "hibernate.archive.scanner";
> String SCANNER_ARCHIVE_INTERPRETER = "hibernate.archive.interpreter";
> String SCANNER_DISCOVERY = "hibernate.archive.autodetection";
> String IMPLICIT_NAMING_STRATEGY = "hibernate.implicit_naming_strategy";
> String PHYSICAL_NAMING_STRATEGY = "hibernate.physical_naming_strategy";
> String ARTIFACT_PROCESSING_ORDER = "hibernate.mapping.precedence";
> String KEYWORD_AUTO_QUOTING_ENABLED = "hibernate.auto_quote_keyword";
> String XML_MAPPING_ENABLED = "hibernate.xml_mapping_enabled";
> String SESSION_FACTORY_NAME = "hibernate.session_factory_name";
> String SESSION_FACTORY_NAME_IS_JNDI = "hibernate.session_factory_name_is_jndi";
> String SHOW_SQL ="hibernate.show_sql";
> String FORMAT_SQL ="hibernate.format_sql";
> String USE_SQL_COMMENTS ="hibernate.use_sql_comments";
> String MAX_FETCH_DEPTH = "hibernate.max_fetch_depth";
> String DEFAULT_BATCH_FETCH_SIZE = "hibernate.default_batch_fetch_size";
> String USE_STREAMS_FOR_BINARY = "hibernate.jdbc.use_streams_for_binary";
> String USE_SCROLLABLE_RESULTSET = "hibernate.jdbc.use_scrollable_resultset";
> String USE_GET_GENERATED_KEYS = "hibernate.jdbc.use_get_generated_keys";
> String STATEMENT_FETCH_SIZE = "hibernate.jdbc.fetch_size";
> String STATEMENT_BATCH_SIZE = "hibernate.jdbc.batch_size";
> String BATCH_STRATEGY = "hibernate.jdbc.factory_class";
> String BATCH_VERSIONED_DATA = "hibernate.jdbc.batch_versioned_data";
> String JDBC_TIME_ZONE = "hibernate.jdbc.time_zone";
> String AUTO_CLOSE_SESSION = "hibernate.transaction.auto_close_session";
> String FLUSH_BEFORE_COMPLETION = "hibernate.transaction.flush_before_completion";
> String ACQUIRE_CONNECTIONS = "hibernate.connection.acquisition_mode";
> String RELEASE_CONNECTIONS = "hibernate.connection.release_mode";
> String CONNECTION_HANDLING = "hibernate.connection.handling_mode";
> String CURRENT_SESSION_CONTEXT_CLASS = "hibernate.current_session_context_class";
> String USE_IDENTIFIER_ROLLBACK = "hibernate.use_identifier_rollback";
> String USE_REFLECTION_OPTIMIZER = "hibernate.bytecode.use_reflection_optimizer";
> String ENFORCE_LEGACY_PROXY_CLASSNAMES = "hibernate.bytecode.enforce_legacy_proxy_classnames";
> String ALLOW_ENHANCEMENT_AS_PROXY = "hibernate.bytecode.allow_enhancement_as_proxy";
> String QUERY_TRANSLATOR = "hibernate.query.factory_class";
> String QUERY_SUBSTITUTIONS = "hibernate.query.substitutions";
> String QUERY_STARTUP_CHECKING = "hibernate.query.startup_check";
> String CONVENTIONAL_JAVA_CONSTANTS = "hibernate.query.conventional_java_constants";
> String SQL_EXCEPTION_CONVERTER = "hibernate.jdbc.sql_exception_converter";
> String WRAP_RESULT_SETS = "hibernate.jdbc.wrap_result_sets";
> String NATIVE_EXCEPTION_HANDLING_51_COMPLIANCE = "hibernate.native_exception_handling_51_compliance";
> String ORDER_UPDATES = "hibernate.order_updates";
> String ORDER_INSERTS = "hibernate.order_inserts";
> String JPA_CALLBACKS_ENABLED = "hibernate.jpa_callbacks.enabled";
> String DEFAULT_NULL_ORDERING = "hibernate.order_by.default_null_ordering";
> String LOG_JDBC_WARNINGS =  "hibernate.jdbc.log.warnings";
> String BEAN_CONTAINER = "hibernate.resource.beans.container";
> String C3P0_CONFIG_PREFIX = "hibernate.c3p0";
> String C3P0_MAX_SIZE = "hibernate.c3p0.max_size";
> String C3P0_MIN_SIZE = "hibernate.c3p0.min_size";
> String C3P0_TIMEOUT = "hibernate.c3p0.timeout";
> String C3P0_MAX_STATEMENTS = "hibernate.c3p0.max_statements";
> String C3P0_ACQUIRE_INCREMENT = "hibernate.c3p0.acquire_increment";
> String C3P0_IDLE_TEST_PERIOD = "hibernate.c3p0.idle_test_period";
> String PROXOOL_CONFIG_PREFIX = "hibernate.proxool";
> String PROXOOL_PREFIX = PROXOOL_CONFIG_PREFIX;
> String PROXOOL_XML = "hibernate.proxool.xml";
> String PROXOOL_PROPERTIES = "hibernate.proxool.properties";
> String PROXOOL_EXISTING_POOL = "hibernate.proxool.existing_pool";
> String PROXOOL_POOL_ALIAS = "hibernate.proxool.pool_alias";
> String CACHE_REGION_FACTORY = "hibernate.cache.region.factory_class";
> String CACHE_KEYS_FACTORY = "hibernate.cache.keys_factory";
> String CACHE_PROVIDER_CONFIG = "hibernate.cache.provider_configuration_file_resource_path";
> String USE_SECOND_LEVEL_CACHE = "hibernate.cache.use_second_level_cache";
> String USE_QUERY_CACHE = "hibernate.cache.use_query_cache";
> String QUERY_CACHE_FACTORY = "hibernate.cache.query_cache_factory";
> String CACHE_REGION_PREFIX = "hibernate.cache.region_prefix";
> String USE_MINIMAL_PUTS = "hibernate.cache.use_minimal_puts";
> String USE_STRUCTURED_CACHE = "hibernate.cache.use_structured_entries";
> String AUTO_EVICT_COLLECTION_CACHE = "hibernate.cache.auto_evict_collection_cache";
> String USE_DIRECT_REFERENCE_CACHE_ENTRIES = "hibernate.cache.use_reference_entries";
> String DEFAULT_ENTITY_MODE = "hibernate.default_entity_mode";
> String GLOBALLY_QUOTED_IDENTIFIERS = "hibernate.globally_quoted_identifiers";
> String GLOBALLY_QUOTED_IDENTIFIERS_SKIP_COLUMN_DEFINITIONS = "hibernate.globally_quoted_identifiers_skip_column_definitions";
> String CHECK_NULLABILITY = "hibernate.check_nullability";
> String BYTECODE_PROVIDER = "hibernate.bytecode.provider";
> String JPAQL_STRICT_COMPLIANCE= "hibernate.query.jpaql_strict_compliance";
> String PREFER_POOLED_VALUES_LO = "hibernate.id.optimizer.pooled.prefer_lo";
> String PREFERRED_POOLED_OPTIMIZER = "hibernate.id.optimizer.pooled.preferred";
> String QUERY_PLAN_CACHE_MAX_STRONG_REFERENCES = "hibernate.query.plan_cache_max_strong_references";
> String QUERY_PLAN_CACHE_MAX_SOFT_REFERENCES = "hibernate.query.plan_cache_max_soft_references";
> String QUERY_PLAN_CACHE_MAX_SIZE = "hibernate.query.plan_cache_max_size";
> String QUERY_PLAN_CACHE_PARAMETER_METADATA_MAX_SIZE = "hibernate.query.plan_parameter_metadata_max_size";
> String NON_CONTEXTUAL_LOB_CREATION = "hibernate.jdbc.lob.non_contextual_creation";
> String HBM2DDL_AUTO = "hibernate.hbm2ddl.auto";
> String HBM2DDL_DATABASE_ACTION = "javax.persistence.schema-generation.database.action";
> String HBM2DDL_SCRIPTS_ACTION = "javax.persistence.schema-generation.scripts.action";
> String HBM2DDL_CONNECTION = "javax.persistence.schema-generation-connection";
> String HBM2DDL_DB_NAME = "javax.persistence.database-product-name";
> String HBM2DDL_DB_MAJOR_VERSION = "javax.persistence.database-major-version";
> String HBM2DDL_DB_MINOR_VERSION = "javax.persistence.database-minor-version";
> String HBM2DDL_CREATE_SOURCE = "javax.persistence.schema-generation.create-source";
> String HBM2DDL_DROP_SOURCE = "javax.persistence.schema-generation.drop-source";
> String HBM2DDL_CREATE_SCRIPT_SOURCE = "javax.persistence.schema-generation.create-script-source";
> String HBM2DDL_DROP_SCRIPT_SOURCE = "javax.persistence.schema-generation.drop-script-source";
> String HBM2DDL_SCRIPTS_CREATE_TARGET = "javax.persistence.schema-generation.scripts.create-target";
> String HBM2DDL_SCRIPTS_DROP_TARGET = "javax.persistence.schema-generation.scripts.drop-target";
> String HBM2DDL_IMPORT_FILES = "hibernate.hbm2ddl.import_files";
> String HBM2DDL_LOAD_SCRIPT_SOURCE = "javax.persistence.sql-load-script-source";
> String HBM2DDL_IMPORT_FILES_SQL_EXTRACTOR = "hibernate.hbm2ddl.import_files_sql_extractor";
> String HBM2DDL_CREATE_NAMESPACES = "hibernate.hbm2ddl.create_namespaces";
> String HBM2DLL_CREATE_NAMESPACES = "hibernate.hbm2dll.create_namespaces";
> String HBM2DDL_CREATE_SCHEMAS = "javax.persistence.create-database-schemas";
> String HBM2DLL_CREATE_SCHEMAS = HBM2DDL_CREATE_SCHEMAS;
> String HBM2DDL_FILTER_PROVIDER = "hibernate.hbm2ddl.schema_filter_provider";
> String HBM2DDL_JDBC_METADATA_EXTRACTOR_STRATEGY = "hibernate.hbm2ddl.jdbc_metadata_extraction_strategy";
> String HBM2DDL_DELIMITER = "hibernate.hbm2ddl.delimiter";
> String HBM2DDL_CHARSET_NAME = "hibernate.hbm2ddl.charset_name";
> String HBM2DDL_HALT_ON_ERROR = "hibernate.hbm2ddl.halt_on_error";
> String JMX_ENABLED = "hibernate.jmx.enabled";
> String JMX_PLATFORM_SERVER = "hibernate.jmx.usePlatformServer";
> String JMX_AGENT_ID = "hibernate.jmx.agentId";
> String JMX_DOMAIN_NAME = "hibernate.jmx.defaultDomain";
> String JMX_SF_NAME = "hibernate.jmx.sessionFactoryName";
> String JMX_DEFAULT_OBJ_NAME_DOMAIN = "org.hibernate.core";
> String CUSTOM_ENTITY_DIRTINESS_STRATEGY = "hibernate.entity_dirtiness_strategy";
> String USE_ENTITY_WHERE_CLAUSE_FOR_COLLECTIONS = "hibernate.use_entity_where_clause_for_collections";
> String MULTI_TENANT = "hibernate.multiTenancy";
> String MULTI_TENANT_CONNECTION_PROVIDER = "hibernate.multi_tenant_connection_provider";
> String MULTI_TENANT_IDENTIFIER_RESOLVER = "hibernate.tenant_identifier_resolver";
> String INTERCEPTOR = "hibernate.session_factory.interceptor";
> String SESSION_SCOPED_INTERCEPTOR = "hibernate.session_factory.session_scoped_interceptor";
> String STATEMENT_INSPECTOR = "hibernate.session_factory.statement_inspector";
> String ENABLE_LAZY_LOAD_NO_TRANS = "hibernate.enable_lazy_load_no_trans";
> String HQL_BULK_ID_STRATEGY = "hibernate.hql.bulk_id_strategy";
> String BATCH_FETCH_STYLE = "hibernate.batch_fetch_style";
> String DELAY_ENTITY_LOADER_CREATIONS = "hibernate.loader.delay_entity_loader_creations";
> String JTA_TRACK_BY_THREAD = "hibernate.jta.track_by_thread";
> String JACC_CONTEXT_ID = "hibernate.jacc_context_id";
> String JACC_PREFIX = "hibernate.jacc";
> String JACC_ENABLED = "hibernate.jacc.enabled";
> String ENABLE_SYNONYMS = "hibernate.synonyms";
> String EXTRA_PHYSICAL_TABLE_TYPES = "hibernate.hbm2ddl.extra_physical_table_types";
> String DEPRECATED_EXTRA_PHYSICAL_TABLE_TYPES = "hibernate.hbm2dll.extra_physical_table_types";
> String UNIQUE_CONSTRAINT_SCHEMA_UPDATE_STRATEGY = "hibernate.schema_update.unique_constraint_strategy";
> String GENERATE_STATISTICS = "hibernate.generate_statistics";
> String LOG_SESSION_METRICS = "hibernate.session.events.log";
> String LOG_SLOW_QUERY = "hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS";
> String AUTO_SESSION_EVENTS_LISTENER = "hibernate.session.events.auto";
> String PROCEDURE_NULL_PARAM_PASSING = "hibernate.proc.param_null_passing";
> String CREATE_EMPTY_COMPOSITES_ENABLED = "hibernate.create_empty_composites.enabled";
> String ALLOW_JTA_TRANSACTION_ACCESS = "hibernate.jta.allowTransactionAccess";
> String ALLOW_UPDATE_OUTSIDE_TRANSACTION = "hibernate.allow_update_outside_transaction";
> String COLLECTION_JOIN_SUBQUERY = "hibernate.collection_join_subquery";
> String ALLOW_REFRESH_DETACHED_ENTITY = "hibernate.allow_refresh_detached_entity";
> String MERGE_ENTITY_COPY_OBSERVER = "hibernate.event.merge.entity_copy_observer";
> String USE_LEGACY_LIMIT_HANDLERS = "hibernate.legacy_limit_handler";
> String VALIDATE_QUERY_PARAMETERS = "hibernate.query.validate_parameters";
> String CRITERIA_LITERAL_HANDLING_MODE = "hibernate.criteria.literal_handling_mode";
> String PREFER_GENERATOR_NAME_AS_DEFAULT_SEQUENCE_NAME = "hibernate.model.generator_name_as_sequence_name";
> String JPA_TRANSACTION_COMPLIANCE = "hibernate.jpa.compliance.transaction";
> String JPA_QUERY_COMPLIANCE = "hibernate.jpa.compliance.query";
> String JPA_LIST_COMPLIANCE = "hibernate.jpa.compliance.list";
> String JPA_CLOSED_COMPLIANCE = "hibernate.jpa.compliance.closed";
> String JPA_PROXY_COMPLIANCE = "hibernate.jpa.compliance.proxy";
> String JPA_CACHING_COMPLIANCE = "hibernate.jpa.compliance.caching";
> String JPA_ID_GENERATOR_GLOBAL_SCOPE_COMPLIANCE = "hibernate.jpa.compliance.global_id_generators";
> String TABLE_GENERATOR_STORE_LAST_USED = "hibernate.id.generator.stored_last_used";
> String FAIL_ON_PAGINATION_OVER_COLLECTION_FETCH = "hibernate.query.fail_on_pagination_over_collection_fetch";
> String IMMUTABLE_ENTITY_UPDATE_QUERY_HANDLING_MODE = "hibernate.query.immutable_entity_update_query_handling_mode";
> String IN_CLAUSE_PARAMETER_PADDING = "hibernate.query.in_clause_parameter_padding";
> String QUERY_STATISTICS_MAX_SIZE = "hibernate.statistics.query_max_size";
> String SEQUENCE_INCREMENT_SIZE_MISMATCH_STRATEGY = "hibernate.id.sequence.increment_size_mismatch_strategy";
> String OMIT_JOIN_OF_SUPERCLASS_TABLES = "hibernate.query.omit_join_of_superclass_tables";
> ```
>
> 我担心有些同学懒得去看源码，所以就都贴到这里来了，你可以大概了解一下，做到心中有数。
>
> 那么接下来我们看看该怎么使用 AvailableSettings 里面的配置呢？
>
> **AvailableSettings 里面的配置项的用法**
>
> 我们只需要将 AvailableSettings 变量的值放到 spring.jpa.properties 里面即可，如下这些是我们常用的。
>
> 复制代码
>
> ```
> ##开启hibernate statistics的信息，如session、连接等日志：
> spring.jpa.properties.hibernate.generate_statistics=true
> # 格式化 SQL
> spring.jpa.properties.hibernate.format_sql: true
> # 显示 SQL
> spring.jpa.properties.hibernate.show_sql: true
> # 添加 HQL 相关的注释信息
> spring.jpa.properties.hibernate.use_sql_comments: true
> # hbm2ddl的策略 validate, update, create, create-drop, none，建议配置成validate，
> # 这样在我们启动项目的时候就知道生产数据库的表结构是否正确的了，而不用等到运行期间才发现问题。
> spring.jpa.properties.hibernate.hbm2ddl.auto=validate
> # 关联关系的时候取数据的深度，默认是3层，我们可以设置成2级，防止其他开发乱用，提高sql性能
> spring.jpa.properties.hibernate.max_fetch_depth=2
> # 批量fetch大小默认 -1
> spring.jpa.properties.hibernate.default_batch_fetch_size= 100
> # 事务完成之前是否进行flush操作，即同步到db里面去，默认是true
> spring.jpa.properties.hibernate.transaction.flush_before_completion=true
> # 事务结束之后是否关闭session，默认false
> spring.jpa.properties.hibernate.transaction.auto_close_session=false
> # 有的时候不只要批量查询，也会批量更新，默认batch size是15，我们可以根据实际情况自由调整，可以提高批量更新的效率；
> spring.jpa.properties.hibernate.jdbc.batch_size=100
> ```
>
> 其他的配置不经常用，我们就不需要关心了，你只知道在哪里看就好，实际用到时，发现哪些是我没举例的，你直接看源码会非常好理解的。
>
> 这里我为什么要特别强调这个 Hibernate 的配置类呢？因为有的时候我们遇到问题会去网上搜索解决方案，发现别人给的配置可能不对，那么你就可以想到从这个源码中进行查看，并找到解决办法。
>
> 本讲我们只关心了 JpaVendorAdapter 和 properties 的创建逻辑，我们前面在讲数据源的时候也说过这个类，里面有我们关心的 PlatformTransactionManager transactionManager 和 LocalContainerEntityManagerFactoryBean entityManagerFactory 的创建逻辑，而 JpaBaseConfiguration 这个类实现的逻辑还有很多，我在第 22 讲介绍 Session 的配置 open-in-view 的时候还会再详细介绍这个类。
>
> 那么说了这么多加载的类，它们之间是什么关系呢？我们通过一个图来知晓一下。
>
> #### 自动加载过程类之间的关系图
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08UuABFufAAFail5ZuqU603.png)
>
> 从上图中，我们可以看出以下几点内容。
>
> 1. JpaBaseConfiguration 是 Jpa 和 Hibernate 被加载的基石，里面通过 BeanFactoryAware 的接口的 bean 加载生命周期也实现了一些逻辑。
> 2. HibernateJpaConfiguration 是 JpaBaseConfiguration 的子类，覆盖了一些父类里面的配置相关的特殊逻辑，并且里面引用了 JpaPropeties 和 HibernateProperties 的配置项。
> 3. HibernateJpaAutoConfiguration 是 Spring Boot 自动加载 HibernateJpaConfiguration 的桥梁，起到了 importHibernateJpaConfiguration 和加载 HibernateJpaConfiguration 的作用。
> 4. JpaRepositoriesAutoConfiguration 和 HibernateJpaAutoConfiguration、DataSourceAutoConfiguration 分别加载 JpaRepositories 的逻辑和 HibernateJPA、数据源，都是被 spring.factories 自动装配进入到 Spring Boot 里面的，而三者之间有加载的先后顺序。
> 5. 上图的 UML 还展示了几个 Configuration 类的加载顺序和依赖关系，顺序是从上到下进行加载的，其中 DataSourceAutoConfiguration 最先加载、HibernateJpaAutoConfiguration 第二顺序加载、JpaRepositoriesAutoConfiguration 最后加载。
>
> 我们了解完了 Hibernate 5 在 Spring Boot 里面的加载过程，那么来看下 JpaRepositoriesAutoConfiguration 的主要作用有哪些。
>
> ### Spring Data JPA Repositories Bootstrap Mode
>
> 我们通过上面分享的整个加载过程可以发现，DataSourceAutoConfiguration 完成了数据源的加载，HibernateJpaAutoConfiguration 完成了 Hibernate 的加载过程，而 JpaRepositoriesAutoConfiguration 要做的就是解决我们之前定义的 Repositories 相关的实体和接口的加载初始化过程，这是 Spring Data JPA 的主要实现逻辑，和 Hiberante、数据源没什么关系了。
>
> 我们可以通过 JpaRepositoriesAutoConfiguration 的源码发现其主要职责和实现方式，利用异步线程池初始化 repositories，关键源码如下：
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/6F/45/CgqCHl-08VmASV72AAJavgahY_A852.png)
>
> 而其中加载 repositories 有三种方式，即 spring.data.jpa.repositories.bootstrap-mode 的三个值，分别为 deferred、 lazy、 default，下面详细说明。
>
> - deferred：是默认值，表示在启动的时候会进行数据库字段的检查，而 repositories 相关的实例的初始化是 lazy 模式，也就是在第一次用到 repositories 实例的时候再进行初始化。这个比较适合用在测试环境和生产环境中，因为测试不可能覆盖所有场景，万一谁多加个字段或者少一个字段，这样在启动的阶段就可以及时发现问题，不能等进行到生产环境才暴露。
> - lazy：表示启动阶段不会进行数据库字段的检查，也不会初始化 repositories 相关的实例，而是在第一次用到 repositories 实例的时候再进行初始化。这个比较适合用在开发的阶段，可以加快应用的启动速度。如果生产环境中，我们为了提高业务高峰期间水平来扩展应用的启动速度，也可以采用这种模式。
> - default：默认加载方式，但从 Spring Boot 2.0 之后就不是默认值了，表示立即验证、立即初始化 repositories 实例，这种方式启动的速度最慢，但是最保险，运行期间的请求最快，因为避免了第一次请求初始化 repositories 实例的过程。
>
> 我们通过在 application.properties 里面修改这一行代码，来测试一下 lazy 的加载方式。
>
> 复制代码
>
> ```
> spring.data.jpa.repositories.bootstrap-mode=lazy
> ```
>
> 然后启动我们的项目，就会发现在 tomcat 容器加载完之后，没有用到 UserInfoRepository 之前，这个 UserInfoRepository 是不会进行初始化的。而当我们发一个请求用到了 UserInfoRepository，就进行了初始化。
>
> 我们通过日志也可以看到，启动的线程和初始化的线程是不一样的，而初始化的线程是 NIO 线程的名字，表示 request 的 http 线程池里面的线程，具体如下图所示。
>
> ![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/6F/3A/Ciqc1F-08WGAaqVkAAR8a19UBFQ188.png)
>
> 我们在分析 Hibernate 的加载方式的时候，会发现日志的重要性，那么都有哪些日志供我们观察呢？如何开启？
>
> ### Debug 时候，日志的配置
>
> 复制代码
>
> ```
> ### 日志级别的灵活运用
> ## hibernate相关
> # 显示sql的执行日志，如果开了这个,show_sql就可以不用了
> logging.level.org.hibernate.SQL=debug
> # hibernate id的生成日志
> logging.level.org.hibernate.id=debug
> # hibernate所有的操作都是PreparedStatement，把sql的执行参数显示出来
> logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
> # sql执行完提取的返回值
> logging.level.org.hibernate.type.descriptor.sql=trace
> # 请求参数
> logging.level.org.hibernate.type=debug
> # 缓存相关
> logging.level.org.hibernate.cache=debug
> # 统计hibernate的执行状态
> logging.level.org.hibernate.stat=debug
> # 查看所有的缓存操作
> logging.level.org.hibernate.event.internal=trace
> logging.level.org.springframework.cache=trace
> # hibernate 的监控指标日志
> logging.level.org.hibernate.engine.internal.StatisticalLoggingSessionEventListener=DEBUG
> ### 连接池的相关日志
> ## hikari连接池的状态日志，以及连接池是否完好 #连接池的日志效果：HikariCPPool - Pool stats (total=20, active=0, idle=20, waiting=0)
> logging.level.com.zaxxer.hikari=TRACE
> #开启 debug可以看到 AvailableSettings里面的默认配置的值都有哪些，会输出类似下面的日志格式
> # org.hibernate.cfg.Settings               : Statistics: enabled
> # org.hibernate.cfg.Settings               : Default batch fetch size: -1
> logging.level.org.hibernate.cfg=debug
> #hikari数据的配置项日志
> logging.level.com.zaxxer.hikari.HikariConfig=TRACE
> ### 查看事务相关的日志，事务获取，释放日志
> logging.level.org.springframework.orm.jpa=DEBUG
> logging.level.org.springframework.transaction=TRACE
> logging.level.org.hibernate.engine.transaction.internal.TransactionImpl=DEBUG
> ### 分析connect 以及 orm和 data的处理过程更全的日志
> logging.level.org.springframework.data=trace
> logging.level.org.springframework.orm=trace
> ```
>
> 上面是我在分析复杂问题和原理的时候常用的日志配置项目，这里给你提供一个技巧，当我们分析一个问题的时候，如果不知道日志具体在哪个类里面，通过设置 logging.level.root=trace 的话，日志又非常多几乎没有办法看，那么我们可以缩小范围，不如说我们分析的是 hikari 包里面相关的问题。
>
> 我们可以把整个日志级别 logging.level.root=info 设置成 info，把其他所有的日志都关闭，并把 logging.level.com.zaxxer=trace 设置成最大的，保持日志不受干扰，然后观察日志再逐渐减少查看范围。
>
> ### 总结
>
> 这一讲我通过源码分析，帮助你了解了 JpaRepositoriesAutoConfiguration、HibernateJpaAutoConfiguration、DataSourceAutoConfiguration 的主要作用和加载顺序的依赖，还介绍了 Spring Hibernate 的配置项有哪些。
>
> 你在工作中可以举一反三，通过 debug 断点一步一步分析出来这一讲没涉及的东西。比如可以自己做一个项目，跟着我的步骤操作，你会对这部分的内容有更深刻的体会。这样当遇到一些问题，并且网上没有合适的资料时，你可以试着采用本讲中我分享给你的思路来解决。
>
> 下一讲，我会为你介绍一个 Hibernate 实现的 JPA 的概念：Persistence Context。欢迎你提前预习，并结合这一讲内容去思考，有疑问的地方请留言，我会及时给予答复。
>
> > 点击下方链接查看源码（不定时更新）
> >  https://github.com/zhangzhenhuajack/spring-boot-guide/tree/master/spring-data/spring-data-jpa