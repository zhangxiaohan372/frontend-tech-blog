# 深入理解 Vue3 响应式机制

## 1. 为什么需要响应式？

在传统的 jQuery 开发中，数据变化后需要手动操作 DOM 更新视图：

```javascript
let count = 0
$('#btn').click(() => {
  count++
  $('#count').text(count)   // 手动更新
})
```

这样做的问题：显然代码繁琐，逻辑分散，难以维护

**Vue 的响应式系统解决了这个问题：数据变化 → 视图自动更新。**\
开发者只需要关注数据，剩下的交给 Vue。

## 2.从 vue2 的响应式原理开始

### 2.1 核心：`Object.defineProperty`

Vue2 通过 `Object.defineProperty` 劫持对象的属性读写。

```javascript
function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      console.log(`读取 ${key}：`, val)
      return val
    },
    set(newVal) {
      if (newVal !== val) {
        console.log(`设置 ${key}：`, newVal)
        val = newVal
        // 触发视图更新
      }
    }
  })
}

const obj = { name: '张三' }
defineReactive(obj, 'name', obj.name)
obj.name = '李四'   // 触发 set
console.log(obj.name) // 触发 get
```

### 2.2 Vue2 的痛点

| 问题              | 原因                        | 解决方案                             |
| --------------- | ------------------------- | -------------------------------- |
| 无法监听新增属性        | `defineProperty` 需要预先定义属性 | `Vue.set(obj, 'newProp', value)` |
| 无法监听删除属性        | 没有 delete 拦截              | `Vue.delete(obj, 'prop')`        |
| 数组索引赋值不更新       | `arr[0] = 1` 不会触发 setter  | 使用 `$set` 或重写的数组方法               |
| 修改数组 length 不更新 | `arr.length = 0` 无拦截      | 使用 `arr.splice(0)`               |
| 初始化性能差          | 需要递归遍历所有属性                | 无解，Vue3 用 Proxy 解决               |

### 2.3 Vue2 如何处理数组？

Vue2 重写了数组的 7 个变异方法：`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`。\
当你调用这些方法时，Vue 能感知到变化并更新视图。**但直接通过索引修改或修改 length 就无法检测。**

```javascript
// Vue2 中
this.arr[0] = 1      // 不更新
this.arr.length = 0  // 不更新
this.arr.push(1)     // 更新
this.$set(this.arr, 0, 1)  // 更新
```

## 3. Vue3 的响应式原理：Proxy 全面升级

### 3.1 核心流程（一句话概括）

> **用 Proxy 代理数据，读取时收集依赖（track），修改时派发更新（trigger）。**

整个流程拆解为 4 步：

1.  用 `reactive()` 将普通对象包装成 **Proxy 代理对象**
2.  当读取属性时，`get` 拦截器调用 `track`，记录“当前正在执行的副作用函数（effect）”
3.  当修改属性时，`set` 拦截器调用 `trigger`，找出所有依赖该属性的 effect，逐个执行
4.  执行 effect 时重新读取属性，再次触发 `track`，形成闭环

流程图：

<img width="1034" height="1425" alt="image" src="https://github.com/user-attachments/assets/28f2bdfa-a4ae-4257-abfe-ac2b17bfcd99" />


### <font style="color:rgb(15, 17, 21);">3.3</font><font style="color:rgb(15, 17, 21);"> </font>track与trigger 的最小实现（理解依赖收集的核心）

<font style="color:rgb(15, 17, 21);">javascript</font>

```javascript
let activeEffect = null                 // 当前正在执行的副作用函数
const targetMap = new WeakMap()         // 存储所有对象的依赖关系

function track(target, key) {
  if (!activeEffect) return
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))
  dep.add(activeEffect)
}

function trigger(target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return
  const dep = depsMap.get(key)
  if (dep) dep.forEach(effect => effect())
}
```

## 4. Vue2 vs Vue3 响应式对比

| 对比维度         | Vue2 (`Object.defineProperty`) | Vue3 (`Proxy`)                                |
| :----------- | :----------------------------- | :-------------------------------------------- |
| 拦截能力         | 只能拦截 `get` / `set`             | 可拦截 13 种操作（get, set, delete, has, ownKeys...） |
| 新增属性         | 无法监听，需 `$set`                  | 直接赋值 `obj.newProp = 1` 即可                     |
| 删除属性         | 无法监听，需 `$delete`               | 直接 `delete obj.prop` 即可                       |
| 数组索引修改       | `arr[0]=1` 不更新                 | 可更新                                           |
| 数组 length 修改 | `arr.length=0` 不更新             | 可更新                                           |
| 初始化性能        | 递归遍历所有属性，对象越大越慢                | 惰性代理，访问到才处理，初始化快                              |
| 支持数据结构       | 普通对象、数组（需 hack）                | 对象、数组、Map、Set、WeakMap 等                       |
| 代码复杂度        | 需要递归、重写数组方法、单独处理新增/删除          | 逻辑统一在 Proxy handler 中                         |

## 5. ref 与 reactive 详解

### 5.1 核心困惑：为什么不能只用 `reactive`？

**直接原因**：`Proxy` **只能代理对象**，不能代理基本类型（数字、字符串、布尔、null、undefined）。这是因为 Proxy 的设计本质是拦截对象的属性访问、修改等行为，而基本类型是“值类型”，不是对象，没有任何可拦截的属性，无法完成代理逻辑。\
如果你写 `reactive(0)`，Vue 会报错。

**实际开发场景**：我们经常需要管理一个计数器、一个开关状态，这些是基本类型。所以必须有一个方案来处理基本类型的响应式。

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">5.2 ref 的本质：单值响应式包装器</font>

ref\`的核心作用：把任意类型的值（基本类型 / 对象）包装成一个带 value访问器的响应式对象。

#### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">真实简化原理（接近 Vue 源码）</font>

```javascript
class RefImpl {
  constructor(rawValue) {
    this._rawValue = rawValue // 原始值
    this._value = rawValue    // 响应式值
    this.__v_isRef = true     // 标记是 ref
  }

  get value() {
    // 收集依赖
    track(this, 'value')
    return this._value
  }

  set value(newVal) {
    // 更新 + 触发更新
    this._rawValue = newVal
    this._value = toReactive(newVal)
    trigger(this, 'value')
  }
}

  function ref(value) {
    return new RefImpl(value)
  }
```

### 5.3 对比表格

| 特性     | reactive           | ref                   |
| ------ | ------------------ | --------------------- |
| 支持数据类型 | 对象、数组              | 任意类型（基本类型 + 对象）       |
| 返回结构   | Proxy 代理对象         | RefImpl 实例（带 .value）  |
| 访问方式   | 直接访问属性 `state.xxx` | 必须用 `.value`          |
| 底层实现   | ES6 Proxy          | class + getter/setter |
| 响应式范围  | 深度响应式              | 单层响应式，对象自动走 reactive  |
| 解构丢失响应 | 是                  | 否（因为始终是同一个 ref 对象）    |

### 5.5 常见误区与正确理解

**误区1：** `ref` 是专门给基本类型用的，对象必须用 `reactive`。\
**事实：** `ref` 也可以接收对象，内部会调用 `reactive`。所以你可以全程用 `ref`，只是要写很多 `.value`。

**误区2：** `reactive` 返回的对象和原对象不一样，`ref` 返回的对象和原值也不一样。\
**事实：** 两者都返回代理对象。`reactive` 代理原对象；`ref` 代理包装对象。

**误区3：** `ref` 的 `.value` 是多余的。\
**事实：** 因为 `ref` 的本质是 `{ value }` 对象的代理，所以必须通过 `.value` 访问包装对象内部的属性。这是语法代价，换来了对基本类型的支持。

## 6. 常用响应式 API 速查表

| API           | 用途             | 示例                                               |
| :------------ | :------------- | :----------------------------------------------- |
| `reactive`    | 创建响应式对象/数组     | `const state = reactive({ count: 0 })`           |
| `ref`         | 创建响应式基本类型（或对象） | `const count = ref(0)` → `count.value++`         |
| `computed`    | 计算属性（缓存）       | `const double = computed(() => state.count * 2)` |
| `watch`       | 监听指定数据源        | `watch(() => state.count, (val) => {...})`       |
| `watchEffect` | 自动收集依赖，立即执行    | `watchEffect(() => console.log(state.count))`    |
| `toRefs`      | 解构时保持响应式       | `const { name } = toRefs(state)`                 |

### 6.1 computed和watch的区别

*   computed：懒加载，产生新值，有缓存，如果依赖不变调用缓存不重新计算，适用于过滤列表
*   watch：执行副作用，无缓存，可以获取新旧值，适用于异步请求

### 6.2 watch和watchEffect

*   watch：手动指定监听源，懒执行，除非immediate:true
*   watchEffect：函数内所有的响应式数据都被自动收集，立即执行，不能获取旧值

## 7. 经典面试题

### 7.1 为什么 Vue2 不能检测数组索引和 length 变化？

因为 `Object.defineProperty` 无法拦截这些操作。Vue2 只能通过重写数组方法（push/pop 等）来 hack，但直接 `arr[0]=1` 和 `arr.length=0` 无法检测到。

### 7.2 Vue3 如何解决数组问题？

`Proxy` 的 `set` 拦截器可以捕获所有属性设置，包括数字索引和 `length`。所以直接修改即可触发更新。

### 7.3 `ref` 为什么需要 `.value`？能去掉吗？

不能去掉。因为 `ref` 返回的是一个包装对象 `{ value }` 的代理，要访问内部的值就必须通过 `.value`。模板中不需要是因为编译器自动添加了 `.value`。

### 7.4 下面代码中，修改 `state.count` 会触发视图更新吗？

```javascript
const state = reactive({ count: 0 })
let { count } = state
count = 1
```

不会。因为解构后的 `count` 是普通数字，不再响应式。需要使用 `toRefs`：

```javascript
const { count } = toRefs(state)
count.value = 1   // 正确触发更新
```

## 8. 总结一句话

**Vue2 用 `defineProperty`** 劫持属性，有诸多限制；Vue3 用 **`Proxy`** 全面代理，配合 **`track`**/**`trigger`** 实现响应式。**`reactive`** 直接代理对象，**`ref`** 包装基本类型后再代理，两者本质相通。记住：对象用 **`reactive`**，基本类型用 **`ref`**，解构用 **`toRefs`**。
