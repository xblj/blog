---
title: Node.js指南（五）-事件循环（译）
draft: true
date: 2019-01-24 22:50:54
tags:
categories:
  - Node.js指南
---

[原文地址](https://flaviocopes.com/)

# 事件循环

事件循环是理解理解`javascript`最终要的部分，本节将简要的解释它。

- [简介](#简介)
- [阻塞事件循环](#阻塞事件循环)
- [调用栈](#调用栈)
- [简单的介绍事件循环](#简单的介绍事件循环)
- [函数执行队列](#函数执行队列)
- [ES6 任务队列](#ES6-任务队列)
- [nextTick](#nextTick)
- [setImmediate](#setImmediate)
- [Timers](#Timers)

## 简介

事件循环是理解理解`javascript`最终要的部分，本节将简要的解释它。

> 使用`javascript`编程多年，但是依旧没有完全理解它是在如何运行的。不知道事件循环的详细概念也是是可以的，但是通常来说，知道是如何运行的是非常有好处的，目前你可能只是有一点好奇而已。

本篇文章主要解释`javascript`的单线程是如何运行和执行异步函数。

`javascript`代码是单线程运行，同一时间只能做一件事情。

这个限制是非常有用的，它很大程度上简化了并发编程。你只需要关注自己的代码不要阻塞当前线程，如：同步请求或者无限循环。

通常来说，大多数浏览器的每一个 tab 都有一个事件循环，这样能让每一个进程之间相对隔离，避免一个页面的无限循环或者繁重的任务处理阻塞整个浏览器。

浏览器环境管理多个并发事件循环，例如处理 API 调用。[Web Work](https://flaviocopes.com/web-workers/)同样运行在他们自己的事件循环中。

你唯一需要注意的是你的代码运行在一个单一的事件循环中，在编写代码时不要阻塞它。

## 阻塞事件循环

任何 `JavaScript` 代码如果长时间控制着事件循环不返回，那么将会阻塞这个页面的其他代码运行，甚至是阻塞 UI 进程，并且用户不能点击，滚动页面等等。

在 `javascript` 中几乎所有的 I/O 默认都是不阻塞的， 如：网络请求，Node.js 文件系统操作等。阻塞是例外，这也是 `javscript` 如此依赖回调和最近[promises](https://flaviocopes.com/javascript-promises/)和[async/await](https://flaviocopes.com/javascript-async-await/)。

## 调用栈

调用栈是一个`LIFO`队列（后进先出）。

事件循环会一直检测**调用栈**，看是否有函数需要执行。

在这个检测过程中，它将找到的函数添加到调用栈中，并按照顺序执行。

你了解调试器或者浏览器的 console 中错误堆栈信息吗？浏览器通过在当前的调用栈中查找函数名称来向你展示当前函数是由哪个函数调用：

{% asset_img err-stack.png %}

## 简单的介绍事件循环

举个列子：

{% asset_img 2.png %}

输出：

{% asset_img 3.png %}

当这段代码运行时，首先`foo()`被调用。在`foo()`内，我们首先调用`bar()`，然后再调用`baz()`。此时这个调用栈看起来是这样：

{% asset_img 4.png %}

事件循环在每次迭代中都会去检查调用栈中是否有函数需要执行，如果有就会去执行它,直到调用栈被清空位置。

{% asset_img 5.png %}

## 函数执行队列

上面的例子很简单，并没有什么特殊的地方：`javascript` 找出需要执行的函数，并按顺序执行。

让我们看看如何延迟执行一个函数，直到调用栈被清空。

这个例子使用`setTimeout(() => {}, 0)`调用函数，但是执行该函数是在其他函数都执行完之后。例子如下：

{% asset_img 6.png %}

输出如下，可能有点意外：

{% asset_img 7.png %}

当代码运行时，首先`foo()`被调用，在`foo()`中首先调用`setTimeout`，并将`bar`作为参数，通过传递参数`0`使它能尽可能早的运行，然后再调用`baz()`。此时调用栈如下：

{% asset_img 8.png %}

程序中所有函数的运行顺序如下（译注：下图的执行顺序应该是画错了应该是先执行 baz()和 console.log('baz')，再执行 bar()和 console.log('bar')）：

{% asset_img 9.png %}

为什么会这样？

## 消息队列

当 setTimeout()被调用时，浏览器或者是 Node.js 会开启一个 [timer](https://flaviocopes.com/timer-api/)，一旦这个 timer 到期，在我们这个例子中，我们使用 0 作为过期时间，回调函数会立刻被推入到**消息队列**中。

消息队列和用户触发的点击事件、键盘事件或者[fetch](https://flaviocopes.com/fetch-api/)请求一样在代码中排队，一旦触发对应事件就会调用相应的代码相应。或者如[DOM](https://flaviocopes.com/dom/)事件的 `onLoad`一样

**事件循环优先处理调用栈，并且首先处理在调用栈中能找到的任何东西，一旦调用栈没有任何东西，它将去检测事件队列**

我们不需要等待如：`setTimeout`、fetch 或者其它一些函数完成，因为他们是由浏览器提供，他们有自己的线程。举例：如果你将`setTimeout`的过期时间设置为 2 秒，你不需要等待 2 秒-这个等待时间发生在另外的地方。

## ES6 任务队列

[ECMAScript 2015](https://flaviocopes.com/ecmascript/)引入了一个有用于 `Promises` 的任务队列概念（ES6/ES2015 提出的），用于尽快执行 async 函数的结果，而不是放在调用栈的最后。

Promises 在当前函数结束前决议，那么将在当前函数执行完后立即执行。

> 译注：这个地方应该是说的是当前主进程结束前决议，会在主进程结束后立即执行。这篇文章解释比较清楚[Event Loop and the Big Picture — NodeJS Event Loop Part 1 ](https://jsblog.insiderattack.net/event-loop-and-the-big-picture-nodejs-event-loop-part-1-1cb67a182810)，例子如下:

{% asset_img 10.png %}

> 输出如下： {% asset_img 11.png %}

这和游乐园的过山车类似：消息队列是将你放在队列中所有人的最后，而任务队列是就是快速票，可以在你完成了上一个任务后快速的再乘坐一次。

例子: {% asset_img 12.png %}

输出： {% asset_img 13.png %}

Promises（还有基于 promises 的 Async/await）和通过`setTimeout()`或者其他平台 APIs 实现的异步函数有很大的不同。

## nextTick

**Node.js 的 process.nextTick 函数通过一个特殊的方法与事件循环交互**

想要理解[Node.js 事件循环](https://flaviocopes.com/node-event-loop/)，最重要的一部分就是`process.nextTick()`。

事件循环执行一个完整过程，称为一个`tick`。

当我们传递了一个函数给`process.nextTick()`，是要求引擎在当前操作结束后，并在下次事件循环`tick`开始前调用该函数。

```javascript
process.nextTick(() => {
  // do something
});
```

事件循环首先处理当前函数代码。

当前操作结束后，JS 引擎开始运行在当前操作期间传递给`nextTick`的所有函数。

这是另一个方法让 JS 引擎尽快的异步（当前函数（译注：同上解释）之后）执行函数，而不是将它放在队列最后。

调用`setTimeout(() => {}, 0)`将在下一个`tick`执行回调函数，会比使用`nextTick()`更晚调用。

如果你想在下一次事件循环迭代开始之前执行代码，那么就使用`nextTick()`。

## setImmediate

**Node.js 的 setImmediate 函数通过一个特殊方式与事件循环交互**

当你想异步执行代码，但是却不想尽快的执行，一个选择就是使用 Node.js 提供的`setImmediate()`函数。

```javascript
setImmediate(() => {
  // run something
});
```

任何作为参数传递给 setImmediate()的函数，会在下一次事件循环迭代执行。

`setImmediate`与`setTimeout(() => {}, 0)`（0ms 作为过期时间）和`process.nextTick()`有什么不同？

传递给`process.nextTick()`的函数会在你当前事件循环运行结束时调用。会在`setTimeout`和`setImmdiate`之前调用。

`setTimeout()`0ms 延迟回调与`setImmdiate()`是一样的。执行的顺序依赖于不同的因素，但是都是在下一个事件循环迭代。

## Timers

**在 javascript 代码，想要延迟执行函数，学习如何使用 setTimeout 和 setInterval 来安排函数在未来某个时间执行**

- [setTimeout()](#setTimout)
  - [零延迟](#零延迟)
- [递归 setTimeout()](#递归-setTimeout)

### setTimeout

在编写[javascript](https://flaviocopes.com/javascript/)代码时，你可能想要延迟执行函数。

`setTimeout`是一个任务。指定一个稍后执行的函数，和一个想要延迟多久运行的值，单位毫秒：

```javascript
setTimeout(() => {
  // runs after 2 seconds
}, 2000);

setTimeout(() => {
  // runs after 50 milliseconds
}, 50);
```

这个语法定义了一个新的函数。你能调用任何其他你想调用的函数，或者传递一个函数的名称，和一系列的参数：

```javascript
const myFuction = (firstParam, secondParam) => {
  // do something
};

// runs after 2 seconds
setTimeout(myFunction, 2000, firstParam, secondParam);
```

`setTimeout`返回一个计时器 id。通常不会用到这个 id，但是你能暂存，并且通过清除他来取消一个之前计划的函数调用：

```javascript
const id = setTimeout(() => {
  // should run after 2 seconds
}, 2000);

// 我改变主意了
clearTimeout(id);
```

如果指定延迟为`0`，回调会在当前函数执行后尽快执行：

```javascript
setTimeout(() => {
  console.log('after');
});

console.log('before');
```

输出：`before after`

通过将函数排队，来处理集中任务和执行大量计算的函数时避免阻塞 CPU 尤其有用。

> 一些浏览（IE 和 Edge）实现了具有相同功能的`setTmmediate()`方法，但是这并不是标准，并且在其他浏览器上是不支持的。但他是 Node.js 的标准。

### setInterval()

`setInterval`和`setTimeout`相似，有一个不同之处：回调函数并不是运行一次，而是会每隔一段指定时间（毫秒）就会调用一次。

```javascript
setInterval(() => {
  // runs every 2 seconds
});
```

上面的函数会每隔 2 秒运行一次，除非你将`setInterval`返回的 id 传递给`clearInterval`来终止调用：

```javascript
const id = setInterval(() => {
  // runs every 2 seconds
});

clearInterval(id);
```

通常在`setInterval`内部调用`clearInterval`，让计时器自动确定是继续还是停止。例如下面的代码会不断调用，直到 App.somethingIWait 的值为`arrived`：

```javascript
const interval = setInterval(() => {
  if (App.somethingIWait === 'arrived') {
    clearInterval(interval);
    return;
  }
  // otherwise do things
}, 100);
```

### 递归 setTimeout

`setInterval`每隔 n 毫秒就会调用一次函数，不会考虑函数什么时间结束执行。

如果函数运行总是花费相同的时间，一切都很好： {% asset_img 14.png %}

如果依赖网络因素，那么函数每次执行时间可能不一样，例如：{% asset_img 15.png %}

也许上一个长时间执行，与下一个重叠：{% asset_img 16 %}

为了避免上面的情况，可以在回调函数执行完后递归调用`setTimeout`：

```javascript
const myFunction = () => {
  // do something

  setTimeout(myFunction, 1000);
};

setTimeout(myFunction, 1000);
```

实现的场景：{% asset_img 17.png %}

在[Node.js](https://flaviocopes.com/node/)中可以通过[Timers Module](https://nodejs.org/api/timers.html)使用`setTimeout`和`setInterval`。

Node.js 也提供与`setTimeout(() => {}, 0)`功能相同的`setImmdiate()`，主要用于 Node.js 事件循环。
