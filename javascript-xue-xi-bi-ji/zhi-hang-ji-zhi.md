# 执行机制

### 为何 try 里面放 return,finally 还会执行,理解其内部机制

try-catch 是捕捉异常的神奇,不管是调试还是防止软件崩溃,都离不开它。

```
function test(){
    try{
        console.log(1);
    }finally{
        condole.log(2);
    }
}

console.log(test());
<!-- 1  2 -->
//结果：它按顺序执行了
//在 try 中加入 return 语句
function test(){
    try{
        console.log(1);
        return 'try';
    }catch(e){
        //TODO
    }finally{
        console.log(2);
    }
}

console.log(test());
<!-- 1 2 try -->
//在 try和 catch的代码块中,如果碰到 return语句,那么在 return之前,会先执行finally中的内容
```

```
//在 finally中也加入 return语句
function test(){
    try{
        console.log(1);
        return 'try';
    }catch(e){
        //TODO
    }finally{
        console.log(2);
        return 'finally';
    }
}

console.log(test());
<!-- 1 2 finally -->
//按照上一条规则,finally是会先执行的,所以如果 finally里有 return语句,那么就真的return了。
```

#### 结论：

> 1. 不管有没有异常,finally 块中代码都会执行
> 2. 当 try 和 catch 中有 return 时,finally 仍然会执行
> 3. finally 是在 return 后面的表达式运算后执行的(此时并没有返回运算后的值,而是先把要返回的值保存起来,管 finally 中的代码怎么样,返回的值都不会改变,仍然是之前保存的值), **所以函数返回值是在 finally 执行前确定的。**
> 4. finally 中最好不要包含 return,否则程序会提前退出,返回值不是 try 或 catch 中保存的返回值。

> **任何执行 try 或者 catch 中的 return 语句之前,都会先执行 finally 语句(如果 finally 存在的话)**

#### 内部机制

在 try 语句中,在执行 return 语句时,要返回的结果已经准备好了,就在此时,程序转到 finally 执行了。在转去之前,try 中先把要返回的结果存放到不同于 x 的局部变量中去,执行完 finally 之后,再从中取出返回结果。 因此,即使 finally 中对变量 x 进行了改变,但是不会影响返回结果。 他应该使用栈保存返回值。

```
//测试
function test(){
 	var x = 1;
	try{
		x++;
		return x;
    }
	finally{
	    ++x;
    }
}


console.log(test());
<!-- 2 -->
```

### JavaScript 如何实现异步编程,可以详细描述 EventLoop 机制

> 同步编程：即是一种典型的请求——响应模型,当请求调用一个函数或方法后,需等待其响应返回,然后执行后续代码。

一般情况下,同步编程,即是按照代码顺序依次执行,能很好的保证程序的执行,但在某些场景下,比如加载文件内容,请求数据等需要返回大量的内容时,耗费时间较长,此时程序不能继续向下运行,这显然是不友好的。这正需要异步编程大显身手！

> 异步编程：不同于同步编程的请求——响应模式,其是一种事件驱动编程,请求调用函数或方法后,无需立即等待响应,可以继续执行其他任务,而在之前任务响应返回后可以通过状态、通知和回调来通知调用者。

> 通常实现异步方式是多线程,如 C#,即同时开启多个线程,不同操作能并行执行。

> JavaScript 语言试单线程,单线程在程序执行时,所走的程序路径是按照连续顺序排下来的。前面必须处理好,后面才会执行。前面代码报错,后面的程序将不会执行。
>
> JavaScript 单线程异步编程可以实现多任务并发执行。 - 并行：指同一时刻内多任务同时进行 - 并发：指在同一时间段内,多任务同时进行着,但是某一时刻,只有某一任务执行。

#### 并发模型

JavaScript 异步编程使得多个任务可以并发执行,而实现这一功能的基础是 JavaScript 拥有一个基于事件循环的并发模型

> * 堆栈与队列
>   * 堆：内存中某一未被阻止的区域,通常存储对象(引用类型);
>   * 栈：后进先出的顺序存储数据结构,通常存储函数参数和基本类型数值变量(按值访问);
>   * 队列：先进先出顺序存储数据结构

#### 事件循环

JavaScript 引擎负责解析,执行 JavaScript 代码,但它不能单独运行,而需要一个宿主环境,一般为浏览器。调用 Javascript 引擎完成多个 javaScript 代码块的调度,执行,这种机制就称为事件循环。

* JavaScript 执行环境中存在的两个结构需要了解：
  * 消息队列:也叫任务队列,存储待处理消息及对应的回调函数或事件处理程序。
  * 执行栈：也叫执行上下文栈,JavaScript 执行栈,顾名思义,是由执行上下文组成,当函数调用时,创建并插入一个执行上下文,通常称为栈帧,存储着函数参数和局部变量,当该函数执行结束时,弹出该执行栈帧。

#### 事件循环流程

> 1. 宿主环境为 JavaScript 创建线程时,会创建堆(heap)和栈(stack),堆内存储 JavaScript 对象,栈内存储执行上下文；
> 2. 栈内执行上下文的同步任务按序执行,执行完即退栈,而当异步任务执行时,该异步任务进入等待状态（不入栈）,同时通知线程：当触发该事件时（或该异步操作响应返回时）,需向消息队列插入一个事件消息；
> 3. 当事件触发或响应返回时,线程向消息队列插入该事件消息（包含事件及回调）；
> 4. 当栈内同步任务执行完毕后,线程从消息队列取出一个事件消息,其对应异步任务（函数）入栈,执行回调函数,如果未绑定回调,这个消息会被丢弃,执行完任务后退栈；
> 5. 当线程空闲（即执行栈清空）时继续拉取消息队列下一轮消息（next tick,事件循环流转一次称为一次 tick）。

#### setTimeout

在`setTimeout`异步回调函数里当我们输出异步任务注册到执行的时间,发现并不等于我们指定的时间,而且多次执行时间隔也不同！

* 在读取消息队列的消息时,得等同步任务完成,这个是需要耗费时间的
* 消息队列先进先出原则,读取此异步事件消息之前,可能还存在其他消息,执行也需要消耗

所以异步执行时间不精确是必然的。

#### JavaScript 异步实现

关于 JavaScript 的异步实现,以前有：回调函数,发布订阅模式,Promise 三类,而在 ES6 中提出了生成器(Generator)方式实现。

### 宏任务和微任务分别有哪些

JavaScript 中,有两类任务队列：宏任务队列(macro tasks)和微任务队列(micro tasks)。宏任务队列可以有多个,微任务队列只有一个。

微任务和宏任务皆为异步任务,他们都属于一个队列,主要区别在于他们的执行顺序,Event Loop 的走向和取值。

* 宏任务：script(全局任务),setTimeout,setInterval,setImmediate,I/O,UI rendering。
* 微任务：process.nextTick,Promise,Object.observer,MutationObserver。

### 分析一个复杂的异步嵌套逻辑

JavaScript 语言的执行环境是单线程的。

这种执行模式实现简单,执行环境相对单纯。但随着前端业务日渐复杂,事物和请求等日渐增多,这种单线程执行方式在复杂的业务下势必效率低下。

为避免和解决这种问题,JS 语言将任务执行模式分为异步和同步。”同步模式”就是后一个任务等待前一个任务结束,然后再执行,程序的执行顺序与任务的排列顺序是一致的、同步的;”异步模式”则完全不同,每一个任务有一个或多个回调函数,前一个任务结束后,不是执行后一个任务,而是执行回调函数,后一个任务则是不等前一个任务结束就执行,所以程序的执行顺序与任务的排列顺序是不一致的,异步的。

“异步模式”非常重要。在浏览器端,避免浏览器失去响应,最好的例子就是 Ajax 操作。

#### 回调函数

异步编程最基本的方法。

回调函数就是一个参数,将这个函数作为参数传到另一个函数里面,当那个函数执行完之后,再执行传进去的这个函数。这个过程就叫做回调。

在 JavaScript 中,回调函数具体的定义为：函数 A 作为参数(函数引用)传递到另一个函数 B 中,并且这个函数 B 执行函数 A。我们就说函数 A 叫做回调函数。如果没有名称(函数表达式),就叫做匿名回调函数。

```
//定义主函数,回调函数作为参数
function A(callback) {
  callback();
  console.log("我是主函数");
}

//定义回调函数
function B() {
  setTimeout("console.log('我是回调函数')", 3000); //模仿耗时操作
}

//调用主函数,将函数B传进去
A(B);

//输出结果
我是主函数;
我是回调函数;
```

上面的代码中,我们先定义了主函数和回调函数,然后再去调用主函数,将回调函数传进去。

定义主函数的时候,我们让代码先去执行 callback()回调函数,但输出结果却是后输出回调函数的内容。这就说明了主函数不用等待回调函数执行完,可以接着执行自己的代码。所以一般回调函数都用在耗时操作上面。比如 ajax 请求,比如处理文件等。

#### 补充回调函数应用场合和优缺点：

* 资源加载：动态加载 js 文件后执行回调,加载 iframe 后执行回调,ajax 操作回调,图片加载完成执行回调,AJAX 等等。
* DOM 事件及 Node.js 事件基于回调机制(Node.js 回调可能出现多层回调嵌套的问题)。
* setTimeout 的延时时间为 0,这个 hack 经常被用到,setTimeout 调用的函数其实就是一个 callback 的体现。
* 链式调用：链式调用的时候,在赋值器(setter)方法中(或者本身没有返回值的方法中)很容易实现链式调用,而取值器(getter)相对来说不好实现链式调用,因为你需要取值器返回你需要的数据,而不是 this 指针,如果要实现链式方法,可以用回调函数来实现。
* setTimeout、setInterval 的函数调用得到其返回值。由于两个函数都是异步的,即：他们的调用时序和程序的主流程是相对独立的,所以没有办法在主体里面等待他们的返回值,它们被打开的时候程序也不会停下来等待,否则也就失去了 setTimeout 及 setInterval 的意义了,所以用 return 已经没有意义了,只能使用 callback。callback 的意义在于将 Timer 的执行结果通知给代理函数进行及时处理。

回调函数这种方式的优点是比较容易理解,可以绑定多个事件,每个事件可以指定多个回调函数,而且可以”去耦合”,有利于实现模块化。缺点就是整个程序都要变成事件驱动型,运行流程会变得很不清晰。

### 如何在保证页面运行流畅的情况下处理海量数据

> 10w 条记录的数组,一次性渲染到页面,如何处理可以不冻结 ul？

可能在看到这个问题的第一眼,我们会想到用`append`,不过在循环添加过程中,都会修改 DOM 结构,并且由于数据量大,导致循环执行时间长,浏览器的渲染帧率过低。

事实上,包含 10W 个`li`的长列表,用户不会立即看到全部,只会看到少部分。因此,对于大部分的`li`的渲染工作,我们可以延时完成。

我们可以从**减少 DOM 操作次数**和**缩短循环时间**两个方面减少主线程阻塞的时间。

我们知道可以通过`DocumentFragment`的使用,减少 DOM 操作次数,降低回流对性能的影响。

在缩短循环时间方面,我们可以通过**分治**的思想,将`li`分批插入到页面中,并且我们通过`requestAniminationFrame`在页面重绘前插入新节点。

#### 事件绑定

如果我们想监听海量元素,推荐方法是使用 JavaScript 的事件机制,实现事件委托,这样可以显著减少 DOM 事件注册的数量。

#### 解决方案

```
(function() {
  const ulContainer = document.getElementById("list-with-big-data");

  // 防御性编程
  if (!ulContainer) {
    return;
  }

  const total = 100000; // 插入数据的总数
  const batchSize = 4; // 每次批量插入的节点个数,个数越多,界面越卡顿
  const batchCount = total / batchSize; // 批处理的次数
  let batchDone = 0; // 已完成的批处理个数

  function appendItems() {
    // 使用 DocumentFragment 减少 DOM 操作次数,对已有元素不进行回流
    const fragment = document.createDocumentFragment();

    for (let i = 0; i < batchSize; i++) {
      const liItem = document.createElement("li");
      liItem.innerText = batchDone * batchSize + i + 1;
      fragment.appendChild(liItem);
    }

    // 每次批处理只修改 1 次 DOM
    ulContainer.appendChild(fragment);
    batchDone++;
    doAppendBatch();
  }

  function doAppendBatch() {
    if (batchDone < batchCount) {
      // 在重绘之前,分批插入新节点
      window.requestAnimationFrame(appendItems);
    }
  }

  // kickoff
  doAppendBatch();

  // 使用 事件委托 ,利用 JavaScript 的事件机制,实现对海量元素的监听,有效减少事件注册的数量
  ulContainer.addEventListener("click", function(e) {
    const target = e.target;

    if (target.tagName === "LI") {
      alert(target.innerText);
    }
  });
})();
```
