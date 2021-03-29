[JavaScript 核心原理精讲 - 前美团前端技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=601#/detail/pc?id=6189)



> 前两讲我们探讨了 JS 异步编程中 Generator 和 async/await 的相关内容，那么这一讲我们就进入 Node.js 的 events 模块以及 EventEmitter 的学习，并且我将带你在浏览器端实现一遍它的底层逻辑。
>
> 之所以要特地讲解这部分知识，是因为虽然严格意义上来说，events 模块属于 Node.js 服务端的知识，但是由于大多数 Node.js 核心 API 构建用的是异步事件驱动架构，因此这里单独加了一讲来带你学习这部分内容。我希望通过这一讲的学习，你能够自己实现一个EventEmitter。
>
> 那么，在课程开始前请你先思考几个问题：
>
> 1. EventEmitter 采用什么样的设计模式？
> 2. EventEmitter 常用的API 是怎样实现的？
>
> ### Events 基本介绍
>
> 你或多或少会了解一些 Node.js 相关的知识，应该知道Node.js 里面有很多模块，其中 events 就是比较重要的一个模块。
>
> Node.js的events 模块对外提供了一个 EventEmitter 对象，用于对 Node.js 中的事件进行统一管理。因为 Node.js 采用了事件驱动机制，而 EventEmitter 就是 Node.js 实现事件驱动的基础。在 EventEmitter 的基础上，Node.js 中几乎所有的模块都继承了这个类，以实现异步事件驱动架构。
>
> 为了让你对此有一个大概的了解，我们先来看下 EventEmitter的简单使用情况，代码如下。
>
> 复制代码
>
> ```
> var events = require('events');
> var eventEmitter = new events.EventEmitter();
> eventEmitter.on('say',function(name){
>     console.log('Hello',name);
> })
> eventEmitter.emit('say','Jonh');
> ```
>
> 以上代码中，新定义的eventEmitter 是接收 events.EventEmitter 模块 new 之后返回的一个实例，eventEmitter 的 emit 方法，发出 say 事件，通过 eventEmitter 的 on 方法监听，从而执行相应的函数。
>
> 这就是 Node.js的events 模块中 EventEmitter 的基本用法，下面来说说 EventEmitter 的各种方法以及功能的介绍。
>
> ### 常用的 EventEmitter 模块的 API
>
> 除了上面的那段代码中已经使用的 on 和emit 这两个 API，EventEmitter还提供了其他的 API 方法，我通过一个表格简单整理了一下对应的方法和功能总结。
>
> ![图片1.png](https://s0.lgstatic.com/i/image6/M01/0E/14/Cgp9HWA8LMiAEVlGAAJEpecSYyo071.png)
>
> 除此之外，还有两个特殊的事件，不需要额外手动添加，下表所示的就是 Node.js 的 EventEmitter 模块自带的特殊事件。
>
> ![图片2.png](https://s0.lgstatic.com/i/image6/M01/0E/14/Cgp9HWA8LLaAdnhdAADOmTg9zw8428.png)
>
> 从上面的表格可以看出，Node.js的EventEmitter 模块看起来方法很多且复杂，但通过仔细学习，其实其使用和实现并不困难。下面我就来挑几个比较重要 API 方法为你进行讲解。
>
> #### addListener 和 removeListener、on 和 off 方法对比
>
> addListener 方法的作用是为指定事件添加一个监听器，其实和 on 方法实现的功能是一样的，on 其实就是 addListener 方法的一个别名。二者实现的作用是一样的，同时 removeListener 方法的作用是为移除某个事件的监听器，同样 off 也是 removeListener 的别名。
>
> 下面我们来看看addListener 和removeListener 的用法，请看下面一段示例代码。
>
> 复制代码
>
> ```
> var events = require('events');
> var emitter = new events.EventEmitter();
> function hello1(name){
>   console.log("hello 1",name);
> }
> function hello2(name){
>   console.log("hello 2",name);
> }
> emitter.addListener('say',hello1);
> emitter.addListener('say',hello2);
> emitter.emit('say','John');
> //输出hello 1 John 
> //输出hello 2 John
> emitter.removeListener('say',hello1);
> emitter.emit('say','John');
> //相应的，监听say事件的hello1事件被移除
> //只输出hello 2 John
> ```
>
> 结合代码和注释来理解我上面的描述，是不是对于 addListener 和 removeListener、on 和 off 这两组方法的对比就一目了然了呢？下面我再来说说 removeListener 和 removeAllListeners 的对比。
>
> #### removeListener 和 removeAllListeners
>
> removeListener 方法是指移除一个指定事件的某一个监听器，而 removeAllListeners 指的是移除某一个指定事件的全部监听器。
>
> 这里举一个removeAllListeners 的代码例子，请看。
>
> 复制代码
>
> ```
> var events = require('events');
> var emitter = new events.EventEmitter();
> function hello1(name){
>   console.log("hello 1",name);
> }
> function hello2(name){
>   console.log("hello 2",name);
> }
> emitter.addListener('say',hello1);
> emitter.addListener('say',hello2);
> emitter.removeAllListeners('say');
> emitter.emit('say','John');
> //removeAllListeners 移除了所有关于 say 事件的监听
> //因此没有任何输出
> ```
>
> 同样的，这两者的对比，通过代码和注释也比较好理解。
>
> #### on 和 once 方法区别
>
> on 和 once 的区别是：on 的方法对于某一指定事件添加的监听器可以持续不断地监听相应的事件；而 once 方法添加的监听器，监听一次后，就会被消除。
>
> 看一下这段代码，你就会明白了。
>
> 复制代码
>
> ```
> var events = require('events');
> var emitter = new events.EventEmitter();
> function hello(name){
>   console.log("hello",name);
> }
> emitter.on('say',hello);
> emitter.emit('say','John');
> emitter.emit('say','Lily');
> emitter.emit('say','Lucy');
> //会输出 hello John、hello Lily、hello Lucy，之后还要加也可以继续触发
> emitter.once('see',hello);
> emitter.emit('see','Tom');
> //只会输出一次 hello Tom
> ```
>
> 也就是说，on 方法监听的事件，可以持续不断地被触发，而 once 方法只会触发一次。
>
> 讲到这里，你已经基本熟悉了Node.js 下的 EventEmitter 的基本情况。那么如果在浏览器端，我们想实现一个 EventEmitter 方法，应该用什么样的思路呢？请你再往下看。
>
> ### 带你实现一个 EventEmitter
>
> 从上面的讲解中可以看到，EventEmitter 是在Node.js 中 events 模块里封装的，那么在浏览器端实现一个这样的 EventEmitter 是否可以呢？其实自己封装一个能在浏览器中跑的EventEmitter，并应用在你的业务代码中还是能带来不少方便的，它可以帮你实现自定义事件的订阅和发布，从而提升业务开发的便利性。
>
> 那么结合上面介绍的内容，我们一起来实现一个基础版本的EventEmitter，包含基础的on、 of、emit、once、allof 这几个方法。
>
> 首先，请你看一下 EventEmitter的初始化代码。
>
> 复制代码
>
> ```
> function EventEmitter() {
>     this.__events = {}
> }
> EventEmitter.VERSION = '1.0.0';
> ```
>
> 从上面的代码中可以看到，我们先初始化了一个内部的__events 的对象，用来存放自定义事件，以及自定义事件的回调函数。
>
> 其次，我们来看看如何实现 EventEmitter的 on 的方法，请看下面的代码。
>
> 复制代码
>
> ```
> EventEmitter.prototype.on = function(eventName, listener){
> 	  if (!eventName || !listener) return;
>       // 判断回调的 listener 是否为函数
> 	  if (!isValidListener(listener)) {
> 	       throw new TypeError('listener must be a function');
> 	  }
> 	   var events = this.__events;
> 	   var listeners = events[eventName] = events[eventName] || [];
> 	   var listenerIsWrapped = typeof listener === 'object';
>        // 不重复添加事件，判断是否有一样的
>        if (indexOf(listeners, listener) === -1) {
>            listeners.push(listenerIsWrapped ? listener : {
>                listener: listener,
>                once: false
>            });
>        }
> 	   return this;
> };
> // 判断是否是合法的 listener
>  function isValidListener(listener) {
>      if (typeof listener === 'function') {
>          return true;
>      } else if (listener && typeof listener === 'object') {
>          return isValidListener(listener.listener);
>      } else {
>          return false;
>      }
> }
> // 顾名思义，判断新增自定义事件是否存在
> function indexOf(array, item) {
>      var result = -1
>      item = typeof item === 'object' ? item.listener : item;
>      for (var i = 0, len = array.length; i < len; i++) {
>          if (array[i].listener === item) {
>              result = i;
>              break;
>          }
>      }
>      return result;
> }
> ```
>
> 从上面的代码中可以看出，on 方法的核心思路就是，当调用订阅一个自定义事件的时候，只要该事件通过校验合法之后，就把该自定义事件 push 到 this.__events 这个对象中存储，等需要出发的时候，则直接从通过获取 __events 中对应事件的 listener 回调函数，而后直接执行该回调方法就能实现想要的效果。
>
> 然后，我们再看看 emit 方法是怎么实现触发效果的，请看下面的代码实现逻辑。
>
> 复制代码
>
> ```
> EventEmitter.prototype.emit = function(eventName, args) {
>      // 直接通过内部对象获取对应自定义事件的回调函数
>      var listeners = this.__events[eventName];
>      if (!listeners) return;
>      // 需要考虑多个 listener 的情况
>      for (var i = 0; i < listeners.length; i++) {
>          var listener = listeners[i];
>          if (listener) {
>              listener.listener.apply(this, args || []);
>              // 给 listener 中 once 为 true 的进行特殊处理
>              if (listener.once) {
>                  this.off(eventName, listener.listener)
>              }
>          }
>      }
>      return this;
> };
> 
> EventEmitter.prototype.off = function(eventName, listener) {
>      var listeners = this.__events[eventName];
>      if (!listeners) return;
>      var index;
>      for (var i = 0, len = listeners.length; i < len; i++) {
> 	    if (listeners[i] && listeners[i].listener === listener) {
>            index = i;
>            break;
>         }
>     }
>     // off 的关键
>     if (typeof index !== 'undefined') {
>          listeners.splice(index, 1, null)
>     }
>     return this;
> };
> ```
>
> 从上面的代码中可以看出 emit 的处理方式，其实就是拿到对应自定义事件进行 apply 执行，在执行过程中对于一开始 once 方法绑定的自定义事件进行特殊的处理，当once 为 true的时候，再触发 off 方法对该自定义事件进行解绑，从而实现自定义事件一次执行的效果。
>
> 最后，我们再看下 once 方法和 alloff的实现。
>
> 复制代码
>
> ```
> EventEmitter.prototype.once = function(eventName, listener）{
>     // 直接调用 on 方法，once 参数传入 true，待执行之后进行 once 处理
>      return this.on(eventName, {
>          listener: listener,
>          once: true
>      })
>  };
> EventEmitter.prototype.allOff = function(eventName) {
>      // 如果该 eventName 存在，则将其对应的 listeners 的数组直接清空
>      if (eventName && this.__events[eventName]) {
>          this.__events[eventName] = []
>      } else {
>          this.__events = {}
>      }
> };
> ```
>
> 从上面的代码中可以看到，once 方法的本质还是调用 on 方法，只不过传入的参数区分和非一次执行的情况。当再次触发 emit 方法的时候，once 绑定的执行一次之后再进行解绑。
>
> 这样，alloff 方法也很好理解了，其实就是对内部的__events 对象进行清空，清空之后如果再次触发自定义事件，也就无法触发回调函数了。
>
> 到这里，浏览器端的EventEmitter的基础版本就基本实现了，如果你对其他方法有兴趣，也可以尝试在上面基础版本的基础上进行扩展和添加。
>
> ### 总结
>
> 这一讲，我把 EventEmitter 相关知识点带你梳理了一遍，并且最后也带你实现了一个浏览器端的EventEmitter。
>
> 现在，你可以回过头思考一下我在开篇提到的问题：EventEmitter 采用的是什么样的设计模式？其实通过上面的学习你不难发现，EventEmitter 采用的正是发布-订阅模式。
>
> 另外，观察者模式和发布-订阅模式有些类似的地方，但是在细节方面还是有一些区别的，你要注意别把这两个模式搞混了。发布-订阅模式其实是观察者模式的一种变形，区别在于：**发布-订阅模式在观察者模式的基础上，在目标和观察者之间增加了一个调度中心**。
>
> 通过这一学习，你应该基本能实现一个 EventEmitter 了。单就浏览器端使用场景来说，其实也有运用同样的思路解决问题的工具，在 Vue 框架中不同组件之间的通讯里，有一种解决方案叫 EventBus。和 EventEmitter的思路类似，它的基本用途是将 EventBus 作为组件传递数据的桥梁，所有组件共用相同的事件中心，可以向该中心注册发送事件或接收事件，所有组件都可以收到通知，使用起来非常便利，其核心其实就是发布-订阅模式的落地实现。
>
> 好了，这一讲就先探讨到这，下一讲我们将继续进阶，带你实现一个符合规范的Promise。同时希望你能自己手动实现一遍代码，也欢迎你留言提出自己在学习过程中遇到的困惑，以及学习感悟等，让我们共同进步。