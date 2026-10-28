# ES Module（ESM）/ CommonJS（CJS）解释

## 1. 它们是什么？

ESM 和 CommonJS 都是 JavaScript 的模块系统，用于将代码拆分成多个文件，并在文件之间导入、导出功能。

## 2. 语法区别

### ES Module（ESM）

JavaScript 官方标准模块系统，现代前端项目广泛使用。

```
// math.js：导出
export function add(a, b) {
  return a + b
}

// index.js：导入
import { add } from './math.js'

console.log(add(1, 2))
```

常见语法：`import`、`export`。

### CommonJS（CJS）

Node.js 传统的模块系统，在许多旧项目和 Node.js 工具中仍然常见。

```
// math.cjs：导出
function add(a, b) {
  return a + b
}

module.exports = { add }

// index.cjs：导入
const { add } = require('./math.cjs')

console.log(add(1, 2))
```

常见语法：`require()`、`module.exports`。

## 3. 核心区别

| 对比         | ES Module     | CommonJS         |
| ---------- | ------------- | ---------------- |
| 简称         | ESM           | CJS              |
| 导入         | `import`      | `require()`      |
| 导出         | `export`      | `module.exports` |
| 加载机制       | 支持静态分析、异步加载   | 传统上同步加载          |
| 常见场景       | 现代前端、现代 JS 项目 | 传统 Node.js 项目    |
| Node.js 支持 | 支持            | 支持               |

注意：ESM 也可以动态导入（`import()`）；CommonJS 也能在适当环境中使用 ESM 的部分功能，两者并非完全互斥。

## 4. 为什么 Rollup 要输出两种格式？

你在 OpenMAIC 的配置中看到了：

```
output: [
  {
    file: 'dist/index.js',
    format: 'module'
  },
  {
    file: 'dist/index.cjs',
    format: 'commonjs'
  }
]
```

意思是：同一份源码，打包成两种模块格式，以兼容不同的使用环境。

- `index.js`：供支持 ESM 的项目使用。
- `index.cjs`：供使用 CommonJS 的 Node.js 项目使用。

这样，其他开发者引入这个库时，可以根据自己的项目环境选择合适的版本。

## 5. 记住这三个概念

- ESM / CJS：模块规范，规定代码如何导入和导出。
- Rollup：打包工具，可以将源码构建成 ESM、CJS 等格式。
- `.js` / `.cjs`：文件扩展名；在 Node.js 中，模块类型还会受到 `package.json` 中 `"type"` 配置的影响。

可以先记住：ESM 是现代 JavaScript 项目常用的模块标准，CommonJS 是 Node.js 传统模块标准，Rollup 可以同时生成两种格式。