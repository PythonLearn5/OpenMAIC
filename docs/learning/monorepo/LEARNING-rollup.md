# 3. Rollup 是什么？

[Rollup doc](https://rollupjs.org/)

Rollup 是一个 JavaScript 模块打包工具，可以将多个模块及其依赖关系处理后，生成可供使用或发布的文件。

简单理解：开发时，代码分散在很多文件中；Rollup 负责把它们组织、打包成目标产物。

## 3.1 工作示例

```
src/
├── index.ts
├── utils.ts
└── components/
    └── Button.tsx
        │
        ▼
      Rollup
        │
        ▼
dist/
├── index.js
└── index.d.ts
```

这是示意结构。实际输出取决于 Rollup 配置和插件。

## 3.2 Rollup 的主要作用

- 模块打包：分析 `import`、`export`，处理模块依赖。
- Tree Shaking：尽可能移除没有使用的代码。
- 多格式输出：可以输出 ESM、CommonJS 等模块格式。
- 插件扩展：通过插件处理 TypeScript、CSS、图片等资源。
- 构建组件库：常用于将共享组件、工具库打包，方便其他项目引用。

## 3.3 常见配置

```
export default {
  input: "src/index.ts",
  output: {
    file: "dist/index.js",
    format: "es"
  }
};
```

| 配置项             | 作用                         |
| --------------- | -------------------------- |
| `input`         | 指定入口文件                     |
| `output.file`   | 指定输出文件                     |
| `output.format` | 指定输出模块格式，`es` 表示 ES Module |

## 3.4 Rollup 与 pnpm workspace、Turbopack 的区别

| 技术             | 主要职责                                  |
| -------------- | ------------------------------------- |
| pnpm workspace | 管理多个项目和包之间的依赖                         |
| Rollup         | 打包 JavaScript / TypeScript 模块，常用于库的构建 |
| Turbopack      | 为 Next.js 等应用提供开发和生产构建相关能力，重点优化构建速度   |

在 Monorepo 中，可以使用 pnpm workspace 管理多个包，再用 Rollup 将共享包打包，供 Web 应用或其他项目使用。

学习重点： Rollup 关注的是模块如何被解析、优化和输出。它不是包管理器，也不是 React 框架。