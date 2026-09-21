# OpenMAIC 总体架构设计文档

> 职责说明：本文档从分层架构、核心领域模型、运行模式、关键管线与数据流、扩展机制等维度系统性地描述 OpenMAIC 的架构设计，帮助贡献者理解系统边界与模块协作。

---

## 1. 架构总览

OpenMAIC 是一套**以 AI 为核心的生成式课堂平台**，其架构围绕以下三条设计主线展开：

1. **DSL 优先的契约驱动** — 所有课堂数据以 `@openmaic/dsl` 为唯一真源，渲染、编辑、生成、持久化围绕同一套类型协作。
2. **Provider 中立的可插拔架构** — LLM、TTS、ASR、图像、视频、搜索、存储全部通过抽象层隔离，可在运行时热切换 Provider。
3. **双模式运行时** — 「经典一键生成」与「Pro 工作台 Durable Agent」共享同一套课程数据模型，但拥有独立的生成管线与会话管理。

### 1.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Presentation Layer (UI)                          │
│   Next.js App Router · React Components · Zustand Stores · Hooks     │
│   ─────────────────────────────────────────────────────────────      │
│   app/*  ·  components/*  ·  lib/hooks  ·  lib/store  ·  lib/i18n    │
├─────────────────────────────────────────────────────────────────────┤
│                    Orchestration Layer                                │
│   Playback Engine · Action Engine · Director Loop · PI Agent         │
│   ─────────────────────────────────────────────────────────────      │
│   lib/playback  ·  lib/action  ·  lib/chat/pi  ·  lib/orchestration  │
├─────────────────────────────────────────────────────────────────────┤
│                   Generation & Course Logic                          │
│   Two-Stage Pipeline · PBL v2 Agents · Pro Workbench Runner         │
│   ─────────────────────────────────────────────────────────────      │
│   packages/@openmaic/generation  ·  lib/pbl/v2  ·  lib/server/       │
│                                                   agent-runtime      │
├─────────────────────────────────────────────────────────────────────┤
│                  Domain Contract Layer (DSL)                         │
│   Stage · Scene · Slide · Quiz · Interactive · PBL · Action          │
│   ─────────────────────────────────────────────────────────────      │
│                     packages/@openmaic/dsl                           │
├─────────────────────────────────────────────────────────────────────┤
│                 Rendering & Editing Kernel                           │
│   Slide Renderer · Slide Editor · Whiteboard · Rich Text             │
│   ─────────────────────────────────────────────────────────────      │
│   packages/@openmaic/renderer  ·  packages/@openmaic/editor          │
│   lib/whiteboard  ·  lib/prosemirror                                 │
├─────────────────────────────────────────────────────────────────────┤
│                 Abstraction Layer (Provider-Neutral)                 │
│   AI SDK · Audio · Media · Search · PDF · Storage                   │
│   ─────────────────────────────────────────────────────────────      │
│   lib/ai  ·  lib/audio  ·  lib/media  ·  lib/web-search              │
│   lib/pdf  ·  packages/@openmaic/storage                             │
├─────────────────────────────────────────────────────────────────────┤
│                 Persistence & I/O Layer                              │
│   Browser (IndexedDB/Dexie) · PostgreSQL · S3 · External Providers   │
│   ─────────────────────────────────────────────────────────────      │
│   lib/persistence  ·  lib/storage  ·  lib/document-store             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心领域模型 (`@openmaic/dsl`)

### 2.1 课堂骨架（Stage / Scene）

一切课堂内容以 **Stage → Scene** 的两级树形结构组织：

```
Stage (课堂)
├── id, meta (title, description, locale...)
├── agentRoster (多智能体配置：teacher/student/assistant...)
├── mode (StageMode：slide | edit | whiteboard | live...)
└── scenes[] (有序场景列表)
    └── Scene<Action, SceneContent>
        ├── id, outlineId, order
        ├── type        ← 与 content.type 联动（判别联合）
        ├── content     ← SceneContent 四选一
        │   ├── SlideContent         (幻灯片：Canvas + Element[])
        │   ├── QuizContent          (测验：Question[])
        │   ├── InteractiveContent   (交互式 HTML iframe + WidgetConfig)
        │   └── PBLContent           (项目式学习：projectConfig + projectV2)
        ├── narration   (旁白文字脚本)
        ├── actions[]   (可执行动作列表，见 §2.2)
        └── whiteboard  (共享白板状态)
```

关键设计：

- **判别联合 (Discriminated Union)**：`Scene` 是泛型联合，通过 `type === 'slide' | 'quiz' | 'interactive' | 'pbl'` 自动窄化，编辑器与渲染器无需类型断言。
- **内容与表现分离**：`Scene.content` 保存纯数据，视觉效果由 `Renderer`/`Editor`/`PlaybackEngine` 解释。
- **动作与内容解耦**：`actions[]` 独立于内容存在，同一 Slide 可拥有多套动作编排（不同语速、不同特效）。

参见：[`lib/types/stage.ts`](../lib/types/stage.ts)（应用层对 DSL 的扩宽定义）。

### 2.2 动作协议 (Action Protocol)

所有可执行效果统一抽象为 `Action` 联合类型，由 [ActionEngine](../lib/action/engine.ts) 消费。

```typescript
type Action =
  | SpeechAction          // TTS 语音播报（带 agentId、音色、音量）
  | SpotlightAction       // 幻灯片聚光灯（定位元素 id、尺寸、时长）
  | LaserAction           // 激光指针轨迹（SVG Path）
  | PlayVideoAction       // 视频元素播放/暂停
  | WbDrawTextAction      // 白板：文本
  | WbDrawShapeAction     // 白板：形状
  | WbDrawChartAction     // 白板：图表
  | WbDrawLatexAction     // 白板：公式
  | WbDrawTableAction     // 白板：表格
  | WbDrawLineAction      // 白板：线条
  | WbDrawCodeAction      // 白板：代码块
  | WbEditCodeAction      // 白板：代码编辑动画
  | WbDeleteAction        // 白板：删除对象
  | DiscussionAction      // 触发多智能体讨论
  | WidgetHighlightAction // 交互元素高亮
  | WidgetSetStateAction  // 交互元素状态设置
  | WidgetAnnotationAction// 交互元素批注
  | WidgetRevealAction;   // 交互元素逐步揭示
```

**两种执行语义**：
- **即发即弃**：Spotlight、Laser — 调度后立即返回，由 `choreography` 常量控制自动清除。
- **同步等待**：Speech、Whiteboard Drawing、Discussion — 必须完成才推进光标，确保口头讲解与视觉同步。

### 2.3 多智能体花名册 (MultiAgentConfig)

```typescript
interface GeneratedAgentConfig {
  id: string;
  name: string;
  role: 'teacher' | 'student' | 'assistant' | 'moderator';
  avatar: string;           // 头像 SVG/PNG
  persona: string;          // Prompt 用性格描述
  voiceId?: string;         // 绑定的 TTS 音色
  model?: string;           // 可选：覆盖默认模型
  temperature?: number;     // 可选：个性参数
}
```

PI Director（见 §4）根据花名册决定每个 agent 的发言时机与内容风格。

---

## 3. 两大运行模式

### 3.1 模式对比

| 维度 | 经典一键生成 (Classic) | Pro 工作台 Durable Agent |
| --- | --- | --- |
| 入口 | `app/page.tsx` → Generate | `app/workbench/new/*` → Chat + Course Tabs |
| 生成管线 | `packages/@openmaic/generation` 两阶段同步管线 | `lib/server/agent-runtime` 基于租约的持久会话 Runner |
| 会话存储 | 浏览器 `IndexedDB` | PostgreSQL `AgentSession` 表（带心跳/取消/恢复） |
| 人机交互 | 生成前 1 次输入 → 生成中不可干预 | 流式对话，生成中可随时插入指令 / 取消 / 恢复 |
| 工具集 | 生成管线内建 outline / scene / media | 40+ 个显式工具（课程编辑/材料/技能/生成/导入/搜索/白板） |
| 故障恢复 | 刷新后丢失生成进度 | 崩溃后 Runner 按租约自动续跑；修订计数器增量拉取 |
| 开关 | 默认开启 | `NEXT_PUBLIC_PRO_WORKBENCH_ENABLED=true` + `DATABASE_URL` |

### 3.2 经典两阶段生成管线

```
 用户输入 (主题 + 材料)
        │
        ▼
 ┌──────────────────────┐
 │   Outline Generator  │  调用 LLM：输入 → 结构化大纲 JSON
 │  (packages/generation │  { lessons: [{ title, type, keyPoints... }] }
 │   outline-generator)  │
 └─────────┬────────────┘
           │ 大纲数组
           ▼
 ┌──────────────────────┐
 │   Scene Builder      │  遍历大纲 → 对每个大纲项并行/串行生成场景
 │  (packages/generation │  Slide  → content + actions + narration
 │   scene-builder)      │  Quiz   → questions[] + answers + 讲解
 │                       │  PBL    → projectConfig + 里程碑
 │                       │  Interactive → HTML + widgetConfig
 └─────────┬────────────┘
           │ 媒体插槽 (image/video slot)
           ▼
 ┌──────────────────────┐
 │   Media Orchestrator │  解析媒体请求 → 分发到对应 Image/Video Provider
 │  (lib/media)          │  → AssetPool 分配 id → 写回 Scene.mediaRefs
 └─────────┬────────────┘
           │ 旁白文本
           ▼
 ┌──────────────────────┐
 │   TTS Allocator      │  按 agent 花名册分配音色 → 生成音频 → 解析时长
 │  (lib/audio)          │  → 写回 Scene.narration.audioRef
 └─────────┬────────────┘
           ▼
        Stage 持久化 (browser kv or server document store)
```

### 3.3 Pro 工作台 Durable Runner

[`lib/server/agent-runtime/runner.ts`](../lib/server/agent-runtime/runner.ts) 实现了**租约协调的后台执行**模型：

```
┌────────────────────────────────────────────────────────┐
│                    PostgreSQL                          │
│  AgentSession (id, owner, status, lease_id, heartbeat) │
│  AgentEvent   (id, session_id, seq, type, payload)     │
│  UserMessage  (id, session_id, content, created_at)    │
│  DocumentStore (stage revisions, scene revisions)      │
└───────────────▲────────────────────────────────────────┘
                │ SELECT ... FOR UPDATE SKIP LOCKED
                │ (竞争领取待处理会话)
┌───────────────┴────────────────────────────────────────┐
│              AgentRunner (每进程 1 实例)                │
│  ┌──────────────────────────────────────────────┐      │
│  │ Lease Loop：每 N 秒扫描 → 领取租约 → 续心跳  │      │
│  └──────────────┬───────────────────────────────┘      │
│                 ▼                                       │
│  ┌──────────────────────────────────────────────┐      │
│  │ buildAgent() → @earendil-works Session       │      │
│  │  · Director Prompt + Skill Preload + Context │      │
│  │  · 40+ tools 打包为可调用函数                  │      │
│  └──────────────┬───────────────────────────────┘      │
│                 ▼                                       │
│  ┌──────────────────────────────────────────────┐      │
│  │ Tool Call Integrity + Mutation Fence         │      │
│  │  · 幂等工具调用追踪                           │      │
│  │  · 课程写入前 Stage 所有权校验                │      │
│  │  · per-scene revision counter ↑              │      │
│  └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
                │ SSE 流式推送 AgentEvent
                ▼
┌────────────────────────────────────────────────────────┐
│              Workbench Client (lib/workbench)          │
│  · 订阅事件流 → 增量追加聊天 / 课程 tab                 │
│  · 监听 per-scene revision ↑ → 只拉取变更的场景        │
│  · 用户消息 → 立即写入 UserMessage → 唤醒 Lease Loop  │
└────────────────────────────────────────────────────────┘
```

**核心保证**：
- **租约隔离**：同一时刻只有一个进程持有会话租约（基于 Postgres `FOR UPDATE SKIP LOCKED`）。
- **崩溃续跑**：进程挂掉后心跳过期（默认 30s），其他进程自动接管并从最后一个完整 `tool_call` 结果恢复。
- **修订单调性**：每次场景写入递增 `revision`，客户端使用条件请求避免过时覆盖。
- **突变围栏 (Mutation Fence)**：课程写入前校验会话 owner === stage owner，防止跨租户污染。

---

## 4. 多智能体编排架构

OpenMAIC 实现了两层多智能体系统：**课堂讨论层（PI Director）** 与 **PBL v2 教学层（4-Agent Pipeline）**。

### 4.1 PI Director 循环 (课堂讨论/白板)

[`lib/chat/pi/director-loop.ts`](../lib/chat/pi/director-loop.ts)

```
用户提问 / DiscussionAction 触发
        │
        ▼
 ┌────────────────────────────────────┐
 │  Director (顶层控制器)             │
 │  · 读取当前 Scene + 白板账本        │
 │  · 读取 Agent 花名册                │
 │  · 构建 Prompt：                   │
 │    - System: 角色分配 + 白板规则    │
 │    - History: 对话摘要 (Token 压缩) │
 │    - Evidence: 当前场景快照         │
 │  · 工具集：                         │
 │    1. call_agent(id, prompt)       │ ← 调用子 agent
 │    2. read_scene()                 │ ← 取场景证据
 │    3. cue_user(question)           │ ← 提问用户
 │    4. web_search(query)            │ ← 联网搜索
 │    5. close_session()              │ ← 结束讨论
 │    6. classroom_actions()          │ ← 播放/暂停/跳转
 │    7. native_whiteboard_*          │ ← 原生白板写入
 └──────────┬─────────────────────────┘
            │ LLM 选择工具 (多次迭代)
            ▼
 ┌────────────────────────────────────┐
 │  Child Runtime (子 Agent 池)       │
 │  · 并行 N 个 @earendil Session     │
 │  · 每个 Agent 拥有独立 System Prompt│
 │  · 共享白板账本，避免重复绘制       │
 └──────────┬─────────────────────────┘
            │ AgentTurnSummary[]
            ▼
 ┌────────────────────────────────────┐
 │  Whiteboard Ledger + Summarizer    │
 │  · 记录每个 agent 的白板操作        │
 │  · 构建「对话摘要」给下一轮 Director │
 │  · 避免上下文窗口爆炸               │
 └────────────────────────────────────┘
```

### 4.2 PBL v2 四智能体教学管线

[`lib/pbl/v2/`](../lib/pbl/v2/)

PBL v2 将一个学习项目分解为四个角色协同的**有状态工作流**：

| Agent | 职责 | 状态写入 |
| --- | --- | --- |
| **Planner** | 输入学习目标 → 输出项目大纲、里程碑、交付物、难度曲线 | `projectV2.milestones[]` |
| **Instructor** | 每一轮驱动：给出任务、讲解知识点、提示可用资源 | `projectV2.learnerState.lastInstructorMessage` |
| **Simulator** | 扮演环境 / 对话伙伴 / 虚拟队友 / 情境模拟器 | `projectV2.simulationHistory[]` |
| **Evaluator** | 里程碑交付物评估：熟练度、参与度、完成度，决定是否推进下一里程碑 | `projectV2.progress.*` + `projectV2.proficiency.*` |

**运行时关键机制**：
- **Fold 算子**：每个 Agent 输出 patch → 通过 `fold()` 原子合并进 `projectV2`，避免整字段回写冲突。
- **Drain 算子**：消费学习者的输入，按 **Kernel Operations**（engagement / proficiency / progress / task-completion / runtime-events）分别更新状态。
- **Hydration**：服务端存档 → 前端加载时的渐进反序列化 + 兼容迁移。

---

## 5. 播放管线架构

```
                     ┌──────────────────────┐
                     │  PlaybackEngine      │  [lib/playback/engine.ts]
                     │  状态机:             │
  start() ──────────▶│  idle → playing →    │
                     │  paused ↔ live       │
  pause()/resume() ─▶│                      │
                     │  维护 cursor(scene,  │
                     │  action)             │
                     └─────────┬────────────┘
                               │ next Action
                               ▼
                     ┌──────────────────────┐
                     │  Choreography Layer  │  [lib/choreography/]
                     │  · resolvePlayback-  │
                     │    Cursor()           │
                     │  · estimateSpeech-   │
                     │    DurationMs()       │
                     │  · 时序常量表         │
                     └─────────┬────────────┘
                               │ 规范化 Action
                               ▼
                     ┌──────────────────────┐
                     │  ActionEngine        │  [lib/action/engine.ts]
                     │  · Speech → TTS →    │
                     │    AudioPlayer        │
                     │  · Spotlight/Laser → │
                     │    Canvas Overlay     │
                     │  · WbDraw* →         │
                     │    Whiteboard Store   │
                     │  · Widget* →         │
                     │    PostMessage iframe │
                     │  · Discussion →      │
                     │    PI Director Loop   │
                     └─────────┬────────────┘
                               │ side effects
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  Canvas Overlay        Whiteboard Store      AudioPlayer
  (React DOM)           (Zustand)             (HTMLAudio/Web Audio)
```

**关键设计点**：
- **ActionEngine 单例复用**：在线（讨论中 Agent 即时动作）与离线（播放预录制 actions）两条路径共享同一执行层，避免双套实现漂移。
- **光标可逆性**：`lib/playback/action-navigation.ts` 提供「可重放前缀」判定 — 只有纯函数副作用的动作允许前进/后退跳转（白板绘制在跨越分支点时需要重置重绘）。
- **讨论打断机制**：`DiscussionAction` 触发时保存 `savedSceneIndex/ActionIndex`，讨论结束后 `confirmDiscussion()` 回到快照。

---

## 6. Provider 中立抽象层

### 6.1 抽象层总览

每类能力都通过 **Provider Interface → Registry → Dynamic Resolve** 的结构实现，对上层业务完全屏蔽具体供应商。

```
业务代码 (generation / playback / agent)
        │ 只依赖统一 API
        ▼
   Abstraction Interface
        │
        ├── lib/ai/providers.ts         → LLM Chat / Completions / Embeddings
        ├── lib/audio/tts-providers.ts  → Text → Speech
        ├── lib/audio/asr-providers.ts  → Speech → Text
        ├── lib/media/image-providers.ts→ Prompt → Image
        ├── lib/media/video-providers.ts→ Prompt → Video
        ├── lib/web-search/index.ts     → Query → SearchResults
        ├── lib/pdf/pdf-providers.ts    → FileBytes → StructuredDocument
        └── packages/@openmaic/storage  → Document / Runtime / Assets / Sessions
        │
        ▼
   Registry (运行时根据 .env + 用户设置动态启用)
        │
        ├── OpenAI · Azure · Anthropic · Bedrock · Google · DeepSeek · Qwen ·
        │   Kimi · MiniMax · Grok · OpenRouter · TokenDance · Doubao · GLM ·
        │   Hunyuan · Xiaomi · Ollama · Lemonade (本地) ...
        │
        ├── TTS: Azure / MiniMax / Qwen / ElevenLabs / VoxCPM / Lemonade /
        │   BrowserSpeech ...
        │
        ├── ASR: Azure / FunASR / Lemonade / BrowserSpeech ...
        │
        └── Search: Brave / Exa / Tavily / Baidu / Bocha / SearXNG /
            Doubao / Claude / MiniMax ...
```

### 6.2 解析链（以 LLM 为例）

[`lib/ai/providers.ts`](../lib/ai/providers.ts) + [`lib/server/resolve-model.ts`](../lib/server/resolve-model.ts)

```
请求 model="openai:gpt-5.5"
        │
        ▼
resolveModel()
  ├─ 1. 解析 provider 前缀 → 匹配 REGISTRY
  ├─ 2. 检查该 provider 的 *_ENABLED 环境变量是否为 true
  │       (支持启动时全局强杀: LLM_OPENAI_ENABLED=false)
  ├─ 3. 校验 API Key / Base URL 配置完整性
  ├─ 4. 若是 *-completions 方言走 LangChainAdapter
  │    若是 *-responses 方言走 @ai-sdk Provider
  └─ 5. 附加 Pinned Dispatcher (代理/超时/SSRF 防护)
        │
        ▼
   可执行 LanguageModel 实例
```

**启动期失败快**：启动时 `config-validation.ts` 扫描所有已配置 provider，报告坏配置而不是等请求时才 500。

### 6.3 SSRF 与安全性

- `lib/server/ssrf-guard.ts`：对外出 fetch URL 进行校验（拒绝私有网段 / 元数据端点 / 未解析主机）。
- `lib/server/proxy-fetch.ts`：通过 pinned undici dispatcher 统一出口，应用 HTTP 代理、超时、重试、限流。
- `lib/server/attempt-limiter.ts`：访问码登录等敏感入口每 IP 每窗口限流。
- `lib/server/ssrf-guard.ts` + `fetch-with-redirect-validation.ts`：防止开放重定向链绕过 SSRF。

---

## 7. 持久化架构

OpenMAIC 实现了**双层持久化**，构建时开关决定使用哪一层。

### 7.1 层对比

| 能力 | 浏览器层（默认） | 服务端层（Postgres / S3） |
| --- | --- | --- |
| 开关 | `NEXT_PUBLIC_PERSISTENCE` 未设置 | `NEXT_PUBLIC_PERSISTENCE=1` |
| 课程文档 | `Dexie (IndexedDB)` 键值 | `/api/persistence` → `DocumentStore` HTTP 契约 |
| 运行时状态 | `lib/runtime/store.ts` | `RuntimeStore` HTTP 契约 |
| KV（设置/播放进度） | `store/kv-persist.ts` | `KVStore` HTTP 契约 |
| 资产字节 | `image-storage.ts` (IndexedDB Blob) | `AssetStore`：Postgres BYTEA 或 S3 + 重定向 egress |
| Pro Agent 会话 | 不可用 | `AgentSessionStore`（租约 + 事件流） |
| 迁移策略 | 首次访问服务端时惰性迁移每个 course | — |

### 7.2 资产生命周期

```
       upload asset bytes
              │
              ▼
   AssetStore.allocate(owner, size, ttl=24h pending)
              │
              ▼
   pending 状态 → 分配 assetId
              │
 用户写入 document："我引用这个 assetId"
              │
              ▼
   DocumentStore.write() 时解析引用
              │
              ▼
   AssetStore.commit(assetId, owner) → live 状态
              │
              │ （后台收集器每 15 分钟）
              ▼
   扫描 live 资产 → 最后一份文档引用离开超过宽限期(1h)
              │
              ▼
   release entry → 字节再等 1 宽限 → 实际 DELETE
```

**配额**：`ASSET_QUOTA_BYTES`（默认 10 GiB）在写入事务内原子校验，并发上传不超配额。

### 7.3 文档版本化

```typescript
interface DocumentRecord {
  owner: string;
  stageId: string;
  revision: number;          // 单调递增
  sceneRevisions: Record<SceneId, number>; // 每场景独立版本
  payload: Stage;            // @openmaic/dsl JSON
  createdAt; updatedAt;
}
```

前端工作bench通过 `sceneRevisions` 只拉取有变化的场景，减少下载量。

---

## 8. 渲染与编辑内核

### 8.1 Renderer / Editor 分离

```
                    ┌─────────────────────────────┐
                    │     @openmaic/dsl           │
                    │   (Scene / SlideContent /   │
                    │    CanvasElement 类型)       │
                    └──────┬─────────────┬────────┘
                           │             │
            React Context  │             │ 编辑事件(拖拽/快捷键)
              只读订阅     │             │
                           ▼             ▼
              ┌─────────────────┐  ┌──────────────────┐
              │  @openmaic/     │  │  @openmaic/      │
              │  renderer       │  │  editor          │
              │  SlideCanvas    │  │  EditableSlide-  │
              │  (纯渲染+特效)  │  │  Canvas          │
              │  Base*Element   │  │  + UI Toolbars   │
              │  (不可交互)     │  │  + Intent Layer  │
              └─────────────────┘  └──────────────────┘
```

**Renderer 元素体系**（每个元素一个 `BaseXxxElement` + 可选高阶装饰器）：
- `BaseTextElement` → 富文本（ProseMirror schema → React DOM）
- `BaseImageElement` → 剪切路径 / 滤镜 / 图像轮廓
- `BaseShapeElement` → SVG 路径 + 渐变 / 图案填充
- `BaseChartElement` → ECharts 实例化
- `BaseLatexElement` → KaTeX / Temml 渲染
- `BaseLineElement` → 折线 / 曲线 / 连接线 / 箭头
- `BaseTableElement` → 静态 / 交互表格
- `BaseCodeElement` → Shiki + 代码行高亮
- `BaseAudioElement` / `BaseVideoElement` → 媒体播放

**特效叠加层**：`SpotlightOverlay` / `LaserOverlay` / `HighlightOverlay` / `ZoomWrapper` — 绝对定位在 SlideCanvas 之上，由 ActionEngine 写入 store 触发动画。

### 8.2 Editor 意图模型

[`packages/@openmaic/editor/src/react/core/intent.ts`]

所有手势（拖拽 / 缩放 / 旋转 / 框选 / 连线）不直接改 DOM，而是产出 `EditIntent` 对象：

```typescript
type EditIntent =
  | DragIntent       { deltaX, deltaY }
  | ResizeIntent     { handle, width, height }
  | RotateIntent     { angle }
  | CreateLineIntent { points: [] }
  | MarqueeIntent    { rect }
  | ...
```

→ 统一 `transaction()` 规约 → 不可变应用到 Canvas Model → 触发 React 重渲染。

好处：撤销/重做、命令录制、协同 OT 适配器全部只需拦截 Intent 层。

---

## 9. 关键数据流

### 9.1 课堂生成 → 播放的完整链路

```
1. User: POST /api/generate (topic + materials)
      │
2. Server:
   ├─ extract: document → 文本 / 图片 (lib/document)
   ├─ generateOutline
   ├─ buildScenes → 填充 actions[] + narration[]
   ├─ media orchestration → 生成图像/视频 → AssetPool
   ├─ TTS allocator → narration audio → 解析 duration
   └─ documentStore.write(stage)
      │
3. Client:
   ├─ redirect → /classroom/[id]
   ├─ useStageStore.load(stageId)
   ├─ render <Stage/> → <SlideCanvas/>
   └─ user 点击 "开始授课"
      │
4. PlaybackEngine.start()
   ├─ cursor = (scene=0, action=0)
   ├─ for each action: ActionEngine.execute(action)
   │    ├─ speech → audioPlayer.play()
   │    ├─ wbDraw → WhiteboardStore.dispatch()
   │    └─ ...
   ├─ discussion 触发 → PI Director 接管(见 §4.1)
   └─ cursor 前进直到末尾
```

### 9.2 Pro 工作台：用户消息 → 课程变更的链路

```
1. User 在 Chat Pane 输入："把第 3 张改成讲数据结构，配个图"
      │ POST /api/agent/sessions/[id]/events
2. Server:
   ├─ UserMessage 入库 (owner, session_id, text)
   ├─ EventNotifyBus.publish() → 唤醒持有租约的 Runner
   ├─ Runner.claim → 追加消息到 AgentSession
   ├─ buildAgent():
   │    └─ tools.curriculum_tools.patch_scene(scene_id, patch)
   │       └─ withOwnerStageAuthorization(owner, stage_id)
   │          └─ apply() → scene revision++
   └─ 每条 tool_result 写 AgentEvent (seq ↑)
3. Client:
   ├─ SSE: on AgentEvent → 渲染聊天气泡 / 代码块
   ├─ 订阅 stage revisions → 发现 scene3 rev 3→4
   └─ 只 FETCH /api/stages/[id]/scenes/3?revision=4 → 更新右侧预览
```

---

## 10. 可扩展点与插件机制

| 扩展点 | 接入方式 | 参考示例 |
| --- | --- | --- |
| **新增 LLM Provider** | `lib/ai/providers.ts` 实现 `ProviderFactory` → `MODEL_REGISTRY` 注册 → `@openmaic/dsl` 模型别名 | 搜索 `createOpenAI` 同类写法 |
| **新增 TTS / ASR Provider** | `lib/audio/{tts,asr}-providers.ts` 实现 `TTSProvider` / `ASRProvider` 接口 | `voxcpm.ts`, `funasr.ts` |
| **新增图像/视频 Provider** | `lib/media/adapters/*.ts` 实现 `ImageAdapter` / `VideoAdapter` → `image-providers.ts` 注册 | `qwen-image-adapter.ts` |
| **新增 Web Search Provider** | `lib/web-search/{name}.ts` 实现 `SearchProvider` → `index.ts` 注册 + Settings UI | `brave.ts`, `bocha.ts` |
| **新增课堂场景类型** | `@openmaic/dsl` 扩 `SceneContent` → `lib/types/stage.ts` 扩宽 → 渲染器注册 `SceneRenderer` | `InteractiveContent` 实现路径 |
| **新增 Agent 动作 (Action)** | `@openmaic/dsl` 扩 `Action` 联合 → `ActionEngine` 新增 case → `choreography` 常量 | `WbDrawCodeAction` 是最新范式 |
| **Pro 工作台新增 Tool** | `lib/server/agent-runtime/*-tools.ts` 编写工具 + Prompt 块 → `assembleRunnerTools` 注册 → `tool-call-integrity` 幂等键 | `course-tools.ts`（DSL 编辑工具集） |
| **新增 Pro Skill** | 浏览器端：POST `/api/skills` 上传 Markdown → Runner 启动时 `skill-preload` 注入 Prompt | `skills/openmaic/SKILL.md` 标准格式 |
| **持久化层切换** | 实现 `@openmaic/storage` 的 `DocumentStore / AssetStore / ...` 接口 → `lib/persistence/bootstrap.ts` 选择实现 | `pg.ts` vs `plain-json.ts` |
| **白板扩展** | `lib/whiteboard/runtime/store.ts` 扩 action 类型 → `validate.ts` 加 schema → 前端 `components/whiteboard/` 注册绘制 | `wb_draw_code` 路径 |

---

## 11. 质量与评估体系

| 体系 | 工具 | 目的 |
| --- | --- | --- |
| **单元测试** | Vitest + fake-indexeddb | 覆盖 `lib/*` 与 `packages/@openmaic/*` 的纯逻辑层 |
| **E2E** | Playwright (`tests/e2e/`) | 首页生成 → 课堂播放 → 编辑模式关键路径 |
| **Agent 质量评估** | `eval/` tsx 脚本 | PBL Planner 质量、大纲语言质量、白板布局、编排回答准确率 |
| **Prompt 回归** | Vitest Snapshot (`tests/pbl/` + `tests/media/`) | Prompt 模板意外修改 → 快照 Diff 告警 |
| **i18n 一致性** | `scripts/check-i18n-keys.mjs` | 所有 locale 缺失 key 阻断构建 |
| **构建期校验** | `check:node-engine` / `check:package-versions` | Node 版本、包版本一致性 CI Gate |

---

## 12. 架构约束与设计原则回顾

1. **DSL 作为唯一真相源**：任何场景数据变更必须先落在 `@openmaic/dsl` 类型，再扩 `lib/types/stage.ts`，禁止组件中出现 ad-hoc 字段。
2. **Provider 绝不泄漏到业务层**：`lib/playback`、`lib/generation`、`lib/orchestration` 禁止直接 import `@ai-sdk/openai` 等供应商包。
3. **服务器与浏览器严格分离**：`lib/server/*` 只能在 API Route 与 Runner 中 import；浏览器端若误引入会在 `next build` 时因 Node API 引用失败。
4. **写入路径走事务**：课程写入、资产分配、配额校验均通过 SQL 事务或 IndexedDB 事务，避免部分成功。
5. **失败快于静默成功**：Provider 配置校验、模型路由、修订冲突、动作参数非法全部**显式抛错**而非降级（用户可看见 Toast 报错，避免数据悄悄损坏）。
6. **时间线可重放**：任何可播放动作必须是纯函数副作用或可撤销，保障 `action-navigation` 跳转语义成立。

---

> 下一步阅读建议：若需要深入某一块，按「PROJECT-STRUCTURE.md 目录导航决策树」定位入口文件，再结合本架构文档对应章节理解上下游。
