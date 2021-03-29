[前端面试宝典之 React 篇](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5808)



> 关于 React Hooks 还有一个高概率命中的题，就是聊设计模式。跟组件一样，如何协调地组合相关的代码是一个强需求，所以设计模式是一个不可避免的问题。
>
> ### 审题
>
> 但我个人感觉，这是一个处于只能谈一谈，或者说聊一聊的问题。这是为什么呢？自然有它的原因。
>
> （1）Hooks 整体发展时间不长。
>
> 从 18 年推出 Hooks 到现在，整体时间不算长，从成熟度上来讲还不能够与类组件的开发模式相提并论，有些问题还没得到解决。比如在官方文档中给出了获取上一轮 props 或 state 的方案，代码如下所示：
>
> 复制代码
>
> ```
> function Counter() {
>   const [count, setCount] = useState(0);
>   const prevCountRef = useRef();
>   useEffect(() => {
>     prevCountRef.current = count;
>   });
>   const prevCount = prevCountRef.current;
>   return <h1>Now: {count}, before: {prevCount}</h1>;
> }
> ```
>
> 在这段代码里，先是通过 useRef 函数生成一个 ref，然后将当前的 count 写入 ref.current 作为缓存，以此保证每次能够获取上一轮的 state。
>
> 这个设计使用了些许小技巧，虽然能解决问题，但不够直观。虽然官方也提出了，可以将这个过程封装为一个 usePrevious 的 Hooks，社区也提供了这样的 Hooks，但就开发心智而言，确实是个负担。
>
> 与此类似的还有在 Hooks 的开发模式下完成 getDerivedStateFromProps 的操作，对心智的挑战也很大。这个时候你会发现有时使用类组件可能更好理解。
>
> （2）Hooks 并不会改变组件本身的设计模式。
>
> 比如在第 05 讲“如何设计 React 组件？”提到的展示组件、容器组件、高阶组件等概念，并不会因为使用 Hooks 而改变。因为 Hooks 并不是解决组件如何复用的问题，而是解决内部逻辑抽象复用的问题。以前是通过生命周期的方式思考逻辑如何布局，而现在是以事务的角度归纳合并。所以这个变化对于开发者的心智挑战也很大。
>
> 所以在缺乏绝对权威方案的情况下，对于 React Hooks 的设计模式只能是处于谈一谈、聊一聊的开放式状态。对于这类开放式问题，我们依然需要遵循一定的逻辑来陈述自己的观点。最好还能由浅及深，逐步递进地展开自己的想法。
>
> - 在论述前需要先建立一个认知基础，表明自身对 Hooks 整体的运用理解，这就是“道”；
> - 再结合实践谈技巧上的常规操作与个人实践，这就是“术”；
> - 最后呢，结合自身经验聊一聊 Hooks 在工程实践中的组合方式，这就是“势”。
>
> 当然，这并不是唯一的答案，只是提供了一个思路，要注意，只要是逻辑自洽的答案都是好答案。
>
> ### 承题
>
> 我们就可以根据上文分析到的“道、术、势”来答题了：道就是认知基础；术就是常规操作与个人实践；势就是工程实践上的运用。
>
> ![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/08/34/CgpVE2AKhZCAHGhGAABO7J0ONKw763.png)
>
> ### 破题
>
> #### 道
>
> Dan 在 React Hooks 的介绍中曾经说过：“忘记生命周期，以 effects 的方式开始思考”，这样的思维转换，能帮助我们更好地处理代码。
>
> 下面还是用一个聊天组件来举例，代码如下所示：
>
> 复制代码
>
> ```
> class ChatChannel extends Component {
>   state = {
>     messages: [];
>   }
>   componentDidMount() {
>     this.subscribeChannel(this.props.channelId);
>   }
> 
>   componentDidUpdate(prevProps) {
>     if (this.props.channelId !== prevProps.channelId) {
>       this.unSubscribeChannel(prevProps.channelId);
>       this.subscribeChannel(this.props.channelId);
>     }
>   }
>   componentWillUnmount() {
>     this.unSubscribeChannel(this.props.channelId);
>   }
> 
> 
>   subscribeChannel = (channelId) => {
>     ChatAPI.subscribe(
>       channelId,
>       message => {
>         this.setState(state => {
>           return { messages: [...state.messages, message] };
>         });
>       }
>     );
>   }
> 
>   unSubscribeChannel = (channelId) => {
>      ChatAPI.unSubscribe(channelId);
>   }
>   render() {
>     // 组件样式
>     // ...
>   }
> }
> ```
>
> 这个组件命名为**ChatChannel**，它的作用是**根据传入的 channelId 订阅相关 channel 的信息**，也就是 messages，并展示内容。因为展示界面 UI 不是要说明的重点，所以这里的代码就暂时省略了。
>
> 其中用于订阅 channel 的函数是 subscribeChannel，用于取消订阅 channel 的函数是 unSubscribeChannel，内部都是调用 ChatAPI 实现。
>
> 根据生命周期的思路，在 componentDidMount 去订阅 channel，在 componentWillUnmount 取消。那如果外部传入的 channelId 变化了呢？这里通过 componentDidUpdate 去处理，先取消上一次订阅的 channel，再订阅新的 channel。你会发现整个逻辑不只我讲起来拗口，代码看起来也费劲。
>
> 如果从生命周期转换到 Hooks 去思考，这段代码将会被大量简化。因为你只需要抓住一个点就够了，那就是 channelId 与 channel 一一对应关联，channelId 切换时，自动取消订阅。写成的代码就像下面这样：
>
> 复制代码
>
> ```
> const ChatChannel = ({ channelId }) => {
>   const [messages, setMessages] = useState([]);
>   useEffect(() => {
>    ChatAPI.subscribe(
>       channelId,
>       message => {
>         this.setState(state => {
>           return { messages: [...state.messages, message] };
>         });
>       }
>     );
>     return () => ChatAPI.unSubscribe(channelId));
>   }, [channelId]);
> 
>   // 组件样式
>   return ...
> }
> ```
>
> 整个逻辑只需要一个 effect 就可以完成了，订阅的操作是 useEffect 的第一个函数，这个函数的返回值是一个取消订阅的函数。useEffect 的第二个参数是 channelId，当 channelId 变化时会优先执行取消订阅的函数，再执行订阅。
>
> 经过 Hooks 的改造后，只需要一个 effect 就完整实现了需要多个生命周期函数才能完成的业务逻辑。如果希望进一步抽象与复用，只需要将 effect 中的代码抽为自定义 Hook 就行了。这远比生命周期来得简单。
>
> #### 术
>
> 接下来将介绍一些 Hooks 中容易遇到的问题。
>
> **（1）React.memo vs React.useMemo**
>
> 在第 04 讲[“类组件与函数组件有什么区别呢？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0#/detail/pc?id=5794)中有提到，React.memo 是一个高阶组件，它的效果类似于 React.pureComponent。但在 Hooks 的场景下，更推荐使用 React.useMemo，因为它存在这样一个问题。就像如下的代码一样：
>
> 复制代码
>
> ```
> function Banner() {
>   let appContext = useContext(AppContext);
>   let theme = appContext.theme;
>   return <Slider theme={theme} />
> }
> export default React.memo(Banner)
> ```
>
> 这段代码的意义是这样的，通过 useContext 获取全局的主题信息，然后给 Slider 组件换上主题。但是如果给最外层的 Banner 组件加上 React.memo，那么外部更新 appContext 的值的时候，Slider 就会被触发重渲染。
>
> 当然，我们可以通过**分拆组件**的方式阻断重渲染，但使用 React.useMemo 可以实现更精细化的控制。就像下面的代码一样，为 Slider 组件套上 React.useMemo，写上 theme 进行控制。
>
> 复制代码
>
> ```
> function Banner() {
>   let appContext = useContext(AppContext);
>   let theme = appContext.theme;
>   return React.useMemo(() => {
>     return <Slider theme={theme} />;
>   }, [theme])
> }
> export default React.memo(Banner)
> ```
>
> 所有考虑到更宽广的使用场景与可维护性，更推荐使用 React.useMemo。
>
> **（2）常量**
>
> 由于函数组件每次渲染时都会重新执行，所以常量应该放置到函数外部去，避免每次都重新创建。而如果定义的常量是一个函数，且需要使用组件内部的变量做计算，那么一定要使用 useCallback 缓存函数。
>
> **（3）useEffect 第二个参数的判断问题**
>
> 在设计上它同样是进行浅比较，如果传入的是引用类型，那么很容易会判定不相等，所以尽量不要使用引用类型作为判断条件，很容易出错。
>
> #### 势
>
> 你在开发时，想过这样一个问题吗，就是如何将业务逻辑的 Hooks 组合起来？每个团队肯定都有自己的见解和方案，那么下面将举一个出自 The Facade pattern and applying it to React Hooks 的案例来看看如何组合 Hooks。
>
> 在这个案例中将 User 的所有操作归到一个自定义 Hook 中去操作，最终返回的值有 users、addUsers 及 deleteUser。其中 users 是通过 useState 获取；addUser 是通过 setUsers 添加 user 完成；deleteUser 通过过滤 userId 完成。代码如下所示：
>
> 复制代码
>
> ```
> function useUsersManagement() {
>   const [users, setUsers] = useState([]);
>  
>   function addUser(user) {
>     setUsers([
>       ...users,
>       user
>     ])
>   }
>  
>   function deleteUser(userId) {
>     const userIndex = users.findIndex(user => user.id === userId);
>     if (userIndex > -1) {
>       const newUsers = [...users];
>       newUsers.splice(userIndex, 1);
>       setUsers(
>         newUsers
>       );
>     }
>   }
>  
>   return {
>     users,
>     addUser,
>     deleteUser
>   }
> }
> ```
>
> 第二部分是通过 useAddUserModalManagement 这一个自定义 Hook 控制 Modal 的开关。与上面的操作类似。isAddUserModalOpened 表示了当前处于 Modal 开关状态，openAddUserModal 则是打开，closeAddUserModal 则是关闭。如下代码所示：
>
> 复制代码
>
> ```
> function useAddUserModalManagement() {
>   const [isAddUserModalOpened, setAddUserModalVisibility] = useState(false);
>  
>   function openAddUserModal() {
>     setAddUserModalVisibility(true);
>   }
>  
>   function closeAddUserModal() {
>     setAddUserModalVisibility(false);
>   }
>   return {
>     isAddUserModalOpened,
>     openAddUserModal,
>     closeAddUserModal
>   }
> }
> ```
>
> 最后来看看在代码中运用的情况，引入 useUsersManagement 和 useAddUserModalManagement 两个自定义 Hook，然后在组件 UsersTable 与 AddUserModal 直接使用。UsersTable 直接展示 users 相关信息，通过操作 deleteUser 可以控制删减 User。AddUserModal 通过 isAddUserModalOpened 控制显隐，完成 addUser 操作。代码如下所示：
>
> 复制代码
>
> ```
> import React from 'react';
> import AddUserModal from './AddUserModal';
> import UsersTable from './UsersTable';
> import useUsersManagement from "./useUsersManagement";
> import useAddUserModalManagement from "./useAddUserModalManagement";
>  
> const Users = () => {
>   const {
>     users,
>     addUser,
>     deleteUser
>   } = useUsersManagement();
>   const {
>     isAddUserModalOpened,
>     openAddUserModal,
>     closeAddUserModal
>   } = useAddUserModalManagement();
>  
>   return (
>     <>
>       <button onClick={openAddUserModal}>Add user</button>
>       <UsersTable
>         users={users}
>         onDelete={deleteUser}
>       />
>       <AddUserModal
>         isOpened={isAddUserModalOpened}
>         onClose={closeAddUserModal}
>         onAddUser={addUser}
>       />
>     </>
>   )
> };
>  
> export default Users;
> ```
>
> 在上面的例子中，我们可以看到组件内部的逻辑已经被自定义 Hook 完全抽出去了。外观模式很接近第 05 讲[“如何设计 React 组件？”](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=566&sid=20-h5Url-0#/detail/pc?id=5795)提到的容器组件的概念，即在组件中通过各个自定义 Hook 去操作业务逻辑。每个自定义 Hook 都是一个独立的子模块，有属于自己的领域模型。基于这样的设计就可以避免 Hook 之间逻辑交叉，提升复用性。
>
> ### 答题
>
> > React Hooks 并没有权威的设计模式，很多工作还在建设中，在这里我谈一下自己的一些看法。
> >
> > 首先用 Hooks 开发需要抛弃生命周期的思考模式，以 effects 的角度重新思考。过去类组件的开发模式中，在 componentDidMount 中放置一个监听事件，还需要考虑在 componentWillUnmount 中取消监听，甚至可能由于部分值变化，还需要在其他生命周期函数中对监听事件做特殊处理。在 Hooks 的设计思路中，可以将这一系列监听与取消监听放置在一个 useEffect 中，useEffect 可以不关心组件的生命周期，只需要关心外部依赖的变化即可，对于开发心智而言是极大的减负。这是 Hooks 的设计根本。
> >
> > 在这样一个认知基础上，我总结了一些在团队内部开发实践的心得，做成了开发规范进行推广。
> >
> > 第一点就是 React.useMemo 取代 React.memo，因为 React.memo 并不能控制组件内部共享状态的变化，而 React.useMemo 更适合于 Hooks 的场景。
> >
> > 第二点就是常量，在类组件中，我们很习惯将常量写在类中，但在组件函数中，这意味着每次渲染都会重新声明常量，这是完全无意义的操作。其次就是组件内的函数每次会被重新创建，如果这个函数需要使用函数组件内部的变量，那么可以用 useCallback 包裹下这个函数。
> >
> > 第三点就是 useEffect 的第二个参数容易被错误使用。很多同学习惯在第二个参数放置引用类型的变量，通常的情况下，引用类型的变量很容易被篡改，难以判断开发者的真实意图，所以更推荐使用值类型的变量。当然有个小技巧是 JSON 序列化引用类型的变量，也就是通过 JSON.stringify 将引用类型变量转换为字符串来解决。但不推荐这个操作方式，比较消耗性能。
> >
> > 这是开发实践上的一些操作。那么就设计模式而言，还需要顾及 Hooks 的组合问题。在这里，我的实践经验是采用外观模式，将业务逻辑封装到各自的自定义 Hook 中。比如用户信息等操作，就把获取用户、增加用户、删除用户等操作封装到一个 Hook 中。而组件内部是抽空的，不放任何具体的业务逻辑，它只需要去调用单个自定义 Hook 暴露的接口就行了，这样也非常利于测试关键路径下的业务逻辑。
> >
> > 以上就是我在设计上的一些思考。
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image2/M01/08/32/Cip5yGAKhcCAKZY2AAB3d7s2Ur4216.png)
>
> ### 总结
>
> React Hook 是未来仍然需要关注的一个技术点，由于与类组件不同的心智模型，也希望你在开发中能更多地去使用它。你在使用 Hooks 时遇到过什么问题？有什么独特的解决方案？不妨在留言区留言，我将和你一起探讨。
>
> 从下一讲开始就进入 React 周边生态相关的面试题目，首先是 React Router。 React Router 是一个非常长寿的库，可以说在 React 生态中处于屹立不倒的位置，笑看其他库流行起来又衰落下去。在下一讲中，我将与你一起探索它的秘密。