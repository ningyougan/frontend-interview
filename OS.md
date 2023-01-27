# 前端与操作系统

## 进程、线程和协程

### 并发和并行

并发（Concurrency）：单核CPU调度执行造成多核假象。

并行（Parallel）：多核CPU同时执行。

### Future和Promise

“基于承诺的异步”是现在普遍使用的编写异步代码的模式，在创建异步任务时用一个数据结构表示异步任务的结果，这个数据结构通常有三种状态：Pending、Fulfilled和Rejected，分别表示任务仍在执行、任务已成功完成且结果已填入该数据结构和任务执行失败。通常任务状态由Pending变为Fulfilled或Rejected之后就不允许改变。

之前写过一些[代码片段](https://www.everseenflash.com/CS/Snippets/GeneratorAutoRun.md)大致表达了我对JS中Promise的理解：`Promise`和`async/await`是Coroutine的语法糖，Coroutine和Channel的抽象有异曲同工的地方，Channel的底层可以想见是信号量、锁、原子操作这些老东西。

#### `join`和`detach`

顺便提一下C++标准库`thread`中的`join`和`detach`，`t.join()`会阻塞主线程直到`t`执行完成并退出，而`t.detach()`则把`t`丢在一边执行（类似守护进程），不会阻塞主线程。如果主线程退出时`t`还未执行完成，`t`的执行会被挂起，其局部资源将被析构，这很可能不是我们所期望的行为。因此使用`detach`经常会借助`future`、`promise`等措施来判断`t`是否已经完成执行。这个例子改编自[cppreference](https://en.cppreference.com/w/cpp/thread/thread/detach)：

```cpp
#include <iostream>
#include <chrono>
#include <thread>

class Demo {
 public:
  ~Demo() {
    std::cout << "Destructing Demo" << std::endl;
  }
};

void independentThread() {
  Demo d;
  std::cout << "Starting concurrent thread." << std::endl;
  std::this_thread::sleep_for(std::chrono::seconds(1));
  std::cout << "Exiting concurrent thread." << std::endl;
}

int main() {
  std::cout << "Starting main thread." << std::endl;
  std::thread t(independentThread);
  t.join();
  std::cout << "Exiting main thread." << std::endl;
}
```

此时输出：

```
Starting main thread.
Starting concurrent thread.
Exiting concurrent thread.
Destructing Demo
Exiting main thread.
```

如果将`t.join()`改成`t.detach()`，大概率只得到输出，在主线程退出时子线程直接被“shutdown”了：

```
Starting main thread.
Exiting main thread.
Starting concurrent thread.
```

很多时候我们不想阻塞主线程（比如创建了很多线程，不太可能挨个`join`），又需要保证主线程退出时所有子线程都执行完成，没完成再等一下它们，这就是`future`和`promise`可以派上用场的地方：

```cpp
void independentThread(std::promise<void>&& p) {
  Demo d;
  std::cout << "Starting concurrent thread." << std::endl;
  std::this_thread::sleep_for(std::chrono::seconds(1));
  std::cout << "Exiting concurrent thread." << std::endl;

  p.set_value_at_thread_exit();
}

int main() {
  std::cout << "Starting main thread." << std::endl;
  std::promise<void> p;
  std::future<void> f = p.get_future();
  std::thread t(independentThread, std::move(p));
  t.detach();
  std::cout << "Exiting main thread." << std::endl;
  f.get(); // will wait until t exit
}
```

将得到输出：

```
Starting main thread.
Starting concurrent thread.
Exiting main thread.
Exiting concurrent thread.
Destructing Demo
```

## 文件系统

### 文件描述符

文件描述符是操作系统对计算机资源的抽象，其根源大概是Unix的“万物皆文件”哲学。文件描述符的出现给上层开发者提供了方便，可以使用一套统一的API处理文件、套接字、管道、终端等各种资源。再比如NodeJS中cluster等模块的IPC看起来好像可以传递TCP连接UDP套接字等，其实都是传递的文件描述符信息然后在另一个进程拿到对应操作系统资源重新建立等价上层实例，毕竟不是什么东西都适合序列化的。

于是，`fs`模块不止可以写入文件系统，也可以往标准输出上写，这毫不奇怪：

```js
const fs = require('fs');

fs.writeSync(process.stdout.fd, 'Hello World\n');
```

### 软链和硬链

首先要理解什么是inode，inode是文件系统的底层抽象，是磁盘上的一些区块，一个文件在底层不过是若干inode，而目录是一类特殊的inode，其中存放着从人类可读路径到inodes的**链接**。文件系统的首要任务便是划分磁盘并管理这些inodes。

硬链接：在目录中添加一项指向相同inodes，因此硬链接文件和原文件在底层是同一个inode编号，大小也相同。文件系统通过引用计数跟踪有多少文件链接到当前inode，只有引用计数为0才会真正释放inode及相关数据结构。所以通常所说的删除操作其实都是unlink操作。由于引用计数的存在，硬链接无法成环，所以不允许创建目录的硬链接，防止目录之间形成循环依赖。并且由于与底层inode实现的耦合，通常无法创建其他文件系统的硬链接。

软链接（符号链接）：建立一个新文件（有自己的inodes），存放若干信息，文件系统处理时会找到原始文件进行处理，比如Windows的快捷方式。因为软链接其实只是一些数据结构，所以它可以成环，并且即使删除了原始文件软链接也依然存在，形成“悬挂引用”。

下面是某次`ls -i`的结果，其中`a`是原始文件，`c`是`ln -s`创建的符号链接，`d`是`ln`创建的硬链接，可以看到`a`和`d`的inode编号都是相同的，和`c`不同，`c`权限列表开头的`l`指明它是一个软链接：

```bash
24943013 .rw-rw-r-- everseen everseen   6 B  Thu Jan 19 07:53:24 2023  a
24943386 lrwxrwxrwx everseen everseen   1 B  Thu Jan 19 07:36:49 2023  c ⇒ a
24943013 .rw-rw-r-- everseen everseen   6 B  Thu Jan 19 07:53:24 2023  d
```

### 内存文件系统

## 终端与TTY
