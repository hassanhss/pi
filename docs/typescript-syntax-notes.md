# TypeScript 语法笔记

本文档记录了项目中常见的 TypeScript/JavaScript 语法知识点。

---

## 1. 展开运算符（Spread Operator）`...`

`...` 的作用是把一个数组"拆开"，将里面的元素逐个放入当前数组中。

### 基本用法

```ts
const a = [1, 2, 3];
const b = [4, 5];
const result = [...a, ...b]; // [1, 2, 3, 4, 5]
```

### 项目中的示例

`packages/coding-agent/src/main.ts` 中收集诊断信息的代码：

```ts
const diagnostics: AgentSessionRuntimeDiagnostic[] = [
  ...services.diagnostics,
  ...collectSettingsDiagnostics(settingsManager, "runtime creation"),
  ...resourceLoader.getExtensions().errors.map(({ path, error }) => ({
    type: "error" as const,
    message: `Failed to load extension "${path}": ${error}`,
  })),
];
```

这段代码将三个不同来源的诊断信息展开并合并为一个扁平数组：

1. **`...services.diagnostics`** — 来自服务层的诊断信息
2. **`...collectSettingsDiagnostics(...)`** — 来自设置管理器的诊断信息
3. **`...resourceLoader.getExtensions().errors.map(...)`** — 来自扩展加载器的错误，映射为统一的诊断格式

> 没有 `...` 的话，结果会是嵌套数组 `[[...], [...], [...]]`；有了 `...`，结果是一个扁平数组。

---

## 2. 逻辑或运算符 `||`

`||` 的判断规则：**如果左边是"真值（truthy）"，返回左边；否则返回右边。**

### 什么是 falsy 和 truthy？

JavaScript 中以下值被视为 **falsy**（假值），在条件判断中等同于 `false`：

| falsy 值    | 说明         |
| ----------- | ------------ |
| `false`     | 布尔假       |
| `0`         | 数字零       |
| `-0`        | 负零         |
| `0n`        | BigInt 零    |
| `""`        | 空字符串     |
| `null`      | 空值         |
| `undefined` | 未定义       |
| `NaN`       | 非数字       |

除此之外的所有值都是 **truthy**（真值），包括：`1`、`"hello"`、`[]`（空数组）、`{}`（空对象）、`"0"`、`"false"` 等等。

### `||` 的行为

```ts
// 左边是 truthy → 返回左边
1 || "default"        // 1
"hello" || "default"  // "hello"
[1, 2] || "default"   // [1, 2]

// 左边是 falsy → 返回右边
0 || "default"        // "default"
"" || "default"       // "default"
false || "default"    // "default"
null || "default"     // "default"
undefined || "default" // "default"
```

### 常见用法：设置默认值

```ts
// 如果用户没有输入名字，使用 "匿名"
const name = userInput || "匿名";

// 如果没有配置端口，使用 3000
const port = config.port || 3000;
```

### ⚠️ 陷阱

`||` 对所有 falsy 值一视同仁，可能会产生意外行为：

```ts
// 例1：用户想设置数量为 0，但 || 认为 0 是 falsy，跳过了它
const count = props.count || 10;  // props.count = 0 时，结果是 10 而不是 0！

// 例2：用户想设置空字符串，但 || 认为 "" 是 falsy
const label = props.label || "默认"; // props.label = "" 时，结果是 "默认" 而不是 ""！

// 例3：用户想设置 enabled = false，但 || 认为 false 是 falsy
const enabled = props.enabled || true; // props.enabled = false 时，结果是 true 而不是 false！
```

> 在这些场景下，应该使用 `??` 代替 `||`。

---

## 3. 空值合并运算符（Nullish Coalescing Operator）`??`

`??` 的判断规则：**如果左边是 `null` 或 `undefined`，返回右边；否则返回左边。**

它只关心两个值：`null` 和 `undefined`，其他所有值（包括 `0`、`""`、`false`）都被视为有效值。

### `??` 的行为

```ts
// 左边是 null/undefined → 返回右边
null ?? "default"      // "default"
undefined ?? "default" // "default"

// 左边是其他任何值 → 返回左边（哪怕它是 falsy）
0 ?? "default"         // 0
"" ?? "default"        // ""
false ?? "default"     // false
NaN ?? "default"       // NaN
```

### 项目中的示例

```ts
const modelPatterns = parsed.models ?? settingsManager.getEnabledModels();
```

- `parsed.models` 有值 → 使用它
- `parsed.models` 是 `null` 或 `undefined` → 回退到 `settingsManager.getEnabledModels()`

---

## 4. `??` 与 `||` 全面对比

### 完整对照表

| 左边的值      | `??` 的结果     | `||` 的结果     | 是否不同 |
| ------------- | --------------- | --------------- | -------- |
| `1`           | `1`             | `1`             | 相同     |
| `"hello"`     | `"hello"`       | `"hello"`       | 相同     |
| `true`        | `true`          | `true`          | 相同     |
| `0`           | **`0`**         | **`"default"`** | ⚠️ 不同  |
| `""`          | **`""`**        | **`"default"`** | ⚠️ 不同  |
| `false`       | **`false`**     | **`"default"`** | ⚠️ 不同  |
| `null`        | `"default"`     | `"default"`     | 相同     |
| `undefined`   | `"default"`     | `"default"`     | 相同     |

### 怎么选？

| 场景                                   | 用哪个 | 原因                                |
| -------------------------------------- | ------ | ----------------------------------- |
| 只在 `null`/`undefined` 时提供默认值   | `??`   | 不会误判 `0`、`""`、`false`         |
| 对任何 falsy 值都提供默认值            | `||`   | 把 `0`、`""`、`false` 也视为"没有值" |
| 数量、计数器等可能为 `0` 的值          | `??`   | `0` 是有效值，不应该被覆盖           |
| 布尔开关可能为 `false` 的值            | `??`   | `false` 是有效值，不应该被覆盖       |
| 字符串可能为空 `""` 的值               | `??`   | `""` 是有效值，不应该被覆盖          |
| 短路执行（条件成立才执行右侧函数）     | `||`   | `fn1() || fn2()` — fn1 失败时执行 fn2 |

### 不能混用

`??` 和 `||` 不能直接组合使用，必须加括号明确优先级：

```ts
// ❌ 语法错误
const a = x ?? y || z;

// ✅ 明确优先级
const b = (x ?? y) || z;
const c = x ?? (y || z);
```

### 链式使用

两者都支持链式写法，返回第一个"命中"的值：

```ts
// || 返回第一个 truthy 值
const a = null || 0 || "" || "hello" || "world";
// 结果: "hello"（第一个 truthy）

// ?? 返回第一个非 null/undefined 的值
const b = null ?? undefined ?? 0 ?? "" ?? "hello";
// 结果: 0（第一个非 null/undefined）
```

### 结合可选链 `?.` 使用

`??` 经常和可选链 `?.` 配合使用，安全地访问深层属性并提供默认值：

```ts
// 如果 user 或 user.profile 或 user.profile.name 是 null/undefined，返回 "匿名"
const name = user?.profile?.name ?? "匿名";
```
