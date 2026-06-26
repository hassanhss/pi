# 异步编程（async / Promise）详解

JavaScript 是**单线程**语言（只有一个调用栈）。遇到耗时操作（网络请求、文件 IO、定时器）时，如果同步等待，整个线程会被阻塞。异步编程让主线程能"发起任务 → 先去做别的 → 任务完成后再回来处理结果"。

异步在 JS 中的演进路径：`回调（callback）→ Promise → async/await`，三者本质是同一件事的逐步语法糖化。

---

## 一、Promise —— 异步的基石

### 1. 三种状态（state）

一个 Promise 只能处于以下三种状态之一：

| 状态 | 含义 | 是否可变 |
|------|------|----------|
| `pending` | 初始，进行中 | 可变 |
| `fulfilled` | 成功完成 | 不可逆 |
| `rejected` | 失败 | 不可逆 |

状态一旦从 `pending` 变为 `fulfilled` 或 `rejected`，就**永久锁定**，且会绑定一个最终值（value 或 reason）。

### 2. 创建 Promise

```ts
// 通过 new Promise(executor) 创建
// executor 接收两个参数：resolve 和 reject
const p = new Promise<string>((resolve, reject) => {
  const ok = Math.random() > 0.5;
  if (ok) {
    resolve("成功了"); // 把状态置为 fulfilled，值为 "成功了"
  } else {
    reject(new Error("失败了")); // 把状态置为 rejected，原因为 error
  }
});
```

`new Promise<T>(...)` 的泛型 `T` 是 **fulfilled 时 value 的类型**。注意 `reject` 的参数类型约定是 `any`（通常是 `Error`）。

### 3. 消费 Promise：then / catch / finally

```ts
p.then((value) => console.log(value))   // 处理成功
  .catch((err) => console.error(err))    // 处理失败
  .finally(() => {
    console.log("无论成败都执行");        // 清理逻辑，不接收值
  });
```

关键点：

- `then` 可接收两个回调 `(onFulfilled, onRejected)`，`catch` 只是 `then(undefined, onRejected)` 的语法糖。
- 这几个方法**都返回一个新的 Promise**，因此可以链式调用。
- 回调里的**返回值**会成为下一个 then 接收的值。
- 回调里**抛出异常**或返回 rejected Promise，会让链跳到最近的 catch。

```ts
fetchUser()
  .then((user) => user.id)
  .then((id) => fetchOrders(id)) // 返回 Promise 会被自动解包（拆箱）
  .then((orders) => console.log(orders));
```

### 4. 静态组合方法（并发控制）

| 方法 | 行为 | 成功条件 | 失败条件 |
|------|------|----------|----------|
| `Promise.all` | 等全部完成 | 全部 fulfilled | 任一 rejected 即整体 rejected |
| `Promise.allSettled` | 等全部落定 | 永不 reject，返回 `{status, value/reason}[]` | — |
| `Promise.race` | 第一个落定的（成功或失败）即决定结果 | 第一个是 fulfilled | 第一个是 rejected |
| `Promise.any` | 第一个**成功**的 | 任一 fulfilled 即整体 fulfilled | 全部 rejected 才 reject `AggregateError` |

```ts
const [users, config] = await Promise.all([
  fetchUsers(),
  fetchConfig(),
]); // 两个请求并发，都成功才返回
```

```ts
// allSettled 的返回类型很关键，要用 status 缩小类型
const results = await Promise.allSettled([a(), b()]);
results.forEach((r) => {
  if (r.status === "fulfilled") {
    console.log(r.value); // TS 知道这里有 value
  } else {
    console.error(r.reason); // 这里有 reason
  }
});
```

### 5. 两个常用的"快捷"工厂

```ts
Promise.resolve(42); // 立即 fulfilled 的 Promise
Promise.reject(err); // 立即 rejected 的 Promise
```

---

## 二、async / await —— Promise 的语法糖

`async` 函数**永远返回 Promise**；`await` 只能在 `async` 函数（或顶层 ES module）里使用，用来"暂停"等待一个 Promise 落定，并取出它的值。

### 1. 基本用法

```ts
async function loadOrders(userId: string): Promise<Order[]> {
  const user = await fetchUser(userId); // 暂停，等 fetchUser 落定
  const orders = await fetchOrders(user.id);
  return orders; // 返回值会被自动包成 Promise<Order[]>
}
```

等价的 Promise 写法：

```ts
function loadOrders(userId: string): Promise<Order[]> {
  return fetchUser(userId).then((user) => fetchOrders(user.id));
}
```

### 2. async 的"自动装箱"规则

| 函数体里做的事 | async 函数实际返回 |
|----------------|---------------------|
| `return 42`（普通值） | `Promise.resolve(42)` → fulfilled |
| `return anotherPromise`（已经是 Promise） | **原样返回，不会双层包裹** |
| `throw err` / 运行时报错 | `Promise.reject(err)` → rejected |
| 没有 return | `Promise.resolve(undefined)` |

> **结论：函数一旦用了 `async`，返回值就永远是 Promise，无论函数体里 return 的是什么。**

```ts
async function f1() { return 42; }        // Promise<number>
async function f2() { return "hi"; }      // Promise<string>
async function f3() { /* 无 return */ }   // Promise<void>

const result = f1();
console.log(result); // Promise { 42 }，不是 42
const value = await result; // 42
```

特别注意第二条——返回 Promise 不会被双层包裹：

```ts
async function getPromise(): Promise<string> {
  return fetch("/api").then((r) => r.text());
}
// 不会变成 Promise<Promise<string>>，TS 也知道类型就是 Promise<string>
```

### 3. 关键规则

- **async 函数返回值自动装箱**：`return 42` → `Promise<number>`；`return promiseX` → `Promise<T>`（不会双层包裹）；抛异常 → rejected Promise。
- **await 拆箱**：对非 Promise 值 await 直接返回原值（`await 5 === 5`）。
- **await 只能用于"等待"**，不会阻塞主线程，本质是让出执行权。
- **await 只能在 async 函数（或顶层 ES module）中使用**。

---

## 三、async 不是"返回 Promise 的原因"

> **`async` ≈ "请帮我把这个函数的返回值自动包成 Promise"**

反过来不成立：**返回 Promise 的函数不一定要写 async**。下面两种写法等价：

```ts
// 写法 A：手动返回 Promise
function fetchUser(id: string): Promise<User> {
  return fetch(`/users/${id}`).then((r) => r.json());
}

// 写法 B：async 语法糖
async function fetchUser(id: string): Promise<User> {
  const r = await fetch(`/users/${id}`);
  return r.json();
}
```

任何函数，只要 return 一个 Promise，它就返回 Promise。`async` 只是帮你**自动包装**而已，并让你能在函数体内用 `await`。

---

## 四、错误处理

async/await 用 `try/catch`，更接近同步代码的写法：

```ts
async function run() {
  try {
    const data = await fetchData();
    return process(data);
  } catch (err) {
    console.error("出错了:", err);
    return fallback;
  }
}
```

> ⚠️ 注意：`try/catch` 只能捕获**当前执行路径**上的错误。如果你并发启动了多个 Promise 但不 await，它们的 rejection 可能变成**未捕获的拒绝（unhandled rejection）**。

### 失败重试模式

```ts
async function retry<T>(fn: () => Promise<T>, times = 3): Promise<T> {
  let lastErr: unknown;
  for (let i = 0; i < times; i++) {
    try {
      return await fn();
    } catch (err) {
      lastErr = err;
      await sleep(500 * (i + 1)); // 指数退避
    }
  }
  throw lastErr;
}
```

---

## 五、并发 vs 串行（常见性能陷阱）

**串行（慢）** —— 错误地一个一个 await：

```ts
// ❌ 三个请求串行，总耗时 = a + b + c
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();
```

**并发（快）** —— 先启动，再统一等：

```ts
// ✅ 三个请求并发，总耗时 ≈ max(a, b, c)
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
```

> 经验法则：**没有依赖关系**的异步操作，先全部发起，最后再用 `await Promise.all([...])` 收口。

---

## 六、TypeScript 中的类型要点

### 1. 函数返回类型

```ts
// 显式标注返回 Promise<T> 更清晰、更安全
async function getUser(id: string): Promise<User> { /* ... */ }
```

### 2. 取出 Promise 的内部类型

```ts
type OrdersResult = ReturnType<typeof loadOrders>; // Promise<Order[]>
type Order = Awaited<OrdersResult>; // Order（拆包）
```

`Awaited<T>` 会递归拆解 `Promise`，在 TS 4.5+ 是标准工具类型。

### 3. 注意 `Promise<T>` 与 `() => Promise<T>` 的区别

```ts
function cache<T>(p: Promise<T>): void { /* p 已经在跑了，无法重跑 */ }
function run<T>(factory: () => Promise<T>): void { /* 工厂可重复调用 */ }
```

传 `Promise` 还是传"产生 Promise 的函数"，决定了它能不能被重试/重新执行。

---

## 七、常见陷阱

| 陷阱 | 说明 |
|------|------|
| **忘记 await** | `const x = asyncFn();` 拿到的是 Promise 而不是值 |
| **forEach 里 await 无效** | `arr.forEach(async ...)` 不会等，需用 `for...of` 或 `Promise.all(arr.map(...))` |
| **未捕获的 rejection** | 没 catch 的 rejected Promise 在新版 Node 会崩进程 |
| **async 函数顶层的 return** | 仍是 Promise，调用方仍需 await |
| **过早 await 丢失并发** | 见第五节 |

```ts
// ❌ forEach 不会等待内部 async
urls.forEach(async (url) => await fetch(url));
console.log("done"); // 在所有 fetch 完成前就打印了

// ✅ 正确写法
await Promise.all(urls.map((url) => fetch(url)));
console.log("done");
```

---

## 总结

**Promise 是异步值的容器（三状态 + 链式调用）；async/await 是写它的同步风格语法糖。**

掌握这四个要点，日常异步代码就稳了：

1. **类型表达**：把异步值表达为 `Promise<T>`。
2. **取值**：运行时用 `await` 取值。
3. **兜错**：用 `try/catch` 或 `.catch()` 捕获错误。
4. **并发**：用 `Promise.all` 做并发收口。

> 记住一句话：**`async` 函数的返回值永远是 Promise，`await` 用来从 Promise 里取出真正的值。**
