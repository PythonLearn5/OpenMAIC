# OpenMAIC：`playwright.config.ts` 配置说明

作用： 配置 Playwright 端到端测试（E2E），自动启动 OpenMAIC、运行浏览器测试，并收集测试失败信息。

## 1. 测试配置

| 配置                    | 说明                         |
| --------------------- | -------------------------- |
| `testDir`             | 测试文件目录：`./e2e/tests`       |
| `fullyParallel: true` | 允许测试用例并行执行                 |
| `forbidOnly`          | CI 环境禁止使用 `.only`，避免遗漏其他测试 |
| `retries`             | CI 失败重试 2 次，本地不重试          |
| `workers`             | CI 使用 2 个工作进程，本地使用默认值      |
| `reporter`            | CI 生成 HTML 报告，本地输出终端列表     |

## 2. 浏览器配置 `use`

| 配置           | 说明                           |
| ------------ | ---------------------------- |
| `baseURL`    | 测试地址：`http://localhost:3002` |
| `trace`      | 首次重试时记录执行轨迹                  |
| `screenshot` | 仅在测试失败时截图                    |

## 3. 浏览器项目 `projects`

```
projects: [
  {
    name: 'chromium',
    use: { ...devices['Desktop Chrome'] },
  },
],
```

使用 Chromium 浏览器，模拟桌面 Chrome 环境执行测试。

## 4. 自动启动服务 `webServer`

| 配置                    | 说明                                 |
| --------------------- | ---------------------------------- |
| `command`             | 本地运行 `pnpm dev`，CI 运行 `pnpm start` |
| `url`                 | 等待 `http://localhost:3002` 可访问     |
| `reuseExistingServer` | 本地复用已有服务，CI 不复用                    |
| `timeout`             | 服务启动等待时间最多 120 秒                   |
| `env`                 | 设置端口及测试所需的功能开关                     |

测试环境变量：

- `PORT=3002`：指定服务端口。
- `NEXT_PUBLIC_MAIC_EDITOR_ENABLED=true`：启用 MAIC Editor。
- `NEXT_PUBLIC_COURSEWARE_REFERENCE_ENABLED=true`：启用课件引用功能。

注意：CI 使用 `pnpm start` 启动已经构建好的应用，因此 `NEXT_PUBLIC_*` 变量需要在之前的生产构建步骤中设置。

## 5. 数据库依赖

OpenMAIC 的 E2E 测试依赖 PostgreSQL：

- 本地：运行 `pnpm db:up`，配置 `DATABASE_URL`。
- CI：启动 PostgreSQL 服务并配置 `DATABASE_URL`。
- 测试数据：通过 `e2e/fixtures/server-seed.ts` 创建课程数据。

## 6. 学习重点

1. `testDir`：测试代码放在哪里。
2. `fullyParallel`、`workers`：如何控制测试并行。
3. `webServer`：如何自动启动被测应用。
4. `trace`、`screenshot`：如何排查测试失败。
5. CI 与本地环境：如何分别运行开发服务和生产服务。

总结： 该文件负责配置 OpenMAIC 的浏览器自动化测试流程，包括测试目录、执行策略、服务启动、数据库依赖和失败诊断。