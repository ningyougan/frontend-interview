# Javascript

> 以下问题来自<https://www.frontendinterviewhandbook.com/zh/javascript-questions>

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

再实现`derive`，就是个对象合并，合并的时候优先用派生类定义的属性：

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
makeObject = (getter) => (prop, value) => (value !== undefined  // 根据value是否为undefined判断是set还是get
  ? makeObject((p) => (p === prop ? value : getter(p)))         // set其实创建了一个新的“对象”
  : getter(prop));

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

## CJS、ESM、IIFE和UMD
