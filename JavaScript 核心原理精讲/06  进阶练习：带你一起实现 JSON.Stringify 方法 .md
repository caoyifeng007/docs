[JavaScript 核心原理精讲 - 前美团前端技术专家 - 拉勾教育](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=601#/detail/pc?id=6179)



> 我在上一讲为你剖析了闭包这个难点，带你了解了作用域、闭包产生的原因及表现形式。那么这一讲，我们一起来手工实现一个 JSON.stringify 的方法。
>
> 这个方法能够站在全局考察你对 JS 各种数据类型理解的深度，对各种极端的边界情况处理能力，以及 JS 的编码能力。之所以将这篇作为这一模块的进阶，是因为我想把整个数据类型的知识点串起来，让你理解得更加融会贯通，能够更上一层楼。
>
> 在大厂的前端面试过程中，这个题目也经常会被问到。大部分候选人只知道这个方法的作用，而如果让他自己实现一个 JSON.Srtingify 方法的话，大多数人都不一定能写出来，或者即便能写出来一些，但是考虑的问题又不够全面。
>
> 因此你要想夯实自身 JavaScript 的编程基础，通过实践来实现一些 JS API 方法，是非常有必要的，所以这一讲我就来帮你搞懂它。
>
> 那么，到底什么是 JSON.stringify 方法？
>
> ### 方法基本介绍
>
> JSON.stringify 是日常开发中经常用到的 JSON 对象中的一个方法，JSON 对象包含两个方法：一是用于解析成 JSON 对象的 parse()；二是用于将对象转换为 JSON 字符串方法的 stringify()。下面我们分别来看下两个方法的基本使用情况。
>
> #### JSON.parse
>
> JSON.parse 方法用来解析 JSON 字符串，构造由字符串描述的 JavaScript 值或对象。该方法有两个参数：第一个参数是需要解析处理的 JSON 字符串，第二个参数是可选参数提供可选的 reviver 函数，用在返回之前对所得到的对象执行变换操作。
>
> > 该方法的语法为：JSON.parse(text[, reviver])
>
> 下面通过一段代码来看看这个方法以及 reviver 参数的用法，如下所示。
>
> 复制代码
>
> ```
> const json = '{"result":true, "count":2}';
> const obj = JSON.parse(json);
> console.log(obj.count);
> // 2
> console.log(obj.result);
> // true
> /* 带第二个参数的情况 */
> JSON.parse('{"p": 5}', function (k, v) {
>     if(k === '') return v;     // 如果k不是空，
>     return v * 2;              // 就将属性值变为原来的2倍返回
> });                            // { p: 10 }
> ```
>
> 上面的代码说明了，我们可以将一个符合 JSON 格式的字符串转化成对象返回；带第二个参数的情况，可以将待处理的字符串进行一定的操作处理，比如上面这个例子就是将属性值乘以 2 进行返回。
>
> 下面我们来了解一下 JSON.stringify 的基本情况。
>
> #### JSON.stringify
>
> JSON.stringify 方法是将一个 JavaScript 对象或值转换为 JSON 字符串，默认该方法其实有三个参数：第一个参数是必选，后面两个是可选参数非必选。第一个参数传入的是要转换的对象；第二个是一个 replacer 函数，比如指定的 replacer 是数组，则可选择性地仅处理包含数组指定的属性；第三个参数用来控制结果字符串里面的间距，后面两个参数整体用得比较少。
>
> > 该方法的语法为：JSON.stringify(value[, replacer [, space]])
>
> 下面我们通过一段代码来看看后面几个参数的妙用，如下所示。
>
> 复制代码
>
> ```
> JSON.stringify({ x: 1, y: 2 });
> // "{"x":1,"y":2}"
> JSON.stringify({ x: [10, undefined, function(){}, Symbol('')] })
> // "{"x":[10,null,null,null]}"
> /* 第二个参数的例子 */
> function replacer(key, value) {
>   if (typeof value === "string") {
>     return undefined;
>   }
>   return value;
> }
> var foo = {foundation: "Mozilla", model: "box", week: 4, transport: "car", month: 7};
> var jsonString = JSON.stringify(foo, replacer);
> console.log(jsonString);
> // "{"week":4,"month":7}"
> /* 第三个参数的例子 */
> JSON.stringify({ a: 2 }, null, " ");
> /* "{
>  "a": 2
> }"*/
> JSON.stringify({ a: 2 }, null, "");
> // "{"a":2}"
> ```
>
> 从上面的代码中可以看到，增加第二个参数 replacer 带来的变化：通过替换方法把对象中的属性为字符串的过滤掉，在 stringify 之后返回的仅为数字的属性变成字符串之后的结果；当第三个参数传入的是多个空格的时候，则会增加结果字符串里面的间距数量，从最后一段代码中可以看到结果。
>
> 下面我们再看下 JSON.stringify 的内部针对各种数据类型的转换方式。
>
> ### 如何自己手动实现？
>
> 为了让你更好地理解实现的过程，请你回想一下“[01 | 代码基本功测试（上）：JS 的数据类型你了解多少](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=601&sid=20-h5Url-0#/detail/pc?id=6174)”中的基本知识，我们当时讲了那么多种数据类型，如果它们都使用这个方法，返回的结果又会是怎么样的呢？
>
> #### 分析各种数据类型及边界情况
>
> 我们来分析一下都有哪些数据类型传入，传入了之后会有什么返回，通过分析的结果我们之后才能更好地实现编码。大致的分析汇总如下表所示（可参考 MDN 文档）。
>
> ![Lark20210122-160329.png](https://s0.lgstatic.com/i/image/M00/90/40/Ciqc1GAKhuuASWc7AAHWTgdfPTc220.png)
>
> 上面这个表中，基本整理出了各种数据类型通过 JSON.stringify 这个方法之后返回对应的值，但是还有一个特殊情况需要注意：对于包含循环引用的对象（深拷贝那讲中也有提到）执行此方法，会抛出错误。
>
> 那么根据上面梳理的这个表格，我们来一起看下代码怎么编写吧。
>
> #### 代码逻辑实现
>
> 我们先利用 typeof 把基础数据类型和引用数据类型分开，分开之后再根据不同情况来分别处理不同的情况，按照这个逻辑代码实现如下。
>
> 复制代码
>
> ```
> function jsonStringify(data) {
>   let type = typeof data;
> 
>   if(type !== 'object') {
>     let result = data;
>     //data 可能是基础数据类型的情况在这里处理
>     if (Number.isNaN(data) || data === Infinity) {
>        //NaN 和 Infinity 序列化返回 "null"
>        result = "null";
>     } else if (type === 'function' || type === 'undefined' || type === 'symbol') {
>       // 由于 function 序列化返回 undefined，因此和 undefined、symbol 一起处理
>        return undefined;
>     } else if (type === 'string') {
>        result = '"' + data + '"';
>     }
>     return String(result);
>   } else if (type === 'object') {
>      if (data === null) {
>         return "null"  // 第01讲有讲过 typeof null 为'object'的特殊情况
>      } else if (data.toJSON && typeof data.toJSON === 'function') {
>         return jsonStringify(data.toJSON());
>      } else if (data instanceof Array) {
>         let result = [];
>         //如果是数组，那么数组里面的每一项类型又有可能是多样的
>         data.forEach((item, index) => {
>         if (typeof item === 'undefined' || typeof item === 'function' || typeof item === 'symbol') {
>                result[index] = "null";
>            } else {
>                result[index] = jsonStringify(item);
>            }
>          });
>          result = "[" + result + "]";
>          return result.replace(/'/g, '"');
>       } else {
>          // 处理普通对象
>          let result = [];
>          Object.keys(data).forEach((item, index) => {
>             if (typeof item !== 'symbol') {
>               //key 如果是 symbol 对象，忽略
>               if (data[item] !== undefined && typeof data[item] !== 'function' && typeof data[item] !== 'symbol') {
>                 //键值如果是 undefined、function、symbol 为属性值，忽略
>                 result.push('"' + item + '"' + ":" + jsonStringify(data[item]));
>               }
>             }
>          });
>          return ("{" + result + "}").replace(/'/g, '"');
>         }
>     }
> }
> ```
>
> 手工实现一个 JSON.stringify 方法的基本代码如上面所示，有几个问题你还是需要注意一下：
>
> 1. 由于 function 返回 'null'， 并且 typeof function 能直接返回精确的判断，故在整体逻辑处理基础数据类型的时候，会随着 undefined，symbol 直接处理了；
> 2. 由于 01 讲说过 typeof null 的时候返回'object'，故 null 的判断逻辑整体在处理引用数据类型的逻辑里面；
> 3. 关于引用数据类型中的数组，由于数组的每一项的数据类型又有很多的可能性，故在处理数组过程中又将 undefined，symbol，function 作为数组其中一项的情况做了特殊处理；
> 4. 同样在最后处理普通对象的时候，key （键值）也存在和数组一样的问题，故又需要再针对上面这几种情况（undefined，symbol，function）做特殊处理；
> 5. 最后在处理普通对象过程中，对于循环引用的问题暂未做检测，如果是有循环引用的情况，需要抛出 Error；
> 6. 根据官方给出的 JSON.stringify 的第二个以及第三个参数的实现，本段模拟实现的代码并未实现，如果有兴趣你可以自己尝试一下。
>
> 整体来说这段代码还是比较复杂的，如果在面试过程中让你当场手写，其实整体还是需要考虑很多东西的。当然上面的代码根据每个人的思路不同，你也可以写出自己认为更优的代码，比如你也可以尝试直接使用 switch 语句，来分别针对特殊情况进行处理，整体写出来可能看起来会比上面的写法更清晰一些，这些可以根据自己情况而定。
>
> #### 实现效果测试
>
> 上面的这个方法已经实现了，那么用起来会不会有问题呢？我们就用上面的代码，来进行一些用例的检测吧。
>
> 上面自己实现的这个 jsonStringify 方法和真正的 JSON.stringify 想要得到的效果是否一样呢？请看下面的测试结果。
>
> 复制代码
>
> ```
> let nl = null;
> console.log(jsonStringify(nl) === JSON.stringify(nl));
> // true
> let und = undefined;
> console.log(jsonStringify(undefined) === JSON.stringify(undefined));
> // true
> let boo = false;
> console.log(jsonStringify(boo) === JSON.stringify(boo));
> // true
> let nan = NaN;
> console.log(jsonStringify(nan) === JSON.stringify(nan));
> // true
> let inf = Infinity;
> console.log(jsonStringify(Infinity) === JSON.stringify(Infinity));
> // true
> let str = "jack";
> console.log(jsonStringify(str) === JSON.stringify(str));
> // true
> let reg = new RegExp("\w");
> console.log(jsonStringify(reg) === JSON.stringify(reg));
> // true
> let date = new Date();
> console.log(jsonStringify(date) === JSON.stringify(date));
> // true
> let sym = Symbol(1);
> console.log(jsonStringify(sym) === JSON.stringify(sym));
> // true
> let array = [1,2,3];
> console.log(jsonStringify(array) === JSON.stringify(array));
> // true
> let obj = {
>     name: 'jack',
>     age: 18,
>     attr: ['coding', 123],
>     date: new Date(),
>     uni: Symbol(2),
>     sayHi: function() {
>         console.log("hi")
>     },
>     info: {
>         sister: 'lily',
>         age: 16,
>         intro: {
>             money: undefined,
>             job: null
>         }
>     }
> }
> console.log(jsonStringify(obj) === JSON.stringify(obj));
> // true
> ```
>
> 通过上面这些测试的例子可以发现，我们自己实现的 jsonStringify 方法基本和 JSON.stringify 转换之后的结果是一样的，不难看出 jsonStringify 基本满足了预期结果。
>
> 本讲的内容也就先介绍到这里。
>
> ### 总结
>
> 这一讲，我利用原理结合实践的方式，带你实现了一个 JSON.stringify 的方法。从中你可以看到，要想自己实现一个 JSON.stringify 方法整体上来说并不容易，它依赖很多数据类型相关的知识点，而且还需要考虑各种边界情况。
>
> 希望你多加实践，如果在面试中也让你当场实现一个 JSON.stringify 方法，你才能够轻松应对。
>
> 另外，如果把本讲中的题目作为面试题的话，其实是对你的 JS 编码能力的一个很全面的考察，因此对于数据类型的相关知识还是很有必要系统性地学习，尤其是对于 JSON 的这两个方法，不常用的那几个参数你是否有了解？还有引用数据类型中对数组以及普通对象的处理，这部分手写起来会比基础数据类型复杂一些，在一些细节处理上会遇到问题。因此，你要好好理解。
>
> 那么讲到这里，第一个模块的内容就介绍完毕了，涉及数据类型相关的知识就暂时告一段落了，马上我们进入全新的第二个模块深入数组的学习。在后续的课时中，我将带领你深入学习 JS 的数组相关知识。我们下一讲再见~