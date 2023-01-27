# NodeJS

## 模块机制

### 缓存

NodeJS的导入的模块默认都会被缓存，因此一个模块多次导入得到的是同一个模块对象。

```js
const d = require('./d'); 
const d2 = require('./d'); 

console.log(d == d2); // true
```

通过调整`require.cache`，可以控制缓存刷新，但简单地清除一个模块的缓存有很多注意点，例如在Windows上文件名是不区分大小写的，而NodeJS以文件名标识模块，仅清除`d.js`并不会导致`D.js`模块缓存被清理：

+ d.js

    ```js
    console.log(`${require.main.filename} loading ${__filename}`);
    ```

+ c.js

    ```js
    require('./d.js'); // c.js loading d.js
    require('./D.js'); // c.js loading D.js

    delete require.cache['d.js'];

    require('./D.js'); // no effect
    require('./d.js'); // c.js loading d.js
    ```

> 在v14+的NodeJS中，可以通过`node:`前缀跳过缓存直接访问到NodeJS原生模块。Mock原生模块的时候可能会用到。

再如，多进程架构中，往往清理一个进程的模块缓存还不够，要同步使其他进程的缓存失效，这个问题与清除多核CPU缓存面临的问题是一样的：

+ c.js

    ```js
    const cp = require('child_process');

    require('./d'); // c.js loading d.js

    cp.fork('a.js', { stdio: 'inherit' });

    require('./d'); // no effect
    ```

+ a.js

    ```js
    require('./d'); // a.js loading d.js

    delete require.cache['d.js']
    ```

### 钩子函数

在NodeJS生态经常看到各种各样的Registers可以控制NodeJS加载模块的行为，是通过修改`require.extensions`来指示NodeJS如何处理某类后缀的文件实现的，社区也有[pirates](https://github.com/danez/pirates)这样的工具库，修改的是`Module._extensions`，其实是等价的：

```js
const m = require('module');

console.log(require.extensions === m._extensions, require.extensions === m.Module._extensions); // true true
```

例如，要改动NodeJS加载`.js`文件时的默认动作，并支持加载`.txt`文件：

```js
const fs = require('fs');
const oldJSLoader = require.extensions['.js'];

require.extensions['.js'] = function(module, filename) {
  console.log(`loading ${filename}`);
  oldJSLoader(module, filename); // fallback
};
require.extensions['.txt'] = function(module, filename) {
  console.log(fs.readFileSync(filename, 'utf8'));
}

require('./d.js'); // 空文件
require('./e.txt'); // 文件内容为 hello world
```


```
loading d.js
hello world
```

不过NodeJS长期将这种机制标记为废弃，并明确指出会降低性能或造成微妙的BUG。在ESM模块中，NodeJS又提供了一套实验性的[Loader API](https://nodejs.org/docs/latest-v14.x/api/esm.html#esm_loaders)，我还没有用过。

## `spawn`、`fork`和`exec`

`spawn`用来创建一个子进程，而`fork`和`exec`可以当作在`spawn`上的二次封装，其中`exec`会创建一个新的shell并在其中执行命令，而`fork`会创建一个新的NodeJS进程并建立一条父子进程之间的IPC信道，创建的子进程是完全独立的，有自己的事件循环、内存空间和v8实例。故`exec`可理解为`spawn('sh')`而`fork`可理解为`spawn('node')`。

虽然名字是`exec`，但`exec`与POSIX系统调用`execve(2)`截然不同，`execve`会替换掉当前进程所执行的程序，包括整个程序的栈、堆和数据段：

```c 
char* args[] = {"", "hello world", NULL};

execve("/bin/echo", args, envp); // hello world
printf("will not print because current process is replaced");
```

而child_process的`exec`只不过是在子进程中运行shell。如果要执行的本身就是个可执行文件，还可以跳过shell这一层，直接使用`execFile`执行，获得少量的性能提升。

虽然名字是`fork`，但`fork`和POSIX系统调用`fork(2)`也截然不同，`fork(2)`会克隆当前进程的内存空间，因此才有关于`fork`的经典问题：

```c
int i;
for (i = 0; i < 2; ++i) {
  fork();
  printf("-");
}
wait(NULL);
```

`i=0`的时候主进程`fork`了一个子进程，子进程的状态也是`i=0`，随后执行`printf`打印两个`-`，接着`i=1`，两个进程各自又克隆了两个子进程，于是打印四个`-`，所以理论上应该输出6次`-`，但实际上执行这段程序通常只有很小的概率能得到6个`-`，更多时候是8个。这是因为标准输出的缓存区也会被克隆，故第一次克隆后缓存区有两个，各放入一个`-`，第二次克隆后缓存区是四个，又放入一个`-`共计8个`-`，只有极少数情况下刚好在第一次`printf`之后，第二次`fork`前发生了缓冲区flush才能得到6个`-`的输出，我们也可以主动在`printf`语句后面加入`fflush(stdout)`来冲刷缓冲区。

所以child_process中的`fork`只是个借名。真说起来，和`fork(2)`行为有点类似的反而是`cluster`，下面代码会输出6个`-`，但是它输出6个`-`的原因和`fork(2)`也不能一概而论，这里只有主进程会调用`fork`，共创建了两个，主进程自己打印了两次`-`，两个子进程各自重新执行此文件也打印了两次，共计6次：

```js
const cluster = require('cluster');

for (let i = 0; i < 2; ++i) {
  if (cluster.isMaster) {
    cluster.fork();
  }
  process.stdout.write("-");
}
```

## child_process、cluster和worker_threads

cluster背后是child_process的`fork`，主要用于服务器多进程架构。我最初看到NodeJS文档上给出的示例代码时非常奇怪，因为NodeJS文档上说它在非Windows平台上采用了常见的“主进程Listen并Accept TCP连接然后RR调度给子进程”的模式，但从代码上来看却好像是每个子进程各自建立了一个服务器，而且还监听同一个端口却不会冲突：

```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  console.log(`Master ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  http.createServer((req, res) => {
    console.log(`Worker ${process.pid} handling`);
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8080);

  console.log(`Worker ${process.pid} started`);
}
```

后面把NodeJS文档及cluster模块的源码反复看了N遍才大致理解cluster如何实现子进程之间共享TCP连接，不得不吐嘈NodeJS内部模块之间各种强耦合，真不知道他们是怎么维护得下去的：在`net`或`dgram`内部能看到`server.listen`或者`socket.bind`都针对cluster的worker环境改写过，会调用`cluster._getServer`函数从主进程“获取”`handle`，该方法定义在cluster模块的子进程代码内，实质是通过进程间通信向主进程发送一个动作为`queryServer`的消息，主进程内处理此消息的方法就叫做`queryServer`，它是真正负责创建`handle`的函数，核心代码如下：

```js
if (schedulingPolicy !== SCHED_RR ||
    message.addressType === 'udp4' ||
    message.addressType === 'udp6') {
  handle = new SharedHandle(key, address, message);
} else {
  handle = new RoundRobinHandle(key, address, message);
}
```

`handle`是服务器资源及调度算法的抽象，`SCHED_RR`为非Windows平台的默认调度策略，因此`handle`实际只存在于主进程中。`RoundRobinHandle`即处理TCP连接的默认模式，而UDP协议及Windows平台则使用`SharedHandle`，这是因为UDP协议是无连接协议，而Windows是平台API自身的问题（可以在注释中看到）。`RoundRobinHandle`关键在于一个`handoff`函数，在主进程监听到`onconnection`事件后会将`handle`发送给挑选的子进程；而采用`SharedHandle`意味着套接字也由主进程创建，但是由子进程自己去获取`handle`并做Accept连接等动作。这变相相当于把调度权交给了操作系统的进程调度，按照NodeJS文档的[说法](https://nodejs.org/docs/latest-v14.x/api/cluster.html#cluster_how_it_works)，性能可能还不如RR模式。

worker_threads我将其看成Web Worker在NodeJS中的等价物，按照文档的说法，worker_threads适用于CPU密集型任务，对IO密集型任务的提升并不比NodeJS其他基于libuv的异步API大。翻看worker_threads的源码能看到worker_threads的和`spawn('node')`非常相似，工作线程也有自己的v8实例和事件循环，只是创建工作线程比起创建进程来说要更轻量一点。不过频繁、大量创建线程的开销也是不可接受的，实践中往往需要线程池。

+ [worker_threads相关代码](https://github.com/nodejs/node/blob/main/src/node_worker.cc)：

    ```c++
    // libuv eventloop
    int ret = uv_loop_init(&loop_);
    if (ret != 0) {
      char err_buf[128];
      uv_err_name_r(ret, err_buf, sizeof(err_buf));
      w->Exit(ExitCode::kGenericUserError, "ERR_WORKER_INIT_FAILED", err_buf);
      return;
    }

    uv_loop_configure(&loop_, UV_METRICS_IDLE_TIME);

    // ...

    // v8 isolate
    Isolate* isolate =
        NewIsolate(&params, &loop_, w->platform_, w->snapshot_data());
    if (isolate == nullptr) {
      w->Exit(ExitCode::kGenericUserError,
              "ERR_WORKER_INIT_FAILED",
              "Failed to create new Isolate");
      return;
    }

    // ...

    w_->isolate_ = isolate;
    ```

worker_threads与child_process和cluster的区别是主线程和工作线程之间除了消息通信外，还可以共享内存，通过`ArrayBuffer`或者`SharedArrayBuffer`传递数据。

## 守护进程

守护进程是个较普遍的需求，如果在shell里运行程序，一般shell会等待子进程退出才可以交互，很多时候我们希望程序运行在后台，并且即使shell进程退出了也不会影响到该程序的继续运行（这意味着创建的子进程之父不能是shell进程，而应该是其他存活时间更长的进程，在Linux上通常是init进程）。

在Linux中可以使用`nohup`命令创建守护进程，不过作为前端开发者我用得比较多的无疑是`pm2`，要自己实现的话可参考POSIX API`daemon(3)`。

## stream

“流”这个名词出现太多次了，以至于如果问我到底什么是“流”我觉得我答不上来。我倾向于把“流”当作是一种“抽象”，当出于性能或其他考虑，不是对事物的整体进行处理，而是一次处理一点点的时候，流这个概念就应运而生了。它可以是利用有限缓冲区处理大文件的文件流、亦或是在迭代器与生成器抽象上实现的生产者、甚至是《SICP》中那种利用惰性求值实现的工厂函数。NodeJS中的流以第一类字节流居多，出于本身的高度抽象有好用的一面，但我特别想吐嘈的是stream模块糟糕的API设计，一会儿`read`一会儿`_read`一会儿`push`一会儿`unshift`的，让人眼花缭乱。

steam模块提供了三种流的抽象：可读流`Readable`、可写流`Writable`以及双工流`Duplex`，还有一种转换流`Transform`建立在`Duplex`的抽象之上。

`Writable`流比较值得一提的是`drain`事件，该事件在流遇到“背压”（backpressure，由于内部缓冲区达到预设容量`highWaterMark`）失去流动性，之后又变得可用时触发，因此常被用于避免在背压期间写入流可能导致的内存问题（在背压期间试图写入流的数据并不会丢失，而是会被缓存起来，造成大量的内存浪费及GC问题）：

```js
function write(data, cb) {
  if (!stream.write(data)) {  // 背压
    stream.once('drain', cb); // 恢复流动性之后调用cb
  } else {
    process.nextTick(cb);     // 一轮IO结束
  }
}

// Wait for cb to be called before doing any other write.
write('hello', () => {
  console.log('Write completed, do more writes now.');
});
```

`Readable`流提供了两个模式：`flow`和`paused`，其实就是观察者模式的推模型和拉模型，前者一旦数据就绪就会通过`data`事件提供出去（或者pipe到一个`Writable`），后者需要自己主动调用`read`从流中读取数据。两种模式可以视情况[相互转换](https://nodejs.org/docs/latest-v14.x/api/stream.html#stream_two_reading_modes)。`Readable`也有背压问题，在实现`Readable`时，若调用`push`方法返回`false`，就说明内部缓冲区已满，应该暂停给`Readable`提供数据。

`Duplex`实现了`Writable`和`Readable`的全部接口，其中读写端是相互独立的，包括内部的缓冲区也是独立的，因为读写速度未必一致。而`Transform`则在`Duplex`上再封装了一层，适合用于实现流的过滤和转换等。

### Reactive Stream

响应式流在[RxJS](./Framework.md#H3afa1894d4d39d1e)这里有所介绍。

## 异常处理的模式

NodeJS异常处理通常有三种：`try/catch`，`emitter.on('error')`和`promise.catch`。这里提一嘴异常是因为我想起了以前在别处看来的对动态语言中异常处理模式的批评，我相当赞同的一个观点：像JS这类动态语言开发的项目规模大起来之后，程序中很容易遗漏未处理的异常。根本原因在于动态语言不像Java、C#那样对异常处理有严格的要求，底层的异常会一层层传递上去，每层该兜住哪些会传递哪些职责明确，而动态语言中非常依赖开发者的自觉和对异常捕获体系的良好设计，这么一看国内一些团队力主`error.cause`进入ECMA Script标准还是很有必要的。

## async_hooks

## 调试

我主要通过`node --inspect`或`node --inspect-brk`在Chrome Debugger中调试NodeJS程序，好像没什么特别想说的。

## 内存快照

## CPU profile

## 插件开发

现在流行的趋势似乎是用系统编程语言例如Rust/C++开发WASM插件，也是我觉得性价比很高的一种模式。不过WASM插件比起NodeJS原生支持的`.node`插件来说还存在问题：

1. 性能不足：运行在WASM虚拟机上比起直接运行二进制文件还是有一定性能差距；
2. 功能不足：WASI目前还处于测试阶段，很多编写插件的功能都需要由宿主环境提供。
