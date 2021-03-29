[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5958)



> 前几讲我们分别介绍了 Node.js 在同构项目以及性能守卫服务中的应用。结合当下热点，这一讲我们继续深入讲解 Node.js 另外一个重要的应用场景：企业级 BFF 网关。
>
> 网关这个话题可以和微服务、Serverless 等概念相结合，想象空间无限大，同时我们又要深入网关实现代码，抽丝剥茧。下面我们就开始今天内容的学习，请你做好准备。
>
> ### BFF 网关介绍和优缺点梳理
>
> 首先，我们对 BFF 网关做一个定义。BFF 即 Backend For Frontend，翻译过来就是服务于前端的后端。这个概念最早在[Pattern: Backends For Frontends](https://samnewman.io/patterns/architectural/bff/?fileGuid=xxQTRXtVcqtHK6j8)中提出，它不是一种技术，而是一种**逻辑分层**：在后端普遍采用微服务的技术背景下，**作为适配层能够更好地为前端服务，而传统业务后端只需要关注自己的微服务**即可。
>
> 我们结合下图进行拆解：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image6/M01/21/5E/Cgp9HWBURl6AGdLyAAGLpDcAPtI043.png)
>
> BFF 网关拆解图
>
> 如图所示，我们把用户体验适配和 API 网关聚合层合称为广义的 BFF 层，在 BFF 层的上游是各种后端业务微服务，在 BFF 下游就是各端应用。从职责上看，BFF 层向下给端提供 HTTP 接口，向上通过调用 HTTP 或 RPC 获取数据进行加工，最终完成整个 BFF 层的闭环。
>
> 对比传统的架构，我们可以得出 BFF 层设计的优势：
>
> - **降低沟通成本**，领域模型与页面数据更好地解耦；
> - 提供**更好的用户体验**，比如可以做到多端应用适配，根据不同端，提供更精简的数据。
>
> 但是 BFF 层需要谁来开发呢？这就引出了 BFF 的一些痛点：
>
> - **需要解决分工问题**，作为衔接前与后的环节，需要界定前后端职责，明确开发归属；
> - **链路复杂**，引入 BFF 层之后，流程变得更加烦琐；
> - **资源浪费**，BFF 层会带来一定额外资源的占用，需要有较好的弹性伸缩扩容机制。
>
> 通过分析 BFF 层的优缺点，我们可以明确打造一个 BFF 网关需要考虑的问题。而对于前端开发者来说，使用 Node.js 实现一个 BFF 网关则是一项当仁不让的工作。我们继续往下看。
>
> ### 打造 BFF 网关需要考虑的问题
>
> #### 数据处理
>
> 这里的数据处理，主要包括了：
>
> - 数据聚合和裁剪
> - 序列化格式转换
> - 协议转换
> - Node.js 调用 RPC
>
> 在微服务体系结构中，各个微服务的数据实体可能并不统一和规范，如果没有 BFF 层的统一处理，在端上进行不同数据格式的聚合会是一件非常痛苦的事情。因此，数据裁剪和聚合对于 BFF 网关来说就变得尤为重要了。
>
> 同时，**不同端可能也会需要不同的数据序列化格式**。比如，某个微服务使用 JSON，而某个客户只能使用 XML，那么 JSON 转换为 XML 的工作，也应当合理地在 BFF 网关层实现。
>
> 再比如微服务架构一般允许多语言协议传输，比如客户端需要通过 HTTP REST 进行所有的通信，而某个微服务内部使用了 gRPC 或 GraphQL，其中的语言协议转换，也需要在 BFF 网关层解决。
>
> 还需要你了解的是，在传统开发模式中，前端通过 Node.js 实现 BFF 的模式：前端请求 BFF 提供的接口，BFF 直接通过 HTTP Client 或者 cURL 方式透传给微服务——这种模式有其优势，但是可以做到精益求精。相比 BFF 不做任何逻辑处理，**Node.js 是一个 Proxy，我们可以思考如何让 Node.js 调用 RPC**，以最大限度地发挥 BFF 层能力。
>
> #### 流量处理
>
> 这里的流量处理主要是指：
>
> - 请求分发能力、代理能力；
> - 可用性保障。
>
> 在 BFF 层网关中，我们需要执行一些代理操作，比如将请求路由到特定服务。**在 Node.js 中，我们可以使用**`http-proxy`来简单代理特定服务。
>
> 我们需要考虑**网关层如何维护分发路由**这个关键问题。简单来说，我们可以 hard coding 写在代码里，同时也可以实现网关层的服务发现。比如，在 URL 规范化的基础上，网关层进行请求匹配时，可以只根据 URL 内容对应不同的 namespace 进而对应到不同的微服务。当然也可以使用中心化配置，通过配置来维护网关层路由分发。
>
> 除此之外，网关层也要考虑条件路由，即对具有特定内容（或者一定流量比例）的请求进行筛选并分发到特定实例组上，这种条件路由能力是实现灰度发布、蓝绿发布、AB Test 等功能的基础。
>
> 另外，BFF 网关直面用户，因此这一层也需要有良好的限速、隔离、熔断降级、负载均衡和缓存能力。
>
> 关于这些内容，我们会在后半部分代码环节中进一步体现。
>
> #### 安全问题
>
> 鉴于 BFF 层承上启下的位置，BFF 要考虑数据流向的安全性，需要完成必要的校验逻辑。其原则是：
>
> - BFF 层不需要完成全部的校验逻辑，部分业务校验应该留在微服务中完成；
> - BFF 需要完成必要的检查，比如请求头检查和必要的数据消毒；
> - 合理使用 Content-Security-Policy；
> - 使用 HTTPS/HSTS；
> - 设置监控报警以及调用链追踪能力。
>
> 同时，在使用 Node.js 做 BFF 层时，需要开发者**时刻注意依赖包的安全性**，可以考虑在 CI/CD 环节使用`nsp`、`npm audit`等工具进行安全审计。
>
> #### 权限与校验设计
>
> 在上面提到的安全问题中，一个关键的设计就是 BFF 层的用户权限校验。这里我们单独展开说明。
>
> 对于大多数微服务基础架构来说，需要将身份验证和权限校验等共享逻辑放入网关层，这样不仅能够帮助后端开发者缩小服务的体积，也能让后端开发者更专注于自身领域。
>
> 在网关中，一般我们需要支持基于 cookie 或 token 的身份验证。关于身份验证的话题这里我们不详细展开，值得一提的是，需要开发者关注 SSO 单点登录的设计。
>
> 关于权限问题，一般主流采用 ACL 或 RBAC 的方式，这就需要开发者系统学习权限设计知识。简单来说，**ACL 即访问控制列表，它的核心在于用户直接和权限挂钩。RBAC 的核心是用户只和角色关联，而角色对应了权限**，这样设计的优势在于：对用户而言，只需分配角色即可以实现权限管理，而某角色可以拥有各种各样的权限并可以继承。
>
> ACL 和 RBAC 相比，缺点在于由于用户和权限直接挂钩，导致在授予时的复杂性；虽然可以利用组（角色）来简化这个复杂性，但 RBAC 仍然会导致系统不好理解，而且在判断用户是否有该权限时比较困难，一定程度上影响了效率。
>
> 总之，设计一个良好的 BFF 网关，要求开发者具有较强的综合能力。下面，我们就来实现一个精简的网关系统，该网关只保留了最核心的能力，以性能为重要目标，同时支持能力扩展。
>
> ### 实现一个 lucas-gateway
>
> 如何设计一个扩展性良好的 BFF 层，以灵活支持上述需要考量的问题呢？我们来看几个关键的思路。
>
> - **插件化**：一个良好的 BFF 层设计可以内置或可插拔多种插件，比如 Logger 等，也可以接受第三方插件。
> - **中间件化**：SSO、限流、熔断等策略可以通过中间件形式实现，类似插件，中间件也可以进行定制和扩展。
>
> 下面我们就实战实现一个 BFF 网关，请随我一起深入代码。该实现代码我主要 fork 了[jkyberneees 的 fast-gateway](https://github.com/jkyberneees/fast-gateway?fileGuid=xxQTRXtVcqtHK6j8)，源代码放在了：[HOUCe](https://github.com/HOUCe?fileGuid=xxQTRXtVcqtHK6j8)/[fast-gateway](https://github.com/HOUCe/fast-gateway?fileGuid=xxQTRXtVcqtHK6j8)当中。
>
> 我们先看看开发这个网关的必要依赖。
>
> - [fast-proxy](https://www.npmjs.com/package/fast-proxy?fileGuid=xxQTRXtVcqtHK6j8)：支持 HTTP、HTTPS、HTTP2 三种协议，可以高性能完成请求的转发、代理。
> - [@polka/send-type](https://www.npmjs.com/package/@polka/send-type?fileGuid=xxQTRXtVcqtHK6j8)：处理 HTTP 响应的工具函数。
> - [http-cache-middleware](https://www.npmjs.com/package/http-cache-middleware?fileGuid=xxQTRXtVcqtHK6j8)：是一个高性能的 HTTP 缓存中间件。
> - [restana](https://www.npmjs.com/package/restana?fileGuid=xxQTRXtVcqtHK6j8)：一个极简的 REST 风格的 Node.js 框架。
>
> 我们的设计主要从
>
> - 基本反代理
> - 中间件
> - 缓存
> - Hooks
>
> 几个方向展开。
>
> #### 基本反代理
>
> 设计使用方式如下代码：
>
> 复制代码
>
> ```
> const gateway = require('lucas-gateway')
> const server = gateway({
>   routes: [{
>     prefix: '/service',
>     target: 'http://127.0.0.1:3000'
>   }]
> })
> server.start(8080)
> ```
>
> 网关暴露出`gateway`方法进行请求反向代理。如上代码，我们将 prefix 为`/service`的请求反向代理到`http://127.0.0.1:3000`地址。我们来看看`gateway`核心函数的实现：
>
> 复制代码
>
> ```
> const proxyFactory = require('./lib/proxy-factory')
> // 一个简易的高性能 Node.js 框架
> const restana = require('restana')
> // 默认的代理 handler
> const defaultProxyHandler = (req, res, url, proxy, proxyOpts) => proxy(req, res, url, proxyOpts)
> // 默认支持的方法，包括 ['get', 'delete', 'put', 'patch', 'post', 'head', 'options', 'trace']
> const DEFAULT_METHODS = require('restana/libs/methods').filter(method => method !== 'all')
> // 一个简易的 HTTP 响应库
> const send = require('@polka/send-type')
> // 支持 HTTP 代理
> const PROXY_TYPES = ['http']
> const gateway = (opts) => {
>   opts = Object.assign({
>     middlewares: [],
>     pathRegex: '/*'
>   }, opts)
> 	// 运行开发者传一个 server 实例，默认则使用 restana server
>   const server = opts.server || restana(opts.restana)
>   // 注册中间件
>   opts.middlewares.forEach(middleware => {
>     server.use(middleware)
>   })
>   // 一个简易的接口 `/services.json`，该接口罗列出网关代理的所有请求和相应信息
>   const services = opts.routes.map(route => ({
>     prefix: route.prefix,
>     docs: route.docs
>   }))
>   server.get('/services.json', (req, res) => {
>     send(res, 200, services)
>   })
>   // 路由处理
>   opts.routes.forEach(route => {
>     if (undefined === route.prefixRewrite) {
>       route.prefixRewrite = ''
>     }
>     const { proxyType = 'http' } = route
>     if (!PROXY_TYPES.includes(proxyType)) {
>       throw new Error('Unsupported proxy type, expecting one of ' + PROXY_TYPES.toString())
>     }
>     // 加载默认的 Hooks
>     const { onRequestNoOp, onResponse } = require('./lib/default-hooks')[proxyType]
>     // 加载自定义的 Hooks，允许开发者拦截并响应自己的 Hooks
>     route.hooks = route.hooks || {}
>     route.hooks.onRequest = route.hooks.onRequest || onRequestNoOp
>     route.hooks.onResponse = route.hooks.onResponse || onResponse
>     // 加载中间件，允许开发者自己传入自定义中间件
>     route.middlewares = route.middlewares || []
>     // 支持正则形式的 route path
>     route.pathRegex = undefined === route.pathRegex ? opts.pathRegex : String(route.pathRegex)
>     // 使用 proxyFactory 创建一个 proxy 实例
>     const proxy = proxyFactory({ opts, route, proxyType })
>     // 允许开发者自定义传入一个 proxyHandler，否则使用默认的 defaultProxyHandler
>     const proxyHandler = route.proxyHandler || defaultProxyHandler
>     // 设置超时时间
>     route.timeout = route.timeout || opts.timeout
>     const methods = route.methods || DEFAULT_METHODS
>     const args = [
>       // path
>       route.prefix + route.pathRegex,
>       // route middlewares
>       ...route.middlewares,
>       // 相关 handler 函数
>       handler(route, proxy, proxyHandler)
>     ]
>     methods.forEach(method => {
>       method = method.toLowerCase()
>       if (server[method]) {
>         server[method].apply(server, args)
>       }
>     })
>   })
>   return server
> }
> const handler = (route, proxy, proxyHandler) => async (req, res, next) => {
>   try {
>     // 支持 urlRewrite 配置
>     req.url = route.urlRewrite
>       ? route.urlRewrite(req)
>       : req.url.replace(route.prefix, route.prefixRewrite)
>     const shouldAbortProxy = await route.hooks.onRequest(req, res)
>     // 如果 onRequest hooks 返回一个 falsy 值，则执行 proxyHandler，否则停止代理
>     if (!shouldAbortProxy) {
>       const proxyOpts = Object.assign({
>         request: {
>           timeout: req.timeout || route.timeout
>         },
>         queryString: req.query
>       }, route.hooks)
>       proxyHandler(req, res, req.url, proxy, proxyOpts)
>     }
>   } catch (err) {
>     return next(err)
>   }
> }
> module.exports = gateway
> ```
>
> 上述代码主要流程并不复杂，我已经加入了相应的注释。`gateway`函数是整个网关的入口，包含了所有核心流程。这里我们对`proxyFactory`函数进行简单梳理：
>
> 复制代码
>
> ```
> const fastProxy = require('fast-proxy')
> module.exports = ({ proxyType, opts, route }) => {
>   let proxy = fastProxy({
>       base: opts.targetOverride || route.target,
>       http2: !!route.http2,
>       ...(route.fastProxy)
>     }).proxy
>   return proxy
> }
> ```
>
> 如上代码所示，我们使用了`fast-proxy`库，并支持开发者以`fastProxy`字段进行对`fast-proxy`库的配置。具体配置信息你可以参考`fast-proxy`库，这里我们不再展开。
>
> 其实通过以上代码分析，我们已经把大体流程梳理了一遍。但是上述代码只实现了基础的代理功能，只是网关的一部分能力。接下来，我们从网关扩展层面，继续了解网关的设计和实现。
>
> #### 中间件
>
> 中间件化思想已经渗透到前端编程理念中，开发者颇为受益。中间件能够帮助我们在解耦合的基础上，实现能力扩展。
>
> 我们来看看这个网关的中间件能力，如下代码：
>
> 复制代码
>
> ```
> const rateLimit = require('express-rate-limit')
> const requestIp = require('request-ip')
> gateway({
>   // 定义一个全局中间件
>   middlewares: [
>     // 记录访问 IP
>     (req, res, next) => {
>       req.ip = requestIp.getClientIp(req)
>       return next()
>     },
>     // 使用 RateLimit 模块
>     rateLimit({
>       // 1 分钟窗口期
>       windowMs: 1 * 60 * 1000, // 1 minutes
>       // 在窗口期内，同一个 IP 只允许访问 60 次
>       max: 60,
>       handler: (req, res) => res.send('Too many requests, please try again later.', 429)
>     })
>   ],
>   // downstream 服务代理
>   routes: [{
>     prefix: '/public',
>     target: 'http://localhost:3000'
>   }, {
>     // ...
>   }]
> })
> ```
>
> 上面代码中，我们实现了两个中间件。第一个中间通过[request-ip](https://www.npmjs.com/package/request-ip?fileGuid=xxQTRXtVcqtHK6j8)这个库获取访问的真实 IP 地址，并将 IP 值挂载在 req 对象上。第二个中间件通过[express-rate-limit](https://www.npmjs.com/package/express-rate-limit?fileGuid=xxQTRXtVcqtHK6j8)进行“在窗口期内，同一个 IP 只允许访问 60 次”的限流策略。因为[express-rate-limit](https://www.npmjs.com/package/express-rate-limit?fileGuid=xxQTRXtVcqtHK6j8)库默认使用`req.ip`作为[keyGenerator](https://github.com/nfriedly/express-rate-limit/blob/master/lib/express-rate-limit.js#L16?fileGuid=xxQTRXtVcqtHK6j8)，所以我们的第一个中间件将 IP 记录在了`req.ip`上面。
>
> 这是一个简单的运用中间件实现限流的案例，开发者可以通过自己动手实现，或依赖其他库实现相关策略。
>
> #### 缓存策略
>
> 缓存能够有效提升网关对于请求的处理能力和吞吐量。我们的网关设计支持了多种缓存方案，如下代码是一个使用 Node 内存缓存的案例：
>
> 复制代码
>
> ```
> // 使用 http-cache-middleware 作为缓存中间件
> const cache = require('http-cache-middleware')()
> // enable http cache middleware
> const gateway = require('fast-gateway')
> const server = gateway({
>   middlewares: [cache],
>   routes: [...]
> })
> ```
>
> 如果不担心缓存数据的丢失，即缓存数据不需要持久化，且只有一个网关实例，使用内存缓存是一个很好的选择。
>
> 当然，也支持使用 Redis 进行缓存，如下代码：
>
> 复制代码
>
> ```
> // 初始化 Redis
> const CacheManager = require('cache-manager')
> const redisStore = require('cache-manager-ioredis')
> const redisCache = CacheManager.caching({
>   store: redisStore,
>   db: 0,
>   host: 'localhost',
>   port: 6379,
>   ttl: 30
> })
> // 缓存中间件
> const cache = require('http-cache-middleware')({
>   stores: [redisCache]
> })
> const gateway = require('fast-gateway')
> const server = gateway({
>   middlewares: [cache],
>   routes: [...]
> })
> ```
>
> 在网关的设计中，我们依赖了[http-cache-middleware](https://github.com/jkyberneees/http-cache-middleware?fileGuid=xxQTRXtVcqtHK6j8)库作为缓存，参考其[源码](http://http-cache-middleware?fileGuid=xxQTRXtVcqtHK6j8)，我们可以看到缓存使用了`req.method + req.url + cacheAppendKey`作为缓存的 key，`cacheAppendKey`出自`req`对象，因此开发者可以通过设置`req.cacheAppendKey = (req) => req.user.id`的方式，自定义缓存 key。
>
> 当然，我们可以对某个接口 Endpoint 禁用缓存，这也是通过中间件实现的：
>
> 复制代码
>
> ```
> routes: [{
>   prefix: '/users',
>   target: 'http://localhost:3000',
>   middlewares: [(req, res, next) => {
>     req.cacheDisabled = true
>     return next()
>   }]
> }]
> ```
>
> #### Hooks 设计
>
> 有了中间件还不够，我们还可以以 Hooks 的方式，允许开发者介入网关处理流程。比如以下代码：
>
> 复制代码
>
> ```
> const { multipleHooks } = require('fg-multiple-hooks')
> const hook1 = async (req, res) => {
>   console.log('hook1 with logic 1 called')
>   // 返回 falsy 值，不会阻断请求处理流程
>   return false
> }
> const hook2 = async (req, res) => {
>   console.log('hook2 with logic 2 called')
>   const shouldAbort = true
>   if (shouldAbort) {
>     res.send('handle a rejected request here')
>   }
>   // 返回 true，则终端处理流程
>   return shouldAbort
> }
> gateway({
>   routes: [{
>     prefix: '/service',
>     target: 'http://127.0.0.1:3000',
>     hooks: {
>     	// 使用多个 Hooks 函数，处理 onRequest
>       onRequest: (req, res) => multipleHooks(req, res, hook1, hook2), 
>       rewriteHeaders (handlers) {
>       	// 可以在这里设置 response header
>       	return headers
>       }
>       // 使用多个 Hooks 函数，处理 onResponse
>       onResponse (req, res, stream) {
>         
>       }
>     }
>   }]
> }).start(PORT).then(server => {
>   console.log(`API Gateway listening on ${PORT} port!`)
> })
> ```
>
> 对应源码处理相应的 Hooks 流程已经在前面部分有所涉及，这里不再一一展开。
>
> 最后，我们再通过一个实现负载均衡的场景，来加强对该网关的设计理解，如下代码：
>
> 复制代码
>
> ```
> const gateway = require('../index')
> const { P2cBalancer } = require('load-balancers')
> const targets = [
>   'http://localhost:3000',
>   'xxxxx',
>   'xxxxxxx'
> ]
> const balancer = new P2cBalancer(targets.length)
> gateway({
>   routes: [{
>     // 自定义 proxyHandler
>     proxyHandler: (req, res, url, proxy, proxyOpts) => {
>       // 使用 P2cBalancer 实例进行负载均衡
>       const target = targets[balancer.pick()]
>       if (typeof target === 'string') {
>         proxyOpts.base = target
>       } else {
>         proxyOpts.onResponse = onResponse
>         proxyOpts.onRequest = onRequestNoOp
>         proxy = target
>       }
>       return proxy(req, res, url, proxyOpts)
>     },
>     prefix: '/balanced'
>   }]
> })
> ```
>
> 通过如上代码我们看出，网关的设计既支持默认的 proxyHandler，又支持开发者自定义的 proxyHandler，对于自定义的 proxyHandler，网关层面提供：
>
> - req
> - res
> - req.url
> - proxy
> - proxyOpts
>
> 相关参数，方便开发者发挥。
>
> 至此，我们就从源码和设计层面对一个成熟的网关实现进行了解析。你可以结合源码进行学习。
>
> ### 总结
>
> 这一讲我们深入讲解了 Node.js 另外一个重要的应用场景：企业级 BFF 网关。我们详细介绍了 BFF 网关的优缺点、打造 BFF 网关需要考虑的问题。总之，设计一个良好的 BFF 网关，要求开发者具有较强的综合能力。接下来我们实现一个精简的网关系统，并结合源码和设计层面对其实现进行了解析，帮助你深入了解网关的构建。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image6/M00/21/5C/CioPOWBURoyAGqyFAAdwJg_hSMw100.png)
>
> 事实上，BFF 网关理念已经完全被业界接受，业界著名的网关包括但不限于：
>
> - Netflix API Gateway: Zuul
> - Amazon AWS 网关
> - Kong Gateway
> - SwaggerHub
> - Express API Gateway
> - Azure API Gateway
>
> 作为前端开发者，向 BFF 进军是一个有趣且必要的发展方向。
>
> 另外，Serverless 是一种无服务器架构，它的弹性伸缩、按需使用、无运维等特性都是未来的发展方向。而 Serverless 结合 BFF 网关设计理念，业界也推出了 SFF（Serverless For Frontend）的概念。
>
> 其实，这些概念万变不离其宗，掌握了 BFF 网关，能够设计一个高可用的网关层，会让你在技术上收获颇多，同时也能为业务带来更大的收益。
>
> 下一讲，我们就迎来了课程的最后内容——实现高可用：使用 Puppeteer 生成性能最优的海报系统。我们将介绍 Puppeteer 的各种应用场景，并重点讲解基于 Puppeteer 设计实现的海报服务系统。下节内容同样精彩，请你继续阅读。