# Node定时器详解

- 原文在[这里](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)

JavaScript 是单线程运行，异步操作特别重要

只要用到引擎之外的功能，就需要跟外部交互，从而形成异步操作。由于异步操作实在太多，JavaScript 不得不提供很多异步语法。这就好比，有些人老是受打击， 他的抗打击能力必须变得很强，否则他就完蛋了

Node 的异步语法比浏览器更复杂，因为它可以跟内核对话，不得不搞了一个专门的库 libuv 做这件事。这个库负责各种回调函数的执行时间，毕竟异步任务最后还是要回到主线程，一个个排队执行。


为了协调异步任务，Node 居然提供了四个定时器，让任务可以在指定的时间运行。

```
- setTimeout()
- setInterval()
- setImmediate()
- process.nextTick()
```

前两个是语言的标准，后两个是 Node 独有的。它们的写法差不多，作用也差不多，不太容易区别

你能说出下面代码的运行结果吗？

```
//test.js
setTimeout(() => console.log(1));
setImmedaite(() => console.log(2));
process.nextTick(() => console.log(3));
Promise.resolve().then(()=> console.log(4));
(()=>console.log(5))();

```

运行结果如下：
```
5
3
4
1
2
```
如果你能一口说对，可能就不需要再看下去了。本文详细解释，Node怎么处理各种定时器，或者更广义地说，libuv库怎么安排异步任务在主线程上执行。

# 一.同步任务和异步任务
首先，同步任务总是比异步任务更早执行

前面的代码，只有最后一行是同步代码，因此最早执行。
```
(()=> console.log(5));
```

# 二.本轮循环和次轮循环

异步任务可以分成两种。

```
- 追加在本轮的异步任务
- 追加在次轮的异步任务
```
所谓"循环"，指的是事件循环（event loop）。这是 JavaScript 引擎处理异步任务的方式，后文会详细解释。这里只要理解，本轮循环一定早于次轮循环执行即可

Node 规定，process.nextTick和Promise的回调函数，追加在本轮循环，即同步任务一旦执行完成，就开始执行它们。而setTimeout、setInterval、setImmediate的回调函数，追加在次轮循环.

这就是说，文首那段代码的第三行和第四行，一定比第一行和第二行更早执行。

```
// 下面两行，次轮循环执行
setTimeout(() => console.log(1));
setImmediate(() => console.log(2));
// 下面两行，本轮循环执行
process.nextTick(() => console.log(3));
Promise.resolve().then(() => console.log(4));

//同步任务
(()=>console.log(5))
```

# 三. process。nextTick()

```process.nextTick```这个名字有点误导，它是在本轮循环执行的，而且是所有异步任务里面最快执行的。

Node执行完所有同步任务
完所有同步任
