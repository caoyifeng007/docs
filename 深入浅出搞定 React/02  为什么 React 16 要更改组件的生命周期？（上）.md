React 生命周期已经是一个老生常谈的话题了，几乎没有哪一门 React 入门教材会省略对组件生命周期的介绍。然而，入门教材在设计上往往追求的是“简单省事、迅速上手”，这就导致许多同学对于生命周期知识的刻板印象为“背就完了、别想太多”。

“背就完了”这样简单粗暴的学习方式，或许可以帮助你理解“What to do”，到达“How to do”，但却不能帮助你去思考和认知“Why to do”。作为一个专业的 React 开发者，我们必须要求自己在知其然的基础上，知其所以然。

在本课时和下一个课时，我将抱着帮你做到“知其所以然”的目的，以 React 的基本原理为引子，**对 React 15、React 16 两个版本的生命周期进行探讨、比对和总结，通过搞清楚一个又一个的“Why”，来帮你建立系统而完善的生命周期知识体系**。

### 生命周期背后的设计思想：把握 React 中的“大方向”

在介绍具体的生命周期之前，我想先带你初步理解 React 框架中的一些关键的设计思想，以便为你后续的学习提供不可或缺的“加速度”。

如果你经常翻阅 React 官网或者 React 官方的一些文章，你会发现“**组件**”和“**虚拟 DOM**”这两个词的出镜率是非常高的，它们是 React 基本原理中极为关键的两个概念，也是我们这个小节的学习切入点。

#### 虚拟 DOM：核心算法的基石

通过 01 课时的学习，你已经知晓了虚拟 DOM 节点的基本形态，现在我们需要简单了解下虚拟 DOM 在整个 React 工作流中的作用。

组件在初始化时，会通过调用生命周期中的 render 方法，**生成虚拟 DOM**，然后再通过调用 ReactDOM.render 方法，实现虚拟 DOM 到真实 DOM 的转换。

当组件更新时，会再次通过调用 render 方法**生成新的虚拟 DOM**，然后借助 diff（这是一个非常关键的算法，我将在“模块二：核心原理”重点讲解）**定位出两次虚拟 DOM 的差异**，从而针对发生变化的真实 DOM 作定向更新。

以上就是 React 框架核心算法的大致流程。对于这套关键的工作流来说，“虚拟 DOM”是所有操作的大前提，是核心算法的基石。

#### 组件化：工程化思想在框架中的落地

组件化是一种优秀的软件设计思想，也是 React 团队在研发效能方面所做的一个重要的努力。

在一个 React 项目中，几乎所有的可见/不可见的内容都可以被抽离为各种各样的组件，每个组件既是“封闭”的，也是“开放”的。

所谓“封闭”，主要是针对“渲染工作流”（指从**组件数据改变**到**组件实际更新发生的**过程）来说的。在组件自身的渲染工作流中，每个组件都只处理它内部的渲染逻辑。在没有数据流交互的情况下，组件与组件之间可以做到“各自为政”。

而所谓“开放”，则是针对组件间通信来说的。React 允许开发者基于“单向数据流”的原则完成组件间的通信。而组件之间的通信又将改变通信双方/某一方内部的数据，进而对渲染结果构成影响。所以说在数据这个“红娘”的牵线搭桥之下，组件之间又是彼此开放的，是可以相互影响的。

这一“开放”与“封闭”兼具的特性，使得 React 组件**既专注又灵活**，具备高度的可重用性和可维护性。

#### 生命周期方法的本质：组件的“灵魂”与“躯干”

之前我曾经在社区读过一篇文章，文中将 render 方法形容为 React 组件的“灵魂”。当时我对这句话产生了非常强烈的共鸣，这里我就想以这个曾经打动过我的比喻为引子，帮助你从宏观上建立对 React 生命周期的感性认知。

注意，这里提到的 render 方法，和我们 01 课时所说的 ReactDOM.render 可不是一个东西，它指的是 React 组件内部的这个生命周期方法：

复制代码

```
class LifeCycle extends React.Component {

  render() {
    console.log("render方法执行");
    return (
      <div className="container">
        this is content
      </div>
    );
  }
}
```

前面咱们介绍了虚拟 DOM、组件化，倘若把这两块知识整合一下，你就会发现这两个概念似乎都在围着 render 这个生命周期打转：虚拟 DOM 自然不必多说，它的生成都要仰仗 render；而组件化概念中所提及的“渲染工作流”，这里指的是从**组件数据改变**到**组件实际更新发生的**过程，这个过程的实现同样离不开 render。

由此看来，render 方法在整个组件生命周期中确实举足轻重，它担得起“灵魂”这个有分量的比喻。那么如果将 render 方法比作组件的“**灵魂**”，render 之外的生命周期方法就完全可以理解为是组件的“**躯干**”。

“躯干”未必总是会做具体的事情（比如说我们可以选择性地省略对 render 之外的任何生命周期方法内容的编写），而“灵魂”却总是充实的（render 函数却坚决不能省略）；倘若“躯干”做了点什么，往往都会直接或间接地影响到“灵魂”（因为即便是 render 之外的生命周期逻辑，也大部分是在为 render 层面的效果服务）；“躯干”和“灵魂”一起，共同构成了 React 组件完整而不可分割的“生命时间轴”。

### 拆解 React 生命周期：从 React 15 说起

我发现时下许多资料在讲解 React 生命周期时，喜欢直接拿 React 16 开刀。这样做虽然省事儿，却也模糊掉了新老生命周期变化背后的“Why”（关于两者的差异，我们会在“03 课时”中详细讲解）。这里为了把这个“Why”拎出来，我将首先带你认识 React 15 的生命周期流程。

在 React 15 中，大家需要关注以下几个生命周期方法：

复制代码

```
constructor()
componentWillReceiveProps()
shouldComponentUpdate()
componentWillMount()
componentWillUpdate()
componentDidUpdate()
componentDidMount()
render()
componentWillUnmount()
```

> 如果你接触 React 足够早，或许会记得还有 getDefaultProps 和 getInitState 这两个方法，它们都是 React.createClass() 模式下初始化数据的方法。由于这种写法在 ES6 普及后已经不常见，这里不再详细展开。

这些生命周期方法是如何彼此串联、相互依存的呢？这里我为你总结了一张大图：

![1.png](https://s0.lgstatic.com/i/image/M00/5E/31/Ciqc1F-GZbGAGNcBAAE775qohj8453.png)

接下来，我就围绕这张大图，分阶段讨论组件生命周期的运作规律。在学习的过程中，下面这个 Demo 可以帮助你具体地验证每个阶段的工作流程：

复制代码

```
import React from "react";
import ReactDOM from "react-dom";
// 定义子组件
class LifeCycle extends React.Component {
  constructor(props) {
    console.log("进入constructor");
    super(props);
    // state 可以在 constructor 里初始化
    this.state = { text: "子组件的文本" };
  }
  // 初始化渲染时调用
  componentWillMount() {
    console.log("componentWillMount方法执行");
  }
  // 初始化渲染时调用
  componentDidMount() {
    console.log("componentDidMount方法执行");
  }
  // 父组件修改组件的props时会调用
  componentWillReceiveProps(nextProps) {
    console.log("componentWillReceiveProps方法执行");
  }
  // 组件更新时调用
  shouldComponentUpdate(nextProps, nextState) {
    console.log("shouldComponentUpdate方法执行");
    return true;
  }

  // 组件更新时调用
  componentWillUpdate(nextProps, nextState) {
    console.log("componentWillUpdate方法执行");
  }
  // 组件更新后调用
  componentDidUpdate(preProps, preState) {
    console.log("componentDidUpdate方法执行");
  }
  // 组件卸载时调用
  componentWillUnmount() {
    console.log("子组件的componentWillUnmount方法执行");
  }
  // 点击按钮，修改子组件文本内容的方法
  changeText = () => {
    this.setState({
      text: "修改后的子组件文本"
    });
  };
  render() {
    console.log("render方法执行");
    return (
      <div className="container">
        <button onClick={this.changeText} className="changeText">
          修改子组件文本内容
        </button>
        <p className="textContent">{this.state.text}</p>
        <p className="fatherContent">{this.props.text}</p>
      </div>
    );
  }
}
// 定义 LifeCycle 组件的父组件
class LifeCycleContainer extends React.Component {

  // state 也可以像这样用属性声明的形式初始化
  state = {
    text: "父组件的文本",
    hideChild: false
  };
  // 点击按钮，修改父组件文本的方法
  changeText = () => {
    this.setState({
      text: "修改后的父组件文本"
    });
  };
  // 点击按钮，隐藏（卸载）LifeCycle 组件的方法
  hideChild = () => {
    this.setState({
      hideChild: true
    });
  };
  render() {
    return (
      <div className="fatherContainer">
        <button onClick={this.changeText} className="changeText">
          修改父组件文本内容
        </button>
        <button onClick={this.hideChild} className="hideChild">
          隐藏子组件
        </button>
        {this.state.hideChild ? null : <LifeCycle text={this.state.text} />}
      </div>
    );
  }
}
ReactDOM.render(<LifeCycleContainer />, document.getElementById("root"));
```

该入口文件对应的 index.html 中预置了 id 为 root 的真实 DOM 节点作为根节点，body 标签内容如下：

复制代码

```
<body>
  <div id="root"></div>
</body>
```

这个 Demo 渲染到浏览器上大概是这样的：

![Drawing 1.png](https://s0.lgstatic.com/i/image/M00/5D/CC/Ciqc1F-FU-yAMLh0AABeqOeqLek815.png)

此处由于我们强调的是对生命周期执行规律的验证，所以样式上从简，你也可以根据自己的喜好添加 CSS 相关的内容。

接下来我们就结合这个 Demo 和开头的生命周期大图，一起来看看挂载、更新、卸载这 3 个阶段，React 组件都经历了哪些事情。

#### Mounting 阶段：组件的初始化渲染（挂载）

挂载过程在组件的一生中仅会发生一次，在这个过程中，组件被初始化，然后会被渲染到真实 DOM 里，完成所谓的“首次渲染”。

在挂载阶段，一个 React 组件会按照顺序经历如下图所示的生命周期：

![3.png](https://s0.lgstatic.com/i/image/M00/5E/32/Ciqc1F-GZ1OAWETTAAA3Am2CwU0383.png)

首先我们来看 constructor 方法，该方法仅仅在挂载的时候被调用一次，我们可以在该方法中对 this.state 进行初始化：

复制代码

```
constructor(props) {
  console.log("进入constructor");
  super(props);
  // state 可以在 constructor 里初始化
  this.state = { text: "子组件的文本" };
}
```

componentWillMount、componentDidMount 方法同样只会在挂载阶段被调用一次。其中 componentWillMount 会在执行 render 方法前被触发，一些同学习惯在这个方法里做一些初始化的操作，但这些操作往往会伴随一些风险或者说不必要性（这一点大家先建立认知，具体原因将在“03 课时”展开讲解）。

接下来 render 方法被触发。注意 render 在执行过程中并不会去操作真实 DOM（也就是说不会渲染），它的职能是**把需要渲染的内容返回出来**。真实 DOM 的渲染工作，在挂载阶段是由 ReactDOM.render 来承接的。

componentDidMount 方法在渲染结束后被触发，此时因为真实 DOM 已经挂载到了页面上，我们可以在这个生命周期里执行真实 DOM 相关的操作。此外，类似于异步请求、数据初始化这样的操作也大可以放在这个生命周期来做（侧面印证了 componentWillMount 真的很鸡肋）。

这一整个流程对应的其实就是我们 Demo 页面刚刚打开时，组件完成初始化渲染的过程。下图是 Demo 中的 LifeCycle 组件在挂载过程中控制台的输出，你可以用它来验证挂载过程中生命周期顺序的正确性：

![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/5D/D8/CgqCHl-FU_6AeWUcAAB8X4bjwqE102.png)

#### Updating 阶段：组件的更新

组件的更新分为两种：一种是由父组件更新触发的更新；另一种是组件自身调用自己的 setState 触发的更新。这两种更新对应的生命周期流程如下图所示：

![2.png](https://s0.lgstatic.com/i/image/M00/5E/3C/CgqCHl-GZf-AUjsLAACmOsiQl3M485.png)

**componentWillReceiProps 到底是由什么触发的？**

从图中你可以明显看出，父组件触发的更新和组件自身的更新相比，多出了这样一个生命周期方法：

复制代码

```
componentWillReceiveProps(nextProps)
```

在这个生命周期方法里，nextProps 表示的是接收到新 props 内容，而现有的 props （相对于 nextProps 的“旧 props”）我们可以通过 this.props 拿到，由此便能够感知到 props 的变化。

写到这里，就不得不在“变化”这个动作上深挖一下了。我在一些社区文章里，包括一些候选人面试时的回答里，都不约而同地见过/听过这样一种说法：**componentWillReceiveProps 是在组件的 props 内容发生了变化时被触发的。**

**这种说法不够严谨**。远的不说，就拿咱们上文给出的 Demo 开刀，该界面的控制台输出在初始化完成后是这样的：

![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/5D/CC/Ciqc1F-FVA6AYiD4AADSl2lr-_Q663.png)

注意，我们代码里面，LifeCycleContainer 这个父组件传递给子组件 LifeCycle 的 props 只有一个 text：

复制代码

```
<LifeCycle text={this.state.text} />
```

假如我点击“修改父组件文本内容”这个按钮，父组件的 this.state.text 会发生改变，进而带动子组件的 this.props.text 发生改变。此时一定会触发 componentWillReceiveProps 这个生命周期，这是毋庸置疑的：

![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/5D/CC/Ciqc1F-FVBWAEqTGAAEdsvX2TAM747.png)

但如果我现在对父组件的结构进行一个小小的修改，给它一个和子组件完全无关的 state（this.state.ownText），同时相应地给到一个修改这个 state 的方法（this.changeOwnText），并用一个新的 button 按钮来承接这个触发的动作。

改变后的 LifeCycleContainer 如下所示：

复制代码

```
// 定义 LifeCycle 组件的父组件
class LifeCycleContainer extends React.Component {
  // state 也可以像这样用属性声明的形式初始化
  state = {
    text: "父组件的文本",
    // 新增的只与父组件有关的 state
    ownText: "仅仅和父组件有关的文本",
    hideChild: false
  };
  changeText = () => {
    this.setState({
      text: "修改后的父组件文本"
    });
  };
  // 修改 ownText 的方法
  changeOwnText = () => {
    this.setState({
      ownText: "修改后的父组件自有文本"
    });
  };
  hideChild = () => {
    this.setState({
      hideChild: true
    });
  };
  render() {
    return (
      <div className="fatherContainer">
        {/* 新的button按钮 */}
        <button onClick={this.changeOwnText} className="changeText">
          修改父组件自有文本内容
        </button>
        <button onClick={this.changeText} className="changeText">
          修改父组件文本内容
        </button>
        <button onClick={this.hideChild} className="hideChild">
          隐藏子组件
        </button>
        <p> {this.state.ownText} </p>
        {this.state.hideChild ? null : <LifeCycle text={this.state.text} />}
      </div>
    );
  }
}
```

新的界面如下图所示：

![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/5D/CD/Ciqc1F-FVCGAVX_GAAFADHW8-9A107.png)

可以看到，this.state.ownText 这个状态和子组件完全无关。但是当我点击“修改父组件自有文本内容”这个按钮的时候，componentReceiveProps 仍然被触发了，效果如下图所示：

![Drawing 8.png](https://s0.lgstatic.com/i/image/M00/5D/D8/CgqCHl-FVCqASZNkAAGmF-R62cg649.png)

耳听为虚，眼见为实。面对这样的运行结果，我不由得要带你复习一下 React 官方文档中的这句话：

![图片7.png](https://s0.lgstatic.com/i/image/M00/5D/E1/Ciqc1F-FaGuADV5vAACZ2YRV6qQ941.png)

**componentReceiveProps 并不是由 props 的变化触发的，而是由父组件的更新触发的**，这个结论，请你谨记。

**组件自身 setState 触发的更新**

this.setState() 调用后导致的更新流程，前面大图中已经有体现，这里我直接沿用上一个 Demo 来做演示。若我们点击上一个 Demo 中的“修改子组件文本内容”这个按钮：

![Drawing 9.png](https://s0.lgstatic.com/i/image/M00/5D/D8/CgqCHl-FVDWABuVmAADVzZuKCO0699.png)

这个动作将会触发子组件 LifeCycle 自身的更新流程，随之被触发的生命周期函数如下图增加的 console 内容所示：

![Drawing 10.png](https://s0.lgstatic.com/i/image/M00/5D/CD/Ciqc1F-FVDuASw5bAAEhb9melJQ452.png)

先来说说 componentWillUpdate 和 componentDidUpdate 这一对好基友。

componentWillUpdate 会在 render 前被触发，它和 componentWillMount 类似，允许你在里面做一些不涉及真实 DOM 操作的准备工作；而 componentDidUpdate 则在组件更新完毕后被触发，和 componentDidMount 类似，这个生命周期也经常被用来处理 DOM 操作。此外，我们也常常将 componentDidUpdate 的执行作为子组件更新完毕的标志通知到父组件。

**render 与性能：初识 shouldComponentUpdate**

这里需要重点提一下 shouldComponentUpdate 这个生命周期方法，它的调用形式如下所示：

复制代码

```
shouldComponentUpdate(nextProps, nextState)
```

render 方法由于伴随着对虚拟 DOM 的构建和对比，过程可以说相当耗时。而在 React 当中，很多时候我们会不经意间就频繁地调用了 render。为了避免不必要的 render 操作带来的性能开销，React 为我们提供了 shouldComponentUpdate 这个口子。

React 组件会根据 shouldComponentUpdate 的返回值，来决定是否执行该方法之后的生命周期，进而决定是否对组件进行**re-render**（重渲染）。shouldComponentUpdate 的默认值为 true，也就是说“无条件 re-render”。在实际的开发中，我们往往通过手动往 shouldComponentUpdate 中填充判定逻辑，或者直接在项目中引入 PureComponent 等最佳实践，来实现“有条件的 re-render”。

关于 shouldComponentUpdate 及 PureComponent 对 React 的优化，我们会在后续的性能小节中详细展开。这里你只需要认识到 shouldComponentUpdate 的基本使用及其**与 React 性能之间的关联关系**即可。

#### Unmounting 阶段：组件的卸载

组件的销毁阶段本身是比较简单的，只涉及一个生命周期，如下图所示：

![图片6.png](https://s0.lgstatic.com/i/image/M00/5D/EC/CgqCHl-FaHuAVGc_AABE6JqN9E0073.png)

对应上文的 Demo 来看，我们点击“隐藏子组件”后就可以把 LifeCycle 从父组件中移除掉，进而实现卸载的效果。整个过程如下图所示：

![Drawing 12.png](https://s0.lgstatic.com/i/image/M00/5D/CD/Ciqc1F-FVFeABZvpAAO9lJVFKhs335.png)

这个生命周期本身不难理解，我们重点说说怎么触发它。组件销毁的常见原因有以下两个。

- 组件在父组件中被移除了：这种情况相对比较直观，对应的就是我们上图描述的这个过程。
- 组件中设置了 key 属性，父组件在 render 的过程中，发现 key 值和上一次不一致，那么这个组件就会被干掉。

在本课时，只要能够理解到 1 就可以了。对于 2 这种情况，你只需要先记住有这样一种现象，这就够了。至于组件里面为什么要设置 key，为什么 key 改变后组件就必须被干掉？要回答这个问题，需要你先理解 React 的“调和过程”，而“调和过程”也会是我们第二模块中重点讲解的一个内容。这里我先把这个知识点点出来，方便你定位我们整个知识体系里的**重难点**。

### 总结

在本课时，我们对 React 设计思想中的“虚拟 DOM”和“组件化”这两个关键概念形成了初步的理解，同时也对 React 15 中的生命周期进行了系统的学习和总结。到这里，你已经了解到了 React 生命周期在很长一段“过去”里的形态。

而在 React 16 中，组件的生命周期其实已经发生了一系列的变化。这些变化到底是什么样的，它们背后又蕴含着 React 团队怎样的思量呢？

古人说“以史为镜，可以知兴衰”。在下个课时，我们将一起去“照镜子”，对 React 新旧生命周期进行对比，并探求变化的动机。