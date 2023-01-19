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
5. `npm install pkg@version`之后在`package.json`中会生成`^version`这样范围式的版本，通常我会尽可能修改为确定的版本，上生产环境的内容要的不是新潮而是稳定。缺点当然也有，容易造成依赖树的臃肿，例如两个范围声明`^2.6.1`、`^2.6.2`在依赖树根目录下安装一个`2.6.2`版本就够用了，而两个具体声明`2.6.1`、`2.6.2`则无法扁平化；
6. 优先使用`npm ci`以便获得项目过去预期的行为。

## `dependencies`、`devDependencies`和`peerDependencies`

`dependencies`和`devDependencies`从语义上来说，一个代表开发调试依赖，通常是Webpack、Eslint之类的东西，一个代表运行时依赖，例如Vue、React等。但实际上放在哪里影响不大，npm安装依赖时默认会把它的`dependencies`和`devDependencies`都安装，只有明确使用了`--production`的时候才仅安装`dependencies`。

`peerDependencies`在开发组件时经常会用到，通常指代一些需要宿主应用注入给我们的依赖，比如某Vue组件库，组件库运行时的Vue应该由宿主应用提供而非定死在`dependencies`内。由于安装后在宿主应用的`package.json`及依赖树内，我们与我们所需的依赖是并列的，故得名peer。

考虑这样一个情形：宿主应用使用了依赖A，A又有一个依赖B，我们记为B1，并且B1声明在A的`dependencies`或`devDependencies`内，那么宿主应用安装A的时候会连带着安装B1，并且存在一定几率在扁平化依赖后B1直接位于依赖树的顶层，即`node_modules`的根目录下。假如宿主应用开发者不加留意，在项目中使用了B1：`require('B')`，由于B1在`node_modules`根目录下，根据NodeJS的[解析规则](https://nodejs.org/api/modules.html#all-together)这时不会报错；但未来某一天宿主应用自己也需要安装B，或者通过引入依赖C间接又引入了B，我们记为B2，好巧不巧B2与B1是Semver不兼容的，那么扁平化之后很可能（若A自己引入B2，则必定）存在于`node_modules`之下的是B2，而B1位于A的`node_modules`下：

```bash
+ code.js # `require('B')` that based on B1
+ node_modules/
  + A
    + node_modules/
      + B1
  + B2
```

这时原先建立在B1假设之上的代码就会出问题。为了避免这种情形，除了我们要注意不使用未声明的依赖之外，对这些组件库来说可能它们的最佳实践是将Vue这种外部依赖声明为`peerDependencies`，然后由宿主应用统筹兼顾所用到各组件的`peerDependencies`，找出能符合各自需求的一个版本并安装之，甚至是自己魔改的版本，这给了宿主应用很多的自由度。但换个角度想，这是把一部分依赖管理的工作转交给了宿主应用的开发者，显得极不可靠。

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

Loader主要指明Webpack该如何加载某一类型的文件，而Plugin利用Webpack暴露出来的事件钩子可以实现非常丰富多彩的功能，两者的职责有一定重叠之处。

#### WDS和HMR

#### Webpack5

### Rollup

### Vite、Turbopack

### Babel、SWC、Esbuild

### Eslint、Stylelint

顺便还提一下Prettier和editorconfig，它们主要用于规范团队代码风格，例如换行、空行、缩进等，不过在Eslint有了`--fix`选项之后我就很少用到它们了。

### Browserslist

### Less、Sass、Postcss

### TSC

### Git hooks

### Standard Version

## Headless Browser
