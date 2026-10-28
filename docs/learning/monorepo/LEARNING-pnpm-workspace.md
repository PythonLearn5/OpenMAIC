# 2. pnpm workspace 是什么？

[pnpm workspace doc](https://pnpm.io/workspaces)

pnpm workspace 是 pnpm 提供的多包管理机制，可以在一个 Monorepo 仓库中统一管理多个应用和共享包，并支持它们之间的本地依赖。

简单理解：Monorepo 负责把代码放在一起，pnpm workspace 负责管理这些代码包之间的关系。

## 2.1 目录示例

```
openmaic/
├── apps/
│   └── web/             # Web 应用
├── packages/
│   ├── ui/              # 共享 UI 组件
│   └── utils/           # 共享工具库
├── package.json
├── pnpm-workspace.yaml  # 声明工作区
└── pnpm-lock.yaml       # 锁定依赖版本
```

以上是示意结构，不代表 OpenMAIC 的实际目录。

## 2.2 核心配置

根目录的 `pnpm-workspace.yaml`：

```
packages:
  - "apps/*"
  - "packages/*"
```

含义：

- `apps/*`：将 `apps` 下的项目纳入工作区。
- `packages/*`：将 `packages` 下的共享包纳入工作区。

每个子项目通常都有自己的 `package.json`，定义名称、依赖和脚本。

## 2.3 如何引用共享包？

例如，`packages/utils/package.json`：

```
{
  "name": "@openmaic/utils",
  "version": "1.0.0"
}
```

在 `apps/web/package.json` 中：

```
{
  "dependencies": {
    "@openmaic/utils": "workspace:*"
  }
}
```

`workspace:*` 表示优先引用当前工作区里的本地包，而不是从 npm 仓库下载同名包。

这样，Web 应用就能复用共享工具库的代码。

## 2.4 常用命令

| 命令                                    | 作用                                  |
| ------------------------------------- | ----------------------------------- |
| `pnpm install`                        | 安装工作区依赖                             |
| `pnpm --filter web dev`               | 在名称匹配 `web` 的包中运行 `dev` 脚本          |
| `pnpm --filter @openmaic/utils build` | 构建指定工具包                             |
| `pnpm -r build`                       | 递归执行各包的 `build` 脚本，具体执行顺序取决于依赖关系及配置 |

## 2.5 pnpm workspace 的价值

1. 共享代码：多个应用直接使用同一个组件库或工具库。
2. 统一依赖：减少重复安装，统一管理依赖锁定。
3. 按需运行：可以只启动或构建指定项目。
4. 本地联动：共享包修改后，依赖它的应用可以使用更新后的代码。

记住： `pnpm workspace` 负责管理包和依赖关系，不负责把代码打包成最终产物。打包工作通常由 Rollup、Turbopack 等构建工具完成。