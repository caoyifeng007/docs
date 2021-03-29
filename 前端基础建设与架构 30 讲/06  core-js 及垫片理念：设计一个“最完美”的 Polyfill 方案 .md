[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5911)



> 即便你不知道 [core-js](https://github.com/zloirock/core-js)，也一定在项目中直接或间接地使用过它。core-js 是一个 JavaScript 标准库，它包含了 ECMAScript 2020 在内的多项特性的 polyfills，以及 ECMAScript 在 proposals 阶段的特性、WHATWG/W3C 新特性等。因此它是一个现代化前端项目的“标准套件”。
>
> 除了 core-js 本身的重要性，它的实现理念、设计方式都值得我们学习。事实上，core-js 是一扇大门：
>
> - 通过 core-js，我们可以窥见**前端工程化**的方方面面；
> - core-js 又和 Babel 深度绑定，因此学习 core-js，也能帮助开发者**更好地理解 babel 生态**，进而加深对前端生态的理解；
> - 通过对 core-js 的解析，我们正好可以梳理前端一个极具特色的概念——**polyfill（垫片/补丁）**。
>
> 这一讲，就让我们深入谈谈以上内容。
>
> ### core-js 工程一览
>
> core-js 是一个由 [Lerna](https://github.com/lerna/lerna) 搭建的 Monorepo 风格的项目，在它的 [packages](https://github.com/zloirock/core-js/tree/master/packages) 中，我们能看到五个相关包：
>
> - core-js
> - core-js-pure
> - core-js-compact
> - core-js-builder
> - core-js-bundle
>
> 我们先从 core-js 入手。**core-js 实现的基础垫片能力，是整个 core-js 的逻辑核心**。
>
> 比如我们可以按照如下代码引入全局 polyfills：
>
> 复制代码
>
> ```
> import 'core-js';
> ```
>
> 或者按照：
>
> 复制代码
>
> ```
> import 'core-js/features/array/from';
> ```
>
> 的方式，按需在业务项目的入口引入某些 polyfills。
>
> core-js 为什么有这么多的 packages 呢？实际上，它们各司其职，又紧密配合，接下来我们来具体分析。
>
> **core-js-pure 提供了不污染全局变量的垫片能力**，比如我们可以按照：
>
> 复制代码
>
> ```
> import _from from 'core-js-pure/features/array/from';
> import _flat from 'core-js-pure/features/array/flat';
> ```
>
> 的方式，来实现独立的导出命名空间，进而避免全局变量的污染。
>
> **core-js-compact 维护了按照**[browserslist](https://github.com/browserslist/browserslist)**规范的垫片需求数据**，来帮助我们找到“符合目标环境”的 polyfills 需求集合，比如以下代码：
>
> 复制代码
>
> ```
> const {
>   list, // array of required modules
>   targets, // object with targets for each module
> } = require('core-js-compat')({
>   targets: '> 2.5%'
> });
> ```
>
> 就可以筛选出全球使用份额大于 2.5% 的浏览器范围，并提供在这个范围下需要支持的垫片能力。
>
> **core-js-builder 可以结合 core-js-compact 以及 core-js，并利用 webpack 能力，根据需求打包出 core-js 代码**。比如：
>
> 复制代码
>
> ```
> require('core-js-builder')({
>   targets: '> 0.5%',
>   filename: './my-core-js-bundle.js',
> }).then(code => {}).catch(error => {});
> ```
>
> 将会把符合需求的 core-js 垫片打包到`my-core-js-bundle.js`文件当中。整个流程可以用代码演示为：
>
> 复制代码
>
> ```
> require('./packages/core-js-builder')({ filename: './packages/core-js-bundle/index.js' }).then(done).catch(error => {
>   // eslint-disable-next-line no-console
>   console.error(error);
>   process.exit(1);
> });
> ```
>
> 总之，根据分包的设计，我们能发现，**core-js 将自身能力充分解耦，提供出的多个包都可以被其他项目所依赖**。比如：
>
> - core-js-compact 可以被 Babel 生态使用，由 Babel 分析出根据环境需要按需加载的垫片；
> - core-js-builder 可以被 Node.js 服务使用，构建出不同场景的垫片包。
>
> 宏观上的设计，体现了工程复用能力。下面我们通过一个微观 polyfill 案例，从一个具体的垫片实现，进一步加深理解。
>
> ### 如何复用一个 Polyfill 实现
>
> [Array.prototype.every](https://tc39.es/ecma262/#sec-array.prototype.every) 是一个常见且常用的数组原型上的方法。该方法用于测试一个数组内所有元素是否都能通过某个指定函数的测试，并最终返回一个布尔值来表示测试是否通过。它的浏览器兼容性[如下图](https://www.caniuse.com/?search=array.prototype.every)所示：
>
> ![Drawing 0.png](https://s0.lgstatic.com/i/image/M00/8C/4F/Ciqc1F_q7bKAcYXcAALU37lw2JY310.png)
>
> Array.prototype.every 的函数签名如下：
>
> 复制代码
>
> ```
> arr.every(callback(element[, index[, array]])[, thisArg])
> ```
>
> 对于一个有经验的前端程序员来说，如果浏览器不支持 Array.prototype.every，手写实现一个 Array.prototype.every 的 polyfill 并不困难，下面是 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/every) 的一个实现：
>
> 复制代码
>
> ```
> if (!Array.prototype.every) {
>   Array.prototype.every = function(callbackfn, thisArg) {
>     'use strict';
>     var T, k;
>     if (this == null) {
>       throw new TypeError('this is null or not defined');
>     }
>     var O = Object(this);
>     var len = O.length >>> 0;
>     if (typeof callbackfn !== 'function') {
>       throw new TypeError();
>     }
>     if (arguments.length > 1) {
>       T = thisArg;
>     }
>     k = 0;
>     while (k < len) {
>       var kValue;
>       if (k in O) {
>         kValue = O[k];
>         var testResult = callbackfn.call(T, kValue, k, O);
>         if (!testResult) {
>           return false;
>         }
>       }
>       k++;
>     }
>     return true;
>   };
> }
> ```
>
> 核心思路很好理解：我们通过遍历数组，对数组的每一项调用 CALLBACK 求值进行返回是否通过测试。但是站在工程化的角度，从 core-js 这样一个大型项目的视角出发，就不是这么简单了。比如，我们知道 core-js-pure 不同于 core-js，它提供了**不污染命名空间**的引用方式，因此上述 Array.prototype.every 的 polyfill 核心逻辑实现，就需要被 core-js-pure 和 core-js 同时引用，只要**区分最后导出的方式**即可，那么按照这个思路，我们如何实现最大限度的复用呢？
>
> 实际上，Array.prototype.every 的 polyfill 核心逻辑实现在`./packages/core-js/modules/es.array.every.js`中，[源码](https://github.com/zloirock/core-js/blob/master/packages/core-js/modules/es.array.every.js)如下：
>
> 复制代码
>
> ```
> 'use strict';
> var $ = require('../internals/export');
> var $every = require('../internals/array-iteration').every;
> var arrayMethodIsStrict = require('../internals/array-method-is-strict');
> var arrayMethodUsesToLength = require('../internals/array-method-uses-to-length');
> var STRICT_METHOD = arrayMethodIsStrict('every');
> var USES_TO_LENGTH = arrayMethodUsesToLength('every');
> $({ target: 'Array', proto: true, forced: !STRICT_METHOD || !USES_TO_LENGTH }, {
>   every: function every(callbackfn /* , thisArg */) {
>     // 调用 $every 方法
>     return $every(this, callbackfn, arguments.length > 1 ? arguments[1] : undefined);
>   }
> });
> ```
>
> 对应`$every`[源码](https://github.com/zloirock/core-js/blob/master/packages/core-js/internals/array-iteration.js#L58)：
>
> 复制代码
>
> ```
> var bind = require('../internals/function-bind-context');
> var IndexedObject = require('../internals/indexed-object');
> var toObject = require('../internals/to-object');
> var toLength = require('../internals/to-length');
> var arraySpeciesCreate = require('../internals/array-species-create');
> var push = [].push;
> // 对 `Array.prototype.{ forEach, map, filter, some, every, find, findIndex }` 等方法进行接模拟和接入
> var createMethod = function (TYPE) {
>   // 通过魔法数字来表示具体需要对哪种方法进行模拟
>   var IS_MAP = TYPE == 1;
>   var IS_FILTER = TYPE == 2;
>   var IS_SOME = TYPE == 3;
>   var IS_EVERY = TYPE == 4;
>   var IS_FIND_INDEX = TYPE == 6;
>   var NO_HOLES = TYPE == 5 || IS_FIND_INDEX;
>   return function ($this, callbackfn, that, specificCreate) {
>     var O = toObject($this);
>     var self = IndexedObject(O);
>     // 通过 bind 方法创建一个 boundFunction，保留 this 指向
>     var boundFunction = bind(callbackfn, that, 3);
>     var length = toLength(self.length);
>     var index = 0;
>     var create = specificCreate || arraySpeciesCreate;
>     var target = IS_MAP ? create($this, length) : IS_FILTER ? create($this, 0) : undefined;
>     var value, result;
>     // 遍历循环并执行回调方法
>     for (;length > index; index++) if (NO_HOLES || index in self) {
>       value = self[index];
>       result = boundFunction(value, index, O);
>       if (TYPE) {
>         if (IS_MAP) target[index] = result; // map
>         else if (result) switch (TYPE) {
>           case 3: return true;              // some
>           case 5: return value;             // find
>           case 6: return index;             // findIndex
>           case 2: push.call(target, value); // filter
>         } else if (IS_EVERY) return false;  // every
>       }
>     }
>     return IS_FIND_INDEX ? -1 : IS_SOME || IS_EVERY ? IS_EVERY : target;
>   };
> };
> module.exports = {
>   forEach: createMethod(0),
>   map: createMethod(1),
>   filter: createMethod(2),
>   some: createMethod(3),
>   every: createMethod(4),
>   find: createMethod(5),
>   findIndex: createMethod(6)
> };
> ```
>
> 同样是使用了遍历的方式，并由`../internals/function-bind-context`提供 this 绑定能力，用魔法常量处理`forEach、map、filter、some、every、find、findIndex`这些数组原型方法的不同方法。
>
> 重点来了，在 core-js 中，作者通过`../internals/export`方法导出实现原型，[源码](https://github.com/zloirock/core-js/blob/master/packages/core-js/internals/export.js)如下：
>
> 复制代码
>
> ```
> module.exports = function (options, source) {
>   var TARGET = options.target;
>   var GLOBAL = options.global;
>   var STATIC = options.stat;
>   var FORCED, target, key, targetProperty, sourceProperty, descriptor;
>   if (GLOBAL) {
>     target = global;
>   } else if (STATIC) {
>     target = global[TARGET] || setGlobal(TARGET, {});
>   } else {
>     target = (global[TARGET] || {}).prototype;
>   }
>   if (target) for (key in source) {
>     sourceProperty = source[key];
>     if (options.noTargetGet) {
>       descriptor = getOwnPropertyDescriptor(target, key);
>       targetProperty = descriptor && descriptor.value;
>     } else targetProperty = target[key];
>     FORCED = isForced(GLOBAL ? key : TARGET + (STATIC ? '.' : '#') + key, options.forced);
>     if (!FORCED && targetProperty !== undefined) {
>       if (typeof sourceProperty === typeof targetProperty) continue;
>       copyConstructorProperties(sourceProperty, targetProperty);
>     }
>     if (options.sham || (targetProperty && targetProperty.sham)) {
>       createNonEnumerableProperty(sourceProperty, 'sham', true);
>     }
>     redefine(target, key, sourceProperty, options);
>   }
> };
> ```
>
> 对应我们的 Array.prototype.every[源码](https://github.com/zloirock/core-js/blob/master/packages/core-js/modules/es.array.every.js)，参数为：`target: 'Array', proto: true`，表明 coe-js 需要在数组 Array 的原型上，以“污染数组原型”的方式来扩展方法。
>
> 而 core-js-pure 则单独维护了一份 export 镜像`../internals/export`，其[源码](https://github.com/zloirock/core-js/blob/master/packages/core-js-pure/override/internals/export.js)我在这里不做过多讲解，你可以在本节内容学习后进一步查看。
>
> 同时，core-js-pure 包中的 Override 文件，实际上是在构建阶段，复制了 packages/core-js/ 内的核心逻辑（[源码在这里](https://github.com/zloirock/core-js/blob/master/Gruntfile.js#L73)），同时提供了复写这些核心 polyfills 逻辑的能力，也是通过构建流程，进行 core-js-pure/override 替换覆盖：
>
> 复制代码
>
> ```
> {
> 	expand: true,
> 	cwd: './packages/core-js-pure/override/',
> 	src: '**',
> 	dest: './packages/core-js-pure',
> }
> ```
>
> 这是一种非常巧妙的“利用构建能力，实现复用”的方案。但我认为，既然是 Monorepo 风格的仓库，也许一种更好的设计是将**core-js 核心 polyfills 再单独拆到一个包中，core-js 和 core-js-pure 分别进行引用**——这种方式更能利用 Monorepo 能力，且减少了构建过程中的魔法处理。
>
> ### 寻找最佳 Polyfill 方案
>
> 前文多次提到了 polyfill/垫片/补丁（下文混用这三种说法），这里我们正式对 polyfill 进行一个定义：
>
> > A polyfill, or polyfiller, is a piece of code (or plugin) that provides the technology that you, the developer, expect the browser to provide natively. Flattening the API landscape if you will.
>
> 简单来说，**polyfill 就是用社区上提供的一段代码，让我们在不兼容某些新特性的浏览器上，使用该新特性**。
>
> 随着前端的发展，尤其是 ECMAScript 的迅速成长以及浏览器的频繁更新换代，前端使用 polyfills 技术的情况屡见不鲜。**那么如何能在工程中，寻找并设计一个“最完美”的 polyfill 方案呢？\**注意，这里的完美指的是\**侵入性最小，工程化、自动化程度最高，业务影响最低**。
>
> 第一种方案：**手动打补丁**。这种方式最为简单直接，也能天然做到“按需打补丁”，但是这不是一种工程化的解决方式，方案原始而难以维护，同时对于 polyfill 的实现要求较高。
>
> 于是，es5-shim 和 es6-shim 等“轮子”出现了，它们伴随着前端开发走过了一段艰辛岁月。但 es5-shim 和 es6-shim 这种笨重的方案很快被 babel-polyfill 取代，babel-polyfill 融合了 core-js 和 regenerator-runtime。
>
> 但如果粗暴地使用 babel-polyfill 一次性全量导入到项目中，不和 @babel/preset-env 等方案结合，babel-polyfill 会将其所包含的所有补丁都应用在项目当中，这样直接造成了项目 size 过大的问题，且存在污染全局变量的潜在问题。
>
> 于是，**babel-polyfill 结合 @babel/preset-env + useBuiltins（entry） + preset-env targets 的方案**如今更为流行，@babel/preset-env 定义了 Babel 所需插件预设，同时由 Babel 根据 preset-env targets 配置的支持环境，自动按需加载 polyfills，使用方式如下：
>
> 复制代码
>
> ```
> {
>   "presets": [
>     ["@babel/env", {
>       useBuiltIns: 'entry',
>       targets: { chrome: 44 }
>     }]
>   ]
> }
> ```
>
> 这样我们在工程代码入口处的：
>
> 复制代码
>
> ```
> import '@babel/polyfill';
> ```
>
> 会被编译为：
>
> 复制代码
>
> ```
> import "core-js/XXXX/XXXX";
> import "core-js/XXXX/XXXXX";
> ```
>
> 这样的方式省力省心。也是 core-js 和 Babel 深度绑定并结合的典型案例。
>
> 上文提到了 babel-polyfill 融合了 core-js 和 regenerator-runtime，既然如此，我们也可以不使用 babel-polyfill，而直接使用 core-js。这里我根据 [babel-polyfill vs core-js vs es5-shim vs es6-shim](https://www.npmtrends.com/babel-polyfill-vs-core-js-vs-es5-shim-vs-es6-shim) 的使用频率情况，进行比对，如下图所示：
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/8C/4F/Ciqc1F_q7dKAanOXAAHwZCycIb4392.png)
>
> babel-polyfill vs core-js vs es5-shim vs es6-shim 使用频率对比图
>
> 我们看到，**core-js 使用最多**，这是因为它既可以在项目中单独使用，也可以和 Babel 绑定，作为更低层的依赖出现。
>
> 我们再思考一个问题：如果某个业务代码中，并没有用到配置环境填充的 polyfills，那么这些 polyfills 的引入依然出现了引用浪费的情况。实际上环境需要是一回事儿，代码是否需要却是另一回事儿。比如，我的 MPA（多页面应用）项目需要提供 Promise Polyfill，但是某个业务页面中，并没有使用 Promise 特性，理想情况并不需要在当前页面中引入 Promise Polyfill bundle。
>
> 针对这个问题，@babel/preset-env + useBuiltins（usage） + preset-env targets 方案就出现了，**注意这里的 useBuiltins 配置为 usage，它可以真正根据代码情况，分析 AST（抽象语法树）进行更细粒度的按需引用**。但是这种基于静态编译的按需加载补丁也是相对的，因为 JavaScript 是一种弱规则的动态语言，比如这样的代码：`foo.includes(() => {//...})`，我们无法判断出这里的 `includes` 是数组原型方法还是字符串原型方法，因此一般做法只能将数组原型方法和字符串原型方法同时打包为 polyfill bundle。
>
> 除了在打包构建阶段植入 polyfill 以外，另外一个思路是“在线动态打补丁”，这种方案以 [Polyfill.io](https://polyfill.io/v3/) 为代表，它提供了 CDN 服务，使用者可以按照所需环境，[生成打包链接](https://polyfill.io/v3/url-builder/)：
>
> ![Lark20201230-104425.png](https://s0.lgstatic.com/i/image/M00/8C/5A/Ciqc1F_r6aWAUh6OAAGLnnSGGnY780.png)
>
> 如`https://polyfill.io/v3/polyfill.min.js?features=es2015`，在业务中我们可以直接引入 polyfills bundle：
>
> 复制代码
>
> ```
> <script src="https://polyfill.io/v3/polyfill.min.js?features=es2015"></script>
> ```
>
> **在高版本浏览器上，可能会返回空内容，因为该浏览器已经支持了 ES2015 特性。如果在低版本浏览器上，将会得到真实的 polyfills bundle**。
>
> 从工程化的角度来说，**一个趋于完美的 polyfill 设计应该满足的核心原则是按需加载补丁**，这个按需加载主要包括两方面：
>
> - 按照用户终端环境
> - 按照业务代码使用情况
>
> 因为按需加载补丁，意味着更小的 bundle size，直接决定了应用的性能。
>
> ### 总结
>
> 从对前端项目的影响来讲，core-js 不只是一个 polyfill 仓库；从前端技术设计的角度来看，core-js 也能让我们获得更多启发和灵感。这一讲我们分析了 core-js 的设计实现，并由此延展出了工程中 polyfill 设计的方方面面。但依然留下了几个问题：
>
> - core-js 和 Babel 生态绑定在一起，它们到底有什么联系，如何实现密切配合？
> - core-js 如何和 @babel/preset-env + useBuiltins（usage）配合，并利用 AST 技术，实现代码级别的按需引入？
>
> 前端基础建设和工程化，每一个环节都相互关联，我们将会在“梳理混乱的 Babel，不再被编译报错困扰”一讲中，继续进行更多探索。