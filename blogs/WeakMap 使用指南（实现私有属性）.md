---
layout: default
tags: "Js"
---

# WeakMap 使用指南（实现私有属性）
最近在读 ESLint 的源码的时候，看到作者在实例化 ESLint 类的时候会将一些变量存储到 WeakMap 中，觉得这个用法应该好好记录一下。这里研究一下为什么使用 WeakMap 来进行存储，以及哪些变量/数据需要使用 WeakMap 进行存储，学习一下 WeakMap 的使用场景。

> 背 1000 道面试题，不如认真研究一个技术应用。🤷‍♀️

首先，上一下 ESLint 的源代码：

```js
const privateMembers = new WeakMap();
class ESLint {
  constructor(options = {}) {
    const processedOptions = processOptions(options);
    const warningService = new WarningService();
    const linter = createLinter(processedOptions, warningService);
    const cacheFilePath = getCacheFile(processedOptions.cacheLocation, processedOptions.cwd);

    const lintResultCache = createLintResultCache(processedOptions,  cacheFilePath);
    const defaultConfigs = createDefaultConfigs(options.plugins);
    this.#configLoader = createConfigLoader(processedOptions, defaultConfigs, linter, warningService);

    privateMembers.set(this, {
      options: processedOptions,
      linter,
      cacheFilePath,
      lintResultCache,
      defaultConfigs,
      configs: null,
      configLoader: this.#configLoader,
      warningService,
    });
  }
  // more code ...
}
```

这里，可以看到 ESLint 作者在封装 ESLint 类时，将一些变量存储在 WeakMap 中，以实现私有化。对外暴露的 node.js API —— ESLint 类，在外部一经实例化，是不能访问这些私有化成员的。

## WeakMap 简介

### 特点

（1）WeakMap 是一个键值对的集合，键必须是对象或**非全局注册的symbol**，也就是键必须是可被**垃圾回收**的。当 WeakMap 的键被垃圾回收时，其相应的值也会被垃圾回收。

（2）WeakMap 是不能够被遍历和序列化的（`JSON.stringify()`），这是因为 WeakMap 的键值状态依赖于垃圾回收的状态，是不确定的，因此不允许观察其键值的生命周期。因此如果你想访问键值列表，则应该使用 Map，其有一系列的遍历方法：forEach、entries、keys、values。而 WeakMap 是没有这些方法的。

### 示例

```js
const a = Symbol(); // 唯一
const b = Symbol(); // 唯一
const c = {};
const m = new WeakMap();

m.set(a, {a: 1});
m.set(b, {b: 2});
m.set(c, [1, 2, 3]);
console.log(JSON.stringify(m)); // 返回一个 {} ，没有内容

console.log(m.get(a)); // {a: 1}
console.log(m.get(b)); // {b: 2}
m.has(c); // true
m.delete(c);
m.has(c); // false
```
### 补充

#### 对“非全局注册的 symbol” 的理解

> 官方：使用 `Symbol()` 函数的语法，不会在你的整个代码库中创建一个可用的全局的 symbol 类型。要创建跨文件可用的 symbol，甚至跨域（每个都有它自己的全局作用域），使用 [`Symbol.for()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for) 方法和 [`Symbol.keyFor()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/keyFor) 方法从全局的 symbol 注册表设置和取得 symbol。

这句话的意思是说，如果不传递任何描述（description）、直接使用 `symbol()` 方法创建的 symbol 是不会将该 symbol 注册到全局的 symbol 注册表中的。也就是说无法通过 `Symbol.for()` 或 `Symbol.keyFor()` 注册、查找 symbol。

通常，Symbol 可以分为以下三种：

- （1）非全局注册 Symbol —— `Symbol()`
- （2）全局注册 Symbol —— `Symbol(description)`，`Symbol.for(key)`如果在全局注册中未查找到对应的 symbol 也会全局注册。
- （3）[内置 Symbol](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol#%E5%86%85%E7%BD%AE%E9%80%9A%E7%94%A8%EF%BC%88well-known%EF%BC%89symbol)

![Description](/images/weakmap_symbol.png)

## 为什么通过 WeakMap 实现私有化成员？

这篇文章中列举了几个私有化的实现方式：[Hiding Implementation Details with ECMAScript 6 WeakMaps](https://fitzgen.com/2014/01/13/hiding-implementation-details-with-e6-weakmaps.html)

### （1）在变量或方法前使用下划线 `_` 来命名

```js
function Public() {
  this._private = "foo";
}

Public.prototype.method = function () {
  // Do stuff with `this._private`...
};
```

这种方式需要开发人员自觉遵守私有变量的使用规则，如果用户在使用私有变量或方法时，完全可以对这些变量或方法进行重写，这种私有化方式只是形式上的，并不是真正实现私有化。

### （2）使用闭包

```js
function Public() {
  const closedOverPrivate = "foo";
  this.method = function () {
    // Do stuff with `closedOverPrivate`...
  };
}

// Or

function makePublic() {
  const closedOverPrivate = "foo";
  return {
    method: function () {
      // Do stuff with `closedOverPrivate`...
    }
  };
}
```
这种通过闭包的方式的确将变量封装在函数或类的内部，在调用函数或实例化类后，通常外部是不能访问到内部的变量的。但是，这种方式的缺点也很明显，就是如果这种变量多了会占用内存，影响性能。前面也提到了**垃圾回收**，在使用闭包时，如果一直保持对函数内部变量的引用，就会影响变量进行垃圾回收，内存无法释放而导致**内存泄漏**。

### （3）使用 ES6 的 Symbol

```js
const privateFoo = Symbol("foo");

function Public() {
  this[privateFoo] = "bar";
}

Public.prototype.method = function () {
  // Do stuff with `this[privateFoo]`...
};

module.exports = Public;
```
通过 symbol 作为实例属性名，用户对这些属性“只可远观而不可亵玩焉”，看上去确实实现了私有化。但是！！！，通过 `Object.getOwnPropertySymbols()`、`Reflect.ownKeys()` 依然可以得到对象自有属性 symbols 组成的数组，通过这个方式依旧可以实现对实例私有属性的访问甚至修改。

```js
const p = Symbol('private');
class Test {
    constructor() {
        this[p] = 123;
    }
}

const t = new Test(); // Test {Symbol(private): 123}
const symbols = Object.getOwnPropertySymbols(t); // [Symbol(private)]0: Symbol(private)length: 1[[Prototype]]: Array(0)
console.log(t[symbols[0]]); // 123 （访问）
t[symbols[0]] = 345; // 修改
console.log(t); // Test {Symbol(private): 345} （可以看到私有属性被修改了）
```

其他方式：

### （4）ES2022 实现 class 私有属性

ES2022 正式为 class 添加了私有属性，方法是在属性名之前使用 `#` 表示。在类的外部使用私有属性是会报错的。

```js
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Getting the current value!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}

const counter = new IncreasingCounter();
counter.#myCount // 报错！！（但是在控制台中使用是不会报错的，这样是为了方便调试）
```
由于是 ES2022 版本才有的，使用这种方式时要考虑兼容性问题。

最终，就要说到 WeakMap 了 😁

### （5）WeakMap

```js
const privateMembers = new WeakMap();
class Test() {
  constructor() {
    privateMembers.set(this, {
      // private properties ...
    })
  },
  method1() {
    const {
      // use private properties
    } = privateMembers.get(this);
  }
}
```

通过 WeakMap 来使用私有属性，可以细数出以下优点：

- a. 既能实现统一管理私有属性，又能方便垃圾回收，避免内存泄漏。
- b. ES6 开始支持 WeakMap 数据结构，兼容性相对来说会更好。
- c. 真正实现私有化。

基于上述优点，一些常见框架比如 Vue 在实现响应式时，也使用到了 WeakMap，对响应式变量的订阅进行收集。一旦响应式变量被回收，那么相应的一系列副作用也会被回收。一是方便进行统一管理（后续对副作用的执行），二是方便进行垃圾回收。 ( [深入响应式系统](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html#how-reactivity-works-in-vue))

## 使用场景

插件实现：私有属性，不希望被外部访问、修改，实现封装。

一些变量/方法需要被回收。以及兼容性考虑。

---
# 参考文章

[MDN WeakMap](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)

[MDN WeakMap 对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Keyed_collections#weakmap_object)

[Hiding Implementation Details with ECMAScript 6 WeakMaps](https://fitzgen.com/2014/01/13/hiding-implementation-details-with-e6-weakmaps.html)

[MDN Symbol](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol#%E5%85%A8%E5%B1%80%E5%85%B1%E4%BA%AB%E7%9A%84_symbol)

[MDN Object.getOwnPropertySymbols()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols)

[Github ESLint 源代码 —— ESLint 类](https://github.com/eslint/eslint/blob/main/lib/eslint/eslint.js#L563)

[阮一峰 《ES6入门》—— class 私有属性](https://es6.ruanyifeng.com/#docs/class#%E7%A7%81%E6%9C%89%E5%B1%9E%E6%80%A7%E7%9A%84%E6%AD%A3%E5%BC%8F%E5%86%99%E6%B3%95)
