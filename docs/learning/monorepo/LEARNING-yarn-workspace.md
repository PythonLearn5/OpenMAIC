# Yarn workspace 使用

如果你习惯使用 Yarn，不一定需要切换到 pnpm。Yarn 也支持 Monorepo 和 Workspace，基本思想相同。

## 1. pnpm 与 Yarn 对比

| 功能           | pnpm                    | Yarn                     |
| ------------ | ----------------------- | ------------------------ |
| Monorepo 管理  | 支持                      | 支持                       |
| Workspace 配置 | `pnpm-workspace.yaml`   | 根目录 `package.json`       |
| 安装依赖         | `pnpm install`          | `yarn install`           |
| 运行指定项目       | `pnpm --filter web dev` | `yarn workspace web dev` |
| 本地包引用        | `workspace:*`           | `workspace:*`（现代 Yarn）   |

## 2. Yarn 如何配置 Workspace？

例如，项目结构如下：

```
openmaic/
├── apps/
│   └── web/
├── packages/
│   └── utils/
└── package.json
```

在根目录 `package.json` 中配置：

```
{
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

这样，Yarn 就可以识别这些子项目并统一管理依赖。

## 3. 如何引用本地包？

假设 `packages/utils/package.json`：

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

注意：`workspace:*` 的支持情况取决于 Yarn 版本；现代 Yarn（Berry，通常为 Yarn 2 及以上）支持该协议。Yarn Classic（Yarn 1）有不同的 Workspace 解析规则。

## 4. 常用命令对照

| 操作        | pnpm                          | Yarn                           |
| --------- | ----------------------------- | ------------------------------ |
| 安装依赖      | `pnpm install`                | `yarn install`                 |
| 启动 `web`  | `pnpm --filter web dev`       | `yarn workspace web dev`       |
| 运行构建      | `pnpm --filter web build`     | `yarn workspace web build`     |
| 添加依赖到指定项目 | `pnpm --filter web add react` | `yarn workspace web add react` |

## 5. OpenMAIC 项目应该怎么选？

如果你使用 Yarn，先检查 OpenMAIC 根目录的 `package.json` 和锁文件。

- 有 `pnpm-lock.yaml`、`pnpm-workspace.yaml`：项目预期使用 pnpm。
- 有 `yarn.lock` 和 `workspaces` 配置：通常使用 Yarn。
- 如果项目使用 pnpm 的特定功能或脚本，直接切换 Yarn 可能需要调整配置。

建议：学习阶段可以使用 Yarn 理解 Workspace 的概念；实际安装和运行 OpenMAIC 时，优先遵循项目指定的包管理器。 不要随意混用 `pnpm-lock.yaml` 和 `yarn.lock`，否则容易出现依赖版本不一致的问题。