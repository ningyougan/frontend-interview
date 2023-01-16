# Typescript

说实话大多数TS面试题我觉得没有问的必要，只要理解TS类型系统的图灵完备性，理解编译期和运行期的差异，多数问题都可以元编程解决。这些题目的阻碍往往是TS在工作实践中应用**较少**的一些技巧。现在假定你已经了解实现TS类型元编程的各种“素材”，如数字表示和基本运算，列表及列表操作等，网络上有很多教程，我也写过[一篇](https://www.everseenflash.com/CS/Snippets/Type%20Metaprogram.md)，下面选取[type-challenges](https://github.com/type-challenges/type-challenges)标记为“地狱”难度的两道题目为例说明这一点：

1. [获取只读字段](https://github.com/type-challenges/type-challenges/blob/main/questions/00005-extreme-readonly-keys/README.zh-CN.md)

    这题首先要知道TS修改属性修饰符例如只读`readonly`、可选`?`的方法，不然根本无从下手：

    ```ts
    export type freeze<T> = {
      +readonly [P in keyof T]: T[P]
    }

    export type thaw<T> = {
      -readonly [P in keyof T]: T[P]
    }

    export type free<T> = {
      [P in keyof T]+?: T[P]
    }

    export type required<T> = {
      [P in keyof T]-?: T[P]
    }
    ```

    进而实现移除指定属性`readonly`的`remove_readonly`，为了方便使用了内置的`Omit`函数，后面介绍Conditional Types的时候会介绍`Omit`的实现：

    ```ts
    // 移除指定K的readonly
    export type remove_readonly<T, K extends keyof T> = { -readonly [P in K]: T[P] } extends infer PartA
      ? Omit<T, K> extends infer PartB
        ? (PartA & PartB) extends infer R
          ? { [P in keyof R]: R[P] } // 直接返回PartA & PartB无法通过`is_identical`的相等性测试，见下文
          : never
        : never
      : never;
    ```

    然后要知道如何判断两个类型相等，思路和判断两集合相等差不多：

    ```ts
    export type is_same<T, U> = T extends U? (U extends T? true : false) : false;
    ```

    但这个实现有缺陷，属性仅带有`readonly`标记并不会被认为是不同类型，归根结底因为TS类型系统是 _structural_ 而非 _nominal_ 的，类型只要结构相同就被认为相等，`{foo?: 42}`等价于`{} | {foo: 42}`，而`readonly`对类型的“形状”来说并无差别：

    ```ts
    type foo = { foo: 42 }

    type x = is_same<partial<foo>, foo>; // false
    type y = is_same<partial<foo>, {} | foo>; // true
    type z = is_same<freeze<foo>, foo>; // true
    ```

    我们需要更严格的相等，怎么办，这时就该求助于[官方资源](https://github.com/Microsoft/TypeScript/issues/27024#issuecomment-421529650)了，当前TS版本4.9，以"Two conditional types"为关键字在[该文件](https://github.com/microsoft/TypeScript/blob/main/src/compiler/checker.ts)中搜索可以看到一些线索，TS在判断两个条件类型`T1 extends U1 ? X1 : Y1`和`T2 extends U2 ? X2 : Y2`是否相等的时候，依次进行如下判断：

    1. 先检查`extends`右边的类型`extendsType`，对应`U1`和`U2`是否相同`isTypeIdenticalTo`；
    2. 再检查`extends`左边的类型`checkType`，对应`T1`和`T2`，任意一个是否与另一个关联`isRelatedTo`；
    3. 最后检查对应分支类型是否相互关联，即`X1`是否与`X2`、`Y1`是否与`Y2`关联。

    `isTypeIdenticalTo`涉及了对内部类型标记的判断，这是此解法将`{a:1}&{b:2}`和`{a:1,b:2}`视为不相等的关键，也是得以区分`readonly`的关键。但TSC在处理条件类型时会先将其实例化（对条件类型求分支具体值）再作类型检查，因此需要设法让TSC惰性求值，也是下面`<P>() => P`的作用，构造一个Deferred Conditional Type，让TSC比对两个条件类型的数据结构而非推导出的具体分支值。

    总之，我们得到如下解法，首先构造两个条件类型`CondType1`和`CondType2`，使待判断目标`T`和`U`处在`extends`右边对应位置，然后再用一个条件类型对`CondType1`和`CondType2`进行相等性判断，以利用底层的比对机制：

    ```ts
    export type is_identical<T, U> =
      (<P>() => P extends T ? true : false) extends infer CondType1
      ? (<Q>() => Q extends U ? true : false) extends infer CondType2
        ? CondType1 extends CondType2 ? true : false
        : never
      : never;

    type foo = { foo: 42 }

    type x = is_identical<partial<foo>, foo>; // false
    type y = is_identical<partial<foo>, {} | foo>; // false
    type z = is_identical<freeze<foo>, foo>; // false
    ```

    必要的技能已经掌握，回过头来看这道题目就没什么难度可言了，无非是遍历属性键值，判断该属性去掉`readonly`之后得到的对象类型和原对象类型是否相等，用伪码表达：

    ```js
    helper = (target, keys, readonlyKeys) => {
      if (keys == []) return readonlyKeys;

      firstKey = first(keys);

      if (removeReadonly(target, firstKey) != target) { // 若去掉readonly之后和原对象不等
        return helper(target, rest(keys), concat(readonlyKeys, [firstKey]));
      }

      return helper(target, rest(keys), readonlyKeys);
    }

    getReadonlyKeys = (target) => helper(target, keyof target, []);
    ```

    可以利用各种技巧燃烧脑细胞写出精炼的代码，但我选择放弃思考，转换为对应的元编程语言：

    ```ts
    type helper<Target, Keys, ReadonlyKeys> = Keys extends []
      ? ReadonlyKeys
      : first<Keys> extends infer FirstKey
        ? is_identical<remove_readonly<Target, FirstKey>, Target> extends false
          ? helper<Target, rest<Keys>, concat<ReadonlyKeys, [FirstKey]>>
          : helper<Target, rest<Keys>, ReadonlyKeys>
        : never;

    type get_readonly_keys<Target> = helper<Target, union_to_tuple<keyof Target>, []>;
    ```

2. [排序](https://github.com/type-challenges/type-challenges/blob/main/questions/00741-extreme-sort/README.md)

    这题甚至还简单些，在已有类型列表的情况下，需要的只是在类型系统中比较大小的手段，因为所谓的数字均是类型而非运行时的值，且TS并不直接支持`type x = 1 > 2`这样的编译期计算。实现方法有很多，这里介绍我知道比较简单的一种，利用列表的`length`属性，将问题归约为我们熟知的列表操作：

    ```ts
    // 关键技巧
    type x = [42, 42]['length']; // 2

    // 根据数字创建对应长度的列表
    type create_list_helper<Num, List extends ArrayLike<unknown>> = is_same<List['length'], Num> extends true
      ? List
      : create_list_helper<Num, concat<List, [0]>>;

    type create_list<Num> = create_list_helper<Num, []>;

    // create_list用例
    type l = create_list<4>; // [0,0,0,0]

    // 比较数字大小不过是判断列表是否相等，以小于为例
    // 让I从空列表开始增长，先到达Lhs对应的列表L则小于成立
    type lt_helper<L, R, I> = is_identical<I, L> extends true
      ? is_identical<I, R> extends true
        ? false // =
        : true  // <
      : is_identical<I, R> extends true
        ? false // >
        : lt_helper<L, R, concat<I, [0]>>; // 均未到达，继续增长I

    type lt<Lhs, Rhs> = create_list<Lhs> extends infer L
      ? create_list<Rhs> extends infer R
        ? lt_helper<L, R, []>
        : never
      : never;

    // lt用例
    type x = lt<2, 3>; // true
    type y = lt<3, 2>; // false
    type z = lt<3, 3>; // false
    ```

    有了列表操作和数值比较，实现个排序算法不成问题。至于说对负数、大数和浮点数的支持，既然能理解该类型元编程系统是图灵完备的，实现一个IEEE754标准或者其他什么标准来表示数字只是耐心问题，没啥意义。

整个过程看下来，除了因为使用的语言是元编程语言所以显得晦涩之外，和日常开发需求没有什么区别，需要的只是多看文档多写Demo形成积累，让我们对那些故弄玄虚的人说“不”吧。

下面总结了TS类型元编程常用的几个技巧，在[TS官方文档](https://www.typescriptlang.org/docs/)上有更系统的介绍。另外我发现一个值得去做的事情是把TS的[发布日志](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html)都看一遍，从整个语言的发展过程来理解各种特性出现的动机，这里有很多其他文档中只字未提的技术细节，完全可以当作进阶教程来读。

## Contional Types和`infer`

有心人也许注意到上面贴出的TS发布日志链接到的是2.8版本而不是当前最新的4.9版本，因为那是首次引入Conditional Types的地方，有很详细的介绍。我对Conditional Types和`infer`关键字的理解是：Conditional Types提供了分支的能力，这是实现图灵完备性的关键；而`infer`关键字提供了定义临时变量的能力，允许我们提取类型中感兴趣的信息并构造新类型用于接下来的匹配，前文例子中已多次体现。很多TS元编程代码之所以难以读懂，就是因为作者并没有善用`infer`组织代码，结果形成了一行特别长的逻辑。还是前面`remove_readonly`的例子：

```ts
export type remove_readonly<T, K extends keyof T> = { -readonly [P in K]: T[P] } extends infer PartA
  ? Omit<T, K> extends infer PartB
    ? (PartA & PartB) extends infer R
      ? { [P in keyof R]: R[P] }
      : never
    : never
  : never;
```

不难理解为结构化的代码：

```js
function remove_readonly(T, K: keyof T) {
  PartA = { -readonly [P in K]: T[P];
  PartB = Omit<T, K>;
  R = PartA & PartB;

  return { [P in keyof R]: R[P] };
}
```

Conditional Types另一个特点是Distributive性质，当`extends`左边的`checkType`是联合类型时，联合类型将被展开分派。这是很多Conditional Types“怪异”行为的来源，也常用于实现对联合类型的filter、map等操作：

```ts
type remove_from_union<U, T> = U extends T ? never : U;

type r = remove_from_union<42 | undefined, undefined>; // 42

type map_to_tuple<U> = U extends unknown ? [U] : never;

type a = map_to_tuple<1 | 2>; // [1] | [2]
```

`42 | undefined`在条件类型中展开为`42 extends undefined ? never : 42`和`undefined extends undefined ? never : undefined`，然后再将它们的结果联合，得到`42 | never`即`42`。

再比如，以前我一直不理解为什么前面实现的`is_same`用`boolean`做参数时，`is_same<boolean, boolean>`结果不是`true`而是`boolean`，把`boolean`理解成`true | false`就豁然开朗了：

```ts
type r = is_same<boolean, boolean>; // boolean
type r = boolean extends boolean ? (boolean extends boolean ? true : false) : false;

// 展开
type r = r1 | r2;
type r1 = true extends boolean ? (r3 | r4) : false; // (true | false) = boolean
type r2 = false extends boolean ? (r5 | r6) : false;
type r3 = true extends true ? true : false; // true
type r4 = false extends true ? true : false; // false
```

实现TS内置函数`Omit`：

```ts
export type exclude<Keys, K> = Keys extends K ? never : Keys;

export type omit<T, K extends keyof any> = {
  [P in exclude<keyof T, K>]: T[P]
}
```

`exclude`利用分派将`Keys`（来自于`keyof K`，所以是一个联合类型）分解为多个子项，再检查是否属于`K`，是则抛弃，不是则保留。不过，即使我们不知道这个技巧，也完全可以用列表操作实现：

```ts
export type exclude<Keys, K> = union_to_tuple<Keys> extends infer L
  ? union_to_tuple<K> extends infer I
    ? list_remove<L, I> extends infer R
      ? tuple_to_union<R>
      : never
    : never
  : never;
```

## Mapped Types

Mapped Types提供操作对象类型（接口）的手段，前面已经写得铺天盖地了。套路无非是对属性的操作或者对值的操作：

```ts
type fn<O> = {
  [P in <提供属性列表（联合类型）> as <操作属性>]: <操作值O[P]>
}
```

这里的`as`是4.1新增的特性，可以用于过滤属性，也可以和Template Literal Types结合起到修改属性的效果：

```ts
type foo<T> = {
  [P in keyof T as `${string & exclude<P, 'a'>}ar`]: T[P]
}

type r = { a: 1, b: 2 }

type t = foo<r>; // { bar: 2 }
```

## Template Literal Types

Template Literal Types同样在4.1版本引入，它提供了操作类型字符串的能力，这对TS类型系统的能力有多大的拓展无须多言。随便来几个字符串常用函数：

```ts
type string_concat_helper<L, S extends string> = L extends [infer First, ...infer Rest]
  ? string_concat_helper<Rest, `${S}${string & First}`>
  : S

export type string_concat<L> = string_concat_helper<L, ''>;
export type string_first<S> = S extends `${infer F}${infer _}` ? F : never;
export type string_rest<S> = S extends `${infer _}${infer R}` ? R : never;

export type string_length<S> = S extends ''
  ? 0
  : concat<[0], create_list<string_length<string_rest<S>>>> extends infer L
    ? L extends ArrayLike<unknown>
      ? L['length']
      : never
    : never;

type x = string_concat<["hello", " ", "world"]>; // hello world
type f = string_first<x>; // h
type l = string_length<x>; // 11
```

留意`string_first`，TS会把`F`推断为字符串的首个字符（空串则是`never`），这是我们得以实现`string_length`的关键。当下如果直接对一个字符串类型求`['length']`，得到的是`number`而非具体数值，所以才借助于列表操作：

```ts
type l = 'str'['length']; // number, not 3
```

## 协变和逆变

很容易看出TS类型系统和集合论的关联，`never`对应∅，`unknown`对应全集，联合类型对应集合的并，交叉类型对应集合的交……

> `any`只是TS对JS的妥协，为了方便JS到TS的迁移~抢占市场~，设计一个`any`类型混合了`unknown`和`never`的特点，所有的类型都可以赋给`any`，此时`any`表现如`unknown`；`any`也可以赋值给所有类型，此时`any`表现如`never`。

下面的`foo`而`bar`都代表了具有相应形状的所有对象的集合，`union`对对象的要求放松了，`intersect`则收紧，因此我们把`foo`理解成`union`的子类型（子集），`intersect`理解成`foo`的子类型。

```ts
type foo = { foo: 42 };
type bar = { bar: 'hello ts' };

type union = foo | bar;
type tuple = [foo, bar];
type intersect = foo & bar;
```

我从编程语言相关的资料中陆续看到过一些关于协变和逆变的介绍，但始终没有得到一个完善的系统的描述。因此我决定不想得那么复杂，协变和逆变到底是什么？它们是“原有父子关系的一对集合（类型、范畴……），进行某变换之后是否能够保持该关系的**变换的性质**”的描述。

协变即保持这种关系，通常比较符合我们的直觉，我们所接受的关于OOP的教育也强调了这一点（里氏代换原则）：任何基类可以出现的地方，子类都可以出现。那么`union[]`可以出现的地方，`foo[]`一定可以出现，换言之就是类型为`foo[]`的对象可以赋值给类型为`union[]`的变量。假如如存在一个变换将类型`T`转换为类型`T[]`：

```ts
type transform<T> = T[];

let x: transform<foo> = [];
let y: transform<union> = x;

let a: transform<union> = [];
let b: transform<foo> = a; // Type 'transform<union>' is not assignable to type 'transform<foo>'
```

于是我们知道变换`transform`是协变的，`transform`之后的类型保留了父子关系，`foo[]`是`union[]`的子类型。再比如函数返回值，也是协变的：

```ts
type transform<T> = () => T;

let x: transform<foo> = () => ({ foo: 42 });
let y: transform<union> = x;

let a: transform<union> = () => ({ foo: 42 });
let b: transform<foo> = a; // Type 'transform<union>' is not assignable to type 'transform<foo>'
```

逆变反之，变换后逆转了两种结果类型的父子关系，典型的例子是函数参数：

```ts
type transform<T> = (_: T) => void;

let x: transform<foo> = (_: foo) => { };
let y: transform<union> = x; // Type 'transform<foo>' is not assignable to type 'transform<union>'

let a: transform<union> = (_: union) => { };
let b: transform<foo> = a;
```

仔细一想就能理解为什么有这样反直觉的设计，假如我们有一个函数`fn`，当`fn`的类型是`(_: foo) => void`时，意味着`fn`出现的地方可能会有`fn({ foo: 42 })`之类的调用，当我们实际定义的函数是`(_: union) => { }`时不会有问题；反之如果`fn`的类型是`(_: union) => void`，意味着可能有`fn({ bar: 'hello ts' })`这样的用例，若实际定义的函数是`(_: foo) => { }`就出错了。

TS中直接体现类型父子关系的无疑是Conditional Types，官方文档中值得留意的是这两句话：

1. multiple candidates for the same type variable in co-variant positions causes a union type to be inferred（出现在协变位置上同一类型变量的多个候选将被推断为联合类型）：

    ```ts
    type f1 = () => foo;
    type f2 = () => bar;

    type x = (f1 | f2) extends () => infer U ? U : never; // foo | bar
    ```

    这一点可以解释很多TS不符合预期的行为，也可以帮助我们判断一个类型变换是协变还是逆变的，这是查资料时无意间看到的一个[Issue](https://github.com/microsoft/TypeScript/issues/46579)：

    ```ts
    type transform<T, P> = ([T] | [P]) extends [infer U] ? U : never;

    type x = transform<string, number>; // string | number
    ```

2. multiple candidates for the same type variable in contra-variant positions causes an intersection type to be inferred（出现在逆变位置上同一类型变量的多个候选将被推断为交叉类型）：

    ```ts
    type f1 = (_: foo) => void;
    type f2 = (_: bar) => void;

    type x = (f1 | f2) extends (_: infer I) => void ? I : never; // foo & bar
    ```

现在我们有能力完成一个TS经典问题：**Tuple、Union和Intersection类型的相互转换**：

首先是Tuple到Union，借助列表操作，从`never`开始，每次从Tuple中取出一个类型与之相联合，但还存在一个简单到令人发指的方法：

```ts
export type tuple_to_union<T extends ArrayLike<unknown>> = T[number];

type x = is_identical<tuple_to_union<tuple>, union>; // true
```

其次是Union到Intersection，衍生自第二句话给出的例子，首先利用联合类型的Distributive特性分解出每一项，利用函数参数是逆变的构造交集类型：

```ts
export type union_to_intersect<T> = (T extends unknown ? (_: T) => void : never) extends ((_: infer I) => void)
  ? I
  : never;

type x = is_identical<union_to_intersect<union>, intersect>; // true
```

顺便实现Union到Tuple，因为接下来会用。说是“顺便”，其实并不容易，又用到一个鲜为人知的技巧，思路来自[这里](https://github.com/microsoft/TypeScript/issues/13298#issuecomment-468114901)，核心内容抽象为下面这段代码：

```ts
type fn = ((_: 1) => void) & ((_: 2) => void);

type x = fn extends ((_: infer Last) => void) ? Last : never; // 2
```

目前我还没有找到这个技巧的详细解释，等找到了再来更新。有了这个就可以提取出Union的最后一项，先利用联合类型的分派机制构造交叉函数类型，从中提取出Union的最后一项，于是问题归约为我们熟知的列表操作：

```ts
type union_to_intersect_fn<T> = union_to_intersect<T extends unknown ? (_: T) => void : never>;

export type union_last<T> = union_to_intersect_fn<T> extends ((_: infer Last) => void) ? Last : never;

export type union_to_tuple<T> = is_same<T, never> extends true
  ? []
  : union_last<T> extends infer Last
    ? remove_from_union<T, Last> extends infer Remain
      ? concat<union_to_tuple<Remain>, [Last]>
      : never
    : never;

type x = union_to_tuple<1 | 2 | 3>; // [1, 2, 3]
```

最后是Intersection转Tuple完成闭环。特别一提，`string`、`number`等Primitive类型相交集之后得到的是`never`，是没办法还原出原来的类型的，所以我们主要谈论的是对象类型的交集。

实现的难点和分解Union类似，在没有额外信息的情况下，该如何提取出Intersection每个项呢？本来从交集得到原始集合听起来就匪夷所思，交集类型也不像联合类型那样有分派特性，因此还是依赖TSC的实现细节。还记得前面的`is_identical`吗？它会将`{a:1}&{b:2}`和`{a:1,b:2}`判断为不相等，这就是我们实现的关键。方法其实很笨，先把Intersection转为等价的对象类型，求出所有属性的子集并转为相应对象类型，将这些对象类型相互组合（等价于将对象类型构成的大集合再求一次子集），然后看各个集合内部元素相交是否与原有的交集类型相等，相等则该集合就是解。集合的子集数量是有限的，所以算法一定会终止，只是很低效，因为子集个数是2的幂次，两次求子集意味着回溯树非常庞大，大到tsserver提示`Type instantiation is excessively deep and possibly infinite.`以及我的IDE一度卡爆。

```ts
// 关键技巧
type x = is_identical<{ a: 1 } & { b: 2 }, { b: 2 } & { a: 1 }>; // true
type y = is_identical<{ a: 1 } & { b: 2 }, { a: 1, b: 2 }>; // false
```

先写出伪码，回溯实现求全部子集，类型元编程中需充分利用纯函数和递归思想，故改写一般的回溯算法框架后形成了两个相互嵌套的函数`gs_foreach`和`gs_helper`，显得异常复杂，没有耐心的话直接看`intersect_to_tuple`的注释吧：

```js
gs_foreach = (choices, subset, result) => { // for循环遍历choices
  if (choices == []) return result;

  choice = first(choices);
  remain = rest(choices);

  // choose
  s = concat(subset, [choice]);
  // backtrace
  r = gs_helper(remain, s, result);
  return gs_foreach(remain, /* unchoose */ subset, concat(result, r));
}

gs_helper = (choices, subset, result) => {
  r = concat(result, [subset]);

  return gs_foreach(choices, subset, r);
}

get_subsets = (U) => rest(gs_helper(U, [], [])); // rest移除开头的空集[]

// 对L中的每个集合U，比对U内部元素相交后是否等于I
i2t_helper = (I, L) => {
  U = first(L);

  return tuple_to_intersect(U) == I
    ? U
    : i2t_helper(rest(L));
}

intersect_to_tuple = (I) => { // 假设I有三个属性A、B、C，属性值都是0
  // subsets = [[A], [B], [C], [A, B] ,[B, C], [A, C], [A, B, C]]
  subsets = get_subsets(union_to_tuple(keyof I));

  // 对subsets中的每个集合K，映射为对应的 { [P in K]: I[K] } 对象类型
  // 因此将得到[{A:0}, {B:0}, {C:0}, {A:0,B:0}, ...]
  objtypes = get_objtypes(T, subsets);

  // compositions将有2**7-1多个项
  // 生成compositions的过程可以进一步优化，因为单独一个[A]是不可能构成原有类型T的，
  // 必须A、B、C都出现，如[[A], [B], [C]]，[[A], [B, C], [A, B, C]]等
  compositions = get_subsets(objtypes);

  return i2t_helper(I, compositions);
}
```

转化为元编程代码：

```ts
type gs_foreach<choices, subset, result> = choices extends []
  ? result
  : first<choices> extends infer choice
    ? rest<choices> extends infer remain
      ? concat<subset, [choice]> extends infer s
        ? gs_helper<remain, s, result> extends infer r
          ? gs_foreach<remain, subset, r>
          : never
        : never
      : never
    : never;

type gs_helper<choices, subset, result> = concat<result, [subset]> extends infer r
  ? gs_foreach<choices, subset, r>
  : never;

export type get_subsets<U> = rest<gs_helper<U, [], []>>;

// type x = get_subsets<[1, 2, 3]>; // type x = [[1], [1, 2], [1, 2, 3], [1, 3], [2], [2, 3], [3]]

export type tuple_to_intersect<L> = L extends []
  ? unknown
  : first<L> & tuple_to_intersect<rest<L>>;

type i2t_helper<I, L> = first<L> extends infer U
  ? is_identical<tuple_to_intersect<U>, I> extends true
    ? U
    : i2t_helper<I, rest<L>>
  : never;

// type x = i2t_helper<foo & bar, [[foo], [foo, bar], []]>; // x = [foo, bar]

type get_objtypes<T, subsets> = subsets extends []
  ? []
  : first<subsets> extends infer set
    ? tuple_to_union<set> extends infer K
      ? K extends keyof T
        ? { [P in K]: T[P] } extends infer obj
          ? concat<[obj], get_objtypes<T, rest<subsets>>>
          : never
        : never
      : never
    : never;

export type intersect_to_tuple<I> = get_subsets<union_to_tuple<I>> extends infer subsets
  ? get_objtypes<I, subsets> extends infer objtypes
    ? get_subsets<objtypes> extends infer compositions
      ? i2t_helper<I, compositions>
      : never
    : never
  : never;
```

好吧，我承认当前TSC还不支持这么深的函数堆栈，也许要把这个实现进一步优化，尤其是借助尾递归优化后才能应用于现实工程，但分步进行的话证实这个思路是可行的：

```ts
type r = intersect_to_tuple<intersect>; // Type instantiation is excessively deep and possibly infinite.

// 分步进行
type s = get_subsets<union_to_tuple<keyof intersect>>; // type s = [["foo"], ["foo", "bar"], ["bar"]]
type o = get_objtypes<intersect, s>; // ...
type c = get_subsets<o>; // ...
type t = i2t_helper<intersect, c>; // type t = [foo, bar]
```

## 单元测试

类型推断发生在编译期，因此要对类型元编程的结果进行测试通常借助 ~鼠标悬停查看类型推导的结果~ TSC，利用`tsc --noEmit`进行类型检查。

## 类型系统

