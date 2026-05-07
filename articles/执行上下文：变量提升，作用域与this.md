# 深入理解 JavaScript 执行上下文

## 1. 为什么需要执行上下文？

JavaScript 代码在执行前，引擎会先进行一次**解析（Parsing）**。这一步要完成：

*   **语法检查**：有没有 `SyntaxError`。
*   **变量/函数声明的收集**：确定当前作用域中有哪些标识符。
*   **作用域规则的建立**：决定变量从哪找、函数能否提前调用。

这些信息需要被存储在一个“环境盒子”里，以便在后续执行阶段使用。这个“环境盒子”就是**执行上下文（Execution Context）**。

> 简单来说：执行上下文是 JS 引擎在代码执行前，为当前运行环境创建的执行环境结构， 用于记录变量、函数声明、作用域链以及 this 的绑定规则。

## 2. 执行上下文的类型

| 类型                | 说明                       | 数量       | 何时销毁    |
| ----------------- | ------------------------ | -------- | ------- |
| **全局执行上下文 (GEC)** | 最外层环境，浏览器中即 `window` 对象。 | 只有一个     | 页面关闭时   |
| **函数执行上下文 (FEC)** | 每次调用函数时创建。               | 每次调用创建一个 | 函数执行完毕后 |
| **eval 执行上下文**    | `eval()` 内的代码。           | 不常用      | —       |

## 3. 执行上下文的生命周期

每个执行上下文都经历两个阶段：**创建阶段** → **执行阶段**。如下图：

<!-- 这是一张图片，ocr 内容为： -->

<img width="922" height="730" alt="image" src="https://github.com/user-attachments/assets/8241f7a6-734a-4d64-8541-64e8a7b6a404" />


### 3.1 创建阶段（Creation Phase）

这是引擎“读懂代码”的阶段，主要做三件事：

1.  **创建变量对象（Variable Object, VO）**
    *   收集当前作用域中所有 `var` 声明的变量 → 提升并初始化为 `undefined`。
    *   收集所有函数声明 → 提升并**完整保存函数体**（可提前调用）。
    *   收集 `let` 和 `const` 声明的变量 → 提升但**不初始化**，存入**词法环境**并进入 **暂时性死区（TDZ）**。

> ES6 后，`let/const` 存储在独立的“词法环境”中，但理解上仍可认为“提升但不可访问”。

2.  **创建作用域链（Scope Chain）**
    *   当前上下文的变量对象 + 所有父级上下文的变量对象。
    *   决定了变量查找的顺序：从当前开始，逐级向外，直到全局。
3.  确定this的值
    *   **全局上下文**：`this` 在**创建阶段就永久绑定**为全局对象（浏览器 `window`），执行阶段不会改变。
    *   **函数上下文**：`this` 在**创建阶段仅预留位置，不赋值**，实际值在**执行阶段（函数被调用时）**，由**调用方式**动态确定（普通调用、对象方法、`call/apply/bind`、构造函数、箭头函数等规则不同）。
    *   特殊：箭头函数无自身 `this`，继承外层词法作用域的 `this`；

### 3.2 执行阶段（Execution Phase）

*   代码逐行执行，变量被赋实际值，函数被调用，表达式求值。
*   当执行到 `let/const` 声明行时，变量才完成初始化（离开 TDZ）。

## 4. 调用栈（Call Stack）

调用栈是 JS 引擎用来**跟踪函数调用顺序**的机制，遵循 **后进先出（LIFO）** 原则。如下图：

<!-- 这是一张图片，ocr 内容为： -->

<img width="1061" height="952" alt="image" src="https://github.com/user-attachments/assets/cee05b74-9988-4520-a31c-b999685729b9" />


**示例**：

```javascript
function inner() { console.log('inner'); }
function outer() { inner(); }
outer();
```

**栈变化过程**：

1.  程序启动 → 压入 **全局上下文**
2.  调用 `outer()` → 压入 **outer 上下文**
3.  `outer` 中调用 `inner()` → 压入 **inner 上下文**
4.  `inner` 执行完 → 弹出 **inner 上下文**
5.  `outer` 执行完 → 弹出 **outer 上下文**
6.  页面关闭 → 弹出 **全局上下文**

## 5. 变量提升详解

### 5.1 `var` 的提升

```javascript
console.log(a);  // undefined
var a = 10;
```

编译后等价于：

```javascript
var a;           // 提升并初始化为 undefined
console.log(a);  // undefined
a = 10;
```

### 5.2 函数声明的提升（完整提升）

```javascript
greet();         // 输出 "Hello"
function greet() {
  console.log("Hello");
}
```

函数声明连同函数体一起提升，所以可以在声明前使用。

### 5.3 `let` 和 `const` 的提升（暂时性死区）

```javascript
console.log(b);  // ReferenceError: Cannot access 'b' before initialization
let b = 20;
```

`let/const` 也会提升，但从代码块开始到声明语句之间是 **暂时性死区（TDZ）**，访问会报错。

### 5.4 函数表达式不提升

```javascript
greet2();        // TypeError: greet2 is not a function
var greet2 = function() {
  console.log("Hi");
};
```

`var greet2` 提升为 `undefined`，调用时还不是函数。

### 5.5 函数声明与 `var` 声明的优先级

当同一作用域中同时存在**函数声明**和 `var`\*\* 变量声明\*\*（同名）时，**函数声明的提升优先级更高**。

```javascript
console.log(typeof foo);   // "function"
function foo() {}
var foo = 1;
console.log(typeof foo);   // "number"
```

**编译阶段**：

*   函数声明 `function foo() {}` 被提升，`foo` 指向函数。
*   `var foo` 声明被**忽略**（因为同名标识符已存在）。

**执行阶段**：

*   第一行输出 `"function"`。
*   执行到 `var foo = 1` 时，赋值覆盖为 `1`，第二行输出 `"number"`。

> **规则**：函数声明会覆盖同名的 `var` 变量声明（但不会覆盖后续赋值）。反过来，`var` 声明不会覆盖已存在的函数声明。

## 6. 变量环境 vs 词法环境（ES6+）

| 概念       | 存放内容                    | 提升行为                       |
| -------- | ----------------------- | -------------------------- |
| **变量环境** | `var` 声明、函数声明           | 创建阶段初始化为 `undefined` 或函数引用 |
| **词法环境** | `let`、`const`、块级作用域内的声明 | 提升但不初始化（TDZ）               |

查找变量时，先查词法环境，再查变量环境。

## 7. 执行上下文与闭包

闭包的本质：**内部函数持有外部函数变量对象的引用，即使外部函数已执行完毕**。

```javascript
function outer() {
  let word = 'Hello';
  function inner() {
    console.log(word);
  }
  return inner;
}
const fn = outer();
fn();  // 输出 'Hello'
```

**原理**：

*   `outer` 执行时创建了变量对象（包含 `word`）。
*   `inner` 定义时，其内部属性 `[[Scope]]` 记录了当前作用域链（即 `outer` 的变量对象）。
*   `outer` 执行完毕弹出调用栈，但 `inner` 仍引用着 `outer` 的变量对象，所以 `word` 不会被回收。
*   调用 `fn()` 时，`inner` 通过 `[[Scope]]` 找到 `word`，输出 `'Hello'`。

## 8. 经典面试题

### 8.1 变量提升优先级（再次强调）

```javascript
console.log(typeof foo);   // ?
function foo() {}
var foo = 1;
console.log(typeof foo);   // ?
```

**答案**：`"function"` → `"number"`。

### 8.2 暂时性死区陷阱

```javascript
console.log(typeof x);   // ?
let x = 1;
```

**答案**：`ReferenceError`（不是 `"undefined"`）。\
**解释**：`let x` 的 TDZ 导致访问即报错，不会执行 `typeof` 运算。

### 8.3 循环中的 `var` 与 `let`

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 输出：3 3 3

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 输出：0 1 2
```

**解释**：`var` 函数作用域，所有回调共享同一个 `i`；`let` 块级作用域，每次迭代创建新绑定。

### 8.4 执行上下文数量

```javascript
function A() {
  function B() { }
  B();
}
A();
```

**答案**：3 个（全局 + `A` + `B`）。

## 9. 总结一句话

> **执行上下文是 JS 引擎在执行前为代码创建的环境盒子，用于存储变量、函数声明、作用域链和 **`this`**。它解释了变量提升、作用域、闭包等核心行为。**`var`\*\* 提升并初始化为 **`undefined`**，函数声明完整提升且优先级高于 **`var`**，**`let/const`** 提升但不初始化（TDZ）。调用栈以后进先出的方式管理函数执行顺序。

掌握执行上下文，你就掌握了 JS 作用域、闭包和 `this` 的底层原理。
