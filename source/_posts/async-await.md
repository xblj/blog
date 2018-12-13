---
layout: hello
title: async await
date: 2018-12-11 10:11:56
tags: async awaait
---

# async/await 到底是怎么回事？

首先我们来看看 async 函数  的基本语法：

```javascript
function req(id) {
  return new Promise(reslove => {
    //  这里用定时器模拟ajax请求
    setTimeout(() => {
      reslove('ok_' + id);
    }, 1000);
  });
}

async function main() {
  let data = await req(12);
  console.log(data);
}

main();
```

`async/await`能让我们在`main`函数内采用”同步“的方式进行异步编程，这大大减轻了我们的思维难度，避免了回调地狱和`promise`的不断`then`，它是如此的强大 。 但它是怎么做到的呢？难道 js 真能在不阻塞线程的情况下进行同步请求？当然不是这样，接下来我们就一步步分析，看看它到底时如何实现的。首先我们得依次了解以下几个概念：“迭代器”、“可迭代对象”和“生成器”（关于这几个概念的东西其实并不止本文讲的这些，还有很多，这里讲的只是与本次主题相关的部份），然后会一步步的推导出`async/await`的原理。

## 迭代器

> 迭代器就是一种实现了特定接口的对象， 所有迭代器都有一个`next`方法，每次调用`next`方法都会  返回一个对象。该对象包含两个属性：`done/value`。`done`是一个`boolean`类型的值，表示迭代是否结束字段，`value`表示返回值。

```javascript
// 一个简单的迭代器，当a大于3时表示该迭代结束。
let it = {
  a: 1,
  next() {
    if (this.a > 3) {
      return { done: true, value: this.a };
    }
    return { done: false, value: this.a++ };
  },
};

it.next(); // {done: false, value: 1}
it.next(); // {done: false, value: 2}
it.next(); // {done: false, value: 3}
it.next(); // {done: true, value: 4}
```

### for...of

for...of 是 es6 新增的，用于遍历迭代器。我们最常用的是用来遍历数组。

语法：

```javascript
// 遍历数组
let arr = [1, 2, 3];
for (let item of arr) {
  console.log(item);
}
// 1
// 2
// 3
```

既然上面说了`for...of`是用于遍历迭代器的，那我们在第一节[迭代器](##迭代器)中写的`it`就能用`for...of`来遍历了哦，下面我们来试试：

```javascript
for (let item of it) {
  console.log(item);
}

// Uncaught TypeError: it is not iterable
```

上面的代码是不会  成功运行，报错  信息上面已经写出，翻译过来就是“it 不是可迭代的”，什么叫不是可迭代的呢，这就需要  知道另外一个概念：[可迭代对象](##可迭代对象)

## 可迭代对象

我们先将第一节[迭代器](##迭代器)的代码  做一点改变：

```javascript
let it = {
  a: 1,
  // 加了一个函数
  [Symbol.iterator]: function() {
    return this;
  },
  next() {
    if (this.a > 3) {
      return { done: true, value: this.a };
    }
    return { done: false, value: this.a++ };
  },
};

// 再次运用
for (let item of it) {
  console.log(item);
}
// 1
// 2
// 3
```

`[Symbol.iterator]`这个函数有点奇怪，但是这个不是本次的重点，有机会我们下次专门来聊它，现在只需要记住加上这个函数后就能成功的遍历出值。其实这里的`it`在添加上这个函数后就变成了  一个可迭代对象。`for...of`是专门用来遍历  可迭代对象的， 在运行时会首先去调用可迭代对象的`[Symbol.iterator]`函数，并判断返回的的值  是否是一个迭代器，也就是是否有`next`函数。 然后根据`next`函数返回的对象的`done`属性的值，如果是`false`就继续调用`next`函数，将`value`的值赋值给上面`item`,如果是`true`遍历结束。

一般我们没有自己写迭代器，规范里为我们定义了一类名为[生成器](##生成器)的函数，调用  它就能得到一个可迭代对象。

## 生成器

> 生成器就是一个值的生产者，我们通过迭代器接口的 `next()` 调用一次提取出一个值。

语法：

```javascript
// 生成器
function *generator() {
    let a = 2 + (yield 3);
    return a;
}

// 调用返回一个迭代器
let it = generator();

// 调用迭代器的next方法回依次返回yield的值
let varVal_1 = it.next(); // varVal=>{done:false, value: 3} 这里返回的是第一个yield出来的值

let varVal_2 = it.next(2); // {done: true, value: 4} 这里返回值是由最后的return 返回的
```

这里有两点需要注意：

> - 1、函数名前面的`*`，表示这个函数是一个生成器函数，而不是普通函数，调用这个函数会返回一个迭代器;
> - 2、函数体内的`yield`，是只能用在生成器函数中使用，在其他函数中调用会报错;

### yield

`yield`有两个重要的作用：

> - 1、 暂停当前生成器函数的执行，并向外抛出一个值，而不阻塞主线程，直到  下次调用`next`函数,至于为什么能暂停，请看上一节[迭代器](##迭代器)。
> - 2、 接受外部通过`next(val)`调用传入的  值`val`，供函数内部  使用

### 生产器执行过程

> 1. 调用生成器函数`generator`,然后返回一个迭代器`it`;
> 1. 通过调用  迭代器`it`的`next()`方法（第一次调用`next`，不能传递参数，传了也会被忽略），让生成器运行到第一个`yield`处，并返回`yield`后的值；
> 1. 再次调用迭代器`it`的`next(val)`方法（`val`值可选，需要就传）， 让生成器运行到下个`yield`处，并返回`yield`后的值（返回的值是一个对象：`{done: boolean, value: yield后面的值}`）；
> 1. 重复的调用迭代器`it`的`next(val)`方法，都会执行第 3 步的过程，直到运行到生成器函数的结束或者遇到`return`;

下面我们看看我们发送 ajax 请求的一般操作：

在没有`Promise`之前， 我们发送请求，都是通过回调的方式：

```javascript
function req(id, cb) {
  ajax('http://xxxx?id=' + id, cb);
}

req(12, function(res) {
  console.log(res);
});
```

下面是`Promise`版本：

```javascript
function req(id) {
  return new Promise(reslove => {
    //  这里用定时器模拟ajax请求
    setTimeout(() => {
      reslove('ok_' + id);
    }, 1000);
  });
}

req(12).then(res => {
  console.log(res);
});
```

`Promise`版本解决了  令人诟病的回调地狱。但是这样就是最好的解决方案了吗？其实按照我们一般人的  思维来说更喜欢写  下面这样的代码：

```javascript
function req(id) {
  // 发送请求
  let res = ajax('http://xxxx?id='+id)；
}
function main() {
  let data = req(12);
  console.log(data);
}
main();
```

上面代码更加符合我们一般人的思维方式，没有思维负担，发送请求，等待  请求返回，使用返回值。没有回调，没有 then，一切都像是同步执行一样。但是上面的代码并不会像我们想的那样去运行， 那有  没有解决方案  来实现  呢？当然有，通过生成器就能办到。

接下来是生成器版本：

```javascript
// 调用生产起函数“main”，得到一个可迭代对象“it”
let it = main(12);
// 调用it，会运行到第一个yield
let res = it.next();

// 模拟ajax请求
function req(id) {
  return fetch(id).then(res => {
    it.next('ok_' + res);
  });
}
// 生成器
function* main(id) {
  let data = yield req(id);
  console.log(data);
}

// 模拟请求库
function fetch(id) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(id);
    }, 1000);
  });
}
```

上面看着好像比之前的代码更加的长了，写的东西更加的多了，但是先不看其他部份， 我们先看看`main`函数的内部，发送请求， 然后直接打印返回值`res`，这不是实现了同步的方式调用么？

 我们来  一步一步的分析下是如何实现在`main`方法内“同步 ”执行的：

> 1. 首先调用`main(12)`函数（生成器函数）， 返回一个迭代器`it`；
> 1. 调用`it`的`next()`方法，让`main`运行到`yield`处，然后就会去执行`yield`后的语句`req(12)`,`main`会“暂停”在这里，直到下次`next` 执行；
> 1. 执行环境进入`req`函数体内， 执行请求`fetch`返回一个`promise`；
> 1. 当数据返回后调用`then`,在`then`中执行`it.next(res)`，恢复`main` 执行，并将`res`赋值给`data`；
> 1. 打印`data`。

虽然上面的`main`函数内实现了“同步”运行，但是问题还是有很多，首先`it`变量在全局中定义，`req`函数中还依赖了这个变量，这样请求就无所谓封装了，所以也就根本没法在真实项目中有实际的用处。

其实 tj 大神就为我们写好了这样一个工具函数[co](https://github.com/tj/co)，能自动的去运行`main`函数，而不需要我们自己去不断的反复调用`next`运行，有了[co](https://github.com/tj/co)后，我们可以将上面的例子改写入如下:

```javascript
var co = require('co');

// 模拟请求库
function fetch(id) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(id);
    }, 1000);
  });
}

function req(id) {
  return fetch(id).then(res => {
    return res;
  });
}

co(function* main(id) {
  let data = yield req(id);
  console.log(data);
});
```

这样就好很多了，我们只需要关注`main`函数内容的逻辑编写，而且还是按照我们希望的“同步”方式编写，没有了烦人的回调地狱，没有了 then，整个世界都变得如此的美好了。

但是如此牛逼的工具到是怎么做到的，为了探索其中的秘密，我们再进一步看看它到底是什么原理。我们可以自己写一个类似的工具函数，这里借用下《你不知道的 javascript》里面的代码，去掉了异常处理：

```javascript
function run(gen) {
  let args = [].slice.call( arguments, 1),;
  // 在当前上下文中初始化生成器
  let it = gen.apply( this, args );
  // 返回一个promise用于生成器完成
  return Promise.resolve()
    .then(function handleNext(value){ // 对下一个yield出的值运行
      let next = it.next( value );
      return (function handleResult(next){
        // 生成器运行完毕了吗?
        if (next.done) {
          return next.value;
        } else {
        // 否则继续运行
        return Promise.resolve(next.value)
          .then(handleNext);
        }
      })(next);
  });
}
function req(id) {
   return new Promise(reslove => {
    //  这里用定时器模拟ajax请求
    setTimeout(() => {
      reslove('ok_' + id);
    }, 1000);
  });
}

function *main(id) {
  let data = yield req(id);
  console.log(data);
}

run(main, 12);
```

还是来分析下`run`函数的运行步骤：

> 1. 首先调用`gen`函数返回一个迭代器(`it`这里采用了 apply 调用是为了绑定当前上下文和处理参数，这不是我们本次研究的重点，不清楚的可以看看 apply);
> 1. 然后会返回一个立即决议的 promise，紧接着就会去调用 promise 后的 then，运行`handleNext`函数，此时没有任何值传人，所以`value`为`undefined`；
> 1. 然后调用`it`的`next(value)`方法，由于迭代器的第一次`next`调用不用传值，由上一步可知此时`value`就是`undefined`，和没有传值是一样的，`main`函数运行至第一个`yield`处，然后会去计算`req(id)`并返回一个`promise`（为了区分这里的用`promise_1`代替），然后赋值给`run`函数内的`next`变量，此时`next`的值为`{done:false, value: promise_1}`
> 1. 接着会调用立即执行函数`handleNext({done:false, value: promise_1})`，判断迭代器`it`是否完成`if (next.done)`,此时的`next.done`为`false`，所以会运行`else`,调用`Promise.resolve(promise_1)`等待`promise_1`决议后调用后面的`then`方法内的`handleNext`, 并将决议后的值（"ok_12"）传人`handleNext`函数内。
> 1. 再次调用`it.next( value )`，此时的`value`为`"ok_12"`,恢复`main`的执行，并将`"ok_12"`赋值给`data`。
> 1. `mian`函数继续往下运行打印`data`，没有`yield`和`return`，函数运行结束,默认返回`undefined`。
> 1. 第 5 步调用`it.next( value )`， 得到的返回值`{done:true, value: undefined}`。
> 1.  然后调用`handleResult`，进行`if` 判断，此时`next.done`的值为`true`， 表示迭代结束，直接返回`next.value`(undefined), `run`函数运行完毕。

这段运行还是有点  麻烦，需要结合着前面讲的[迭代器](##迭代器)、[可迭代对象](##可迭代对象)来看，这里  只是对生产器的整个过程做了一个大致演示，到这里我们基本上就实现了异步编程的“同步 ”化，和`async/await`是不是很像了呢？

本次的关于`async/await`就这些了，时间比较匆忙所以错误的地方在所难免，希望看官们多多的指正。
