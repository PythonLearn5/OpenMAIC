# OpenMAIC 项目 `tsconfig.json` 配置说明

该文件是 TypeScript 编译配置，决定 使用什么语言标准、如何检查类型、如何解析模块，以及检查哪些项目文件。

## 1. 编译选项 `compilerOptions`

| 配置                                       | 作用                                            |
| ---------------------------------------- | --------------------------------------------- |
| `target: ES2017`                         | 指定 JavaScript 目标语法版本为 ES2017。                 |
| `lib: ["dom", "dom.iterable", "esnext"]` | 引入浏览器 DOM、可迭代 DOM 对象及最新 ECMAScript API 的类型定义。 |
| `allowJs: true`                          | 允许项目中包含 JavaScript 文件。                        |
| `skipLibCheck: true`                     | 跳过第三方声明文件的类型检查，减少检查时间。                        |
| `strict: true`                           | 启用严格类型检查，减少潜在类型错误。                            |
| `noEmit: true`                           | 不生成编译后的 JavaScript 文件，主要进行类型检查。               |
| `esModuleInterop: true`                  | 改善 CommonJS 与 ES Module 的导入兼容性。               |
| `module: esnext`                         | 保留现代 ES Module 模块语法。                          |
| `moduleResolution: bundler`              | 按现代打包工具的规则解析模块，适合 Next.js 项目。                 |
| `resolveJsonModule: true`                | 允许导入 JSON 文件，并推导其类型。                          |
| `isolatedModules: true`                  | 确保代码适合由单文件方式进行编译转换。                           |
| `jsx: react-jsx`                         | 使用 React 新版 JSX 转换方式，无须手动导入 `React` 才能使用 JSX。 |
| `incremental: true`                      | 保存增量编译信息，加快后续检查。                              |

## 2. Next.js 插件

```
"plugins": [
  {
    "name": "next"
  }
]
```

启用 Next.js 的 TypeScript 语言服务插件，为编辑器提供框架相关的类型检查和开发辅助。

## 3. 路径别名 `paths`

```
"paths": {
  "@/*": ["./*"]
}
```

将 `@/` 映射到项目根目录。

例如：

```
import { foo } from '@/lib/utils';
```

对应项目中的：

```
lib/utils
```

这样可以避免大量使用 `../../..` 等相对路径。

注意：`paths` 主要影响 TypeScript 的模块解析，不会自动改写输出路径。Next.js 通常能够识别这种配置。

## 4. 文件范围 `include`

```
"include": [
  "next-env.d.ts",
  "**/*.ts",
  "**/*.tsx",
  ".next/types/**/*.ts",
  ".next/dev/types/**/*.ts",
  "**/*.mts"
]
```

指定 TypeScript 需要纳入项目检查的文件：

- `next-env.d.ts`：Next.js 相关类型声明。
- `**/*.ts`：所有匹配的 TypeScript 文件。
- `**/*.tsx`：包含 JSX 的 TypeScript 文件。
- `.next/types/**/*.ts`：Next.js 生成的类型文件。
- `.next/dev/types/**/*.ts`：开发环境生成的类型文件。
- `**/*.mts`：使用 `.mts` 扩展名的 TypeScript 模块文件。

## 5. 排除范围 `exclude`

```
"exclude": [
  "node_modules",
  "dist",
  "packages/*/src",
  "packages/docs",
  "openclaw",
  "e2e",
  "render-service"
]
```

这些目录不会作为常规文件扫描的入口。

| 目录               | 排除目的（推测）            |
| ---------------- | ------------------- |
| `node_modules`   | 避免检查第三方依赖源码。        |
| `dist`           | 排除构建产物。             |
| `packages/*/src` | 排除指定子包源码，可能由独立配置管理。 |
| `packages/docs`  | 排除文档相关代码。           |
| `openclaw`       | 排除独立模块或外部集成代码。      |
| `e2e`            | 排除端到端测试代码。          |
| `render-service` | 排除独立渲染服务代码。         |

注意： `exclude` 并不保证这些目录中的文件绝对不会被检查。如果其他文件通过 `import` 引用了其中的文件，TypeScript 仍可能将它们纳入程序。

## 6. 学习重点

建议重点理解以下四项：

1. `strict: true`：严格类型检查如何帮助发现代码错误。
2. `moduleResolution: bundler`：Next.js 如何解析依赖模块。
3. `paths`：如何使用路径别名组织大型项目代码。
4. `include` 与 `exclude`：如何控制类型检查的文件范围。

总结： 这份配置为 OpenMAIC 提供了严格的 TypeScript 类型检查、现代模块解析、React JSX 支持和根目录路径别名，同时排除部分独立模块和测试目录，减少主项目的检查负担。