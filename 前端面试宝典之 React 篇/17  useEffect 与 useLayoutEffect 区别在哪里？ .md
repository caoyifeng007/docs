[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5807)



> 在 React 的面试中会对 Hooks API 进行一个区分度考察，重点往往会落在 useEffect 与 useLayoutEffect 上。很有意思，光从名字来看，它们就很相像，所以被点名的概率就很高。这一讲我们来重点讲解下这部分内容。
>
> ### 审题
>
> 在第 04 讲[“类组件与函数组件有什么区别呢？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0#/detail/pc?id=5794)中讲过该类型的题目，其中提到**描述区别，就是求同存异的过程。** 那我们可以直接用同样的思路来思考下这道题。
>
> 先挖掘 useEffect 与 useLayoutEffect 的共性，它们被用于解决什么问题，其次发掘独特的个性、各自适用的场景、设计原理以及未来趋势等。
>
> ### 承题
>
> 根据以上的分析，该讲所讲解题目的答题思路就有了。
>
> 首先是论述**共同点**：
>
> - **使用方式**，也就是列举使用方式上有什么相似处，共同用于解决什么问题；
> - **运用效果**，使用后的执行效果上有什么相似处。
>
> 其次是**不同点**：
>
> - **使用场景**，两者在使用场景的区分点在哪里，为什么可以或者不可以混用；
> - **独有能力**，什么能力是其独有的，而另外一方没有的；
> - **设计原理**，即从内部设计挖掘本质的原因；
> - **未来趋势**，两者在未来的发展趋势上有什么区别，是否存在一方可能在使用频率上胜出或者淡出的情况。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/90/3F/Ciqc1GAKhGuAeJVzAABnKbg5gv0029.png)
>
> ### 破题
>
> #### 共同点
>
> **使用方式**
>
> 如果你读过 React Hooks 的官方文档，你可能会发现这么一段描述：useLayoutEffect 的函数签名与 useEffect 相同。
>
> 那什么是**函数签名呢**？函数签名就像我们在银行账号上签写的个人签名一样，独一无二，具有法律效应。以下面这段代码为例：
>
> 复制代码
>
> ```
> MyObject.prototype.myFunction(value)
> ```
>
> 应用 MDN 的描述，在 JavaScript 中的签名通常包括这样几个部分：
>
> - 该函数是安装在一个名为 MyObject 的[对象](https://developer.mozilla.org/zh-CN/docs/Glossary/Object)上；
> - 该函数安装在 MyObject 的原型上（因此它是一个[实例方法](https://developer.mozilla.org/en-US/docs/Glossary/instance_method)，而不是一个[静态方法/类方法](https://developer.mozilla.org/en-US/docs/Glossary/static_method)）；
> - 该函数的名称是 myFunction；
> - 该函数接收一个叫 value 的参数，且没有进一步定义。
>
> 那为什么说 useEffect 与 useLayoutEffect 函数签名相同呢？它们俩的名字完全不同啊。这是因为在源码中，它们调用的是同一个函数。下面的这段代码是 React useEffect 与 useLayoutEffect 在 ReactFiberHooks.js 源码中的样子。
>
> 复制代码
>
> ```
> // useEffect
> useEffect(
>    create: () => (() => void) | void,
>    deps: Array<mixed> | void | null,
>  ): void {
>    currentHookNameInDev = 'useEffect';
>    mountHookTypesDev();
>    checkDepsAreArrayDev(deps);
>    return mountEffect(create, deps);
>  },
>  
> function mountEffect(
> 	  create: () => (() => void) | void,
> 	  deps: Array<mixed> | void | null,
> 	): void {
> 	  if (__DEV__) {
> 	    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
> 	    if ('undefined' !== typeof jest) {
> 	      warnIfNotCurrentlyActingEffectsInDEV(currentlyRenderingFiber);
> 	    }
> 	  }
> 	  return mountEffectImpl(
> 	    UpdateEffect | PassiveEffect | PassiveStaticEffect,
> 	    HookPassive,
> 	    create,
> 	    deps,
> 	  );
> 	}
>  
> // useLayoutEffect
> useLayoutEffect(
>    create: () => (() => void) | void,
>    deps: Array<mixed> | void | null,
>  ): void {
>    currentHookNameInDev = 'useLayoutEffect';
>    mountHookTypesDev();
>    checkDepsAreArrayDev(deps);
>    return mountLayoutEffect(create, deps);
>  },
>  
> function mountLayoutEffect(
> 	  create: () => (() => void) | void,
> 	  deps: Array<mixed> | void | null,
> 	): void {
> 	  return mountEffectImpl(UpdateEffect, HookLayout, create, deps);
> }
> ```
>
> 可以看出：
>
> - useEffect 先调用 mountEffect，再调用 mountEffectImpl；
> - useLayoutEffect 会先调用 mountLayoutEffect，再调用 mountEffectImpl。
>
> 那么你会发现最终调用的都是同一个名为 mountEffectImpl 的函数，入参一致，返回值也一致，所以函数签名是相同的。
>
> 从代码角度而言，虽然是两个函数，但使用方式是完全一致的，甚至一定程度上可以相互替换。
>
> **运用效果**
>
> 从运用效果上而言，useEffect 与 useLayoutEffect 两者都是**用于处理副作用**，这些副作用包括改变 DOM、设置订阅、操作定时器等。在函数组件内部操作副作用是不被允许的，所以需要使用这两个函数去处理。
>
> 虽然看起来很像，但在执行效果上仍然有些许差异。React 官方团队甚至直言，如果不能掌握 useLayoutEffect，不妨直接使用 useEffect。在使用 useEffect 时遇到了问题，再尝试使用 useLayoutEffect。
>
> #### 不同点
>
> **使用场景**
>
> 虽然官方团队给出了一个看似友好的建议，但我们并不能将这样模糊的结果作为正式答案回复给面试官。所以两者的差异在哪里？我们不如用代码来说明。下面通过一个案例来讲解两者的区别。
>
> 先使用 useEffect 编写一个组件，在这个组件里面包含了两部分：
>
> - 组件展示内容，也就是 className 为 square 的部分会展示一个圆圈；
> - 在 useEffect 中操作修改 square 的样式，将它的样式重置到页面正中间。
>
> 复制代码
>
> ```
> import React, { useEffect } from "react";
> import "./styles.css";
> export default () => {
>   useEffect(() => {
>     const greenSquare = document.querySelector(".square");
>     greenSquare.style.transform = "translate(-50%, -50%)";
>     greenSquare.style.left = "50%";
>     greenSquare.style.top = "50%";
>   });
>   return (
>     <div className="App">
>       <div className="square" />
>     </div>
>   );
> };
> ```
>
> 下面再补充一下样式文件，其中 App 样式中规中矩设置间距，square 样式主要设置圈儿的颜色与大小，接下来就可以看看它的效果了。
>
> 复制代码
>
> ```
> .App {
>   text-align: center;
>   margin: 0;
>   padding: 0;
> }
> .square {
>   width: 100px;
>   height: 100px;
>   position: absolute;
>   top: 50px;
>   left: 0;
>   background: red;
>   border-radius: 50%;
> }
> ```
>
> 然后执行代码， 你就会发现红圈在渲染后出现了肉眼可见的瞬移，一下飘到了中间。
>
> ![GIF1.gif](https://s0.lgstatic.com/i/image2/M01/08/32/Cip5yGAKhLOADIYUAAIoKyWYNqU863.gif)
>
> 那如果使用 useLayoutEffect 又会怎么样呢？其他代码都不需要修改，只需要像下面这样，将 useEffect 替换为 useLayoutEffect：
>
> 复制代码
>
> ```
> import React, { useLayoutEffect } from "react";
> import "./styles.css";
> export default () => {
>   useLayoutEffect(() => {
>     const greenSquare = document.querySelector(".square");
>     greenSquare.style.transform = "translate(-50%, -50%)";
>     greenSquare.style.left = "50%";
>     greenSquare.style.top = "50%";
>   });
>   return (
>     <div className="App">
>       <div className="square" />
>     </div>
>   );
> };
> ```
>
> 接下来再看执行的效果，你会发现红圈是静止在页面中央，仿佛并没有使用代码强制调整样式的过程。
>
> ![GIF2.gif](https://s0.lgstatic.com/i/image2/M01/08/34/CgpVE2AKhLyANGs2AAIwo7CyA_E780.gif)
>
> 虽然在实际的项目中，我们并不会这么粗暴地去调整组件样式，但这个案例足以说明两者的区别与使用场景。在 React 社区中最佳的实践是这样推荐的，大多数场景下可以直接使用**useEffect**，但是如果你的代码引起了页面闪烁，也就是引起了组件突然改变位置、颜色及其他效果等的情况下，就推荐使用**useLayoutEffect**来处理。那么总结起来就是如果有直接操作 DOM 样式或者引起 DOM 样式更新的场景更推荐使用 useLayoutEffect。
>
> 那既然内部都是调用同一个函数，为什么会有这样的区别呢？在探讨这个问题时就需要从 Hooks 的设计原理说起了。
>
> **设计原理**
>
> 首先可以看下这个图：
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image2/M01/08/34/CgpVE2AKhP6AFNRnAAB9M55aj8I408.png)
>
> 这个图表达了什么意思呢？首先所有的 Hooks，也就是 useState、useEffect、useLayoutEffect 等，都是导入到了 Dispatcher 对象中。在调用 Hook 时，会通过 Dispatcher 调用对应的 Hook 函数。所有的 Hooks 会按顺序存入对应 Fiber 的状态队列中，这样 React 就能知道当前的 Hook 属于哪个 Fiber，这里就是[上一讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0#/detail/pc?id=5806)所提到的**Hooks 链表**。但 Effect Hooks 会有些不同，它涉及了一些额外的处理逻辑。每个 Fiber 的 Hooks 队列中保存了 effect 节点，而每个 effect 的类型都有可能不同，需要在合适的阶段去执行。
>
> 那么 LayoutEffect 与普通的 Effect 都是 effect，但标记并不一样，所以在调用时，就会有些许不同。回到前面的底层代码，你会发现只有第一个参数和第二个参数是不一样的，其中 UpdateEffect、PassiveEffect、PassiveStaticEffect 就是 Fiber 的标记；HookPassive 和 HookLayout 就是当前 Effect 的标记。如下代码所示：
>
> 复制代码
>
> ```
> // useEffect 调用的底层函数
> function mountEffect(
> 	  create: () => (() => void) | void,
> 	  deps: Array<mixed> | void | null,
> 	): void {
> 	  if (__DEV__) {
> 	    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
> 	    if ('undefined' !== typeof jest) {
> 	      warnIfNotCurrentlyActingEffectsInDEV(currentlyRenderingFiber);
> 	    }
> 	  }
> 	  return mountEffectImpl(
> 	    UpdateEffect | PassiveEffect | PassiveStaticEffect,
> 	    HookPassive,
> 	    create,
> 	    deps,
> 	  );
> 	}
> // useLayoutEffect 调用的底层函数
> function mountLayoutEffect(
> 	  create: () => (() => void) | void,
> 	  deps: Array<mixed> | void | null,
> 	): void {
> 	  return mountEffectImpl(UpdateEffect, HookLayout, create, deps);
> 	}
> ```
>
> 标记为 HookLayout 的 effect 会在所有的 DOM 变更之后同步调用，所以可以使用它来读取 DOM 布局并同步触发重渲染。但既然是同步，就有一个问题，计算量较大的耗时任务必然会造成阻塞，所以这就需要根据实际情况酌情考虑了。如果非必要情况下，使用标准的 useEffect 可以避免阻塞。这段代码在 react/packages/react-reconciler/src/ReactFiberCommitWork.new.js 中，有兴趣的同学可以研读一下。
>
> ### 答题
>
> 那么以上就是本讲的全部知识点了，内容并不太多，重点主要在于**分析思路**。那么下面就可以进入答题环节了。
>
> > useEffect 与 useLayoutEffect 的区别在哪里？这个问题可以分为两部分来回答，共同点与不同点。
> >
> > 它们的共同点很简单，底层的函数签名是完全一致的，都是调用的 mountEffectImpl，在使用上也没什么差异，基本可以直接替换，也都是用于处理副作用。
> >
> > 那不同点就很大了，useEffect 在 React 的渲染过程中是被异步调用的，用于绝大多数场景，而 LayoutEffect 会在所有的 DOM 变更之后同步调用，主要用于处理 DOM 操作、调整样式、避免页面闪烁等问题。也正因为是同步处理，所以需要避免在 LayoutEffect 做计算量较大的耗时任务从而造成阻塞。
> >
> > 在未来的趋势上，两个 API 是会长期共存的，暂时没有删减合并的计划，需要开发者根据场景去自行选择。React 团队的建议非常实用，如果实在分不清，先用 useEffect，一般问题不大；如果页面有异常，再直接替换为 useLayoutEffect 即可。
>
> ![Drawing 6.png](https://s0.lgstatic.com/i/image2/M01/08/32/Cip5yGAKhRCAX99HAAD0YKYP40c980.png)
>
> ### 总结
>
> 本题仍然是一个讲区别的点，所以在整体思路上找相同与不同就可以。
>
> 这里还有一个很有意思的地方，React 的函数命名追求“望文生义”的效果，这里不是贬义，它在设计上就是希望你从名字猜出真实的作用。比如 componentDidMount、componentDidUpdate 等等虽然名字冗长，但容易理解。从 LayoutEffect 这样一个命名就能看出，它想解决的也就是页面布局的问题。
>
> 那么在实际的开发中，还有哪些你觉得不太容易理解的 Hooks？或者容易出错的 Hooks？不妨在留言区留言，我会与你一起交流讨论。
>
> 这一讲就到这了，在下一讲中，将主要介绍 React Hooks 的设计模式，到时见。