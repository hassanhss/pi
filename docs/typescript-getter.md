# TypeScript Getter 详解

详细讲解 JavaScript/TypeScript 中 `get` 关键字的用法、原理和项目中的实际应用。

---

## 1. 基本概念

`get` 是语法糖，让你**用属性访问的语法去执行一个函数**：

```ts
class Example {
    private _name = "hassan";

    get name(): string {
        console.log("getter 被执行了");
        return this._name;
    }
}

const ex = new Example();
ex.name; // 控制台输出: "getter 被执行了"，返回 "hassan"
```

看起来像在读属性，实际**每次访问都会执行函数体**。

---

## 2. Getter + Setter 成对使用

通常 getter 和 setter 配合，控制属性的读写逻辑：

```ts
class Person {
    private _age = 0;

    get age(): number {
        return this._age;
    }

    set age(value: number) {
        if (value < 0) throw new Error("年龄不能为负数");
        this._age = value;
    }
}

const p = new Person();
p.age = 25;         // 调用 setter
console.log(p.age); // 调用 getter，输出 25
p.age = -1;         // 抛出 Error: 年龄不能为负数
```

外部代码感知不到 `_age` 的存在，只看到一个普通的 `age` 属性。

---

## 3. 只有 Getter（只读属性）

只定义 `get` 不定义 `set`，属性就是**只读**的：

```ts
class Config {
    get version(): string {
        return "1.0.0";
    }
}

const c = new Config();
c.version;       // ✅ "1.0.0"
c.version = "2"; // ❌ TypeError: Cannot set property version
```

---

## 4. 为什么用 Getter 而不是普通属性

### 普通属性的问题

普通属性在赋值时就固定了，后续引用变化无法感知：

```ts
class InteractiveMode {
    private session: AgentSession;

    constructor(runtimeHost: AgentSessionRuntime) {
        this.session = runtimeHost.session; // 只在构造时取一次
    }
}
```

如果 `runtimeHost` 后来切换了 session（比如用户执行 `/session` 切换会话），`this.session` 还是旧的引用。

### Getter 的优势

每次访问都动态获取最新值：

```ts
class InteractiveMode {
    private get session(): AgentSession {
        return this.runtimeHost.session; // 每次访问时动态获取
    }
}
```

---

## 5. 项目中的实际应用

### 链式 Getter

`interactive-mode.ts` 中定义了多个 getter，形成链式委托：

```ts
// interactive-mode.ts:370-381
private get session(): AgentSession {
    return this.runtimeHost.session;
}

private get agent() {
    return this.session.agent;
}

private get sessionManager() {
    return this.session.sessionManager;
}

private get settingsManager() {
    return this.session.settingsManager;
}
```

访问 `this.agent` 时的完整解析链路：

```
this.agent
  → this.session.agent
    → this.runtimeHost.session.agent
      → AgentSessionRuntime._session.agent
```

任何一层的引用变了，后续访问都能拿到最新值，不需要手动同步。

### session 从哪里来

完整创建和传递链路：

```
main.ts:676
  │  createAgentSessionRuntime(createRuntime, options)
  ▼
agent-session-runtime.ts:404
  │  new AgentSessionRuntime(result.session, ...)
  │  → this._session = _session
  ▼
agent-session-runtime.ts:95
  │  get session(): AgentSession → return this._session
  ▼
main.ts:746
  │  new InteractiveMode(runtime, ...)
  │  → this.runtimeHost = runtime
  ▼
interactive-mode.ts:370
  │  get session() → return this.runtimeHost.session
```

---

## 6. Getter vs 方法的区别

```ts
// Getter — 当作属性用
get session(): AgentSession {
    return this.runtimeHost.session;
}
this.session; // 像访问属性一样，不加括号

// 方法 — 需要括号
getSession(): AgentSession {
    return this.runtimeHost.session;
}
this.getSession(); // 需要括号调用
```

### 选择原则

| 场景 | 用 Getter | 用方法 |
|------|-----------|--------|
| 获取一个值，无副作用 | ✅ | |
| 需要传参数 | | ✅ |
| 有副作用（发请求、改状态） | | ✅ |
| 计算成本很高 | | ✅（避免意外频繁调用） |
| 每次返回最新状态 | ✅ | |

---

## 7. 底层原理

`get` 本质上是 `Object.defineProperty` 的语法糖：

```ts
// 这两种写法等价：

// 语法糖写法
class Foo {
    get bar() { return 1; }
}

// 等价于底层写法
function Foo() {}
Object.defineProperty(Foo.prototype, "bar", {
    get() { return 1; },
    enumerable: true,
    configurable: true,
});
```

getter 是定义在**原型（prototype）的属性描述符**上的，访问时 JavaScript 引擎会自动调用描述符中的 `get` 函数。
