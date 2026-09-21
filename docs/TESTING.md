# OpenMAIC 测试脚本使用说明（Vitest / Playwright E2E / 评估集）

> 相关文档：
> - 项目结构：[PROJECT-STRUCTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/PROJECT-STRUCTURE.md)
> - 总体架构：[ARCHITECTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/ARCHITECTURE.md)
> - 学习大纲：[LEARNING-OUTLINE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/LEARNING-OUTLINE.md)

---

## 0. 总览：3 类测试/评估 + 4 类质量检查脚本

| 类别 | 目录位置 | 运行框架 | package.json 主命令 |
|---|---|---|---|
| **单元 / 集成 / API / 领域测试**（400+ 用例） | [tests/](file:///D:/GITHUB_chat/OpenMAIC/tests) | Vitest | `pnpm test` |
| **端到端浏览器测试（E2E）** | [e2e/tests/](file:///D:/GITHUB_chat/OpenMAIC/e2e) | Playwright | `pnpm test:e2e` / `pnpm test:e2e:ui` |
| **LLM 质量评估集（Eval）** | [eval/](file:///D:/GITHUB_chat/OpenMAIC/eval) | tsx + 自研 runner | `pnpm eval:*` 系列 |
| 质量检查（格式化 / Lint / i18n / 版本契约） | 根 | Prettier + ESLint + Node 脚本 | `pnpm check` / `lint` / `check:*` |

---

## 1. 先决条件

- **Node.js ≥ 22.19.0**（见 [package.json](file:///D:/GITHUB_chat/OpenMAIC/package.json#L6-L8) `engines.node`）。可用：
  ```
  pnpm check:node-engine
  ```
  在跑测试前先检查当前 Node 版本。
- **包管理器：pnpm**（工作区 workspace 依赖本仓库各子包，不建议用 npm/yarn）。
- **安装依赖**（仅需跑一次，含各子包 build）：
  ```
  pnpm install
  ```
  `postinstall` 会自动触发 [build:packages](file:///D:/GITHUB_chat/OpenMAIC/package.json#L10-L11)，依次构建 `packages/` 下 `mathml2omml / pptxgenjs / @openmaic/dsl / @openmaic/generation / @openmaic/storage / @openmaic/importer / @openmaic/renderer / @openmaic/editor` 8 个子包。
- **（可选）PostgreSQL**：文件名带 `.pg.test.ts` 的用例需要真实 Postgres 连接（默认跳过，仅在 CI `storage-pg-contract.yml` Job 中跑）。本地跑这些时需要设置 `DATABASE_URL=postgres://...`。

---

## 2. Vitest 单元与集成测试（tests/ 目录）

### 2.1 配置说明

Vitest 根配置：[vitest.config.ts](file:///D:/GITHUB_chat/OpenMAIC/vitest.config.ts#L1-L14)

```
include:  tests/**/*.test.ts
setupFiles: tests/setup-env.ts
alias:    @/ → <项目根>
```

⚠️ **默认"密封式"运行（Hermetic）**：见 [tests/setup-env.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/setup-env.ts#L1-L41) — **不自动加载 `.env.local`**，避免开发者本地的 API Key / Base URL 让用例产生"单机有 bug、CI 全绿"的幽灵失败。

| 环境变量 | 作用 | 推荐使用场景 |
|---|---|---|
| `TEST_LOAD_LOCAL_ENV=1` | 加载 `.env.local` 的全部 KV 到 `process.env`（不覆盖 shell 已导出的） | 冒烟测试、需要真实 LLM/TTS/联网搜索端点时的单次本地复现 |
| `DATABASE_URL=postgres://...` | 让 `*.pg.test.ts` 用例连接真实 Postgres | 本地跑 storage/agent-runtime 契约用例 |

### 2.2 常用命令

```
# 1) 一次性跑完所有 tests/**/*.test.ts（CI 与日常验证）
pnpm test

# 2) 开启 watch 模式（开发 / TDD 推荐）
npx vitest

# 3) 跑单个子目录：tests/audio/ 下所有 TTS/ASR 用例
npx vitest run tests/audio

# 4) 跑单个文件：场景 TTS 生成链路
npx vitest run tests/audio/scene-tts.test.ts

# 5) 按名称过滤：只跑描述里含 "voice-clone" 的用例
npx vitest run tests/audio -t voice-clone

# 6) 单个冒烟用例需要真实 LLM 端点：打开本地 .env.local
$env:TEST_LOAD_LOCAL_ENV=1
npx vitest run tests/ai/openai-provider.test.ts
```

### 2.3 tests/ 目录结构（25+ 子领域）

tests 下按领域划分目录，**全部走 Vitest**，每个 `*.test.ts` 对应一条链路或一个模块：

| 目录 | 测什么 | 代表用例 |
|---|---|---|
| [tests/ai/](file:///D:/GITHUB_chat/OpenMAIC/tests/ai) | LLM 提供商适配层 | `openai-provider.test.ts`、`thinking-config.test.ts`、`llm-dispatcher.test.ts` |
| [tests/api/](file:///D:/GITHUB_chat/OpenMAIC/tests/api) | Next.js Route Handlers | `generate-classroom.test.ts`、`verify-model.test.ts`、`stage-api-scene.test.ts` |
| [tests/audio/](file:///D:/GITHUB_chat/OpenMAIC/tests/audio) | TTS + ASR + 声音合成链路 | `azure-tts.test.ts`、`qwen-voice-clone.test.ts`、`funasr-asr.test.ts` |
| [tests/document/](file:///D:/GITHUB_chat/OpenMAIC/tests/document) | PDF/Doc 抽取 + 解析器注册 | `alidocmind.smoke.test.ts`、`mineru-cloud.test.ts`、`extractor-registry.test.ts` |
| [tests/generation/](file:///D:/GITHUB_chat/OpenMAIC/tests/generation) | 课程生成管线 | `foreground-retry.test.ts`、`scene-builder-pbl-v2.test.ts`、`widget-actions-direct-pipeline.test.ts` |
| [tests/edit/](file:///D:/GITHUB_chat/OpenMAIC/tests/edit) | 编辑器操作 / Pro 工作台 UI 状态 | `slide-ops.test.ts`、`slide-edit-session.test.ts`、`scene-timeline.test.ts`、round-trip 子目录 |
| [tests/media/](file:///D:/GITHUB_chat/OpenMAIC/tests/media) | 图片/视频资源编排 + Asset Pool | `media-orchestrator.test.ts`、`asset-pool.test.ts`、`server-backed-media-orchestrator.test.ts` |
| [tests/pbl/v2/](file:///D:/GITHUB_chat/OpenMAIC/tests/pbl/v2) | PBL v2 完整运行时 | `planner.test.ts`、`instructor.test.ts`、`simulator.test.ts`、`workspace.test.ts` |
| [tests/agent-runtime/](file:///D:/GITHUB_chat/OpenMAIC/tests/agent-runtime) | Pro Durable Agent 会话 + 执行器 | `run-start.test.ts`、`runner-tool-timeout-cancel.test.ts`、`event-notify-bus.test.ts` |
| [tests/persistence/](file:///D:/GITHUB_chat/OpenMAIC/tests/persistence) | Postgres 持久化契约（部分 `.pg.test.ts`） | `asset-collector-schedule.test.ts`、`bootstrap.test.ts` |
| [tests/server/](file:///D:/GITHUB_chat/OpenMAIC/tests/server) | 服务端侧安全、SSRF、限流、Model 路由 | `ssrf-guard.test.ts`、`model-routes.test.ts`、`config-validation.test.ts` |
| [tests/lib/](file:///D:/GITHUB_chat/OpenMAIC/tests/lib) | 纯函数/工具/钩子/运行时 | `lib/chat/pi/*`（Pi Director 主循环）、`lib/agent/runtime/*`、`lib/utils/iframe.test.ts` |
| [tests/i18n/](file:///D:/GITHUB_chat/OpenMAIC/tests/i18n) | 多语言 locale 完整性 | `editor-ui-locales.test.ts`、`tts-enablement-locales.test.ts` |
| [tests/security/](file:///D:/GITHUB_chat/OpenMAIC/tests/security) | Iframe 沙箱、XSS 策略 | `iframe-sandbox.test.ts` |
| 其余子目录 | action、chat、classroom、components、config、document-store、eval、export、fixtures、generation-preview、hooks、import、orchestration、packages、playback、prompts、prosemirror、providers、quiz、rag、runtime、scene-renderers、settings、slide-renderer、workflows、lint、setup-env 等 | 按文件名直接映射到被测模块 |

> 目录命名规则：路径 `tests/<子领域>/<被测名>.test.ts` 基本对应源码 `lib/<子领域>/<被测名>.ts`，或者 `app/api/<被测>/route.ts` 之类的 Route Handler。

### 2.4 `.pg.test.ts`：需要真实 Postgres 的契约用例

文件名带 `.pg.test.ts` 的用例，是"对真实 Postgres 行为的契约回归测试"（例如 PG NOTIFY、LISTEN 通道、事务提交可见性）。这些用例默认在 CI `storage-pg-contract.yml` 里跑。本地要跑时：

```
# 起一个临时 PG（docker-compose 默认已包含 postgres 服务）
pnpm up postgres -d  # 或 docker 自己起一个

# 设置 DATABASE_URL，并选跑 storage 契约
DATABASE_URL=postgres://openmaic:password@localhost:5432/openmaic \
  npx vitest run tests/persistence/document-asset-references.pg.test.ts
```

如果只是本地开发、不想依赖 PG：跑 `pnpm test` 时这些用例没 `DATABASE_URL` 会自动抛 `Agent runtime requires DATABASE_URL` 类错；所以 `pnpm test` 默认在"没有配 PG 的机器"上其实只跑了纯内存/非 PG 的子集。

### 2.5 常用 Vitest 进阶参数

```
# 只跑改动过的相关用例（受 git 影响）
npx vitest related src/foo.ts

# 失败时保留 trace 到磁盘
npx vitest run --reporter=verbose --pool=forks

# 查看某个用例的 console.log 并指定并发 worker 数
npx vitest run tests/audio/qwen-voice-clone.test.ts --poolOptions.forks.singleFork=true --reporter=verbose
```

---

## 3. Playwright 端到端测试（E2E）

### 3.1 配置总览

Playwright 根配置：[playwright.config.ts](file:///D:/GITHUB_chat/OpenMAIC/playwright.config.ts#L1-L42)

```
testDir:      ./e2e/tests
baseURL:      http://localhost:3002
projects:     chromium（桌面 Chrome）
fullyParallel: true
retries:      CI=2 / 本地=0
workers:      CI=2  / 本地=默认核数
reporter:     CI=html / 本地=list
trace:        首次失败时录制 trace
screenshot:   失败时保存失败截图
```

### 3.2 启动 Web 应用 + 跑 E2E

Playwright 会自动按 [webServer](file:///D:/GITHUB_chat/OpenMAIC/playwright.config.ts#L23-L41) 段拉起 Next.js：

- **CI 环境**：跑 `pnpm build` 之后再 `pnpm start`（生产构建）
- **本地环境**：直接 `pnpm dev`（开发服务器，热更新）
- 固定端口 `3002`，并强制注入三个 Pro 功能开关（见 [L36-L39](file:///D:/GITHUB_chat/OpenMAIC/playwright.config.ts#L36-L39)）：
  ```
  NEXT_PUBLIC_MAIC_EDITOR_ENABLED=true
  NEXT_PUBLIC_PI_CHAT_ENABLED=true
  NEXT_PUBLIC_COURSEWARE_REFERENCE_ENABLED=true
  ```
  以保证 e2e 里可以覆盖课堂编辑器、Pi 聊天、课件引用这些默认未开的功能。

常用命令：

```
# 1) 一次性跑完全部 E2E（首次运行会自动下载 Playwright 浏览器二进制，比较慢）
pnpm test:e2e

# 2) 交互模式（推荐日常调试）：打开 Playwright UI，逐步回放/选跑
pnpm test:e2e:ui

# 3) 跑单个 E2E spec 文件
npx playwright test e2e/tests/classroom-layout.spec.ts

# 4) 重放某个失败用例的 trace（第一次失败后会生成 test-results/*/trace.zip）
npx playwright show-trace test-results/failing-case/trace.zip
```

> 首次运行 `pnpm test:e2e` 报错缺少浏览器时，跑：
> ```
> npx playwright install chromium
> ```

### 3.3 e2e 现有目录结构

| 目录 | 作用 |
|---|---|
| [e2e/tests/](file:///D:/GITHUB_chat/OpenMAIC/e2e/tests) | 放 `.spec.ts` 测试文件（用例本身） |
| [e2e/pages/](file:///D:/GITHUB_chat/OpenMAIC/e2e/pages) | Page Object Model：`home.page.ts` / `classroom.page.ts` |
| [e2e/fixtures/](file:///D:/GITHUB_chat/OpenMAIC/e2e/fixtures) | base 夹具、mock-api、公共依赖 |

---

## 4. LLM 质量评估集（Eval Runner）

这些不是 pass/fail 的测试，而是"对真实 LLM 做批量评估 + 生成 markdown 报告"，用于回归 prompt/模型路由。**需要真实可用的 LLM API Key**（配合 `.env.local`，通过 shell 变量或 `TEST_LOAD_LOCAL_ENV` 注入）。

### 4.1 Eval 命令清单（全部 `tsx eval/*/runner.ts`）

| package.json 命令 | 入口文件 | 评估对象 |
|---|---|---|
| `pnpm eval:pbl-v2-planner` | [eval/pbl-v2-planner/runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/pbl-v2-planner/runner.ts) | PBL v2 Planner prompt 的可完成度/判断正确性 |
| `pnpm eval:whiteboard` | [eval/whiteboard-layout/runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/whiteboard-layout/runner.ts) | 白板布局渲染正确性 |
| `pnpm eval:outline-language` | [eval/outline-language/runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/outline-language/runner.ts) | 大纲语言、结构、多语言一致性 |
| `pnpm eval:orchestration` | [eval/orchestration/runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/orchestration/runner.ts) | 编排全链路总体得分 |
| `pnpm eval:orchestration:answering` | [eval/orchestration/answering-runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/orchestration/answering-runner.ts) | 编排中回答生成子环节 |
| `pnpm eval:orchestration:answer-content` | [eval/orchestration/answer-content-runner.ts](file:///D:/GITHUB_chat/OpenMAIC/eval/orchestration/answer-content-runner.ts) | 回答内容质量深度评估 |

评估产物（markdown 报告、判定日志）会落到 `eval/<子项>/runs/<时间戳>/`。公共工具位于 [eval/shared/](file:///D:/GITHUB_chat/OpenMAIC/eval/shared)：
- `resolve-model.ts`：评估用哪条 MODEL_ROUTE / DEFAULT_MODEL
- `run-dir.ts`：管理 runs 目录
- `markdown-report.ts`：生成可读 markdown 报告

### 4.2 典型用法

```
# 评估 PBL v2 Planner（先确保有 DEFAULT_MODEL / MODEL_ROUTES）
TEST_LOAD_LOCAL_ENV=1 pnpm eval:pbl-v2-planner

# 只评估某个子场景集（如果 runner 支持 --filter，见具体 runner 参数说明）
TEST_LOAD_LOCAL_ENV=1 pnpm eval:outline-language -- --only zh-CN
```

---

## 5. 其它 test:* 与 check:lint/format/i18n 脚本

### 5.1 一次性冒烟测试

```
# 打包产物 smoke：验证各 workspace 子包 tarball 能被 require() 到
pnpm test:package-tarballs
```

### 5.2 质量检查（pre-commit / CI 常用）

```
# 1) Prettier 格式校验（整仓库）：失败 = 有未格式化文件
pnpm check

# 2) 自动格式化所有可格式化文件（写回）
pnpm format

# 3) ESLint（不自动修复）
pnpm lint

# 4) ESLint（自动修复，能修的自动写回）
pnpm lint --fix

# 5) i18n locale 键完整性检查（新增 UI 必须加中英/多语 key）
pnpm check:i18n-keys

# 6) package 版本 bump 检查：防止发布未改版本号
pnpm check:package-versions
```

CI 顺序（参考 [.github/workflows/ci.yml](file:///D:/GITHUB_chat/OpenMAIC/.github/workflows/ci.yml)）：

```
check:node-engine → prettier check → eslint → check:i18n-keys → pnpm test → pnpm build
```

---

## 6. 调试与故障排查速查表

| 症状 | 可能原因 | 解决 |
|---|---|---|
| `pnpm test` 报 `@openmaic/storage` import 错 | 还没 build packages | 跑一次 `pnpm install` 或 `pnpm build:packages` |
| 测到外部 API 的用例全部红，但浏览器里正常 | `setup-env.ts` 默认没加载 `.env.local` | 跑单个冒烟用例时加 `TEST_LOAD_LOCAL_ENV=1` |
| `*.pg.test.ts` 红、报 `DATABASE_URL` 未配置 | 用例需要真实 PG | 起 Postgres + 设 `DATABASE_URL=postgres://...`，或直接跳过 |
| Playwright 报错 `Executable doesn't exist` | 还没装浏览器 | 跑 `npx playwright install chromium` |
| Playwright 报 `baseURL localhost:3002` 连接不上 | dev server 120s 内没起来（包构建慢 / 端口被占） | 查端口占用、或手动 `PORT=3002 pnpm dev` 之后再跑 playwright（它会 reuseExistingServer） |
| vitest watch 时某用例偶发红、单跑绿 | 用例共用内存 store 互相污染 | 加 `--poolOptions.forks.singleFork=true` 跑单个文件复现；在该用例 `beforeEach` 里 reset 对应 Zustand store |

---

## 7. 新增测试的放置约定（贡献指南）

1. **测试新模块 / 新函数**：放到 `tests/<与源码 lib/ 对齐的子目录>/<被测模块名>.test.ts`。
2. **测试新 Route Handler**：放到 `tests/api/<路由路径 segment>.test.ts`。
3. **需要真实 Postgres 的事务/通知语义**：文件名后缀 `.pg.test.ts`，**不要**在非 PG 环境里也跑（它会抛错阻断普通 `pnpm test`）。
4. **端到端浏览器交互**：放到 `e2e/tests/*.spec.ts`，抽公共操作到 `e2e/pages/*.page.ts`。
5. **评估 prompt/模型质量**：新增一个 `eval/<子项>/runner.ts`，写对应 prompt / fixtures / 判定逻辑，然后在 [package.json scripts](file:///D:/GITHUB_chat/OpenMAIC/package.json#L29-L34) 里加 `eval:*` 脚本。
