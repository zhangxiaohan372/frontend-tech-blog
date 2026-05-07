# Promise原理、手写与 async、await

## 1. 为什么需要 Promise？

JavaScript 是单线程语言，为了避免阻塞 UI，大量操作（网络请求、定时器、文件读写）被设计为异步。早期使用**回调函数**，但多个异步任务嵌套会导致“回调地狱”：

```javascript
getUser(1, (user) => {
  getPosts(user.id, (posts) => {
    getComments(posts[0].id, (comments) => {
      console.log(comments);
    });
  });
});
```

### **小结：回调函数的缺点**

*   嵌套复杂（回调地狱问题）。
*   难以管理错误（多个异步嵌套后，错误处理困难）。
*   可读性差（功能逻辑分散，难以理解代码流程）。

Promise 通过**状态机**和**链式调用**，将异步代码写得像同步一样清晰，解决了回调地狱、错误处理困难和难以组合的问题。

***

## 2. Promise 核心概念

### 2.1 三种状态

| 状态          | 含义       | 触发方式                | 是否可逆                   |
| ----------- | -------- | ------------------- | ---------------------- |
| `pending`   | 初始状态，未完成 | `new Promise` 时     | 可变为 fulfilled/rejected |
| `fulfilled` | 操作成功     | 调用 `resolve(value)` | 不可变                    |
| `rejected`  | 操作失败     | 调用 `reject(reason)` | 不可变                    |

**小结：Promise 的状态特性**

*   从 `pending` 转为 `fulfilled` 或 `rejected` 后状态不能再变。
*   `resolve(value)` → 将状态变为 `fulfilled`。
*   `reject(reason)` → 将状态变为 `rejected`。

***

### 2.2 实例方法

```javascript
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve('成功'), 1000);
});

p.then(
  value => console.log(value),  // 成功回调
  reason => console.error(reason) // 失败回调
).catch(error => {
  // 捕获链上任何未处理的错误
}).finally(() => {
  // 无论成败都执行
});
```

### 2.3 promise执行流程图

<!-- 这是一张图片，ocr 内容为： -->

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9e0ad482867e4a0f94a0e89e2f457b87~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6aW65a2Q5LiN5ZCD6YaL:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMjY4NDM2MjkwNTYxNzA2NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1778751279&x-orig-sign=X6rjag724haVYOBvLHS%2Fjwm2QRI%3D)

#### **小结：实例方法**

1.  `then`: 接收 `fulfilled` 和 `rejected` 回调，返回新的 Promise。
2.  `catch`: 用于捕获链中的失败（代替 `then(null, onRejected)`）。
3.  `finally`: 无论成功或失败都执行，不改变返回值。

***

## 3. JS事件循环与Promise

Promise 的回调是**异步执行**的，会被放入**微任务队列**（Microtask Queue），优先级高于宏任务（如 `setTimeout`）。

### 示例：

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
// 输出顺序：1 4 3 2
```

#### **小结：事件循环与 Promise 的关系**

1.  同步代码 → 微任务队列 → 宏任务队列。
2.  Promise 的 `then` 回调属于**微任务**，优先级高于宏任务。

***

## 4. 链式调用与错误处理

### 4.1 值传递与穿透

每个 `then` 返回一个新 Promise，上一个 `then` 的返回值会传递给下一个 `then`：

```javascript
Promise.resolve(1)
  .then(x => x + 1)       // 返回 2
  .then(x => { throw x }) // 抛出错误
  .catch(err => err + 1)  // 捕获错误，返回 3
  .then(console.log);     // 输出 3
```

如果 `then` 没有传入回调，值会穿透：

```javascript
Promise.resolve(1).then().then(v => console.log(v)); // 输出 1
```

#### 场景示例：异步加载资源

```javascript
Promise.resolve('开始加载')
  .then(() => fetch('/user')) // 模拟异步数据请求
  .then(res => res.json())
  .then(data => console.log('加载完成:', data))
  .catch(err => {
    console.error('加载失败:', err);
  });
```

***

### 4.2 错误处理最佳实践

*   始终使用 `.catch` 而不是`then`的第二个参数：

```javascript
// 推荐做法（catch 更全面）
Promise.resolve()
  .then(() => { throw new Error('失败'); })
  .catch(err => console.log('捕获错误:', err));
```

*   将 `.catch`放在链式末尾，捕获前面所有错误。

***

## 5. 静态方法：组合多个 Promise

### 常见方法与场景：

| 方法                   | 行为                               | 典型场景           |
| -------------------- | -------------------------------- | -------------- |
| `Promise.all`        | 全部成功 → 结果数组；任一失败 → 立即 reject     | 多个请求必须全部成功     |
| `Promise.allSettled` | 等待所有 settled，返回每个的状态和值/原因        | 不关心个别失败，只要全部完成 |
| `Promise.race`       | 返回最先 settled 的结果（成功或失败）          | 超时控制           |
| `Promise.any`        | 返回第一个成功的结果；全部失败 → AggregateError | 多个备用接口         |
| `Promise.resolve`    | 将值转为 resolved Promise            | 统一异步处理         |
| `Promise.reject`     | 返回 rejected Promise              | 快速抛出异步错误       |

#### 示例：请求超时控制（`Promise.race`）

```javascript
function fetchWithTimeout(url, timeout = 3000) {
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('请求超时')), timeout)
  );
  return Promise.race([fetch(url), timeoutPromise]);
}

fetchWithTimeout('/data')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err));
```

***

## 6. 手写 Promise

### 小结：手写 Promise 的核心要点

1.  状态管理：`pending` → `fulfilled`/`rejected`。
2.  链式调用：`then` 返回新的 Promise，并传递返回值。
3.  微任务队列：通过 `queueMicrotask` 模拟异步执行。
4.  错误处理：支持 `catch` 和错误冒泡。

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    const resolve = (value) => {
      if (this.state !== 'pending') return;
      this.state = 'fulfilled';
      this.value = value;
      this.onFulfilledCallbacks.forEach(fn => fn());
    };

    const reject = (reason) => {
      if (this.state !== 'pending') return;
      this.state = 'rejected';
      this.reason = reason;
      this.onRejectedCallbacks.forEach(fn => fn());
    };

    try { executor(resolve, reject); } catch (err) { reject(err); }
  }

  then(onFulfilled, onRejected) {
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v;
    onRejected = typeof onRejected === 'function' ? onRejected : err => { throw err; };

    const promise2 = new MyPromise((resolve, reject) => {
      const runFulfilled = () => {
        queueMicrotask(() => {
          try {
            const x = onFulfilled(this.value);
            resolve(x);
          } catch (err) { reject(err); }
        });
      };
      const runRejected = () => {
        queueMicrotask(() => {
          try {
            const x = onRejected(this.reason);
            resolve(x);
          } catch (err) { reject(err); }
        });
      };
      if (this.state === 'fulfilled') runFulfilled();
      else if (this.state === 'rejected') runRejected();
      else {
        this.onFulfilledCallbacks.push(runFulfilled);
        this.onRejectedCallbacks.push(runRejected);
      }
    });
    return promise2;
  }

  catch(onRejected) { return this.then(null, onRejected); }

  static resolve(value) { return new MyPromise(resolve => resolve(value)); }
  static reject(reason) { return new MyPromise((_, reject) => reject(reason)); }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const results = [];
      let count = 0;
      if (promises.length === 0) return resolve(results);
      promises.forEach((p, i) => {
        MyPromise.resolve(p).then(
          val => { results[i] = val; count++; if (count === promises.length) resolve(results); },
          reject
        );
      });
    });
  }

  static race(promises) {
    return new MyPromise((resolve, reject) => {
      promises.forEach(p => MyPromise.resolve(p).then(resolve, reject));
    });
  }
}
```

***

## 7. Async/Await：更优雅的异步方案

### 示例：用户数据加载与嵌套调用

```javascript
async function getData(id) {
  try {
    const user = await fetch(`/user/${id}`).then(res => res.json());
    const posts = await fetch(`/posts/${user.id}`).then(res => res.json());
    console.log(`用户：${user.name} 的文章：`, posts);
  } catch (err) {
    console.error('加载失败:', err);
  }
}
```

***

### 小结：`async/await` 的优势

1.  语法糖，异步代码像同步代码。
2.  内置错误处理（`try/catch`），更自然。
3.  避免 Promise 链嵌套问题。

***

## 8. 常见追问与扩展

*   `then`为什么返回新 Promise？
    保证状态不可变，支持链式调用，避免修改原 Promise 的值。
*   微任务如何模拟？
    可用 `queueMicrotask` 或 `Promise.resolve().then()`；手写时也可用 `setTimeout` 但需说明那是宏任务。
*   `all` 和 `race` 的区别？
    `all` 等待全部成功或任一失败；`race` 返回最先 settled 的结果。
*   如何实现 `finally`？
    无论成败都执行，且不改变返回值，可用 `then` 链式调用实现。
*   `async/await`相比 Promise 有什么优势？
    代码更简洁、可读性更高，错误处理使用 `try/catch` 更自然；但本质上仍是 Promise 的语法糖。
*   `await`后面的代码何时执行?
    `await`表达式本身会立即执行其右侧的 Promise（或非 Promise 值），然后暂停当前 `async` 函数的执行，将后续代码包装为一个微任务。当 await 的 Promise 决议后，该微任务被推入微任务队列，在所有同步代码执行完毕后执行。

### 错误处理案例：未捕获承诺错误的危害

```javascript
async function bad() {
  const data = await fetch('/api'); // 如果 fetch 失败，错误会被吞掉
  return data;
}
```

解决方案：

*   准确使用 `try/catch` 或全局事件：

```javascript
window.addEventListener('unhandledrejection', (event) => {
  console.error('未捕获的 Promise 错误:', event.reason);
});
```

***

## 9. 全文总结与面试技巧

### 小结：主要知识点

| 知识点         | 核心要点                                     |
| ----------- | ---------------------------------------- |
| Promise 状态机 | `pending` → `fulfilled/rejected`，不可逆。    |
| 链式调用        | 每个 `then` 返回新 Promise，值穿透，错误冒泡。          |
| 微任务         | `then/catch/finally` 回调进入微任务队列，优先级高于宏任务。 |
| async/await | 更简洁的异步处理，避免嵌套。                           |
