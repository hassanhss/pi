# 解构赋值（Destructuring Assignment）详解

解构赋值是 ES6 引入的语法，可以从**数组**或**对象**中提取值，按照对应位置或属性名赋给变量。

---

## 一、对象解构

### 1. 基本用法

```js
const person = { name: "Hassan", age: 25, city: "Beijing" };

const { name, age, city } = person;
// 等价于：
// const name = person.name;
// const age = person.age;
// const city = person.city;
```

### 2. 重命名变量

用 `属性名: 新变量名` 的形式：

```js
const { name: userName, age: userAge } = person;

console.log(userName); // "Hassan"
console.log(userAge);  // 25
// 注意：此时 name 和 age 变量不存在
```

### 3. 默认值

属性不存在时使用默认值：

```js
const { name, gender = "unknown" } = person;

console.log(gender); // "unknown"（person 中没有 gender 属性）
```

重命名 + 默认值可以组合：

```js
const { name: userName = "Anonymous" } = person;
```

### 4. 嵌套解构

```js
const data = {
  user: {
    profile: {
      email: "test@example.com"
    }
  }
};

const { user: { profile: { email } } } = data;
console.log(email); // "test@example.com"
```

### 5. 剩余属性（Rest Pattern）

```js
const { name, ...rest } = person;

console.log(name); // "Hassan"
console.log(rest); // { age: 25, city: "Beijing" }
```

### 6. 计算属性名

```js
const prop = "name";
const { [prop]: value } = person;

console.log(value); // "Hassan"
```

---

## 二、数组解构

### 1. 基本用法

按位置一一对应：

```js
const colors = ["red", "green", "blue"];

const [first, second, third] = colors;
console.log(first);  // "red"
console.log(second); // "green"
console.log(third);  // "blue"
```

### 2. 跳过元素

用 `,` 占位：

```js
const [first, , third] = colors;
console.log(first); // "red"
console.log(third); // "blue"
// "green" 被跳过
```

### 3. 默认值

```js
const [a, b, c, d = "yellow"] = colors;
console.log(d); // "yellow"（数组中没有第 4 个元素）
```

### 4. 剩余元素

```js
const [first, ...rest] = colors;
console.log(first); // "red"
console.log(rest);  // ["green", "blue"]
```

### 5. 嵌套数组解构

```js
const matrix = [[1, 2], [3, 4]];

const [[a, b], [c, d]] = matrix;
console.log(a, b, c, d); // 1 2 3 4
```

---

## 三、实际应用场景

### 1. 函数参数解构

```js
// 对象参数解构（非常常见）
function greet({ name, age, greeting = "Hello" }) {
  console.log(`${greeting}, ${name}! You are ${age}.`);
}

greet({ name: "Hassan", age: 25 });
// "Hello, Hassan! You are 25."
```

```jsx
// React 中极为常见
function UserCard({ name, email, avatar }) {
  return <div>{name} - {email}</div>;
}
```

### 2. 函数返回值解构

```js
function useState(initial) {
  let value = initial;
  const setValue = (v) => { value = v; };
  return [value, setValue];
}

const [count, setCount] = useState(0);
```

### 3. 交换变量

```js
let a = 1, b = 2;
[a, b] = [b, a];
console.log(a); // 2
console.log(b); // 1
```

### 4. 从 import 中解构

```js
import { useState, useEffect, useCallback } from "react";
```

### 5. 配置项提取

```js
function createServer(config) {
  const { port = 3000, host = "localhost", ssl = false } = config;
  // ...
}

createServer({ port: 8080 }); // host 和 ssl 使用默认值
```

### 6. JSON 响应处理

```js
const response = await fetch("/api/user");
const { data: { id, name }, status } = await response.json();
```

### 7. for...of 中解构

```js
const entries = [["a", 1], ["b", 2], ["c", 3]];

for (const [key, value] of entries) {
  console.log(`${key}: ${value}`);
}

// 也可以用 Map
const map = new Map([["x", 10], ["y", 20]]);
for (const [key, value] of map) {
  console.log(`${key} = ${value}`);
}
```

---

## 四、注意事项

```js
// ❌ 错误：单独一行花括号会被解析为代码块，而非解构
{ a, b } = { a: 1, b: 2 };

// ✅ 正确：用括号包裹
({ a, b } = { a: 1, b: 2 });

// ❌ 已声明的 const 不能重新解构赋值
const x = 1;
const { x } = obj; // SyntaxError: 重复声明

// ❌ 解构 null 或 undefined 会报错
const { a } = null;        // TypeError
const { b } = undefined;   // TypeError

// ✅ 提供默认值保护
const { a = 1 } = null;     // 仍然报错！默认值不能防 null/undefined

// ✅ 正确做法：先给整个表达式默认值
const { a = 1 } = null ?? {};  // a = 1
```

---

## 五、TypeScript 中的类型标注

```ts
const { name, age }: { name: string; age: number } = person;

// 函数参数解构 + 类型
function greet({ name, age }: { name: string; age: number }) {
  console.log(name, age);
}

// 接口结合
interface User {
  name: string;
  email: string;
  role?: string;
}

const { name, email, role = "user" }: User = data;
```

---

## 总结

解构赋值是现代 JS/TS 中最常用的语法之一，核心就是**从对象按属性名提取**或**从数组按位置提取**，配合默认值、重命名、剩余语法，可以写出简洁优雅的代码。
