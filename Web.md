# Web相关

## 浏览器引擎渲染过程

以前写过一则[阅读笔记](https://www.everseenflash.com/CS/Frontend/Browser%20Engine.md)，包括`<style>`和`<script>`在文档中位置的安排、回流重绘等内容。

## HTML

### `<script async>`和`<script defer>`

两者都不会阻塞浏览器解析DOM的过程，`defer`和`async`的区别在于，`defer`脚本还包含在整个加载流程内，它总是在DOM解析完毕，`DOMContentLoaded`事件之前执行；而`async`脚本完全独立，什么时候加载好就什么时间执行，和`DOMContentLoaded`事件的顺序无法确定。

### `<div>`和`<span>`

### `<img>`的`srcset`属性

## 窗口大小和像素比

## Ajax和`fetch`

不会吧不会吧不会吧，不会真有人在2023年了还在用古老的JSONP和AJAX吧。

## DOM Event

## 事件循环相关

### Web平台

### NodeJS平台

下面这段来自NodeJS[官方文档](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)，不过这篇文章挺毒的，很多地方含糊不清：

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │   pending callbacks   │
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

事件循环自NodeJS启动后就开始，运行在主线程中，如果没有待执行的回调就会退出。每轮循环有图中几个阶段（Phase，每个方框代表一个阶段），每个阶段都有一个FIFO队列存放待处理的回调。

+ timers: 已到期的`setTimeout`、`setInterval`回调在这里执行；
+ pending callbacks: 系统调用的结果回调，例如试图上报TCP错误`ECONNREFUSED`的行为；
+ idle, prepare: NodeJS内部使用；
+ poll: 取回新的I/O事件，处理除了`close`、timer和`setImmediate`回调之外几乎所有的I/O回调；
+ check: `setImmediate`回调在这里执行；
+ close callbacks: `close`事件有关的回调，例如`socket.on('close', callback)`。

其中最核心的阶段是poll，如果进入poll阶段时poll队列为空，则会在此处等待：

1. 若有`setImmediate`回调则进入check；
2. 若没有`setImmediate`回调则会在此处一直等待，随后可能发生的情况是：计时器到期，则有回调进入timers队列，于是“折返”到timers阶段进行新一轮循环。

通过一些例子加深理解：

1. 下面这段代码中，`setImmediate`始终在`setTimeout`前面执行：

    ```js
    fs.readFile('some file', () => {
      setTimeout(() => console.log('setTimeout'));
      setImmediate(() => console.log('setImmediate'));
    });
    ```

    `readFile`的回调在poll阶段执行，随后poll队列为空，但是回调执行过程中调用了`setImmediate`，于是进入check阶段，下一轮再执行`setTimeout`的回调。

2. 如果去掉外侧的`readFile`则不一定了，下面代码`setTimeout`有可能在`setImmediate`前面执行：

    ```js
    setTimeout(() => console.log('setTimeout'));
    setImmediate(() => console.log('setImmediate'));
    ```

    `setTimeout`默认的等待间隔为1ms，主线程执行后，开始事件循环进入timers阶段时，如果还没到1ms，则timers队列为空，依次进入下面的阶段，先执行`setImmediate`回调，反之先执行`setTimeout`的回调。

3. 这个例子说明了`setTimeout`设定的等待间隔并不可靠，NodeJS只保证到时间后timer回调被”尽快“地执行：

    ```js
    const scheduledTime = performance.now();

    setTimeout(() => {
      console.log(`elapsed time: ${performance.now() - scheduledTime}ms`);
    }, 300);

    fs.readFile('a big file', () => {
      const start = performance.now();

      while (performance.now() - start < 1000);
      console.log('called');
    });
    ```

    这里让`readFile`读取一个大文件，耗时在200ms左右。因此poll阶段等待一段时间后`readFile`完成，其回调进入poll队列，回调也执行完成后才进入下一轮循环执行timers，最终先输出`called`，随后打印的耗时在1200ms左右；极小的情况下，poll阶段等待过程中`setTimeout`先于`readFile`完成，则“折返”到timers阶段先执行timer回调，然后依次又一次进入poll阶段，这时打印的耗时是300ms多，随后才输出的`called`。

#### `process.nextTick()`

NodeJS自己也吐嘈了，它的`nextTick`有立即执行的作用，而`setImmediate`却有下一个“tick”才执行的作用，这都是历史遗留的因素。按照官方的文档，不管当前是事件循环的哪一个阶段，nextTickQueue会在当前“操作”完成后立即处理，这里“操作”是来自NodeJS底层的概念，我推测代表的就是回调任务。nextTickQueue总是在进入下一个阶段前处理完，因此如果递归`nextTick`的话可能因为迟迟进入不了poll阶段造成I/O“饥饿”。

作为微任务，`process.nextTick()`优先级甚至比`Promise`还高，下面的代码会先输出`2`再输出`1`。

```js
new Promise<void>((res) => res()).then(() => console.log('1'));

process.nextTick(() => console.log('2'));
```

在有了`queueMicrotask`之后，`process.nextTick()`可以被视为不推荐使用了，`queueMicrotask`创建的是和`Promise`同等优先级的微任务，便于梳理程序执行流。

## 二进制数据的处理

[ZSBD](https://www.everseenflash.com/CS/Frontend/Binary.md)

## hash和history路由

## `cookie`、`sessionStorage`和`localStorage`

## CORS

## XSS和CSRF

## Web Component

## 移动端兼容性问题

### `1px`边框

### H5键盘

### 微信白屏
