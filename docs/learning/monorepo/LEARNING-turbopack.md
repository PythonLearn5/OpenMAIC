# 4. Turbopack 是什么？

[Turbopack doc](https://nextjs.org/docs/app/api-reference/turbopack)

Turbopack 是由 Vercel 开发的 JavaScript / TypeScript 增量打包工具，集成在 Next.js 中，用于加速开发和构建。

简单理解：修改代码后，Turbopack 尽可能只重新处理受影响的部分，减少等待时间。

## 4.1 工作流程

```
Next.js 项目
    │
    ▼
Turbopack
    ├── 解析模块依赖
    ├── 处理 TS / TSX、CSS 等资源
    ├── 增量更新
    └── 生成浏览器可运行的资源
           │
           ▼
      浏览器显示页面
```

## 4.2 核心特点

- 增量编译：尽量只重新处理发生变化的模块。
- 快速热更新（HMR）：修改代码后，浏览器通常无需完整刷新就能看到变化。
- 依赖分析：追踪模块之间的引用关系。
- Next.js 集成：用于 Next.js 的开发和生产构建流程，具体支持情况取决于 Next.js 版本及配置。

## 4.3 在 OpenMAIC 中怎么理解？

例如，OpenMAIC 使用 Next.js 开发：

```
pnpm dev
```

如果项目脚本配置为：

```
{
  "scripts": {
    "dev": "next dev --turbopack"
  }
}
```

那么开发服务器就会使用 Turbopack。

注意：是否启用 Turbopack，要以 OpenMAIC 实际的 `package.json` 和 Next.js 配置为准。 上面只是示例。

## 4.4 与 Rollup 的区别

| 对比   | Rollup                 | Turbopack               |
| ---- | ---------------------- | ----------------------- |
| 定位   | 通用模块打包工具               | 面向增量构建的打包工具             |
| 常见用途 | 构建组件库、工具库              | Next.js 应用开发与构建         |
| 典型优势 | 库打包、多格式输出、Tree Shaking | 增量处理、开发热更新、与 Next.js 集成 |
| 使用方式 | 通常单独配置                 | Next.js 内置集成            |

## 4.5 总结

结合 Monorepo 记忆：

- Monorepo：多个项目放在一个代码仓库。
- pnpm workspace：管理这些项目及其依赖关系。
- Rollup：常用于打包共享库。
- Turbopack：加速 Next.js 应用的开发和构建。

这四者处于不同层次，可以协同工作，并不互相替代。