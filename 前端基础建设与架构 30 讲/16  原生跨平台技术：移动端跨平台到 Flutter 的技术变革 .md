[前端基础建设与架构 30 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=584&sid=20-h5Url-0&buyFrom=2&pageId=1pz4#/detail/pc?id=5921)



> 跨平台其实是一个老生常谈的话题，技术方案也是历经变迁，但始终热点不断，究其原因有二：
>
> - 首先，移动端原生技术需要配备 iOS 和 Android 两套团队和技术栈，且存在**发版周期限制**，开发效率上存在天然缺陷；
> - 其次，原生跨平台技术虽然“出道”较早，但是各方案都难以做到完美，因此也**没有大一统的技术垄断**。
>
> 这一讲我们就从历史角度出发，剖析原生跨平台技术的原理，同时梳理相关技术热点，和你聊一聊 Flutter 背后的技术变革。
>
> ### 移动端跨平台技术原理和变迁
>
> 移动端跨平台是一个美好的愿景。我们先来看一下移动端跨平台技术发展的时间线：
>
> ![Lark20210202-183723.png](https://s0.lgstatic.com/i/image/M00/94/AD/Ciqc1GAZK5OADuyQAAIWqX8rFbk913.png)
>
> #### 基于 WebView 到 JSBridge 的 Hybrid 方案发展
>
> 最早的实践就是通过 WebView 双端运行 Web 代码。事实上，虽然 iOS 和 Android 系统难以统一，但是它们都对 Web 技术开放，于是有的人开玩笑：“不管是 Mac、Windows、Linux、iOS、Android 还是其他平台，只要给一个浏览器，连‘月球’上它都能跑。”因此，**Hybrid 方案**算是最古老，但也是**最成熟、应用最为广泛**的技术。
>
> 在 iOS 和 Android 系统上运行 JavaScript 并不是一件难事儿，但是对于一个真正意义上的跨平台应用来说，还需要做到**H5（即 WebView 容器）和原生平台的交互**，于是 JSBridge 技术就诞生了。
>
> JSBridge 原理很简单，我们知道，在原生平台中，JavaScript 代码是运行在一个单独的 JS Context 中（比如 WebView 的 WebKit 引擎、JavaSriptCore 等），这个独立的上下文和原生能力的交互过程是双向的，我们可以从两个方面简要分析。
>
> 1. JavaScript 调用 Native，方法有：
>    - 注入 APIs；
>    - 拦截 URL Scheme。
> 2. Native 调用 JavaScript。
>
> 前端调用原生能力主要有两种方式，注入 APIs 其实就是原生平台通过 WebView 提供的接口，向 JavaScript Context 中（一般使用 Window 对象），注入相关方案和数据；另一种拦截 URL Scheme 就更加简单了，前端通过发送定义好的 URL Scheme 请求，并将相关数据放在请求体中，该请求被原生平台拦截后，由原生平台做出响应。
>
> Native 调用 JavaScript，实现起来也很简单。因为 Native 实际上是 WebView 的宿主，因此 Native 具有更大权限，故而**原生平台可以通过 WebView APIs 直接执行 JavaScript 代码**。
>
> 随着 JSBridge 这种方式实现跨平台技术的成熟，社区上出现了 Cordova、Ionic 等框架，它们**本质上都是使用 HTML、CSS 和 JavaScript 进行跨平台原生应用的开发**。该方案说到底是在 iOS 和 Androd 上运行 Web 应用，因此也存在较多问题，比如：
>
> - JavaScript Context 和原生通信频繁，导致性能体验较差；
> - 页面逻辑由前端负责，组件也是前端渲染，也造成了性能短板；
> - 运行 JavaScript 的 WebView 内核在各平台上不统一；
> - 国内厂商对于系统的深度定制，导致内核碎片化。
>
> 因此，新一代的 Hybrid 跨平台方式，以**React Native 为代表**的方案就诞生了，这种方案主要思想是：开发者依然**使用 Web 语言（如 React 框架或其他 DSL），但渲染基本交给原生平台处理。** 这样一来，在视图层面就可以摆脱 WebView 的束缚，保障了开发体验和效率，以及使用性能。我把这种技术叫作基于 OEM 的 Hybrid 方案。
>
> React Native 脱胎于 React 理念，它将数据与视图相隔离，**React Native 代码中的标签映射为虚拟节点，由原生平台解析虚拟节点并渲染出原生组件**。美好的愿景是：开发者使用 React 语法，同时开发原生应用和 Web 应用，其中组件渲染、动画效果、网络请求等都由原生平台来负责。整体技术架构如下图：
>
> ![Lark20210202-183726.png](https://s0.lgstatic.com/i/image/M00/94/AD/Ciqc1GAZK5-ASl0rAAMYrAXB4Po374.png)
>
> React Native 技术架构图
>
> 如上图，React Native 主要由：
>
> - JavaScript
> - C++ 适配层
> - iOS/Androd
>
> 三层组成，最重要的**C++ 层实现了动态链接库**，起到了衔接适配前端和原生平台作用，这个衔接具体指：使用 JavaScriptCore 解析 JavaScript 代码（iOS 上不允许用自己的 JS Engine，iOS 7+ 默认使用 JavaScriptCore，Android 也默认使用 JavaScriptCore），通过 [MessageQueue.js](https://github.com/facebook/react-native/blob/master/Libraries/BatchedBridge/MessageQueue.js) 实现双向通信，**实际上通信格式类似 JSON-RPC**。
>
> 这里我们以一个从 JavaScript 传递数据给原生平台 [UIManager](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/uimanager/UIManagerModule.java) 更新页面视图为例，了解下数据信息内容：
>
> 复制代码
>
> ```
> 2584 I ReactNativeJS: Running application "MoviesApp" with appParams: {"initialProps":{},"rootTag":1}. __DEV__ === false, development-level warning are OFF, performance optimizations are ON
> 2584 I ReactNativeJS: JS->N : UIManager.createView([4,"RCTView",1,{"flex":1,"overflow":"hidden","backgroundColor":-1}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([5,"RCTView",1,{"flex":1,"backgroundColor":0,"overflow":"hidden"}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([6,"RCTView",1,{"pointerEvents":"auto","position":"absolute","overflow":"hidden","left":0,"right":0,"bottom":0,"top":0}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([7,"RCTView",1,{"flex":1,"backgroundColor":-1}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([8,"RCTView",1,{"flexDirection":"row","alignItems":"center","backgroundColor":-5658199,"height":56}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([9,"RCTView",1,{"nativeBackgroundAndroid":{"type":"ThemeAttrAndroid","attribute":"selectableItemBackgroundBorderless"},"accessible":true}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([10,"RCTImageView",1,{"width":24,"height":24,"overflow":"hidden","marginHorizontal":8,"shouldNotifyLoadEvents":false,"src":[{"uri":"android_search_white"}],"loadingIndicatorSrc":null}])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([9,[10]])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([12,"AndroidTextInput",1,{"autoCapitalize":0,"autoCorrect":false,"placeholder":"Search a movie...","placeholderTextColor":-2130706433,"flex":1,"fontSize":20,"fontWeight":"bold","color":-1,"height":50,"padding":0,"backgroundColor":0,"mostRecentEventCount":0,"onSelectionChange":true,"text":"","accessible":true}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([13,"RCTView",1,{"alignItems":"center","justifyContent":"center","width":30,"height":30,"marginRight":16}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([14,"AndroidProgressBar",1,{"animating":false,"color":-1,"width":36,"height":36,"styleAttr":"Normal","indeterminate":true}])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([13,[14]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([8,[9,12,13]])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([15,"RCTView",1,{"height":1,"backgroundColor":-1118482}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([16,"RCTView",1,{"flex":1,"backgroundColor":-1,"alignItems":"center"}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([17,"RCTText",1,{"marginTop":80,"color":-7829368,"accessible":true,"allowFontScaling":true,"ellipsizeMode":"tail"}])
> 2584 I ReactNativeJS: JS->N : UIManager.createView([18,"RCTRawText",1,{"text":"No movies found"}])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([17,[18]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([16,[17]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([7,[8,15,16]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([6,[7]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([5,[6]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([4,[5]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([3,[4]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([2,[3]])
> 2584 I ReactNativeJS: JS->N : UIManager.setChildren([1,[2]])
> ```
>
> 下面数据是一个 touch 交互信息，JavaScriptCore 传递用户 touch 信息给原生平台：
>
> 复制代码
>
> ```
> 2584 I ReactNativeJS: N->JS : RCTEventEmitter.receiveTouches(["topTouchStart",[{"identifier":0,"locationY":47.9301872253418,"locationX":170.43936157226562,"pageY":110.02542877197266,"timestamp":2378613,"target":26,"pageX":245.4869842529297}],[0]])
> 2584 I ReactNativeJS: JS->N : Timing.createTimer([18,130,1477140761852,false])
> 2584 I ReactNativeJS: JS->N : Timing.createTimer([19,500,1477140761852,false])
> 2584 I ReactNativeJS: JS->N : UIManager.setJSResponder([23,false])
> 2584 I ReactNativeJS: N->JS : RCTEventEmitter.receiveTouches(["topTouchEnd",[{"identifier":0,"locationY":47.9301872253418,"locationX":170.43936157226562,"pageY":110.02542877197266,"timestamp":2378703,"target":26,"pageX":245.4869842529297}],[0]])
> 2584 I ReactNativeJS: JS->N : UIManager.clearJSResponder([])
> 2584 I ReactNativeJS: JS->N : Timing.deleteTimer([19])
> 2584 I ReactNativeJS: JS->N : Timing.deleteTimer([18])
> ```
>
> 除了 UI 渲染、交互信息、网络调用等通信也都是通过 MessageQueue 来完成，如下数据， JavaScriptCore 传递网络请求信息给原生平台：
>
> 复制代码
>
> ```
> 5835 I ReactNativeJS: JS->N : Networking.sendRequest(["GET","http://api.rottentomatoes.com/api/public/v1.0/lists/movies/in_theaters.json?apikey=7waqfqbprs7pajbz28mqf6vz&page_limit=20&page=1",1,[],{"trackingName":"unknown"},"text",false,0])
> 5835 I ReactNativeJS: N->JS : RCTDeviceEventEmitter.emit(["didReceiveNetworkResponse",[1,200,{"Connection":"keep-alive","X-Xss-Protection":"1; mode=block","Content-Security-Policy":"frame-ancestors 'self' rottentomatoes.com *.rottentomatoes.com ;","Date":"Sat, 22 Oct 2016 13:58:53 GMT","Set-Cookie":"JSESSIONID=63B283B5ECAA9BBECAE253E44455F25B; Path=/; HttpOnly","Server":"nginx/1.8.1","X-Content-Type-Options":"nosniff","X-Mashery-Responder":"prod-j-worker-us-east-1b-115.mashery.com","Vary":"User-Agent,Accept-Encoding","Content-Language":"en-US","Content-Type":"text/javascript;charset=ISO-8859-1","X-Content-Security-Policy":"frame-ancestors 'self' rottentomatoes.com *.rottentomatoes.com ;"},"http://api.rottentomatoes.com/api/public/v1.0/lists/movies/in_theaters.json?apikey=7waqfqbprs7pajbz28mqf6vz&page_limit=20&page=1"]])
> 5835 I ReactNativeJS: N->JS : RCTDeviceEventEmitter.emit(["didReceiveNetworkData",[1,"{\"total\":128,\"movies\":[{\"id\":\"771419323\",\"title\":\"The Accountant\",\"year\":2016,\"mpaa_rating\":\"R\",\"runtime\":128,\"critics_consensus\":\"\",\"release_dates\":{\"theater\":\"2016-10-14\"},\"ratings\":{\"critics_rating\":\"Rotten\",\"critics_score\":50,\"audience_rating\":\"Upright\",\"audience_score\":86},\"synopsis\":\"Christian Wolff (Ben Affleck) is a math savant with more affinity for numbers than people. Behind the cover of a small-town CPA office, he works as a freelance accountant for some of the world's most dangerous criminal organizations. With the Treasury Department's Crime Enforcement Division, run by Ray King (J.K. Simmons), starting to close in, Christian takes on a legitimate client: a state-of-the-art robotics company where an accounting clerk (Anna Kendrick) has discovered a discrepancy involving millions of dollars. But as Christian uncooks the books and gets closer to the truth, it is the body count that starts to rise.\",\"posters\":{\"thumbnail\":\"https://resizing.flixster.com/r5vvWsTP7cdijsCrE5PSmzle-Zo=/54x80/v1.bTsxMjIyMzc0MTtqOzE3MTgyOzIwNDg7NDA1MDs2MDAw\",\"profile\":\"https://resizing.flixster.com/r5vvWsTP7cdijsCrE5PSmzle-Zo=/54x80/v1.bTsxMjIyMzc0MTtqOzE3MTgyOzIwNDg7NDA1MDs2MDAw\",\"detailed\":\"https://resizing.flixster.com/r5vvWsTP7cdijsCrE5PSmzle-Zo=/54x80/v1.bTsxMjIyMzc0MTtqOzE3MTgyOzIwNDg7NDA1MDs2MDAw\",\"original\":\"https://resizing.flixster.com/r5vvWsTP7cdijsCrE5PSmzle-Zo=/54x80/v1.bTsxMjIyMzc0MTtqOzE3MTgyOzIwNDg7NDA1MDs2MDAw\"},\"abridged_cast\":[{\"name\":\"Ben Affleck\",\"id\":\"162665891\",\"characters\":[\"Christian Wolff\"]},{\"name\":\"Anna Kendrick\",\"id\":\"528367112\",\"characters\":[\"Dana Cummings\"]},{\"name\":\"J.K. Simmons\",\"id\":\"592170459\",\"characters\":[\"Ray King\"]},{\"name\":\"Jon Bernthal\",\"id\":\"770682766\",\"characters\":[\"Brax\"]},{\"name\":\"Jeffrey Tambor\",\"id\":\"162663809\",\"characters\":[\"Francis Silverberg\"]}],\"links\":{\"self\":\"http://api.rottentomatoes.com/api/public/v1.0/movies/771419323.json\",\"alternate\":\"http://www.rottentomatoes.com/m/the_accountant_2016/\",\"cast\":\"http://api.rottentomatoes.com/api/public/v1.0/movies/771419323/cast.json\",\"reviews\":\"http://api.rottentomatoes.com/api/public/v1.0/movies/771419323/reviews.json\",\"similar\":\"http://api.rottentomatoes.com/api/public/v1.0/movies/771419323/similar.json\"}},{\"id\":\"771359360\",\"title\":\"Miss Peregrine's Home for Peculiar Children\",\"year\":2016,\"mpaa_rating\":\"PG-13\",\"runtime\":127,\"critics_consensus\":\"\",\"release_dates\":{\"theater\":\"2016-09-30\"},\"ratings\":{\"critics_rating\":\"Fresh\",\"critics_score\":64,\"audience_rating\":\"Upright\",\"audience_score\":65},\"synopsis\":\"From visionary director Tim Burton, and based upon the best-selling novel, comes an unforgettable motion picture experience. When Jake discovers clues to a mystery that spans different worlds and times, he finds a magical place known as Miss Peregrine's Home for Peculiar Children. But the mystery and danger deepen as he gets to know the residents and learns about their special powers...and their powerful enemies. Ultimately, Jake discovers that only his own special \\\"peculiarity\\\" can save his new friends.\",\"posters\":{\"thumbnail\":\"https://resizing.flixster.com/H1Mt4WpK-Mp431M7w0w7thQyfV8=/54x80/v1.bTsxMTcwODA4MDtqOzE3MTI1OzIwNDg7NTQwOzgwMA\",\"profile\":\"https://resizing.flixster.com/H1Mt4WpK-Mp431M7w0w7thQyfV8=/54x80/v1.bTsxMTcwODA4MDtqOzE3MTI1OzIwNDg7NTQwOzgwMA\",\"detailed\":\"https://resizing.flixster.com/H1Mt4WpK-Mp431M7w0w7thQyfV8=/54x80/v1.bTsxMTcwODA4MDtqOzE3MTI1OzIwNDg7NTQwOzgwMA\",\"original\":\"https://resizing.flixster.com/H1Mt4WpK-Mp431M7w0w7thQyfV8=/54x80/v1.bTsxMTcwODA4MDtqOzE3MTI1OzIwNDg7NTQwOzgwMA\"},\"abridged_cast\":[{\"name\":\"Eva Green\",\"id\":\"162652241\",\"characters\":[\"Miss Peregrine\"]},{\"name\":\"Asa Butterfield\",\"id\":\"770800323\",\"characters\":[\"Jake\"]},{\"name\":\"Chris O'Dowd\",\"id\":\"770684214\",\"char
> 5835 I ReactNativeJS: N->JS : RCTDeviceEventEmitter.emit(["didCompleteNetworkResponse",[1,null]])
> ```
>
> 这样的效果显而易见，**通过前端能力，实现了原生应用的跨平台，快速编译、快速发布**。但是缺点也比较明显：上述数据通信过程是异步的，**通信成本很高**。除此之外，目前 React Native 仍有部分组件和 API 并没有实现平台统一，也在一定程度上需要开发者了解原生开发细节。正因如此，社区上也出现了著名文章[《React Native at Airbnb》](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c)，文中表示 Airbnb 团队在技术选型上将会放弃 React Native。
>
> 在我看来，放弃 React Native，拥抱新的跨平台技术并不是每个团队都有实力和魄力施行的，而改造 React Native 是另外一些团队做出的选择。
>
> 比如**携程的 CRN**（Ctrip React Native）。它在 React Native 基础上，抹平了 iOS 和 Android 端组件开发差异，做了大量性能提升的工作。更重要的是，依托于 CRN，它在后续的产品 CRN-Web 也做了 Web 支持和接入。再比如，更加出名的**Alibaba 出品的 WEEX**也是基于 React Native 思想进行的某种改造，只不过 WEEX 基于 Vue，除了支持原生平台，还支持了 Web 平台，实现了端上的大一统。WEEX 技术架构如下图（出自官网）：
>
> ![Lark20210202-183709.png](https://s0.lgstatic.com/i/image/M00/94/AD/Ciqc1GAZK82AKeB3AADlYd3QMbY205.png)
>
> WEEX 技术架构图
>
> 话题再回到 React Native，针对一些固有缺陷，React Native 进行了技术上的重构，我认为这是基于 OEM Hybrid 方案的 2.0 演进，下面我们进一步探究。
>
> #### 从 React Native 技术重构出发，分析原生跨平台技术栈方向
>
> 上文我们提到，React Native 通过数据通信架起了 Web 和原生平台的桥梁，而这个数据通信方式是异步的。React 工程经理 Sophie Alpert 将这种异步通信方式称为 Asynchronous Bridge，这样的设计获得了线程隔离的便利，具备了尽可能的灵活性，但是这也意味着 JavaScript 逻辑与原生能力永远无法处在同一个时空，无法共享一个内存空间。
>
> 老架构图：
>
> ![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/94/8A/Ciqc1GAYzjeAD4rFAAxfnuLVOGI871.png)
>
> 对比新的架构图：
>
> ![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/94/95/CgqCHmAYzj-ANDk6AAyPBuKOtmk965.png)
>
> 基于这个问题，新的 React Native 技术架构将从三个方面进行革新。
>
> 1. **改变线程模型**（Threading Model），以往 React Native 的 UI 更新需要在三个不同的线程进行，新的方案使具有高优先级更新的线程，直接同步调用 JavaScript；同时低优先级的 UI 更新任务不会占用主线程。这里提到的三个并行线程是：
>    - **JavaScript 线程**，在这个线程中，Metro 负责生成 JS bundles，JavaScriptCore 负责在应用运行时解析执行 JavaScript 代码；
>    - **原生线程**，这个线程负责用户界面，每当需要更新 UI 时，该线程将会与 JavaScript 线程通信，可以细分为原生模块和原生 UI；
>    - **Shadow 线程**，该线程负责计算布局，React Native 具体通过 Yoga 布局引擎来解析并计算 Flexbox 布局，并将结果发送回原生 UI 线程。
> 2. **引入**[异步渲染能力](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)，实现不同优先级的渲染，同时简化渲染数据信息。
> 3. **简化 Bridge 实现**，使之更轻量可靠，使 JavaScript 和原生平台的调用更加高效。
>
> 举个例子，这些改造能够使得“**手势处理**”——这个 React Native 老大难问题得到更好的解决，比如新的线程模型能够使手势触发的交互和 UI 渲染效率更高，减少异步通信更新 UI 成本，使视图尽快响应用户的交互。
>
> 我们从更细节的角度来加深理解。上述重构的核心之一其实是使用基于 JavaScript Interface (JSI) 的新 Bridge 方案来取代之前的 Bridge 方案。
>
> 新的 Bridge 方案由两部分组成：
>
> - **Fabric，新的 UIManager**；
> - **TurboModules，新的原生模块**。
>
> 其中 Fabric 运行 UIManager**直接用 C++ 生成 Shadow Tree**，而不需要走一遍老架构的 React → Native → Shadow Tree → Native UI 路径。这就**减少了通信成本，提升交互性能**。这个过程依赖于 JSI，JSI 并不和 JavaScriptCore 绑定，因此我们可以实现引擎互换（比如使用 V8，或任何其他版本的 JavaScriptCore）。
>
> 同时，JSI 可以获取 C++ Host Objects，并调用 Host Objects 上的方法，这样能够完成**JavaScript 和原生平台的直接感知**，达到“所有线程之间的互相调用操作”，因此我们就不再依赖“将传递消息序列化，并进行异步通信”了。这也就消除了异步通信带来的拥塞等问题。
>
> 新的方案也允许 JavaScript 代码**仅在真正需要时加载每个模块**，如果应用中并不需要使用 Native Modules（例如蓝牙功能），那么它就不会在程序打开时被加载，这样就可以提升应用的启动时间。
>
> 总之，新的 React Native 架构将会在 Hybrid 思想上将性能优化做到极致，我们可以密切关注相关技术的发展。一条路优化到底是我们需要学习的，另辟蹊径也是我们需要掌握的，接下来，我们看看 Flutter 技术如何从另一个赛道出发，革新了跨平台技术方案。
>
> ### Flutter 新贵背后的技术变革
>
> Flutter**采用了 Dart 编程语言**，它在技术设计上不同于 React Native 的一个显著特点是：**Flutter 并非使用原生平台组件进行渲染**。比如在 React Native 中，一个`<view>`组件会被最终编译为 iOS 平台的 UIView Element 以及 Android 平台的 View Element，但在 Flutter 中，Flutter 自身提供一组组件集合，这些组件集合被 Flutter 框架和引擎直接接管。如下图（出自 [Cross-Platform Mobile Development Using Flutter](https://www.codemag.com/Article/1909091/Cross-Platform-Mobile-Development-Using-Flutter)）：
>
> ![Lark20210202-183731.png](https://s0.lgstatic.com/i/image/M00/94/AD/Ciqc1GAZK76AWfZpAApvFObEYiY127.png)
>
> Flutter 工作原理图
>
> Flutter 组件依靠自身高性能的渲染引擎进行视图的渲染。具体来说，每一个组件会被渲染在 Skia 上，Skia 是一个 2D 的绘图引擎库，具有跨平台特点。Skia 唯一需要的就是**原生平台提供 Canvas 接口，实现绘制**。我们再通过一个横向架构图来了解实现细节：
>
> ![Lark20210202-183729.png](https://s0.lgstatic.com/i/image/M00/94/AD/Ciqc1GAZK9iAVaYlAAna3hqvMhs586.png)
>
> Flutter 技术方案架构图
>
> Flutter 技术方案主要分为三层：Framework、Engine、Embedder。其中**Framework 层由 Dart 语言实现，业务代码直接运行在这一层**。框架的 Framework 层提供了 Material Design 风格的组件，以及[适合 iOS 的 Cupertino 风格组件](https://flutterchina.club/widgets/cupertino/)。以 Cupertino 风格的 button 组件为例，其[源码](https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/cupertino/button.dart)如下：
>
> 复制代码
>
> ```
> // 引入基础及组件
> import 'package:flutter/foundation.dart';
> import 'package:flutter/widgets.dart';
> // 引入相关库
> import 'colors.dart';
> import 'constants.dart';
> import 'theme.dart';
> const EdgeInsets _kButtonPadding = EdgeInsets.all(16.0);
> const EdgeInsets _kBackgroundButtonPadding = EdgeInsets.symmetric(
>   vertical: 14.0,
>   horizontal: 64.0,
> );
> // 一个 Cupertino 风格的按钮组件，继承了 StatefulWidget
> class CupertinoButton extends StatefulWidget {
>   const CupertinoButton({
>     Key? key,
>     required this.child,
>     this.padding,
>     this.color,
>     this.disabledColor = CupertinoColors.quaternarySystemFill,
>     this.minSize = kMinInteractiveDimensionCupertino,
>     this.pressedOpacity = 0.4,
>     this.borderRadius = const BorderRadius.all(Radius.circular(8.0)),
>     required this.onPressed,
>   }) : assert(pressedOpacity == null || (pressedOpacity >= 0.0 && pressedOpacity <= 1.0)),
>        assert(disabledColor != null),
>        _filled = false,
>        super(key: key);
>   const CupertinoButton.filled({
>     Key? key,
>     required this.child,
>     this.padding,
>     this.disabledColor = CupertinoColors.quaternarySystemFill,
>     this.minSize = kMinInteractiveDimensionCupertino,
>     this.pressedOpacity = 0.4,
>     this.borderRadius = const BorderRadius.all(Radius.circular(8.0)),
>     required this.onPressed,
>   }) : assert(pressedOpacity == null || (pressedOpacity >= 0.0 && pressedOpacity <= 1.0)),
>        assert(disabledColor != null),
>        color = null,
>        _filled = true,
>        super(key: key);
>   final Widget child;
>   final EdgeInsetsGeometry? padding;
>   final Color? color;
>   final Color disabledColor;
>   final VoidCallback? onPressed;
>   final double? minSize;
>   final double? pressedOpacity;
>   final BorderRadius? borderRadius;
>   final bool _filled;
>   bool get enabled => onPressed != null;
>   @override
>   _CupertinoButtonState createState() => _CupertinoButtonState();
>   @override
>   void debugFillProperties(DiagnosticPropertiesBuilder properties) {
>     super.debugFillProperties(properties);
>     properties.add(FlagProperty('enabled', value: enabled, ifFalse: 'disabled'));
>   }
> }
> // _CupertinoButtonState 类，继承了 CupertinoButton，同时应用 mixin
> class _CupertinoButtonState extends State<CupertinoButton> with SingleTickerProviderStateMixin {
>   static const Duration kFadeOutDuration = Duration(milliseconds: 10);
>   static const Duration kFadeInDuration = Duration(milliseconds: 100);
>   final Tween<double> _opacityTween = Tween<double>(begin: 1.0);
>   late AnimationController _animationController;
>   late Animation<double> _opacityAnimation;
>   // 初始化状态
>   @override
>   void initState() {
>     super.initState();
>     _animationController = AnimationController(
>       duration: const Duration(milliseconds: 200),
>       value: 0.0,
>       vsync: this,
>     );
>     _opacityAnimation = _animationController
>       .drive(CurveTween(curve: Curves.decelerate))
>       .drive(_opacityTween);
>     _setTween();
>   }
>   // 相关生命周期
>   @override
>   void didUpdateWidget(CupertinoButton old) {
>     super.didUpdateWidget(old);
>     _setTween();
>   }
>   void _setTween() {
>     _opacityTween.end = widget.pressedOpacity ?? 1.0;
>   }
>   @override
>   void dispose() {
>     _animationController.dispose();
>     super.dispose();
>   }
>   bool _buttonHeldDown = false;
>   // 处理 tap down 事件
>   void _handleTapDown(TapDownDetails event) {
>     if (!_buttonHeldDown) {
>       _buttonHeldDown = true;
>       _animate();
>     }
>   }
>   // 处理 tap up 事件
>   void _handleTapUp(TapUpDetails event) {
>     if (_buttonHeldDown) {
>       _buttonHeldDown = false;
>       _animate();
>     }
>   }
>   // 处理 tap cancel 事件
>   void _handleTapCancel() {
>     if (_buttonHeldDown) {
>       _buttonHeldDown = false;
>       _animate();
>     }
>   }
>   // 相关动画处理
>   void _animate() {
>     if (_animationController.isAnimating)
>       return;
>     final bool wasHeldDown = _buttonHeldDown;
>     final TickerFuture ticker = _buttonHeldDown
>         ? _animationController.animateTo(1.0, duration: kFadeOutDuration)
>         : _animationController.animateTo(0.0, duration: kFadeInDuration);
>     ticker.then<void>((void value) {
>       if (mounted && wasHeldDown != _buttonHeldDown)
>         _animate();
>     });
>   }
>   @override
>   Widget build(BuildContext context) {
>     final bool enabled = widget.enabled;
>     final CupertinoThemeData themeData = CupertinoTheme.of(context);
>     final Color primaryColor = themeData.primaryColor;
>     final Color? backgroundColor = widget.color == null
>       ? (widget._filled ? primaryColor : null)
>       : CupertinoDynamicColor.resolve(widget.color, context);
>     final Color? foregroundColor = backgroundColor != null
>       ? themeData.primaryContrastingColor
>       : enabled
>         ? primaryColor
>         : CupertinoDynamicColor.resolve(CupertinoColors.placeholderText, context);
>     final TextStyle textStyle = themeData.textTheme.textStyle.copyWith(color: foregroundColor);
>     return GestureDetector(
>       behavior: HitTestBehavior.opaque,
>       onTapDown: enabled ? _handleTapDown : null,
>       onTapUp: enabled ? _handleTapUp : null,
>       onTapCancel: enabled ? _handleTapCancel : null,
>       onTap: widget.onPressed,
>       child: Semantics(
>         button: true,
>         child: ConstrainedBox(
>           constraints: widget.minSize == null
>             ? const BoxConstraints()
>             : BoxConstraints(
>                 minWidth: widget.minSize!,
>                 minHeight: widget.minSize!,
>               ),
>           child: FadeTransition(
>             opacity: _opacityAnimation,
>             child: DecoratedBox(
>               decoration: BoxDecoration(
>                 borderRadius: widget.borderRadius,
>                 color: backgroundColor != null && !enabled
>                   ? CupertinoDynamicColor.resolve(widget.disabledColor, context)
>                   : backgroundColor,
>               ),
>               child: Padding(
>                 padding: widget.padding ?? (backgroundColor != null
>                   ? _kBackgroundButtonPadding
>                   : _kButtonPadding),
>                 child: Center(
>                   widthFactor: 1.0,
>                   heightFactor: 1.0,
>                   child: DefaultTextStyle(
>                     style: textStyle,
>                     child: IconTheme(
>                       data: IconThemeData(color: foregroundColor),
>                       child: widget.child,
>                     ),
>                   ),
>                 ),
>               ),
>             ),
>           ),
>         ),
>       ),
>     );
>   }
> }
> ```
>
> 通过上面代码，我们可以感知 Dart 语言风格以及设计一个组件的关键点：Flutter 组件分为两种类型，**StatelessWidget 无状态组件和 StatefulWidget 有状态组件**。上面 button 显然是一个有状态组件，它包含了 _CupertinoButtonState 类，并继承 State`<CupertinoButton>`。通常一个有状态组件的声明为：
>
> 复制代码
>
> ```
> class MyCustomStatefulWidget extends StatefulWidget {
>   //---constructor with named // argument: country--- MyCustomStatefulWidget( {Key key, this.country}) : super(key: key);
>   //---used in _DisplayState--- final String country;
>   @override _DisplayState createState() => _DisplayState();
> }
> 
> class _DisplayState extends State<MyCustomStatefulWidget> {
>   @override Widget build(BuildContext context) { 
>     return Center( 
>       //---country defined in StatefulWidget // subclass--- child: Text(widget.country), 
>      ); 
>   }
> }
> ```
>
> Framework 的下一层是 Engine 层，这一层是**Flutter 内部核心**，它主要由 C++ 实现，包含**Skia 图形处理函数库、Dart runtime、Garbage Collection（GC）以及 Text（文本渲染）**。在这一层中，通过内置的 Dart 运行时，Flutter 提供了在 Debug 模式下的 JIT（Just in time）支持，以及在 Release 和 Profile 模式下的 AOT（Ahead of time）编译时编译成原生 ARM 代码的能力。
>
> 最底层为 Embedder 嵌入层，在这一层中，Flutter 的主要工作是 Surface Setup、接入原生插件、设置线程等。也许你并不了解具体底层的知识，这里只需要清楚：Flutter 的 Embedder 层已经很低，**原生平台只需要提供画布，而 Flutter 处理了其余所有逻辑**。正是因为这样，Flutter 有了更好的跨端一致性和稳定性，以及更高性能表现。
>
> 目前来看，Flutter 具备其他跨平台方案所不具备的技术优势，加上 Dart 语言加持，未来前景大好。但作为后入场者，也存在生态小、学习成本高等障碍。作为前端开发者，对于 Flutter 技术的持续观察和深入学习并不矛盾，现在正是学习 Flutter 技术的良好时机。
>
> ### 总结
>
> 大前端概念并不是虚无的。大前端的落地、纵向上依靠 Node.js 技术的发展，横向上依靠对端平台的深钻。上一讲我们介绍了小程序跨端的相关知识，这一讲我们分析了原生平台的跨端方案发展和技术设计。跨端这一技术也许会在未来由一个统一方案进行实现，相关话题也许会告一段落，但是深入该话题后学习到的端知识，将会是我们的宝贵财富。
>
> 本讲内容如下：
>
> ![Drawing 7.png](https://s0.lgstatic.com/i/image/M00/94/8A/Ciqc1GAYznaAGKheAAH7M50bq2I025.png)
>
> 从下一讲开始，我们将进入核心框架原理与代码设计模式的学习，下一讲我们将从设计一个请求库需要考虑的问题出发，带你学习 axios 的设计思想。今天的内容到这里就要结束了，如果有任何疑问欢迎在留言区和我互动，我们下一讲再见！