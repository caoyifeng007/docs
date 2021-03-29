[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5913)



> 前端生态有着与生俱来的混乱和与之抗衡的秩序，有着新生力量的崛起以及随之而来的规范约束。在这个背景下，正面来看，欣欣向荣的前端生态带来了广阔的发展前景，但也造成了一些困扰。比如，我们都经历过在前端基础设施建设中，被各种冗杂的配置项困扰，一不小心就是 Error，步履蹒跚。也许我们可以通过搜索引擎暂时解决问题，但是恍恍惚惚、难以洞悉问题本源。
>
> 另一方面，前端生态的重要一环是公共库。公共库的模块化规范、编译标准，甚至压缩方式都有讲究，同时公共库与使用它们的业务项目也要密切配合，这样才能打造一个顺滑的基建结果。请你仔细审视手上的项目，编译构建过程是否做到了最高效，产出代码是否达到了最高级别的安全保障，是否做到了性能体验的最佳实践？
>
> 这一讲，就让我们从公共库的角度出发，梳理当前前端生态，最终还原一个趋近完美的公共库设计标准。
>
> ### 从一个公共库处理的问题，谈如何做好“扫雷人”
>
> 这一部分，让我们以一篇网红文章“[报告老板，我们的 H5 页面在 iOS 11 系统上白屏了！](https://juejin.im/post/6856815533749338125)”开始，我先简单梳理和总结一下文章内容：
>
> - 笔者发现某些机型上出现页面白屏情况；
> - 出现在报错页面上的信息非常明显，即当前浏览器不支持`...`扩展运算符；
> - 出错的代码（使用了扩展运算符的代码）属于某个公共库代码，它没有使用 Babel 插件进行降级处理，因此线上源代码出现了`...`扩展运算符。
>
> 现在问题找到了，或许直接将出现问题的公共库代码用 Babel 进行编译降级就可以了，但问题似乎还会更加复杂。在文中环境下，需要在`vue.config.js`中加入对问题公共库`module-name/library-name`的 Babel 编译流程：
>
> 复制代码
>
> ```
> transpileDependencies: [
>   'module-name/library-name' // 出现问题的那个库
> ],
> ```
>
> vue-cli 对[transpileDependencies](https://cli.vuejs.org/zh/config/#transpiledependencies) 也有如下说明：
>
> > 默认情况下 babel-loader 会忽略所有`node_modules`中的文件。如果你想要通过 Babel 显式转译一个依赖，可以在这个选项中列出来。
>
> 按照上述操作，却得到了新的报错：`Uncaught TypeError: Cannot assign to read only property 'exports' of object '#<Object>'`。究其原因，`module-name/library-name`这个库对外输出的是 CommonJS 类型源码，而项目基础设施中 babel-transform-runtime 在编译时增加的 helper 代码，使用的是 import 引入。**最终编译结果出现了 ESM 包含 CommonJS 的情况，是不会被 Webpack 处理的**。
>
> 出现问题的原因总结如下：
>
> - plugin-transform-runtime 会根据 sourceType 选择注入 import 或者 require，sourceType 的默认值是 module，就会默认注入 import；
> - Webpack 不会处理包含 import/export 的文件中的 module.exports 导出，所以需要让 Babel 自动判断 sourceType，根据文件内是否存在 import/export 来决定注入什么样的代码。
>
> 为了适配上述问题，Babel 设置了`sourceType`属性，`sourceType：unambiguous`表示 Babel 会根据文件上下文（比如是否含有 import/export）来决定是否按照 ESM 语法处理文件。
>
> 这时候就需要配置 Babel 内容了：
>
> 复制代码
>
> ```
> module.exports = {
>   ...  // 省略的配置
>   sourceType: 'unambiguous',
>   ...  // 省略的配置
> }
> ```
>
> 但是这种做法在工程上并不推荐，上述更改方式对所有编译文件都生效，但也**增加了编译成本**（因为设置`sourceType：unambiguous`后，编译时需要做的事情更多），还有个[潜在问题](https://babeljs.io/docs/en/options#sourcetype)：
>
> > Unambiguous can be quite useful in contexts where the type is unknown, but it can lead to false matches because it's perfectly valid to have a module file that does not use import/export statements.
>
> 翻译过来，就是说并不是所有的 ESM 模块（这里指使用 ESNext 特性的文件）都含有 import/export，因此即便某个待编译文件属于 ESM 模块，也可能被 Babel 错误地判断为 CommonJS 模块而引发误判。
>
> 基于这一点，一个更合适的做法是：只对目标第三方库`'module-name/library-name'`使用`sourceType：unambiguous`，这时，Babel [overrides 属性](https://babeljs.io/docs/en/options#overrides)就派上用场了：
>
> > Allows users to provide an array of options that will be merged into the current configuration one at a time. This feature is best used alongside the "test"/"include"/"exclude" options to provide conditions for which an override should apply.
>
> 具体使用方式：
>
> 复制代码
>
> ```
> module.exports = {
> 	...  // 省略的配置
> 	overrides: [
> 		{ include: './node_modules/module-name/library-name/name.common.js',  // 使用的第三方库
> 		sourceType: 'unambiguous'
> 		}
> 	], 
> 	 ...  // 省略的配置
> };
> ```
>
> 至此，这个“iOS 11 系统白屏”问题就算告一段落了。我整理了解决路线，如下图所示：
>
> ![Lark20210105-174532.png](https://s0.lgstatic.com/i/image2/M01/04/A5/CgpVE1_0NWKAaju0AAMFDVpRq7Y221.png)
>
> 我们回过头再来看这个问题，实际上**业务方对线上测试回归不彻底**是造成问题的直接原因，但问题其实出现在一个公共库上，因而前端生态的混乱和复杂也许是更本质的原因。这里涉及两方面问题：
>
> - 作为公共库，我应该如何构建编译代码，让业务方更有保障地使用？
> - 作为使用者，我应该如何处理第三方公共库，是否还需要对其进行额外编译和处理？
>
> 被动地发现问题、解决问题只会让我们被“牵着鼻子走”——这不是我们的目的。我们应该从更底层拆解问题，下面我们就来看看更底层的内容。
>
> ### 应用项目构建和公共库构建的差异
>
> 首先我们要认清应用项目构建和公共库构建的差别。作为前端团队，我们构建了很多应用项目，对于一个应用项目来说，“只要能在需要兼容的环境中跑起来”就达到了基本目的。而对于一个公共库来说，我们的公共库可能被各种环境所引用或需要支持各种兼容需求，因此**公共库就要兼顾性能和易用性，要注重质量和广泛度**。由此看来，公共库理论上构建机制就更加复杂。
>
> 说到底，如果你能够设计好一个公共库，那么通常也能使用好一个公共库。因此，下面我们重点讨论如何设计并产出一个企业级公共库，以及如何在业务中更好地配合使用。
>
> ### 制定一个企业级公共库的设计原则
>
> 这里说的企业级公共库主要是指在企业内复用的公共库，它可以被发布到 npm 上进行社区共享，也可以在企业内的私有 npm 中限定范围地共享。总之，企业级公共库是需要在业务中被使用的。我认为一个企业级公共库的设计原则应该包括以下几点。
>
> 1. **对于开发者共创公共库，最大化确保开发体验**：
>
> - 最快地搭建调试和开发环境
> - 安全地发版维护
>
> 1. **对于使用者，最大化确保使用体验**：
>
> - 公共库文档建设完善
> - 公共库质量有保障
> - 接入和使用负担最小
>
> 基于上述原则，在团队里，设计一个公共库前，你需要考虑：
>
> - 自研公共库还是使用社区已有轮子；
> - 公共库的运行环境是什么，这将决定公共库的编译构建目标；
> - 公共库是偏向业务还是业务 free，这将决定公共库的职责和边界。
>
> 上述内容并非纯理论原则，而是直接决定了公共库实现的技术选型。比如，为了实现更完善的文档建设，尤其是 UI 组件类文档，可以考虑部署静态组件展示站点，进行组件展示以及用法说明。更智能、工程化的内容，我们可以考虑使用类似 [JSDoc](https://github.com/jsdoc/jsdoc) 来实现 JavaScript API 文档生成，组件类公共库可以考虑 [Storybook](https://storybook.js.org/) 或者 [Styleguides](http://styleguides.io/) 作为标准接入方案。
>
> 再比如，我们的**公共库适配环境**是什么？一般来讲可能需要兼容：浏览器/Node.js/同构环境等。不同环境对应了不同的编译和打包标准，这就需要你进行思考：如果目标是浏览器环境，那么如何才能充分实现性能最优解？如帮助业务方实现 tree-shaking 等优化技术。
>
> 同时，为了减轻业务使用负担，作为企业级公共库，以及对应使用这些企业级公共库的应用项目，可以指定标准规范的 babel-preset，保证编译产出的统一。这样一来，业务项目（即使用公共库方）可以以统一的接入标准引入。
>
> 下面是我基于对目前前端生态的理解，草拟的一份 babel-preset（该 preset 设计方案具有时效性）。请继续阅读下文，我们来对 @lucas/babel-preset 一探究竟。
>
> ### 制定一个统一标准化 babel-preset
>
> 企业中，所有公共库或应用项目都使用一套 @lucas/babel-xxx-preset，按照 @lucas/babel-xxx-preset 的编译要求进行编译，这样业务使用时，接入标准统一。
>
> 原则上讲，这样的统一化能够有效避免本文开头的“线上问题”。同时这个 @lucas/babel-preset 应该能够适应各种项目需求，比如使用 TypeScript/Flow 等扩展语法的项目。
>
> 这里给出一份设计方案，供你参考和讨论。
>
> 1. 支持 NODE_ENV = 'development' | 'production' | 'test' 三种环境，并有对应的优化。
> 2. 配置插件默认不开启 Babel`loose: true`配置，让插件的行为尽可能地遵循规范，但对有较严重性能损耗或有兼容性问题的情况保留修改入口。
> 3. 这份设计方案落地后产出，应该**支持应用编译和公共库编译**，即可以按照 @lucas/babel-preset/app，@lucas/babel-preset/dependencies 和 @lucas/babel-preset/library，@lucas/babel-preset/library/compact 进行区分使用（具体不同预设集合的角色，下面会详细展开）。
>
> @lucas/babel-preset/app，@lucas/babel-preset/dependencies 都可以作为编译应用项目的预设使用，但他们也有所差别，具体如下：
>
> - @lucas/babel-preset/app 负责编译除`node_modules`外的业务代码；
> - @lucas/babel-preset/dependencies 编译`node_modules`第三方代码；
> - @lucas/babel-preset/library 和 @lucas/babel-preset/library/compact 作为编译公共库使用的预设，他们也有所差别，@lucas/babel-preset/library 按照当前 Node 环境编译输出代码，而 @lucas/babel-preset/library/compact 会编译降级为 ES5。
>
> 1. 对于企业级公共库，建议使用标准 ES 特性发布；对 tree-shaking 有强烈需求的库，应同时发布 ES module 格式代码。
> 2. 对于企业级公共库，发布的代码不包含 polyfills，由使用方统一处理。
> 3. 对于应用编译，使用 @babel/preset-env 同时编译应用代码与第三方库代码。
> 4. 对于应用编译，需要对`node_modules`进行编译，并且为`node_modules`配置`sourceType: 'unambiguous'`，以确保第三方依赖包中的 CommonJS 模块能够被正确处理。
> 5. 对于应用编译，启用 plugin-transform-runtime，避免同样的 helper 代码被重复注入多个文件，以缩减打包后文件的体积。同时自动注入 regenerator-runtime，避免污染全局变量。
> 6. 注入绝对路径引用的 @babel/runtime 包中对应的 helper，以确保能够引用到正确版本的 @babel/runtime 包中的文件。
>
> 此外，第三方库可能通过 dependencies 依赖自己的 @babel/runtime，而 @babel/runtime 不同版本之间不能确保兼容 (比如 6.x 和 7.x 之间)，这就需要确保我们为`node_modules`内代码经过 Babel 编译注入 runtime 时使用正确路径的 @babel/runtime 包。
>
> 基于以上设计，对于 CSR 应用的 Babel 编译流程，预计业务方使用预设为：
>
> 复制代码
>
> ```
> // webpack.config.js
> module.exports = {
>   presets: ['@lucas/babel-preset/app'],
> }
> // 相关 webpack 配置
> module.exports = {
>   module: {
>     rules: [
>       {
>         test: /\.js$/,
>         oneOf: [
>           {
>             exclude: /node_modules/,
>             loader: 'babel-loader',
>             options: {
>               cacheDirectory: true,
>             },
>           },
>           {
>             loader: 'babel-loader',
>             options: {
>               cacheDirectory: true,
>               configFile: false,
>               // 使用我们的 preset
>               presets: ['@lucas/babel-preset/dependencies'],
>               compact: false,
>             },
>           },
>         ],
>       },
>     ],
>   },
> }
> ```
>
> 我们可以看到，上述使用方式按照`node_modules`进行了区分，对于`node_modules`我们开启 cacheDirectory 缓存。对于应用，我们直接使用 babel-loader 进行编译，并使用`@lucas/babel-preset/dependencies`这个 Babel preset，其内容为：
>
> 复制代码
>
> ```
> const path = require('path')
> const {declare} = require('@babel/helper-plugin-utils')
> const getAbsoluteRuntimePath = () => {
>   return path.dirname(require.resolve('@babel/runtime/package.json'))
> }
> module.exports = ({
>   targets,
>   ignoreBrowserslistConfig = false,
>   forceAllTransforms = false,
>   transformRuntime = true,
>   absoluteRuntime = false,
>   supportsDynamicImport = false,
> } = {}) => {
>   return declare(
>     (
>       api,
>       {modules = 'auto', absoluteRuntimePath = getAbsoluteRuntimePath()},
>     ) => {
>       api.assertVersion(7)
>       // 返回配置内容
>       return {
>         // https://github.com/webpack/webpack/issues/4039#issuecomment-419284940
>         sourceType: 'unambiguous',
>         exclude: /@babel\/runtime/,
>         presets: [
>           [
>             require('@babel/preset-env').default,
>             {
>               // 统一 @babel/preset-env 配置
>               useBuiltIns: false,
>               modules,
>               targets,
>               ignoreBrowserslistConfig,
>               forceAllTransforms,
>               exclude: ['transform-typeof-symbol'],
>             },
>           ],
>         ],
>         plugins: [
>           transformRuntime && [
>             require('@babel/plugin-transform-runtime').default,
>             {
>               absoluteRuntime: absoluteRuntime ? absoluteRuntimePath : false,
>             },
>           ],
>           require('@babel/plugin-syntax-dynamic-import').default,
>           !supportsDynamicImport &&
>             !api.caller(caller => caller && caller.supportsDynamicImport) &&
>             require('babel-plugin-dynamic-import-node'),
>           [
>             require('@babel/plugin-proposal-object-rest-spread').default,
>             {loose: true, useBuiltIns: true},
>           ],
>         ].filter(Boolean),
>         env: {
>           test: {
>             presets: [
>               [
>                 require('@babel/preset-env').default,
>                 {
>                   useBuiltIns: false,
>                   targets: {node: 'current'},
>                   ignoreBrowserslistConfig: true,
>                   exclude: ['transform-typeof-symbol'],
>                 },
>               ],
>             ],
>             plugins: [
>               [
>                 require('@babel/plugin-transform-runtime').default,
>                 {
>                   absoluteRuntime: absoluteRuntimePath,
>                 },
>               ],
>               require('babel-plugin-dynamic-import-node'),
>             ],
>           },
>         },
>       }
>     },
>   )
> }
> ```
>
> 基于以上设计，对于 SSR 应用的编译流程（需要编译适配 Node.js 环境）使用方法为：
>
> 复制代码
>
> ```
> // webpack.config.js
> const target = process.env.BUILD_TARGET // 'web' | 'node'
> module.exports = {
>   target,
>   module: {
>     rules: [
>       {
>         test: /\.js$/,
>         oneOf: [
>           {
>             exclude: /node_modules/,
>             loader: 'babel-loader',
>             options: {
>               cacheDirectory: true,
>               presets: [['@lucas/babel-preset/app', {target}]],
>             },
>           },
>           {
>             loader: 'babel-loader',
>             options: {
>               cacheDirectory: true,
>               configFile: false,
>               presets: [['@lucas/babel-preset/dependencies', {target}]],
>               compact: false,
>             },
>           },
>         ],
>       },
>     ],
>   },
> }
> ```
>
> 我们同样按照`node_modules`进行了区分，对于`node_modules`第三方依赖，使用`@lucas/babel-preset/dependencies`预设，同时传入`target`参数。对于非`node_modules`的业务代码，使用`@lucas/babel-preset/app`这个预设，同时设定相应环境 target，`@lucas/babel-preset/app`内容为：
>
> 复制代码
>
> ```
> const path = require('path')
> const {declare} = require('@babel/helper-plugin-utils')
> const getAbsoluteRuntimePath = () => {
>   return path.dirname(require.resolve('@babel/runtime/package.json'))
> }
> module.exports = ({
>   targets,
>   ignoreBrowserslistConfig = false,
>   forceAllTransforms = false,
>   transformRuntime = true,
>   absoluteRuntime = false,
>   supportsDynamicImport = false,
> } = {}) => {
>   return declare(
>     (
>       api,
>       {
>         modules = 'auto',
>         absoluteRuntimePath = getAbsoluteRuntimePath(),
>         react = true,
>         presetReactOptions = {},
>       },
>     ) => {
>       api.assertVersion(7)
>       return {
>         presets: [
>           [
>             require('@babel/preset-env').default,
>             {
>               useBuiltIns: false,
>               modules,
>               targets,
>               ignoreBrowserslistConfig,
>               forceAllTransforms,
>               exclude: ['transform-typeof-symbol'],
>             },
>           ],
>           react && [
>             require('@babel/preset-react').default,
>             {useBuiltIns: true, runtime: 'automatic', ...presetReactOptions},
>           ],
>         ].filter(Boolean),
>         plugins: [
>           transformRuntime && [
>             require('@babel/plugin-transform-runtime').default,
>             {
>               useESModules: 'auto',
>               absoluteRuntime: absoluteRuntime ? absoluteRuntimePath : false,
>             },
>           ],
>           // https://github.com/facebook/create-react-app/issues/4263
>           [
>             require('@babel/plugin-proposal-class-properties').default,
>             {loose: true},
>           ],
>           require('@babel/plugin-syntax-dynamic-import').default,
>           !supportsDynamicImport &&
>             !api.caller(caller => caller && caller.supportsDynamicImport) &&
>             require('babel-plugin-dynamic-import-node'),
>           [
>             require('@babel/plugin-proposal-object-rest-spread').default,
>             {loose: true, useBuiltIns: true},
>           ],
>           require('@babel/plugin-proposal-nullish-coalescing-operator').default,
>           require('@babel/plugin-proposal-optional-chaining').default,
>         ].filter(Boolean),
>         env: {
>           development: {
>             presets: [
>               react && [
>                 require('@babel/preset-react').default,
>                 {
>                   useBuiltIns: true,
>                   development: true,
>                   runtime: 'automatic',
>                   ...presetReactOptions,
>                 },
>               ],
>             ].filter(Boolean),
>           },
>           test: {
>             presets: [
>               [
>                 require('@babel/preset-env').default,
>                 {
>                   useBuiltIns: false,
>                   targets: {node: 'current'},
>                   ignoreBrowserslistConfig: true,
>                   exclude: ['transform-typeof-symbol'],
>                 },
>               ],
>               react && [
>                 require('@babel/preset-react').default,
>                 {
>                   useBuiltIns: true,
>                   development: true,
>                   runtime: 'automatic',
>                   ...presetReactOptions,
>                 },
>               ],
>             ].filter(Boolean),
>             plugins: [
>               [
>                 require('@babel/plugin-transform-runtime').default,
>                 {
>                   useESModules: 'auto',
>                   absoluteRuntime: absoluteRuntimePath,
>                 },
>               ],
>               require('babel-plugin-dynamic-import-node'),
>             ],
>           },
>         },
>       }
>     },
>   )
> }
> ```
>
> 而对于一个公共库，使用方式为：
>
> 复制代码
>
> ```
> // babel.config.js
> module.exports = {
>   presets: ['@lucas/babel-preset/library'],
> }
> ```
>
> 对应`@lucas/babel-preset/library`内容为：
>
> 复制代码
>
> ```
> const create = require('../app/create')
> module.exports = create({
>   targets: {node: 'current'},
>   ignoreBrowserslistConfig: true,
>   supportsDynamicImport: true,
> })
> ```
>
> 这里的预设会将公共库按照当前 Node.js 环境标准编译。如果需要将该公共库编译降级到 ES5，需要使用`@lucas/babel-preset/library/compact`内容为：
>
> 复制代码
>
> ```
> const create = require('../app/create')
> module.exports = create({
>   ignoreBrowserslistConfig: true,
>   supportsDynamicImport: true,
> })
> ```
>
> 这里的`../app/create.js`即为上述`@lucas/babel-preset/app`内容。
>
> 我们通过图示来表述整体架构：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/8C/C6/CgqCHl_0Bs2AS-oEABN0rwjVVsY144.png)
>
> 需要说明以下内容。
>
> **1. @lucas/babel-preset/app：**
>  应用项目使用，编译项目代码。SSR 项目可以配置参数 target: 'web' | 'node'。**默认支持 JSX 语法，并支持一些常用的语法提案**(如 class properties)。
>
> - web：开启全部 transforms
> - node：编译到 node: 'current'
>
> **2. @lucas/babel-preset/dependencies**：应用项目使用，编译 node_modules。SSR 项目可以配置参数 target: 'web' | 'node'。**只支持当前 ES 规范包含的语法，不包含 JSX 语法及提案中的语法支持**。
>
> - web：开启全部 transforms
> - node：编译到 node: 'current'
>
> **3. @lucas/babel-preset/library：** 公共库项目使用。用于 prepare 阶段的 Babel 编译。(如 ./src 通过 Babel 编译到 ./lib)。**默认支持 JSX 语法，并支持一些常用的语法提案**(如 class properties)。编译到 node: 'current'。如果需要将 library 编译为 ES5，需要使用 @lucas/babel-preset/library/compat；library 打包编译为 UMD，使用 @lucas/babel-preset/app。
>
> 上述设计，我参考了[facebook/create-react-app](https://github.com/facebook/create-react-app/blob/master/packages/babel-preset-react-app/create.js)部分内容，建议你阅读源码，并结合注释理解其细节，比如[对于 transform-typeof-symbol 的编译](https://github.com/facebook/create-react-app/blob/master/packages/babel-preset-react-app/create.js#L87)：
>
> 复制代码
>
> ```
> (isEnvProduction || isEnvDevelopment) && [
>     // 最新稳定的 ECMAScript 特性
>     require('@babel/preset-env').default,
>     {
>       useBuiltIns: 'entry',
>       corejs: 3,
>       // 排除 transform-typeof-symbol，这会让编译过慢
>       exclude: ['transform-typeof-symbol'],
>     },
>   ],
> ```
>
> 在使用`@babel/preset-env`时，使用了`useBuiltIns: 'entry'`设置 polyfills，同时将`@babel/plugin-transform-typeof-symbol`排除在外，这是因为`@babel/plugin-transform-typeof-symbol`会劫持 typeof 特性，使得代码运行时变慢。相关讨论可参见 [issue](https://github.com/facebook/create-react-app/pull/5278)。
>
> 最后，这里的预设规范并不代表完全的最佳实践，而是以 React 技术栈风格，统一一个企业级公共库和接入准则，Babel 编译预设可以按照实际项目和企业的需要进行设计，这里更多是一个启迪的作用。
>
> ### 总结
>
> 这一讲我们从一个“线上问题”出发，剖析了公共库和应用方的不同编译理念，并通过设计一个万能 Babel 预设，阐明了公共库的编译和应用的使用需要密切配合，才能在当前前端生态中保障一个更合理的基础建设根基。相关知识并未完结，我们将在下一讲中，从零打造一个公共库来实践说明相关理论。
>
> ![前端基建 金句.png](https://s0.lgstatic.com/i/image2/M01/04/A4/Cip5yF_0OaOAPb48AAf5-twsvFI497.png)