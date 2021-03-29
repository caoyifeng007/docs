[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5923)



> 上一讲，我们通过分析 axios 源码，延伸了“如何设计一个请求公共库”，其中提到了不同层次级别的分层理念。这一讲，我们继续讨论代码设计这一话题，聚焦中间件化和插件化理念。并通过实现一个中间件化的请求库和上一节内容融会贯通。
>
> ### 以 Koa 为代表的 Node.js 中间件化设计
>
> 说到中间件，很多开发者都会想到 Koa.js，其中间件设计无疑是前端中间件思想的典型代表之一。我们先来剖析 Koa.js 的设计和实现。
>
> 先来看一下 Koa.js 中间件的实现和应用：
>
> 复制代码
>
> ```
> // 最外层中间件，可以用于兜底 Koa 全局错误
> app.use(async (ctx, next) => {
>   try {
>     // console.log('中间件 1 开始执行')
>     // 执行下一个中间件
>     await next();
>     // console.log('中间件 1 执行结束')
>   } catch (error) {
>     console.log(`[koa error]: ${error.message}`)
>   }
> });
> // 第二层中间件，可以用于日志记录
> app.use(async (ctx, next) => {
>   // console.log('中间件 2 开始执行')
>   const { req } = ctx;
>   console.log(`req is ${JSON.stringify(req)}`);
>   await next();
>   console.log(`res is ${JSON.stringify(ctx.res)}`);
>   // console.log('中间件 2 执行结束')
> });
> ```
>
> 如上代码，我们看 Koa 实例，通过`use`方法注册和串联中间件，其源码实现部分精简表述为：
>
> 复制代码
>
> ```
> use(fn) {
>     this.middleware.push(fn);
>     return this;
> }
> ```
>
> 如上代码，我们的中间件被存储进`this.middleware`数组中，那么中间件是如何被执行的呢？参考下面源码：
>
> 复制代码
>
> ```
> // 通过 createServer 方法启动一个 Node.js 服务
> listen(...args) {
>     const server = http.createServer(this.callback());
>     return server.listen(...args);
> }
> ```
>
> Koa 框架通过 `http` 模块的 `createServer` 方法创建一个 Node.js 服务，并传入 this.callback() 方法， this.callback() 方法源码精简实现如下：
>
> 复制代码
>
> ```
> callback() {
>     // 从 this.middleware 数组中，组合中间件
>     const fn = compose(this.middleware);
> 
>     // handleRequest 方法作为 `http` 模块的 `createServer` 方法参数，该方法通过 `createContext` 封装了 `http.createServer` 中的 `request` 和 `response`对象，并将这两个对象放到 ctx 中
>     const handleRequest = (req, res) => {
>         const ctx = this.createContext(req, res);
>         // 将 ctx 和组合后的中间件函数 fn 传递给 this.handleRequest 方法
>         return this.handleRequest(ctx, fn);
>     };
> 
>     return handleRequest;
> }
> handleRequest(ctx, fnMiddleware) {
>     const res = ctx.res;
>     res.statusCode = 404;
>     const onerror = err => ctx.onerror(err);
>     const handleResponse = () => respond(ctx);
>     // on-finished npm 包提供的方法，该方法在一个 HTTP 请求 closes，finishes 或者 errors 时执行
>     onFinished(res, onerror);
>     // 将 ctx 对象传递给中间件函数 fnMiddleware
>     return fnMiddleware(ctx).then(handleResponse).catch(onerror);
> }
> ```
>
> 如上代码，我们将 Koa 一个中间件组合和执行流程梳理为以下步骤。
>
> 1. 通过`compose`方法组合各种中间件，返回一个中间件组合函数`fnMiddleware`
> 2. 请求过来时，会先调用`handleRequest`方法，该方法完成：
>    - 调用`createContext`方法，对该次请求封装出一个`ctx`对象；
>    - 接着调用`this.handleRequest(ctx, fnMiddleware)`处理该次请求。
> 3. 通过`fnMiddleware(ctx).then(handleResponse).catch(onerror)`执行中间件。
>
> 其中，一个核心过程就是使用`compose`方法组合各种中间件，其源码实现精简为：
>
> 复制代码
>
> ```
> function compose(middleware) {
>     // 这里返回的函数，就是上文中的 fnMiddleware
>     return function (context, next) {
>         let index = -1
>         return dispatch(0)
> 
>         function dispatch(i) {
>         	  // 
>             if (i <= index) return Promise.reject(new Error('next() called multiple times'))
>             index = i
>             // 取出第 i 个中间件为 fn
>             let fn = middleware[i]
> 
>             if (i === middleware.length) fn = next
> 
>             // 已经取到了最后一个中间件，直接返回一个 Promise 实例，进行串联
>             // 这一步的意义是保证最后一个中间件调用 next 方法时，也不会报错
>             if (!fn) return Promise.resolve()
> 
>             try {
>                 // 把 ctx 和 next 方法传入到中间件 fn 中，并将执行结果使用 Promise.resolve 包装
>                 // 这里可以发现，我们在一个中间件中调用的 next 方法，其实就是dispatch.bind(null, i + 1)，即调用下一个中间件
>                 return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
>             } catch (err) {
>                 return Promise.reject(err)
>             }
>         }
>     }
> }
> ```
>
> 源码实现中我已加入了相关注释，如果对于你来说还是晦涩难懂，不妨看一下下面这个 hard coding 的例子，通过下面代码，表示三个 Koa 中间件的执行情况：
>
> 复制代码
>
> ```
> async function middleware1() {
>   ...
>   await (async function middleware2() {
>     ...
>     await (async function middleware3() {
>       ...
>     });
>     ...
>   });
>   ...
> }
> ```
>
> 这里我们来做一个简单的总结：
>
> - Koa 的中间件机制被社区形象地总结为洋葱模型；
>
> > 所谓洋葱模型，就是指每一个 Koa 中间件都是一层洋葱圈，它即可以掌管请求进入，也可以掌管响应返回。换句话说：外层的中间件可以影响内层的请求和响应阶段，内层的中间件只能影响外层的响应阶段。
>
> - `dispatch(n)`对应第 n 个中间件的执行，第 n 个中间件可以通过`await next()`来执行下一个中间件，同时在最后一个中间件执行完成后，依然有恢复执行的能力。即，通过洋葱模型，`await next()`控制调用 “下游”中间件，直到 “下游”没有中间件且堆栈执行完毕，最终流回“上游”中间件。这种方式有个优点，特别是**对于日志记录以及错误处理等需要非常友好**。
>
> 这里我们稍微做一下扩展，引申出 Koa v1 版本中中间件的实现，Koa1 的中间件实现利用了 **Generator 函数 + co 库**（一种基于 Promise 的 Generator 函数流程管理工具），来实现协程运行。本质上，Koa v1 版本中间件和 Koa v2 版本中间件思想是类似的，只不过 Koa v2 主要是用了 **Async/Await** 来替换 Generator 函数 + co 库，整体实现更加巧妙，代码更加优雅、简易。
>
> ### 对比 Express，再谈 Koa 中间件
>
> 说起 Node.js 框架，我们自然忘不了 Express.js。它的中间件机制同样值得我们学习、比对。Express 不同于 Koa，它继承了**路由、静态服务器和模板引擎等功能**，因此看上去比 Koa 更像是一个框架。通过学习 [Express 源码](https://github.com/expressjs/express)，我们可以总结出它的工作机制。
>
> 1. 通过`app.use`方法注册中间件。
> 2. 一个中间件可以理解为一个 Layer 对象，其中包含了当前路由匹配的正则信息以及 handle 方法。
> 3. 所有中间件（Layer 对象）使用`stack`数组存储起来。
> 4. 因此，每个 Router 对象都是通过一个`stack`数组，存储了相关中间件函数。
> 5. 当一个请求过来时，会从 REQ 中获取请求 path，根据 path 从`stack`中找到匹配的 Layer，具体匹配过程由`router.handle`函数实现。
> 6. `router.handle`函数通过`next()`方法遍历每一个 layer 进行比对：
>    1. `next()`方法通过闭包维持了对于 Stack Index 游标的引用，当调用`next()`方法时，就会从下一个中间件开始查找；
>    2. 如果比对结果为 true，则调用`layer.handle_request`方法，`layer.handle_request`方法中会调用`next()`方法 ，实现中间件的执行。
>
> 我们将上述过程总结为下图，帮助你理解：
>
> ![202127-92025.png](https://s0.lgstatic.com/i/image6/M00/03/C9/Cgp9HWAfsPuAMXAzAAIE4xCY0WY258.png)
>
> Express 工作机制
>
> 通过上述内容，我们可以看到，Express 的`next()`方法维护了遍历中间件列表的 Index 游标，中间件每次调用`next()`方法时，会通过**增加 Index 游标的方式**找到下一个中间件并执行。我们采用类似的 hard coding 形式帮助大家理解 Express 插件作用机制：
>
> 复制代码
>
> ```
> ((req, res) => {
>   console.log('第一个中间件');
>   ((req, res) => {
>     console.log('第二个中间件');
>     (async(req, res) => {
>       console.log('第三个中间件 => 是一个 route 中间件，处理 /api/test1');
>       await sleep(2000)
>       res.status(200).send('hello')
>     })(req, res)
>     console.log('第二个中间件调用结束');
>   })(req, res)
>   console.log('第一个中间件调用结束')
> })(req, res)
> ```
>
> 如上代码，Express 中间件设计并不是一个洋葱模型，它是基于回调实现的线形模型，不利于组合，不利于互操，在设计上并不像 Koa 一样简单。如果想实现一个记录请求响应的中间件，就需要：
>
> 复制代码
>
> ```
> var express = require('express')
> var app = express()
> var requestTime = function (req, res, next) {
>   req.requestTime = Date.now()
>   next()
> }
> app.use(requestTime)
> app.get('/', function (req, res) {
>   var responseText = 'Hello World!<br>
> 
> '
>   responseText += '<small>Requested at: ' + req.requestTime + '</small>'
>   res.send(responseText)
> })
> app.listen(3000)
> ```
>
> 我们可以看到，上述实现就对业务代码有一定程度的侵扰，甚至会造成不同中间件间的耦合。
>
> 我们回退到“上帝视角”发现，毫无疑问 Koa 的洋葱模型更加先进，而**Express 的线形机制不容易实现拦截处理逻辑**：比如异常处理和统计响应时间——这在 Koa 里，一般只需要一个中间件就能全部搞定。
>
> 当然，Koa 本身只提供了 HTTP 模块和洋葱模型的最小封装，Express 是一种更高形式的抽象，其设计思路和面向目标也有不同。
>
> ### Redux 中间件设计和实现
>
> 通过前文，我们了解了 Node.js 两个当红框架的中间件设计，我们再换一个角度：从 Redux 这个状态管理方案的中间件设计，了解更全面的中间件系统。
>
> 类似 Koa 中的 koa-compose 实现，Redux 也实现了一个`compose`方法，完成中间件的注册和串联：
>
> 复制代码
>
> ```
> function compose(...funcs: Function[]) {
> 	return funcs.reduce((a, b) => (...args: any) => a(b(...args)));
> }
> ```
>
> `compose`方法的执行效果如下代码：
>
> 复制代码
>
> ```
> compose([fn1, fn2, fn3])(args)
> =>
> compose(fn1, fn2, fn3) (...args) = > fn1(fn2(fn3(...args)))
> ```
>
> 简单来说，`compose`方法是一种高阶聚合，先执行 fn3，并将执行结果作为参数传给 fn2，以此类推。我们使用 Redux 创建一个 store 时，完成对`compose`方法的调用，Redux 精简源码类比为：
>
> 复制代码
>
> ```
> // 这是一个简单的打日志中间件
> function logger({ getState, dispatch }) {
>     // next 代表下一个中间件包装过后的 dispatch 方法，action 表示当前接收到的动作
>     return next => action => {
>         console.log("before change", action);
>         // 调用下一个中间件包装的 dispatch 
>         let val = next(action);
>         console.log("after change", getState(), val);
>         return val;
>     };
> }
> // 使用 logger 中间件，创建一个增强的 store
> let createStoreWithMiddleware = Redux.applyMiddleware(logger)(Redux.createStore)
> function applyMiddleware(...middlewares) {
>   // middlewares 为中间件列表，返回一个接受原始 createStore 方法（Redux.createStore）作为参数的函数
>   return createStore => (...args) => {
>     // 创建原始的 store
>     const store = createStore(...args)
>     // 每个中间件都会被传入 middlewareAPI 对象，作为中间件参数
>     const middlewareAPI = {
>       getState: store.getState,
>       dispatch: (...args) => dispatch(...args)
>     }
> 
>     // 给每个中间件传入 middlewareAPI 参数
>     // 中间件的统一模板为 next => action => next(action) 格式
>     // chain 中保存的都是 next => action => {next(action)} 的方法
>     const chain = middlewares.map(middleware => middleware(middlewareAPI))
> 
>     // 传入最原始 store.dispatch 方法，作为 compose 二级参数，compose 方法最终返回一个增强的dispatch 方法
>     dispatch = compose(...chain)(store.dispatch)
> 
>     return {
>       ...store,
>       dispatch  // 返回一个增强版的 dispatch
>     }
>   }
> }
> ```
>
> 如上代码，我们将 Redux 中间件特点总结为：
>
> - Redux 中间件接收`getState`和`dispatch`两个方法组成的对象作为参数；
> - Redux 中间件返回一个函数，该函数接收下一个`next`方法作为参数，并返回一个接收 action 的新的`dispatch`方法；
> - Redux 中间件通过手动调用`next(action)`方法，执行下一个中间件。
>
> 我们将 Redux 的中间件作用机制总结为下图：
>
> ![202127-92020.png](https://s0.lgstatic.com/i/image6/M00/03/C9/Cgp9HWAfsQuABGmXAAGNk0fcn-c946.png)
>
> Redux 的中间件作用机制
>
> 看上去也像是一个洋葱圈模型，但是对于同步调用和异步调用稍有不同，以三个中间件为例。
>
> - 三个中间件均是正常同步调用`next(action)`，则执行顺序为：中间件 1 before next → 中间件 2 before next → 中间件 3 before next → dispatch 方法调用 → 中间件 3 after next → 中间件 2 after next → 中间件 1 after next。
> - 第二个中间件没有调用`next(action)`，则执行顺序为：中间件 1 befoe next → 中间件 2 逻辑 → 中间件 1 after next，注意**此时中间件 3 没有被执行**。
> - 第二个中间件异步调用`next(action)`，其他中间件均是正常同步调用`nextt(action)`，则执行顺序为：中间件 1 before next → 中间件 2 同步代码部分 → 中间件 1 after next → 中间件 2 异步代码部分 before next → 中间件 3 before next → dispatch 方法调用 → 中间件 3 after next → 中间件 2 异步代码部分 after next。
>
> ### 利用中间件思想，实现一个中间件化的 Fetch 库
>
> 上面我们分析了前端中的中间件化思想，这一部分，我们活学活用，利用中间件思路，结合上一讲内容，实现一个中间件化的 Fetch 库。
>
> 我们先来思考一个中间件化的 Fetch 库有哪些优点呢？Fetch 库的核心实现请求的发送，而各种业务逻辑以中间件化的插件模式进行增强，这样一来，实现了特定业务需求和请求库的解耦，更加灵活，也是一种分层思想的体现。具体来说，一个中间件化的 Fetch 库：
>
> - 支持业务方递归扩展底层 Fetch API 能力；
> - 方便测试；
> - 一个中间件化的 Fetch 库，天然支持各类型的 Fetch 封装（比如 Native Fetch、fetch-ponyfill、fetch-polyfill 等）。
>
> 我们给这个中间件化的 Fetch 库取名为：fetch-wrap，借助 [fetch-wrap](https://github.com/benjamine/fetch-wrap) 的实现，预期使用方式为：
>
> 复制代码
>
> ```
> const fetchWrap = require('fetch-wrap');
> // 这里可以接入自己的核心 Fetch 底层实现，比如使用原生 Fetch，或同构的 isomorphic-fetch 等
> let fetch = require('isomorphic-fetch');
> // 扩展 Fetch 中间件
> fetch = fetchWrap(fetch, [
>   middleware1,
>   middleware2,
>   middleware3,
> ]);
> // 一个典型的中间件
> function middleware1(url, options, innerFetch) {
> 	// ...
> 	// 业务扩展
> 	// ...
> 	return innerFetch(url, options);
> }
> // 一个更改 URL 的中间件
> function(url, options, fetch) {
> 	// modify url or options
> 	return fetch(url.replace(/^(http:)?/, 'https:'), options);
> },
> // 一个修改返回结果的中间件
> function(url, options, fetch) {
> 	return fetch(url, options).then(function(response) {
> 	  if (!response.ok) {
> 	    throw new Error(result.status + ' ' + result.statusText);
> 	  }
> 	  if (/application\/json/.test(result.headers.get('content-type'))) {
> 	    return response.json();
> 	  }
> 	  return response.text();
> 	});
> }
> // 一个做错误处理的中间件
> function(url, options, fetch) {
> 	// catch errors
> 	return fetch(url, options).catch(function(err) {
> 	  console.error(err);
> 	  throw err;
> 	});
> }
> ```
>
> 核心实现也不困难，观察`fetchWrap`使用方式，我们实现源码为：
>
> 复制代码
>
> ```
> // 接受第一个参数为基础 Fetch，第二个参数为中间件数组或单个中间件
> module.exports = function fetchWrap(fetch, middleware) {
>     // 没有使用中间件，则返回原生 fetch
> 	if (!middleware || middleware.length < 1) {
> 		return fetch;
> 	}
> 	
> 	// 递归调用 extend 方法，每次递归时剔除出 middleware 数组中的首项
> 	var innerFetch = middleware.length === 1 ? fetch : fetchWrap(fetch, middleware.slice(1));
> 	
> 	var next = middleware[0];
> 	
> 	return function extendedFetch(url, options) {
> 		try {
> 		  // 每一个 Fetch 中间件通过 Promsie 来串联
> 		  return Promise.resolve(next(url, options || {}, innerFetch));
> 		} catch (err) {
> 		  return Promise.reject(err);
> 		}
> 	};
> }
> ```
>
> 我们可以看到，每一个中间件都接收一个`url`和`options`参数，因此具有了改写`url`和`options`的能力；同时接收一个`innerFetch`方法，`innerFetch`为上一个中间件包装过的`fetch`方法，而每一个中间件也都返回一个包装过的`fetch`方法，将各个中间件依次调用串联。
>
> 另外，社区上的 [umi-request](https://www.npmjs.com/package/umi-request) 的中间件机制也是类似的，其核心代码：
>
> 复制代码
>
> ```
> class Onion {
>   constructor() {
>     this.middlewares = [];
>   }
>   // 存储中间件
>   use(newMiddleware) {
>     this.middlewares.push(newMiddleware);
>   }
>   // 执行中间件
>   execute(params = null) {
>     const fn = compose(this.middlewares);
>     return fn(params);
>   }
> }
> export default function compose(middlewares) {
>   return function wrapMiddlewares(params) {
>     let index = -1;
>     function dispatch(i) {
>       index = i;
>       const fn = middlewares[i];
>       if (!fn) return Promise.resolve();
>       return Promise.resolve(fn(params, () => dispatch(i + 1)));
>     }
>     return dispatch(0);
>   };
> }
> ```
>
> 我们可以看到，上述源码更像 Koa 的实现了，但其实道理和上面的 fetch-wrap 大同小异。至此，相信你已经了解了中间件的思想，也能够体会洋葱模型的精妙设计。
>
> ### 总结
>
> 这一讲，我们通过分析前端不同框架的中间件设计，剖析了中间件化这一重要思想。中间件化意味着插件化，这也是上一讲提到的分层思想的一种实现，同时，这种实现思路灵活且扩展能力强，能够和核心逻辑相解耦，需要你细心体会。
>
> 本讲主要内容如下：
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/94/A4/CgqCHmAY-AqAIZqkAAO1O62z-y4965.png)
>
> 在下一讲中，我们将继续围绕着代码设计中的灵活性和定制性这一话题展开，同时也给大家留一个思考题：你在平时开发中，见过或者使用过哪些插件化的工程或技术呢？欢迎在留言区和我分享你的观点，我们下一讲再见。