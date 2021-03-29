[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5796)



> 本讲我们一起来探讨“setState 是同步更新还是异步更新”，这个问题在面试中应该如何回答。
>
> ### 破题
>
> “是 A 还是 B ”是一个在面试中经常会被问到的问题类型，这类问题有相当强的迷惑性，因为在不同的场景中会有不同的选择：
>
> - 可能是 A；
> - 也可能是 B；
> - 甚至 A 和 B 同时存在的可能性也是有的。
>
> 所以就需要把问题放在具体的场景中探讨，才能有更加全面准确的回答。在面对类似的问题时，要先把场景理清楚，再去思考如何回答，一定不要让自己犯“想当然”的错误。这是回答类似问题第一个需要注意的点。
>
> 回到 setState 本身上来，setState 用于变更状态，触发组件重新渲染，更新视图 UI。有很多应聘者，并不清楚 state 在什么时候会被更新，所以难以解释到底是同步的还是异步的，也不清楚这个问题具体涉及哪些概念？
>
> 本题也是大厂面试中的一道高频题，常被用作检验应聘者的资深程度。
>
> 以上就是这个问题的“碎碎念”了，接下来是整理答题思路。
>
> ### 承题
>
> 回到问题本身上来，其实思路很简单，只要能说清楚什么是同步场景，什么是异步场景，那问题自然而然就解决了。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/01/3D/Cip5yF_YUpOAALIlAABNx0PyF94306.png)
>
> ### 入手
>
> 在分析场景之前，需要先补充一个很重要的知识点，即合成事件，同样它也是 React 面试中很容易被考察的点。合成事件与 setState 的触发更新有千丝万缕的关系，也只有在了解合成事件后，我们才能继续聊 setState。
>
> #### 合成事件
>
> 在没有合成事件前，大家是如何处理事件的呢？由于很多同学都是直接从 React 和 Vue 开始入门的，所以很可能不太清楚这样一个在过去非常常见的场景。
>
> 假设一个列表的 ul 标签下面有 10000 个 li 标签。现在需要添加点击事件，通过点击获取当前 li 标签中的文本。那该如何操作？如果按照现在 React 的编写方式，就是为每一个 li 标签添加 onclick 事件。有 10000 个 li 标签，则会添加 10000 个事件。这是一种非常不友好的方式，会对页面的性能产生影响。
>
> 复制代码
>
> ```
> <ul>
>   <li onclick="geText(this)">1</li>
>   <li onclick="geText(this)">2</li>
>   <li onclick="geText(this)">3</li>
>   <li onclick="geText(this)">4</li>
>   <li onclick="geText(this)">5</li>
>    ...
>   <li onclick="geText(this)">10000</li>
> </ul>
> ```
>
> 那该怎么优化呢？最恰当的处理方式是采用**事件委托**。通过将事件绑定在 ul 标签上这样的方式来解决。当 li 标签被点击时，由事件冒泡到父级的 ul 标签去触发，并在 ul 标签的 onclick 事件中，确认是哪一个 li 标签触发的点击事件。
>
> 复制代码
>
> ```
> <ul id="test">
>   <li>1</li>
>   <li>2</li>
>   <li>3</li>
>   <li>4</li>
>   <li>5</li>
>   <li>10000</li>
> </ul>
> <script>
>   function getEventTarget(e) {
>       e = e || window.event;
>       return e.target || e.srcElement; 
>   }
>   var ul = document.getElementById('test');
>   ul.onclick = function(event) {
>       var target = getEventTarget(event);
>       alert(target.innerHTML);
>   };
> </script>
> ```
>
> 同样，出于性能考虑，合成事件也是如此：
>
> - React 给 document 挂上事件监听；
> - DOM 事件触发后冒泡到 document；
> - React 找到对应的组件，造出一个合成事件出来；
> - 并按组件树模拟一遍事件冒泡。
>
> ![Drawing 2.png](https://s0.lgstatic.com/i/image2/M01/01/3E/CgpVE1_YUqKAA-jWAACt3Mh2xk8536.png)
>
> React 17 之前的事件冒泡流程图
>
> 所以这就造成了，在一个页面中，只能有一个版本的 React。如果有多个版本，事件就乱套了。值得一提的是，这个问题在 React 17 中得到了解决，事件委托不再挂在 document 上，而是挂在 DOM 容器上，也就是 ReactDom.Render 所调用的节点上。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/01/3E/Cip5yF_YUzCAWTyoAAB1ljK7rSM539.png)
>
> React 17 后的事件冒泡流程图
>
> 那到底哪些事件会被捕获生成合成事件呢？可以从 React 的源码测试文件中一探究竟。下面的测试快照中罗列了大量的事件名，也只有在这份快照中的事件，才会被捕获生成合成事件。
>
> 复制代码
>
> ```
> // react/packages/react-dom/src/__tests__/__snapshots__/ReactTestUtils-test.js.snap
> Array [
> 	  "abort",
> 	  "animationEnd",
> 	  "animationIteration",
> 	  "animationStart",
> 	  "auxClick",
> 	  "beforeInput",
> 	  "blur",
> 	  "canPlay",
> 	  "canPlayThrough",
> 	  "cancel",
> 	  "change",
> 	  "click",
> 	  "close",
> 	  "compositionEnd",
> 	  "compositionStart",
> 	  "compositionUpdate",
> 	  "contextMenu",
> 	  "copy",
> 	  "cut",
> 	  "doubleClick",
> 	  "drag",
> 	  "dragEnd",
> 	  "dragEnter",
> 	  "dragExit",
> 	  "dragLeave",
> 	  "dragOver",
> 	  "dragStart",
> 	  "drop",
> 	  "durationChange",
> 	  "emptied",
> 	  "encrypted",
> 	  "ended",
> 	  "error",
> 	  "focus",
> 	  "gotPointerCapture",
> 	  "input",
> 	  "invalid",
> 	  "keyDown",
> 	  "keyPress",
> 	  "keyUp",
> 	  "load",
> 	  "loadStart",
> 	  "loadedData",
> 	  "loadedMetadata",
> 	  "lostPointerCapture",
> 	  "mouseDown",
> 	  "mouseEnter",
> 	  "mouseLeave",
> 	  "mouseMove",
> 	  "mouseOut",
> 	  "mouseOver",
> 	  "mouseUp",
> 	  "paste",
> 	  "pause",
> 	  "play",
> 	  "playing",
> 	  "pointerCancel",
> 	  "pointerDown",
> 	  "pointerEnter",
> 	  "pointerLeave",
> 	  "pointerMove",
> 	  "pointerOut",
> 	  "pointerOver",
> 	  "pointerUp",
> 	  "progress",
> 	  "rateChange",
> 	  "reset",
> 	  "scroll",
> 	  "seeked",
> 	  "seeking",
> 	  "select",
> 	  "stalled",
> 	  "submit",
> 	  "suspend",
> 	  "timeUpdate",
> 	  "toggle",
> 	  "touchCancel",
> 	  "touchEnd",
> 	  "touchMove",
> 	  "touchStart",
> 	  "transitionEnd",
> 	  "volumeChange",
> 	  "waiting",
> 	  "wheel",
> 	]
> ```
>
> 在有了合成事件的基础后，就更容易理解后续的内容了。
>
> #### 调用顺序
>
> setState 是不是异步的？我们来从头梳理。
>
> **异步场景**
>
> 通常我们认为 setState 是异步的，就像这样一个例子：
>
> 复制代码
>
> ```
> class Test extends Component {
>     state = {
>         count: 0
>     }
> 
>     componentDidMount(){
>         this.setState({
>            count: 1
>          }, () => {
>             console.log(this.state.count) //1
>          })
>         console.log(this.state.count) // 0
>     }
> 
>     render(){
>         ...
>     }
> }
> ```
>
> 由于我们接受 setState 是异步的，所以会认为回调函数是异步回调，打出 0 的 console.log 会先执行，打出 1 的会后执行。
>
> 那接下来这个案例的答案是什么呢？
>
> 复制代码
>
> ```
> class Test extends Component {
>     state = {
>         count: 0
>     }
> 
>     componentDidMount(){
>         this.setState({
>            count: this.state.count + 1
>          }, () => {
>             console.log(this.state.count)
>          })
>          this.setState({
>            count: this.state.count + 1
>          }, () => {
>             console.log(this.state.count)
>          })
>     }
> 
>     render(){
>         ...
>     }
> }
> ```
>
> 如果你觉得答案是 1,2，那肯定就错了。这种迷惑性极强的考题在面试中非常常见，因为它反直觉。
>
> 如果重新仔细思考，你会发现当前拿到的 this.state.count 的值并没有变化，都是 0，所以输出结果应该是 1,1。
>
> 当然，也可以在 setState 函数中获取修改后的 state 值进行修改。
>
> 复制代码
>
> ```
> class Test extends Component {
>     state = {
>         count: 0
>     }
> 
>     componentDidMount(){
>         this.setState(
>           preState=> ({
>             count:preState.count + 1
>         }),()=>{
>            console.log(this.state.count)
>         })
>         this.setState(
>           preState=>({
>             count:preState.count + 1
>         }),()=>{
>            console.log(this.state.count)
>         })
>     }
> 
>     render(){
>         ...
>     }
> }
> ```
>
> 这些通通是异步的回调，如果你以为输出结果是 1,2，那就又错了，实际上是 2,2。
>
> 为什么会这样呢？当调用 setState 函数时，就会把当前的操作放入队列中。React 根据队列内容，合并 state 数据，完成后再逐一执行回调，根据结果更新虚拟 DOM，触发渲染。所以回调时，state 已经合并计算完成了，输出的结果就是 2,2 了。
>
> 这非常反直觉，那为什么 React 团队选择了这样一个行为模式，而不是同步进行呢？一种常见的说法是为了优化。通过异步的操作方式，累积更新后，批量合并处理，减少渲染次数，提升性能。但同步就不能批量合并吗？这显然不能完全作为 setState 设计成异步的理由。
>
> 在 17 年的时候就有人提出这样一个疑问“[为什么 setState 是异步的](https://github.com/facebook/react/issues/11527)”，这个问题得到了官方团队的回复，原因有 2 个。
>
> - **保持内部一致性**。如果改为同步更新的方式，尽管 setState 变成了同步，但是 props 不是。
> - **为后续的架构升级启用并发更新**。为了完成异步渲染，React 会在 setState 时，根据它们的数据来源分配不同的优先级，这些数据来源有：事件回调句柄、动画效果等，再根据优先级并发处理，提升渲染性能。
>
> 从 React 17 的角度分析，异步的设计无疑是正确的，使异步渲染等最终能在 React 落地。那什么情况下它是同步的呢？
>
> **同步场景**
>
> 异步场景中的案例使我们建立了这样一个认知：setState 是异步的，但下面这个案例又会颠覆你的认知。如果我们将 setState 放在 setTimeout 事件中，那情况就完全不同了。
>
> 复制代码
>
> ```
> class Test extends Component {
>     state = {
>         count: 0
>     }
> 
>     componentDidMount(){
>         this.setState({ count: this.state.count + 1 });
>         console.log(this.state.count);
>         setTimeout(() => {
>           this.setState({ count: this.state.count + 1 });
>           console.log("setTimeout: " + this.state.count);
>         }, 0);
>     }
> 
>     render(){
>         ...
>     }
> }
> ```
>
> 那这时输出的应该是什么呢？如果你认为是 0,0，那么又错了。
>
> 正确的结果是 0,2。因为 setState 并不是真正的异步函数，它实际上是通过队列延迟执行操作实现的，通过 isBatchingUpdates 来判断 setState 是先存进 state 队列还是直接更新。值为 true 则执行异步操作，false 则直接同步更新。
>
> ![图片1.png](https://s0.lgstatic.com/i/image2/M01/01/47/Cip5yF_YYfCAXIxiAAEJsQbj_hs785.png)
>  在 onClick、onFocus 等事件中，由于合成事件封装了一层，所以可以将 isBatchingUpdates 的状态更新为 true；在 React 的生命周期函数中，同样可以将 isBatchingUpdates 的状态更新为 true。那么在 React 自己的生命周期事件和合成事件中，可以拿到 isBatchingUpdates 的控制权，将状态放进队列，控制执行节奏。而在外部的原生事件中，并没有外层的封装与拦截，无法更新 isBatchingUpdates 的状态为 true。这就造成 isBatchingUpdates 的状态只会为 false，且立即执行。所以在 addEventListener 、setTimeout、setInterval 这些原生事件中都会同步更新。
>
> ### 回答
>
> 接下来我们可以答题了。
>
> > setState 并非真异步，只是看上去像异步。在源码中，通过 isBatchingUpdates 来判断
> >  setState 是先存进 state 队列还是直接更新，如果值为 true 则执行异步操作，为 false 则直接更新。
> >
> > 那么什么情况下 isBatchingUpdates 会为 true 呢？在 React 可以控制的地方，就为 true，比如在 React 生命周期事件和合成事件中，都会走合并操作，延迟更新的策略。
> >
> > 但在 React 无法控制的地方，比如原生事件，具体就是在 addEventListener 、setTimeout、setInterval 等事件中，就只能同步更新。
> >
> > 一般认为，做异步设计是为了性能优化、减少渲染次数，React 团队还补充了两点。
> >
> > 1. 保持内部一致性。如果将 state 改为同步更新，那尽管 state 的更新是同步的，但是 props不是。
> > 2. 启用并发更新，完成异步渲染。
>
> 综上所述，我们可以整理出下面的知识导图。
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image2/M01/01/3E/CgpVE1_YU2KAStLdAAFVKxh7Dyg317.png)
>
> ### 进阶
>
> 这是一道经常会出现的 React setState 笔试题：下面的代码输出什么呢？
>
> 复制代码
>
> ```
> class Test extends React.Component {
>   state  = {
>       count: 0
>   };
> 
>     componentDidMount() {
>     this.setState({count: this.state.count + 1});
>     console.log(this.state.count);
> 
>     this.setState({count: this.state.count + 1});
>     console.log(this.state.count);
> 
>     setTimeout(() => {
>       this.setState({count: this.state.count + 1});
>       console.log(this.state.count);
> 
>       this.setState({count: this.state.count + 1});
>       console.log(this.state.count);
>     }, 0);
>   }
>  
>   render() {
>     return null;
>   }
> };
> ```
>
> 我们可以进行如下的分析：
>
> - 首先第一次和第二次的 console.log，都在 React 的生命周期事件中，所以是异步的处理方式，则输出都为 0；
> - 而在 setTimeout 中的 console.log 处于原生事件中，所以会同步的处理再输出结果，但需要注意，虽然 count 在前面经过了两次的 this.state.count + 1，但是每次获取的 this.state.count 都是初始化时的值，也就是 0；
> - 所以此时 count 是 1，那么后续在 setTimeout 中的输出则是 2 和 3。
>
> 所以完整答案是 0,0,2,3。
>
> ### 总结
>
> 在本讲中，我们掌握了判断 setState 是同步还是异步的核心关键点：更新队列。不得不再强调一下，看 setState 的输出结果是面试的常考点。所以在面试前，可以再针对性的看一下这部分内容，然后自己执行几次试试。
>
> 下一讲我将为你介绍另一个常考点，React 的跨组件通信。