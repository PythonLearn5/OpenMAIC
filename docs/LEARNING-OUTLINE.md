# OpenMAIC 主要技术点 · 学习大纲

> 职责说明：本文档按技术领域枚举 OpenMAIC 的核心技术点，给出「技术说明」「学习建议」「项目落地位置」与「入门阅读顺序」，帮助贡献者分阶段上手。

---

## 目录

- [0. 学习路线总览](#0-学习路线总览)
- [1. 基础开发栈（TypeScript / Node / Next.js / React 19）](#1-基础开发栈typescript--node--nextjs--react-19)
- [2. 构建 & Monorepo（pnpm workspace / Rollup / Turbopack）](#2-构建--monorepopnpm-workspace--rollup--turbopack)
- [3. 前端 UI（Tailwind v4 / shadcn/ui / Radix / CSS 变量主题）](#3-前端-uitailwind-v4--shadcnui--radix--css-变量主题)
- [4. 状态管理（Zustand / Immer / Context / Selectors）](#4-状态管理zustand--immer--context--selectors)
- [5. 测试体系（Vitest / Playwright / fake-indexeddb）](#5-测试体系vitest--playwright--fake-indexeddb)
- [6. LLM 应用开发（Vercel AI SDK / @earendil-works / LangGraph）](#6-llm-应用开发vercel-ai-sdk--earendil-works--langgraph)
- [7. Prompt 工程（模板加载器 / Snippets / Markdown Prompt）](#7-prompt-工程模板加载器--snippets--markdown-prompt)
- [8. 多智能体编排（PI Director / PBL v2 / Orchestration Summarizer）](#8-多智能体编排pi-director--pbl-v2--orchestration-summarizer)
- [9. Provider 中立抽象（LLM / TTS / ASR / 图像 / 视频 / 搜索 / PDF / Storage）](#9-provider-中立抽象llm--tts--asr--图像--视频--搜索--pdf--storage)
- [10. 数据契约（Zod / TypeBox / @openmaic/dsl 判别联合）](#10-数据契约zod--typebox--openmaicdsl-判别联合)
- [11. 播放与动作引擎（状态机 / Choreography 时序 / Action 协议）](#11-播放与动作引擎状态机--choreography-时序--action-协议)
- [12. 渲染内核（@openmaic/renderer · Canvas 元素 / 特效叠加层）](#12-渲染内核openmaicrenderer--canvas-元素--特效叠加层)
- [13. 编辑内核（@openmaic/editor · Edit Intent / 事务模型 / 富文本 ProseMirror）](#13-编辑内核openmaiceditor--edit-intent--事务模型--富文本-prosemirror)
- [14. 白板系统（Store Fold / Validate / 视图投影 / 账本）](#14-白板系统store-fold--validate--视图投影--账本)
- [15. 课程生成（两阶段管线：Outline → Scene / Retry / JSON Repair）](#15-课程生成两阶段管线outline--scene--retry--json-repair)
- [16. Pro 工作台 Durable Agent（租约 / 事件溯源 / 幂等工具 / SSE）](#16-pro-工作台-durable-agent租约--事件溯源--幂等工具--sse)
- [17. 持久化（IndexedDB / Dexie vs PostgreSQL / 双层切换 / 资产生命周期）](#17-持久化indexeddb--dexie-vs-postgresql--双层切换--资产生命周期)
- [18. 多媒体（音频 TTS/ASR / 图像视频生成 / AssetPool）](#18-多媒体音频-ttsasr--图像视频生成--assetpool)
- [19. 文档 & 材料解析（PDF / DOCX / PPTX / MinerU / AliDocMind / 提取管线）](#19-文档--材料解析pdf--docx--pptx--mineru--alidocmind--提取管线)
- [20. 导入导出（PPTX / ZIP / 交互式 HTML / 视频 Hyperframes）](#20-导入导出pptx--zip--交互式-html--视频-hyperframes)
- [21. 检索增强 RAG（Chunking / Ingest / 词法索引）](#21-检索增强-ragchunking--ingest--词法索引)
- [22. 联网搜索（Query Builder / Provider 聚合 / 结果标准化）](#22-联网搜索query-builder--provider-聚合--结果标准化)
- [23. i18n 多语言（i18next / 12 语种 / CI Key 检查）](#23-i18n-多语言i18next--12-语种--ci-key-检查)
- [24. 安全防护（SSRF Guard / SSRF 重定向链 / 访问令牌 / 限流 / 资产所有权）](#24-安全防护ssrf-guard--ssrf-重定向链--访问令牌--限流--资产所有权)
- [25. DevOps & 部署（Docker / docker-compose / Vercel / 健康检查）](#25-devops--部署docker--docker-compose--vercel--健康检查)
- [26. 项目脚本与 CI（pnpm scripts / mjs 脚本 / GitHub Actions）](#26-项目脚本与-cipnpm-scripts--mjs-脚本--github-actions)

---

## 0. 学习路线总览

建议按阶段推进，避免一次性陷入细节：

| 阶段 | 目标 | 章节 | 预估投入 |
| --- | --- | --- | --- |
| **Phase 1**：能跑起来 & 改个按钮 | 熟悉构建、UI、状态、路由 | 1~5 | 1 天 |
| **Phase 2**：能改播放/生成逻辑 | 掌握 DSL、播放引擎、生成管线 | 10~12、15 | 2~3 天 |
| **Phase 3**：能加 Provider / 工具 | 熟悉抽象层与测试 | 9、18、22 | 1~2 天 |
| **Phase 4**：能加新智能体场景 | PI + PBL + Durable Runner | 6~8、16 | 3~5 天 |
| **Phase 5**：贡献到 SDK 包 | Renderer / Editor / DSL / Storage | 12~13、17、20 | 一周+ |
| **Phase 6**：架构级贡献 | 持久化、安全、可观测 | 17、24、25 | 按专题深入 |

---

## 1. 基础开发栈（TypeScript / Node / Next.js / React 19）

### 技术说明

| 条目 | 版本 / 要点 |
| --- | --- |
| **Node.js** | `>=22.19`：使用原生 `fetch`、AbortSignal、ESM-only 生态 |
| **TypeScript** | v5 + 严格模式；分配置：`tsconfig.json`(开发) vs `tsconfig.build.json`(生产构建) |
| **React** | 19.2.3：服务器组件默认 + `useActionState` / `ref` 返回值变化 |
| **Next.js** | 16.3.3：App Router + Turbopack + Route Handlers + Server Actions；React Server Component (RSC) 默认 |

### 学习建议

1. 先读 [`next.config.ts`](../next.config.ts) 与 [`tsconfig.json`](../tsconfig.json) 理解构建约束。
2. 理解 Next.js App Router 的文件命名约定：`page.tsx` / `layout.tsx` / `route.ts` / `loading.tsx`。
3. React 19 重点：`use()` Promise 风格、Server Actions 参数结构、无 `forwardRef`。
4. Node 22：留意 `undici` 作为 `fetch` 底层，因此本项目直接复用 pinned dispatcher（见 §24）。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Node 版本锁定 | [`.nvmrc`](../.nvmrc) |
| 引擎 & 脚本 | [`package.json`](../package.json) |
| Next 构建配置 | [`next.config.ts`](../next.config.ts) |
| TS 配置 | [`tsconfig.json`](../tsconfig.json) / [`tsconfig.build.json`](../tsconfig.build.json) |
| 路由入口 | [`app/page.tsx`](../app/page.tsx) / [`app/layout.tsx`](../app/layout.tsx) |
| Route Handlers 范例 | [`app/api/stages/`](../app/api/stages/) / [`app/api/agent/`](../app/api/agent/) |

### 入门阅读顺序

`next.config.ts` → `app/layout.tsx` → `app/page.tsx` → `components/header.tsx` → `lib/logger.ts`

---

## 2. 构建 & Monorepo（pnpm workspace / Rollup / Turbopack）

### 技术说明

- **pnpm workspace**：根 `pnpm-workspace.yaml` 声明工作区，`@openmaic/*` 包通过 `workspace:*` 互相引用。
- **后安装构建链**：`postinstall` 依次 build `mathml2omml` → `pptxgenjs` → `@openmaic/{dsl,generation,storage,importer,renderer,editor}` → sync importer vendor。
- **每个 SDK 独立 Rollup 构建**：保证产物可脱离 Next.js 独立发布。
- **Next 16 默认 Turbopack**：`next dev` 使用 Turbopack，`next build` 仍用 Webpack。

### 学习建议

1. 先运行一次 `pnpm install` 观察 `postinstall` 日志，理解包依赖顺序。
2. 打开任意 `packages/@openmaic/*/package.json`，阅读 `scripts.build`，理解 Rollup 的 `rollup.config.ts`。
3. 修改 `@openmaic/dsl` 后重新 `pnpm run postinstall` 是重建依赖链的最快方式。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 工作空间声明 | [`pnpm-workspace.yaml`](../pnpm-workspace.yaml) |
| 后安装脚本 | [`package.json#L9-L34`](../package.json) |
| 工作空间包清单 | `packages/@openmaic/{dsl,generation,storage,importer,renderer,editor}` |
| importer 同步脚本 | [`scripts/sync-maic-importer.mjs`](../scripts/sync-maic-importer.mjs) |
| Rollup 配置参考 | 各 `@openmaic/*/rollup.config.ts` |

---

## 3. 前端 UI（Tailwind v4 / shadcn/ui / Radix / CSS 变量主题）

### 技术说明

| 条目 | 作用 |
| --- | --- |
| **Tailwind CSS v4** | 零 JS 配置，通过 `@import "tailwindcss"` 引入；设计 token 走 CSS 变量 |
| **shadcn/ui v3** | 代码拷贝式组件库（`shadcn add xxx`），清单在 `components.json` |
| **Radix UI** | 无障碍原语（Checkbox / Popover / Slider / Switch …），被 shadcn/ui 消费 |
| **next-themes** | 深浅模式切换，底层改 `data-theme` 属性 |
| **CSS 变量主题** | 颜色、圆角、阴影等全部走 `:root { --color-* }`，在 [`app/globals.css`](../app/globals.css) 中定义 |
| **工具类组合** | `clsx + tailwind-merge + class-variance-authority (CVA)` → 统一入口 `lib/utils/cn.ts` |
| **Lucide** | 图标集，统一用 `lucide-react` |
| **动画** | `motion`（Framer Motion 新一代）+ `animate.css` + `tw-animate-css` |
| **sonner** | 全局 Toast 通知 |

### 学习建议

1. 先从 `cn()` 辅助函数入手，任何带样式的组件都应经过它拼接。
2. 新组件先在 `components/ui/` 搜索是否已有对应 shadcn 原语，**不要自行造轮子**。
3. 配色：硬编码颜色 → 搜 `--color-` 选对应语义变量（primary / secondary / muted / accent / danger）。
4. 主题：读 `configs/theme.ts` + `app/globals.css`，了解 CSS 变量命名约定（BEM 风格类名 + 语义化 token）。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| `cn()` 工具 | [`lib/utils/cn.ts`](../lib/utils/cn.ts) |
| 全局样式 & 主题变量 | [`app/globals.css`](../app/globals.css) |
| 主题配置 | [`configs/theme.ts`](../configs/theme.ts) |
| shadcn 清单 | [`components.json`](../components.json) |
| UI 原语目录 | [`components/ui/`](../components/ui/) |
| 图标库 | 全局搜索 `from 'lucide-react'` |
| 字体配置 | [`configs/font.ts`](../configs/font.ts) |
| 快捷键配置 | [`configs/hotkey.ts`](../configs/hotkey.ts) |
| 颜色选择器组件 | 搜 `react-colorful` 用法 |

---

## 4. 状态管理（Zustand / Immer / Context / Selectors）

### 技术说明

- **Zustand 5**：主状态容器，每个子域独立 store，不使用 Redux。
- **Immer**：对「深嵌套不可变更新」用 `produce` 语法糖包裹 reducer。
- **React Context**：只用于 `SceneData`（按场景切换）和 `MediaStageContext`（媒体播放器桥）等**窄范围作用域**，不用来做全局 store。
- **create-selectors**：自研 `lib/utils/create-selectors.ts` 自动为 Zustand store 生成类型安全的点式 selector（`.use.x()`）。
- **持久化**：`lib/store/kv-persist.ts` 给 settings / stage store 加 IndexedDB 持久化（支持作用域版本）。

### 学习建议

1. 先读 [`lib/store/index.ts`](../lib/store/index.ts) 拿到核心 store 列表。
2. 任选一个 store（如 `lib/store/settings.ts`）看它：状态 → actions → persist middlewares → selectors 四段式结构。
3. 读 `lib/utils/create-selectors.ts` 理解 `.use.xxx()` 是如何自动生成的，避免性能陷阱（组件粒度越大，selector 越具体）。
4. 新增全局状态：**先确认不能放在 React local state / Context**，然后再开 Zustand store。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Store 出口 | [`lib/store/index.ts`](../lib/store/index.ts) |
| 每个 Store 文件 | [`lib/store/*.ts`](../lib/store/)（`canvas / stage / snapshot / keyboard / settings / …`） |
| Selector 生成器 | [`lib/utils/create-selectors.ts`](../lib/utils/create-selectors.ts) |
| KV 持久化中间件 | [`lib/store/kv-persist.ts`](../lib/store/kv-persist.ts) |
| Scene 作用域 Context | [`lib/contexts/scene-context.tsx`](../lib/contexts/scene-context.tsx) |
| 自定义 Hooks | [`lib/hooks/`](../lib/hooks/)（55+ 个，可按名称搜） |

---

## 5. 测试体系（Vitest / Playwright / fake-indexeddb）

### 技术说明

| 条目 | 用途 |
| --- | --- |
| **Vitest v4** | 单元测试；`test/` 目录结构与 `lib/` 一一对应 |
| **fake-indexeddb** | 让 IndexedDB 在 Node 端可跑，服务于 `storage` 层测试 |
| **pglite（仅特定用例）** | `@electric-sql/pglite`：在内存跑 Postgres 兼容用例，`.pg.test.ts` 后缀 |
| **Playwright** | E2E：`test:e2e` 跑首页 + 课堂播放路径 |
| **tsx eval 脚本** | 离线评估：PBL Planner / 白板布局 / 大纲语言 / 编排能力，生成可对比数据 |
| **CI Gate**：i18n key 一致性 / 包版本 / Node 引擎 | 用 mjs 脚本在 build 前阻断 |

### 学习建议

1. 改一个模块必须先跑对应 test：`pnpm vitest tests/<对应路径>`。
2. 新增 provider / 工具必须加单测，推荐「先写一个最小失败 case → 改代码 → 通过」。
3. `.pg.test.ts` 说明依赖 Postgres 行为（SELECT FOR UPDATE、事务隔离），本地必须可用才能跑。
4. 想测 UI 组件 → 先看 `tests/components/` / `tests/edit/` 已有的模式。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Vitest 配置 | [`vitest.config.ts`](../vitest.config.ts) |
| 测试根目录（300+ 测试文件） | [`tests/`](../tests/) |
| 测试环境准备 | [`tests/setup-env.ts`](../tests/setup-env.ts) |
| E2E 测试 | [`e2e/`](../e2e/) |
| 评估脚本 | [`eval/`](../eval/) |
| 质量脚本 | [`scripts/check-i18n-keys.mjs`](../scripts/check-i18n-keys.mjs) / `check-node-engine-*` / `check-package-*` |

---

## 6. LLM 应用开发（Vercel AI SDK / @earendil-works / LangGraph）

### 技术说明

| 条目 | 作用 |
| --- | --- |
| **`ai` (Vercel AI SDK 6.x)** | 统一 `generateText / streamText / embed` API；对 OpenAI / Anthropic / Google / Azure / Bedrock 等提供通用适配 |
| **`@ai-sdk/*` provider 家族** | 每个云厂商一个包，本项目统一走 `lib/ai/providers.ts` 再封装 |
| **`@earendil-works/pi-agent-core` / `pi-ai`** | Durable Agent 框架：会话管理、工具调用、思考流、Director 循环；Pro 工作台 Runner 的主依赖 |
| **`@langchain/langgraph` + `@langchain/core`** | 用于 Orchestration Layer：`director-graph.ts` 定义多智能体状态图（可选择路径） |
| **`@modelcontextprotocol/sdk`** | MCP 适配预留（用于工具标准协议化） |
| **`copilotkit`** | 旧版 Copilot 集成（保留兼容，新代码用 @earendil） |

### 学习建议

1. 先学 Vercel AI SDK 基础：`streamText({ model, messages, tools, … })` 返回 `textStream / toolCalls`。
2. 再读 `lib/ai/providers.ts` 看项目如何给 AI SDK 加了「全局强制开关 / API Key 校验 / pinned dispatcher / 思考选项」。
3. Pro 工作台方向：读 `lib/server/agent-runtime/runner.ts` 理解如何把 @earendil Session 包装成「租约 + 事件流」。
4. LangGraph 方向：读 `lib/orchestration/director-graph.ts` 看状态机节点定义。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Provider 总封装 | [`lib/ai/providers.ts`](../lib/ai/providers.ts) |
| 模型别名 & 元数据 | [`lib/ai/model-aliases.ts`](../lib/ai/model-aliases.ts) / [`lib/ai/model-metadata.ts`](../lib/ai/model-metadata.ts) |
| 推理 SSE 解析 | [`lib/ai/reasoning-sse.ts`](../lib/ai/reasoning-sse.ts) |
| 思考上下文 / 思考配置 | [`lib/ai/thinking-context.ts`](../lib/ai/thinking-context.ts) / [`lib/ai/thinking-config.ts`](../lib/ai/thinking-config.ts) |
| Pro Runner 主循环 | [`lib/server/agent-runtime/runner.ts`](../lib/server/agent-runtime/runner.ts) |
| Director Graph（LangGraph） | [`lib/orchestration/director-graph.ts`](../lib/orchestration/director-graph.ts) |
| 思考选项单测 | [`tests/ai/llm-thinking-options.test.ts`](../tests/ai/llm-thinking-options.test.ts) |

---

## 7. Prompt 工程（模板加载器 / Snippets / Markdown Prompt）

### 技术说明

- **Markdown 原生 Prompt**：不把 Prompt 塞进 JS 字符串，而是以 `.md` 文件存放在 `lib/prompts/templates/`，版本控制友好、可独立审阅。
- **模板层级**：`system.md` + 可选 `user.md`；按场景划分子目录（director / interactive-outlines / task-engine-outlines / web-search-query-rewrite / agent-system-*）。
- **Snippets**：`lib/prompts/snippets/` 存放可复用片段（动作类型、元素类型、白板规则、语音指南）。
- **加载器**：`lib/prompts/loader.ts` 负责「读取 → 片段注入 → 变量替换 → 去重」。
- **PBL / Director Prompt** 进一步走专用 loader：`lib/pbl/v2/prompts/loader.ts`、`lib/chat/pi/prompts.ts`。

### 学习建议

1. **改 Prompt 优先改 `.md` 文件，不改 TypeScript 字面量**。
2. 修改 Prompt 后跑「Prompt 回归测试」：`tests/prompts/` 下的快照测试会在输出变化时报错。
3. 变量替换使用 `{{var}}` 约定，阅读 loader 了解哪些内置变量可用。
4. 想加新 Prompt 家族 → 复制 `templates/xxx/` 子目录结构，在 `lib/prompts/index.ts` 注册入口。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Prompt 根（README 说明约定） | [`lib/prompts/`](../lib/prompts/) |
| Snippets 片段库 | [`lib/prompts/snippets/`](../lib/prompts/snippets/) |
| 模板库 | [`lib/prompts/templates/`](../lib/prompts/templates/) |
| 通用加载器 | [`lib/prompts/loader.ts`](../lib/prompts/loader.ts) |
| PBL v2 Prompt 库 | [`lib/pbl/v2/prompts/`](../lib/pbl/v2/prompts/) |
| PI Director Prompt | [`lib/chat/pi/prompts.ts`](../lib/chat/pi/prompts.ts) |
| Prompt 回归测试 | [`tests/prompts/`](../tests/prompts/) |

---

## 8. 多智能体编排（PI Director / PBL v2 / Orchestration Summarizer）

### 技术说明

| 系统 | 定位 | 角色数 |
| --- | --- | --- |
| **PI Director** | 课堂讨论/白板问答控制器 | 1 Director + N 子 Agent（花名册可配置） |
| **PBL v2** | 项目式学习闭环 | Planner / Instructor / Evaluator / Simulator 4 角色固定 |
| **Orchestration Summarizer** | 通用上下文压缩：对话、代码、白板冲突账本 | 纯工具（非 Agent） |

Director 工具链：`call_agent` / `cue_user` / `read_scene` / `web_search` / `close_session` / `classroom_actions` / `native_whiteboard_*` / `native_spotlight`。

PBL v2 关键概念：**Fold（patch 合并）**、**Drain（输入消费）**、**Kernel Operations**（5 种状态更新原子）。

### 学习建议

1. 入门：先看单测，单测最能说明「输入什么 → 期望输出什么」。
2. PI Director：`tests/lib/chat/pi/` 下任意一个测试跑通，再读 `director-loop.ts` 主循环。
3. PBL v2：按 `types.ts` → `fold.ts` → `drain.ts` → `hydration.ts` → 4 个 agent 文件的顺序阅读。
4. Orchestration：只在要做「上下文压缩/摘要」时才深入，一般不关心内部。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Director 主循环 | [`lib/chat/pi/director-loop.ts`](../lib/chat/pi/director-loop.ts) |
| Director 9 个工具 | [`lib/chat/pi/tools/`](../lib/chat/pi/tools/) |
| Director 单测（推荐起点） | [`tests/lib/chat/pi/`](../tests/lib/chat/pi/) |
| PBL v2 类型 & 运行时 | [`lib/pbl/v2/types.ts`](../lib/pbl/v2/types.ts) / [`lib/pbl/v2/runtime/`](../lib/pbl/v2/runtime/) |
| PBL v2 四 Agent | [`lib/pbl/v2/agents/`](../lib/pbl/v2/agents/) |
| PBL v2 完整评估接口 | [`app/api/pbl/v2/evaluate/route.ts`](../app/api/pbl/v2/evaluate/route.ts) |
| PBL v2 单测 | [`tests/pbl/v2/`](../tests/pbl/v2/) |
| 上下文压缩 | [`lib/orchestration/summarizers/`](../lib/orchestration/summarizers/) |
| PBL Planner 离线评估 | [`eval/pbl-v2-planner/runner.ts`](../eval/pbl-v2-planner/runner.ts) |

---

## 9. Provider 中立抽象（LLM / TTS / ASR / 图像 / 视频 / 搜索 / PDF / Storage）

### 技术说明

所有外部能力都按「**Interface → Registry → Dynamic Resolve**」三层解耦：

- 业务代码永远不直接引用具体供应商 SDK。
- 启动期（`lib/server/config-validation.ts`）校验每个启用 Provider 的配置完整度。
- 运行期按「用户设置 > 环境变量 > 默认回退」三级挑选实现。
- 强制开关：`LLM_OPENAI_ENABLED=false` 这种可在部署时全局强杀某 provider。

### 学习建议

1. 要加一个新能力，优先复制现有最简单的 provider。推荐路径：
   - LLM：复制 `lib/ai/providers.ts` 里 `createOpenAI` 同款写法 → 注册到 `MODEL_REGISTRY`。
   - TTS：看 `lib/audio/tts-providers.ts` 接口 → 模仿 `voxcpm.ts`。
   - 搜索：看 `lib/web-search/index.ts` → 模仿 `brave.ts`。
2. 必须为新 provider 写 enablement / provider-neutrality 单测，见 `tests/providers/`。
3. 供应商出错时要提供有信息量的错误 → `lib/server/llm-error-response.ts` 标准化。

### 落地位置（抽象入口 + 部分代表实现）

| 能力 | 抽象层文件 | 代表实现 |
| --- | --- | --- |
| LLM | [`lib/ai/providers.ts`](../lib/ai/providers.ts) | openai / anthropic / azure / bedrock / google / minimax / xiaomi |
| TTS | [`lib/audio/tts-providers.ts`](../lib/audio/tts-providers.ts) | azure / qwen / minimax / elevenlabs / voxcpm / lemonade / browser |
| ASR | [`lib/audio/asr-providers.ts`](../lib/audio/asr-providers.ts) | azure / funasr / lemonade / browser |
| 图像生成 | [`lib/media/image-providers.ts`](../lib/media/image-providers.ts) | openai / qwen / minimax / grok / comfyui / lemonade / happyhorse |
| 视频生成 | [`lib/media/video-providers.ts`](../lib/media/video-providers.ts) | grok / kling / minimax / seedance / seedream / veo / nano-banana |
| 联网搜索 | [`lib/web-search/index.ts`](../lib/web-search/index.ts) | brave / exa / tavily / baidu / bocha / searxng / doubao / claude / minimax |
| PDF 解析 | [`lib/pdf/pdf-providers.ts`](../lib/pdf/pdf-providers.ts) | 自托管 MinerU / 云端 MinerU / AliDocMind |
| 持久化 KV / 文档 / 资产 | [`packages/@openmaic/storage`](../packages/@openmaic/storage) | `pg.ts`（Postgres）/ `plain-json.ts`（浏览器 kv） |

### 关键配套

| 作用 | 文件 |
| --- | --- |
| 启动期配置校验 | [`lib/server/config-validation.ts`](../lib/server/config-validation.ts) |
| 模型路由 & 强制开关 | [`lib/server/resolve-model.ts`](../lib/server/resolve-model.ts) |
| Provider 中立性测试（禁止业务代码直接 import 具体厂商） | [`tests/providers/provider-neutrality-guard.test.ts`](../tests/providers/provider-neutrality-guard.test.ts) |

---

## 10. 数据契约（Zod / TypeBox / @openmaic/dsl 判别联合）

### 技术说明

| 技术 | 作用 |
| --- | --- |
| **`@openmaic/dsl` 包** | 课程领域唯一真源：`Stage / Scene / SlideContent / QuizContent / InteractiveContent / PBLContent / Action` 全部判别联合 |
| **Zod v4** | 新代码优先用 Zod 做运行时校验；DSL 中的 Schema 很多已迁移 |
| **TypeBox** | 老代码在用的 Schema 定义（`JSON Schema ↔ TS` 双向）；主要在老 API / course-tools |
| **判别联合（Discriminated Union）** | 核心抽象手段：通过 `type` 字段自动窄化，渲染器/编辑器/动作引擎无需 as 断言 |
| **JSON Repair** | LLM 生成的 JSON 常不完整 → `jsonrepair` + `partial-json` 双保险补救 |
| **校验入口**：`lib/document-store/validators.ts`、`lib/runtime/payload-validators.ts` | 所有外部输入（浏览器存、用户上传、Agent 返回）过校验后才能落库 |

### 学习建议

1. 先通读 `@openmaic/dsl` 包的导出结构，尤其 `Scene` / `Action` 两个联合。
2. 理解「判别联合」价值：不要写 `if ((scene as any).content.questions)`，让编译器窄化。
3. 修改 DSL 后必须跑：`postinstall`（重 build SDK）+ 所有涉及 tests。
4. 新增校验器：优先 Zod schema + `safeParse` + 错误映射到 i18n key。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| DSL 真源 | [`packages/@openmaic/dsl`](../packages/@openmaic/dsl) |
| 应用层类型扩宽（兼容老字段） | [`lib/types/stage.ts`](../lib/types/stage.ts) / [`lib/types/action.ts`](../lib/types/action.ts) |
| Document Store 校验 | [`lib/document-store/validators.ts`](../lib/document-store/validators.ts) |
| 运行时输入校验 | [`lib/runtime/payload-validators.ts`](../lib/runtime/payload-validators.ts) |
| JSON Repair 用法 | [`tests/generation/json-repair.test.ts`](../tests/generation/json-repair.test.ts) |
| Zod / TypeBox Schema 范例 | 搜 `z.object(` 与 `Type.Object(` |

---

## 11. 播放与动作引擎（状态机 / Choreography 时序 / Action 协议）

### 技术说明

- **PlaybackEngine**：状态机 `idle → playing → paused → live`，维护 `cursor(sceneId, actionIndex)`。
- **Choreography**：时序常量（`WB_DRAW_MS`、`EFFECT_AUTO_CLEAR_MS`）+ 解析动作时长 + 时间线反推。
- **ActionEngine**：统一执行 `Action` 联合（18 种），是在线（讨论时即时动作）与离线（播放）两条路径的共享层。
- **Action Navigation**：把动作分为「可重放前缀」与「分支点」，支持跳转。
- **Live 打断**：讨论中触发 `DiscussionAction` 会快照光标，结束后恢复。

### 学习建议

1. 先跑通一个最小场景：打开任意课堂 → 点击播放 → 在浏览器调试器打断点在 `ActionEngine.execute`。
2. 读 `engine.ts` 的 switch case 结构，观察每个 Action 的副作用落点。
3. 改 Action 协议要同步改 4 处：DSL → `lib/types/action.ts` → ActionEngine case → Choreography 时长估算。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Playback Engine | [`lib/playback/engine.ts`](../lib/playback/engine.ts) |
| Action Engine | [`lib/action/engine.ts`](../lib/action/engine.ts) |
| Choreography 时序 & 时间线 | [`lib/choreography/timeline.ts`](../lib/choreography/timeline.ts) / [`lib/choreography/timing.ts`](../lib/choreography/timing.ts) |
| Action 18 种类型定义 | [`lib/types/action.ts`](../lib/types/action.ts) |
| 跳转与可重放前缀 | [`lib/playback/action-navigation.ts`](../lib/playback/action-navigation.ts) |
| Action 单测 | [`tests/action/`](../tests/action/) |

---

## 12. 渲染内核（@openmaic/renderer · Canvas 元素 / 特效叠加层）

### 技术说明

- **渲染器 = SlideCanvas（网格 + 容器）+ Base*Element（每个元素一个组件）+ 特效叠加层**。
- **Base 元素体系**（Text/Image/Shape/Chart/Table/Latex/Line/Code/Audio/Video）= 数据驱动的纯展示组件，**不可交互**。
- **特效层绝对定位叠在 Canvas 上**：Spotlight（聚光灯）、Laser（激光笔）、Highlight（高亮）、ZoomWrapper（局部缩放）。
- **元素坐标是相对 Canvas 的归一化 + 实际像素两态**，渲染器只认实际像素，编辑后再归一化存。
- **图表 ECharts**、**数学 KaTeX / Temml**、**代码 Shiki**、**表格自定义 Table** 都作为 Base*Element 插件化接入。

### 学习建议

1. 改一个视觉效果先确定落在哪一层：是元素本身（改 Base*）、全局特效（改 Overlay）、还是容器层布局（改 SlideCanvas）。
2. 用 `tests/slide-renderer/` 跑渲染快照/渲染后属性测试。
3. 想加新的「幻灯片元素类型」→ 先在 DSL 加，再注册 Renderer，再注册 Editor（§13）。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Renderer 包根 | [`packages/@openmaic/renderer`](../packages/@openmaic/renderer) |
| 特效叠加层 | `SpotlightOverlay / LaserOverlay / HighlightOverlay / ZoomWrapper`（在 renderer/react 内） |
| 渲染相关单测 | [`tests/slide-renderer/`](../tests/slide-renderer/) |
| 渲染器架构单测（必读） | [`tests/edit/surfaces/slide/renderer-editor-architecture.test.ts`](../tests/edit/surfaces/slide/renderer-editor-architecture.test.ts) |
| 图表 / 代码 / 数学接入 | 分别搜 `echarts` / `shiki` / `katex` / `temml` 在 renderer 中的用法 |

---

## 13. 编辑内核（@openmaic/editor · Edit Intent / 事务模型 / 富文本 ProseMirror）

### 技术说明

- **Edit Intent 层**：手势 → 归一化 `EditIntent` 对象（Drag / Resize / Rotate / Marquee / CreateLine …）→ 统一 `transaction()` 规约 → 不可变更新 Model。
- **事务模型**：撤销/重做、命令录制、协同 OT 都是在 Intent 层拦截实现，不直接改 DOM。
- **ProseMirror 富文本**：Schema（Nodes/Marks）→ Commands（对齐/缩进/列表/样式）→ Plugins（占位符/光标/快捷键/输入规则）。
- **Scene Editor Registry**：`lib/edit/scene-editor-registry.ts` 按场景类型（Slide / Quiz / Interactive / PBL）注册对应编辑 Surface。
- **Regen Lock**：`lib/edit/regen-lock.ts` 控制「重新生成某 scene」时的编辑冲突锁。

### 学习建议

1. 改编辑交互：先在 `packages/@openmaic/editor` 看 intent 层定义，优先加新 intent 再改 React 层。
2. ProseMirror 修改路径：先扩展 Schema → 加 Command → 加插件/快捷键 → 在工具栏 UI 暴露。
3. 想做撤销/恢复 → 99% 场景已经自动支持，除非你引入了「绕过 transaction 的副作用」。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Editor 包根 | [`packages/@openmaic/editor`](../packages/@openmaic/editor) |
| Edit Intent 与事务 | `editor/src/react/core/intent.ts`、`transaction.ts`（在包内） |
| ProseMirror 集成 | [`lib/prosemirror/`](../lib/prosemirror/)（schema/commands/plugins 三层） |
| Scene 编辑注册表 | [`lib/edit/scene-editor-registry.ts`](../lib/edit/scene-editor-registry.ts) |
| 编辑层配置/桥接 | [`lib/edit/`](../lib/edit/)（30+ 文件） |
| Editor 单测大全 | [`tests/edit/`](../tests/edit/)（round-trip、surfaces/slide、surfaces/quiz 等） |

---

## 14. 白板系统（Store Fold / Validate / 视图投影 / 账本）

### 技术说明

- **白板 = Action 追加日志 + Fold（归约函数）→ 当前快照**。白板从不直接改对象树，每次是 fold 一遍 action list。
- **Validate**：所有 WbDraw*Action 在 fold 前通过 schema 验证，禁止脏数据。
- **视图投影**：`browser-projection.ts` 做「逻辑坐标 ↔ 浏览器坐标」变换，支持缩放/平移。
- **白板账本（Orchestration 层）**：`whiteboard-ledger.ts` 按 Agent 记录绘制，避免讨论时多人重复画同一区域。
- **白板可见性**：`whiteboard-visibility.ts` 控制「只在讨论期显示/永久保留」策略。

### 学习建议

1. 读单测 `tests/lib/whiteboard/runtime-store.test.ts`，理解「push action → fold → expect 快照」的模式。
2. 新增白板对象类型：在 `WbDraw*Action` 加类型 → 扩展 validate → 扩展 store.fold → 扩展投影。
3. 不要手动改白板快照对象，**永远通过 action fold**。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 白板 Store & Fold & Validate | [`lib/whiteboard/runtime/store.ts`](../lib/whiteboard/runtime/store.ts) / [`validate.ts`](../lib/whiteboard/runtime/validate.ts) / [`fold.ts`](../lib/whiteboard/runtime/fold.ts) |
| 浏览器投影 & 视口 | [`lib/whiteboard/runtime/browser-projection.ts`](../lib/whiteboard/runtime/browser-projection.ts) / [`viewport.ts`](../lib/whiteboard/viewport.ts) |
| 白板账本（Orchestration） | [`lib/orchestration/summarizers/whiteboard-ledger.ts`](../lib/orchestration/summarizers/whiteboard-ledger.ts) |
| 白板单测 | [`tests/lib/whiteboard/`](../tests/lib/whiteboard/) |
| 白板历史（Undo/Redo Store） | [`lib/store/whiteboard-history.ts`](../lib/store/whiteboard-history.ts) |

---

## 15. 课程生成（两阶段管线：Outline → Scene / Retry / JSON Repair）

### 技术说明

```
输入 → GenerateOutline → [OutlineItem[]] → SceneBuilder(并行/串行) → Stage
```

- **阶段一（Outline）**：`task-engine-outlines` / `interactive-outlines` Prompt 家族 → 结构化大纲。
- **阶段二（SceneBuilder）**：按大纲项类型（slide/quiz/interactive/pbl）路由到不同 builder。
- **重试策略**：`scene-api-retry-boundary` 对可恢复错误（网络/速率限制/JSON 破损）指数退避。
- **JSON 修复**：`jsonrepair` + `partial-json` 兜底 LLM 半拉子输出。
- **媒体插槽**：`slide-media-slots.ts` 从场景中识别要补图/补视频的位置 → 交给 Media Orchestrator。
- **服务端两层包装**：`app/api/generate/*`（经典）与 `lib/server/agent-runtime/generation-tools.ts`（Pro）都复用同一生成内核。

### 学习建议

1. 入门路径：先跑通首页「最小生成」一次（.env 最少配一个 LLM），再断点进 `scene-builder`。
2. 调 Prompt → 走 §7 流程，改 `.md` 文件 + 跑 prompts 快照测试。
3. 要加新的场景生成器 → 把类型注册到 `SceneContent` 判别联合，再给 SceneBuilder 加路由分支。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 生成 SDK | [`packages/@openmaic/generation`](../packages/@openmaic/generation) |
| 服务端场景生成调度 | [`lib/server/scene-generation.ts`](../lib/server/scene-generation.ts) / [`lib/server/classroom-generation.ts`](../lib/server/classroom-generation.ts) |
| 媒体 Orchestrator | [`lib/media/media-orchestrator.ts`](../lib/media/media-orchestrator.ts) |
| 媒体插槽解析 | [`lib/media/slide-media-slots.ts`](../lib/media/slide-media-slots.ts) |
| 生成管线 API（经典） | [`app/api/generate/`](../app/api/generate/) |
| 生成工具（Pro） | [`lib/server/agent-runtime/generation-tools.ts`](../lib/server/agent-runtime/generation-tools.ts) |
| 生成重试/恢复单测 | [`tests/generation/scene-api-retry-boundary.test.ts`](../tests/generation/scene-api-retry-boundary.test.ts) |

---

## 16. Pro 工作台 Durable Agent（租约 / 事件溯源 / 幂等工具 / SSE）

### 技术说明

- **租约协调（Lease-based Claim）**：Postgres 行锁 `SELECT ... FOR UPDATE SKIP LOCKED`，保证同 Session 单 Runner。
- **事件溯源模型**：`AgentEvent(seq↑)` 不可变追加；客户端 SSE 订阅，按 seq 合并本地视图。
- **幂等工具调用追踪**：`tool-call-integrity.ts` 给每个工具调用算 idempotency key，崩溃恢复时跳过已完成工具。
- **40+ 工具**：课程编辑（curriculum / dsl / course-edit）、生成（generation）、材料（material）、白板/技能、导入 PPTX、WebSearch、VoiceClone、AskUser、FetchURL…
- **Mutation Fence**：任何「写课程」前校验 owner 归属，防止跨租户篡改。
- **Scene 修订单调性**：每次写入 `scene.revision++`，客户端只拉已变更场景。

### 学习建议

1. 这是项目中最复杂的模块，不建议 Phase 1-2 就改。
2. 先从 Runner 单测入手：`tests/agent-runtime/runner-event-order.test.ts`、`runner-tool-call-integrity.test.ts`。
3. 加工具的推荐顺序：复制 `material-tools.ts`（最简单读工具）→ `curriculum-tools.ts`（有写入）→ 看 mutation-fence 如何接入。
4. 想理解 SSE 链路：打开一个工作台会话 → 浏览器 DevTools → 找到 `/api/agent/sessions/:id/events` 流。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| Runner 主实现 | [`lib/server/agent-runtime/runner.ts`](../lib/server/agent-runtime/runner.ts) |
| 工具集合（40+） | [`lib/server/agent-runtime/*-tools.ts`](../lib/server/agent-runtime/) 目录 |
| 幂等追踪 | [`lib/server/agent-runtime/tool-call-integrity.ts`](../lib/server/agent-runtime/tool-call-integrity.ts) |
| 突变围栏（写权限） | [`lib/server/agent-runtime/mutation-fence.ts`](../lib/server/agent-runtime/mutation-fence.ts) |
| 事件总线 & 唤醒 | [`lib/server/agent-runtime/event-notify-bus.ts`](../lib/server/agent-runtime/event-notify-bus.ts) |
| Pro 工作台路由 | [`app/api/agent/`](../app/api/agent/) |
| 前端工作bench逻辑 | [`lib/workbench/`](../lib/workbench/)（50+ 文件） |
| Runner 单测（必读） | [`tests/agent-runtime/`](../tests/agent-runtime/) |

---

## 17. 持久化（IndexedDB / Dexie vs PostgreSQL / 双层切换 / 资产生命周期）

### 技术说明

| 层 | 引擎 | 开关 |
| --- | --- | --- |
| 浏览器层 | IndexedDB（Dexie 封装 + `image-storage.ts` Blob 存储） | 默认 |
| 服务端层 | PostgreSQL（`@openmaic/storage` HTTP 契约实现）+ S3（可选资产重定向） | `NEXT_PUBLIC_PERSISTENCE=1` + `DATABASE_URL` |

- **惰性迁移**：切到服务端层时，每个 course 首次访问自动从 Dexie → Postgres。
- **资产生命周期**：`pending(24h) → commit(live) → grace(1h) → release → delete`，后台每 15 分钟扫一次。
- **配额**：`ASSET_QUOTA_BYTES` 默认 10 GiB，在 SQL 事务内原子校验。
- **所有者绑定**：所有写入通过 `OwnerBoundDocumentStore` 自动带 `owner` 条件，杜绝 SQL 泄漏。

### 学习建议

1. 读 `lib/persistence/bootstrap.ts` 能最快理解「两层如何切换」。
2. 想加新的存储实体 → 先在 `@openmaic/storage` 定义 HTTP 接口，再在 pg.ts 与 plain-json.ts 各实现一份。
3. 资产相关 bug → 先看 `asset-collector-schedule` 日志，再看 `tests/persistence/asset-*.test.ts`。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 持久化引导 & 切换 | [`lib/persistence/bootstrap.ts`](../lib/persistence/bootstrap.ts) |
| 浏览器实现（plain json + KV） | [`lib/persistence/plain-json.ts`](../lib/persistence/plain-json.ts) / [`lib/storage/client.ts`](../lib/storage/client.ts) |
| 存储 HTTP 契约 & Postgres 实现 | [`packages/@openmaic/storage`](../packages/@openmaic/storage) |
| 资产配额 / 收集 / TTL | [`lib/persistence/asset-quota.ts`](../lib/persistence/asset-quota.ts) / [`asset-collector-schedule.ts`](../lib/persistence/asset-collector-schedule.ts) / [`asset-pending-ttl.ts`](../lib/persistence/asset-pending-ttl.ts) |
| 所有者绑定 & 访问控制 | [`lib/persistence/owner-bound-document-store.ts`](../lib/persistence/owner-bound-document-store.ts) / [`document-access.ts`](../lib/persistence/document-access.ts) |
| 持久化 API 路由 | [`app/api/stages/`](../app/api/stages/)、[`app/api/folders/`](../app/api/folders/)、[`app/api/persistence/`](../app/api/persistence/) |
| 持久化单测 | [`tests/persistence/`](../tests/persistence/)、[`tests/document-store/`](../tests/document-store/) |

---

## 18. 多媒体（音频 TTS/ASR / 图像视频生成 / AssetPool）

### 技术说明

- **音频双端**：
  - TTS：分配音色 → 生成 wav/mp3 → `audio-duration.ts` 解析时长 → 写回 `narration.audioRef`。
  - ASR：麦克风音频 → Provider → 文本转录 → 注入聊天气泡或用户输入。
- **图像/视频**：`media-orchestrator.ts` → adapter 抽象 → `asset-pool.ts`（统一分配 id、缓存、失败重试）→ `asset-replacement-events.ts` 通知前端刷新。
- **Agent 音色绑定**：`voice-design.ts` + `agent-voice.ts` 给每个 agent 智能分配/记忆音色。
- **媒体轮询式 Provider**：Kling / Seedance 等长任务视频 → `polled-task.ts` 通用轮询器。

### 学习建议

1. 新 TTS/ASR：先实现 `tts-providers.ts` / `asr-providers.ts` 接口，再加一个 `*.ts` 文件，最后在 UI Settings 暴露。
2. 新图像/视频：`lib/media/adapters/` 新建 adapter → `image-providers.ts` / `video-providers.ts` 注册 → 给 `adapters/` 写单测。
3. 媒体失败排查：先看 `media-failure.ts` 中枚举的错误类型，再开 `?debug_media=1` 查前端日志。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| TTS 抽象 & 实现 & 工具 | [`lib/audio/tts-providers.ts`](../lib/audio/tts-providers.ts) + `lib/audio/*.ts` 其他 |
| ASR 抽象 & 实现 | [`lib/audio/asr-providers.ts`](../lib/audio/asr-providers.ts) + `lib/audio/*asr*` |
| AssetPool & 编排 | [`lib/media/asset-pool.ts`](../lib/media/asset-pool.ts) / [`media-orchestrator.ts`](../lib/media/media-orchestrator.ts) |
| 图像 / 视频 Provider 注册 | [`lib/media/image-providers.ts`](../lib/media/image-providers.ts) / [`video-providers.ts`](../lib/media/video-providers.ts) |
| Adapter 实现目录（重点） | [`lib/media/adapters/`](../lib/media/adapters/)（14 个 adapter 范例） |
| 通用轮询任务 | [`lib/media/polled-task.ts`](../lib/media/polled-task.ts) |
| Agent 音色 | [`lib/audio/agent-voice.ts`](../lib/audio/agent-voice.ts) / [`voice-design.ts`](../lib/audio/voice-design.ts) |
| 媒体单测 | [`tests/media/`](../tests/media/) + [`tests/audio/`](../tests/audio/)（共 90+ 文件） |

---

## 19. 文档 & 材料解析（PDF / DOCX / PPTX / MinerU / AliDocMind / 提取管线）

### 技术说明

```
原始文件(bytes)
  → Extractors (text / pdf / images / local-media / manifest)
  → Transforms (noise-removal / normalize / pipeline)
  → 结构化 CourseMaterials
```

- **支持三档 PDF 解析器**（自托管 MinerU / 云端 MinerU / 阿里云 DocMind）成本与精度不同，用户可配置。
- **本地提取：** `lib/document/extractors/local-media.ts` 可把 docx/pptx 内嵌媒体拆出来。
- **服务端侧：** `lib/server/material-extraction/*` 负责上传、调度、进度通知。
- **MIME 识别**：`lib/document/mime.ts` 做文件类型嗅探（避免伪造后缀）。

### 学习建议

1. 要加一种新的文档类型（如 epub）：
   - 在 `extractors/registry.ts` 注册 extractor；
   - 写文本 + 图片提取；
   - 可选地加一个对应 Transform。
2. 解析失败排查：`scripts/probe-mineru-cloud.mjs` 可离线发一次请求验证云端链路。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 文档提取核心 | [`lib/document/`](../lib/document/)（extractors / transforms / bundle / types） |
| PDF Provider | [`lib/pdf/`](../lib/pdf/)（alidocmind / mineru-cloud / mineru-parser） |
| PPTX 解析 SDK（import 用） | [`packages/@openmaic/importer`](../packages/@openmaic/importer) |
| 服务端材料提取调度 | [`lib/server/material-extraction/`](../lib/server/material-extraction/) |
| 材料上传 & 列表 API | [`app/api/materials/`](../app/api/materials/) |
| PDF 解析验证脚本 | [`scripts/probe-mineru-cloud.mjs`](../scripts/probe-mineru-cloud.mjs) |
| 文档单测 | [`tests/document/`](../tests/document/) |

---

## 20. 导入导出（PPTX / ZIP / 交互式 HTML / 视频 Hyperframes）

### 技术说明

| 方向 | 格式 | 实现 |
| --- | --- | --- |
| 导入 | PPTX | `@openmaic/importer` → OOXML → `SlideContent[]` |
| 导入 | 课堂 ZIP | `lib/import/use-import-classroom.ts`（含校验、合并媒体） |
| 导出 | PPTX | `pptxgenjs` fork + `latex→omml`（mathml2omml）+ svg→png base64 |
| 导出 | 课堂 ZIP | `lib/export/use-export-classroom.ts`（完整重放包） |
| 导出 | 讲稿 | `use-export-script.ts`（Markdown 讲稿大纲 + 旁白） |
| 导出 | 交互式 HTML | `lib/export/html-parser/*`（自研 HTML Lexer+Parser → 内联所有 CDN → 脱机包） |
| 导出 | 视频 MP4 | Stage → Hyperframes 项目（IR + Pass Pipeline）→ `render-service`（Chromium + FFmpeg）输出 MP4 |

### 学习建议

1. PPTX 导入错误 → 先看 importer 包单测，再写个最小 PPTX 复现。
2. 导出 HTML 失败：先跑 `tests/export/html-stringify.test.ts`，定位是 tag 还是 asset。
3. 视频导出链路长，本地调试建议：
   - 先看 `lib/video-export/compile.ts` 生成的 IR；
   - 再用 `render-service` 单独启动容器跑 MP4。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 导入 PPTX / ZIP | [`lib/import/`](../lib/import/) + [`packages/@openmaic/importer`](../packages/@openmaic/importer) |
| 导出 PPTX / ZIP / Script | [`lib/export/`](../lib/export/)（含 `use-export-pptx.ts` / `use-export-classroom.ts` / `use-export-script.ts`） |
| 自研 HTML 解析器 | [`lib/export/html-parser/`](../lib/export/html-parser/)（lexer / parser / stringify / tags） |
| MathML → OMML | [`packages/mathml2omml`](../packages/mathml2omml) |
| PPTX 生成库 fork | [`packages/pptxgenjs`](../packages/pptxgenjs) |
| 视频导出 IR + Pass 管线 | [`lib/video-export/`](../lib/video-export/)（compile / passes / emit-hyperframes） |
| 视频导出前端 UI | [`lib/video-export-app/`](../lib/video-export-app/) |
| 视频渲染微服务 | [`render-service/`](../render-service/) |
| 导入导出单测 | [`tests/export/`](../tests/export/) / [`tests/import/`](../tests/import/) |

---

## 21. 检索增强 RAG（Chunking / Ingest / 词法索引）

### 技术说明

- **Chunking 双策略**：`document.ts`（按段落+语义块）+ `grapheme.ts`（按字素块，适合 CJK 短上下文）。
- **Ingest**：`lib/rag/ingest/document.ts` 负责切块 → 写索引。
- **索引层**：默认 `in-memory-lexical-index`（内存词法 + BM25-like），可替换为向量库（接口已定义）。
- **Provider 层可扩展**：`lib/rag/providers/index.ts` 注册索引实现。

### 学习建议

1. 当前 RAG 以「词法 + 结构化分块」为主，若要换向量，先在 providers 下实现 `VectorIndex` 接口。
2. 先读单测 `tests/rag/*` 了解输入输出。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| RAG 根 | [`lib/rag/`](../lib/rag/) |
| 分块器 | [`lib/rag/chunking/`](../lib/rag/chunking/) |
| 摄取器 | [`lib/rag/ingest/`](../lib/rag/ingest/) |
| 索引 Provider | [`lib/rag/providers/`](../lib/rag/providers/)（默认 `in-memory-lexical-index`） |
| RAG 单测 | [`tests/rag/`](../tests/rag/) |

---

## 22. 联网搜索（Query Builder / Provider 聚合 / 结果标准化）

### 技术说明

- **Query Builder**：`lib/server/search-query-builder.ts` 把自然语言请求改写为 provider-specific 关键词（加 site:、filetype:、语言约束等）。
- **Provider 聚合**：`lib/web-search/index.ts` 支持「单一 provider」或「多路取并集去重」模式。
- **结果标准化**：统一 `SearchResult {title, url, snippet, source, score}`，下游无需区分。
- **SSR 时重写**：Prompt 中有 `web_search` 需求时，先搜索再注入 context，防止幻觉。

### 学习建议

1. 新增搜索 Provider → 复制 `brave.ts` 最短实现，加 `web-search-query-rewrite` prompt 再做改写。
2. 搜索效果不好：先看 query builder 是否把限制条件丢了。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 主入口 + 注册 | [`lib/web-search/index.ts`](../lib/web-search/index.ts) |
| Provider 实现目录（10 个） | [`lib/web-search/`](../lib/web-search/) |
| Query Builder（服务端） | [`lib/server/search-query-builder.ts`](../lib/server/search-query-builder.ts) |
| 查询改写 Prompt | [`lib/prompts/templates/web-search-query-rewrite/`](../lib/prompts/templates/web-search-query-rewrite/) |
| 搜索 API 路由 | [`app/api/web-search/route.ts`](../app/api/web-search/route.ts) |
| 搜索配置校验 | [`lib/server/web-search-config.ts`](../lib/server/web-search-config.ts) |
| 搜索单测 | `tests/server/web-search-config.test.ts` + `tests/agent-runtime/web-search.test.ts` |

---

## 23. i18n 多语言（i18next / 12 语种 / CI Key 检查）

### 技术说明

- **运行时双加载**：课堂 UI locale 与 工作台 UI locale 分离加载（`lib/i18n/index.ts` vs `workbench.ts`）。
- **12 语种**：zh-CN / en-US / ja-JP / ko-KR / ru-RU / ar-SA / pt-BR / es-MX / fr-FR / vi-VN / de-DE / zh-TW。
- **CI Key 检查**：`scripts/check-i18n-keys.mjs` 保证所有 locale key 数一致，缺 key 直接阻断 CI。
- **日期/数字**：`use-i18n.tsx` 封装 locale-aware format。

### 学习建议

1. 加新字符串：只改 `zh-CN.json` → CI 失败，其他语种英文兜底 → 再由译者补齐。
2. 加新语种：复制现有 json 文件，在 `lib/i18n/locales.ts` 注册 locale code。
3. Key 命名遵循 BEM 风格域前缀：`classroom.playback.start` 而非 `playStart`。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| i18n 配置 & 入口 | [`lib/i18n/`](../lib/i18n/)（config / index / types / locales） |
| 课堂语言包（12 份） | [`lib/i18n/locales/`](../lib/i18n/locales/) |
| 工作台语言包（10 份） | [`lib/i18n/workbench-locales/`](../lib/i18n/workbench-locales/) |
| 翻译指南 | [`lib/i18n/TRANSLATION_GUIDE.md`](../lib/i18n/TRANSLATION_GUIDE.md) |
| i18n Key 检查脚本 | [`scripts/check-i18n-keys.mjs`](../scripts/check-i18n-keys.mjs) |
| i18n 相关单测 | [`tests/i18n/`](../tests/i18n/) |

---

## 24. 安全防护（SSRF Guard / SSRF 重定向链 / 访问令牌 / 限流 / 资产所有权）

### 技术说明

| 防护 | 实现位置 | 说明 |
| --- | --- | --- |
| SSRF URL Guard | `lib/server/ssrf-guard.ts` | `ipaddr.js` 拒绝私有网段、元数据端点、未解析主机 |
| 重定向链 SSRF | `lib/server/fetch-with-redirect-validation.ts` | 每次 3xx 仍重新过 SSRF 校验 |
| 超时 / 重试 / UA | `lib/server/fetch-with-timeout.ts` | 给外部调用统一超时 |
| 访问码登录 | `lib/server/access-token.ts` + `edge` | 可发放短期 token，适配 Vercel Edge |
| 限流（访问码等） | `lib/server/attempt-limiter.ts` | 每 IP 每窗口 N 次 |
| Pinned Dispatcher（代理 & 连接复用） | `lib/server/pinned-dispatcher.ts` | undici dispatcher；支持 HTTP 代理（符合您的代理环境偏好） |
| 所有权读写校验 | `lib/persistence/owner-bound-document-store.ts` + `mutation-fence.ts` | 确保跨租户不可串数据 |
| SSR 防护 | `lib/server/ssr-guard.ts` | 敏感 API 禁止被缓存或 SSR 泄漏 |
| iFrame Sandbox | `security/iframe-sandbox.test.ts` | Interactive 嵌入 HTML 强制沙箱 |

### 学习建议

1. 新增任何对外 HTTP 调用：必须走 `proxy-fetch` + `ssrf-guard` 组合，不要裸 `fetch`。
2. 任何用户写 API：先想「owner 校验是否加了」，对照 `with-owner.ts` 的包装器。
3. 配置任何可配置 base_url：都要过「URL Guard 不变式」（见 `tests/server/url-guard-unconditional-invariant.test.ts`）。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| SSRF Guard | [`lib/server/ssrf-guard.ts`](../lib/server/ssrf-guard.ts) |
| 重定向校验版 fetch | [`lib/server/fetch-with-redirect-validation.ts`](../lib/server/fetch-with-redirect-validation.ts) |
| 超时 fetch | [`lib/server/fetch-with-timeout.ts`](../lib/server/fetch-with-timeout.ts) |
| 通用 Proxy Fetch + Dispatcher | [`lib/server/proxy-fetch.ts`](../lib/server/proxy-fetch.ts) / [`pinned-dispatcher.ts`](../lib/server/pinned-dispatcher.ts) |
| 访问令牌 | [`lib/server/access-token.ts`](../lib/server/access-token.ts) / [`access-token-edge.ts`](../lib/server/access-token-edge.ts) |
| 限流 | [`lib/server/attempt-limiter.ts`](../lib/server/attempt-limiter.ts) |
| 服务器鉴权包装器 | [`lib/server/agent-runtime/with-owner.ts`](../lib/server/agent-runtime/with-owner.ts) |
| 安全单测 | [`tests/security/`](../tests/security/) + [`tests/server/ssrf-guard.test.ts`](../tests/server/ssrf-guard.test.ts) + [`tests/server/*security*.test.ts`](../tests/server/) |

---

## 25. DevOps & 部署（Docker / docker-compose / Vercel / 健康检查）

### 技术说明

| 部署方式 | 说明 |
| --- | --- |
| **Dockerfile（主应用）** | Node 22 Slim 基础镜像 → 构建 → next start |
| **docker-compose 3 Profiles**： | |
| - `default` | 仅主应用 |
| - `server-persistence` | 主应用 + Postgres（启用服务端持久化） |
| - `video-export` | 在上面基础上再加 render-service（Chromium + FFmpeg） |
| **Vercel** | `vercel.json`：函数区域、缓存、路由 |
| **健康检查** | `app/api/health/route.ts` + compose 中的 healthcheck |
| **HTTP Range** | `lib/server/http-range.ts` 支持音视频断点续传 |

### 学习建议

1. 本地体验完整服务：`docker compose --profile video-export up`。
2. Vercel 部署：先开启 `NEXT_PUBLIC_PERSISTENCE=1`，再配置 `DATABASE_URL`（建议 Supabase/Neon 托管 PG）。
3. 视频导出慢/失败：查 render-service 日志，通常是字体 / Chromium GPU / FFmpeg codec 问题。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 主应用镜像 | [`Dockerfile`](../Dockerfile) |
| Compose 3 套 profile | [`docker-compose.yml`](../docker-compose.yml) |
| Vercel 配置 | [`vercel.json`](../vercel.json) |
| 健康检查 | [`app/api/health/route.ts`](../app/api/health/route.ts) |
| 渲染微服务 | [`render-service/`](../render-service/) |
| 环境变量模板 | [`.env.example`](../.env.example) |

---

## 26. 项目脚本与 CI（pnpm scripts / mjs 脚本 / GitHub Actions）

### 技术说明

| 脚本类型 | 代表脚本 | 作用 |
| --- | --- | --- |
| 质量检查 | `check:i18n-keys` / `check:node-engine` / `check:package-versions` | CI 阻断器 |
| 生成资源 | `gen:video-export-katex` / `gen:video-export-noto-cjk` / `gen:video-export-noto-script-fonts` | 视频导出字体/KaTeX 资源生成 |
| 冒烟测试 | `generation-node-smoke.mjs` | 无 UI 场景下验证生成链路 |
| 包发布 | `smoke-test-package-tarballs.mjs` + `verify-package-artifacts.mjs` | SDK 包发布前校验 |
| CI 并行 | `scripts/ci-run-parallel.sh` | 工作流分片执行加速 |
| CI 工作流 | `.github/workflows/ci.yml`（见仓库） | PR / main 触发 |

### 学习建议

1. 提交前必跑：`pnpm format && pnpm check:i18n-keys && pnpm test`（跑核心用例）。
2. 想加 CI Gate → 新增 `scripts/check-*.mjs` 加到 `scripts` 中，再更新 `.github/workflows/ci.yml`。

### 落地位置

| 内容 | 位置 |
| --- | --- |
| 所有 npm scripts | [`package.json#L9-L34`](../package.json) |
| 脚本目录（17 个 mjs/sh） | [`scripts/`](../scripts/) |
| CI 合同断言（包/PG） | [`scripts/assert-pg-contract-suites.mjs`](../scripts/assert-pg-contract-suites.mjs) / `assert-vendor-maic-importer.mjs` |

---

## A. 从 0 到贡献者的最短路径（Checklist）

- [ ] Node 22.19+ / pnpm 安装完成 → `nvm use`
- [ ] 复制 `.env.example` → `.env.local`，最少开启 1 个 LLM + 1 个 TTS
- [ ] `pnpm install` 成功（观察 postinstall 8 个包都 build 完）
- [ ] `pnpm dev` 启动 → 首页能打开
- [ ] 用首页生成最小课堂 → 播放页能播放（TTS 成功、图像生成成功各 1 次）
- [ ] 修改 `components/header.tsx` 加一行文字 → 热更新可见
- [ ] 跑 `pnpm vitest tests/utils -t "cn"` 任意单测通过
- [ ] 按 §0 Phase 顺序渐进深入
- [ ] 提 PR 前：`pnpm lint + pnpm format --check + pnpm test`
