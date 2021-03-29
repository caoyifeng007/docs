[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5801)



> 解释 React 的渲染流程是一道面试中的高频题，这讲我会带你探讨这个问题。
>
> ### 破题
>
> 你知道面试官是怎样通过一个回答评判你的能力层次吗：
>
> - 如果只是简单的复述流程，缺乏重点侧写，那只是到了**知道**的程度；
> - 如果解释清楚了设计理念，并能将核心流程穿插在具象化的抽象概念中进行描述，那才是真正吃透了理念，具备了基本的架构能力；
> - 在上一点基础上，加上自己的理论心得、工程实践，辅以具体的落地成果，那在能力评定上，肯定是架构师以上的级别了。
>
> 虽然我们的能力还没到架构师的级别，但在**思考总结**与**阐述观点**上要往这个方向走，并在实践中不断锻炼和改进自己的思考和表达方式，做到“**讲话有重点**，**层次要分明**”。
>
> 关于本讲的问题，求职者就很容易跑偏，一个劲儿地背诵渲染流程中涉及的函数。我不止一次遇到过这样的面试场景，这样的回答非常冗长，缺乏对关键内容的提炼升华，需要听者自行完成观点的剥离，所以面试官很难听进去。这种情况就违背了“讲话有重点，层次要分明”的原则。
>
> 在第 4 讲“[类组件与函数组件有什么区别呢？](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566#/detail/pc?id=5794)”中提过一个**突出重点**的方式，即通过主线串联整个分散的论点。本题也可以采用同样的方法。
>
> 那就需要先清楚什么是渲染过程中的重点？以及渲染过程的层次该如何划分？
>
> ### 承题
>
> 整合前面几讲提到的关于渲染流程的知识点：React 渲染节点的挂载、React 组件的生命周期、setState 触发渲染更新、diff 策略与 patch 方案。你会发现渲染流程中包含的内容很繁杂，有各种大大小小需要处理的事，而这些事用计算机科学中的专业术语来说，就是**事务**。事务是无法被分割的，必须作为一个整体执行完成，不可能存在部分完成的事务。所以这里需要注意，事务具有**原子性**，不可再分。
>
> 了解了事务的基本概念后，还需要知道事务是通过**调度**的方式协调执行的。
>
> 虽然有了全局规划的调度，也有了具体的事务，但工作仍然不是一蹴而就的，在实际的工作中，我们往往是按**阶段进行划分**的，比如项目启动阶段、项目开发阶段、项目提测阶段等。这样的划分模式以里程碑为节点，可以拆分子项，降低整体的复杂度，所以在渲染流程中也存在这样的阶段划分。
>
> 这样，以不同阶段的事务与策略为主线，就可以做到“讲话有重点”了；以阶段划分节点，就可以做到“层次要分明”了。
>
> 初步的答题框架就形成了。
>
> ![图片1.png](https://s0.lgstatic.com/i/image/M00/8C/7A/Ciqc1F_tr2KAUDQKAADuU-A-myg780.png)
>
> ### 入门
>
> 在逐级梳理之前，我们先讲一个在渲染流程中绝对绕不开的概念——协调。
>
> #### 协调
>
> 协调，在 React 官方博客的原文中是 Reconciler，它的本意是“和解者，调解员”。当你搜索与 Reconciler 相关的图片时，会出现很多握手、签字、相互拥抱的图片。
>
> 协调是怎么跟 React 扯上关系的呢？React 官方文档在介绍协调时，是这样说的：
>
> > React 提供的声明式 API 让开发者可以在对 React 的底层实现没有具体了解的情况下编写应用。在开发者编写应用时虽然保持相对简单的心智，但开发者无法了解内部的实现机制。本文描述了在实现 React 的 “diffing” 算法中我们做出的设计决策以保证组件满足更新具有可预测性，以及在繁杂业务下依然保持应用的高性能性。
>
> 从上文中我们可以看出，Reconciler 是协助 React 确认状态变化时要更新哪些 DOM 元素的 diff 算法，这看上去确实有点儿调解员的意思。这是狭义上的 Reconciler，也是[第 10 讲“与其他框架相比，React 的 diff 算法有何不同？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566#/detail/pc?id=5800)中提过的内容。
>
> 而在 React 源码中还有一个叫作 reconcilers 的模块，它通过抽离公共函数与 diff 算法使声明式渲染、自定义组件、state、生命周期方法和 refs 等特性实现跨平台工作。
>
> Reconciler 模块以 React 16 为分界线分为两个版本。
>
> - **Stack Reconciler**是 React 15 及以前版本的渲染方案，其核心是以**递归的方式**逐级调度栈中子节点到父节点的渲染。
> - **Fiber Reconciler**是 React 16 及以后版本的渲染方案，它的核心设计是**增量渲染**（incremental rendering），也就是将渲染工作分割为多个区块，并将其分散到多个帧中去执行。它的设计初衷是提高 React 在动画、画布及手势等场景下的性能表现。
>
> 两者的性能差距究竟有多大呢？既然主打的是高性能场景，那么在一般的中后台页面、前端 H5 下，很难看出两者之间的性能差距。但你在尝试这个 [demo](https://claudiopro.github.io/react-fiber-vs-stack-demo) 之后，就能明显地体会到了。
>
> #### 渲染
>
> 为了更好地理解两者之间的差异，我们需要先梳理一遍 Stack Reconciler。
>
> **Stack Reconciler**
>
> Stack Reconciler 没有单独的包，并没有像 Fiber Reconclier 一样抽取为独立的[React-Reconciler 模块](https://github.com/facebook/react/tree/16.3-dev/packages/react-reconciler)。但这并不妨碍它成为一个经典的设计。在 React 的官方文档中，是通过伪代码的形式介绍其[实现方案](https://react.html.cn/docs/implementation-notes.html)的。与官方文档略有不同，下面我会介绍一些真实代码的信息。
>
> **挂载**
>
> 这里的挂载与生命周期一讲中的挂载不同，它是将整个 React 挂载到 ReactDOM.render 之上，就像以下代码中的 App 组件挂载到 root 节点上一样。
>
> 复制代码
>
> ```
> class App extends React.Component {
>   render() {
>     return (
>         <div>Hello World</div>
>       )
>   }
> } 
> ReactDOM.render(<App />, document.getElementById('root'))
> ```
>
> 还记得在 JSX 一讲中所提到的吗？JSX 会被 Babel 编译成 React.creatElemnt 的形式：
>
> 复制代码
>
> ```
> ReactDOM.render(React.creatElement(App), document.getElementById('root'))
> ```
>
> 但一定要记住，这项工作发生在本地的 Node 进程中，而不是通过浏览器中的 React 完成的。在以往的面试中，就有应聘的同学以为 JSX 是通过 React 完成编译，这是完全不正确的。
>
> ReactDOM.render 调用之后，实际上是**透传参数给 ReactMount.render**。
>
> - ReactDOM 是对外暴露的模块接口；
> - 而 ReactMount 是实际执行者，完成初始化 React 组件的整个过程。
>
> 初始化第一步就是通过 React.creatElement 创建 React Element。不同的组件类型会被构建为不同的 Element：
>
> - App 组件会被标记为 type function，作为用户自定义的组件，被 ReactComponentsiteComponent 包裹一次，生成一个对象实例；
> - div 标签作为 React 内部的已知 DOM 类型，会实例化为 ReactDOMComponent；
> - "Hello World" 会被直接判断是否为字符串，实例化为 ReactDOMComponent。
>
> ![图片3.png](https://s0.lgstatic.com/i/image/M00/8C/85/CgqCHl_tr1KABoL9AAELd_UE-Q4687.png)
>
> 这段逻辑在 React 源码中大致是这样的，其中 isInternalComponentType 就是判断当前的组件是否为内部已知类型。
>
> 复制代码
>
> ```
> if (typeof element.type === 'string') {
>     instance = ReactHostComponent.createInternalComponent(element);
>   } else if (isInternalComponentType(element.type)) {
>     instance = new element.type(element);
>   } else {
>     instance = new ReactCompositeComponentWrapper();
> }
> ```
>
> 到这里仅仅完成了实例化，我们还需要与 React 产生一些联动，比如改变状态、更新界面等。在 setState 一讲中，我们提到在状态变更后，涉及一个变更收集再批量处理的过程。在这里 ReactUpdates 模块就专门**用于批量处理**，而批量处理的前后操作，是由 React 通过建立事务的概念来处理的。
>
> React 事务都是基于 Transaction 类继承拓展。每个 Transaction 实例都是一个封闭空间，保持不可变的任务常量，并提供对应的事务处理接口 。一段事务在 React 源码中大致是这样的：
>
> 复制代码
>
> ```
> mountComponentIntoNode: function(rootID, container) {
>       var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
>       transaction.perform(
>         this._mountComponentIntoNode,
>         this,
>         rootID,
>         container,
>         transaction
>       );
>       ReactComponent.ReactReconcileTransaction.release(transaction);
>  }
> ```
>
> 如果有操作数据库经验的同学，应该看到过相似的例子。React 团队将其从后端领域借鉴到前端是因为事务的设计有以下优势。
>
> - 原子性: 事务作为一个整体被执行，要么全部被执行，要么都不执行。
> - 隔离性: 多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
> - 一致性: 相同的输入，确定能得到同样的执行结果。
>
> 上面提到的事务会调用 ReactCompositeComponent.mountComponent 函数进入 React 组件生命周期，它的源码大致是这样的。
>
> 复制代码
>
> ```
> if (inst.componentWillMount) {
>     inst.componentWillMount();
>     if (this._pendingStateQueue) {
>         inst.state = this._processPendingState(inst.props, inst.context);
>     }
> }
> ```
>
> 首先会判断是否有 componentWillMount，然后初始化 state 状态。当 state 计算完毕后，就会调用在 App 组件中声明的 render 函数。接着 render 返回的结果，会处理为新的 React Element，再走一遍上面提到的流程，不停地往下解析，逐步递归，直到开始处理 HTML 元素。到这里我们的 App 组件就完成了首次渲染。
>
> **更新**
>
> 接下来我们用同样的方式解析下当调用 setState 时会发生什么。setState 时会调用 Component 类中的 enqueueSetState 函数。
>
> 复制代码
>
> ```
> this.updater.enqueueSetState(this, partialState)
> ```
>
> 在执行 enqueueSetState 后，会调用 ReactCompositeComponent 实例中的_pendingStateQueue，将新的状态变更加入实例的等待更新状态队列中，再调用ReactUpdates 模块中的 enqueueUpdate 函数执行更新。这个过程会检查更新是否已经在进行中：
>
> - 如果是，则把组件加入 dirtyComponents 中；
> - 如果不是，先初始化更新事务，然后把组件加入 dirtyComponents 列表。
>
> 这里的初始化更新事务，就是 setState 一讲中提到的 batchingstrategy.isBatchingUpdates 开关。接下来就会在更新事务中处理所有记录的 dirtyComponents。
>
> **卸载**
>
> 对于自定义组件，也就是对 ReactCompositeComponent 而言，卸载过程需要递归地调用生命周期函数。
>
> 复制代码
>
> ```
> class CompositeComponent{
>   unmount(){
>     var publicInstance = this.publicInstance
>     if(publicInstance){
>       if(publicInstance.componentWillUnmount){
>         publicInstance.componentWillUnmount()
>       }
>     }
>     var renderedComponent = this.renderedComponent
>     renderedComponent.unmount()
>   }
> }
> ```
>
> 而对于 ReactDOMComponent 而言，卸载子元素需要清除事件监听器并清理一些缓存。
>
> 复制代码
>
> ```
> class DOMComponent{
>   unmount(){
>     var renderedChildren = this.renderedChildren
>     renderedChildren.forEach(child => child.unmount())
>   }
> }
> ```
>
> 那么到这里，卸载的过程就算完成了。
>
> **小结**
>
> 从以上的流程中我们可以看出，React 渲染的整体策略是**递归**，并通过事务建立 React 与虚拟DOM 的联系并完成调度。如果对整体函数调用流程感兴趣的同学，可以查看这个[全景大图](https://bogda n-lyashenko.github.io/Under-the-hood-ReactJS/stack/images/intro/all-page-stack-reconciler.svg)。
>
> **Fiber Reconciler**
>
> 为了避免全文过于冗长，也因为主要流程大致相同，所以我就不再赘述与 Stack Reconciler 相似的地方，主要讲一讲不一样的地方。那第一个不同点是，Stack 和 Fiber 的不同。Stack 是栈，那 Fiber 是什么呢？我们需要先理解什么是 Fiber。
>
> **Fiber**
>
> Fiber 同样是一个借来的概念，在系统开发中，指一种**最轻量化**的线程。与一般线程不同的是，Fiber 对于系统内核是不可见的，也不能由内核进行调度。它的运行模式被称为**协作式多任务**，而线程采用的模式是**抢占式多任务**。
>
> 这有什么不同呢？
>
> - 在协作式多任务模式下，线程会定时放弃自己的运行权利，告知内核让下一个线程运行；
> - 而在抢占式下，内核决定调度方案，可以直接剥夺长耗时线程的时间片，提供给其他线程。
>
> 回到浏览器中，浏览器无法实现抢占式调度，那为了提升可用性与流畅度，React 在设计上只能采用合作式调度的方案：将渲染任务拆分成多段，每次只执行一段，完成后就把时间控制权交还给主线程，这就是得名 Fiber Reconciler 的原因。
>
> 在 Fiber Reconciler 还引入了两个新的概念，分别是 Fiber 与 effect。
>
> - 在 React Element 的基础上，通过 createFiberFromElement 函数创建 Fiber 对象。Fiber 不仅包含 React Element，还包含了指向父、子、兄弟节点的属性，保证 Fiber 构成的虚拟 DOM 树成为一个双向链表。
> - effect 是指在协调过程中必须执行计算的活动。
>
> 有了 Fiber 的基础认知后，我们就可以进入 Fiber Reconciler 的协调过程了。
>
> **协调**
>
> React 团队的 Dan Abramov 画了一张基于 Fiber Reconciler 生命周期阶段图，其中协调过程被分为了两部分：Render 和 commit。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/8C/78/Ciqc1F_tm3OADjaqAAGVxU_0Bpg907.png)
>
> 图片来自 React 官网
>
> **Render**
>
> Render 阶段主要是通过构造 workInProgress 树计算出 diff。以 current 树为基础，将每个 Fiber 作为一个基本单位，自下而上逐个节点检查并构造 workInProgress 树。这个过程不再是递归，而是基于**循环**来完成。
>
> 在执行上通过 requestIdleCallback 来调度执行每组任务，每组中的每个计算任务被称为 work，每个 work 完成后确认是否有优先级更高的 work 需要插入，如果有就让位，没有就继续。优先级通常标记为动画或者 high 的会先处理。每完成一组后，将调度权交回主线程，直到下一次 requestIdleCallback 调用，再继续构建 workInProgress 树。
>
> **Commit**
>
> 在 Commit 阶段处理 effect 列表，这里的 effect 列表包含了根据 diff 更新 DOM 树、回调生命周期、响应 ref 等。
>
> 但一定要注意，这个阶段是同步执行的，不可中断暂停，所以不要在 componentDidMount、componentDidUpdate、componentWiilUnmount 中执行重度消耗算力的任务。
>
> **小结**
>
> 在上面的讲述中，省去了挂载与更新流程，这里稍微补充下，在挂载阶段， ReactMount 模块已经不存在了，是直接构造 Fiber 树。而更新流程大致一样，依然通过 IsBatchingUpdates 控制。那么 Fiber Reconciler 最大的不同有两点：
>
> - 协作式多任务模式；
> - 基于循环遍历计算 diff。
>
> ### 答题
>
> > React 的渲染过程大致一致，但协调并不相同，以 React 16 为分界线，分为 Stack Reconciler 和 Fiber Reconciler。这里的协调从狭义上来讲，特指 React 的 diff 算法，广义上来讲，有时候也指 React 的 reconciler 模块，它通常包含了 diff 算法和一些公共逻辑。
> >
> > 回到 Stack Reconciler 中，Stack Reconciler 的核心调度方式是递归。调度的基本处理单位是事务，它的事务基类是 Transaction，这里的事务是 React 团队从后端开发中加入的概念。在 React 16 以前，挂载主要通过 ReactMount 模块完成，更新通过 ReactUpdate 模块完成，模块之间相互分离，落脚执行点也是事务。
> >
> > 在 React 16 及以后，协调改为了 Fiber Reconciler。它的调度方式主要有两个特点，第一个是协作式多任务模式，在这个模式下，线程会定时放弃自己的运行权利，交还给主线程，通过requestIdleCallback 实现。第二个特点是策略优先级，调度任务通过标记 tag 的方式分优先级执行，比如动画，或者标记为 high 的任务可以优先执行。Fiber Reconciler的基本单位是 Fiber，Fiber 基于过去的 React Element 提供了二次封装，提供了指向父、子、兄弟节点的引用，为 diff 工作的双链表实现提供了基础。
> >
> > 在新的架构下，整个生命周期被划分为 Render 和 Commit 两个阶段。Render 阶段的执行特点是可中断、可停止、无副作用，主要是通过构造 workInProgress 树计算出 diff。以 current 树为基础，将每个 Fiber 作为一个基本单位，自下而上逐个节点检查并构造 workInProgress 树。这个过程不再是递归，而是基于循环来完成。
> >
> > 在执行上通过 requestIdleCallback 来调度执行每组任务，每组中的每个计算任务被称为 work，每个 work 完成后确认是否有优先级更高的 work 需要插入，如果有就让位，没有就继续。优先级通常是标记为动画或者 high 的会先处理。每完成一组后，将调度权交回主线程，直到下一次 requestIdleCallback 调用，再继续构建 workInProgress 树。
> >
> > 在 commit 阶段需要处理 effect 列表，这里的 effect 列表包含了根据 diff 更新 DOM 树、回调生命周期、响应 ref 等。
> >
> > 但一定要注意，这个阶段是同步执行的，不可中断暂停，所以不要在 componentDidMount、componentDidUpdate、componentWiilUnmount 中去执行重度消耗算力的任务。
> >
> > 如果只是一般的应用场景，比如管理后台、H5 展示页等，两者性能差距并不大，但在动画、画布及手势等场景下，Stack Reconciler 的设计会占用占主线程，造成卡顿，而 fiber reconciler 的设计则能带来高性能的表现。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/8C/83/CgqCHl_tm5OAarlJAAJVi8u3-KU747.png)
>
> ### 进阶
>
> 在面试中，在你回答完以上讲到的内容后，面试官还会补充提问一个类似脑筋急转弯的小问题：
>
> **为什么 Stack Reconciler 模式下 render 函数不支持 return 数组？**
>
> 你想呀**，**Stack Reconciler 采用的是递归遍历的模式，那么在递归的情况下就只能返回一个节点元素，肯定就不支持数组了。
>
> ### 总结
>
> 在本讲中，从渲染流程的角度解析了 React 协调这一重要概念，但值得注意的是 Fiber Reconciler 还不是一个最终的完成品，其中并发模式并不是默认启用，还处于开发阶段，目前仍然是在同步模式下启用。
>
> 在介绍渲染流程之后，下一讲我将从工程化的角度讲解，渲染异常后会出现什么情况以及该怎么办，到时见！