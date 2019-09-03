---
title: "不懂不写系列：JavaScript的异步机制，EventLoop，macrotask，microtask，nextTick终结篇"
date: 2019-08-22T20:19:23+08:00
draft: true
---

经常看到下面这样的一个例子：

```
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

如果你能立马得出答案，感觉就是小菜一碟，那么你可以跳过本文了。。。😄

如果你模棱两可的猜测执行过程，那你肯定没有理解清楚JavaScript的异步机制，本文就是为你准备的。我们经常会听到说JavaScript是单线程无阻塞的，单线程怎么能做到无阻塞呢？有的人可能会说因为JavaScript支持异步回调，但是为什么JavaScript支持异步呢？其实理解清楚异步背后的事件循环机制，也就是Event Loop，像上面这些问题和文章开头看起来变态的例子就很容易理解了，并且可以弄懂一次就永不出错。

## 下面开始终结之旅
---
先看下面一张图，花几秒钟时间先自己想象Event Loop的执行过程...

![event-loop.jpg](https://cdn.steemitimages.com/DQmUyZ7SruH55V1TunUpaCLXqG4iYaat7WEoSdBpoLUYa5o/event-loop.jpg)

几秒钟过后...... 


好了，可以直观的看出图中包含三个主要的概念：

`Event Loop`，`macrotask`，`microtask`

因为事关JavaScript的执行过程，所以还会牵扯出一些其它的基础概念：

* 进程和线程
* 栈和队列
* 函数调用栈/执行栈

概念很多，我们各个击破...
# 进程和线程
浏览器是是多进程的，每打开一个新的tab或者新窗口都相当于创建了一个新的进程，每个进程都有占有独立的cpu和内存资源，多个进程间互不影响，所以一个页面挂了不会影响其它页面。每个进程包含多个线程，比如GUI渲染线程，JS引擎线程，定时器线程，HTTP异步请求线程,事件触发线程等。经常说的JavaScript是单线程的一般是指js内核引擎线程，比如v8引擎。**进程是系统分配资源的最小单位,线程是CPU调度的最小单位。**另外除了进程和线程外，还有一种微线程，一般被成为协程,一个线程可以包含多个协程，协程是用户自己调度的，没有上下文切换消耗,使用协程比较有代表性的就是Golang。
# Stack（栈）和 Queue（队列）
Stack和Queue都是两种基本的数据结构，而常用的数据结构有八种，还有六种数据结构分别是：数组（Array）、散列表（Hash）、树（Tree）、链表（Linked List）、堆（Heap）、图（Graph）。

**`Stack：`**Stack是一种FILO（First In, First Out）的数据结构，也就是`先进后出`。eg：就像羽毛球筒一样，最先放进去最后才能拿出来

**`Queue：`**队列是一种 FIFO (First In, First Out) 的数据结构，它的特点就是`先进先出`。eg：去食堂排队吃饭，去的早的排前面的先吃

# Call Stack(函数调用栈)
调用栈，也叫执行栈，既然是栈，也是`先进后出`的，调用栈用于存储代码在执行期间创建的所有`执行上下文`。首次运行JS代码时，会创建一个全局执行上下文并Push到当前的调用栈中。每当发生函数调用，引擎都会为该函数创建一个新的函数执行上下文并Push到当前调用栈的栈顶。当栈顶函数运行完成后，其对应的函数执行上下文将会从调用栈中Pop出，上下文控制权将移到当前执行栈的下一个执行上下文。（关于执行上下文下次单独拎出来写）

看下面一个例子：

```
function bar() {
	console.log('bar');
}
function foo() {
	console.log('foo');
	bar();
}
foo();
```
先用文字描述下调用栈执行的过程：

1. 调用foo，`foo`入栈，目前栈里面有 [foo]
2. `console.log('foo')`入栈，目前栈里面有 [console.log('foo'),foo]
3. 执行`console.log('foo')`并出栈，目前栈里面有 [foo]
4. 调用bar，`bar`入栈，目前栈里面有 [bar,foo]
5. 执行bar，`console.log('bar')`入栈，目前栈里面有 [console.log('bar'),bar,foo]
6. 执行`console.log('bar')`并出栈，目前栈里面有 [bar,foo]
5. bar执行完毕被弹出，目前栈里面有 [foo]
6. foo执行完毕被弹出，目前栈为空 []

下面是copy过来的图：
![](https://cdn.steemitimages.com/DQmStyEA5rYjaWeGLRbpxuMAs6uY21nLS2n4xwbb2fEsKAY/image.png)


回顾完上面的基础概念后，下面开始本文的重要概念。
# [Event Loop是什么？](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)

#### 先看WHATAG的定义：

>To coordinate events, user interaction, scripts, rendering, networking, and so forth, [user agents](https://tc39.es/ecma262/#sec-agents) must use event loops as described in this section. Each agent has an associated event loop.
>An event loop has one or more task queues. A task queue is a set of tasks.

翻译过来就是，为了协调事件、用户交互、脚本、渲染、网络等，`用户代理`必须使用这一小节描述的`事件循环`。每个`代理`都有一个关联的事件循环。一个事件循环具有一个或多个`任务队列`。任务队列是一组任务。关于任务队列后面会详细说。其中[user agents](https://tc39.es/ecma262/#sec-agents)在ecma262中定义，感兴趣的可以点击上面链接查看详情。这里可以简单理解为一个包含执行上下文堆栈，一组作业队列，执行线程等这些东西的概念。

### 再看MDN中“事件循环“的描述：

之所以称之为事件循环，是因为它经常按照类似如下的方式来被实现：

 ```
  while (queue.waitForMessage()) {
    queue.processNextMessage();
  }
  
```

并且还有"`执行至完成`"的特点：
>每一个消息完整地执行后，其它消息才会被执行。这为程序的分析提供了一些优秀的特性，包括：一个函数执行时，它永远不会被抢占，并且在其他代码运行之前完全运行（且可以修改此函数操作的数据）。这与C语言不同，例如，如果函数在线程中运行，它可能在任何位置被终止，然后在另一个线程中运行其他代码。这个模型的一个缺点在于当一个消息需要太长时间才能处理完毕时，Web应用就无法处理用户的交互，例如点击或滚动。浏览器用“程序需要过长时间运行”的对话框来缓解这个问题。一个很好的做法是缩短消息处理，并在可能的情况下将一个消息裁剪成多个消息。

![](https://cdn.steemitimages.com/DQmZi7BW4ughjDezLyNmLEEXDXybUG63Fc2vcN9gw4ThHqs/image.png)

综合各种规范和来自“民间”的共识，我得出的结论是这样的：
#### `事件循环就是不断读取和执行任务队列中任务的过程。任务队列中的任务可能包含同步任务和异步任务，并且一个任务执行的过程是不能被中断的。`

有没有发现，说了半天其实真正的主角是`任务队列`, 别急😂，说好的各个击破。。。

# [任务队列](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue)

>Per its source field, each task is defined as coming from a specific task source. For each event loop, every task source must be associated with a specific task queue.

>Each event loop has a currently running task, which is either a task or null. Initially, this is null. It is used to handle reentrancy.

>Each event loop has a microtask queue, which is a queue of microtasks, initially empty. A microtask is a colloquial way of referring to a task that was created via the queue a microtask algorithm.

>Each event loop has a performing a microtask checkpoint boolean, which is initially false. It is used to prevent reentrant invocation of the perform a microtask checkpoint algorithm.

>Each event loop has associated values event loop begin and event loop end, which are initially unset.

规范中写得很详细，这里只引用了部分重要的概念。总结如下：

* 每个任务来自特定任务源，对于每个事件循环，每个任务源都必须与特定任务队列相关联。

* 每个事件循环有一个微任务队列，初始值为空。微任务属于任务的一种。

* 每个事件循环都有一个微任务检查点布尔值，最初为false。它用于防止执行微任务检查点算法的重入调用。

* 每个事件循环都具有事件循环开始和事件循环结束的关联值，这些最初未设置。


#### 另外规范中提到需要注意的地方：

⚠️ 任务队列是Sets(集合)，不是队列！！因为事件循环处理模型的第一步是从所选队列中获取第一个可运行任务，而不是使第一个任务出列（dequeue）。Sets在规范中的定义是没有相同item的有序list。这里顺便提一下，[ECMA T39](https://tc39.es/ecma262/#sec-jobs-and-job-queues)规范中把任务叫做job，任务队列叫做 Job Queues。并且提到 "A Job Queue is a FIFO queue of PendingJob records"。也就是说ECMA把任务队列当成先进先出的`Queue`数据结构，WHATWG强调不是`Queue`，很多文章会直接把任务队列直接当作`队列`，这里需要注意下。

⚠️ 任务源主要作用是用于区分逻辑上不同类型的任务

# [任务源 Generic task sources](https://html.spec.whatwg.org/multipage/webappapis.html#generic-task-sources)

* **DOM操作任务源：**
此任务源被用来相应dom操作，例如一个元素以非阻塞的方式插入文档。

* **用户交互任务源：**
此任务源用于对用户交互作出反应，例如键盘或鼠标输入。响应用户操作的事件（例如click）必须使用task队列。

* **网络任务源：**
网络任务源被用来响应网络活动。

* **history traversal任务源：**
当调用history.back()等类似的api时，将任务插进task队列。

task任务源种类非常多，比如ajax的onload，鼠标click事件，基本上我们经常绑定的各种dom事件都是task任务源，还有数据库操作（IndexedDB ），需要注意的是setTimeout、setInterval、setImmediate也是task任务源。总结来说task任务源有这几种：

* setTimeout
* setInterval
* setImmediate
* I/O
* UI rendering

# 任务分类

>所有任务可以分成两种，一种是同步任务（synchronous），另一种是异步任务（asynchronous）。同步任务指的是，在`主线程`上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务；异步任务指的是，不进入主线程、而进入`"任务队列"`（task queue）的任务，只有"任务队列"通知主线程，某个异步任务可以执行了，该任务才会进入主线程执行。

上面一段是来自阮一峰博客，虽然写于2014年，但是这么归类现在看也没错，但是需要修订一下。😄
#### `任务一般可以分成两种，一种是宏任务（macrotask），另一种是微任务（microtask）。宏任务包含同步任务和异步任务，微任务都是异步任务。`
在WHATWG的规范中并没有找到关于macrotask的定义，但是一般大家的共识是把task就当作macrotask，与之相对的是microtask。

⚠️ Node环境和ES6之后才出现微任务的概念

# macrotask
script主代码，事件回调，XHR回调，定时器（setTimeout/setInterval/setImmediate），IO操作，UI render
# microtask
promise回调，MutationObserver，process.nextTick，Object.observe

# 任务执行顺序
基本需要的概念都有了，现在开始说最本文最重要的部分，macrotask和microtask执行顺序

1. script主代码被推入`执行栈`，执行其中的同步代码，遇到异步代码会依次将任务分类放入macrotask队列和microtask队列

2. 检测macrotask队列是否有宏任务待执行，如果有则取出第一个执行，注意一次循环只能执行一个宏任务⚠️

3. 检测microtask队列是否有微任务待执行，如果有则取出第一个执行，然后再检测，如果还有再取出再执行，直到microtask队列为空

4. 执行渲染更新UI，这里注意渲染ui不是必须的，和浏览器策略又关，不同浏览器可能会有不同表现

5. 重复上述步骤，直到两个任务队列都为空


上面是基本任务执行过程，更详细的过程中可能还会包含Web worker，requestAnimationFrame和requestIdleCallback。很多文章把requestAnimationFrame归类为macrotask，但是我并没有找到出处，requestAnimationFrame属于异步任务但是不在任务队列中，我更偏向于认为既不属于macrotask也不属于microtask，有知道出处的朋友欢迎来交流。


## 再来看开篇的问题：
```
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

下面是copy过来的一张图，执行过程非常清晰：

![browser-deom1-excute-animate.gif](https://cdn.steemitimages.com/DQmWmZxQbUkx4w5u7jiydS5yGCxpYmf79Aj4JMcghnu7nHN/browser-deom1-excute-animate.gif)



<!-- 
JavaScript引擎遇到一个异步事件后并不会一直等待其返回结果，而是会将这个事件挂起，继续执行执行栈中的其他任务。当一个异步事件返回结果后，js会将这个事件加入与当前执行栈不同的另一个队列，我们称之为事件队列。被放入事件队列不会立刻执行其回调，而是等待当前执行栈中的所有任务都执行完毕， 主线程处于闲置状态时，主线程会去查找事件队列是否有任务。如果有，那么主线程会从中取出排在第一位的事件，并把这个事件对应的回调放入执行栈中，然后执行其中的同步代码...，如此反复，这样就形成了一个无限的循环。这就是这个过程被称为“事件循环（Event Loop）”的原因。
 -->
<!-- `也就是说JavaScript引擎通过事件循环来处理异步任务，遇到异步任务并不会直接放到主线程的执行栈来执行，而是交给其它线程来处理，比如HTTP异步请求线程,定时器线程`
`事件循环只能算JS实现异步的一种方式，其它编程语言还有很多不同的实现方式`  
 -->

# 用JS实现的Event Loop过程

```
class JsEngine {
    ...
    // 与event-loop中的初始化对应
    constructor(tasks) {
        this.jsStack = tasks;
        this.runScript(this.runScriptHandler);
    }
    runScript(task) {
    	this.macroTaskQueue.push(task);
    }
	runScriptHandler = () => {
        let curTask = this.jsStack.shift();
        while (curTask) {
          	this.runTask(curTask);
          	curTask = this.jsStack.shift();
        }
    }
    runMacroTask() {
        const { microTaskQueue, macroTaskQueue } = this;
		// 根据上述规律，定义macroTaskQueue与microTaskQueue执行的先后顺序
        macroTaskQueue.forEach(macrotask => {
        	macrotask();
          	if (microTaskQueue.length) {
            	let curMicroTask = microTaskQueue.pop();
            	while (curMicroTask) {
              		this.runTask(microTaskQueue);
             		curMicroTask = microTaskQueue.pop();
            	}
        	}
        });
    }
	// 运行task
    runTask(task) {
    	new Function(task)();
    }
}

```

# 最后来个小菜🍖

```
console.log('Start')

setTimeout(() => console.log('Timeout 1'), 0)
setTimeout(() => console.log('Timeout 2'), 0)

Promise.resolve().then(() => {
  for(let i=0; i<100000; i++) {}
  console.log('Promise 1')
})
Promise.resolve().then(() => console.log('Promise 2'))

console.log('End');



```

# 最最后的变态甜点🍮

```
let button = document.querySelector('#button');

button.addEventListener('click', function CB1() {
  console.log('Listener 1');

  setTimeout(() => console.log('Timeout 1'))

  Promise.resolve().then(() => console.log('Promise 1'))
});

button.addEventListener('click', function CB1() {
  console.log('Listener 2');

  setTimeout(() => console.log('Timeout 2'))

  Promise.resolve().then(() => console.log('Promise 2'))
});

```

如果这个两个问题你都能人肉parse在脑中得出正确结果，那说明你真的懂了。否则，再重新多看几遍，下面的参考资料值得都看一遍。

不懂不写，就酱。


# 需要研究的内容太多，未完待续。。。


参考资料：

---
* [WHATWG官方event-loop规范](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)
* [ecma262 sec-jobs-and-job-queues](https://tc39.es/ecma262/#sec-jobs-and-job-queues)
* [ECMA262中Agent的定义](https://tc39.es/ecma262/#sec-agents)
* [浏览器进程？线程？傻傻分不清楚！](https://imweb.io/topic/58e3bfa845e5c13468f567d5)
* [tasks-microtasks-queues-and-schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
* [从event loop规范探究javaScript异步及浏览器更新渲染时机](https://github.com/aooy/blog/issues/5)
* [JS:macrotask和microtask](http://www.kenote.me/notes/notedetail.html?fileId=388)
* [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)
* [深入理解js事件循环机制（浏览器篇）](http://lynnelv.github.io/js-event-loop-browser)
* [JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
* [event-loop规范翻译](https://whatwg-cn.github.io/html/multipage/webappapis.html#%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF)
* [关于JavaScript单线程的一些事](https://github.com/JChehe/blog/blob/master/posts/%E5%85%B3%E4%BA%8EJavaScript%E5%8D%95%E7%BA%BF%E7%A8%8B%E7%9A%84%E4%B8%80%E4%BA%9B%E4%BA%8B.md)
* [深入浏览器的事件循环 (GDD@2018)](https://zhuanlan.zhihu.com/p/45111890)
* [Reverting nextTick to Always Use Microtask](https://gist.github.com/yyx990803/d1a0eaac052654f93a1ccaab072076dd)
* [模拟实现 JS 引擎：深入了解 JS机制 以及 Microtask and Macrotask](https://juejin.im/post/5c4041805188252420629086#heading-0)
* [总结：JavaScript异步、事件循环与消息队列、微任务与宏任务](https://juejin.im/post/5be5a0b96fb9a049d518febc)
* [JS引擎线程的执行过程的三个阶段](https://juejin.im/post/5c7a9b92518825153f784e14)
* [你真的理解$nextTick么](https://juejin.im/post/5cd9854b5188252035420a13)
* [JavaScript异步机制详解](https://juejin.im/post/5a6ad46ef265da3e513352c8)
* [Node的事件循环](https://juejin.im/post/5c337ae06fb9a049bc4cd218)