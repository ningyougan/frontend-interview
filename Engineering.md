# 前端工程化

## Semver版本规范

[Semver](https://semver.org/)是前端社区普遍使用的一套版本迭代规范，一个Semver版本号由三个部分组成：Major主版本号，代表破坏性变更，不能向下兼容；Minor次版本号，代表新功能，能够向下兼容；Patch版本号，代表向下兼容的BUG修复。Release版本号格式为`Major.Minor.Patch`，有时还会在版本号后面加上`-`及由`.`分割的若干标识符，用来表示这是一个unstable的Dev版本，例如`1.0.0-alpha.0`、`2.0.0-x.y.z`，在维护组件时经常用到。还可以在版本号后面加上`+`号并连接一段元信息，如`1.0.0+20230101123456`，`2.0.0-next+sha256.123456fa`，这些元信息在行使版本的功能时会被忽略。

比较版本号时按照Major、Minor、Patch的顺序进行，若Patch后面还有`-`连接的Dev标识符，则该版本优先级要小于无标识的，即`1.0.0-alpha.0`小于`1.0.0`。实践中直接用[semver](https://www.npmjs.com/package/semver)这个库来比较就是了。

Major版本号为`0`时情况稍微特殊点，通常用于表示这是一个试验性的早期版本，规范中其实没有对此时版本号迭代的行为做出建议，也没有向下兼容不兼容的保证。不过根据[semver](https://www.npmjs.com/package/semver)的版本范围规范，通常将左起第一个不为0的版本号作为主版本号。

Semver规范和npm等依赖管理工具结合起来时有很多注意点，其中最重要的是npm的“incremental install”特性，在`package.json`中声明的依赖版本不只是某一个确定的版本，还可以是一个版本范围，在很多情况下，`npm install`总是会试图拉取范围内最新的版本（见下文）。比较好理解的表示版本范围的方式`1.2.*`、`1.2.7 || >= 1.3.0`不用说，特别提一下`^`和`~`两个表示范围的方式，前者称作“Compatible with version”，其实就是保持向下兼容（Major版本号不变）的一个范围，例如`^1.2.0`等价于`>= 1.2.0 < 2.0.0`；后者称作“Approximately equivalent to version”，在有Minor版本号的情况下代表仅Patch版本号能变化的一个范围，例如`~1.2.0`等价于`>= 1.2.0 < 1.3.0`，无Minor版本号的情况下和`^Major`等价。

## `package-lock.json`和`npm ci`

首先理清楚`npm install`的运行逻辑：

1. 检查现有的`node_modules`文件夹，生成依赖树，并制作一份克隆；
2. 获取`package.json`并分析声明的依赖版本信息，比对并更新克隆树，这个过程有两个要求：
    1. 依赖尽可能扁平化，即缺失新增的依赖会尽可能靠近树的上层，对应`node_modules`根目录；
    2. 不能破坏其他已有模块的依赖关系；
3. 比对两棵树，生成相应的安装、修改等动作并执行。

`package-lock.json`（也包括`yarn.lock`等）的出现是为了解决“相同的`package.json`安装后却可能得到不同的`node_modules`依赖树”的问题，造成这一问题的原因有很多，最主要的就是`npm install`那个万恶的“incremental install”特性，在Semver版本范围内尽可能拉取最新的包，应该说其初衷是好的，使用户获取的依赖保持在一个较新的版本，囊括各种新功能和BUG修复，方便扁平化依赖树，但现实中却带来了不少工程上的问题：

1. 依赖提供方难免因为疏忽甚至恶意投毒，提供了不符合Semver规范的版本号，例如有破坏性变更却未更新Major版本号，在npm社区已经出现多起恶性事件，连我自己也被2021年colors的`LIBERTY`事件所波及；
2. 有时我们需要用到某些依赖的具体实现，这些内部细节随时可能改变且不至破坏Semver规范，如果未对`package.json`中声明的依赖范围做严格限定的话某次更新后功能就可能被破坏；
3. 即使一切都符合规范，开发者执行`npm install`的时机、本地是否有缓存、使用的镜像源是否相同乃至下载依赖请求的顺序不同都可能导致同一版本声明实际生成的依赖树却不同，再加上开发者自身的问题，容易造成以下场景：声明版本为`^2.6.0`，A实际安装的是`2.7.12`版本，B实际安装的是`2.6.12`版本，两者都符合Semver规范，但A没有按照声明仅使用`2.6.x`范围内的功能，而使用了`2.7.x`的新功能，当B合并了A的代码后出现问题，而A自己复现不了，成为典型的X/Y问题。

`package-lock.json`存放的就是这么一个依赖树结构。当项目中没有`package-lock.json`，或者`package-lock.json`中的依赖不符合`package.json`的要求时，执行`npm install`或者`npm update`等涉及依赖树变更的命令将同步`package-lock.json`到最新；如果已有`package-lock.json`，且和`package.json`按照Semver规则相适配，那皆大欢喜，执行`npm install`时会直接根据`package-lock.json`的结构拉取依赖，使不同开发者、不同时机看到的依赖的行为符合预期。

`package-lock.json`出现后解决了一部分“incremental install”带来的问题，但总有些时候，由于开发者的疏忽没有提交与当前`package.json`相适配的`package-lock.json`，而`npm install`的逻辑不是报告问题而是默默地更新`package-lock.json`，这就还会带来依赖不一致的问题，在流水线环境这种对`package-lock.json`的静默更改我们甚至感知不到。于是`npm ci`应运而生，它提供了更严格的安装策略：

1. 如果项目没有`package-lock.json`，或者`package.json`与`package-lock.json`对不上，直接报错；
2. 先删除本地已有的`node_modules`文件夹并重新安装；
3. 严格按照`package-lock.json`的声明安装依赖，且不会变更之。

应该说在不涉及依赖变更的情况下，例如仅仅是拉取项目复现一下问题，`npm ci`是更合适的安装依赖的手段，有效避免莫名其妙的自动升级所带来的干扰。虽然每次都会删除`node_modules`重新安装，但`npm ci`跳过了扫描`node_modules`比对依赖树等步骤，安装速度实际上还快了不少。

最后是一些我自己总结的npm最佳实践：

1. 永远不要手动修改`package.json`中的依赖版本，应使用`npm install`以便同步`package-lock.json`；
2. 不使用`package.json`中未明确声明的依赖或某版本新功能，不对`node_modules`的结构抱有任何额外假设；
3. `package-lock.json`始终保持与`package.json`相匹配且总是提交到Git仓库，判断`package-lock.json`是否为最新的方法是执行`npm install`看`package-lock.json`有没有变更；
4. 不同分支同时变更项目依赖，合并时容易造成`package-lock.json`大范围的冲突，我的解决方法一般是找到两者变更之前的`package-lock.json`，手动执行`npm install`，或者基于一方的`package-lock.json`再施加另一方的变更。无论采取哪一种最后都需要对功能回归测试；
5. `npm install pkg@version`之后在`package.json`中会生成`^version`这样范围式的版本，通常我会尽可能修改为确定的版本，上生产环境的内容要的不是新潮而是稳定。缺点当然也有，容易造成依赖树的臃肿，例如两个范围声明`^2.6.1`、`^2.6.2`在依赖树根目录下安装一个`2.6.2`版本就够用了，也保证了单例，而两个具体声明`2.6.1`、`2.6.2`则无法扁平化；
6. 优先使用`npm ci`以便获得项目过去预期的行为。

## `dependencies`、`devDependencies`和`peerDependencies`

`dependencies`和`devDependencies`从语义上来说，一个代表开发调试依赖，通常是Webpack、ESLint之类的东西，一个代表运行时依赖，例如Vue、React等。但实际上放在哪里影响不大，npm安装依赖时默认会把它的`dependencies`和`devDependencies`都安装，只有明确使用了`--production`的时候才仅安装`dependencies`。

`peerDependencies`在开发组件时经常会用到，指代一些需要宿主应用注入给该组件的依赖，比如某Vue组件库，组件库运行时的Vue应该由宿主应用提供而非定死在`dependencies`内。由于安装后在宿主应用的`package.json`及依赖树内，该组件与该组件声明的`peerDependencies`是并列的，故得名peer。

考虑这样一个情形：宿主应用使用了依赖A，A又有一个依赖B，我们记为B1，并且B1声明在A的`dependencies`或`devDependencies`内，那么宿主应用安装A的时候会连带着安装B1，并且存在一定几率在扁平化依赖后B1直接位于依赖树的顶层，即`node_modules`的根目录下。假如宿主应用开发者不加留意，在项目中使用了B1：`require('B')`，由于B1在`node_modules`根目录下，根据NodeJS的[解析规则](https://nodejs.org/api/modules.html#all-together)这时不会报错；但未来某一天宿主应用自己也需要安装B，或者通过引入依赖C间接又引入了B，我们记为B2，好巧不巧B2与B1是Semver不兼容的，那么扁平化之后很可能（若A自己引入B2，则必定）存在于`node_modules`之下的是B2，而B1位于A的`node_modules`下：

```bash
+ code.js # `require('B')` that based on B1
+ node_modules/
  + A
    + node_modules/
      + B1
  + B2
```

这时原先建立在B1假设之上的代码就会出问题。为了避免这种情形，除了我们要注意不使用未声明的依赖之外，对这些组件库来说可能它们的最佳实践是将Vue这种外部依赖声明为`peerDependencies`，然后由宿主应用统筹兼顾所用到各组件的`peerDependencies`，找出能符合各自需求的一个版本并安装之，甚至是自己魔改的版本，这给了宿主应用很多的自由度，也是一种“单例模式”。但换个角度想，这是把一部分依赖管理的工作转交给了宿主应用的开发者，显得极不可靠。

## `npm`、`yarn`和`pnpm`

当下npm和yarn classic都采用了扁平化依赖以解决一部分`node_modules`臃肿的问题，yarn classic可以看作是npm的上位替代，在速度上有一定的提升。但生成`node_modules`是个IO密集型任务，即使有本地缓存也由于大量的文件复制动作使速度的提升有限，每个项目都有一个自己的`node_modules`也造成了磁盘空间的浪费（某知名梗图：宇宙中最重的东西.jpg）。在这个基础上，开发者们不约而同地想到了跳过安装`node_modules`，由依赖管理工具告知软件真正的依赖存放在哪里。于是出现了yarn berry和pnpm，yarn berry的脑回路比较奇怪，可能是时代局限性，它选择侵入NodeJS的模块加载过程，提供了一个需要预执行的脚本来告知模块放置的位置，而且这个脚本很大，不少于500K，我感觉这或许也是yarn berry一直不温不火的原因。而pnpm的思路比较自然，采用软链和硬链，软链针对目录结构，硬链针对模块文件，在`node_modules`这一层尽量不做扁平化，这使得pnpm的安装速度极快并避免了扁平化带来的很多问题，但也带来了依赖嵌套层级过深在Windows上可能导致的问题，在pnpm的Q&A有更多介绍，我就是因为以前在Windows上使用pnpm时遇到太多问题，一直没留下太多好感。

写到这里忽然想起来去年看过的某互联网大厂团队对依赖管理工具的改进，除了用Rust重写之外，我还有印象的是他们将规划依赖树的任务从客户端提到了服务器端，安装依赖的时候直接基于服务器下发的依赖关系进行安装，感觉这或许是解决npm诸多依赖问题的一个重要改进。


最后讲个笑话，某次求助`npm run --help`的时候看到`npm run`的别名，乐了半天：

```bash
❯ npm run --help

# balabala...

aliases: run, rum, urn
```

## Monorepo

想象一下正在维护一个依赖其他组件的组件会经历什么，在开发阶段需要协调上游组件迭代新功能并提供测试版本给我们使用，我们要提供测试版本给下游使用，上线时整个链路一般要一起发布才能保证功能，并且我们不可能同时只维护一个版本，往往是多个生产版本，多个测试版本全捏在手上。混乱的依赖，乱七八糟的版本，相互扯皮的团队，与之相比`npm link`只能算是个人项目的玩具，很多企业内会建立私有npm镜像用于内部组件的测试发布。组件团队也不应该各自为战，而是集中在一个仓库内，使用Monorepo工具进行管理，这样的仓库就是所谓的Monorepo，而前端常见的Monorepo工具有[yarn workspace](https://yarnpkg.com/features/workspaces)、[lerna](https://lerna.js.org/)等。

Monorepo管理工具通常会提供如下功能：

1. 提炼各组件的共同依赖统一管理，节省磁盘空间的同时保证行为一致；
2. 链接各种交叉依赖，仓库内各组件不需要手动安装到自己的`node_modules`中；
3. 站在仓库角度执行命令，例如统一构建统一发布等；
4. 分析各组件的依赖关系及变更内容来决定发布内容，当然也支持独立发布；
5. 生成依赖图便于掌握项目全局信息。

> 2022年听说lerna已经不再更新了，原班人马都跑去做Nx去了，虽然文档上写得很漂亮“Your favorite tool is alive and well”。

## 常见工具

### Webpack

虽然已经2023年了还谈论Webpack显得有点不合时宜，但作为一名曾经的“Webpack API工程师”，我的良心不允许我将它放在不重要的位置:)。

一般的说法是Webpack适合构建应用，而Rollup适合构建组件，不过配置得当的话这些工作它们其实都能做。Webpack采用了基于事件的体系结构，底层是他们自己开发的[Tapable](https://github.com/webpack/tapable)，这个库提供了同步、异步、并行或瀑布式等各种事件钩子，整个Webpack的执行流就是用这些钩子组合起来的。

#### Loader和Plugin

Webpack以模块为基础单位，在内部会建立模块依赖关系图并分析构建次序。但Webpack默认只内置了对js、json等基础文件类别的Loader，Webpack@5补充了对常见图片、字体格式的支持但依然不足以满足前端丰富生态的需要。Loader就是用来做这个事情的，通过`module.rules`配置Loader并指明Webpack该如何处理某一类型的文件以及把它们转化为模块。而Plugin利用Webpack暴露出来的事件钩子可以实现更花里胡哨的功能，两者的职责确实有一定重叠之处。要自己实现的话，Loader接口在[这里](https://webpack.js.org/api/loaders/)，Plugin接口在[这里](https://webpack.js.org/api/compiler-hooks/)，实际上挺不完善的，尤其是Webpack@4，经常需要结合打印出来的各种信息及源码去推测API的行为。

很多时候一类文件要添加多个Loader依次处理，比如下面的例子：

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.s[ac]ss$/i,
        use: [
          "style-loader",
          "css-loader",
          "sass-loader",
        ],
      },
    ],
  },
};
```

Loader应用的顺序默认与声明的顺序是相反的。这里先由`sass-loader`将`.sass`或`.scss`文件处理为CSS，然后由`css-loader`处理导入的CSS模块，再由`style-loader`将输出的CSS代码以`<style>`标签的形式注入到DOM中。显然这个顺序是不可以乱掉的。如果真需要手动调整Loader应用的顺序，可参考[Rule.enforce](https://webpack.js.org/configuration/module/#ruleenforce)选项。

#### Resolve

Resolve控制Webpack如何进行模块解析，和模块解析有关的比如路径别名、文件类型支持、解析算法查找过程都在这个范围内。Resolve模块背后是[enhanced-resolve](https://github.com/webpack/enhanced-resolve)这个包，建立在Tapable之上，基本可以看作是一个功能更强也更复杂的[`require.resolve`](https://nodejs.org/docs/latest-v14.x/api/modules.html#modules_require_resolve_request_options)实现。假如我们要实现类似http import或者glob import之类的功能，就可以从Resolve插件入手。

#### WDS和HMR

WDS的原理说简单也挺简单的，用FS Watcher监听本地文件变更，用Web Socket实现服务端推送。源码主体就一个文件，核心是两个中间件`webpack-dev-middleware`和`webpack-hot-middleware`，前者拉起一个Webpack实例进行构建，构建结果直接存放在内存文件系统中，后者用于实现HMR，即所谓的“模块热重载”。要我说，都能监听和推送文件变更了，实现个模块热插拔真没啥可讲的。稍微有点意思的可能是[`react-refresh`](https://www.npmjs.com/package/react-refresh)这种东西，但很多时候我又不需要组件在刷新时还保持状态，所以用的比较少。

#### Optimization

Webpack默认的构建优化配置已经很完善了，一般很少动Optimization配置，最多是将`minimize`设置为`false`以观察构建产物的内容。Optimization最主要的配置项可能是`splitChunks`，控制构建产物chunks的质量和数量，因为和应用加载时的脚本解析时间与网络请求数挂钩，将直接影响到应用性能。比如Webpack默认会将`node_modules`下的依赖打包到同一个文件`vendor`里，在项目外部依赖较多时可能出现几MB的`vendor`，严重影响性能，有时还需要进一步划分。

#### Devtools

Devtools主要控制Source Map的生成，Source Map在性能优化这里有单独的一节[介绍](./PerformanceOptimization.md#H205b751bbde92ea0)。

#### Externals

Externals也是比较常用的一个配置项，被标记为External的模块其源码并不会被Webpack打包到构建产物中，因此别人使用构建产物的时候需要自己安装并注入这些模块。External模块的角色对构建产物来说类似于静态链接库。假如我们正在开发一个基于Vue的组件，是不大可能把Vue源码也打包到组件产物中的，不仅避免臃肿和浪费磁盘空间，也避免各组件“自备”一套Vue实现可能产生的各种问题。

关于Externals我还想起一个Webpack的BUG，综合性很强，我花了整整一个下午才找到问题的根源。找出原因之后可以用如下简化的`main.js`文件及`webpack.config.js`复现问题：

+ main.js

    ```js
    import('./lazy')
    ```

+ webpack.config.js

    ```js
    module.exports = {
      mode: 'development',
      entry: './main.js',
      output: {
        chunkFilename: '[name].[contenthash:8].js'
      },
      externals: /lazy/,
    }
    ```

此时进行构建，将得到牛头不对马嘴的报错`Cannot convert undefined or null to object`：

```bash

❯ npx webpack
Version: webpack 4.46.0
Entrypoint main =
[./lazy] external "./lazy" 42 bytes {main} [built]
[./main.js] 17 bytes {main} [built]

ERROR in chunk main [entry]
Cannot convert undefined or null to object

ERROR in chunk main [entry]
main.js
Cannot convert undefined or null to object
```

如果将`main.js`中的导入改为同步导入，或者让`webpack.config.js`中`output.chunkFilename`配置不要使用`contenthash`占位符，就可以解决问题。错误的原因其实挺好理解，当试图给异步模块`lazy`的构建产物计算`contenthash`时，由于`lazy`恰好被标记为`externals`，压根没参与打包，就没有所谓的`contenthash`可言了，若同步加载通常不会生成额外chunk文件刚好跳过了这段逻辑。解决之道在于别让External模块成为chunk中的唯一模块，是也别计算`contenthash`。Webpack@5貌似已经修复了这个问题。

真实情况当然要复杂很多，上面给出的`output.chunkFilename`其实是`@vue/cli@3`的默认配置，我们有一个项目模板基于`@vue/cli@3`，某业务开发团队同事使用该项目模板时遭遇此问题，提交Issue之后我感到非常奇怪，因为这个模板已经存在很久了一直蛮正常的，我自己现场克隆仓库也没复现问题。于是和该同事沟通确实能100%复现并给了个分支给我们，对方拉取模板之后作了一些变更，但只是写了些业务代码，也没有动过构建配置，看起来和其他使用该模板的应用别无二致。那就没什么办法了，先用`git bitsect`和“控制变量法”找到最早出问题的变更，比对更改，各种尝试大致推断出和某模块引入与否（其实是代码规模）有关，再根据报错信息尝试去找Webpack中出错的源头，然而`Cannot convert undefined or null to object`甚至不是Webpack源码的一部分，而是JS的一个常见报错，比如`Object.keys(undefined)`，因此想到从Webpack的统计信息查起，以`webpack/lib/Stats.js`为起点通过打印日志一点点往上找，在`stats.toJson`函数适当位置添加`console.log(compilation.errors)`才算有了点眉目，原始的报错堆栈其实是这样：

```
ChunkRenderError: Cannot convert undefined or null to object
    at Compilation.createHash (/node_modules/webpack/lib/Compilation.js:1981:22)
    at /node_modules/webpack/lib/Compilation.js:1386:9
    at AsyncSeriesHook.eval [as callAsync] (eval at create (/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:6:1)
    at AsyncSeriesHook.lazyCompileHook (/node_modules/tapable/lib/Hook.js:154:20)
    at Compilation.seal (/node_modules/webpack/lib/Compilation.js:1342:27)
    at /node_modules/webpack/lib/Compiler.js:675:18
    at /node_modules/webpack/lib/Compilation.js:1261:4
    at AsyncSeriesHook.eval [as callAsync] (eval at create (/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:15:1)
    at AsyncSeriesHook.lazyCompileHook (/node_modules/tapable/lib/Hook.js:154:20)
    at Compilation.finish (/node_modules/webpack/lib/Compilation.js:1253:28)
```

从`Compilation.createHash`开始将添加打印日志和盲猜出错可能并修改配置验证结合起来，又过去很长时间，中途一度自闭到想放弃，应该说最终能找出问题本质有很大的运气成分，也多亏Webpack源码没有压缩混淆。Issue上我给出的复盘如下：

1. 我们在`@vue/cli`配置的基础上，对Webpack的`splitChunks`配置及`externals`配置做了进一步修改，一些模块会被打包到同一个chunk里面去以减少网络请求数，还有些模块被设置为`externals`由App原生缓存预拉取；
2. 该项目有两个异步模块A和B刚好被归类到同一个`cacheGroup`内，不过模块A被标记为`externals`，所以产物chunk里面只有模块B，`contenthash`也是根据B计算的；
3. 最近一次变更模块B新增了很多内容，超出了该`cacheGroup`的文件大小要求，现在该`cacheGroup`只有模块A，而模块A又是个`externals`依赖，于是寄了；
4. 问题难以定位一来是因为该报错会打断Webpack构建，没有构建产物可以分析比对，而业务开发团队的行为一切正常；另外就是那个让人摸不着头脑的报错信息，Webpack对出错信息的处理掩盖了最初的错误堆栈，进一步增加了排查难度……
5. 为了减少影响面，同时也观察到他们这个场景想发生有很多巧合，我给出的修复方案是在该项目内对`splitChunks.cacheGroups`配置稍作修改，覆盖模板项目的默认配置，后面模板项目迭代的时候再将相应配置调整集成进去。

#### Webpack5

Webpack4到Webpack5的变化并不是很大，做Migration也不麻烦，我觉得比较值得关注的变更有这几点：

1. 内置了对常见图片、字体格式的支持，不再需要去配置`url-loader`和`file-loader`；
2. 缓存算法、Tree Shaking算法的改进，尤其是对CJS包Tree Shaking的部分支持；
3. Module Federation，独立构建，联合部署，站在更高的角度看待应用和组件，思路并不新鲜，甚至可以说有点老套。在微前端领域可能会有点作用。

坊间传闻某些场景Webpack5甚至比Webpack4还要慢，也不知道是不是真的。更重要的是同一时期Vite火起来了。

### Rollup

能理解Webpack就不难理解Rollup，很多构建工具的概念并非Webpack独享，在Rollup那都能找到对应物。插件配置得当，让两者的行为看起来完全相同也是可以的。唯一一个我想到必须得使用Rollup的地方可能是Webpack@4不支持输出ESM格式的包。此外Rollup很轻量，适合构建组件，而Webpack构建生产级前端应用的能力是得到了时间验证的，构建大型应用一般优先考虑Webpack。

### Vite

“天下苦Webpack久矣”这话真不是开玩笑。Webpack成熟、稳定但也复杂、臃肿，如果项目庞大且磁盘性能不佳，冷启动一次动辄1分多种的耗时着实让人抓狂。Vite吸取了Webpack等构建工具的诸多教训，构建于Rollup之上，将“按需加载”的思想应用到开发服务器中，同时复用了Esbuild这类系统编程语言开发的编译构建工具，获得了相当大的性能改进以及与之适配的成功。对于常在前端工程化领域折腾的人来说，Vite还有一些顺应技术发展而生的特性，确实方便：

1. 默认对TS配置文件的支持与配置变更后的自动重载，在Webpack时代我通常是借助[nodemon](https://www.npmjs.com/package/nodemon)之类的工具实现；
2. 对SSR支持的重视，在我心里SSR是前端渲染技术发展的必由之路，相比之下在Webpack中实现SSR会麻烦一点；
3. 核心功能围绕ESM模块而非是CJS模块展开，顺应ECMA标准与前端生态的发展。

不过，Vite的核心能力是它的开发服务器来，构建时也聚焦于构建前端应用，因此假如要行使构建工具的其他功能比如CJS Bundler，也就是通过Vite去使用其底层的Rollup或Esbuild的功能，还需要额外做一些配置，这种情况下可能还是直接使用Webpack、Rollup更好些。

2022年社区还推出了Turborepo和Turbopack，沾染了前端工程化工具时下流行的风气，使用Rust开发，号称比Vite还快引得尤大下场辟谣。我之前去它们的官网看了下还只是个雏形，作为构建工具来说缺胳膊少腿的，也不知道是不是又一个卷绩效搞出的东西，暂且保持观望态度。

### Babel、SWC、Esbuild

Babel是一个JS的转译器，也即在生成AST之后没有生成IR生成Binary这样的步骤，而是依然转换为JS，通常是兼容性更广泛的JS代码。将使用ES6标准编写的代码转换为ES5或更早标准以便在IE 8、Android 6这样的平台上运行是Babel的主要功能之一，其他各种功能大多数是基于AST而来。

Babel基于[Acorn](https://github.com/acornjs/acorn)，并在[ESTree](https://github.com/estree/estree)基础上建立了自己的AST标准，说到底就是个递归下降的编译器前端，而且代码很烂，像对TS和JSX的支持都有强耦合的成分，魔改自由度非常低。我之前突发奇想打算修改语法实现类似Python那样的列表表达式，根据官方的Contribute指南折腾了两天，在得到自定义的AST之后就光速失去了兴趣。无怪去年传出团队不合的新闻。

梳理了一些Babel重点概念：

#### Plugin和Preset

Babel的代码转换功能以Plugin的形式提供，Preset则是内聚在一起以提供特定功能的插件集合。例如`@babel/preset-react`，由`@babel/plugin-syntax-jsx`、`@babel/plugin-transform-react-jsx`等插件组成，前者开启内置的JSX编译支持，后者转换为合适的React API。

#### `core-js`和polyfill

`core-js`是ECMA Script最新API标准的ES5实现。Babel虽然可以将ES6以上的语法转译为ES5，但为了提高转译速度，可以不转译标准中的一些新API，比如如`Promise`、`Array.from`等，然后采用预先导入`core-js`对应实现的方式去执行Babel转译后的代码。早期的时候为了省事可以直接导入`@babel/polyfill`（内置了`core-js`和`regenerator-runtime`），Babel 7.4.0之后`@babel/polyfill`已经废弃，官方的建议是导入`core-js/stable`，不过polyfill代码比起原生实现通常更低效，即使GZip后也有100多KB，占用加载时间，既然并非所有平台都需要polyfill，通常会根据用户机型按需加载。

`core-js`也是学习ECMA Script标准的好方法，虽说里面的代码继承了早期JS的各种糟粕，全局变量漫天飞……我就干过参考`core-js`在Lua中模拟`Promise`的[事儿](https://github.com/EverSeenTOTOTO/async-await-in-lua)，但我还没有掌握真正在Lua中实现并行（执行宏任务微任务）的技术，创建多个Lua虚拟机可以，但这样两个虚拟机之间只能交换一些便于序列化的内容，很难传递函数闭包。

#### `@babel/preset-env`

`@babel/preset-env`提供了Babel最主要的用例：对较新ECMA Script标准的转换支持。稍微值得一提的是配置项中的`include`和`exclude`，Babel和构建工具一起使用时，可能存在Babel处理后的代码新增了导入`core-js`模块的语句，构建工具检测到新模块再次交给Babel进行打包造成循环依赖并导致生成的代码执行出错的情况，这时可要借助`include`或`exclude`控制Babel处理的范围，其他一些已是ES5的代码也可以跳过转译以加快转译速度。

#### `@babel/plugin-transform-runtime`

由Babel转译的代码存在一些普遍使用的工具函数，例如实现继承的`_extend`、实现Generator的`regenerator`等，如果在每个生成的每份代码中都包含一遍这些工具函数的实现，无疑是很大的浪费，`@babel/plugin-transform-runtime`就是用来解决这个问题的，其中搜集了所有转译代码会用到的工具函数，在Babel转译后用到这些方法的地方，直接导入`@babel/plugin-transform-runtime`的相应实现就可以了。构建工具或者平台的模块机制一般能够确保这些代码仅被加载一次。

Babel很好用，但毕竟是JS实现的编译器前端，有个明显的不足就是速度慢，Babel加Webpack简直是开发者噩梦。因此陆续出现了使用系统编程语言编写的同质工具SWC和Esbuild。SWC使用Rust编写，目的比较纯粹，目的就是能无痛替换掉Babel；Esbuild使用Go编写，野心不小，目的是一个Universal Bundler。我的工作实践是，考虑到Babel久经考验以及转译为ES5或更早标准的需要，在生产构建时使用Babel，在开发阶段使用Esbuild或SWC。

### ESlint、Stylelint

两者都是基于AST的Lint工具，ESLint基于estree的AST，而Stylelint基于postcss的AST，应该说能理解[AST和访问者模式](./Compiler.md#H87d55bb42b784a04)就不难理解它们的功能和如何实现各种Rules。既然是基于AST的工具，理论上所有基于AST的操作它们都能做，而仅有AST还不足以进行的一些分析优化它们都不能做。顺便提一下Prettier和editorconfig，它们主要用于规范团队代码风格，例如换行、空行、缩进等，不过在ESLint有了`--fix`和`--format`选项之后我就很少用到了。

### Browserslist

前端开发将会面对海量的平台兼容性问题，这就是[Can I Use](https://caniuse.com/)和Browserslist出现的原因。前者是一个查询Web API在各平台上支持度的网站，后者则是一个浏览器版本信息的数据库。很多工程化工具都内置或者照搬了对Browserslist的支持，比如Babel、Webpack等，通过配置项目的目标平台版本，可以控制这些工具产出适应该平台的代码。

### Less、Sass、Postcss

CSS自身糟糕的语法设计使得书写CSS代码是一件繁琐又低效的差事，于是出现了Less和Sass/Scss，Less和Sass定义的DSL我不是很喜欢，平时用的也很少因此谈不上多了解，主要用的是Scss，Scss的语法设计非常接近于原始的CSS，比起Less和Sass也更符合人们的直觉。不过我在CSS这块的造诣没多深，即使是现在也经常需要翻阅Scss的文档去看看要复用某部分CSS代码该怎么写。

Less和Sass是所谓的CSS预处理工具，它们最终还是编译为CSS文件，而Postcss则是所谓的CSS后处理工具，在CSS基础上做变更。Postcss之于CSS正如Babel之于Javascript，它提供了一套编译处理CSS的完整工具链，可以通过插件拓展其功能，如支持最新的CSS标准、格式化CSS/Less/Scss文件、兼容多浏览器平台（[autoprefixer](https://github.com/postcss/autoprefixer)）等。

### TSC

TSC在前端工程化领域里主要被用于类型检查和生成类型文件，虽然TSC本身也提供了编译TS文件的功能，不过实践中为了统一性通常都是用Webpack/Rollup等构建JS代码，TSC使用`--emitDeclarationOnly`仅输出类型定义。这种情况下有个比较麻烦的东西是路径别名，Webpack/Rollup/TSC各自的别名不能互通，需要多次配置，且TSC编译后并不处理别名，往往还要用ts-alias之类的库处理之。

### Git hooks

Git hooks可以在执行Git操作的时候执行各种自动化脚本，是前端工程的重要组成部分，这里介绍我熟悉的几个工具：

1. [husky](https://github.com/typicode/husky)：在Git原生钩子的基础上做了简单的封装，使之更适合前端工程；

2. [lint-staged](https://github.com/okonet/lint-staged)：使用ESLint这些Lint工具的时候往往只希望对本次需求改动的内容进行检查，而不用对整个code base做检查，一者为了效率，而来也避免在Lint规则变动之后影响到以前已投产的代码。lint-staged顾名思义，可以让Lint工具只检查Git staged工作区的文件，通常与husky配合使用，在pre-commit阶段检查。

3. [commitlint](https://github.com/conventional-changelog/commitlint)：与husky配合使用，用commit-msg钩子检查提交信息，让每一次提交的说明信息更规范可读，便于生成Changelog。

### Standard Version

项目发布时使用，按照[Semver规范](#H70e0acd94076cc45)自动迭代版本并生成Changelog信息。

## Headless Browser

不知道是不是该翻译成“无头浏览器”，听着怪惊悚的。指的是没有UI层、靠代码控制的浏览器内核，浏览器的其他功能基本都有。目前主流PC端浏览器都有无头版本，安卓也可以通过`adb`达成类似效果，iOS我不是很了解，其中比较出名的可能是Chrome的Puppeteer，我拿来做个一段时间的高级爬虫，定期爬取一些新闻和财经信息，只不过我爬了也懒得看后来就废弃了。在前端工程化领域无头浏览器主要被用于测试与性能分析：

1. e2e测试，有现成的框架Cypress和Playwright，都挺好用的；
2. 页面性能分析，例如Google的Lighthouse，我有段时间致力于在Lighthouse上做个二次封装以便集成到我们的流水线上去，宏愿是自动分析前端应用性能并将生成的报告邮件到项目开发者，不过Lighthouse当时正处于变革期，从早期只能进行单个页面冷启动的性能分析往能够[根据用户操作追踪多页面生命周期的模式](https://github.com/GoogleChrome/lighthouse/blob/main/docs/user-flows.md)迁移，文档与代码均相当混乱，我在捏着鼻子通过翻源码翻Issue做到生成报告这一步之后随着工作重心的迁移就逐渐搁置了；
3. 协助生成应用文档，这是我的一个想法，还没有实施过。结合自己作为新人的经历和带新人的经历，我觉得如果能够将前端应用各个时期的页面都截图录下来并配以必要的文字信息，是非常良好的熟悉项目的方式。从事过项目交接就能体会到，过去的文档写得不管有多好，信息的丢失都是必然的，有时连人员都不一定能找到，想要复现过去的开发环境、测试数据乃至做出重构那真是战战兢兢如履薄冰。这时假如有无头浏览器，自动访问和操作各页面并生成对应截图乃至README.md，对后来者无疑是造福了，整个项目的变迁过程有迹可循，也更生动形象。
