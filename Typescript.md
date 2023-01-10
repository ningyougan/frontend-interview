# Typescript

说实话大多数TS面试题我觉得没有问的必要，只要理解TS类型系统的图灵完备性，理解编译期和运行期的差异，多数问题都可以元编程解决。这些题目的阻碍往往是TS在工作实践中应用**非常少**的一些技巧。现在假定你已经了解实现TS类型元编程的各种“素材”，如数字表示和基本运算，列表及列表操作等，网络上有很多教程，我也写过[一篇](https://www.everseenflash.com/CS/Snippets/Type%20Metaprogram.md)，下面选取[type-challenges](https://github.com/type-challenges/type-challenges)标记为“地狱”难度的两道题目为例说明这一点：

1. [获取只读字段](https://github.com/type-challenges/type-challenges/blob/main/questions/00005-extreme-readonly-keys/README.zh-CN.md)

    这题首先要知道TS修改属性修饰符例如只读`readonly`、可选`?`的方法，不然根本无从下手：

    ```ts
    export type freeze<T> = {
      +readonly [P in keyof T]: T[P]
    }

    export type thaw<T> = {
      -readonly [P in keyof T]: T[P]
    }

    export type partial<T> = {
      [P in keyof T]+?: T[P]
    }

    export type required<T> = {
      [P in keyof T]-?: T[P]
    }
    ```

    其次要知道如何判断两个类型相等，思路和判断两集合相等差不多：

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

    我们需要更严格的相等，怎么办，这时就该求助于[官方资源](https://github.com/Microsoft/TypeScript/issues/27024#issuecomment-421529650)了，涉及TSC的[实现细节](https://github.com/microsoft/TypeScript/blob/main/src/compiler/checker.ts)，当前版本4.9，以"Two conditional types"为关键字在该文件中搜索可以看到一些线索，TS在判断两个条件类型`T1 extends U1 ? X1 : Y1`和`T2 extends U2 ? X2 : Y2`是否相等的时候，依次进行如下判断：

    ```ts
    // Two conditional types 'T1 extends U1 ? X1 : Y1' and 'T2 extends U2 ? X2 : Y2' are related if
    // one of T1 and T2 is related to the other, U1 and U2 are identical types, X1 is related to X2,
    // and Y1 is related to Y2.

    if (isTypeIdenticalTo(sourceExtends, (target as ConditionalType).extendsType) &&
      (isRelatedTo((source as ConditionalType).checkType, (target as ConditionalType).checkType, RecursionFlags.Both) 
        || isRelatedTo((target as ConditionalType).checkType, (source as ConditionalType).checkType, RecursionFlags.Both))) {
      if (result = isRelatedTo(instantiateType(getTrueTypeFromConditionalType(source as ConditionalType), mapper), 
          getTrueTypeFromConditionalType(target as ConditionalType), RecursionFlags.Both, reportErrors)) {
        result &= isRelatedTo(getFalseTypeFromConditionalType(source as ConditionalType), 
          getFalseTypeFromConditionalType(target as ConditionalType), RecursionFlags.Both, reportErrors);
      }
      if (result) {
        return result;
      }
    }
    ```

    1. 先检查`extends`右边的类型`extendsType`，对应`U1`和`U2`是否相同`isTypeIdenticalTo`；
    2. 再检查`extends`左边的类型`checkType`，对应`T1`和`T2`，任意一个是否与另一个关联`isRelatedTo`；
    3. 最后检查对应分支类型是否相互关联，即`X1`是否与`X2`、`Y1`是否与`Y2`关联。

    由此，我们得到如下解法，首先构造两个条件类型`CondType1`和`CondType2`，使待判断目标`T`和`U`处在`extends`右边对应位置，然后再用一个条件类型对`CondType1`和`CondType2`进行相等性判断，以利用底层的比对机制：

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

    必要的技能已经掌握，回过头来看这道题目就没什么难度可言了，无非是遍历属性键值，判断该属性去掉`readonly`之后得到的对象和原对象是否相等，用伪码表达：

    ```js
    helper = (target, keys, readonlyKeys) => {
      if (isEmpty(keys)) return readonlyKeys;

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

    这题甚至还简单些，在已有类型列表的情况下，需要的只是在类型系统中比较大小的手段，因为这里的数字均是类型而非运行时的值，且TS并不直接支持`type x = 1 > 2`这样的编译期计算。实现方法有很多，这里介绍我知道比较简单的一种，利用列表的`length`属性，将问题归约为我们熟知的列表操作：

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

    有了列表操作和数值比较，实现个排序算法不成问题。至于说对大数和浮点数的支持，既然能理解该类型元编程系统是图灵完备的，实现一个IEEE754标准或者其他什么标准来表示数字只是耐心问题，没啥意义。

整个过程看下来，除了因为使用的语言是元编程语言所以显得晦涩之外，和日常开发需求没有什么区别，需要的只是多看文档多写Demo形成积累，让我们对那些故弄玄虚的人说“不”吧。

下面总结了TS类型元编程常用的几个技巧，在[TS官方文档](https://www.typescriptlang.org/docs/)上有更系统的介绍。另外我发现一个值得去做的事情是把TS的[发布日志](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html)都看一遍，从整个语言的发展过程来理解各种特性出现的动机，不必强求自己看一遍就记住。

## Contional Types和`infer`

有心人也许注意到上面我贴出的TS发布日志链接到的是2.8版本而不是当前最新的4.9版本，因为那是首次引入Conditional Types的地方，有很详细的介绍。我对Conditional Types和`infer`关键字的理解是：Conditional Types提供了分支的能力，这是实现图灵完备性的关键；而`infer`关键字提供了定义临时变量的能力，允许我们提取类型中感兴趣的信息并构造新类型用于接下来的匹配，前文例子中已多次体现。很多TS元编程代码之所以难以读懂，就是因为作者并没有善用`infer`关键字组织代码，结果形成了一行特别长的逻辑。还是前面`is_identical`的例子，网上找到的写法大多是：

```ts
export type is_identical<T, U> =
    (<P>() => P extends T ? true : false) extends
    (<Q>() => Q extends U ? true : false) ? true : false;
```

初看上去还是挺复杂的，改写之后逻辑就很清晰了：

```ts
export type is_identical<T, U> =
  (<P>() => P extends T ? true : false) extends infer CondType1
  ? (<Q>() => Q extends U ? true : false) extends infer CondType2
    ? CondType1 extends CondType2 ? true : false
    : never
  : never;
```

不难理解为：

```js
function is_identical(T, U) {
  CondType1 = (<P>() => P extends T ? true : false);
  CondType2 = (<Q>() => Q extends U ? true : false);

  return CondType1 extends CondType2 ? true : false;
}
```

Conditional Types另一个特点是Distributive性质，当`extends`左边的`checkType`是联合类型时，联合类型将被展开分派。这是很多Conditional Types“怪异”行为的来源，也常用于从联合类型中移除特定类型：

```ts
type remove_from_union<U, T> = U extends T ? never : U;

type r = remove_from_union<42 | undefined, undefined>; // 42
```

`42 | undefined`在条件类型中展开为`42 extends undefined ? never : 42`和`undefined extends undefined ? never : undefined`，然后再将它们的结果联合，得到`42 | never`即`42`。

再比如，以前我一直不理解为什么前面的`is_same<boolean, boolean>`结果不是`true`而是`boolean`，把`boolean`理解成`true | false`就豁然开朗了：

```ts
type r = is_same<boolean, boolean>; // boolean
type r = (true | false) extends boolean ? ((true | false) extends T ? true : false) : false;

// 展开
type r = r1 | r2;
type r1 = true extends boolean ? (r3 | r4) : false; // boolean
type r2 = false extends boolean ? (r5 | r6) : false;
type r3 = true extends true ? true : false;
type r4 = false extends true ? true : false;
```

## Mapped Types

## Template Literal Types

## 协变和逆变

很容易看到TS类型系统和集合论的关联，`never`对应∅，`unknown`对应全集，联合类型对应集合的并，交叉类型对应集合的交……

> `any`只是TS对JS的妥协，为了方便JS到TS的迁移~抢占市场~，设计一个`any`类型混合了`unknown`和`never`的特点，所有的类型都可以赋给`any`，此时`any`表现如`unknown`；`any`也可以赋值给所有类型，此时`any`表现如`never`。

实践中，需记住的是两句话：

1. multiple candidates for the same type variable in co-variant positions causes a union type to be inferred（出现在协变位置上同一类型变量的多个候选将被推断为联合类型）：
2. multiple candidates for the same type variable in contra-variant positions causes an intersection type to be inferred（出现在逆变位置上同一类型变量的多个候选将被推断为交叉类型）：

理解这两句话之后，我们可以完成一个TS经典问题：Tuple、Union和Intersection类型的相互转换：

```ts
type foo = { foo: 42 };
type bar = { bar: 'hello ts' };

type tuple = [foo, bar];
type union = foo | bar;
type intersect = foo & bar;
```

## 单元测试

类型推断发生在编译期，因此要对类型元编程的结果进行测试通常借助TSC，利用`tsc --noEmit`进行类型检查。

## 类型系统

