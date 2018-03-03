# The Node.js Event Loop, Timers, and process.nextTick()

## What is the Event Loop? 

- 事件循环允许 node.js 执行非阻塞 I/O 操作 -- 尽管事实上 JavaScript 是单线程的 -- 通过只要有可能就将操作卸载到系统内核

- 大多数现代内核都是多线程的，它们可以处理在后台执行的多个操作。当这些操作中的一个完成时，内核告知 Node.js 以便可以将适当的回调添加到 poll queue（轮询队列）中来被最终执行。

## Event Loop Explained

- 当 Node.js 启动时， 它会初始化事件循环，处理被提供的输入脚本（或者放入 REPL，本文档未涉及），这可能会导致异步 API 调用，调度定时器，或者调用 ``` process.nextTick()``` ，然后开始处理事件循环。

- 下图显示了事件循环的操作顺序的简化概述

``` 
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘


```
<strong>note</strong>: 每一个框将被称为事件循环的一个阶段

- 每个阶段都有一个包含回调的 FIFO queue （先进先出队列）被执行。尽管每个阶段都有其特殊性，但通常来说，当事件循环进入到一个给定的阶段，它将执行特定于该阶段的任何操作，然后执行该阶段的回调，直到该阶段的回调被耗尽，或者已经执行最大数量的回调。当队列中的回调已经被耗尽，或者达到最大被执行数，事件循环将会移动到下一个阶段，以此类推。

- 由于这些操作中的任何一个操作都可能调度更多的操作，并且在 poll phase 轮询阶段中处理的新事件由内核排队，因此轮询事件可以在轮询事件正在被处理的同时排队。结果就是，长时间运行的回调可以允许轮询阶段的运行时间长于定时器给定的阈值。

- <strong>note</strong>: windows 和 Unix/Linux 之间的实现略有差别，但是在这里的演示中不重要。实际上有七八个步骤，但我们关心的那些 -- Node.js实际使用的那些 -- 就是上述的那些。

## 阶段概述

- <strong>timers: </strong> 这个阶段执行被 ``` setTimeout() ``` 和 ``` setInterval() ``` 调度的回调
- <strong>I/O callbacks: </strong> 执行几乎所有的回调，除了关闭回调，定时器计划的回调和 ``` setImmediate() ``` 之外的。
- <strong>idle, prepare: </strong> 只被内部调用
- <strong>poll: </strong> 检索(或者说取回)新的 I/O 事件；适当时节点将在此处阻塞。
- <strong>check: </strong> ``` setImmediate() ``` 回调在这被调用.
- <strong>close callbacks: </strong> 例如 ``` socket.on('close', ...) ```;

- 在事件循环的每次运行之间，Node.js检查它是否正在等待任何一部 I/O 或者定时器，并在没有任何异步 I/O 或定时器是清除关闭

## 阶段详情

### timers

- 一个定时器指定了一个被提供的回调可能被执行的阈值，而不是人们希望的执行的准确时间。定时器回调会在指定的时间过去之后当它们能被调度的时候尽可能早的执行；然后操作系统调度或者运行的其他回调可能会延迟它们。

- <strong>Note: </strong> 技术上来说，poll phase 轮询阶段控制定时器什么被执行

- 例如，假设你安排一个超时时间在100ms的阈值后执行，然后你的脚本异步开始读取需要95ms的文件：

```
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});

```
- 当事件循环进入到（poll phase）轮询阶段，它有一个空队列(``` fs.readFile() ```还没有完成)，因此它将等待剩余的毫秒数直到已经达到最快的定时器阈值(上面例子代码中只有一个定时器并且阈值是100ms)。当它等到95ms过去之后，```fs.readFile()```完成读取文件并且它的将要花费10ms来完成的回调被添加到 poll queue轮询队列，并且被执行。当回调完成后，队列中没有更多的回调，因此事件循环会看到最快的定时器的阈值已经达到（上面例子中的100ms），然后回到 timer phase 定时器阶段来执行定时器阶段的回调。在上面这个例子中，你将会看到调度的定时器和它被执行的回调的总延迟是105ms （95ms + 10ms）

- <strong>Note: </strong> 为了防止轮询阶段造成事件循环饥饿，libuv（实现Node.js事件循环和所有平台的异步行为的C库）也有一个硬性的最大值（取决于系统）来在它停止轮询更多事件之前。

### I/O callbacks
- 这个阶段执行一些例如TCP错误的类型的系统操作的回调。例如，当一个TCP socket 尝试连接时接收到 ```ECONNREFUSED ```， 一个 *nix系统想等待来报错该错误。这个在 I/O callbacks 阶段将会被排队处理。

### poll
- poll phase 轮询阶段有两个主要功能：
		
	1. 为阈值已过的定时器执行脚本，然后
	2. 处理poll queue 轮询队列中的事件

- 当事件循环进入 poll phase（轮询阶段）并且没有定时器要调度，以下两件事之一会发生：
	- 如果 poll queue 轮询队列不是空的，事件循环将会遍历它的回调队列来同步执行这些回调，直到队列耗尽，或者达到系统硬性限制阈值
	- 如果 poll queue 轮询队列是空的，以下事件之一会发生：
		- 如果脚本已经被 ``` setImmediate() ``` 调度，事件循环将终结 poll phase 轮询阶段，并继续执行 check phase 来执行被调度的脚本
		- 如果脚本没有被 ``` setImmediate() ``` 调度，事件循环将会等待回调被添加到队列，然后立刻执行这些回调。
- 一旦 poll queue 轮询队列为空，事件循环将会检查定时器来查看哪个定时器的阈值已经达到。如果一个或者多个定时器已经准备好，事件循环将会折回到 timer phase 定时器阶段来执行这些定时器的回调。

### check

- 这个阶段允许人在 poll queue 轮询阶段完成后立刻执行回调。如果 poll queue 轮询阶段变成 idle （空闲状态） 并且脚本已经被 ```setImmediate()``` 排队，那么事件循环可能会继续执行 check phase 而不是等待。

- ```setImmediate()``` 事实上是一个特定的定时器，它在事件循环的一个单独阶段中运行。它使用 libuv API来调度回调以便在 poll phase 轮询阶段完成后进行执行回调。

- 通常来说，随着代码的执行，事件循环最终进入 poll phase 轮询阶段，在轮询阶段等待传入连接，请求，等等。然而，如果一个回调已经被 ```setImmediate()```调度并且 poll phase 轮询阶段已经变成idle（空闲状态），它将终结poll phase轮询阶段并继续执行 check phase 而不是等待 poll events 轮询事件。

### close callbacks

- 如果一个 socket或者 handle 被突然性关闭（比如 ```socket.destroy()```）， ```close```事件将在这个阶段被发射。否则它将会通过 ```process.nextTick()```。


# ```setImmediate()``` VS ```setTimeout()```

- ```setImmediate()``` 和 ```setTimeout()``` 是相似的，但取决于它们何时被调用，其行为方式不同
	- ```setImmediate()``` 被设计用来一旦当前poll phase 轮询阶段完成后执行一个脚本
	- ```setTimeout()``` 调度脚本以 ms 为单位运行最小阈值后运行脚本

- 定时器的执行顺序是不同的，取决于它们被调用的上下文。如果都从主模块内部中被调用，那么时序将受过程性能的约束（这可能会受到机器上运行的其他应用程序的影响）

- 比如，如果我们运行下面的不在 I/O 周期（比如：主模块）的脚本，下面的两个定时器的执行顺序是不确定的，正如它被执行过程的性能约束：

```
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

- 然而，如果你将两次调用移动到一个 I/O 周期内， immediate callback总是第一个被执行的：

```
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```


- <strong>使用 ```setImmediate()```而不是 ```setTimeout()```的主要优势是 ```setImmediate()```将始终在任何定时器（setTimeout 和setInteravel）之前执行（如果在 I/O 周期内执行），而不考虑存在多少个定时器</strong>


## ```process.nextTick()```

### understanding ``` process.nextTick()```

- 你可能已经注意到 ```process.nextTick()```没有出现在上图中，虽然它也是异步API的一部分。这是因为 ```process.nextTick()```从技术上来说说并不是事件循环的一部分。相反， ``` nextTickQueue``` 将在当前操作完成后被处理，而不管事件循环的当前阶段
