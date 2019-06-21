# 当Node.js遇到CPU密集型任务

Node.js采用事件驱动和异步I/O的方式，实现了一个单线程、高并发的JavaScript运行时环境。Node.js为那些需要能够实时响应大量并发用户请求的web应用程序提供了一个绝佳的平台。

用Node.js处理I/O密集型任务相当简单，只需要调用它准备好的异步非阻塞函数就行了。然而web应用程序并不是只有I/O密集型任务，当碰到CPU密集型任务时，要进行大量的计算，比如要对数据加解密这时候该怎么办呢？我们先来了解下 Node.js 自身的编程模型。

## 常用高并发策略

1. 服务器为每个客户端请求分配一个线程，使用同步I/O，系统通过线程切换来弥补同步I/O调用的时间开销。Java和Apache都是这种策略，这种策略还是很多交互式应用的首选。由于I/O一般都是耗时操作，这种策略很难实现高性能，但非常简单，可以实现复杂的交互逻辑。

2. 服务器用一个线程处理所有客户端请求，使用异步的I/O及事件机制。node.js 采用的就是这种策略。这种策略实现起来比较简单，方便移植，也能提供足够的性能，但无法充分利用多核CPU资源

事实上：大多数网站的服务器端都不会做太多的计算，它们接收到请求以后，把请求交给其它服务来处理（比如读取数据库），然后等着结果返回，最后再把结果发给客户端。因此， Node.js针对这一事实采用了单线程模型来处理，它不会为每个接入请求分配一个线程，而是用一个主线程处理所有的请求，然后对I/O操作进行异步处理，避开了创建、销毁线程以及在线程间切换所需的开销和复杂性。

## 事件循环Event Loop

Node.js 在主线程里维护了一个事件队列，当接到请求后，就将该请求作为一个事件放入这个队列中，然后继续接收其他请求。当主线程空闲时(没有请求接入时)，就开始循环事件队列，检查队列中是否有要处理的事件，这时要分两种情况：
1. 如果是非I/O任务，就亲自处理，并通过回调函数返回到上层调用。
2. 如果是I/O任务或本地API调用，就从线程池中拿出一个线程来处理这个事件，并指定回调函数，然后继续循环队列中的其他事件。当线程中的I/O任务或本地API调用完成以后，就执行指定的回调函数，并把这个完成的事件放到事件队列的尾部，等待事件循环，当主线程再次循环到该事件时，就直接处理并返回给上层调用。 

整个过程就叫事件循环 (Event Loop)

JavaScript代码全在Event Loop这个单线程下运行，当socket上有数据过来，或本地 API 函数返回时，需要有种同步的方式调用对刚发生的这一特定事件感兴趣的JavaScript函数。在发生事件的线程中直接调用JS函数是不安全的，因为那样会遇到常规多线程程序遇到的问题，竞态条件、非原子操作的内存访问等等。所以要以一种线程安全的方式把事件放在队列中，伪代码：
```
lock (queue) {
    queue.push(event);
}
```

然后在执行JavaScript的主线程中即（event loop）：

```
while (true) {
    // tick 开始 

    lock (queue) {
        var tickEvents = copy(queue); // 将当前队列中的条目复制的线程自有的内存中 
        queue.empty(); // 清空共享的队列 
    }

    for (var i = 0; i < tickEvents.length; i++) {
        InvokeJSFunction(tickEvents[i]);
    }

    // tick 结束 
}
```

那么这个队列中的东西都是谁放进来的呢？

+ process.nextTick
+ setTimeout/setInterval
+ I/O (来自 fs、net 等)
+ crypto 中的 CPU 密集型函数(crypto模块提供通用的加密和哈希算法)
+ 所有使用libuv 工作队列异步调用 C/C++ 库的本地模块

## Node.js层次结构

Node.js被分为了四层，分别是应用层、V8引擎层、Node API层 和 LIBUV层

+ 应用层：即JavaScript交互层，常见的就是 Node.js 的模块，比如 http，fs
+ V8引擎层：即利用V8引擎来解析JavaScript语法，进而和下层API交互
+ NodeAPI层：为上层模块提供系统调用，一般是由C语言来实现，和操作系统进行交互 
+ LIBUV层：是跨平台的底层封装，实现了事件循环、文件操作等，是 Node.js 实现异步的核心 

无论是 Linux 平台还是 Windows 平台，Node.js 内部都是通过 线程池 来完成异步 I/O 操作的，而 LIBUV 针对不同平台的差异性实现了统一调用。**因此，Node.js 的单线程仅仅是指 JavaScript 运行在单线程中，而并非 Node.js 是单线程。**

## 当Event loop遇到CPU密集型任务

因为event loop在处理所有事件时，都是沿着事件队列顺序执行的，所以在其中任何一个事件本身没有完成之前，其它的回调、监听器、超时、nextTick()的函数都得不到运行的机会，因为被阻塞的event loop根本没机会处理它们，此时程序最好的情况是变慢，最糟的情况是停滞不动，像死掉一样。所以当 Node.js 遇到高 CPU 占用率的任务时，event loop 会被阻塞住，形成下面这种局面：

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/cpu.jpg" height="300">

### 被阻塞的Event loop

代码清单 1. 快速行进的 event loop
```
(function spinForever() {
  process.stdout.write(".");
  process.nextTick(spinForever);
})();

```
在这段代码中再加上一个计算斐波那契数列的任务
代码清单 2. 被高 CPU 占用率计算阻塞的 event loop
```
function fibo (n) {
  return n > 1 ? fibo(n - 1) + fibo(n - 2) : 1;
}

(function fiboLoop () {
  process.stdout.write(fibo(45).toString());
  setTimeout(fiboLoop, 0);
})();

(function spinForever() {
  process.stdout.write(".");
  setTimeout(spinForever, 0);
})();
```
计算斐波那契数列是一个 CPU 密集型的任务，event loop 要在计算结果出来后才有机会进入下一个 tick，执行输出.的简单任务，感觉就像服务器死掉了一样。

## 被闲置的 CPU 内核
现在的操作系统都支持多核，那么每个线程都可以得到一个自己的内核，实现真正的“并行运算”。在这种情况下，多线程程序可以提高资源使用效率。Node.js是单线程程序，它只有一个event loop，也只占用一个内核。当 Node.j程序的event loop被CPU密集型的任务占用，导致有其它任务被阻塞时，却还有CPU/内核处于闲置的状态，造成资源的浪费。

运行代码清单2中的代码，启动top(查看CPU的使用情况。当node的CPU占用率在100% 左右浮动时，系统的 CPU 占用率却只有30%左右。

既然 Node.js 程序几乎完全运行在单个CPU/ 内核上，所以我们需要做些额外的工作才能提升它的扩展性。Node.js 提供了一组管理进程的API，还允许你给它编写本地扩展，所以有很多种不同的办法可以让程序的代码并行运行
Node.js 中有管理子进程的child_process模块，可以用fork()方法创建新的子进程实例
接下来我们fork()一个子进程，把计算斐波那契数列的任务交给它，这需要两个文件。
代码清单 3. 主进程文件 forkParent.js
```
var cp = require('child_process');

var child = cp.fork(__dirname+'/forkChild.js');

child.on('message', function(m) {
  process.stdout.write(m.result.toString());
});

(function fiboLoop () {
  child.send({v:40});
  setTimeout(fiboLoop, 0);
})();


(function spinForever () {
  process.stdout.write(".");
  setTimeout(spinForever, 0);
})();

```
代码清单 4. 计算斐波那契数列的子进程文件 forkChild.js

```
function fibo (n) {
  return n > 1 ? fibo(n - 1) + fibo(n - 2) : 1;
}

process.on('message', function(m) {
  process.send({ result: fibo(m.v) });
});
```
结果跟我们预期的一样，输出.的任务不再受到阻塞，欢快地在屏幕上刷了一大堆.，然后每隔一段输出一个。我们再用top查看一下资源的使用情况，会发现有两个 node 进程，CPU 占用率也增加了很多。

除了本次演示用到的child_process模块，node官方还提供给了cluster模块，还有热心团队Mozilla Identity开发的node-compute-cluster模块的等等都为node处理CPU密集型任务，感兴趣的同学可以自行查阅~