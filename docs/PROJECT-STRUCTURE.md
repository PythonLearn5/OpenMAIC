# OpenMAIC 项目结构文档

> 职责说明：本文档描述 OpenMAIC 项目的顶层目录组织、monorepo 工作空间布局，以及各核心目录的职责边界，帮助开发者快速定位代码。

---

## 1. 顶层目录总览

```
OpenMAIC/
├── app/                      # Next.js App Router 前端页面与 API 路由
├── assets/                   # 静态展示资源（README 配图、Banner 等）
├── community/                # 社区文档（飞书社群指南等）
├── components/               # React UI 组件层
├── configs/                  # 前端运行时静态配置（动画、图表、快捷键、主题等）
├── docs/                     # 项目文档（ARCHITECTURE、本文件等）
├── e2e/                      # Playwright 端到端测试
├── eval/                     # 离线评估脚本（PBL v2 Planner、白板布局等）
├── lib/                      # 核心业务逻辑层（非 UI 代码的主要存放处）
├── packages/                 # pnpm workspace — 可独立发布的内部 SDK 包
│   ├── @openmaic/            #   项目核心 SDK 家族
│   ├── docs/                 #   独立部署的文档站点（拥有独立 lockfile）
│   ├── mathml2omml/          #   MathML → Office OMML 转换工具
│   └── pptxgenjs/            #   PPTX 生成库的 fork
├── public/                   # Next.js 静态资源（/avatars、/logos、/vendor）
├── render-service/           # MP4 视频导出微服务（Chromium + FFmpeg）
├── scripts/                  # 项目脚本（i18n 检查、CI 并行、包同步等）
├── skills/                   # OpenClaw / Agent Workbench 技能包
├── tests/                    # Vitest 单元测试（与 lib/ 目录对应）
├── types/                    # 全局 .d.ts 类型声明（如 web-speech）
├── .env.example              # 环境变量模板
├── .nvmrc                    # Node.js 版本锁定（>= 22.19）
├── Dockerfile                # 主应用 Docker 镜像
├── docker-compose.yml        # 默认 + server-persistence + video-export profiles
├── next.config.ts            # Next.js 构建配置
├── package.json              # 根 package（脚本、核心依赖）
├── pnpm-workspace.yaml       # pnpm monorepo 声明
├── tsconfig.json             # 开发期 TS 配置
├── tsconfig.build.json       # 生产构建 TS 配置
└── vitest.config.ts          # Vitest 单测配置
```

---

## 2. 页面与 API 路由层 (`app/`)

基于 Next.js 16 App Router，遵循 **Route Handler** 约定。

| 路径 | 职责 |
| --- | --- |
| `app/page.tsx` | 首页（课程生成入口 + Pro 工作台入口） |
| `app/layout.tsx` | 全局布局（主题、字体、i18n Provider） |
| `app/classroom/[id]/page.tsx` | 课堂播放页（Slide / Quiz / PBL / Interactive） |
| `app/workspace/page.tsx` | 课程库管理页 |
| `app/workbench/new/page.tsx` | Pro 工作台：会话 + 课程编辑双栏 |
| `app/generation-preview/page.tsx` | 生成中预览页 |
| `app/eval/whiteboard/page.tsx` | 白板布局评估页 |
| `app/api/agent/*` | Pro 工作台服务端：会话控制、技能、运行时事件 SSE |
| `app/api/chat/*` | 经典多智能体讨论（SSE 流式）与 PI Director 模式 |
| `app/api/generate/*` | 经典两阶段生成管线（outline → scene、image、TTS、video） |
| `app/api/stages/*` | 所有者作用域的课程 CRUD、清单、场景增量拉取 |
| `app/api/folders/*` | 课程文件夹 CRUD（Pro 工作台侧栏） |
| `app/api/materials/*` | 会话材料上传、列表、下载 |
| `app/api/skills/*` | 用户自定义技能 CRUD |
| `app/api/parse-pdf/route.ts` | PDF / 文档解析接入层 |
| `app/api/quiz-grade/route.ts` | Quiz 客观题自动评分 |
| `app/api/transcription/route.ts` | 语音识别（ASR） |
| `app/api/web-search/route.ts` | 联网搜索聚合（Brave / Baidu / Exa / Tavily …） |
| `app/api/usage/route.ts` | 用量统计（浏览器 KV） |
| `app/api/verify-model/route.ts` | 模型连通性探测 |
| `app/api/azure-voices/route.ts` | Azure TTS 音色列表代理 |
| `app/api/health/route.ts` | 健康检查 |
| `app/api/proxy-media/route.ts` | 代理受 CORS 限制的媒体 URL |

---

## 3. UI 组件层 (`components/`)

与业务逻辑解耦的 React 呈现层，按**域**而非技术栈分组。

```
components/
├── agent/                   # 智能体头像、信息条
├── ai-elements/             # AI 计划可视化（Plan / Task / Tool / Edge / Node）
├── chat/                    # 聊天区域组件
├── edit/                    # 编辑模式容器（ContainBox / StageGrid）
├── roundtable/              # 圆桌辩论视图
├── settings/                # 设置面板（模型、TTS、ASR、媒体 Provider）
├── ui/                      # shadcn/ui 基础原语（Button / Dialog / Input / …）
├── whiteboard/              # SVG 白板渲染容器
├── header.tsx               # 顶部导航
├── stage.tsx                # 舞台容器（播放页主壳）
└── user-profile.tsx         # 用户资料 / 访问码登录
```

---

## 4. 核心业务逻辑层 (`lib/`)

OpenMAIC 最大的目录，承载**全部非 UI 代码**。按领域子模块拆分，每个子模块是一个内聚的包。

### 4.1 核心运行时

| 模块 | 职责 | 关键文件 |
| --- | --- | --- |
| `action/` | **动作执行引擎**：统一 speech / spotlight / laser / whiteboard / widget 动作的执行 | [engine.ts](../lib/action/engine.ts) |
| `playback/` | **播放状态机**：`idle → playing → paused → live`，驱动 ActionEngine | [engine.ts](../lib/playback/engine.ts) |
| `choreography/` | **编排层**：时序常量（`WB_DRAW_MS`、`EFFECT_AUTO_CLEAR_MS`）、光标定位、时间线 | [timeline.ts](../lib/choreography/timeline.ts) |
| `orchestration/` | **多智能体编排**：Director Prompt 构建、摘要器、白板账本、LangGraph 适配 | [types.ts](../lib/orchestration/types.ts) |
| `agent-runtime/` | 智能体生命周期钩子、舞台写入工具集 | [lifecycle.ts](../lib/agent-runtime/lifecycle.ts) |
| `agent/` | 客户端智能体工具（重新生成、大纲解析）、运行时配额/白名单/流式封装 | [runtime/](../lib/agent/runtime/) |
| `chat/` | 课堂讨论消息、讲稿、PI Director 循环、子智能体运行时 | [pi/director-loop.ts](../lib/chat/pi/director-loop.ts) |
| `pbl/v2/` | **项目式学习 v2**：4 智能体（Planner / Instructor / Evaluator / Simulator）、运行时 fold/drain/hydration | [types.ts](../lib/pbl/v2/types.ts) |
| `quiz/` | Quiz 运行时、评分、持久化、视图状态 | [runtime.ts](../lib/quiz/runtime.ts) |
| `whiteboard/` | 白板运行时状态、折叠、视图投影、验证 | [runtime/store.ts](../lib/whiteboard/runtime/store.ts) |

### 4.2 课程生成与编辑

| 模块 | 职责 |
| --- | --- |
| `prompts/` | Prompt 模板加载器（`snippets/` + `templates/` 子目录，Markdown 格式） |
| `document/` | 材料解析：PDF / Word / PPTX → 文本 + 图片 + 音频提取管线 |
| `document-store/` | 文档版本化存储、迁移、规范化、校验 |
| `edit/` | 编辑模式：场景 CRUD、元素操作、幻灯片 Schema、HTML 编辑桥接、再生锁 |
| `api/` | **Stage API Facade**：`stage-api.ts` 统一 canvas/scene/mode/navigation 操作 |

### 4.3 AI 与 Provider 抽象

| 模块 | 职责 |
| --- | --- |
| `ai/` | LLM Provider 抽象层：Azure / Bedrock / 推理 SSE / 思考上下文 / 模型别名 / 思考配置 |
| `audio/` | **TTS & ASR**：Azure / 通义 / MiniMax / VoxCPM / FunASR / Lemonade / ElevenLabs / 浏览器原生 |
| `media/` | **图像 & 视频生成**：OpenAI / Qwen / MiniMax / Grok / Kling / ComfyUI / Seedance / Veo … |
| `pdf/` | PDF 解析：AliDocMind / MinerU（云端） + MinerU（自托管） |
| `web-search/` | 联网搜索聚合：Brave / Exa / Tavily / Baidu / Bocha / SearXNG / Doubao / Claude / MiniMax |
| `rag/` | 检索增强：分块（文档 / 字素）、摄取、内存词法索引 |

### 4.4 服务端与持久化

| 模块 | 职责 |
| --- | --- |
| `server/agent-runtime/` | **Pro 工作台 Durable Runner**：租约协调、技能、材料、课程工具、DSL 工具、生成工具、用户消息、事件总线 |
| `server/material-extraction/` | 服务端材料提取（支持 AliDocMind / 本地 ffmpeg） |
| `server/` | 服务器端基础设施：访问令牌、限流、下载、HTTP Range、模型路由、SSR 防护、渲染服务 |
| `persistence/` | 持久化抽象：引导、资产配额、文档访问、所有者绑定、服务器鉴权 |
| `document-store/` | 文档存储：版本化、迁移、规范化 |
| `runtime/` | 运行时存储：学习者键、配置、负载验证 |
| `storage/` | 客户端存储（IndexedDB 包装） |

### 4.5 前端状态与基础设施

| 模块 | 职责 |
| --- | --- |
| `store/` | **Zustand 状态库**：canvas、stage、snapshot、settings、keyboard、user-profile、media-generation、whiteboard-history … |
| `contexts/` | React Context：Scene / MediaStage |
| `hooks/` | 55+ 自定义 Hooks（主题、i18n、ASR/TTS、草稿缓存、视口、流文本、历史快照 …） |
| `i18n/` | 12 种语言（zh-CN / en-US / ja-JP / ko-KR / ru-RU / ar-SA / pt-BR / es-MX / fr-FR / vi-VN / de-DE / zh-TW） |
| `workbench/` | Pro 工作台前端逻辑：会话、课程标签、技能加载、合成器、工作台会话 |
| `config/` | 特性开关、Token Plan 预设 |
| `constants/` | 生成默认常量、智能体默认值 |
| `utils/` | 通用工具：cn、并发、发射器、几何、HTML、数据库、聊天存储、元素指纹 … |
| `types/` | 跨层共享的 TS 类型（action / chat / edit / export / generation / provider / stage / widgets …） |

### 4.6 导入导出

| 模块 | 职责 |
| --- | --- |
| `export/` | PPTX / 课堂 ZIP / 交互式 HTML 导出；LaTeX→OMML；SVG→Base64；内联资源 |
| `export/html-parser/` | 导出用 HTML Lexer + Parser（离线化 CDN 依赖） |
| `import/` | PPTX / 课堂 ZIP 导入 |
| `video-export/` | Hyperframes 项目编译 → IR → Pass → emit（字幕、几何、时间线、可视化） |
| `video-export-app/` | 前端视频导出 UI：打包 ZIP、封面、选项、渲染回调 |

### 4.7 其他

| 模块 | 职责 |
| --- | --- |
| `prosemirror/` | 富文本编辑器：Schema / Commands / Plugins（基于 ProseMirror） |
| `brand/` | 品牌配置与 Context |
| `buffer/` | 流缓冲工具 |
| `live/` | 实时协作 Server API 预留 |
| `media-parse/` | 媒体解析元数据提取 |
| `usage/` | 用量数据标准化 |
| `interactive/` | 交互式元素引用策略、逻辑视口 |
| `classroom/` | 课堂加载、所有权信号、渐进加载策略 |
| `logger.ts` | 统一日志工厂 |

---

## 5. Monorepo 工作空间 (`packages/`)

通过 [`pnpm-workspace.yaml`](../pnpm-workspace.yaml) 声明。根 `postinstall` 会按顺序构建所有内部包。

### 5.1 `@openmaic/*` SDK 家族

| 包 | 职责 | 对外契约 |
| --- | --- | --- |
| `@openmaic/dsl` | **课程数据契约**：Stage / Scene / Slide / Quiz / Interactive / PBL / Action 类型、校验器、版本号 | TypeScript 类型 + Zod 校验 |
| `@openmaic/renderer` | **幻灯片渲染器**：Canvas 元素（Text / Image / Shape / Chart / Table / Latex / Line / Audio / Video）、特效层（聚光灯/激光/高亮/缩放） | React 组件 `SlideCanvas` |
| `@openmaic/editor` | **幻灯片编辑器**：拖拽/缩放/旋转/框选、ProseMirror 富文本、LaTeX、表格、资源插入工具栏、画布快捷键 | React 组件 `EditableSlideCanvasWithUI` |
| `@openmaic/generation` | **两阶段生成管线**：大纲 → 场景，Prompt 模板、PBL Planner、Scene Builder、重试策略、JSON 修复 | `generateOutline()` / `generateScene()` |
| `@openmaic/importer` | **PPTX 解析库**：OOXML 解包、样式解析、形状/文本/图表/数学序列化，输出为 DSL Slide 结构 | `importPptx()` |
| `@openmaic/storage` | **持久化层 HTTP 契约 + Postgres 实现**：Document Store、Runtime Store、KV Store、Asset Store、Agent Session Store | 完整 HTTP 协议文档 + TypeScript 客户端 |

### 5.2 辅助包

| 包 | 职责 |
| --- | --- |
| `mathml2omml` | KaTeX MathML → Word OMML 格式转换（供 PPTX 导出公式使用） |
| `pptxgenjs` | pptxgenjs 的 fork，适配 OpenMAIC 的导出需求 |
| `docs/` | 独立的文档站点（Next.js + Fumadocs），有独立 `package.json` 与 `pnpm-lock.yaml` |

---

## 6. 微服务与辅助进程

| 目录 | 技术栈 | 职责 | 接入方式 |
| --- | --- | --- | --- |
| [`render-service/`](../render-service/) | Node 22 + Chromium + FFmpeg | 将 Hyperframes 项目渲染为 MP4 视频 | 通过 `RENDER_SERVICE_URL` HTTP 调用；docker-compose `video-export` profile |

---

## 7. 测试与评估

| 目录 | 框架 | 覆盖范围 |
| --- | --- | --- |
| `tests/` | **Vitest** + fake-indexeddb | 与 `lib/` 一一对应的单元测试（AI Provider、音频、编辑、持久化、PBL、Quiz、RAG、白板、Web Search …） |
| `e2e/` | **Playwright** | 首页与课堂播放页的端到端流程 |
| `eval/` | tsx 脚本 | PBL v2 Planner、白板布局、大纲语言质量、编排能力的离线评估 |

---

## 8. 配置约定速查

| 场景 | 文件 |
| --- | --- |
| 环境变量 | `.env.example` → 复制为 `.env.local` |
| 代码风格 | `.prettierrc` + `eslint.config.mjs` |
| 组件库 | `components.json`（shadcn/ui 清单） |
| Docker 部署 | `Dockerfile` + `docker-compose.yml`（支持 3 个 profile） |
| Vercel 部署 | `vercel.json` |
| CI | `.github/workflows/ci.yml` |
| 主题/颜色 | `configs/theme.ts` + `app/globals.css`（CSS 变量） |
| 字体 | `configs/font.ts` |
| 快捷键 | `configs/hotkey.ts` |
| MIME 类型 | `configs/mime.ts` |

---

## 9. 目录导航决策树

```
我要改什么？
│
├─ 用户看到的页面 / 路由？                → app/
├─ 按钮 / 弹窗 / 展示组件？                → components/
├─ 播放状态机 / 动作执行 / 编排时序？      → lib/playback + lib/action + lib/choreography
├─ 多智能体讨论逻辑 / PI Director？        → lib/chat + lib/orchestration
├─ 课程生成 Prompt / Outline / Scene？     → packages/@openmaic/generation
├─ DSL 数据结构 / 类型契约？               → packages/@openmaic/dsl
├─ 幻灯片渲染 / 编辑交互？                 → packages/@openmaic/renderer + editor
├─ AI / 音频 / 图像 / 搜索 Provider？      → lib/ai + lib/audio + lib/media + lib/web-search
├─ Pro 工作台 Durable Agent？              → lib/server/agent-runtime + app/api/agent
├─ 持久化 / PostgreSQL？                   → packages/@openmaic/storage + lib/persistence
├─ 前端全局状态？                          → lib/store + lib/hooks + lib/contexts
├─ 导入 / 导出（PPTX / ZIP / 视频）？      → lib/import + lib/export + lib/video-export + render-service
├─ i18n / 多语言？                         → lib/i18n
└─ 单元测试？                              → tests/（与 lib/ 同构）
```
