# 前端工程化

## Semver版本规范

[Semver](https://semver.org/)是前端社区普遍使用的一套版本迭代规范，一个Semver版本号由三个部分组成：Major主版本号，代表破坏性变更，不能向下兼容；Minor次版本号，代表新功能，能够向下兼容；Patch版本号，代表向下兼容的BUG修复。Release版本号格式为`Major.Minor.Patch`，有时还会在版本号后面加上`-`及由`.`分割的若干标识符，用来表示这是一个unstable的Dev版本，例如`1.0.0-alpha.0`、`2.0.0-x.y.z`，在维护组件时经常用到。还可以在版本号后面加上`+`号并连接一段元信息，如`1.0.0+20230101123456`，`2.0.0-next+sha256.123456fa`，这些元信息在行使版本的功能时会被忽略。

比较版本号时按照Major、Minor、Patch的顺序进行，若Patch后面还有`-`连接的Dev标识符，则该版本优先级要小于无Dev标识的，即`1.0.0-alpha.0`小于`1.0.0`。实践中直接用[semver](https://www.npmjs.com/package/semver)这个库来比较就是了。

Major版本号为`0`时情况稍微特殊点，通常用于表示这是一个试验性的早期版本，规范中其实没有对此时版本号迭代的行为做出建议，也没有向下兼容不兼容的保证。不过实践中根据[semver](https://www.npmjs.com/package/semver)的版本范围规范，通常将左起第一个不为0的版本号作为主版本号。

Semver规范和npm等依赖管理工具结合起来还有很多注意点，其中最重要的是npm的“incremental install”特性，在`package.json`中声明的依赖版本可以不只是某一个确定的版本，还可以是一个版本范围，在很多情况下，`npm install`总是会试图拉取范围内最新的版本（见下文）。比较好理解的表示版本范围的方式`1.2.*`、`1.2.7 || >= 1.3.0`不用说，特别提一下`^`和`~`两个表示范围的方式，前者称作“Compatible with version”，其实就是保持向下兼容（Major版本号不变）的一个范围，例如`^1.2.0`等价于`>= 1.2.0 < 2.0.0`；后者称作“Approximately equivalent to version”，在有Minor版本号的情况下代表仅Patch版本号能变化的一个范围，例如`~1.2.0`等价于`>= 1.2.0 < 1.3.0`，无Minor版本号的情况下和`^Major`等价。

## `package-lock.json`和`npm ci`

首先理清楚`npm install`的运行逻辑：

1. 检查现有的`node_modules`文件夹，生成依赖树，并制作一份克隆；
2. 获取`package.json`并分析声明的依赖版本信息，比对并更新克隆树，这个过程有两个要求：
    1. 依赖尽可能扁平化，即缺失新增的依赖会尽可能靠近树的上层，对应`node_modules`根目录；
    2. 不能破坏其他已有模块的依赖关系；
3. 比对两棵树，生成相应的安装、修改等动作并执行。

`package-lock.json`（也包括`yarn.lock`等）的出现是为了解决“相同的`package.json`安装后却可能得到不同的`node_modules`依赖树”的问题，造成这一问题的原因有很多，最主要的就是`npm install`那个万恶的“incremental install”特性，在Semver版本范围内尽可能拉取最新的包，应该说其初衷是好的，使用户获取的依赖保持在一个较新的版本，囊括各种新功能和BUG修复，方便扁平化依赖树，但现实中却带来了不少工程上的问题：

1. 依赖提供方难免因为疏忽甚至恶意投毒，提供了不符合Semver规范的版本号，例如有破坏性变更却未更新Major版本号，在npm社区已经出现多起恶性事件，连我自己也被2021年colors的`LIBERTY`事件所波及；
2. 有时我们需要用到某些依赖本不作为API对外的具体实现，这些内部细节随时可能改变且不至破坏Semver规范，如果未对`package.json`中声明的依赖范围做严格限定的话某次更新后功能就可能被破坏；
3. 即使一切都符合规范，开发者执行`npm install`的时机、本地是否有缓存、使用的镜像源是否相同乃至下载依赖请求的顺序不同都可能导致同一版本声明实际生成的依赖树却不同，再加上开发者自身的问题，容易导致以下场景：声明版本为`^2.6.0`，A实际安装的是`2.7.12`版本，B实际安装的是`2.6.12`版本，两者都符合Semver规范，但A没有按照声明仅使用`2.6.x`范围内的功能，而使用了`2.7.x`的新功能，当B合并了A的代码后出现问题，而A自己复现不了，成为典型的X/Y问题。

`package-lock.json`存放的就是这么一个依赖树结构。当项目中没有`package-lock.json`，或者`package-lock.json`中的依赖不符合`package.json`的要求时，执行`npm install`或者`npm update`等涉及依赖树变更的命令将同步`package-lock.json`到最新；如果已有`package-lock.json`，且和`package.json`按照Semver规则相适配，那皆大欢喜，执行`npm install`时会直接根据`package-lock.json`的结构拉取依赖，使不同开发者、不同时机看到的依赖的行为符合预期。

`package-lock.json`出现后解决了一部分“incremental install”带来的问题，但总有些时候，由于开发者的疏忽没有提交与当前`package.json`相适配的`package-lock.json`，而`npm install`的逻辑不是报告问题而是默默地更新`package-lock.json`，这就还会带来依赖不一致的问题。于是`npm ci`应运而生，它提供了更严格的安装策略：

1. 如果项目没有`package-lock.json`，或者`package.json`与`package-lock.json`对不上，直接报错；
2. 删除本地已有的`node_modules`文件夹；
3. 严格按照`package-lock.json`的声明安装依赖，且不会变更之。

应该说在不涉及依赖变更的情况下，例如仅仅是拉取项目复现一下问题，`npm ci`是更合适的安装依赖的手段，有效避免莫名其妙的自动升级所带来的干扰。同时由于`npm ci`跳过了扫描`node_modules`比对依赖树等步骤，安装速度上实际加快不少，非常适用于CI环境，这也是为什么它叫做`npm ci`。

最后是一些我自己总结的npm最佳实践：

1. 永远不要手动修改`package.json`中的依赖版本，应使用`npm install`以便同步`package-lock.json`；
2. 只使用`package.json`中明确声明的依赖，不对`node_modules`的结构抱有任何额外假设；
3. `package-lock.json`始终保持与`package.json`相适配且总是提交到Git仓库，判断`package-lock.json`是否为最新的方法是执行`npm install`看`package-lock.json`有没有变更；
4. 不同分支同时变更项目依赖，合并时容易造成`package-lock.json`大范围的冲突，我的解决方法一般是找到两者变更之前的`package-lock.json`，手动执行`npm install`，条件允许的话也可以合并`package.json`后直接删掉`package-lock.json`重新生成一个。无论采取哪一种最后都需要对功能回归测试；
5. `npm install pkg@version`之后在`package.json`中会生成`^version`这样范围式的版本，通常我会尽可能修改为确定的版本，上生产环境的内容要的不是新潮而是稳定。缺点当然也有，容易造成依赖树的臃肿，例如两个范围声明`^2.6.1`、`^2.6.2`在依赖树根目录下安装一个`2.6.2`版本就够用了，而两个具体声明`2.6.1`、`2.6.2`则无法扁平化；
6. 优先使用`npm ci`以便获得项目过去预期的行为。

## `dependencies`、`devDependencies`和`peerDependencies`

`dependencies`和`devDependencies`从语义上来说，一个代表开发调试依赖，通常是Webpack、Eslint之类的东西，一个代表运行时依赖，例如Vue、React等。但实际上放在哪里影响不大，npm安装依赖时默认会把它的`dependencies`和`devDependencies`都安装，只有明确使用了`--production`的时候才仅安装`dependencies`。

`peerDependencies`在开发组件时经常会用到，通常指代一些需要宿主应用注入给我们的依赖，比如某Vue组件库，组件库运行时的Vue应该由宿主应用提供而定死在`dependencies`内。考虑这样一个情形：宿主应用使用了依赖A，A自己又有一个依赖B，我们记为B1，并且B1声明在A的`dependencies`或`devDependencies`内，那么宿主应用安装A的时候会连带着安装B1，并且存在一定几率在扁平化依赖后B1直接位于依赖树的顶层，即`node_modules`的根目录下。假如宿主应用开发者不加留意，在项目中使用了B1：`require('B')`，由于B1在`node_modules`根目录下，根据NodeJS的[解析规则](https://nodejs.org/api/modules.html#all-together)这时不会报错；但未来某一天宿主应用自己也需要安装B，或者通过引入依赖C间接又引入了B，我们记为B2，好巧不巧B2与B1是Semver不兼容的，那么扁平化之后很可能存在于`node_modules`之下的是B2，而B1位于A的`node_modules`下，这时原先建立在B1假设之上的代码就会出问题。为了避免这样的情形，除了我们要注意不使用未声明的依赖之外，对这些组件库来说可能它们的最佳实践是将Vue这种外部依赖声明为`peerDependencies`，然后由宿主应用统筹兼顾所用到各组件的`peerDependencies`，找出能符合各自需求的一个版本并安装之。

还有一些依赖可能是推荐安装但并不影响功能的，也适合放在`peerDependencies`内。

## `npm`、`yarn`和`pnpm`

最后讲个笑话，某次求助`npm run --help`的时候看到`npm run`的别名，乐了半天：

```bash
❯ npm run --help

# balabala...

aliases: run, rum, urn
```

## Monorepo

## 常见工具

### Webpack

虽然已经2023年了还谈论Webpack显得有点不合时宜，但作为一名曾经的“Webpack API工程师”，我的良心不允许我将它放在不重要的位置:)。

#### Loader和Plugin

#### WDS和HMR

#### Webpack5

### Rollup

### Vite、Turbopack

### Babel、SWC、Esbuild

### Eslint、Stylelint

### Browserslist

### Less、Sass、Postcss

### TSC

### Git hooks

### Standard Version

## Headless Browser
