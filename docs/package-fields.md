# package.json 字段说明

## 包身份

| 字段 | 值 | 含义 |
|------|-----|------|
| `name` | `@earendil-works/pi-coding-agent` | npm 包名，`@earendil-works` 是作用域（scope） |
| `version` | `0.75.5` | 当前版本号 |
| `description` | `Coding agent CLI...` | 包的简要描述，显示在 npm 上 |
| `type` | `module` | 声明这个包使用 ES Module（`import/export`），而非 CommonJS（`require`） |
| `license` | `MIT` | 开源许可证 |
| `author` | `Mario Zechner` | 作者 |
| `keywords` | `["coding-agent", "ai", ...]` | npm 搜索关键词 |
| `repository` | `{ type, url, directory }` | 代码仓库地址，`directory` 指明这个包在 monorepo 中的位置 |

## 入口与导出

| 字段 | 含义 |
|------|------|
| `bin` | 注册 CLI 命令。`"pi": "dist/cli.js"` 表示安装后全局可以用 `pi` 命令，实际执行 `dist/cli.js` |
| `main` | 旧的入口文件约定（向后兼容），`import ... from "@earendil-works/pi-coding-agent"` 会解析到 `dist/index.js` |
| `types` | 旧的类型入口，IDE 用来提供类型提示 |
| `exports` | **现代入口定义**（优先级高于 `main`） |
| `exports["."]` | 默认入口 → `import { main } from "@earendil-works/pi-coding-agent"` |
| `exports["./hooks"]` | 子路径入口 → `import { ... } from "@earendil-works/pi-coding-agent/hooks"` |
| `files` | 发布到 npm 时**只包含**这些文件/目录。`dist`、`docs`、`examples` 等，源码（`src/`）不会发布 |

## 运行环境

| 字段 | 含义 |
|------|------|
| `engines` | 要求 Node.js >= 22.19.0 |

## 自定义字段

| 字段 | 含义 |
|------|------|
| `piConfig` | 包自己的配置，`configDir: ".pi"` 指定配置目录名 |

## 依赖

| 字段 | 含义 |
|------|------|
| `dependencies` | 运行时必需的依赖。安装这个包时会自动安装 |
| `devDependencies` | 仅开发时需要的依赖（测试框架、类型定义、构建工具）。别人 `npm install` 这个包时**不会**安装 |
| `optionalDependencies` | 可选依赖。安装失败不会阻断流程（比如 `@mariozechner/clipboard` 在某些平台不可用也不影响） |
| `overrides` | 强制覆盖子依赖的版本（比如不管间接依赖要哪个版本的 `rimraf`，都强制用 `6.1.2`） |

## scripts

| 脚本 | 作用 |
|------|------|
| `clean` | 删除 `dist` 目录 |
| `build` | 编译 TypeScript + 设置 CLI 可执行权限 + 复制静态资源（主题、图片、HTML 模板等） |
| `build:binary` | 交叉构建所有依赖包后，用 Bun 编译成独立二进制可执行文件 |
| `test` | 运行 vitest 测试 |
| `shrinkwrap` | 生成 `npm-shrinkwrap.json`（锁定精确依赖树） |
| `prepublishOnly` | `npm publish` 前自动执行：清理 → 构建 → 生成 shrinkwrap（npm 生命周期钩子，名字固定，不需要手动运行） |
