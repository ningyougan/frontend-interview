# Javascript

## 原型继承

原型继承是与虚函数表等不同的一种继承实现方式，通常出现在抽象层次较高的解释型语言中，我知道的如Javascript、Lua等。这种继承机制的特点是“链式”和对象代理。不管是继承还是组合，主要目的都在于代码复用。我们的需求是“当子类覆盖（override）了父类方法时，用子类的，没有时就用父类的，父类也没有就找父类的父类……”层层向上形成一条链路，即原型链。而我们对对象成员的访问大多数时候都可以归结为对`getter`和`setter`方法的调用，因此只要语言提供了模拟或魔改`getter`和`setter`的机制，就可以实现原型链继承。

最典型的实现方式或许是对象代理，可以在代理这一层做各种操作，改变对象访问的行为。在Javascript中ES6提供了`Proxy`可以创建对象代理，可操纵的对象访问行为中就包括`get`和`set`。那么我们要实现简陋的继承就很简单了，说到底创建个对象链表就是。值得留意的是原型模式“写时复制”的优点，实例其实是个空对象，访问实例上的属性，实际是通过代理访问的原型对象，但如果要基于这些属性进行修改时，`set`操作的是实例对象，之后再次访问相同属性就来自于实例了，用一段代码表达：

```js
proto = { foo: 42 }     // 原型
o = {}                  // 实例
self = new Proxy(o, {   // 代理
  get(_, prop) {
    return o[prop] !== undefined 
      ? o[prop]
      : proto[prop];
  }
})

console.log(self.foo)   // 42，访问的属性来自原型
console.log(o)          // {}，实例依然是个空对象
self.foo = self.foo - 18; // 等号右边对foo的访问调用了get，等号左边对foo的访问调用了set，set修改的是o
console.log(o)          // { foo: 24 }
console.log(self.foo)   // 24
console.log(proto.foo)  // 42，原型没有改变
```

有了上述准备，暂时忘掉Javascript的`class`语法糖，不妨假设要用`create`方法创建实例，用`derive`方法创建派生类。为了与Javascript原生关键字区分，用`self`模拟`this`。先写出我们的用例来指导需求的开发：

```js
const Base = {
  foo: 42,
  bar(self) {
    console.log(self.foo);
  }
};

// 模拟extend
const Derived = derive(Base, {
  // 模拟override
  bar(self) {
    self.foo = 24;       // this.foo = 24;
    Base.bar(self);      // super.bar()
  }
});

const Derived2 = derive(Derived);

// 模拟new
const a = create(Base);
const b = create(Derived);
const c = create(Derived2);

a.bar(); // 42
b.bar(); // 24
c.bar(); // 24
```

实现`create`，除了给实例方法手工绑定`self`指针之外没什么特别的：

```js
function create(base) {
  const instance = {};

  const self = new Proxy(instance, {
    get(_, prop) {
      const value = instance[prop] !== undefined
        ? instance[prop]
        : base[prop];

      return typeof value === 'function'
        ? value.bind(null, self)
        : value;
    },
  });

  return self;
}
```

再实现`derive`，就是个对象合并，浅拷贝，合并的时候优先用派生类定义的属性：

```js
function derive(base, override) {
  const derived = Object.create(base);

  for (const prop in override) {
    derived[prop] = override[prop];
  }

  return derived;
}
```

现实中Javascript的原型机制由于有实例属性和静态属性的差别要复杂一点。首先类对象要提供给实例的属性并没有直接放在类对象上，而是单独开辟了一个字段`prototype`，实例则通过`__proto__`访问这些属性。实现派生时，再建立一条查询链，让`Derived.prototype`往上能查询到`Base.prototype`。因此这里其实存在两条链路，一条在实例与类之间，一条在派生类与基类之间，用代码表达：

```js
Base = { prototype: { foo: 42 } } // 基类

o = {}
Derived = { // 派生类
  prototype: new Proxy(o, {
    get(_, prop) {
      // 从派生类到基类
      return o[prop] !== undefined
        ? o[prop]
        : Base.prototype[prop]
    }
  })
}

x = { __proto__: Derived.prototype }
instance = new Proxy(x, { // 实例（代理）
  get(_, prop) {
    // 从实例到实例的__proto__，而实例的__proto__就是类的prototype
    return x[prop] !== undefined
      ? x[prop]
      : x.__proto__[prop];
  }
})

console.log(instance.foo) // 42
instance.foo = 24
console.log(instance.foo) // 24
console.log(Base.prototype.foo) // 42
```

此外`__proto__`在Javascript中已经是一个标记为废弃的属性，新的API是`Object.getPrototypeOf`和`Object.setPrototypeOf`。

这种基于对象代理和原型链的继承可以进一步引申，例如在代理这一层做更细粒度的控制以实现只读属性、私有属性等。甚至“对象”这个语义都不是必须的，用`getter`和`setter`函数表达即可：

```js
makeObject = (predecessor) => (prop, value) => (value !== undefined  // 根据value是否为undefined判断是set还是get
  ? makeObject((p) => (p === prop ? value : predecessor(p)))         // set其实创建了一个新的“对象”
  : predecessor(prop));

get = (obj, prop) => obj(prop);
set = (obj, prop, value) => obj(prop, value);

// proto = { foo: 42 }
proto = makeObject(
  (prop) => (prop === 'foo' ? 42 : undefined),
);

// o = {}
o = makeObject((prop) => undefined);

// self = new Proxy(o, {/*...*/})
self = makeObject((prop) => (get(o, prop) !== undefined
  ? get(o, prop)
  : get(proto, prop)));

console.log(get(self, 'foo')); // 42
self = set(self, 'foo', get(self, 'foo') - 18); // self.foo = self.foo - 18
console.log(get(self, 'foo')); // 24
console.log(get(proto, 'foo')); // 42
```

## 模块机制

### CJS

> 有关NodeJS模块的更多内容，例如模块缓存、`babel-register`等各种Registers的实现等，归类到[NodeJS这里](./NodeJS.md#Hf2e50ce16a2a53e9)了。

CJS即CommonJS，主要应用于NodeJS平台，API是`module.exports`和`require`：

+ `export.js`

    ```js
    module.exports = {
      foo: 42,
      bar() {
        console.log(this.foo);
      }
    }
    ```

    也可以使用如下写法，此时`bar`中的`this`指针指代整个`module.exports`对象：

    ```js
    module.exports.foo = 42;
    module.exports.bar = () => {
      console.log(this.foo);
    }
    ```

+ `import.js`

    ```js
    const m = require('./export.js');

    console.log(m.foo); // 42
    m.bar(); // 42
    ```

    利用JS的对象展开设置导入别名：

    ```js
    const { foo: alias, bar } = require('./export.js');

    console.log(alias);
    bar();
    ```

#### 关于`exports`

`exports`多数时候都可以当作是`module.exports`的别名，只有完全将另一个值赋给它的时候才解除对`module.exports`的引用。印象中我几乎没有遇到过要用`exports`的地方。

```js
module.exports.hello = true; // Exported from require of module
exports = { hello: false };  // Not exported, only available in the module
```

### ESM

ESM即ES Module，以`import`和`export`关键字为代表，常见于TS、Deno等平台。目前已逐渐成为标准和主流，v14+的NodeJS也原生支持ESM模块，只是听说各个平台的支持之间还有一段相互背刺的历史，使两种格式相互迁移的时候存在很多[限制](https://nodejs.org/docs/latest-v14.x/api/esm.html#esm_interoperability_with_commonjs)。

+ `export.js`

    ```js
    const foo = 42;
    const bar = () => console.log(foo);

    export {
      foo,
      bar
    }
    ```

    或者：

    ```js
    export const foo = 42;
    export const bar = () => console.log(foo);
    ```

+ `import.js`

    ```js
    import {foo, bar} from './export.js';

    console.log(foo);
    bar();
    ```

#### 关于`default`

上面的例子如果直接`import m from './export.js'`会发现`m`是`undefined`，因为此时取的是模块的默认导出，对应`export default`，理解成`module.exports.default`就可以：

+ `export.js`：

    ```js
    const foo = 42;
    const bar = () => console.log(foo);

    export {
      foo,
      bar
    }

    export default 24;
    ```

+ `import.js`

    ```js
    import anyname, { foo, bar } from './export.js';

    console.log(anyname); // 24
    console.log(foo); // 42
    bar(); // 42
    ```

能不能像`require`那样用一个变量取得模块的所有导出呢？当然能，使用`import * as`导入：

+ `import.js`

    ```js
    import * as m from './export.js';

    console.log(m); // { foo: 42, bar: [Function: bar], default: 24 }
    ```

### IIFE

IIFE即“立即执行函数”，本身不是模块机制，但联系紧密，也历史悠久。如果非要解释它是什么，用代码表达比较好：`(() => {})()`。按照我的理解，IIFE有两大好处：

1. 将模块代码限制在函数作用域内，合理规避了[变量提升机制](#H4a40d4b2def28e4f)可能造成的作用域混乱及污染全局空间等问题；
2. 方便复用，打个比方，某段代码要同时在Web和NodeJS平台使用，其中在Web平台下要用到`window`对象，在NodeJS平台下是[jsdom](https://github.com/jsdom/jsdom)模拟的`window`，此时可以通过参数注入的方式复用代码，再往深了说是依赖倒置和控制反转，下面的UMD格式就是一个应用。

### UMD

除了上述四类之外，还有System、AMD等我没怎么用过的模块格式，不多赘述。在ESM普及之前，要处理好这些令人眼花缭乱的格式，制作一个在各平台运作良好的模块挺麻烦的，因此有了UMD格式的出现。理解了IIFE再来理解UMD并不难，上面的例子用Rollup打包成UMD格式之后如下，我添加了一些注释：

```js
(function (global, factory) {
  // NodeJS
  typeof exports === 'object' && typeof module !== 'undefined' ? factory(exports) :
  // AMD
  typeof define === 'function' && define.amd ? define(['exports'], factory) :
  // Web，myBundle是构建umd时传入的名称，在Web环境执行后，global.myBundle变量就是加载的模块对象
  (global = typeof globalThis !== 'undefined' ? globalThis : global || self, factory(global.myBundle = {}));
})(this, (function (exports) { 'use strict';

  const foo = 42;
  const bar = () => console.log(foo);

  var main = 24;

  exports.bar = bar;
  exports.default = main;
  exports.foo = foo;

  Object.defineProperty(exports, '__esModule', { value: true });

}));
```

### 循环依赖

模块加载可以说就是把模块文件执行一遍的过程，下面`a.js`和`b.js`的定义显然有循环依赖，如果执行`a.js`将会在`b.js`打印`foo`处报错，借助变量提升机制把`const`改成`var`不会有报错，但打印的是`undefined`。

+ a.js

    ```js
    import { bar } from './b.js';

    console.log(bar);

    export const foo = 42;
    ```

+ b.js

    ```js
    import { foo } from './a.js';

    console.log(foo);

    export const bar = 24;
    ```

如果改用成等价的CJS格式，则不会报错，但效果和变量提升类似也会在`b.js`打印`foo`处输出`undefeind`，这是因为NodeJS为了避免无限循环，会将`a.js`的一个“半成品”提供给`b.js`，其中的`foo`还未定义。

但实际项目中循环依赖很容易发生，故需要控制好模块加载的时序，比如将`b.js`改为如下这样：

```js
import { foo } from './a.js';

import("./a.js").then(({foo}) => {
  console.log(foo);
});

export const bar = 24;
```

执行`a`时遇到对`b`的导入，于是去执行`b`，由于`b`中对`a`的导入是异步的，`b`可以成功加载完成，随后`a`加载完成，再执行`import()`创建的微任务且因为`a`的[模块缓存](https://nodejs.org/docs/latest-v14.x/api/modules.html#modules_caching)而不需要重新导入`a`。

### 动态加载

`require`支持动态拼接模块名，要实现批量加载可以考虑`glob`、`globby`之类的库。在有Polyfill辅助情况下，或者现代Web平台上，`import`也可以作为函数使用`import(filepath)`，返回的是一个`Promise`，加载成功则提供一个模块对象。`import()`是实现[模块分割及懒加载](./PerformanceOptimization.md#Hb077d6089e59b828)的基础。但是`import()`自身不是ESM模块规范的一部分，而是另一个[独立的提案](https://github.com/tc39/proposal-dynamic-import)，实际上由于ESM模块[“静态模块结构”](https://exploringjs.com/es6/ch_modules.html#static-module-structure)的特点，ESM的导入导出只能出现在Top Level，且不能含有动态的部分，这意味着我们可以在编译阶段分析模块的导入导出，这是实现[Tree Shaking](./PerformanceOptimization.md#H38b4ccab34e94afd)的前提。

### 相互转化

实践中经常需要将这些模块格式相互转化，通常是CJS和ESM之间，以及它们到其他格式的转换。可通过构建工具进行，当前常用的是Webpack和Rollup。在Webpack中，关键参数是[output.library](https://webpack.js.org/configuration/output/#outputlibrary)，在Rollup中类似，是[output.format](https://rollupjs.org/guide/en/#outputformat)参数，TSC也有[类似参数](https://www.typescriptlang.org/tsconfig/#module)。

## 箭头函数

如前文所述，JS中的`class`不过是语法糖，在`function`函数体里面使用`this`，可以把这个函数理解成一个构造函数，`this`指向的是构造的实例对象：

```js
function Demo() {
  this.foo = 42;
}

d = new Demo();

console.log(d.foo); // 42

// 等价于：

class Demo2 {
  constructor() {
    this.foo = 42
  }
}

d = new Demo2();

console.log(d.foo); // 42
```

箭头函数并没有这样一个隐含的`this`，它只能捕获外部作用域存在的变量。特别是对象的成员函数，在使用`function`定义的函数体中，与其他语言的OOP风格保持一致，其`this`指向该对象本身：
```js
obj = {
  foo: 42,
  bar: function() { // bar() {}
    console.log(this.foo);
  }
}

obj.bar(); // 42
// 若把obj.bar当构造函数，this指向新建的实例，和obj没有关系
new obj.bar(); // undefined
```

```js
obj = {
  foo: 42,
  bar: () => {
    console.log(this.foo); // this捕获的是外部作用域里的this变量
  }
}

obj.bar(); // undefined

this.foo = 24;

obj.bar(); // 24
// 箭头函数无法作为构造函数
new obj.bar(); // TypeError: obj.bar is not a constructor
```

## `call`、`apply`和`bind`

三者都是`Function.prototype`上的API，因此所有普通的函数都可以使用这三个方法。先说`bind`，`bind`用于创建偏函数，偏函数是来自函数式编程的一个名词，我理解为预提供部分参数创建新函数，例如OOP风格中我们可以在对象的成员函数中使用`this`指针，但通常不需要显式地传入`this`，这时每个成员函数都可以看成是带有隐含参数`this`并且`bind`了实例对象的偏函数，前文`self`指针的模拟就是例子。这一点在FP与OOP混用的时候尤为重要，如果我们要将对象的成员函数作为一等公民传来传去，通常都需要手动绑定`this`：

```js
class Demo {
  foo = 42;
  bar() {
    console.log(this.foo);
  }
}

d = new Demo();

function callFn(fn) {
  fn();
}

d.bar(); // 42
callFn(d.bar.bind(d)); // 42
callFn(d.bar); // TypeError: Cannot read properties of undefined (reading 'foo')
```

这里特别吐嘈一下某些前端框架，为了简化使用主动绑定了组件实例作为`this`，结果往往给不知内情的开发者造成了困扰，好心办坏事。

`call`和`apply`与上面代码片段中`callFn`的作用差不多，是除了`operator()`之外其他调用函数的手段，两者的区别在于一个使用可变参数列表传参，一个使用数组传参：

```js
class Demo {
  foo = 42;
  bar(x) {
    console.log(this.foo + x);
  }
}

d = new Demo();

d.bar.call(d, 2); // 44
d.bar.apply(d, [2]); // 44
d.bar.call(null, 2); // TypeError: Cannot read properties of null (reading 'foo')
```

### 不使用`bind`如何实现`call`和`apply`

面试被问到的时候还真有点懵，没想过这个问题……这里的难点是如何给实例的成员函数绑定`this`，看了一下相关的解答才明白，利用对象里成员函数会将该对象作为`this`的特性，将待调用的函数作为传入对象的一个方法再调用就可以了：

```ts
Function.prototype.myCall = function (instance: any, ...args: any[]) {
  const key = Symbol();

  instance[key] = this;
  instance[key](...args);

  delete instance[key];
};
```

## `let`、`const`、`var`和变量提升（hoisting）

`let`和`const`更符合现代编程语言关于变量作用域的理解，而历史遗留产物`var`和`function`则存在称之为“变量提升”的行为，可以在变量定义之前使用之，之后在同一个作用域声明变量即可。用代码表达这种差异：

```js
console.log(x); // ReferenceError: Cannot access 'x' before initialization

let x = 2;
```

```js
console.log(x); // undefined

var x = 2;

bar(); // 2

function bar() {
  console.log(x);
}
```

如果非说变量提升行为有什么好处的话，我能想到的是使用`function`定义函数非常符合我们的思维模式，假如一个任务分为A、B、C三个步骤，可以顺着思路写下来：

```js
function task() {
  A();
  B();
  C();
}
function A() {}
function B() {}
function C() {}
```

其他情况我们都应该使用`let`和`const`，毕竟都什么年代了还在使用传统`var`定义变量，多少有点说不过去。根据我自己的实践，也几乎没有要用到变量提升特性的地方，如果有往往是代码组织的不好。真有需要，比如兼容性问题，不支持`let`和`const`，就交给Babel之类的转译工具处理。

## `==`和`===`

同样是JS的历史遗留问题，`[] == false`、`[] == ''`、`null == undefined`等很容易造成隐形BUG。目前我也没有遇到必须使用`==`的地方，一律使用`===`，记不住就用ESLint约束自己，~养成习惯之后使用其他语言就很别扭了~。

## `"use strict"`

使作用域内的代码进入“严格模式”，可以避免很多JS的历史遗留问题，比如前面的变量提升，在`x`未声明时使用却不会报错等等。现在前端开发由于有着各种构建工具的辅助，我们已经很少需要自己写`"use strict"`了。

## 类型判断

在运行时判断对象的类型是个很常见的需求，在JS中基础API有`typeof`、`instanceof`、`Array.isArray`等，更多可以参考[Lodash](https://lodash.com/docs/4.17.15#isArguments)的实现。

## `0.1 + 0.2 !== 0.3`

浮点数精度问题，`0.1`和`0.2`在转为二进制浮点数的时候都会发生舍入。以IEEE 32位浮点数标准为例，阶码8位，尾数23位，处理的时候为保持精度留两位附加位，把`0.1`不断乘2取整数部分得到尾数`00 0110 0110 0110 ...`，为了让小数点左边有一个隐含的1，需要左移4位也就是阶码为-4，于是尾数部分还剩下`10 0110 0110 0110 0110 0110 011`，阶码是移码表示的，需加上偏移量127得到123即`0111 1011`，故`0.1`临时表示为`0 | 0111 1011 | 10 0110 0110 0110 0110 0110 011`，同理可得`0.2`临时表示为`0 | 0111 1100 | 10 0110 0110 0110 0110 0110 011`。

两者相加，先对阶，小阶向大阶看齐（大阶向小阶看齐的话可能会丢失有效数字），于是`0.1`的尾数部分右移1位，阶码增加1，右移时别忘了小数点左边隐含的1，此时尾数为`110 0110 0110 0110 0110 0110 01`，与0.2的尾数相加，中途尾数进位会发生一次右移，于是阶码为`0111 1101`，尾数结果需舍入，按照最近舍入方式，结果为`0 0110 0110 0110 0110 0110 10`，整个数是`3E99999A`对应于`0.30000001192092896`，~刚好与0.3的浮点数表示相等，这是32位的情况，总感觉哪里算错了，等我把64位的也算下~。

## Map和WeakMap

`WeakMap`最大的特点是只能用对象作为键，且作为键的对象如果引用计数为0，会被自动GC。

## 正则表达式

想了一下JS的正则表达式好像没什么值得说的。支持反向引用、具名分组和环视，常用API有`RegExp.prototype.test`、`RegExp.prototype.exec`、`String.prototype.match`和`String.prototype.replace`等。

在应用`g`或`y`标志的情况下，JS的正则表达式对象是有状态的，这是个易错点，具体表现为这类正则表达式对象会记住上次匹配的位置（`r.lastIndex`），每次匹配从这个位置开始，直到越界才会重置为0。_global_ 和 _sticky_ 的区别是 _sticky_ 如果`lastIndex`位置处匹配失败，将直接返回`null`，而 _global_ 会从下一个字符继续尝试，在MDN上有更详细的解释：

+ 使用 _global_：

    ```js
    r = /a/g;
    s = "aba";

    console.log(r.exec(s).index); // 0
    console.log(r.exec(s).index); // 2
    ```

+ 使用 _sticky_：

    ```js
    r = /a/y;
    s = "aba";

    console.log(r.exec(s).index); // 0
    console.log(r.exec(s).index); // TypeError: Cannot read properties of null (reading 'index')
    ```

## 函数式编程

这个话题太大了，如果只是从前端开发者的角度，了解一下函数作为一等公民、纯函数、副作用、惰性求值等概念的话，在Github上以`FP`为关键字搜索有几个仓库值得一读。但如果要更深刻的理解，将涉足PL领域，以及群论范畴论拓扑学等听起来就很高大上的东西，像我这种半吊子本科水平是不敢过多置喙的。

以我当前的认知，我们通常学到的计算机起源是图灵等人这一脉，核心是一个非常现实的、可以在物理层面造出来的图灵机，还有一脉则来自于丘奇等人，核心是非常抽象的、完全从逻辑层面出发的Lambda演算，两个计算模型的能力已经被证明是等价的（还有一些其他图灵完备的计算模型如标签系统、生命游戏、SKI组合子演算等）。从图灵机衍生出了我们在计组课上学到的，非常机械的制造计算机的方式，而Lambda演算用函数表达计算，α变换即作用域内参数重命名，β归约即参数代入，在此基础上衍生出了Lisp、OCaml、ML、Haskell等编程语言，这些语言相对冷门但在PL领域教学研究却很常见，我用得比较多的是Lisp之Scheme之方言Racket，学得越多对编程语言的理解越深入，同时也越觉得自己无知。从最早接触到Y组合子的惊为天人，到读《SICP》、《EOPL》时初窥门径，再到面对PL领域论文中各种千奇百怪的符号时头皮发麻，认识到学无止境也是件蛮痛苦的事。
