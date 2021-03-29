[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5805)



> 组件的设计也好，性能优化也好，它们彼此割裂，并不能反映真实的工程质量，也不能反映代码质量。面试中除了探讨造飞机的话题，也需要落地讲讲代码怎么写、怎么放、怎么用，才能更好维护，所以如何提升 React 代码可维护性也是面试官常问的一个问题。
>
> ### 破题
>
> 在探讨 React 代码的可维护性之前，需要先聊一个话题，即当我们在探讨可维护性的时候，我们究竟在聊什么。你会发现很难用一句话解释清楚这样一个模糊的概念，是指代码规范，还是设计模式呢？如果我们要认真探讨这个问题的话，其实可以有很多维度，并没有标准答案。这里，我提供一个不一样的视角看待这个问题：表面上讨论的是 React 代码，实际上是基于 React 开发的项目，所以可以从**软件工程的角度**去尝试理解。
>
> 在软件工程中，**可维护性**对应的单词是 Maintainability，与它相近的概念还有**技术债**和**代码异味**，这些都表示当前代码迭代的难易程度。简而言之，当项目可维护性很差的时候，往往意味着该项目既难以修改，也难以拓展。更专业一些的话，就像 ISO/IEC 9126 的国际标准中指出产品可维护性反映了五个特征。
>
> - **可分析性**，指工程项目拥有定位产品缺陷的能力，暗指定位缺陷的成本。举一个工作中的例子，你的页面在线上出了问题，但是你找不到相关手段去定位哪里有问题，在自己的电脑上也无法复现，线上代码在 uglify 后也无法阅读，这就是缺少可分析性。
> - **可改变性**，指工程项目拥有基本的迭代能力，暗指迭代的成本。这点相对好理解，通常在你看完代码后，如果觉得这次的内容容易修改，那迭代成本就很低；如果觉得实在改不动，那么这就不具备可变性了。
> - **稳定性**，指避免工程项目因为代码修改而造成线上意外。这点也很好理解，更生活化一点的表达就是，你敢不敢改这段代码？你的修改是否会影响线上运行？如何确保每次迭代都不会影响线上环境，那就是保障了稳定性。
> - **易测试性**，指工程项目能够快速发现产品缺陷的能力。如果代码有问题，项目有 Bug，怎么在交付前发现？怎么在引起大规模故障前发现？用什么手段可以发掘？这就是易测试性。
> - **可维护性的依从性**，指遵循相关的标准或约定，即团队开发工程规范，比如代码风格的要求、开发流水线的要求等。
>
> 基于各个框架开发的项目，其可维护性的基本特征是一致的，区别在于使用 React 构建项目，它的可维护性通过什么额外的方案来保障，比如特殊的文档规范、设计准则还是自动化工具。
>
> 所以这里需要建立**特征与方案之间的联系**，比如特征是可分析性，那它对应的方案是什么？方案是否有现成的工具可用，你又是如何规范使用的？
>
> ### 审题
>
> 在有了这样的基本认知下，我们就可以开始整理答题的框架图了。整理上述的思路，本题需要把握两个方面：
>
> - **特征**，也就是**答题维度**，包含了可分析性、可改变性、稳定性、易测试性、可维护性的依从性；
> - **方案**，从每个单一维度出发阐述 React 项目的差异与可使用的规范、工具等。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/05/DF/CgpVE2ABPF-AXR2dAAGZ3QevoI0621.png)
>
> 特征可以帮助我们建立一个基本的答题轮廓，再根据实际经验逐步填充完整，你的答案才会呈现出层次感。
>
> ### 入手
>
> #### 可分析性
>
> 仅凭可分析性的概念，确实难以下手。这里你可以用在第 12 讲[“React 的渲染异常会造成什么后果？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566#/detail/pc?id=5802)中提到的做工程的方案，就是预防与兜底：
>
> - **预防**，即从上线前开始，可以对代码做哪些措施防止出现线上问题；
> - **兜底**，就是上线后又可以做哪些方案加快线上故障的定位速度。
>
> **预防**
>
> 从预防的角度出发，能够提前发掘代码中的问题点，通过使用人工或者工具审查的方式去实现。
>
> **人工审查代码**的方式，标准称谓是 Code Review。基于 React 写法的易错点，团队内部会总结出一些实践准则，比如这篇由网友分享的 [Review checklist](https://gist.github.com/bigsergey/aef64f68c22b3107ccbc439025ebba12) 中的案例就提供了一些实践。
>
> **工具审查**的方式，标准称谓是静态代码检查工具。在 JavaScript 世界中，静态代码检查工具主要有 3 个，分别是**JSLint**、**JSHint**、**ESLint**。从生态发展的角度上，支持配置化与插件拓展 ESLint 获得了最终的胜利。基于 ESLint 有不少大厂给出了自己的最佳实践，最经典的规则方案莫过于 Airbnb 的 [eslint-config-airbnb](https://www.npmjs.com/package/eslint-config-airbnb)。这些规则方案将人工审查工作转化为工具自动化审查，节约了团队内部的时间。
>
> 那是不是说明 Code Review 没用了？并不是，工具并不能检查业务逻辑，所以我更推荐团队内部将 Code Review 的重心放到**代码的业务逻辑**上。
>
> **兜底**
>
> 兜底能够快速**定位线上报错**。在线环境的代码通常是经过 UglifyJS 混淆并压缩的（当然新版 Webpack 将默认的压缩器替换为了 terser），所以直接看报错信息，并不能得知对应的源码是什么样的，不利于排查问题。所以需要考虑线上代码的报错信息收集、汇总与反混淆。
>
> 最理想的情况莫过于**改造编译流水线**，在发布过程中上传 sourcemap 到报错收集平台。这里以部分开源的 Sentry 平台为例，在 Webpack 中添加 sourcemap 相关插件就可以在编译过程直接上传 sourcemap 到 Sentry 的报错平台。如下代码所示：
>
> 复制代码
>
> ```
> const SentryWebpackPlugin = require("@sentry/webpack-plugin");
> module.exports = {
>   configureWebpack: {
>     plugins: [
>       new SentryWebpackPlugin({
>         authToken: process.env.SENTRY_AUTH_TOKEN,
>         org: "exmaple-org",
>         project: "example-project",
>         include: ".",
>         ignore: ["node_modules", "webpack.config.js"],
>       }),
>     ],
>   },
> };
> ```
>
> 在使用 Sentry 捕获报错时，就能够直接查看对应的源码了，如下所示：
>
> 复制代码
>
> ```
> try {
>   aFunctionThatMightFail();
> } catch (err) {
>   Sentry.captureException(err);
> }
> ```
>
> 报错平台具体用哪一家，需要依据自家公司的基建现状，那如果当前的报错平台不支持怎么办呢？可以使用 Mozilla 开源的工具 [sourcemap](https://github.com/mozilla/source-map)，直接恢复对应的源代码信息，如下所示：
>
> 复制代码
>
> ```
> const rawSourceMap = { 
>   version: 3, 
>   file: "min.js", 
>   names: ["bar", "baz", "n"], 
>   sources: ["one.js", "two.js"], 
>   sourceRoot: "http://example.com/www/js/", 
>   mappings: "CAAC,IAAI,IAAM,SAAUA,GAClB,OAAOC,IAAID;CCDb,IAAI,IAAM,SAAUE,GAClB,OAAOA" 
> };
> const whatever = await SourceMapConsumer.with(rawSourceMap, null, consumer => { 
>   console.log(consumer.sources); 
>   // [ 'http://example.com/www/js/one.js', 
>   //   'http://example.com/www/js/two.js' ] 
>   console.log( 
>     consumer.originalPositionFor({ 
>       line: 2, 
>       column: 28 
>     }) 
>   ); 
>   // { source: 'http://example.com/www/js/two.js', 
>   //   line: 2, 
>   //   column: 10, 
>   //   name: 'n' } 
>   console.log( 
>     consumer.generatedPositionFor({ 
>       source: "http://example.com/www/js/two.js", 
>       line: 2, 
>       column: 10 
>     }) 
>   ); 
>   // { line: 2, column: 28 } 
>   consumer.eachMapping(function(m) { 
>     // ... 
>   }); 
>   return computeWhatever(); 
> });
> ```
>
> #### 可改变性
>
> 从代码层面来讲，可变性代表了**代码的可拓展能力**。一提到拓展能力，你可能会想到架构设计或者设计模式一类的概念。在这方面并没有唯一的正确答案，可以说仁者见仁智者见智，但统一的特征是做**划分边界**、**模块隔离**。
>
> 在前面的文章中提供了两个思路来提升代码的可拓展性。
>
> - 第 05 讲[“如何设计 React 组件？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566#/detail/pc?id=5795)中提到的组件设计模式，即从组件的角度出发，通过分离容器组件与展示组件的方式分离模块。其中推荐了 Storybook 来沉淀展示组件。
> - 第 08 讲[“列举一种你了解的 React 状态管理框架”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566#/detail/pc?id=5798)中提到的状态管理框架。状态管理框架中有相对成熟的设计模式，比如 Redux 中的 action、reducer 等，它的边界很清楚，你很容易明白业务逻辑该如何拆解、如何放入模块中。
>
> 只要跟着框架走就完事了吗？不，面试官还要听你的思考。在前面的章节中反复提过思考是什么：思考是调研方案、对比方案，说明选择这个方案的原因，阐述在该方案基础上的实践。
>
> #### 稳定性
>
> 要增强项目稳定性，从代码层面来讲，常规的思路是**加测试**。但写测试并不是一件容易的事。你会发现在前端项目中，无论是单元测试还是集成测试，整体覆盖比例都很低，常常通过人工测试“点点点”的方式保证稳定性。
>
> 究其主要原因，前端测试并不好写。不好写并不是指代码不好写，而是针对 UI 层不好写。即便是做 Android 和 iOS 开发，其 UI 测试也很少有人认真写。因为国内的业务迭代模式都非常快，快到 UI 层难以有稳定的测试代码，所以通常大家也不会花太多时间去写组件的测试，基于实际情况，有条件写测试的话，也是尽量给**核心业务写测试**。
>
> 比如购物车的商品与价格计算逻辑，每次有人修改这段代码必然会心惊胆战。如果可以的话，针对这样的核心业务逻辑，测试用例尽量做到完全覆盖；在流水线上确保每次修改后，都能自动化地跑一次单元测试。
>
> 前端单元测试主要是有 Chai、Mocha 和 Jest，其中 Jest 与 React 生态最为紧密，由 Facebook 主推。
>
> 综上所述，由于 UI 层迭代更新快，所以在代码中，选取**业务核心逻辑编写测试**更利于整体项目的稳定性。
>
> #### 易测试性
>
> 易测试性与整体的代码架构相关。下面采用 Redux 官方的案例说明合理的架构为什么容易测试。
>
> 比如其中的 Action 是一个纯函数，那么编写测试就非常容易，直接输入输出验证一波就完事了。如下代码所示：
>
> 复制代码
>
> ```
> mport * as actions from '../../actions/TodoActions'
> import * as types from '../../constants/ActionTypes'
> describe('actions', () => {
>   it('should create an action to add a todo', () => {
>     const text = 'Finish docs'
>     const expectedAction = {
>       type: types.ADD_TODO,
>       text
>     }
>     expect(actions.addTodo(text)).toEqual(expectedAction)
>   })
> })
> ```
>
> 再比如，它的 Reducer 也是纯函数，那么只需要验证输入输出就可以了。如下代码所示：
>
> 复制代码
>
> ```
> import reducer from '../../structuring-reducers/todos'
> import * as types from '../../constants/ActionTypes'
> describe('todos reducer', () => {
>   it('should return the initial state', () => {
>     expect(reducer(undefined, {})).toEqual([
>       {
>         text: 'Use Redux',
>         completed: false,
>         id: 0
>       }
>     ])
>   })
>   it('should handle ADD_TODO', () => {
>     expect(
>       reducer([], {
>         type: types.ADD_TODO,
>         text: 'Run the tests'
>       })
>     ).toEqual([
>       {
>         text: 'Run the tests',
>         completed: false,
>         id: 0
>       }
>     ])
>     expect(
>       reducer(
>         [
>           {
>             text: 'Use Redux',
>             completed: false,
>             id: 0
>           }
>         ],
>         {
>           type: types.ADD_TODO,
>           text: 'Run the tests'
>         }
>       )
>     ).toEqual([
>       {
>         text: 'Run the tests',
>         completed: false,
>         id: 1
>       },
>       {
>         text: 'Use Redux',
>         completed: false,
>         id: 0
>       }
>     ])
>   })
> })
> ```
>
> 这里有两点值得我们学习：
>
> - **合理的架构划分**，使得模块与模块之间互不干涉、各自分离，可以让测试相对独立；
> - **纯函数**在测试上有着得天独厚的优势，让测试验证的过程变得更为简单，试想，如果是**一个类**，有状态流转变化，有缓存机制，那是否会给测试带来困难，所以现在前端有一种思潮就是**编写一个接一个的函数**，而**比较少写类**。
>
> 比如 [react-native-code-push](https://github.com/microsoft/react-native-code-push) 这个项目中的代码就是在调用一个接一个的函数。这是 React 社区值得关注的一种现象。
>
> #### 依从性
>
> 依从性讲的是**约束**。为什么要约束呢？因为**统一编码规范与代码风格可以提升易读性**，**减少认知差异**，**防止不规范操作埋藏的潜在隐患**。软件开发又是一个过分依赖人的活动，而人往往是最不可靠的。这里需要注意，强调约束时，落地的方案一定要在**工具**上，而非人本身。
>
> 在前端工程中最常见的 Lint 工具包含：
>
> - 针对 JavaScript 的 ESLint；
> - 针对样式的 Stylelint；
> - 针对代码提交的 Commitlint；
> - 针对编辑器风格的 Editorconfig；
> - 针对代码风格的 Prettier。
>
> 这些也都适用于 React 生态。
>
> ### 答题
>
> 通过梳理上述的知识点，可以尝试从软件工程的角度来回答本题。
>
> > 如何提升 React 代码的可维护性，究其根本是考虑如何提升 React 项目的可维护性。从软件工程的角度出发，可维护性包含了可分析性、可改变性、稳定性、易测试性与可维护性的依从性，接下来我从这五个方面对相关工作进行梳理。
> >
> > 可分析性的目标在于快速定位线上问题，可以从预防与兜底两个维度展开工作，预防主要依靠 Lint 工具与团队内部的 Code Review。Lint 工具重在执行代码规划，力图减少不合规的代码；而 Code Review 的重心在于增强团队内部的透明度，做好业务逻辑层的潜在风险排查。兜底主要是在流水线中加入 sourcemap，能够通过线上报错快速定位源码。
> >
> > 可改变性的目标在于使代码易于拓展，业务易于迭代。工作主要从设计模式与架构设计展开。设计模式主要指组件设计模式，通过容器组件与展示组件划分模块边界，隔绝业务逻辑。整体架构设计，采用了 rematch 方案，rematch 中可以设计的 model 概念可以很好地收敛 action、reducer 及副作用，同时支持动态引入 model，保障业务横向拓展的能力。Rematch 的插件机制非常利于做性能优化，这方面后续可以展开聊一下。
> >
> > 接下来是稳定性，目标在于避免修改代码引起不必要的线上问题。在这方面，主要通过提升核心业务代码的测试覆盖率来完成。因为业务发展速度快、UI 变化大，所以基于 UI 的测试整体很不划算，但背后沉淀的业务逻辑，比如购物车计算价格等需要长期复用，不时修改，那么就得加测试。举个个人案例，在我自己的项目中，核心业务测试覆盖率核算是 91%，虽然没完全覆盖，但基本解决了团队内部恐惧线上出错的心理障碍。
> >
> > 然后是易测试性，目标在于发现代码中的潜在问题。在我个人负责的项目中，采用了 Rematch 的架构完成模块分离，整体业务逻辑挪到了 model 中，且 model 自身是一个 Pure Object，附加了多个纯函数。纯函数只要管理好输入与输出，在测试上就很容易。
> >
> > 最后是可维护性的依从性，目标在于建立团队规范，遵循代码约定，提升代码可读性。这方面的工作就是引入工具，减少人为犯错的概率。其中主要有检查 JavaScript 的 ESLint，检查样式的 stylelint，检查提交内容的 commitlint，配置编辑器的 editorconfig，配置样式的 prettier。总体而言，工具的效果优于文档，团队内的项目整体可保持一致的风格，阅读代码时的切入成本相对较低。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/05/DD/Cip5yGABPJ-AX7tWAAGZ3QevoI0417.png)
>
> ### 总结
>
> 本讲提供了一个从软件工程角度论述的思路，但这并不是唯一答案，我们每个人都可以有自己的理论体系，以及讲解重点，只要它体系齐全、逻辑自洽、结构清晰，就能说服面试官。本讲中提到的理论，看一看、读一读，最重要的是有自己的理解，而本讲中提到的相关工具建议重点关注，试一试、用一用，看看是否真的能帮助你提升 React 代码的可维护性。
>
> 你可以尝试用其他思路思考下本题，通过在留言区留言的方式分享自己的见解。我也会在留言区与你互动。
>
> 在下一讲中就正式进入 React Hooks 的环节，先看看 Hooks 在使用上有什么限制。