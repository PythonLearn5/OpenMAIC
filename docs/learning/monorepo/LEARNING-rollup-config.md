# OpenMAIC：`rollup.config.js` 解释说明

## 1. 文件作用

`rollup.config.js` 是 Rollup 的打包配置文件，负责把源码打包成可供其他项目使用的 JavaScript 模块。

OpenMAIC\packages\mathml2omml\rollup.config.js

```
import { nodeResolve } from '@rollup/plugin-node-resolve'

const onwarn = (warning) => {
  // Silence circular dependency warning for moment package
  if (warning.code === 'CIRCULAR_DEPENDENCY') {
    return
  }

  console.warn(`(!) ${warning.message}`)
}

export default {
  input: 'src/index.js',
  output: [
    {
      file: 'dist/index.js',
      format: 'module',
      sourcemap: true
    },
    {
      file: 'dist/index.cjs',
      format: 'commonjs',
      sourcemap: true
    }
  ],
  plugins: [nodeResolve()],
  onwarn
}

```
这份配置的核心功能：

- 指定源码入口。
- 输出两种模块格式：ESM 和 CommonJS。
- 解析第三方依赖。
- 生成 Source Map，方便调试。
- 忽略指定的循环依赖警告。

## 2. 逐段解释

### ① 引入依赖解析插件

```
import { nodeResolve } from '@rollup/plugin-node-resolve'
```

`nodeResolve()` 用于解析 `node_modules` 中的第三方依赖，让 Rollup 能够找到并打包通过 `import` 引入的 npm 包。

### ② 配置警告处理

```
const onwarn = (warning) => {
  if (warning.code === 'CIRCULAR_DEPENDENCY') {
    return
  }

  console.warn(`(!) ${warning.message}`)
}
```

- `CIRCULAR_DEPENDENCY`：循环依赖警告，直接忽略。
- 其他警告：输出到控制台。

注意： 这里忽略的是警告，不代表循环依赖问题已经被修复。

### ③ 指定源码入口

```
input: 'src/index.js'
```

告诉 Rollup 从 `src/index.js` 开始分析依赖关系，并构建整个模块。

可以理解为打包的起点。

### ④ 配置输出文件

```
output: [
  {
    file: 'dist/index.js',
    format: 'module',
    sourcemap: true
  },
  {
    file: 'dist/index.cjs',
    format: 'commonjs',
    sourcemap: true
  }
]
```

同一份源码，打包成两种格式：

| 配置                   | 作用                   |
| -------------------- | -------------------- |
| `dist/index.js`      | 输出 ES Module 格式      |
| `dist/index.cjs`     | 输出 CommonJS 格式       |
| `format: 'module'`   | 使用 ESM 模块语法          |
| `format: 'commonjs'` | 使用 CommonJS 模块语法     |
| `sourcemap: true`    | 生成 Source Map，便于调试源码 |

例如：

```
// ESM
import { foo } from './foo.js'
export { foo }
```

```
// CommonJS
const foo = require('./foo.js')
module.exports = { foo }
```

`.cjs` 扩展名可以明确告诉 Node.js 按 CommonJS 方式处理文件。

### ⑤ 启用插件

```
plugins: [nodeResolve()]
```

启用前面导入的依赖解析插件。

如果源码引用了第三方 npm 包，Rollup 就可以通过该插件解析包的入口。

### ⑥ 注册警告处理函数

```
onwarn
```

把前面定义的 `onwarn` 交给 Rollup 使用，统一控制构建过程中的警告输出。

## 3. 整体执行流程

`src/index.js`

源码入口

Rollup + nodeResolve()

分析依赖、打包源码、处理警告

`dist/index.js`

ESM 格式

`dist/index.cjs`

CommonJS 格式

两种输出均生成 Source Map 文件。

## 4. 学习时重点记住

- Rollup：负责打包 JavaScript 模块。
- `input`：从哪里开始打包。
- `output`：打包后生成什么文件、采用什么模块格式。
- `plugins`：扩展 Rollup 的能力。
- `onwarn`：控制警告的处理方式。
- Source Map：帮助开发者定位打包代码对应的源码。

补充：这份配置只负责构建当前模块，不负责启动 OpenMAIC 的 Web 页面。项目中的 Next.js、Turbopack 等工具属于另一条构建或开发运行链路。