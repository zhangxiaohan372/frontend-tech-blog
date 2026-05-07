# 深入理解 JavaScript 原型链

## 1. 为什么需要原型链？

在 JavaScript 中，我们需要一种机制让对象之间共享方法或属性。如果每个实例都复制一份方法，内存会被大量浪费。原型链机制让对象可以“借用”其构造函数的 `prototype` 上的属性与方法，实现内存共享与行为继承。

简单来说：**原型链就是 JavaScript 对象之间“自己找不到就找长辈”的查找规则。**

## 2. 核心三角：prototype、**proto**、constructor

每一个构造函数（用 `function` 定义或 `class` 声明的函数）都拥有一个 `prototype` 属性，指向一个对象，这个对象就是该构造函数所创建实例的**原型**。原型对象内会有一个 `constructor` 属性指回构造函数。而每一个对象（实例）都拥有一个 `__proto__` 属性指向其构造函数的 `prototype`。这三者构成了一个闭环。

| 属性            | 归属         | 指向      | 作用        |
| ------------- | ---------- | ------- | --------- |
| `prototype`   | 函数（构造函数）   | 原型对象    | 存放共享属性和方法 |
| `__proto__`   | 所有对象（包括函数） | 该对象的原型  | 构成查找链     |
| `constructor` | 原型对象       | 关联的构造函数 | 标识对象的构造来源 |

**速记**：**函数有 prototype，对象有 ****proto****，原型有 constructor**。

```javascript
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  console.log('Hi, I am ' + this.name);
};

const p = new Person('Alice');

// 验证闭环
p.__proto__ === Person.prototype;       // true
Person.prototype.constructor === Person; // true
```

如图：

<img width="820" height="432" alt="image" src="https://github.com/user-attachments/assets/462fb1b6-bb64-4ce3-b78e-566a024d32c3" />


## 2.1 谁有 prototype？——不是所有函数都有

很多人以为“函数都有 `prototype`”，其实**只有可以作为构造函数调用的函数，JavaScript 引擎才会为它自动创建 **`prototype`** 对象**。

### 有 prototype 的函数

```javascript
// 函数声明
function foo() {}
console.log(foo.hasOwnProperty('prototype')); // true

// 匿名函数表达式
const bar = function() {};
console.log(bar.hasOwnProperty('prototype')); // true

// class（本质是函数）
class Person {}
console.log(Person.hasOwnProperty('prototype')); // true

// 内置构造函数
console.log(Object.hasOwnProperty('prototype')); // true
console.log(Array.hasOwnProperty('prototype'));  // true
```

这些函数的 `prototype` 都是引擎自动创建的普通对象，内部结构如下：

```javascript
{
  constructor: 该函数,
  [[Prototype]]: Object.prototype
}
```

### 没有 prototype 的函数

| 函数类型       | 示例                | 为何没有                            |
| ---------- | ----------------- | ------------------------------- |
| 箭头函数       | `() => {}`        | 不能作为构造函数，无 `[[Construct]]` 内部方法 |
| 方法简写       | `{ method() {} }` | 同上，纯粹的执行逻辑                      |
| bind 返回的函数 | `fn.bind(null)`   | 本质仍是原函数，不具备独立构造能力               |

验证：

```javascript
const arrow = () => {};
new arrow(); // TypeError: arrow is not a constructor
```

**记忆口诀**：**能 **`new`** 就有 **`prototype`**，不能 **`new`** 就没有。**

## 2.2 prototype 与 **proto** 的方向相反

站在不同主体的视角看，`prototype` 和 `__proto__` 恰好是反方向的：

| 主体          | 属性          | 指向     | 比喻                 |
| ----------- | ----------- | ------ | ------------------ |
| **构造函数**    | `prototype` | 它的原型对象 | **向下**（父亲为孩子准备的模板） |
| **实例 / 对象** | `__proto__` | 它的原型   | **向上**（孩子去查父亲的模板）  |

**一句话总结**：`prototype` 是构造函数“向下分发”的入口，`__proto__` 是实例“向上查找”的路径。前者是模板的提供方，后者是模板的使用方。原型链正是靠这一下一上的配合，才实现了共享与继承。

## 3. 原型链查找机制

当访问对象的属性或方法时：

1.  先在**自身属性**中找。
2.  找不到，则沿着 `__proto__` 向上一级原型对象中找。
3.  若仍未找到，继续沿 `__proto__` 向上，直到 `Object.prototype`。
4.  `Object.prototype.__proto__` 为 `null`，查找结束，返回 `undefined`。

这就是**原型链**的实质：一条由 `__proto__` 串联起来的对象查找路径。

```javascript
p.toString(); // p → Person.prototype → Object.prototype 找到
p.abc;        // p → Person.prototype → Object.prototype → null 返回 undefined
```

## 4. 内置构造函数与原型链全景

JavaScript 中，所有对象最终都继承自 `Object`，函数也继承自 `Function`，它们共同编织起完整的原型网络。

| 构造类型    | 实例举例               | 原型链路径                                                   |
| ------- | ------------------ | ------------------------------------------------------- |
| 自定义构造函数 | `new Person()`     | 实例 → `Person.prototype` → `Object.prototype` → `null`   |
| 数组      | `[1, 2, 3]`        | 实例 → `Array.prototype` → `Object.prototype` → `null`    |
| 函数      | `function foo(){}` | 实例 → `Function.prototype` → `Object.prototype` → `null` |
| 普通对象    | `{a:1}`            | 实例 → `Object.prototype` → `null`                        |

全景图（含实例）：

<!-- 这是一张图片，ocr 内容为： -->
<img width="2047" height="1177" alt="image" src="https://github.com/user-attachments/assets/20a48b7f-cbb7-4d5f-b79d-2905a10ddbf6" />


## 4.1 特例：Function 的自引用——既是鸡又是蛋

在 JavaScript 中，`Function` 构造函数是所有函数的“母亲”，但 `Function` 自己也是一个函数。这就产生了一个奇特的闭环：

```javascript
typeof Function;                            // "function"
Function.__proto__ === Function.prototype;   // true —— 它就是自己的实例！
```

这意味着 `Function.prototype` 既是所有函数的原型，也是 `Function` 自身的原型。在整个原型链图谱中，这是一个**自引用环**：

<!-- 这是一张图片，ocr 内容为： -->
<img width="863" height="477" alt="image" src="https://github.com/user-attachments/assets/fe2b18cf-0727-45cd-a111-8e7abcc93113" />


这个环为什么成立？

*   **向下（prototype）**：`Function` 作为构造函数，为所有函数实例提供共享方法（`call`、`apply`、`bind` 等），所以它有一个 `prototype`。
*   **向上（****proto****）**：`Function` 自身也是函数，需要沿着原型链去获取自己的方法，所以它的 `__proto__` 只能指向 `Function.prototype`。

| 普通构造函数（如 `Person`）                                  | `Function` 构造函数                                            |
| --------------------------------------------------- | ---------------------------------------------------------- |
| `Person.__proto__ === Function.prototype`（因为它是一个函数） | `Function.__proto__ === Function.prototype`（它也是函数，且只能指回自己） |
| `Person.prototype` 是一个普通对象，与 `Person` 不是同一个引用       | `Function.prototype` 也是 `Function` 的原型，两者形成闭环              |

> **速记**：普通函数是 `Function` 的实例；`Function` 自己是自己的实例。

理解了这个自引用，你就能明白为什么所有函数都能用 `call`、`apply`，因为它们最终都通过 `__proto__` 找到了 `Function.prototype`，包括 `Function` 自己。

## 5. 原型链与继承（ES5 方案演进）

JavaScript 早期没有 `class`，只能通过原型链实现继承。不同方案不断弥补前一代的缺陷。

| 继承方式       | 核心思路                                                | 典型缺陷          |
| ---------- | --------------------------------------------------- | ------------- |
| **原型链继承**  | `Child.prototype = new Parent()`                    | 父类引用值被所有子实例共享 |
| **构造函数继承** | 在 Child 内执行 `Parent.call(this)`                     | 父类原型上的方法无法继承  |
| **组合继承**   | 结合上面两种方式                                            | 父类构造函数被执行两次   |
| **寄生组合继承** | `Child.prototype = Object.create(Parent.prototype)` | 写法略复杂（但最优）    |

**寄生组合继承（最优 ES5 方案）示例**：

```javascript
function Parent(name) { this.name = name; }
Parent.prototype.getName = function() { return this.name; };

function Child(name, age) {
  Parent.call(this, name);
  this.age = age;
}
Child.prototype = Object.create(Parent.prototype); // 取用父原型副本
Child.prototype.constructor = Child;                // 修正 constructor
```

## 6. ES6 class 与原型链

`class` 本质上仍然是**原型链的语法糖**。定义在 `class` 内部的方法会被挂载到 `类.prototype` 上，实例通过 `__proto__` 调用。

```javascript
class Person {
  constructor(name) { this.name = name; }
  sayHi() {} // 等价于 Person.prototype.sayHi
}
class Student extends Person {
  constructor(name, grade) {
    super(name);
    this.grade = grade;
  }
}

const s = new Student('Bob', 3);
s.__proto__ === Student.prototype;              // true
Student.prototype.__proto__ === Person.prototype; // true（extends 设置）
```

`extends` 背后就做了两件事：

*   让 `Student.prototype.__proto__` 指向 `Person.prototype`（原型继承）。
*   让 `Student.__proto__` 指向 `Person`（构造函数的静态继承）。

整套机制与寄生组合继承完全一致，只是语法更直观。

## 7. 常见误区与速记

| 误区                              | 真相                                                  |
| ------------------------------- | --------------------------------------------------- |
| “所有对象都有 prototype”              | 普通函数才有 prototype，对象没有（但都有 `__proto__`）              |
| “`__proto__` 可以随意使用”            | `__proto__` 是非标准访问器，应优先使用 `Object.getPrototypeOf()` |
| “`instanceof` 检查自身构造”           | `instanceof` 是沿着原型链查找，可能误判（如跨 iframe）               |
| “`Object.create(null)` 和普通对象一样” | 用 null 做原型的对象没有 `__proto__`，无任何内置方法                 |

## 8. 总结

原型链是 JavaScript 对象之间共享数据与行为的根本机制。它通过 `__proto__` 指针建立从实例、构造函数的 `prototype`、再到 `Object.prototype` 的层层查找路径。早年的继承方案为弥补原型链的缺陷不断演进，最终沉淀为 ES6 `class` 的底层实现。理解原型链，你就掌握了对象关系、继承原理，以及 `class` 真正在做的是什么。
