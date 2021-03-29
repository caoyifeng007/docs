[JavaScript 核心原理精讲 - 前美团前端技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=601#/detail/pc?id=6177)



> 我在上一讲介绍了继承的概念，同时你也可以看到，其中综合使用了 new、apply 以及 call 的方法，那么这一讲我们就围绕这几个方法进行更深入的讲解，以便于你清楚这几个核心方法的实现思路，更好地去理解继承的原理。
>
> JavaScript 中的 apply、call 和 bind 方法是前端代码开发中相当重要的概念，并且与 this 的指向密切相关。很多人对它们的理解还比较浅显，如果你想拥有扎实的 JavaScript 编程基础，那么必须要了解这些基础常用的方法。希望通过这一讲的学习，你可以彻底掌握它们。
>
> 为了方便你更好地理解本讲的内容，在课程开始前请你先思考几个问题：
>
> 1. 用什么样的思路可以 new 关键词？
> 2. apply、call、bind 这三个方法之间有什么区别?
> 3. 怎样实现一个 apply 或者 call 的方法？
>
> 带着这几个思考，我们开始本课时的学习吧。
>
> ### 方法的基本介绍
>
> #### new 原理介绍
>
> new 关键词的主要作用就是执行一个构造函数、返回一个实例对象，在 new 的过程中，根据构造函数的情况，来确定是否可以接受参数的传递。下面我们通过一段代码来看一个简单的 new 的例子。
>
> 复制代码
>
> ```
> function Person(){
>    this.name = 'Jack';
> }
> var p = new Person(); 
> console.log(p.name)  // Jack
> ```
>
> 这段代码比较容易理解，从输出结果可以看出，p 是一个通过 person 这个构造函数生成的一个实例对象，这个应该很容易理解。那么 new 在这个生成实例的过程中到底进行了哪些步骤来实现呢？总结下来大致分为以下几个步骤。
>
> 1. 创建一个新对象；
> 2. 将构造函数的作用域赋给新对象（this 指向新对象）；
> 3. 执行构造函数中的代码（为这个新对象添加属性）；
> 4. 返回新对象。
>
> 那么问题来了，如果不用 new 这个关键词，结合上面的代码改造一下，去掉 new，会发生什么样的变化呢？我们再来看下面这段代码。
>
> 复制代码
>
> ```
> function Person(){
>   this.name = 'Jack';
> }
> var p = Person();
> console.log(p) // undefined
> console.log(name) // Jack
> console.log(p.name) // 'name' of undefined
> ```
>
> 从上面的代码中可以看到，我们没有使用 new 这个关键词，返回的结果就是 undefined。其中由于 JavaScript 代码在默认情况下 this 的指向是 window，那么 name 的输出结果就为 Jack，这是一种不存在 new 关键词的情况。
>
> 那么当构造函数中有 return 一个对象的操作，结果又会是什么样子呢？我们再来看一段在上面的基础上改造过的代码。
>
> 复制代码
>
> ```
> function Person(){
>    this.name = 'Jack'; 
>    return {age: 18}
> }
> var p = new Person(); 
> console.log(p)  // {age: 18}
> console.log(p.name) // undefined
> console.log(p.age) // 18
> ```
>
> 通过这段代码又可以看出，当构造函数最后 return 出来的是一个和 this 无关的对象时，new 命令会直接返回这个新对象，而不是通过 new 执行步骤生成的 this 对象。
>
> 但是这里要求构造函数必须是返回一个对象，如果返回的不是对象，那么还是会按照 new 的实现步骤，返回新生成的对象。接下来还是在上面这段代码的基础之上稍微改动一下。
>
> 复制代码
>
> ```
> function Person(){
>    this.name = 'Jack'; 
>    return 'tom';
> }
> var p = new Person(); 
> console.log(p)  // {name: 'Jack'}
> console.log(p.name) // Jack
> ```
>
> 可以看出，当构造函数中 return 的不是一个对象时，那么它还是会根据 new 关键词的执行逻辑，生成一个新的对象（绑定了最新 this），最后返回出来。
>
> 因此我们总结一下：new 关键词执行之后总是会返回一个对象，要么是实例对象，要么是 return 语句指定的对象。
>
> ![刘烨的js.png](https://s0.lgstatic.com/i/image/M00/8E/0F/CgqCHmABa_qAP_2zAAVBMulvP2U718.png)
>
> 好了，new 这个关键词内容基本就讲到这里了，我们再看一下 apply 和 call 的基本原理。
>
> #### apply & call & bind 原理介绍
>
> 先来了解一下这三个方法的基本情况，call、apply 和 bind 是挂在 Function 对象上的三个方法，调用这三个方法的必须是一个函数。
>
> 请看这三个函数的基本语法。
>
> 复制代码
>
> ```
> func.call(thisArg, param1, param2, ...)
> func.apply(thisArg, [param1,param2,...])
> func.bind(thisArg, param1, param2, ...)
> ```
>
> 其中 func 是要调用的函数，thisArg 一般为 this 所指向的对象，后面的 param1、2 为函数 func 的多个参数，如果 func 不需要参数，则后面的 param1、2 可以不写。
>
> 这三个方法共有的、比较明显的作用就是，都可以改变函数 func 的 this 指向。call 和 apply 的区别在于，传参的写法不同：apply 的第 2 个参数为数组； call 则是从第 2 个至第 N 个都是给 func 的传参；而 bind 和这两个（call、apply）又不同，bind 虽然改变了 func 的 this 指向，但不是马上执行，而这两个（call、apply）是在改变了函数的 this 指向之后立马执行。
>
> 这几个方法的区别和原理基本讲清楚了，但是理解起来是不是很抽象呢？那么我举个形象的例子再配合着代码一起看下。
>
> 例如，生活中我不经常做饭，家里没有锅，周末突然想给自己做个饭尝尝。但是家里没有锅，而我又不想出去买，所以就问隔壁邻居借了一个锅来用，这样做了饭，又节省了开销，一举两得。
>
> 对应在程序中：A 对象有个 getName 的方法，B 对象也需要临时使用同样的方法，那么这时候我们是单独为 B 对象扩展一个方法，还是借用一下 A 对象的方法呢？当然是可以借用 A 对象的 getName 方法，既达到了目的，又节省重复定义，节约内存空间。
>
> 为了更好地掌握这部分概念，我们结合一段代码再深入理解一下这几个方法。
>
> 复制代码
>
> ```
> let a = {
>   name: 'jack',
>   getName: function(msg) {
>     return msg + this.name;
>   } 
> }
> let b = {
>   name: 'lily'
> }
> console.log(a.getName('hello~'));  // hello~jack
> console.log(a.getName.call(b, 'hi~'));  // hi~lily
> console.log(a.getName.apply(b, ['hi~']))  // hi~lily
> let name = a.getName.bind(b, 'hello~');
> console.log(name());  // hello~lily
> ```
>
> 从上面的代码执行的结果中可以发现，使用这三种方式都可以达成我们想要的目标，即通过改变 this 的指向，让 b 对象可以直接使用 a 对象中的 getName 方法。从结果中可以看到，最后三个方法输出的都是和 lily 相关的打印结果，满足了我们的预期。
>
> 关于这三个方法的原理相关先介绍到这里，我们再看看这几个方法的使用场景。
>
> ### 方法的应用场景
>
> 下面几种应用场景，你多加体会就可以发现它们的理念都是“借用”方法的思路。我们来看看都有哪些。
>
> #### 判断数据类型
>
> 用 Object.prototype.toString 来判断类型是最合适的，借用它我们几乎可以判断所有类型的数据，我在 [01 讲数据类型的判断](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=601#/detail/pc?id=6174)中有介绍过，我将当时总结的用来判断数据类型的那部分代码粘贴在下面了，你可以回忆一下。
>
> 复制代码
>
> ```
> function getType(obj){
>   let type  = typeof obj;
>   if (type !== "object") {
>     return type;
>   }
>   return Object.prototype.toString.call(obj).replace(/^$/, '$1');
> }
> ```
>
> 结合上面这段代码，以及在前面讲的 call 的方法的 “借用” 思路，那么判断数据类型就是借用了 Object 的原型链上的 toString 方法，最后返回用来判断传入的 obj 的字符串，来确定最后的数据类型，这里就不再多做讲解了。
>
> #### 类数组借用方法
>
> 类数组相关知识我会在第二个模块“深入数组”中详细介绍，这里先简单说一下，类数组因为不是真正的数组，所有没有数组类型上自带的种种方法，所以我们就可以利用一些方法去借用数组的方法，比如借用数组的 push 方法，看下面的一段代码。
>
> 复制代码
>
> ```
> var arrayLike = { 
>   0: 'java',
>   1: 'script',
>   length: 2
> } 
> Array.prototype.push.call(arrayLike, 'jack', 'lily'); 
> console.log(typeof arrayLike); // 'object'
> console.log(arrayLike);
> // {0: "java", 1: "script", 2: "jack", 3: "lily", length: 4}
> ```
>
> 从上面的代码可以看到，arrayLike 是一个对象，模拟数组的一个类数组。从数据类型上看，它是一个对象。从上面的代码中可以看出，用 typeof 来判断输出的是 'object'，它自身是不会有数组的 push 方法的，这里我们就用 call 的方法来借用 Array 原型链上的 push 方法，可以实现一个类数组的 push 方法，给 arrayLike 添加新的元素。
>
> 从上面的控制台可以看出，push 满足了我们想要实现添加元素的诉求。
>
> #### 获取数组的最大 / 最小值
>
> 我们可以用 apply 来实现数组中判断最大 / 最小值，apply 直接传递数组作为调用方法的参数，也可以减少一步展开数组，可以直接使用 Math.max、Math.min 来获取数组的最大值 / 最小值，请看下面这段代码。
>
> 复制代码
>
> ```
> let arr = [13, 6, 10, 11, 16];
> const max = Math.max.apply(Math, arr); 
> const min = Math.min.apply(Math, arr);
>  
> console.log(max);  // 16
> console.log(min);  // 6
> ```
>
> #### 继承
>
> 我们在上一讲中说到了继承，它与 new、call 共同实现了各种各样的继承方式。那么下面我们结合着这一讲的内容再来回顾一下组合继承方式，代码如下。
>
> 复制代码
>
> ```
>   function Parent3 () {
>     this.name = 'parent3';
>     this.play = [1, 2, 3];
>   }
> 
>   Parent3.prototype.getName = function () {
>     return this.name;
>   }
>   function Child3() {
>     Parent3.call(this);
>     this.type = 'child3';
>   }
> 
>   Child3.prototype = new Parent3();
>   Child3.prototype.constructor = Child3;
>   var s3 = new Child3();
>   console.log(s3.getName());  // 'parent3'
> ```
>
> 关于继承的内容在这里就不过多讲解了，另外这些方法类似的应用场景还有很多，关键在于它们借用方法的理念，如果对这部分内容不理解的话，你可以再多看几遍。
>
> ### 如何自己实现这些方法
>
> 在互联网大厂的面试中，手写实现 new、call、apply、bind 一直是比较高频的题目，结合本讲的内容，我们一起来手工实现一下这几个方法。
>
> #### new 的实现
>
> 我们刚才在讲 new 的原理时，介绍了执行 new 的过程。那么来看下在这过程中，new 被调用后大致做了哪几件事情。
>
> 1. 让实例可以访问到私有属性；
> 2. 让实例可以访问构造函数原型（constructor.prototype）所在原型链上的属性；
> 3. 构造函数返回的最后结果是引用数据类型。
>
> 那么请你思考一下，自己实现 new 的代码应该如何写呢？下面我给你一个思路。
>
> 复制代码
>
> ```
> function _new(ctor, ...args) {
>     if(typeof ctor !== 'function') {
>       throw 'ctor must be a function';
>     }
>     let obj = new Object();
>     obj.__proto__ = Object.create(ctor.prototype);
>     let res = ctor.apply(obj,  [...args]);
> 
>     let isObject = typeof res === 'object' && res !== null;
>     let isFunction = typeof res === 'function';
>     return isObject || isFunction ? res : obj;
> };
> ```
>
> 接下来我们再看看 apply 和 call 的实现方法。
>
> #### apply 和 call 的实现
>
> 由于 apply 和 call 基本原理是差不多的，只是参数存在区别，因此我们将这两个的实现方法放在一起讲。
>
> 依然是结合方法“借用”的原理，我们一起来思考一下这两个方法如何实现，请看下面实现的代码。
>
> 复制代码
>
> ```
> Function.prototype.call = function (context, ...args) {
>   var context = context || window;
>   context.fn = this;
>   var result = eval('context.fn(...args)');
>   delete context.fn
>   return result;
> }
> Function.prototype.apply = function (context, args) {
>   let context = context || window;
>   context.fn = this;
>   let result = eval('context.fn(...args)');
>   delete context.fn
>   return result;
> }
> ```
>
> 从上面的代码可以看出，实现 call 和 apply 的关键就在 eval 这行代码。其中显示了用 context 这个临时变量来指定上下文，然后还是通过执行 eval 来执行 context.fn 这个函数，最后返回 result。
>
> 要注意这两个方法和 bind 的区别就在于，这两个方法是直接返回执行结果，而 bind 方法是返回一个函数，因此这里直接用 eval 执行得到结果。
>
> 如果让你去执行，这个区别一定要多加注意。紧接着我们就来看下 bind 的实现。
>
> #### bind 的实现
>
> 结合上面两个方法的实现，bind 的实现思路基本和 apply 一样，但是在最后实现返回结果这里，bind 和 apply 有着比较大的差异，bind 不需要直接执行，因此不再需要用 eval ，而是需要通过返回一个函数的方式将结果返回，之后再通过执行这个结果，得到想要的执行效果。
>
> 那么，结合这个思路，我们看下 bind 这个方法的底层逻辑实现的代码是什么样的，如下所示。
>
> 复制代码
>
> ```
> Function.prototype.bind = function (context, ...args) {
>     if (typeof this !== "function") {
>       throw new Error("this must be a function");
>     }
>     var self = this;
>     var fbound = function () {
>         self.apply(this instanceof self ? this : context, args.concat(Array.prototype.slice.call(arguments)));
>     }
>     if(this.prototype) {
>       fbound.prototype = Object.create(this.prototype);
>     }
>     return fbound;
> }
> ```
>
> 从上面的代码中可以看到，实现 bind 的核心在于返回的时候需要返回一个函数，故这里的 fbound 需要返回，但是在返回的过程中原型链对象上的属性不能丢失。因此这里需要用Object.create 方法，将 this.prototype 上面的属性挂到 fbound 的原型上面，最后再返回 fbound。这样调用 bind 方法接收到函数的对象，再通过执行接收的函数，即可得到想要的结果。
>
> 那么讲到这里，你是不是已经清楚了 new、apply、call、bind 这些方法是如何实现的呢？如果还是一知半解，我建议你多动手实践几次。
>
> ### 总结
>
> 这一讲的内容就介绍完了。我们通过原理以及对底层逻辑的剖析，介绍了日常开发中经常用的 new、apply、call、bind 这几种方法，最后带你一起动手进行了实践。
>
> 综上，我们可以看到这几个方法是有区别和联系的，通过下面的表格我们再来梳理一下这些方法的异同点，希望你可以更好地理解。
>
> ![图片5.png](https://s0.lgstatic.com/i/image/M00/8E/04/Ciqc1GABa-2AO2DlAAD5wuBLNn8120.png)
>
> 在日常的前端开发工作中，大家往往会忽视对这些方法的系统性学习，其实这些方法在高级 JavaScript 编程中经常出现，尤其是你去看一些比较好的开源项目，经常会通过“借用”的方式去复用已有的方法，来节约内存、优化代码。
>
> 而且这些方法的底层逻辑的实现，在互联网大厂的前端面试中出现的频率也比较高，每个实现的方法细节也比较零散，很多开发者很难有一个系统的、整体的学习，造成了在面试过程中遇到此类手写底层 API 等问题时，容易临场发怵。
>
> 因此我希望通过这一讲的学习，你能很好地掌握这部分内容，以便在面试中遇到这类题目的时候能够轻松应对。
>
> 在后续的课时中，我将继续带领你深入挖掘闭包的原理和底层知识。同时希望你多动手练习以熟练上面的代码，也欢迎你在下方留言讨论自己在学习过程中遇到的困惑，以及学习感悟等，让我们共同进步。
>
> 我们下一课时再见~