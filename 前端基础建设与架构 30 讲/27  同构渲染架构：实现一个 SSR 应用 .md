[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5956)



> 从这一讲开始，我们正式进入 Node.js 主题学习。作为 Node.js 技术的重要应用场景，同构渲染 SSR 应用尤其重要。不管是服务端渲染还是服务端渲染衍生出的同构应用，现在来看已经并不新鲜了，实现起来也并不困难。可是有的开发者认为：同构应用不就是调用一个`renderToString`（React 中）类似的 API 吗？
>
> 讲道理，确实如此，但同构应用也不只是这么简单。就拿面试来说，同构应用的考察点不是“纸上谈兵”的理论，而是实际实施时的细节。这一讲我们就来一步步实现一个 SSR 应用，并分析 SSR 应用的重点环节。相关内容你可以参考：[实现一个简易 ssr](https://github.com/HOUCe/ssr)。
>
> ### 实现一个简易 SSR 应用
>
> SSR 渲染架构的优势已经非常明显了，不管是对SEO 友好还是性能提升，大部分开发者已经耳熟能详了。这一部分，我们以 React 技术栈为背景，实现一个 SSR 应用。
>
> 首先启动项目：
>
> 复制代码
>
> ```
> npm init --yes
> ```
>
> 配置 Babel 和Webpack，目的是将ESM 和React编译为 Node.js和浏览器能够理解的代码。相关`.babelrc`内容如下代码：
>
> 复制代码
>
> ```
> {
>   "presets": ["@babel/env", "@babel/react"]
> }
> ```
>
> 如上代码，我们直接使用了`@babel/env`和`@babel/react`作为 presets。相关`webpack.config.js`内容如下代码：
>
> 复制代码
>
> ```
> const path = require('path');
> module.exports = {
>     entry: {
>         client: './src/client.js',
>         bundle: './src/bundle.js'
>     },
>     output: {
>         path: path.resolve(__dirname, 'assets'),
>         filename: "[name].js"
>     },
>     module: {
>         rules: [
>             { test: /\.js$/, exclude: /node_modules/, loader: "babel-loader" }
>         ]
>     }
> }
> ```
>
> 配置入口文件为`./src/client.js`和`./src/bundle.js`，打包结果如下。
>
> - `assets/bundle.js`：CSR 架构下浏览器端脚本。
> - `assets/client.js`：SSR 架构下浏览器端脚本，衔接 SSR 部分。
>
> `src/`文件夹包含所有源码，Babel 将会编译该文件内代码到`views/`目录。这里需要你思考：为什么我们要编译源码呢？
>
> 业务源码中，我们使用 ESM 编写 React 和 Redux 代码，**对于低版本 Node.js来说，并不能直接支持 ESM 规范**，因此需要使用 Babel 将`src/`文件夹内代码编译到`views/`目录中。相关命令如下：
>
> 复制代码
>
> ```
> "babel": "babel src -d views"
> ```
>
> 我们对项目目录进行说明：
>
> - `src/components`中我们存放 React 组件；
> - `src/redux/`中我们存放 Redux 相关代码；
> - `assets/`和`media/`中我们存放样式文件及图片；
> - `src/server.js`和`src/template.js`是 Node.js环境相关脚本。
>
> 接下来，我们进入 Node.js相关的`src/server.js`和`src/template.js`脚本的编写。
>
> `src/server.js`如下代码所示：
>
> 复制代码
>
> ```
> import React from 'react';
> import { renderToString } from 'react-dom/server';
> import { Provider } from 'react-redux';
> import configureStore from './redux/configureStore';
> import App from './components/app';
> module.exports = function render(initialState) {
> 	// 初始化 redux store
>   const store = configureStore(initialState);
>   let content = renderToString(<Provider store={store} ><App /></Provider>);
>   const preloadedState = store.getState();
>   return {
>     content,
>     preloadedState
>   };
> };
> ```
>
> 我们展开具体分析：
>
> - `initialState`作为参数传递给`configureStore()`方法，并实例化一个新的Store；
> - 调用`renderToString()`方法，得到服务端渲染的 HTML 字符串`content`；
> - 调用 Redux`getState()`方法，得到状态为`preloadedState`；
> - 返回 HTML 字符串`content`和 preloadedState。
>
> `src/template.js`代码如下：
>
> 复制代码
>
> ```
> export default function template(title, initialState = {}, content = "") {
>   let scripts = ''; 
>   // 是否有 content 内容
>   if (content) {
>     scripts = ` <script>
>                    window.__STATE__ = ${JSON.stringify(initialState)}
>                 </script>
>                 <script src="assets/client.js"></script>
>                 `
>   } else {
>     scripts = ` <script src="assets/bundle.js"> </script> `
>   }
>   let page = `<!DOCTYPE html>
>               <html lang="en">
>               <head>
>                 <meta charset="utf-8">
>                 <title> ${title} </title>
>                 <link rel="stylesheet" href="assets/style.css">
>               </head>
>               <body>
>                 <div class="content">
>                    <div id="app" class="wrap-inner">
>                       ${content}
>                    </div>
>                 </div>
>                   ${scripts}
>               </body>
>               `;
>   return page;
> }
> ```
>
> 我们对上述代码进行解读：`template`函数接受`title`、`state`和`content`作为参数，拼凑成最终的 HTML 文档，并将`state`挂载到`window.__STATE__`中，作为 script 标签内联到 HTML 文档，同时将 SSR 架构下`assets/client.js`脚本或`assets/bundle.js`嵌入。
>
> 下面，我们再聚焦同构部分的浏览器端脚本。
>
> 在CSR 架构下，`src/bundle.js`代码如下：
>
> 复制代码
>
> ```
> import React from 'react';
> import { render } from 'react-dom';
> import { Provider } from 'react-redux';
> import configureStore from './redux/configureStore';
> import App from './components/app';
> // 获取 store
> const store = configureStore();
> render(
>   <Provider store={store} > <App /> </Provider>,
>   document.querySelector('#app')
> );
> ```
>
> 而 SSR 架构下，`src/client.js`代码类似：
>
> 复制代码
>
> ```
> import React from 'react';
> import { hydrate } from 'react-dom';
> import { Provider } from 'react-redux';
> import configureStore from './redux/configureStore';
> import App from './components/app';
> const state = window.__STATE__;
> delete window.__STATE__;
> const store = configureStore(state);
> hydrate(
>   <Provider store={store} > <App /> </Provider>,
>   document.querySelector('#app')
> );
> ```
>
> `src/client.js`对比`src/bundle.js`，比较关键的不同点在于**使用了**`window.__STATE__.`**获取初始状态，同时使用了**`hydrate()`**方法代替了**`render()`。
>
> 至此，我们就实现了一个简易的 SSR 应用。虽然简单，但完全体现了 SSR 架构的原理。然而生产情况复杂多变，我们继续往下看。
>
> ### 同构应用中你容易忽略的细节
>
> 接下来，我们对几个更细节的问题加以分析。这些问题的处理，不再是代码层面的解决方案，更是工程化方向的设计。
>
> #### 环境区分
>
> 我们知道，同构应用实现了客户端代码和服务端代码的基本统一，我们只需要编写一种组件，就能生成适用于服务端和客户端的组件案例。可是你是否知道，大多数情况下服务端代码和客户端代码需要单独处理？下面我简单举几个例子。
>
> - **路由代码差别**
>
> 服务端需要根据请求路径，匹配页面组件；客户端需要通过浏览器中的地址，匹配页面组件。
>
> 客户端代码：
>
> 复制代码
>
> ```
>   const App = () => {
>     return (
>       <Provider store={store}>
>         <BrowserRouter>
>           <div>
>             <Route path='/' component={Home}>
>             <Route path='/product' component={Product}>
>           </div>
>         </BrowserRouter>
>       </Provider>
>     )
>   }
>   ReactDom.render(<App/>, document.querySelector('#root'))
> ```
>
> BrowserRouter 组件根据 window.location 以及 history API 实现页面切换，而服务端肯定是无法获取 window.location 的。
>
> 服务端代码如下：
>
> 复制代码
>
> ```
>   const App = () => {
>     return 
>       <Provider store={store}>
>         <StaticRouter location={req.path} context={context}>
>           <div>
>             <Route path='/' component={Home}>
>           </div>
>         </StaticRouter>
>       </Provider>
>   }
>   Return ReactDom.renderToString(<App/>)
> ```
>
> 在服务端，需要**使用 StaticRouter 组件**，并将请求地址和上下文信息作为 location 和 context 这两个props 传入 StaticRouter 中。
>
> - **打包差别**
>
> 服务端运行的代码如果需要依赖 Node 核心模块或者第三方模块，就不再需要把这些模块代码打包到最终代码中了。因为环境已经安装这些依赖，可以直接引用。这样一来，就需要我们**在 Webpack 中配置 target：node**，并借助 webpack-node-externals 插件，解决第三方依赖打包的问题。
>
> #### 注水和脱水
>
> 什么叫作注水和脱水呢？这个和同构应用中数据的获取有关：在服务器端渲染时，首先服务端请求接口拿到数据，并处理准备好数据状态（如果使用 Redux，就是进行Store 的更新），为了减少客户端的请求，我们需要保留住这个状态。
>
> 一般做法是在服务器端返回 HTML 字符串的时候，将数据 JSON.stringify 一并返回，这个过程，叫作脱水（dehydrate）；在客户端，就不再需要进行数据的请求了，可以直接使用服务端下发下来的数据，这个过程叫注水（hydrate）。
>
> 响应代码前面已经有所体现了，但是在服务端渲染时，服务端如何能够请求所有的 APIs，保障数据全部已经请求呢？
>
> 一般有两种方法进行服务端请求。
>
> - react-router 的解决方案是配置路由route-config，结合 matchRoutes，找到页面上相关组件所需的请求接口的方法并执行请求。这就要求开发者通过路由配置信息，显式地告知服务端请求内容。如下代码：
>
> 复制代码
>
> ```
>   const routes = [
>     {
>       path: "/",
>       component: Root,
>       loadData: () => getSomeData()
>     }
>     // etc.
>   ]
>   
>   import { routes } from "./routes"
>   
>   function App() {
>     return (
>       <Switch>
>         {routes.map(route => (
>           <Route {...route} />
>         ))}
>       </Switch>
>     )
>   }
> ```
>
> 在服务端代码中：
>
> 复制代码
>
> ```
>   import { matchPath } from "react-router-dom"
>   
>   const promises = []
>   routes.some(route => {
>     const match = matchPath(req.path, route)
>     if (match) promises.push(route.loadData(match))
>     return match
>   })
>   
>   Promise.all(promises).then(data => {
>     putTheDataSomewhereTheClientCanFindIt(data)
>   })
> ```
>
> - 另外一种思路类似 Next.js，我们需要在 React 组件上**定义静态方法**。比如定义静态 loadData 方法，在服务端渲染时，我们可以遍历所有组件的 loadData，获取需要请求的接口。
>
> #### 安全问题
>
> 安全问题非常关键，尤其是涉及服务端渲染，开发者要格外小心。这里提出一个点：我们前面提到了注水和脱水过程，其中的代码：
>
> 复制代码
>
> ```
> ctx.body = `
>   <!DOCTYPE html>
>   <html lang="en">
>     <head>
>       <meta charset="UTF-8">
>     </head>
>     <body>
>         <script>
>         window.context = {
>           initialState: ${JSON.stringify(store.getState())}
>         }
>       </script>
>       <div id="app">
>           // ...
>       </div>
>     </body>
>   </html>
> `
> ```
>
> 非常容易遭受 XSS 攻击，JSON.stringify 可能会造成 script 注入。因此，我们需要**严格清洗 JSON 字符串中的 HTML 标签和其他危险的字符**。我习惯使用 serialize-javascript 库进行处理，这也是同构应用中最容易被忽视的细节。
>
> 这里给大家留一个思考题，React`dangerouslySetInnerHTML`API 也有类似风险，React 是怎么处理这个安全隐患的呢？
>
> #### 请求认证处理
>
> 上面讲到服务端预先请求数据，那么请你思考这样一个场景：某个请求依赖 cookie 表明的用户信息，比如请求“我的学习计划列表”。这种情况下服务端请求是不同于客户端的，不会有浏览器添加 cookie 以及不含有其他相关的 header 信息。这个请求在服务端发送时，一定不会拿到预期的结果。
>
> 解决办法也很简单：服务端请求时需要保留客户端页面请求的信息（一般是 cookie），并在 API 请求时携带并透传这个信息（cookie）。
>
> #### 样式问题处理
>
> 同构应用的样式处理容易被开发者忽视，而一旦忽略，就会掉到坑里。比如，我们不能再使用 style-loader 了，因为这个WebpackLoader 会在编译时将样式模块载入到 HTML header 中。但是在服务端渲染环境下，没有Window 对象，style-loader就会报错。一般我们使用 isomorphic-style-loader 来实现：
>
> 复制代码
>
> ```
> {
>     test: /\.css$/,
>     use: [
>         'isomorphic-style-loader',
>         'css-loader',
>         'postcss-loader'
>     ],
> }
> ```
>
> isomorphic-style-loader 的原理是什么呢？
>
> 我们知道，对于Webpack 来说，所有的资源都是模块。WebpackLoader 在编译过程中可以将导入的 CSS 文件转换成对象，拿到样式信息。因此**isomorphic-style-loader 可以获取页面中所有组件样式**。为了实现得更加通用化，isomorphic-style-loader 利用 context API，在渲染页面组件时获取所有 React 组件的样式信息，最终插入 HTML 字符串中。
>
> 在服务端渲染时，我们需要加入这样的逻辑：
>
> 复制代码
>
> ```
> import express from 'express'
> import React from 'react'
> import ReactDOM from 'react-dom'
> import StyleContext from 'isomorphic-style-loader/StyleContext'
> import App from './App.js'
> const server = express()
> const port = process.env.PORT || 3000
> server.get('*', (req, res, next) => {
>   //  css Set 类型来存储页面所有的样式
>   const css = new Set()
>   const insertCss = (...styles) => styles.forEach(style => css.add(style._getCss()))
>   const body = ReactDOM.renderToString(
>     <StyleContext.Provider value={{ insertCss }}>
>       <App />
>     </StyleContext.Provider>
>   )
>   
>   const html = `<!doctype html>
>     <html>
>       <head>
>         <script src="client.js" defer></script>
>         // 将样式内连进 html 当中
>         <style>${[...css].join('')}</style>
>       </head>
>       <body>
>         <div id="root">${body}</div>
>       </body>
>     </html>`
>   res.status(200).send(html)
> })
> server.listen(port, () => {
>   console.log(`Node.js app is running at http://localhost:${port}/`)
> })
> ```
>
> 分析上面代码，我们定义了 css Set 类型来存储页面所有的样式，并定义了 insertCss 方法。该方法通过 context 传给每个 React 组件，这样每个组件就可以调用 insertCss 方法。该方法调用时，会将组件样式加入 css Set 当中。
>
> 最后我们用`[...css].join('')`就可以获取页面的所有样式字符串。
>
> 强调一下，[isomorphic-style-loader 的源码](https://github.com/kriasoft/isomorphic-style-loader)目前已经更新，采用了最新的 ReactHooks API，我推荐给 React 开发者阅读，相信你一定收获很多！
>
> ### 总结
>
> 本小节前半部分我们“手把手”教你实现服务端渲染的同构应用，因为这些知识并不困难，社区上资料也很多。后半部分我们从更高的角度出发，剖析同构应用中那些关键的细节点和疑难问题的解决方案，这些经验源于真刀真枪的线上案例，即使你没有开发过同构应用，也能从中全方位地了解关键信息，一旦掌握了这些细节，同构应用的实现就会更稳、更可靠。
>
> 本讲内容总结如下：
>
> ![同构渲染架构： 实现一个 SSR 应用.png](https://s0.lgstatic.com/i/image6/M00/16/ED/CioPOWBHGf-ANBuWAAJkpsrE7fA808.png)
>
> 同构应用其实远比理论复杂，绝对不是几个 APIs 和几台服务器就能完成的，希望大家多思考、多动手，一定会更有体会。下一讲，我们进入 CI/CD 流程，设计一个性能守卫系统，以此帮助你了解：Node.js 除了同构直出、数据聚合以外，还能做一些重要的，且有趣的服务。