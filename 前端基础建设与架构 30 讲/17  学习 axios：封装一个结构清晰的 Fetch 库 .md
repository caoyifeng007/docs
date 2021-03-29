[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5922)



> 从这一讲开始，我们将进入核心框架原理与代码设计模式的学习。任何一个动态应用的实现，都离不开前后端的互动配合。前端发送请求获取数据是开发者必不可少的场景。正因为如此，每一个前端项目都有必要接入一个请求库。
>
> 那么请求库如何设计，才能保证使用者的顺畅？请求逻辑如何抽象成统一请求库，才能避免出现代码混乱堆积，难以维护的现象呢？下面我们就进入正题。
>
> ### 一个请求库需要考虑哪些问题
>
> 一个请求，纵向向前承载了数据的发送，向后链接了数据的接收和消费，横向还需要处理网络环境和宿主能力，以及业务的扩展需求。因此设计一个好的请求库，首先需要预见可能会发生的问题。下面我们将重点展开几个关键问题。
>
> #### 适配浏览器 or Node.js 环境
>
> 如今，前端开发不再局限于浏览器层面，Node.js 环境的出现，使得请求库的适配需求变得更加复杂。**Node.js 基于 V8 JavaScript Engine，顶层对象是 global，不存在 Window 对象和浏览器宿主**，因此使用传统的 XMLHttpRequest/Fetch 在 Node.js 上发送请求是行不通的。对于搭建了 Node.js 环境的前端来说，请求库的设计实现需要考虑是否同时支持在浏览器和 Node.js 两种环境发送请求。在同构的背景下，如何使不同环境的请求库使用体验趋于一致呢？下面我们将会对这部分内容进一步讲解。
>
> #### XHR or Fetch
>
> 单就浏览器环境发送请求来说，一般存在两种技术方法：
>
> - [XMLHttpRequest 规范](https://xhr.spec.whatwg.org/)
> - [Fetch 规范](https://fetch.spec.whatwg.org/)
>
> 我们先简要对比两种技术的使用方式。
>
> 使用 XMLHttpRequest 发送请求：
>
> 复制代码
>
> ```
> function success() {
>     var data = JSON.parse(this.responseText);
>     console.log(data);
> }
> function error(err) {
>     console.log('Error Occurred :', err);
> }
> var xhr = new XMLHttpRequest();
> xhr.onload = success;
> xhr.onerror = error;
> xhr.open('GET', 'https://xxx');
> xhr.send();
> ```
>
> 简单来说，XMLHttpRequest 存在一些缺点，比如：
>
> - 配置和使用方式较为烦琐；
> - 基于事件的异步模型不够友好。
>
> 而 Fetch 的推出，主要也是为了解决上述问题。
>
> 使用 Fetch 发送一个请求：
>
> 复制代码
>
> ```
> fetch('https://xxx')
>     .then(function (response) {
>         console.log(response);
>     })
>     .catch(function (err) {
>         console.log("Something went wrong!", err);
>     });
> ```
>
> 我们可以看到，Fetch 基于 Promise，**语法更加简洁，语义化更加突出**，但**兼容性不如 XMLHttpRequest**。
>
> 对于一个请求库来说，在浏览器端使用 XMLHttpRequest 还是 Fetch？这是一个问题。下面我们通过 axios 的实现具体展开讲解。
>
> #### 功能设计与抽象粒度
>
> 无论是基于 XMLHttpRequest 还是 Fetch，实现一层封装，屏蔽一些基础能力并暴露给业务方使用，即实现一个请求库，这并不困难。我认为，真正难的是**请求库的功能设计和抽象粒度**。如果功能设计分层不够清晰，抽象方式不够灵活，很容易产出“屎山代码”。
>
> 比如，对于请求库来说，是否要处理以下看似通用，但又具有定制性的功能呢？你需要考虑以下功能点：
>
> - 自定义 headers 添加
> - 统一断网/弱网处理
> - 接口缓存处理
> - 接口统一错误提示
> - 接口统一数据处理
> - 统一数据层结合
> - 统一请求埋点
>
> 这些设计问题如果初期不考虑清楚，那么在业务层面，一旦真正使用了设计不良的请求库，很容易遇到不满足业务需求的场景，而沦为手写 Fetch，势必导致代码库中请求方式多种多样，风格不一。
>
> 这里我们稍微展开，以一个请求库的分层封装为例，其实任何一种通用能力的封装都可以参考下图：
>
> ![202125-101326.png](https://s0.lgstatic.com/i/image6/M01/02/45/Cgp9HWAdGnOAEUmTAADzvClHn5k247.png)
>
> 请求库分层封装示例图
>
> 如图所示，底层能力部分，对应请求库中宿主提供的 XMLHttpRequest 或 Fetch 能力，以及项目中已经内置的框架/类库能力。这一部分对于一个已有项目来说，往往是较难改变或重构的，也是不同项目中可以复用的；而业务层，比如依赖 axios 请求库的更上层封装，我们一般可以分为：
>
> - 项目层
> - 页面层
> - 组件层
>
> 三个方面，它们依次递进，完成最终业务消费。底层能力部分，对许多项目来说都可以使用，而**让不同项目之间的代码质量和开发效率产生差异**的，恰好是容易被轻视的业务级别的封装设计。
>
> 比如设计者在项目层的封装上，如果做了几乎所有事情，囊括了所有请求相关的规则，很容易使封装复杂，过度设计。不同层级的功能和职责是不同的，**错位的使用和设计，是让项目变得更加混乱的诱因之一**。
>
> 合理的设计是，底层部分保留对全局封装的影响范围，而项目层保留对页面层的影响能力，页面层保留对组件层的影响能力。
>
> ![前端基建 金句.png](https://s0.lgstatic.com/i/image6/M00/02/E5/CioPOWAeNr6AYnHYAAVEoQwNcDk580.png)
>
> 比如，我们在项目层提供一个基础请求库封装，在这一层可以提供默认发送 cookie 等（一定需要存在）的行为，同时通过配置 options.fetch 保留覆盖 globalThis.fetch 的能力，这样可以在 Node 等环境中，通过**注入一个 node-fetch npm 库**的方式，支持 SSR 能力。
>
> 这里需要注意的是，我们一定要避免设计一个特别大的 Fetch 方法，通过拓展 options 把所有事情都做了，用 options 驱动一切行为，这比较容易让 Fetch 代码和逻辑变得复杂、难以理解，而且不利于 tree-shaking 和 code-spliting。
>
> 那么如何做到这种层次清晰的基础库呢？接下来，我们就从 axios 的设计分析寻找答案。
>
> ### axios 设计之美
>
> axios 是一个被前端广泛使用的请求库，对应上述分层结构中，属于框架/类库层，我们来总结一下它的功能特点：
>
> - 在浏览器端，使用 XMLHttpRequest 发送请求；
> - 支持 Node.js 端发送请求；
> - 支持 Promise API，使用 Promise 风格语法；
> - 支持请求和响应拦截；
> - 支持自定义修改请求和返回内容；
> - 支持请求取消；
> - 默认支持 XSRF 防御。
>
> 下面，我们主要从拦截器思想、适配器思想、安全思想三方面展开，分析 axios 设计的可取之处。
>
> #### 拦截器思想
>
> 拦截器思想是 axios 带来的最具启发性的思想之一。它赋予了**分层开发时借助拦截行为，注入自定义能力的功能**。简单来说，axios 的拦截器主要由：任务注册 → 任务编排 → 任务调度（执行）三步组成。
>
> 我们先看任务注册，在请求发出前，可以使用`axios.interceptors.request.use`方法注入拦截逻辑，比如：
>
> 复制代码
>
> ```
> axios.interceptors.request.use(function (config) {
>     // 请求发送前做一些事情，比如添加 headers
>     return config;
>   }, function (error) {
>     // 请求出现错误时，处理逻辑
>     return Promise.reject(error);
>   });
> ```
>
> 在请求返回后，用`axios.interceptors.response.use`方法注入拦截逻辑，比如：
>
> 复制代码
>
> ```
> axios.interceptors.response.use(function (response) {
>     // 响应返回 2xx 时，做一些操作，比如响应状态码为 401 时，自动跳转到登录页
>     return response;
>   }, function (error) {
>     // 响应返回 2xx 外响应码时，错误处理逻辑
>     return Promise.reject(error);
>   });
> ```
>
> 任务注册部分的源码实现也不复杂：
>
> 复制代码
>
> ```
> // lib/core/Axios.js
> function Axios(instanceConfig) {
>   this.defaults = instanceConfig;
>   this.interceptors = {
>     request: new InterceptorManager(),
>     response: new InterceptorManager()
>   };
> }
> // lib/core/InterceptorManager.js
> function InterceptorManager() {
>   this.handlers = [];
> }
> InterceptorManager.prototype.use = function use(fulfilled, rejected) {
>   this.handlers.push({
>     fulfilled: fulfilled,
>     rejected: rejected
>   });
>   // 返回当前的索引，用于移除已注册的拦截器
>   return this.handlers.length - 1;
> };
> ```
>
> 如上代码，我们定义的请求/响应拦截器，会在每一个 axios 实例的 Interceptors 属性中维护，`this.interceptors.request`和`this.interceptors.response`也都是一个 InterceptorManager 实例，该实例的`handlers`属性**以数组的形式**存储了使用方定义的一个个拦截器逻辑。
>
> 注册了任务，我们再来看看任务编排时是如何将拦截器串联起来，并在任务调度阶段执行各个拦截器的。如下源码：
>
> 复制代码
>
> ```
> // lib/core/Axios.js
> Axios.prototype.request = function request(config) {
>   config = mergeConfig(this.defaults, config);
>   // ...
>   var chain = [dispatchRequest, undefined];
>   var promise = Promise.resolve(config);
>   // 任务编排
>   this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
>     chain.unshift(interceptor.fulfilled, interceptor.rejected);
>   });
>   this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
>     chain.push(interceptor.fulfilled, interceptor.rejected);
>   });
>   // 任务调度
>   while (chain.length) {
>     promise = promise.then(chain.shift(), chain.shift());
>   }
>   return promise;
> };
> ```
>
> 我们通过`chain`数组来编排调度任务，`dispatchRequest`方法实际执行请求的发送，编排过程实现：**在实际发送请求的方法**`dispatchRequest`前插入请求拦截器，在`dispatchRequest`方法后，插入响应拦截器。
>
> 任务调度其实就是通过一个 While 循环，通过一个 Promise 实例，遍历迭代`chain`数组方法，并基于 Promise 回调特性，将各个拦截器串联执行起来。
>
> 我们通过下图，来加深理解：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/94/9E/CgqCHmAY6WuAA9A_AAEfckV_ZjM495.png)
>
> #### 适配器思想
>
> 前文提到了 axios 同时支持 Node.js 环境和浏览器环境发送请求，在浏览器中我们可以选用 XMLHttpRequest 或 Fetch 方法发送请求，但是在 Node.js 中，需要通过 HTTP 模块发送请求。对此，axiso 是如何设计实现的呢？
>
> 为了支持适配不同环境，axios 实现了适配器：Adapter，具体实现在`dispatchRequest`方法中：
>
> 复制代码
>
> ```
> // lib/core/dispatchRequest.js
> module.exports = function dispatchRequest(config) {
>   // ...
>   var adapter = config.adapter || defaults.adapter;
> 
>   return adapter(config).then(function onAdapterResolution(response) {
>     // ...
>     return response;
>   }, function onAdapterRejection(reason) {
>     // ...
>     return Promise.reject(reason);
>   });
> };
> ```
>
> 如上代码，axios 支持使用方实现自己的 Adapter，自定义不同环境中的请求实现方式，也提供了默认的 Adapter。默认 Adapter 逻辑代码如下：
>
> 复制代码
>
> ```
> function getDefaultAdapter() {
>   var adapter;
>   if (typeof XMLHttpRequest !== 'undefined') {
>     // 浏览器端使用 XMLHttpRequest 方法
>     adapter = require('./adapters/xhr');
>   } else if (typeof process !== 'undefined' && 
>     Object.prototype.toString.call(process) === '[object process]') {
>     // Node.js 端，使用 HTTP 模块
>     adapter = require('./adapters/http');
>   }
>   return adapter;
> }
> ```
>
> 一个 Adapter 需要返回一个 Promise 实例（这是因为**axios 内部通过 Promise 链式调用完成请求调度**），我们分别看看在浏览器端和 Node.js 端具体 Adapter 实现逻辑：
>
> 复制代码
>
> ```
> module.exports = function xhrAdapter(config) {
>   return new Promise(function dispatchXhrRequest(resolve, reject) {
>     var requestData = config.data;
>     var requestHeaders = config.headers;
>     var request = new XMLHttpRequest();
>     var fullPath = buildFullPath(config.baseURL, config.url);
>     request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);
> 
>     // Listen for ready state
>     request.onreadystatechange = function handleLoad() {
>     	// ....
>     };
>     // Handle browser request cancellation (as opposed to a manual cancellation)
>     request.onabort = function handleAbort() {
>       // ...
>     };
>     // Handle low level network errors
>     request.onerror = function handleError() {
>       // ...
>     };
>     // Handle timeout
>     request.ontimeout = function handleTimeout() {
>       // ...
>     };
>     // ...
> 
>     request.send(requestData);
>   });
> };
> ```
>
> 如上代码，就是一个典型的使用 XMLHttpRequest 发送请求的实现内容。在 Node.js 端的实现，精简后代码如下：
>
> 复制代码
>
> ```
> var http = require('http');
> /*eslint consistent-return:0*/
> module.exports = function httpAdapter(config) {
>   return new Promise(function dispatchHttpRequest(resolvePromise, rejectPromise) {
>     var resolve = function resolve(value) {
>       resolvePromise(value);
>     };
>     var reject = function reject(value) {
>       rejectPromise(value);
>     };
>     var data = config.data;
>     var headers = config.headers;
>     var options = {
>       // ...
>     };
> 
>     var transport = http;
>     var req = http.request(options, function handleResponse(res) {
>       // ...
>     });
>     // Handle errors
>     req.on('error', function handleRequestError(err) {
>       // ...
>     });
>     // Send the request
>     if (utils.isStream(data)) {
>       data.on('error', function handleStreamError(err) {
>         reject(enhanceError(err, config, null, req));
>       }).pipe(req);
>     } else {
>       req.end(data);
>     }
>   });
> };
> ```
>
> 上述代码主要是调用 Node.js HTTP 模块，进行请求的发送和处理，当然，真实源码实现还需要考虑 HTTPS 以及 Redirect 等问题，这里我们不再展开。
>
> 讲到这里，可能你会问，什么场景下，才会需要自定义 Adapter 进行请求发送呢？比如**在测试阶段或特殊环境**中，我们可以 mock 请求：
>
> 复制代码
>
> ```
> if (isEnv === 'ui-test') {
> 	adapter = require('axios-mock-adapter')
> }
> ```
>
> 实现一个自定义的 Adapter 也并不困难，说到底它也只是一个 Node.js 模块，导出一个 Promise 实例即可：
>
> 复制代码
>
> ```
> module.exports = function myAdapter(config) {
>   // ...
>   return new Promise(function(resolve, reject) {
>     // ...
>     sendRequest(resolve, reject, response);
>     // ....
>   });
> }
> ```
>
> 相信你学会了这些内容，就对 [axios-mock-adapter](https://github.com/ctimmerm/axios-mock-adapter) 这个库的实现原理了然于胸了。
>
> #### 安全思想
>
> 说到请求，自然关联着安全问题。在本小节最后部分，我们对 axios 中的一些安全机制进行解析，涉及相关攻击手段：CSRF。
>
> > Cross—Site Request Forgery，攻击者盗用了你的身份，以你的名义发送恶意请求，对服务器来说，这个请求是完全合法的，但是却完成了攻击者期望的一个操作，比如以你的名义发送邮件、发消息，盗取你的账号，添加系统管理员，甚至购买商品、虚拟货币转账等。
>
> 在 axios 中，主要依赖双重 cookie 的方式防御 CSRF。具体来说，对于攻击者，获取用户 cookie 是比较困难的，因此，**我们可以在请求中携带一个 cookie 值，来保证请求的安全性**。这里我们将相关流程梳理为：
>
> - 用户访问页面，后端向请求域中注入一个 cookie，一般该 cookie 值为加密随机字符串；
> - 在前端通过 Ajax 请求数据时，取出上述 cookie，添加到 URL 参数或者请求 header 中；
> - 后端接口验证请求中携带的 cookie 值是否合法，不合法（不一致），则拒绝请求。
>
> 我们看 axios 源码：
>
> 复制代码
>
> ```
> // lib/defaults.js
> var defaults = {
>   adapter: getDefaultAdapter(),
>   // ...
>   xsrfCookieName: 'XSRF-TOKEN',
>   xsrfHeaderName: 'X-XSRF-TOKEN',
> };
> ```
>
> 在这里，axios 默认配置了默认`xsrfCookieName`和`xsrfHeaderName`，实际开发中可以按具体情况传入配置。在具体请求时，以`lib/adapters/xhr.js`为例：
>
> 复制代码
>
> ```
> // 添加 xsrf header
> if (utils.isStandardBrowserEnv()) {
>   var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ?
>     cookies.read(config.xsrfCookieName) :
>     undefined;
>   if (xsrfValue) {
>     requestHeaders[config.xsrfHeaderName] = xsrfValue;
>   }
> }
> ```
>
> 由此可见，对一个成熟请求库的设计来说，安全防范这个话题永不过时。
>
> ### 总结
>
> 本讲我们在开篇分析了代码设计、代码分层的方方面面，一个好的设计一定是层次明晰，各司其职的，一个好的设计也会直接帮助业务开发提升效率。封装和设计，是编程领域亘古不变的经典话题，需要每名开发者下沉到业务开发中体会、思考。
>
> 本小节的后半部分，我们从源码入手，分析了 axios 的优秀设计思想。即便你在业务中没有使用过 axios，但对于 axios 的学习始终是必要且重要的。
>
> 主要内容总结如下：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image2/M01/0C/8A/Cip5yGAY6X6ASgWfAAEY6D_f_OM734.png)
>
> 最后，给大家布置一个思考题：axios 支持[请求取消能力](https://github.com/axios/axios#cancellation)，这是如何实现的呢？欢迎在留言区和我分享你的观点。下一讲，我们将继续学习代码设计这一话题，通过对比 Koa 和 Redux，聚焦中间件化和插件化理念。我们下一讲再见。