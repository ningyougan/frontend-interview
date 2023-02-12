# Web相关

## 浏览器引擎渲染过程

以前写过一则[阅读笔记](https://www.everseenflash.com/CS/Frontend/Browser%20Engine.md)，包括`<style>`和`<script>`在文档中位置的安排、回流重绘等内容。

## HTML

### `<script async>`和`<script defer>`

两者都不会阻塞浏览器解析DOM的过程，`defer`和`async`的区别在于，`defer`脚本还包含在整个加载流程内，它总是在DOM解析完毕，`DOMContentLoaded`事件之前执行；而`async`脚本完全独立，什么时候加载好就什么时间执行，和`DOMContentLoaded`事件的顺序无法确定。

### `<div>`和`<span>`

## 窗口大小和像素比

## Ajax和`fetch`

不会吧不会吧不会吧，不会真有人在2023年了还在用古老的JSONP和AJAX吧。

## DOM Event

向上称为冒泡，向下称为捕获。在冒泡处理回调是默认的行为，可用`stopPropagation()`阻断，会按照自底向上的路径传播事件。但如果我们在`addEventListener`的时候设置了`{ capture: true }`属性，这时触发的事件会先从根元素向下走到目标元素，再向上回到根元素，即先捕获再冒泡，此时`stopPropagation`阻断的是捕获的过程。

有时要阻止浏览器对某些事件的默认响应行为，例如点击链接时发生跳转，可使用`preventDefault()`。

## 事件循环相关

### Web平台

宏任务的概念实际上并不存在，只是为了和微任务对应而附会的一个概念。[浏览器标准](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)中相关的概念应该是Task，代表了诸如事件调度、timers回调、DOM、网络等任务。重点在8.1.7.3节Processing Model，每次完成一个任务会有一个微任务检查点，如果当前微任务队列不为空，则持续运行微任务直到队列为空，然后才会做一次Update the rendering。

在屏幕上有一个`position: fixed`的按钮，像下面这段代码，点击按钮能明显看到按钮闪烁了一下，如果将`setTimeout`改为`queueMicrotask`则不会，这或许能作为Task和Microtask执行时机的一个例子，变更DOM是一次Task，执行完之后有一个微任务检查点，因此会先处理掉重置`btn.style.left`的回调再作渲染。而`setTimeout`创建的是Task，用`MessageChannel`和`setTimeout`是同样的效果。比较特殊的是`requestAnimationFrame`，虽然是宏任务，但从标准中可以看出它运行在Update the rendereing阶段，但处在Layout/Paint的小阶段之前，所以如果用`requestAnimationFrame`也不会闪烁。

```js
const btn = document.querySelector('button')!;

btn.style.left = '100px';
setTimeout(() => { btn.style.left = null; });
```

进一步验证，在改变按钮位置后为`100px`后先调用`setTimeout`确保渲染到屏幕上，然后重置按钮位置，但重置后立刻做一个无限嵌套自身的微任务，这时可以观察到主线程直接卡住且按钮的重置并没有被渲染出来。

```js
const btn = document.querySelector('button')!;
const infinite = () => queueMicrotask(() => infinite());

btn.style.left = '100px';

setTimeout(() => {
  btn.style.left = null;
  infinite();
});
```

常见的宏任务API：timers、`requestAnimationFrame`、`requestIdleCallback`、`MessageChannel`等。

常见的微任务API：`process.nextTick()`、`Promise.then()`、`queueMicrotask`、`MutationObserver`等。

### NodeJS平台

下面这段来自NodeJS[官方文档](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)，不过这篇文章挺毒的，很多地方没说清楚：

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

事件循环自NodeJS启动后就开始，运行在主线程中，如果没有待执行的回调主线程就会退出。每轮循环有图中几个阶段（Phase，每个方框代表一个阶段），每个阶段都有一个FIFO队列存放待处理的回调。

+ timers: 已到期的`setTimeout`、`setInterval`回调在这里执行；
+ pending callbacks: 系统调用的结果回调，例如试图上报TCP错误`ECONNREFUSED`的行为；
+ idle, prepare: NodeJS内部使用；
+ poll: 取回新的I/O事件，处理除了`close`、timer和`setImmediate`回调之外几乎所有的I/O回调；
+ check: `setImmediate`回调在这里执行；
+ close callbacks: `close`事件有关的回调，例如`socket.on('close', callback)`。

其中最核心的阶段是poll，如果进入poll阶段时poll队列为空，或者poll队列执行到为空，都会在此处等待：

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

3. 下面这个例子说明了`setTimeout`设定的等待间隔并不可靠，NodeJS只保证到时间后timer回调被”尽快“地执行：

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

    这里让`readFile`读取一个大文件，耗时在200ms左右。因此poll阶段等待一段时间后`readFile`完成，其回调进入poll队列，回调也执行完成后才进入下一轮循环执行timers，最终先输出`called`，随后打印的耗时在1200ms左右；极小的情况下，可能由于磁盘缓存原因`readFile`比预期的慢，则poll阶段等待过程中`setTimeout`先于`readFile`完成，先“折返”（wrap back）到timers阶段执行timer回调，然后依次又一次进入poll阶段，这时打印的耗时是300ms多，随后才输出`called`。

#### `process.nextTick()`

NodeJS自己也吐嘈了，它的`nextTick`有立即执行的作用，而`setImmediate`却有下一个“tick”才执行的作用，这都是历史遗留的因素。按照官方的文档，不管当前是事件循环的哪一个阶段，nextTickQueue会在当前“操作”完成后立即处理，这里“操作”是来自NodeJS底层的概念，我推测代表的就是“宏任务”或者浏览器标准中的Task，一个Task之后清空微任务队列。故nextTickQueue作为微任务队列总是在进入下一个阶段前处理完，因此如果递归`nextTick`的话可能因为迟迟进入不了poll阶段造成I/O“饥饿”，和上面`infinite`的效果类似。

作为微任务，`process.nextTick()`优先级甚至比`Promise`还高，下面的代码会先输出`2`再输出`1`。

```js
Promise.resolve().then(() => console.log('1'));

process.nextTick(() => console.log('2'));
```

在有了`queueMicrotask`之后，`process.nextTick()`可以被视为不推荐使用了，`queueMicrotask`创建的是和`Promise`同等优先级的微任务，便于梳理程序执行流。

特别一提，现在还能找到一些文章说NodeJS的事件循环和浏览器的事件循环不同，NodeJS会在每个阶段之间才检查微任务队列，而非在一个宏任务之后就有一个微任务检查点，这是NodeJS@11版本之前的行为，早改了。

## 二进制数据的处理

[ZSBD](https://www.everseenflash.com/CS/Frontend/Binary.md)

## hash和history路由

hash路由的原理是浏览器在监听到`hashchange`事件之后并不会刷新页面，这个功能本来是给页内锚点用的，所以使用hash路由的问题会导致没法做锚点了，如果还需要类似锚点功能的话可以自己`scrollIntoView()`，同时hash路由传参也不方便，需要序列化为URL字符串，还有长度限制。

history路由的原理在于history的几个API可以改变浏览器路由栈却不刷新页面，其中`pushState`和`replaceState`还可以改变页面URL，再配合`popstate`事件，可以实现路由效果。比较奇怪的是`pushState`和`replaceState`默认不会触发`popstate`事件，需要我们自己调度。

Vue3两种模式的`vue-router`简要实现如下：

```js
export const RouterView = defineComponent({
  render() {
    return h(this.$router.current.value?.component);
  },
});

class BaseRouter {
  routes: Route[];

  current: ShallowRef<Route | undefined>;

  constructor(routes: Route[]) {
    this.routes = routes;
    this.current = shallowRef();
    window.addEventListener('load', this.navigate.bind(this));
  }

  navigate() {
    throw new Error('Method not implemented.');
  }

  install(app: App) {
    app.config.globalProperties.$router = this;
    app.component('RouterView', RouterView);
  }
}

export class HashRouter extends BaseRouter {
  constructor(routes: Route[]) {
    super(routes);
    window.addEventListener('hashchange', this.navigate.bind(this));
  }

  navigate(): void {
    const hash = window.location.hash.slice(1).replace(/^$/, '/');
    const route = this.routes.find((r) => r.path === hash);

    if (!route) throw new Error(`Route not found: ${hash}`);

    this.current.value = route;
  }
}

export class HistoryRouter extends BaseRouter {
  constructor(routes: Route[]) {
    super(routes);
    window.addEventListener('popstate', this.navigate.bind(this));
  }

  navigate(): void {
    const path = window.location.pathname.replace(/^$/, '/');
    const route = this.routes.find((r) => r.path === path);

    if (!route) throw new Error(`Route not found: ${path}`);

    this.current.value = route;
  }

  push(path: string): void {
    window.history.pushState({ path }, '', path);
    window.dispatchEvent(new PopStateEvent('popstate'));
  }
}
```

## 同源策略与CORS

跨域问题也是前端开发中常见的问题了。搞清楚这个首先得了解浏览器的同源策略。

两个网址只有在协议、端口和主机字段相同的时候才被当作是同源的，因此HTTP和HTTPS被认为是不同源，二级域名不同也被认为是不同源。出于安全考虑，在不同源的情况下，Cookie、localStorage和IndexDB无法读取，AJAX请求也不能发送，DOM无法获得（常出现于`<iframe>`）。

很多时候需要绕过这些限制，AJAX请求通常采用下面会介绍的CORS方法，因此这里简要介绍下另外两类问题跨源时绕过的方式：

1. Cookie：对于一级域名相同，二级域名不同的网页，可以通过手动设置`document.domain`的方式共享Cookie；另一种方法是服务器在设置Cookie的时候指定Cookie所属的域名；

2. iframe（囊括了localStorage/IndexDB）：存在一些技巧性的方法，但系统的解决还是推荐`window.postMessage`。

CORS，因为O代表的是“origin”所以它更应该被称为“跨源资源共享”，是一种通过额外的[HTTP响应头](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#the_http_response_headers)，让服务器有能力标示除了自身以外的其它源（域、协议或端口），浏览器看到这些HTTP头会放开默认的同源策略。

日常中其实我更青睐用Nginx做代理的方式解决跨域问题，通过Nginx访问把原本跨域的请求变成同源请求的方式。更进一步，还可以利用http-proxy-middleware之类的库自己搭个简易代理服务器，这样自由调整代理逻辑，比起编写Nginx糟糕的DSL更轻松一点，可能性也更多。我常做的一件事就是利用Git把这些经常变更的nginx.conf或者代理服务器脚本管理起来，随用随分支切换。

下面是一个真实的例子，客户端应用部署在`http://localhost:3001`，服务端应用部署在`http://localhost:5000`，由于端口不同，在客户端应用里`fetch('http://localhost:5000')`将导致跨域：

```js
fetch('http://localhost:5000'); // CORS error
```

我们做个简易的代理服务器，部署在`8080`端口，`/`代理的是原客户端应用，`/api`代理的是原服务端应用，转发的时候重写路径去掉`/api`：

```js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

app.use(
  '/',
  createProxyMiddleware({
    target: 'http://127.0.0.1:3001',
  })
);

app.use(
  '/api',
  createProxyMiddleware({
    target: 'http://127.0.0.1:5000',
    pathRewrite: { '^/api': '' }
  })
);

app.listen(8080);
```

现在统一通过代理服务器访问客户端应用和服务端应用就没有跨域问题了：

```js
fetch('http://localhost:8080/api'); // 200 OK
```

CORS还有个话题是浏览器在发起CORS请求的时候，会将请求区分为简单请求和非简单请求，简单请求的请求方法只能是HEAD、GET、POST且使用的HTTP头有限，不满足这些条件的都是非简单请求。简单请求在发起时会带上Web应用所在的Origin，而非简单请求在发起前会先发送一个“预检请求”，用的是OPTIONS，这当然是一次额外的开销，只是在被拒绝的情况下相比起直接发非简单请求“动静要小”。~这个预检请求让我联想到无线局域网802.11的信道预约请求。~

## XSS、注入和CSRF

XSS分三种，存储型、反射型和DOM型。前两种差别不大，基本都有网页存在注入漏洞让不法分子上传恶意脚本到服务器端的前提，在用户输入二次确认的场景尤其容易出现。DOM型则不涉及服务端，是不法分子拦截了发送到用户客户端的静态资源，然后修改或嵌入了一些错误信息诱使用户点击跳转至恶意网站、或者窃取用户信息的行为。

关于注入我想说的是不要试图用正则表达式匹配的方式去检测用户输入，有一万种代码混淆的方式可以绕过这些正则表达式。合理的做法是始终对用户输入做转义，例如`encodeURIComponent`，如果非要执行用户脚本不可的话，尽量在沙箱环境中执行。

CSRF即“我成替身了”，不法分子通过社工等手段窃取用户的安全令牌、Cookie等身份验证信息，在用户会话还存活（令牌/Cookie还有效）的情况下，伪装成用户进行操作。

## CSP

## Web Component

## 移动端兼容性问题

### `1px`边框

在移动端比如iOS设备默认的设备像素比通常是2，这意味着如果我们设置`border`宽度为`1px`的话看起来要比设计稿上面要粗一点，但如果设置成`0.5px`的话在一些老旧机型上又不支持，常见的解决方案有`border-image`模拟、`box-shadow`模拟、`transform: scale(0.5)`缩放等。

### 键盘

移动端键盘主要解决的问题是如何确保其顶起页面，不会遮挡住处在页面下方的输入框，以及是否会使输入框失去焦点的问题。细分为原生键盘和前端H5键盘两类。

1. 原生键盘：各个系统自带键盘的兼容性问题非常严重，也没有个统一的标准获取键盘弹起之后的时间、高度等信息。所以如果是App内，最佳的解决方案是由原生实现，并提供一些API给前端调用。如果非要在前端考虑各种边界情况的话，可参考[这篇文章](https://segmentfault.com/a/1190000018959389)。

2. H5键盘：前端实现的键盘就比较简单了，可以拿到键盘的实际高度去推高页面。

### 滚动穿透

### `click` 300ms

移动端点击延迟300ms的原因最早是浏览器需要判断一下用户想做的是双击缩放还是单纯的点击。主流解决方案有三，一是著名的FastClick，原理是自定义事件并阻止默认的`click`事件，二是直接设置`<meta>`禁用缩放，三是使用`touch`事件或`pointer`去处理。

### iPhoneX刘海适配

在标题头组件里针对iPhoneX机型做一下适配即可。

### audio autoplay

这个倒不是BUG，而是浏览器为了避免给用户造成困扰的默认行为，audio、video和fullscreen API都有类似不能自动进行的机制。用户必须亲自操作过页面之后才能够自动播放音频，代码模拟`button.click()`都是不行的。一般和业务与组件使用方沟通好，将需要自动播放的场景隐藏在用户需要操作页面之后的某个阶段里。

### 白屏

我知道的白屏问题分为两种，不算SPA加载慢导致的白屏耗时，一类是关键代码出错框架没法正确挂载应用导致的白屏，一般是因为浏览器版本比较老旧不支持新语法导致的，这类问题通常借助构建工具和polyfill手段将代码输出为兼容性更好的版本，条件允许的话还可以做按需加载polyfill；第二类我仅在微信H5页面遇到过，通过抓包分析推测是微信缓存了旧的HTML，一个相当SB的行为，HTML里面的`script`资源链接过时导致的，这个主要通过在服务端设置HTTP缓存头，告知浏览器不要缓存HTML来解决。
