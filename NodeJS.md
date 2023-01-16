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

> 在v14+的NodeJS中，可以通过`node:`前缀跳过缓存直接访问到NodeJS原生模块。

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
❯ node a.js
loading d.js
hello world
```

不过NodeJS长期将这种机制标记为废弃，并明确指出会降低性能或造成微妙的BUG。在ESM模块中，NodeJS又提供了一套目前还是测试阶段的[Loader API](https://nodejs.org/docs/latest-v14.x/api/esm.html#esm_loaders)。

## `spawn`、`fork`和`exec`

`spawn`用来创建一个子进程，而`fork`和`exec`可以当作在`spawn`上的二次封装，其中`exec`会创建一个新的shell并在其中执行给出的命令，而`fork`会创建一个新的NodeJS进程并建立一条父子进程之间的IPC信道，创建的子进程是完全独立的，有自己的内存空间和v8实例。故`exec`可当作`spawn('sh')`而`fork`可当作是`spawn('node')`。

虽然名字是`exec`，但`exec`与POSIX系统调用`execve(2)`截然不同，`execve`会替换掉当前进程所执行的程序，包括整个程序的栈、堆和数据段：

```c 
char* args[] = {"", "hello world", NULL};

execve("/bin/echo", args, envp); // hello world
printf("will not print because current process is replaced");
```

而`exec`只不过是在子进程中运行shell。如果要执行的本身就是个可执行文件，还可以跳过shell这一层，直接使用`execFile`执行，获得少量的性能提升。

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

所以`child_process`中的`fork`只是个借名。真说起来，和`fork(2)`行为有点类似的反而是`cluster`，下面代码会输出6个`-`，但是它输出6个`-`的原因和`fork(2)`也不能一概而论，这里只有主进程会调用`fork`，共创建了两个，主进程自己打印了两次`-`，两个子进程各自重新执行此文件也打印了两次，共计6次：

```js
const cluster = require('cluster');

for (let i = 0; i < 2; ++i) {
  if (cluster.isMaster) {
    cluster.fork();
  }
  process.stdout.write("-");
}
```

## child_process、cluster和worker_threads的区别

## 守护进程

## stream

## 异常处理的模式

## 调试

## 内存快照

## CPU profile

## 插件开发

现在流行的趋势似乎是用系统编程语言例如Rust/C++开发WASM插件，也是我觉得性价比很高的一种模式。不过WASM插件比起NodeJS原生支持的`.node`插件来说还存在问题：

1. 性能不足：运行在WASM虚拟机上比起直接运行二进制文件还是有一定性能差距；
2. 功能不足：WASI目前还处于测试阶段，很多编写插件的功能都需要由宿主环境提供。
