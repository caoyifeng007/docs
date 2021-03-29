[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5795)



> 在第 01 讲中，我们知道了 React 通过组件化的方式，解决了工程实践中代码如何组织的问题，但它并没有指出组件之间应该按照什么样的方式去组合，本讲我们一起来探讨这个问题，即“如何设计 React 组件”。
>
> ### 破题
>
> “如何设计 React 组件？”其实就是在考察你是否了解 React 组件的设计模式。
>
> 你有没有发现在实际的工程实践中，如果缺乏一个指导性的设计模式，而直接开发，代码往往会非常凌乱。常见的情况就是：
>
> - 将一个页面写成一个组件；
> - 一个组件包含两三千行的代码。
>
> 这些都没有明显的模块划分，缺乏组合的思想。所以如何将组件更好地组合，这是需要探讨的第一个问题。
>
> 在明确了“如何组合”这一核心主题后，我们需要思考的是，如何将核心主题以更好的形式展示出来，因为平铺直叙地罗列知识，那内容是非常干瘪的。而基于不同的业务场景，组件的组合形式是不一样的，所以如果结合丰富场景来展示“如何组合”的方式，可以让表述变得有血有肉，也显得你经验十足。
>
> 这里你就需要先搞清楚基于场景的设计分类了。
>
> ### 承题
>
> 通过以上的分析，我们可以得出“如何设计 React 组件？”这一题的答题套路是“一个主题，多个场景”，即围绕“如何组合”这一核心主题，通过列举场景的方式展现设计模式的分类及用途。
>
> 我们先来了解下 React 的组件有哪些分类，这里可以直接采用 React 社区中非常经典的分类模式：
>
> - 把只作展示、独立运行、不额外增加功能的组件，称为**哑组件**，或**无状态组件**，还有一种叫法是**展示组件**；
> - 把处理业务逻辑与数据状态的组件称为有**状态组件**，或**灵巧组件**，灵巧组件一定包含至少一个灵巧组件或者展示组件。
>
> 从分类中可以看出**展示组件的复用性更强，灵巧组件则更专注于业务本身**。那么基于以上的思路，你可以整理出如下的知识导图：
>
> ![react面试05金句.png](https://s0.lgstatic.com/i/image/M00/84/27/CgqCHl_TIYmAVjAWAAUY2rGM2bc188.png)
>
> ![图片1.png](https://s0.lgstatic.com/i/image/M00/84/1C/Ciqc1F_TIY-ANgywAAB0DSyjFv4894.png)
>
> 接下来我将结合各个场景来为你展开讲解这些组件。
>
> ### 入题
>
> 无论是怎样的设计，始终是不能脱离工程实践进行探讨的。回到前端工程中来，如果使用 create-react-app 初始化项目，通常会有类似这样的目录结构：
>
> 复制代码
>
> ```
> .
> ├── README.md
> ├── package.json
> ├── public
> │   ├── favicon.ico
> │   ├── index.html
> │   ├── logo192.png
> │   ├── logo512.png
> │   ├── manifest.json
> │   └── robots.txt
> ├── src
> │   ├── App.css
> │   ├── App.js
> │   ├── App.test.js
> │   ├── index.css
> │   ├── index.js
> │   ├── logo.svg
> │   ├── reportWebVitals.js
> │   └── setupTests.js
> └── yarn.lock
> ```
>
> 在源码目录，也就是 src 目录中，所有组件就像衣服散落在房间里一样堆在了一起，如果继续添置衣物，可以想象这个房间最后会变得有多乱。就像每件衣服总有它适用的场合，组件也有同样的分类。
>
> 我先带你从功能最薄弱的展示组件开始梳理，其次是展示组件中装饰作用的小物件。
>
> #### 展示组件
>
> 展示组件内部是没有状态管理的，正如其名，就像一个个“装饰物”一样，完全受制于外部的 props 控制。展示组件具有极强的**通用性**，**复用率**也很高，往往与当前的前端工程关系相对薄弱，甚至可以做到跨项目级的复用。
>
> 我们先来看一下展示组件中最常用的代理组件。
>
> **代理组件**
>
> **代理组件常用于封装常用属性**，**减少重复代码**。关于代理组件你应该不陌生，可能经常会写。
>
> 举一个最常见的例子，当需要定义一个按钮的时候，需要在按钮上添加 button 属性，代码如下：
>
> 复制代码
>
> ```
> <button type="button">
> ```
>
> 当然在 React 中使用的时候，不可能每次都写这样一段代码，非常麻烦。常见的做法是**封装**：
>
> 复制代码
>
> ```
> const Button = props =>
>   <button type="button" {...props}>
> ```
>
> 在开发中使用 Button 组件替代原生的 button，可以确保 type 保证一致。
>
> 在使用 Antd 开发时，你也会采用类似的设计模式，大致情况如下：
>
> 复制代码
>
> ```
> import { Button as AntdButton } from from 'antd'
> const Button = props =>
>   <AntdButton size="small" type="primary" {...props}>
> 
> export default Button
> ```
>
> 虽然进行封装感觉是多此一举，但切断了外部组件库的**强依赖特性**。在大厂中引入外部组件库需要考虑两点：
>
> - 如果当前组件库不能使用了，是否能实现业务上的无痛切换；
> - 如果需要批量修改基础组件的字段，如何解决？
>
> 代理组件的设计模式很好地解决了上面两个问题。从业务上看，代理组件隔绝了 Antd，仅仅是一个组件 Props API 层的交互。这一层如若未来需要替换，是可以保证兼容、快速替换的，而不需要在原有的代码库中查找修改。其次，如果要修改基础组件的颜色、大小、间距，代理组件也可以相对优雅地解决，使得这些修改都内聚在当前的 Button 组件中，而非散落在其他地方。
>
> 基于展示组件的思想，可以封装类似的其他组件，比如样式组件。
>
> **样式组件**
>
> 样式组件也是一种代理组件，只是又细分了处理样式领域，将当前的关注点分离到当前组件内。你是否还记得在第 02 讲中提到过“关注点分离”的概念，其中就说到“将代码分隔为不同部分，其中每一部分都会有自己的关注焦点”。
>
> 但在工程实践中，我们并不会因为一个按钮需要协商 className 而封装成一个组件，就像下面这样：
>
> 复制代码
>
> ```
> const Button = props => (
>   <button type="button" className="btn btn-primary">
> )
> ```
>
> 这并没有什么意义。真实工程项目的样式管理往往是复杂的，它更接近于下面这个例子：
>
> 复制代码
>
> ```
> import classnames from "classnames";
> 
> const StyleButton = ({ className, primary, isHighLighted,  ...props }) => (
>   <Button
>     type="button"
>     className={classnames("btn", {
>      btn-primary: primary,
>      highLight: isHighLighted,
> }, className)}
>     {...props}
>   />
> );
> ```
>
> 复杂的样式管理对于 Button 是没有意义的，如果直接使用 Button，在属性上修改，对工程代码而言就是编写大量的面条代码。而 StyleButton 的思路是将样式判断逻辑分离到自身上，面向未来改动的时候会更为友好。
>
> 接下来可以看下基于样式组件的优化设计。
>
> **布局组件**
>
> 布局组件的基本设计与样式组件完全一样，但它基于自身特性做了一个小小的优化。
>
> 首先来看下它的基础使用案例，主要用于安放其他组件，类似于这样的用法：
>
> 复制代码
>
> ```
> <Layout
>   Top={<NavigationBar />}
>   Content={<Article />}
>   Bottom={<BottomBar />}
> />
> ```
>
> 布局本身是确定的，不需要根据外部状态的变化去修改内部组件。所以这也是一个可以减少渲染的优化点。（当然，这里的样式结构写得比较简单）
>
> 复制代码
>
> ```
> class Layout extends React.Component {
>   shouldComponentUpdate() {
>     return false;
>   }
>   render() {
>     <div>
>       <div>{this.props.NavigationBar}</div>
>       <div>{this.props.Article}</div>
>       <div>{this.props.BottomBar}</div>
>     </div>
>   }
> }
> ```
>
> 由于布局组件无需更新，所以对应到第 3 讲中提到的生命周期，就可以通过写死**shouldComponentUpdate** 的返回值直接阻断渲染过程。对于大型前端工程，类似的小心思可以带来性能上的提升。当然，这也是基于代理组件**更易于维护**而带来的好处。
>
> #### 灵巧组件
>
> 由于灵巧组件面向业务，所以相对于展示组件来说，其功能更为丰富、复杂性更高，而复用度更低。**展示组件专注于组件本身特性**，**灵巧组件更专注于组合组件**。那么最常见的案例则是容器组件。
>
> **容器组件**
>
> 容器组件几乎没有复用性，它主要用在两个方面：**拉取数据与组合组件**。可以看这样一个例子：
>
> 复制代码
>
> ```
> const CardList = ({ cards }) => (
>   <div>
>     {cards.map(card => (
>       <CardLayout
>         header={<Avatar url={card.avatarUrl} />}
>         Content={<Card {...card} />}
>       />
>         {comment.body}-{comment.author}
>     ))}
>   </div>
> );
> ```
>
> 这是一个 CardList 组件，负责将 cards 数据渲染出来，接下来将获取网络数据。如下代码所示：
>
> 复制代码
>
> ```
> class CardListContainer extends React.Component {
>   state = { cards: [] }
>  
>   async componentDidMount() {
>     const response = await fetch('/api/cards')
>     this.setState({cards: response})
>   }
>  
>   render() {
>     return <CardList cards={this.state.cards} />
>   }
> }
> ```
>
> 像这样切分代码后，你会发现容器组件内非常干净，没有冗余的样式与逻辑处理。你有没有发现这也是采取了关注点分离的策略？其实这一策略还可以直接应用到你的工作中。因为互联网人的工作常常是多线并行，如果想把事做得更漂亮，可以尝试把它切分成多个片段，让自己的关注点在短时间内更为集中，从而做到高效快速地处理。
>
> 回到组件的问题上来，那么对复用性更强的业务逻辑采用什么方式处理呢？
>
> **高阶组件**
>
> React 的官方文档将高阶组件称为 React 中**复用组件逻辑的高级技术**。高阶组件本身并不是 React API 的一部分，它是一种基于 React 的组合特性而形成的设计模式。简而言之，高阶组件的参数是组件，返回值为新组件的函数。
>
> 这样听起来有一些高阶函数的味儿了。那什么是**高阶函数**呢？如果一个函数可以接收另一个函数作为参数，且在执行后返回一个函数，这种函数就称为高阶函数。在 React 的社区生态中，有很多基于高阶函数设计的库，比如 reselector 就是其中之一。
>
> 思想一脉相承，React 团队在组件方向也汲取了同样的设计模式。源自高阶函数的高阶组件，可以同样优雅地抽取公共逻辑。
>
> **抽取公共逻辑**
>
> 用一个常见的例子来说，就是登录态的判断。假设当前项目有订单页面、用户信息页面及购物车首页，那么对于订单页面与用户信息页面都需要检查当前是否已登录，如果没有登录，则应该跳转登录页。
>
> 一般的思路类似于：
>
> 复制代码
>
> ```
> const checkLogin = () => {
>   return !!localStorage.getItem('token')
> }
> class CartPage extends React.Component {
>    ...
> }
> class UserPage extends  React.Component {
>   componentDidMount() {
>     if(!checkLogin) {
>       // 重定向跳转登录页面
>     }
>   }
>   ...
> }
> class OrderPage extends  React.Component {
>   componentDidMount() {
>     if(!checkLogin) {
>       // 重定向跳转登录页面
>     }
>   }
>   ...
>  }
> ```
>
> 虽然已经抽取了一个函数，但还是需要在对应的页面添加**登录态的判断逻辑**。然而如果有高阶组件的话，情况会完全不同。
>
> 复制代码
>
> ```
> const checkLogin = () => {
>   return !!localStorage.getItem('token')
> }
> const checkLogin = (WrappedComponent) => {
>           return (props) => {
>               return checkLogin() ? <WrappedComponent {...props} /> : <LoginPage />;
>           }
> // 函数写法
> class RawUserPage extends  React.Component {
>   ...
> }
> const UserPage = checkLogin(RawUserPage)
> // 装饰器写法
> @checkLogin
> class UserPage extends  React.Component {
>   ...
> }
> @checkLogin
> class OrderPage extends  React.Component {
>   ...
> }
> ```
>
> 从上面的例子中可以看出无论**采用函数**还是**装饰器**的写法，都使得重复代码量下降了一个维度。
>
> 还有一个非常经典的场景就是**页面埋点统计**。如果使用装饰器编写的话，大概是这样的：
>
> 复制代码
>
> ```
> const trackPageView = (pageName) = { 
>    // 发送埋点信息请求
>    ... 
> }
> const PV = (pageName) => {
>   return (WrappedComponent) => {
>     return class Wrap extends Component {
>       componentDidMount() {
>         trackPageView(pageName)
>       }
>  
>       render() {
>         return (
>           <WrappedComponent {...this.props} />
>         );
>       }
>     }
>   };
> }
> @PV('用户页面')
> class UserPage extends  React.Component {
>   ...
> }
> @PV('购物车页面')
> class CartPage extends  React.Component {
>   ...
> }
> @PV('订单页面')
> class OrderPage extends  React.Component {
>   ..
> }
> ```
>
> 就连埋点这样的烦琐操作都变得优雅了起来。那我想同时使用 checkLogin 与 PV 怎么办呢？这里涉及到了一个新的概念，就是链式调用。
>
> **链式调用**
>
> 由于高阶组件返回的是一个新的组件，所以链式调用是默认支持的。基于 checkLogin 与 PV 两个例子，链式使用是这样的：
>
> 复制代码
>
> ```
> // 函数调用方式
> class RawUserPage extends React.Component {
>   ...
> }
> const UserPage = checkLogin(PV('用户页面')(RawUserPage))
> // 装饰器调用方式
> @checkLogin
> @PV('用户页面')
> class UserPage extends  React.Component {
>   ...
> }
> ```
>
> 在链式调用后，装饰器会按照从外向内、从上往下的顺序进行执行。
>
> 除了抽取公用逻辑以外，还有一种修改渲染结果的方式，被称为**渲染劫持。**
>
> **渲染劫持**
>
> 渲染劫持可以通过控制 render 函数修改输出内容，常见的场景是显示加载元素，如下情况所示：
>
> 复制代码
>
> ```
>  function withLoading(WrappedComponent) {
>     return class extends WrappedComponent {
>         render() {
>             if(this.props.isLoading) {
>                 return <Loading />;
>             } else {
>                 return super.render();
>             }
>         }
>     };
> }
> ```
>
> 通过高阶函数中继承原组件的方式，劫持修改 render 函数，篡改返回修改，达到显示 Loading 的效果。
>
> 但高阶组件并非万能，它同样也有缺陷。
>
> **缺陷**
>
> **丢失静态函数**
>
> 由于被包裹了一层，所以静态函数在外层是无法获取的。如下面的案例中 getUser 是无法被调用的。
>
> 复制代码
>
> ```
> // UserPage.jsx
> @PV('用户页面')
> export default class UserPage extends  React.Component {
>   static getUser() {
>       ...
>   } 
> }
> // page.js
> import UserPage from './UserPage'
> UserPage.checkLogin() // 调用失败，并不存在。
> ```
>
> 如果希望外界能够被调用，那么可以在 PV 函数中将静态函数复制出来。
>
> 复制代码
>
> ```
> const PV = (pageName) => {
>   return (WrappedComponent) => {
>     class Wrap extends Component {
>       componentDidMount() {
>         trackPageView(pageName)
>       }
>  
>       render() {
>         return (
>           <WrappedComponent {...this.props} />
>         );
>       }
>     }
>      Wrap.getUser = WrappedComponent.getUser;
>      return Wrap;
>   };
>  }
> ```
>
> 这样做确实能解决静态函数在外部无法调用的问题，但一个类的静态函数可能会有很多，都需要一一手动复制么？其实也有更为简便的处理方案。社区中早就有了现成的工具，通过 hoist-non-react-statics 来处理，可以自动复制所有静态函数。如下代码所示。
>
> 复制代码
>
> ```
> import hoistNonReactStatics from 'hoist-non-react-statics';
> const PV = (pageName) => {
>   return (WrappedComponent) => {
>     class Wrap extends Component {
>       componentDidMount() {
>         trackPageView(pageName)
>       }
>  
>       render() {
>         return (
>           <WrappedComponent {...this.props} />
>         );
>       }
>     }
>      hoistNonReactStatics(Wrap, WrappedComponent);
>      return Wrap;
>   };
>  }
> ```
>
> 虽然缺少官方的解决方案，但社区方案弥补了不足。除了静态函数的问题以外，还有 refs 属性不能透传的问题。
>
> **refs 属性不能透传**
>
> ref 属性由于被高阶组件包裹了一次，所以需要进行特殊处理才能获取。React 为我们提供了一个名为 React.forwardRef 的 API 来解决这一问题，以下是官方文档中的一个案例：
>
> 复制代码
>
> ```
> function withLog(Component) {
>   class LogProps extends React.Component {
>     componentDidUpdate(prevProps) {
>       console.log('old props:', prevProps);
>       console.log('new props:', this.props);
>     }
>     render() {
>       const {forwardedRef, ...rest} = this.props;
>       // 将自定义的 prop 属性 “forwardedRef” 定义为 ref
>       return <Component ref={forwardedRef} {...rest} />;
>     }
>   }
>   // 注意 React.forwardRef 回调的第二个参数 “ref”。
>   // 我们可以将其作为常规 prop 属性传递给 LogProps，例如 “forwardedRef”
>   // 然后它就可以被挂载到被 LogProps 包裹的子组件上。
>   return React.forwardRef((props, ref) => {
>     return <LogProps {...props} forwardedRef={ref} />;
>   });
> }
> ```
>
> 这段代码读起来会有点儿头皮发麻，它正确的阅读顺序应该是从最底下的 React.forwardRef 部分开始，通过 forwardedRef 转发 ref 到 LogProps 内部。
>
> #### 工程实践
>
> 通过以上的梳理，接下来看一下如何在目录中给组件安排位置。
>
> 复制代码
>
> ```
> src
> ├── components
> │   ├── basic
> │   ├── container
> │   └── hoc
> └── pages
> ```
>
> - 首先将最基本的展示组件放入 basic 目录中；
> - 然后将容器组件放入 container；
> - 高阶组件放入 hoc 中；
> - 将页面外层组件放在页面目录中；
> - 通过目录级别完成切分。
>
> 在开发中，针对 basic 组件，建议使用类似 Storybook 的工具进行组件管理。因为Storybook 可以有组织地、高效地构建基础组件，有兴趣的话可以查阅下它的[官网](https://storybook.js.org)。
>
> ### 答题
>
> 通过以上的归类分析，关于 React 组件设计，我们的脑海中就有比较清晰的认知了。
>
> > React 组件应从设计与工程实践两个方向进行探讨。
> >
> > 从设计上而言，社区主流分类的方案是展示组件与灵巧组件。
> >
> > 展示组件内部没有状态管理，仅仅用于最简单的展示表达。展示组件中最基础的一类组件称作代理组件。代理组件常用于封装常用属性、减少重复代码。很经典的场景就是引入 Antd 的 Button 时，你再自己封一层。如果未来需要替换掉 Antd 或者需要在所有的 Button 上添加一个属性，都会非常方便。基于代理组件的思想还可以继续分类，分为样式组件与布局组件两种，分别是将样式与布局内聚在自己组件内部。
> >
> > 灵巧组件由于面向业务，其功能更为丰富，复杂性更高，复用度低于展示组件。最经典的灵巧组件是容器组件。在开发中，我们经常会将网络请求与事件处理放在容器组件中进行。容器组件也为组合其他组件预留了一个恰当的空间。还有一类灵巧组件是高阶组件。高阶组件被 React 官方称为 React 中复用组件逻辑的高级技术，它常用于抽取公共业务逻辑或者提供某些公用能力。常用的场景包括检查登录态，或者为埋点提供封装，减少样板代码量。高阶组件可以组合完成链式调用，如果基于装饰器使用，就更为方便了。高阶组件中还有一个经典用法就是反向劫持，通过重写渲染函数的方式实现某些功能，比如场景的页面加载圈等。但高阶组件也有两个缺陷，第一个是静态方法不能被外部直接调用，需要通过向上层组件复制的方式调用，社区有提供解决方案，使用 hoist-non-react-statics 可以解决；第二个是 refs 不能透传，使用 React.forwardRef API 可以解决。
> >
> > 从工程实践而言，通过文件夹划分的方式切分代码。我初步常用的分割方式是将页面单独建立一个目录，将复用性略高的 components 建立一个目录，在下面分别建立 basic、container 和 hoc 三类。这样可以保证无法复用的业务逻辑代码尽量留在 Page 中，而可以抽象复用的部分放入 components 中。其中 basic 文件夹放展示组件，由于展示组件本身与业务关联性较低，所以可以使用 Storybook 进行组件的开发管理，提升项目的工程化管理能力。
>
> 还可以通过以下知识导图来检验你的学习成果，看是否能将每部分补充完整。
>
> ![图片2 (1).png](https://s0.lgstatic.com/i/image/M00/84/27/CgqCHl_TIbaAEiBdAAEujtJGnY8994.png)
>
> ### 进阶
>
> **“如何在渲染劫持中为原本的渲染结果添加新的样式？”** 这个问题也经常被追问，其实并不难，但是有可能考察手写代码，所以这里我会做一些提示。
>
> 首先回滚上面的案例，在调用 super.render 的时候就可以拿到原本的渲染结果。
>
> 复制代码
>
> ```
> function withLoading(WrappedComponent) {
>     return class extends WrappedComponent {
>         render() {
>             if(this.props.isLoading) {
>                 return <Loading />;
>             } else {
>                 return super.render();
>             }
>         }
>     };
> }
> ```
>
> 那 super.render() 返回的是什么呢？你可以结合 JSX 一讲中的内容思考下。
>
> ### 总结
>
> 在本讲中主要对 React 组件的设计模式进行了梳理与回顾，并探讨了设计模式在工程实践中的作用。
>
> 在面试中面试官不仅希望听到设计模式有哪些，社区的推荐方式有哪些，更希望听到**模式具体用在哪儿**。如果你知道具体的场景，就会显得更有经验。设计模式并非有确定的标准答案，社区流行的分类方式也并非万能。如果你有自己的见解，在面试中与面试官进行探讨，也是非常值得鼓励的。
>
> 下一讲我将会介绍 React 中的一个关于 setState 的经典面试题：“setState 是同步更新还是异步更新”。