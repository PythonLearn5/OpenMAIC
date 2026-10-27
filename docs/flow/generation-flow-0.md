# 课程生成管线 · 总览（Server-side Generation Run）

<!--
  职责：说明 OpenMAIC 「一键生成课程」的**服务端生成管线**总体架构，
  包括：三层职责划分（接入层 API / 编排层 run engine / 能力层 steps）、
  7 态状态机、步骤计划与检查点、Postgres 权威存储、租约与接管、
  事件日志与 SSE、命令幂等、超时/重试/并发常量、失败码体系，
  以及 7 步分册文档的阅读顺序。
  每一步的输入输出契约、提示词、解析与校验细节见同目录
  generation-flow-1.md ~ generation-flow-7.md。
-->

- 相关项目结构文档：[PROJECT-STRUCTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/PROJECT-STRUCTURE.md)
- 相关总体架构文档：[ARCHITECTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/ARCHITECTURE.md)
- 相关学习大纲：[LEARNING-OUTLINE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/LEARNING-OUTLINE.md)
- 分册文档：[第1步 材料分析](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-1.md) ·
  [第2步 联网检索](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-2.md) ·
  [第3步 大纲生成与审查](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-3.md) ·
  [第4步 Agent 画像](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-4.md) ·
  [第5步 场景内容](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md) ·
  [第6步 场景动作](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md) ·
  [第7步 旁白 TTS 与场景入库](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-7.md)

---

## 0. 阅读提示

### 0.1 本文档描述的是「服务端 run」，不是 1.1.x 的浏览器驱动流程

本目录下的旧版文档描述的是**浏览器驱动**的流程：`/generation-preview` 页面依次
`fetch` 一组 `app/api/generate/scene-*` 的 SSE 路由，页面关闭即生成中断，剩余场景
在 `/classroom/:id` 由 [use-scene-generator.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/hooks/use-scene-generator.ts)
继续生成。**这套流程已经被服务端 generation run 取代**，且
`scene-outlines-stream`、`scene-content`、`scene-actions`、`agent-profiles` 等路由
**已从仓库删除**（`app/api/generate/` 现在只剩 `tts / image / video / voice` 四个媒体路由，
供 run 引擎与旧客户端复用）。

> ⚠️ 如果你在代码里找不到文档中提到的 `app/api/generate/scene-content`，
> 不是链接错了，是它确实不存在了。它的逻辑搬进了
> [lib/server/generation/steps/scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts)。

改版后的核心变化：

| 维度 | 1.1.x 浏览器驱动（旧） | 服务端 run（现） |
|---|---|---|
| 编排者 | 浏览器页面 `startGeneration()` | Node 进程内的 runner worker |
| 状态存放 | 浏览器内存 + IndexedDB | Postgres 四张表（唯一权威） |
| 关页面 | 生成中断 | 不受影响，继续跑 |
| 断点续跑 | 无（重头再来） | 步骤级检查点，从最后一个已完成步骤续跑 |
| 失败处理 | 页面报错 | `paused` + 结构化 `errorCode` + `retry` 命令 |
| 大纲审查 | 页面上的确认按钮 | `awaiting_outline_confirmation` 状态 + `confirm-outline` / `hold-outline` 命令 |
| 剩余场景 | 课堂页再生成一次 | 与首场景同一条管线，只是场景序号不同 |

### 0.2 三向映射：旧 UI 5 步 ↔ run 步骤 ↔ 本目录 7 册

「7 步」指的是**生成管线的 7 类工作**，它同时对应三个不同的命名体系，务必分清：

| 本目录分册 | run 的 step kind / step id | 生成预览页的 UI 步骤 | 能力层实现文件 |
|---|---|---|---|
| 第 1 册 | `material-analysis`（id 即同名） | **不占 UI 步骤**（折叠进 web-search） | [steps/material-analysis.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts) |
| 第 2 册 | `research` | `web-search` | [steps/research.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts) |
| 第 3 册 | `outline` | `outline` | [steps/outline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts) |
| 第 4 册 | `agents` | `agent-generation` | [steps/agent-profiles.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts) |
| 第 5 册 | `scene:<n>:content` | `slide-content` | [steps/scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts) |
| 第 6 册 | `scene:<n>:actions` | `actions` | [steps/scene-actions.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts) |
| 第 7 册 | `scene:<n>:narration` + `media:<elementId>` | （narration/media 不占 UI 步骤） | [steps/narration.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts)、[run/media.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts) |

UI 步骤只有 5 个，且是**可选显示**的（不开联网搜索就没有 `web-search`，用预置角色就没有
`agent-generation`），映射函数见
[preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L22-L48)。
其中 `material-analysis` 故意不占 UI 位置——材料的抽取从**上传时**就开始了，run 的这一步
只是「读取（或等待）已经抽好的结果」，所以它运行时 UI 显示的是下一步；它失败时同样会让
run `paused`（见该文件头注释）。

### 0.3 为什么所有代码链接都用绝对路径

> ⚠️ IDE 内置 Markdown 预览中，`#L行号` 锚点如果挂在**相对路径**上，会被当作
> “Markdown 标题跳转”解析，从而报错 `Cannot find header to navigate to`。
> 只有挂在**绝对 `file:///` URL** 上，IDE 才会把它当作“打开文件并跳到指定行”的命令。
>
> 👉 需要把文档推到 GitHub 网页查看（要求相对路径）时，在仓库根目录执行：
>
> ```
> $env:REPO="D:\GITHUB_chat\OpenMAIC"
> $md="docs\flow\generation-flow.md"
> (Get-Content $md -Raw) `
>   .Replace("file:///$($env:REPO.Replace('\','/'))/docs/","../") `
>   .Replace("file:///$($env:REPO.Replace('\','/'))/","../../") `
>   | Set-Content $md -NoNewline
> ```
>
> 分册文件把 `$md` 换成 `docs\flow\generation-flow-1.md` … 逐个执行即可
> （macOS/Linux 用等价的两行 `sed -i`）。

### 0.4 术语约定

| 术语 | 含义 |
|---|---|
| **run** | 一次课程生成任务，数据库里一行 `generation_runs`，id 形如 `run-xxxxxxxxxxxx` |
| **step** | run 计划里一个最小可续跑单元，完成后在 `generation_run_steps` 留下检查点 |
| **checkpoint（检查点）** | 步骤的输出快照，`{ run_id, step_id, output }`；接管后从最后一个检查点之后继续 |
| **event（事件）** | 追加式有序日志（`seq` 每 run 递增），既驱动 SSE 也用于审计 |
| **command（命令）** | 用户对 run 的干预（确认大纲 / 暂停倒计时 / 重试），按 `commandId` 幂等 |
| **lease（租约）** | 某个 worker 对 run 的独占执行权，由 `lease_worker_id + lease_generation + lease_heartbeat_at` 三列表达 |
| **slot（能力槽）** | owner 配置的模型位（如 `course.outline`、`tts`、`image`、`webSearch`），决定某步用什么模型；**请求里不带 key 和 model** |
| **lane（泳道）** | 与主线并行的媒体生成通道，每步独占一个 `media:<elementId>` 检查点 |

---

## 1. 三层架构

```
┌─ 接入层（Next.js Route Handlers, app/api/generation-runs/**）
│    解析请求 → 校验 → 鉴权 owner → 写命令 / 读快照 / 开 SSE
├─ 编排层（lib/server/generation/run/**）
│    状态机 · 计划 · 租约 · 检查点提交 · 事件 · 重试 · 超时 · 失败码 · 媒体泳道 · runner worker
├─ 能力层（lib/server/generation/steps/** + lib/server/generation/outline-schema.ts）
│    纯函数式步骤：(input, ctx) => output；不碰 HTTP、不碰 owner、不解析模型
└─ 权威存储（lib/persistence/generation-runs.ts + @openmaic/storage）
     Postgres：generation_runs / _steps / _events / _commands
```

### 1.1 目录清单

编排层（`lib/server/generation/run/`，共 17 个文件）：

| 文件 | 职责 |
|---|---|
| [types.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts) | 状态、输入、事件类型、快照、失败结构 |
| [plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) | 步骤序列的**唯一定义**：id 格式、阶段、续跑位置 |
| [store.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts) | 四张表的读写、认领、租约、命令、配额、压缩 |
| [engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) | 唯一的执行者：分派步骤、装配输入、提交检查点 |
| [services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) | 把 owner + slot 解析成能力层要的具体模型/连接 |
| [start.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts) | 启动 run：授权、材料校验、槽位校验、唤醒 runner |
| [runner.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/runner.ts) | 进程级常驻 worker：扫描、认领、心跳、压缩、保活 |
| [retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts) | 双层重试中的“路由级”重试封装与状态码归一 |
| [deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | 每类步骤的超时表与 `AbortSignal.any` 封装 |
| [failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts) | 任意异常 → 稳定的 `errorCode` |
| [config.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/config.ts) | 并发/配额/保留期常量与 env 覆盖 |
| [input.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts) | 请求体大小上限与全量参数校验 |
| [sse.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/sse.ts) | SSE 帧、每 owner 流配额、轮询式事件流 |
| [media.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts) | 图片/视频泳道：检查点、放置、失败码、重试性 |
| [document.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts) | 课程文档的围栏写入（建课 / 追加场景 / 收尾） |
| [narration-voice.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts) | tts 槽 → 广告语音表 → 单条 clip 的语音归属 |
| [hook-events.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/hook-events.ts) | 已提交事件 → 宿主（host）通知 |

能力层（`lib/server/generation/steps/`，共 10 个文件）：

| 文件 | 暴露的步骤函数 |
|---|---|
| [context.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/context.ts) | `StepContext`、`StepRefusal`、`StepAbortedError`、视觉图解析类型 |
| [material-analysis.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts) | `analyzeMaterial()` |
| [research.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts) | `research()` |
| [outline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts) | `generateOutlines()`、`prepareOutline()`、`streamOutlines()` |
| [agent-profiles.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts) | `generateAgentProfiles()` |
| [scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts) | `generateSceneContent()` |
| [scene-actions.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts) | `generateSceneActions()` |
| [narration.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts) | `synthesizeNarration()` |
| [image.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/image.ts) | `generateImageStep()` |
| [video.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/video.ts) | `generateVideoStep()`（支持断点续等 provider task） |

### 1.2 能力层的统一契约：`(input, ctx) => output`

所有步骤函数都不 import Next 的 `NextRequest`、不查 owner、不解析模型槽，只接受一个
`StepContext`：

```ts
export interface StepContext<Event = never> {
  signal?: AbortSignal;
  log: StepLogger;
  emit?: (event: Event) => void;   // 只有会“边跑边播”的步骤（outline）用到
}
```

见 [context.ts#L19-L29](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/context.ts#L19-L29)。
模型由调用方作为 `input.model` 注入（类型是 `StepLanguageModel`，
见 [context.ts#L59-L68](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/context.ts#L59-L68)）。

**失败的唯一表达方式是抛 `StepRefusal<Reason>`**（
[context.ts#L70-L79](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/context.ts#L70-L79)）：
`reason` 是**每个步骤自己的小枚举**（例如材料分析有 6 种、Agent 画像有 3 种），
`statusCode` 是给编排层的“建议 HTTP 码”。编排层据此判定可重试性并映射到全局 `errorCode`。
这样能力层可以被路由、run 引擎、测试三处复用而不需要改动。

---

## 2. 一次生成的完整时序

```
浏览器 POST /api/generation-runs { requirement, materialIds, outlineReview, ... }
   └─ route.ts → parseRunInput() → startGenerationRun()
        ├─ 模型槽预检（GENERATION_MODEL_SLOTS 全部要能解析且已授权）
        ├─ 材料/预置 agent 校验 → host authorizeStart 钩子
        └─ createGenerationRun()  ← pg_advisory_xact_lock 串行化 owner 配额
             写 generation_runs(state='preparing') + 首个事件
        返回 202 + runSnapshot(run)

runner worker（常驻，每个进程一个）
   scan() 每 scanIntervalMs：
     ├─ confirmDueGenerationRunOutlines()  ← countdown 模式到点自确认
     └─ claimNextGenerationRun(workerId, maxConcurrent)
          SELECT ... FOR UPDATE SKIP LOCKED → lease_generation+1 → 返回带租约的 run

executeGenerationRun(lease)  —— engine.ts 的主循环
   while (true) {
     advanceRun(plan, checkpoints) → 下一个未完成 step
     runStep(step) {
        能力层步骤函数（withDeadline + withRouteRetry）
        commitGenerationRun(lease, { patch, step:{stepId,output}, events })
          └─ 同一事务：校验租约 → upsert 检查点 → 更新 run 行 → 追加事件 → pg_notify
     }
     按检查点里的 outline 建/追加课程文档（document.ts，走 exclusive() 串行）
     并行媒体泳道（media.ts）
   }
   出口：completed / paused(失败) / ended(课程被删) / interrupted(租约丢或进程退出)

浏览器 GET /api/generation-runs/:id/events?after=<seq>
   └─ SSE：backlog 事件 → caught_up → 之后每个提交的事件实时下发（25s 心跳）
      不能长连接的客户端退化为 GET /api/generation-runs/:id 轮询
```

关键代码位置：
- 启动：[start.ts#L60-L127](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L60-L127)
- 认领：[store.ts#L814-L949](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L814-L949)
- 主循环：[engine.ts#L1750-L1877](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1750-L1877)
- 步骤分派：[engine.ts#L590-L770](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L590-L770)
- 提交：[store.ts#L1074-L1111](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1074-L1111)

---

## 3. 状态机（7 态）

定义在 [types.ts#L23-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L23-L53)，
并被数据库约束死（[generation-runs.ts#L64-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/persistence/generation-runs.ts#L64-L65)）：

| 状态 | 含义 | 算“可执行” | 算“活跃” | 占 owner 配额 |
|---|---|---|---|---|
| `preparing` | 正在跑准备步：材料分析 / 联网检索 | ✅ | ✅ | ✅（active） |
| `outlining` | 正在生成大纲 | ✅ | ✅ | ✅（active） |
| `awaiting_outline_confirmation` | 大纲已就绪，等用户确认（或等倒计时） | ❌ | ✅ | ✅（waiting） |
| `generating` | 大纲已确认，正在生成 Agent/场景/媒体 | ✅ | ✅ | ✅（active） |
| `paused` | 某步最终失败，冻结等 `retry` | ❌ | ✅ | ❌（但 retry 时会再占） |
| `completed` | 全部步骤完成，课程文档已收尾 | ❌ | ❌ | ❌ |
| `ended` | 课程被删除（或 run 被丢弃），终态 | ❌ | ❌ | ❌ |

```
           ┌────────────── confirm-outline / 倒计时到点 ──────────────┐
           │                                                          │
preparing ─┴→ outlining ──→ awaiting_outline_confirmation ──→ generating ──→ completed
   （材料/检索）              （大纲等确认）                        （Agent+场景+媒体）
        │                          │                                   │
        └──────────── 步骤最终失败 ─┴──────────→ paused ── retry ──→ preparing/outlining/generating
        │                                                            （按失败步所在阶段回退）
        └──────── 课程被删除 / discard ─────────────→ ended
```

- “回退到哪个状态”由失败步决定：[plan.ts#L107-L116](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L107-L116) 的 `stateForRetry()`。
- `ACTIVE_RUN_STATES` 与 `LIMITED_RUN_STATES` 是**两套不同口径**：前者用于课程卡片展示
  （包含 `paused`、`awaiting`），后者用于配额计算（只算真正在跑的 3 个）。
  见 [types.ts#L39-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L39-L53)。

---

## 4. 步骤计划：`plan.ts` 是唯一权威

一个 run 的计划由输入决定，**分两阶段生成**：

```ts
// 阶段 A：准备（大纲还没出来时只有这些）
preparationSteps({ materialIds }) =
  [ 'material-analysis'? , 'research' , 'outline' ]

// 阶段 B：大纲确认后按场景数展开
generationSteps(sceneCount) =
  [ 'agents',
    'scene:0:content', 'scene:0:actions', 'scene:0:narration',
    'scene:1:content', 'scene:1:actions', 'scene:1:narration', ... ]
```

- 有材料才插入 `material-analysis`，没联网搜索就不插 `research`
  （[plan.ts#L46-L56](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L56)）。
- 场景步 id 格式由 [plan.ts#L29-L44](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L29-L44) 定义，
  解析用 `parseStepId()`；媒体步是 `media:<elementId>`
  （[plan.ts#L118-L122](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L118-L122)）。
- **“下一个该跑哪步”没有前驱表**，而是 [advanceRun()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L86-L104)
  顺序扫描计划，取**第一个没有检查点**的步骤。因此续跑、retry、接管都复用同一套逻辑，
  也天然容忍“计划变长”（大纲改了 → 场景数变了 → 未完成的步自动重排）。
- `phaseOfStep()`（[plan.ts#L68-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L68-L72)）
  把步骤映射到 `preparing|outlining|generating`，run 的当前状态就是它。
- [mediaItemsOf()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L135-L151)
  从确认后的 outline 抽出所有要生成的图片/视频元素，形成媒体泳道的任务清单。

> 📌 plan.ts 的头注释里有一句设计原则值得记住：
> “**A step whose capability does not resolve still runs and checkpoints that it did nothing.**”
> 即：某步依赖的能力没配置（例如没开联网搜索、没配 tts 槽），这一步**仍然会跑**，
> 并留下“我什么都没做”的检查点。这样计划长度稳定、UI 进度可解释，也避免了
> “跳过步骤”带来的续跑歧义。

---

## 5. 权威存储：四张表

建表 DDL 与迁移全部在 [lib/persistence/generation-runs.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/persistence/generation-runs.ts)
（5 个迁移版本），只有 schema 归它管，语义全在 store.ts。

| 表 | 主键 | 作用 |
|---|---|---|
| `generation_runs` | `id` | run 本体：`input`(JSONB) / `state` / `step` / `outline` / `outline_revision` / `agents` / `stage_id` / `scenes_total` / `scenes_completed` / `error` / `seq` / 租约三列 + `takeovers` / `media_pending` / `narration_unvoiced` / `media_summary` / `outline_auto_confirm_at` / `host_attributes` |
| `generation_run_steps` | `(run_id, step_id)` | **检查点**：每完成一步写一行 `output` JSONB |
| `generation_run_events` | `(run_id, seq)` | **有序事件日志**：`seq` 由 run 行的 `seq` 分配（[store.ts#L396-L412](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L396-L412)） |
| `generation_run_commands` | `(run_id, command_id)` | **幂等命令**：重复命令直接返回第一次的 `result` |

索引都是有语义用途的部分索引：owner 活跃列表、可执行扫描、媒体待办、按 `stage_id`
反查“这门课还有 run 在写”、大纲自动确认到点扫描。见
[generation-runs.ts#L68-L124](file:///D:/GITHUB_chat/OpenMAIC/lib/persistence/generation-runs.ts#L68-L124)。

**归属与 owner 合并**：所有读取都套 `OWNED_BY($n)`（递归 CTE，最多
`MAX_MERGE_HOPS=16` 跳），把一个 owner 合并链上的 run 都视为可见，见
[store.ts#L317-L344](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L317-L344)。

**压缩**：`completed`/`ended` 超过 `finishedRetentionMs` 后，
[compactFinishedGenerationRuns()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L344-L390)
会删掉检查点和大部分事件，但先把媒体统计折算进 `media_summary`、把无音频 clip 数写进
`narration_unvoiced`；此时客户端事件游标落后会收到 `resync` 帧（§7）。

---

## 6. 租约、接管与围栏

这是整套设计里最“分布式”的部分，也是学习重点。

| 机制 | 实现 | 说明 |
|---|---|---|
| worker 身份 | [runner.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/runner.ts) | `workerId = <pid>:<uuid>`，进程实例挂在 `globalThis[Symbol.for('openmaic.generation-runs.runner')]` 上，保证同进程只有一个 |
| 认领 | [claimNextGenerationRun()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L814-L949) | `SELECT ... FOR UPDATE SKIP LOCKED` 取 5 行候选 → 逐个尝试，一次最多领到 `maxConcurrent` 之内 |
| 租约列 | `lease_worker_id / lease_heartbeat_at / lease_generation` | 心跳续 `lease_heartbeat_at`；每次认领 `lease_generation+1` |
| 过期判定 | 同上 + `leaseTtlMs` | 心跳超过 TTL 没更新即视为“孤儿 run”，可被别的 worker **接管** |
| 接管次数 | `takeovers` | 每成功提交一步就归零；连续孤儿化超过 `maxTakeovers` → 直接 `paused` + `INTERNAL_ERROR` + “The step was interrupted too many times” |
| 写围栏 | 每个写路径先 `lockLeased()` | 校验 `id + lease_worker_id + lease_generation` 三元组，不匹配抛 [GenerationRunLeaseLostError](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L68-L79) → 当前 worker **立刻停止写**，被接管的旧 worker 再也写不进来 |
| 唤醒 | `pg_notify` + 兜底轮询 | 提交事务末尾 `notifyOwner`；worker 的 `scan()` 仍按 `scanIntervalMs` 扫，防止 notify 丢失 |
| 唤醒（外部） | [wakeGenerationRunner()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/runner.ts#L42-L44) | start / retry 等路由写完就调它，不必等下一个扫描周期 |
| 配额串行化 | `pg_advisory_xact_lock(hashtextextended(owner))` | owner 级“活跃 run 数 / 等待确认 run 数”检查与插入在同一事务锁内完成，避免并发启动超额（[store.ts#L548-L552](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L548-L552)） |
| 文档写串行 | `store.document.exclusive()` | 场景追加、媒体放置都排队走它，保证同一课程不被两个 worker 撕裂 |
| 课程被生成中保护 | [assertCourseWritableIn()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1170-L1190) | 用户手动编辑一门正在生成的课 → `COURSE_GENERATING` / **423** |

worker 内部的三个定时器与细节见
[runner.ts#L111-L236](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/runner.ts#L111-L236)：
`scan` / `compaction` / `keepAlive`（= `max(1000, min(compactionIntervalMs, 资产 TTL/4))`，
用于给还在引用的生成资产续命）。优雅停机 `stop({ timeoutMs: 15_000 })`。

一个有意思的边界：worker 若发现 `claimNextGenerationRun` 把**自己已在跑的 run** 又交给自己，
会 abort 当前执行并按“未发生接管”释放租约（`releaseGenerationRunLease({ undoTakeover: true })`），
避免同一个 run 被同进程跑两遍。

---

## 7. 事件目录（21 种）与 SSE

事件类型定义在 [types.ts#L264-L319](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L264-L319)：

| 事件 | 何时 | 载荷要点 | 消费者 |
|---|---|---|---|
| `state` | 每次状态迁移 | `{ state, from }` | 进度 UI、状态标签；`paused` 会上报宿主 |
| `step_started` | 一步开始 | 步骤 id、阶段 | “正在生成第 N 页” |
| `step_retry` | 路由级重试触发 | `{ step, attempt, maxAttempts, reason, cause }` | 重试提示 |
| `step_completed` | 一步完成 | 步骤 id | 进度条 |
| `step_failed` | 一步最终失败 | `GenerationRunFailure`（§9） | 错误卡片 + Retry 按钮 |
| `material_kinds` | 材料分析得知材料类型 | 类型清单 | 准备期文案 |
| `material_truncated` | 材料被截断 | `{ textChars?, images?: {total,max} }` | “材料过多已截断”提示 |
| `research_sources` | 联网检索完成 | 来源列表 | 引用来源展示 |
| `outline_reset` | 大纲被重新生成 | revision | 清空已画的大纲 |
| `outline_language_directive` | 流式解析出语言指令 | 文本 | 后续步骤的语言基调 |
| `outline_course_title` | 流式解析出课程标题 | 标题 | 页面标题 |
| `outline_item` | **每解析出一个场景大纲就发一条** | 单条 outline | 大纲逐条“长出来”的效果 |
| `outline_ready` | 大纲全部就绪 | 全量 outline | 进入确认 UI；会上报宿主 |
| `outline_confirmed` | 确认（人工或倒计时） | 确认后的 outline | 切到 generating |
| `outline_review` | 审查模式变化 | 模式 | `hold-outline` 的效果 |
| `agents` | Agent 画像完成 | `{ agents, ... }` | 角色卡片 |
| `course_created` | 首个场景同事务建课 | `{ stageId }` | 跳转到课堂页的时机 |
| `scene_ready` | 一个场景入库完成 | 场景序号、标题 | 逐页出现在课堂上；会上报宿主 |
| `completed` | 整个 run 完成 | 汇总 | 收尾；会上报宿主 |
| `ended` | 课程删除 / run 丢弃 | 原因 | 上报宿主 |
| `media` | 媒体检查点变化 | `GenerationRunMediaEventData`（第 7 册 §5） | 图/视频占位与重试按钮 |

SSE 契约（[run/sse.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/sse.ts) +
[events/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/events/route.ts)）：

- 每帧带 `id: <seq>`；重连时浏览器 `EventSource` 自动回送 `Last-Event-ID`，服务端按游标续传。
- backlog 排空后先发一帧 `caught_up`，之后每个**已提交**事件实时下发；大纲事件是“流式”的，
  所以用户能看到场景一条条出现。
- 空转时每 `RUN_SSE_HEARTBEAT_INTERVAL_MS = 25_000` 发 `: ping` 注释帧。
- 写队列上限 `RUN_SSE_QUEUE_BYTES = 256 * 1024`，超了就断开让客户端重连（背压）。
- 每 owner 最多 `maxRunStreamsPerOwner()`（默认 16）条流，超了返回 **429 + Retry-After: 5**。
- 事件压缩后游标过旧 → 先发 `resync` 帧（客户端应重取快照），再发保留的事件。
- **流只读**：关掉 SSE 不会影响 run 一分钱。
- 兜底轮询间隔 `RUN_EVENTS_POLL_INTERVAL_MS = 5_000`（NOTIFY 通常 ~100ms 内唤醒，这个只兜丢失的 notify）。

---

## 8. 接入层 API 契约

接入层全部在 `app/api/generation-runs/` 下。

### 8.1 路由清单（7 个文件、9 个 handler）

| 方法与路径 | 文件 | 作用 | 关键返回码 |
|---|---|---|---|
| `POST /api/generation-runs` | [route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/route.ts) | 启动 run | 202 快照；400 `INVALID_REQUEST`；403 `INVALID_URL`；429 `ACTIVE_RUN_LIMIT`；宿主 `authorizeStart` 自定义 |
| `GET /api/generation-runs?active=1` | 同上 | 列 owner 活跃 run + `limits{maxActive,maxWaiting}` | 400（不传 `active=1`） |
| `GET /api/generation-runs/:id` | [[id]/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/route.ts) | 快照 + 最后事件 `seq` + 全部 `media` 状态（轮询退化路径） | 404（他人的 run 与不存在的 run 同答） |
| `DELETE /api/generation-runs/:id` | 同上 | 丢弃**还没有课程**的 run（重复丢弃返回同样结果） | 409 `RUN_STATE_CONFLICT`（课程已存在，应改删课） |
| `GET /api/generation-runs/:id/events?after=<seq>` | [[id]/events/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/events/route.ts) | 该 run 的有序事件流（SSE） | 404；429 `Retry-After: 5`（流配额） |
| `POST /api/generation-runs/:id/confirm-outline` | [confirm-outline/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/confirm-outline/route.ts) | 确认大纲（可携带改后的列表） | 409 `RUN_STATE_CONFLICT` |
| `POST /api/generation-runs/:id/hold-outline` | [hold-outline/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/hold-outline/route.ts) | 取消自动确认，转为 `wait` | 409 |
| `POST /api/generation-runs/:id/retry` | [retry/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/retry/route.ts) | `{commandId}` 重跑失败步；或 `{commandId, media:{elementId}}` 只重生成一个媒体 | 409；429 `ACTIVE_RUN_LIMIT` |
| `GET /api/generation-runs/events` | [events/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/events/route.ts) | **owner 级**多 run 变更合流（课程列表用）：先一帧 `runs`，之后每次变更一帧 `run` | 429（流配额） |

> `retry` 的两种形态区别很大：整run重试要求状态是 `paused`；单媒体重试允许 run 处于
> `generating` / `paused` / `completed`，且**只重跑那一个元素**，其余什么都不动。
> 若元素失败原因 Retry 改变不了（如被内容安全拦截），一律 409。
> 快照里每个失败都带 `failureSeq`（报告它的事件 `seq`），客户端的 `commandId` 由它派生，
> 从而“同一个失败点重试两次”天然是同一条幂等命令。

参数校验集中在 [run/input.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts)：

| 项 | 值 |
|---|---|
| 启动请求体上限 | `MAX_START_BODY_BYTES` = 64 KB |
| 命令请求体上限 | `MAX_COMMAND_BODY_BYTES` = 1 MB |
| requirement 长度 | 20,000 字符 |
| 画像单字段长度 | 2,000 字符 |
| 预置 agent 数上限 | 20 |
| voice id 长度 | 256 字符 |
| 媒体 elementId 长度 | 128 字符 |
| `outlineReview` 默认值 | `'wait'` |
| 超体积 | 413（`readJsonBody` 直接拒绝） |

确认大纲时会**按列表位置重排 `order = index + 1`**
（[input.ts#L241-L268](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L241-L268)），
所以用户可以自由增删与排序，服务端不信任客户端传来的 `order`。

请求里**永远不带 API key 和 model 名**——全部由 owner 的能力槽决定，这是本次改版的
核心设计（路由头注释原话：“No keys and no models: the owner's capability slots decide.”）。

---

## 9. 失败体系：从 `StepRefusal` 到 `GenerationRunFailure`

一条失败链有四层，学习时按这个顺序读最省力：

```
① 能力层：throw StepRefusal(reason: 本步专属枚举, statusCode?)
      ↓（retry.ts: StepRefusal→refusalStatus; httpStatus→自身; 否则 upstreamHttpStatus ?? 500）
② 重试层：withRouteRetry 判定“可重试 / 不可重试”
      不可重试 = host.isNonRetryableHostFailure ∥ host.isProviderQuotaRefusal ∥ 4xx 语义错误
      ↓ 重试用尽仍失败
③ 分类层：runFailureCode(error, ...) → 11 个内置 errorCode
      ↓
④ 记录层：GenerationRunFailure { step, message, errorCode, statusCode, resumeState }
      → 写 run.error + 发 step_failed 事件 + state='paused'
```

### 9.1 内置 errorCode（11 个）

[run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts)：

| errorCode | 触发条件（**判定顺序自上而下**） |
|---|---|
| `PROVIDER_DISABLED` / `MISSING_API_KEY` / `INVALID_URL` / `MODEL_CONFIG_INVALID` | `ModelAuthorizationError`（槽位没授权/没 key/URL 非法/配置错） |
| `MODEL_CONFIG_INVALID` / `PROVIDER_DISABLED` | `ModelConfigurationError` 或 `WebSearchConfigError` |
| `GENERATION_FAILED` | `StepRefusal`（能力层主动拒绝，例如大纲 prompt 不可用、装配失败） |
| `PROVIDER_QUOTA_EXHAUSTED` | `isUpstreamQuotaExhausted()`（配额类措辞） |
| `RATE_LIMITED` | HTTP 429 |
| `UPSTREAM_ERROR` | 其他 4xx/5xx |
| `INTERNAL_ERROR` | 兜底；也包括“接管次数超限”“检查点缺失”等引擎内部异常 |

`runRequirements`、宿主 `classifyHostFailure` 钩子可以进一步把 provider 的原始错误
分类成 host 自定义码；`reportGenerationRunEvent` 则把失败事件透传给宿主
（多租户场景记账、告警）。

### 9.2 每步自己的 `StepRefusal.reason`

| 步骤 | reason 枚举 | 定义位置 |
|---|---|---|
| 材料分析 | `provider-cannot-extract` / `unknown-provider` / `unsupported-type` / `endpoint-refused` / `no-content` / `service-required` | [material-analysis.ts#L68-L75](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L68-L75) |
| 联网检索 | 无（不拒绝；能力没配就返回 `null`，由 services 决定是否插步） | [services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) |
| 大纲 | `prompt-unavailable` | [outline.ts#L346](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L346) |
| Agent 画像 | `unparseable` / `too-few-agents` / `teacher-count` | [agent-profiles.ts#L16-L60](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L16-L60) |
| 场景内容 | `generation-failed` | [scene-content.ts#L66](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L66) |
| 场景动作 | `assembly-failed` | [scene-actions.ts#L50](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L50) |
| 旁白 TTS | `voice-missing` / `client-side-provider` / `voice-context-required` / `missing-api-key` | [narration.ts#L38-L46](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L38-L46) |
| 图片 | `missing-api-key` / `missing-model` | [image.ts#L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/image.ts#L33) |
| 视频 | `missing-api-key` / `missing-model` / `task-connection-changed` | [video.ts#L60-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/video.ts#L60-L65) |

---

## 10. 超时、重试、并发常量总表

这一张表是排障时的第一手参考。

### 10.1 步骤超时（[run/deadline.ts#L8-L26](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L8-L26)）

| 步骤 | `STEP_DEADLINES_MS` |
|---|---|
| `materialAnalysis` | 300 000（5 分钟） |
| `research` | 300 000 |
| `outline` | 300 000 |
| `agentProfiles` | 120 000 |
| `sceneContent` | 300 000 |
| `sceneActions` | 60 000 |
| `narrationClip` | 30 000 |
| 媒体（图/视频，[run/media.ts#L63](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts#L63)） | `MEDIA_DEADLINE_MS` = 300 000 |

超时抛 `StepTimeoutError`（`statusCode = 504`，[deadline.ts#L28-L41](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L28-L41)）。
`withDeadline()` 用 `AbortSignal.any([runSignal, timer])` 把“run 被 abort”与“单步超时”区分开：
前者透传原始 reason，后者才造 504。

### 10.2 重试（两层）

| 层 | 位置 | 数值 |
|---|---|---|
| 引擎：首个场景的重试次数 | [retry.ts#L22](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L22) `FIRST_SCENE_MAX_RETRIES` | 2（首屏体验优先，快速失败） |
| 引擎：其余场景的重试次数 | [retry.ts#L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L24) `SCENE_MAX_RETRIES` | 5 |
| 引擎：重试退避基数 | [engine.ts#L423-L500](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L423-L500) `retryOptions()` | 内容/动作 500ms、旁白 400ms 起 |
| 引擎：`takeovers` 上限 | [config.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/config.ts) ← `agentRuntimeConfig.maxAttempts` | 超限即 `paused` |
| 能力层：大纲流式重试 | [outline.ts#L57](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L57) `MAX_STREAM_RETRIES` | 2（另可 fallbackModel 再来一轮） |
| 媒体：放置尝试 | [media.ts#L307](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts#L307) `MAX_PLACEMENT_ATTEMPTS` | 3 |
| 媒体：provider task 记录 | [media.ts#L313](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts#L313) `TASK_RECORD_ATTEMPTS` | 3 |
| LLM SDK 自身重试 | 各步骤 `callLLM(..., maxRetries: 0)` | **0**（重试权归引擎，避免双重放大） |

`withRouteRetry()`（[retry.ts#L71-L108](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L71-L108)）
内部复用 `withGenerationRetry`，但额外做三件事：
① 用 `routeStatus()` 把异常归一成 HTTP 码；② `passthrough` 与
`isProviderQuotaRefusal` 一律不重试；③ 每次重试**先提交 `step_retry` 事件**再退避。
它的 `options.rethrow` 是专门为“挂死的 TTS 请求错误信息里也含 `timed out`，但不该重试”
准备的开关。

### 10.3 运行时配置（[run/config.ts#L48-L60](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/config.ts#L48-L60)）

| 配置 | env 变量 | 默认 |
|---|---|---|
| 单进程并发执行 run 数 | `OPENMAIC_GENERATION_RUN_MAX_CONCURRENT` | 4 |
| 每 owner 同时在跑的 run | `OPENMAIC_MAX_ACTIVE_RUNS_PER_OWNER` | 2 |
| 每 owner 等待确认的 run | `OPENMAIC_MAX_WAITING_RUNS_PER_OWNER` | 10 |
| 已完成 run 保留期 | `OPENMAIC_GENERATION_RUN_RETENTION_HOURS`（小时） | 24 |
| 压缩周期 | — | 60 分钟 |
| 扫描 / 心跳 / 租约 TTL / 最大接管 | 复用 `agentRuntimeConfig`（`OPENMAIC_AGENT_*`） | 见 agent runtime |

### 10.4 大纲与场景的规模上限

| 项 | 值 | 位置 |
|---|---|---|
| 单 run 最多场景数 | 100 | [outline-schema.ts#L20](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L20) |
| 大纲 JSON 上限 | 512 KB | outline-schema.ts |
| 大纲流缓冲上限 | 512 KB | [outline.ts#L61](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L61) |
| 单 run 媒体生成上限 | 20 | outline-schema.ts `MAX_MEDIA_GENERATIONS` |
| 单场景题目数上限 | 100 | outline-schema.ts `MAX_QUESTION_COUNT` |
| 自动确认倒计时 | 2500 ms | [types.ts#L59](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L59) |

---

## 11. 大纲审查：三种模式

| 模式 | 行为 | 状态轨迹 |
|---|---|---|
| `wait`（默认） | 大纲生成后**一直等** `confirm-outline` | `outlining → awaiting_outline_confirmation → (命令) generating` |
| `countdown` | 生成后落 `outline_auto_confirm_at = now+2500ms`，**任意进程**的 runner 到点自确认 | 同上，但无需人工 |
| `auto` | 大纲就绪即同事务确认，UI 基本看不到等待态 | `outlining → generating` |

`hold-outline` 命令的作用是把一个 `countdown` run **降级为 `wait`**（清空
`outline_auto_confirm_at`），语义是“用户点了暂停，别再自动确认了”，见
[store.ts#L1466-L1506](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1466-L1506)。
到点确认由 [confirmDueGenerationRunOutlines()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1508-L1544)
在每次 `scan()` 最前面执行（一次最多 10 个），因此**即使前端全关掉，倒计时也会走完**。

确认命令可以携带**修改过的大纲列表**（增删、改标题、改类型、重排），服务端会
`parseOutlines()` + `normalizeSceneOutlines()` 再校验一遍（第 3 册 §6）。

---

## 12. 提速与并行：prewarm 与媒体泳道

两个“看起来复杂但收益很大”的机制，细节分别在
[第 5 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md) 与
[第 7 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-7.md)。

- **内容 prewarm**：大纲确认后就按 `parallelSceneConcurrency` 用 `lazyBoundedMap`
  并行生成各场景**内容**（第 5 步），主线跑到 `scene:n:content` 时直接取预热结果。
  prewarm 的信号与主线分离，主线被 abort 时 prewarm 也跟着收；
  某个场景 prewarm 失败**不立即中断 run**，而是记录下来、等主线跑到它时再按正常
  重试路径处理，实在不行“最后 pause”——这样不会因为第 40 页抖动而打断第 1–39 页的产出。
  代码入口 [engine.ts#L541-L590](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L541-L590)。
- **媒体泳道**：图片/视频不在主线占步骤，而是每个元素一个 `media:<elementId>` 检查点，
  由 [runMediaLane()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/media.ts#L252-L307)
  逐项推进；产物存好后由 `placeMedia` 在 `exclusive()` 里塞进场景文档。
  视频提交后立刻把 provider task 记进检查点（`submitted`），**进程重启可以续等而不是重投**。
  run 已 `completed`/`paused` 但还有媒体没做完时，`media_pending` 置真，
  claim 会读这个旗标而不是读状态（[generation-runs.ts#L20-L27](file:///D:/GITHUB_chat/OpenMAIC/lib/persistence/generation-runs.ts#L20-L27)）。

---

## 13. 完成与清理

| 环节 | 位置 | 说明 |
|---|---|---|
| 建课 | [document.ts createRunCourse()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts) | 与**首个场景**同一事务写文档，`producer:'server-job'`、`generationComplete:false` |
| 追加场景 | `appendRunScene` / `mutateRunScene` | 都先 `fencedStore()` 校租约，再走 `exclusive()` |
| 课程收尾 | `completeRunCourse` | 置 `generationComplete=true`，此后用户可正常编辑 |
| 课程被删 | `RunCourseDeletedError` → run `ended` | 主循环专门捕获，见 [engine.ts#L1750-L1877](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1750-L1877) |
| 反向保护 | `endGenerationRunsOfDeletedCourseIn()` | 删课事件里把还在写的 run 一律置 `ended` |
| 资产保活 | [keepGenerationRunAssetsAlive()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1319-L1331) | 未完成的 run 定期给引用的生成资产续 TTL |
| 历史压缩 | [compactFinishedGenerationRuns()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L344-L390) | 先折算 `media_summary`/`narration_unvoiced` 再删检查点 |
| 宿主通知 | [hook-events.ts reportCommittedRunEvents()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/hook-events.ts) | `outline_ready→'outline-ready'`、`scene_ready→'scene-appended'`、`state(paused)→'paused'`、`completed→'completed'`、`ended→'ended'` 五类 |

---

## 14. 客户端怎么“看”和“控”一个 run

`lib/generation-run-client/` 是浏览器侧的唯一入口（与 `packages/` 不同，它属于主应用）：

| 文件 | 作用 |
|---|---|
| [api.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/api.ts) | 8 个 fetch/SSE 封装 |
| [reducer.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts) | 事件 → 视图状态（幂等、按 seq 去重） |
| [follower.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts) | SSE 优先、失败退化轮询；`resync` 处理 |
| [run-id.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/run-id.ts) / [run-course.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/run-course.ts) | run 与课程文档的关联读取 |
| [run-media.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/run-media.ts) | 媒体状态与单元素重试 |
| [commands.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/commands.ts) | `commandId` 生成与重试幂等 |
| [outline-review.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/outline-review.ts) | 大纲确认 UI 的数据层 |
| [preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) | run step → UI 步骤（§0.2） |
| [failure-message.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/failure-message.ts) | `errorCode` → 人话 |
| [use-generation-run.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/use-generation-run.ts) / [use-owner-runs.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/use-owner-runs.ts) | React hooks |
| [start.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/start.ts) / [use-course-materials.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/use-course-materials.ts) | 首页启动前的材料与参数组装 |

预览页在 [app/generation-preview/page.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx)。

---

## 15. 建议阅读顺序

1. 本文 §1–§4（分层、时序、状态机、计划）——先建立“一条 run 是怎么被推进的”心智模型。
2. [第 3 册 大纲](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-3.md)——
   它是唯一“流式 + 可人工干预”的步骤，把事件、审查模式、schema 校验一次性串起来。
3. [第 1 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-1.md) →
   [第 2 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-2.md) →
   [第 4 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-4.md)——按管线顺序补齐准备段。
4. [第 5](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md) →
   [第 6](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md) →
   [第 7](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-7.md) 册——生成段（含并行与媒体）。
5. 最后回到本文 §6、§9、§10 精读租约/失败/常量，这时每一行数值都能对上代码。

---

## 16. 动手验证

| 想验证什么 | 怎么做 |
|---|---|
| 计划与续跑语义 | 读 `tests/server/` 下 run plan / store / engine 相关测试（搜 `generation/run`），改场景数看 `advanceRun` 结果 |
| 状态机与命令幂等 | 连续发两次同 `commandId` 的 `confirm-outline`，第二次应返回第一次的 `result` |
| 接管与围栏 | 跑起一个 run 后 `kill` 掉进程，等 `leaseTtlMs` 看 `takeovers` 递增与新 worker 认领 |
| 事件流 | `curl -N 'http://localhost:3000/api/generation-runs/<run-id>/events?after=0'`，观察 `caught_up` 与 `outline_item` 的逐条下发 |
| 失败码 | 故意把 `course.outline` 槽的 key 清空后启动，看 `step_failed.errorCode` 与 `run.error` |
| 超时 | 把 `sceneActions` 的 deadline 临时改小，观察 504 与 `step_retry` 事件 |

---

## 17. 一句话总结

**生成管线 = 一份可重放的计划（plan.ts）+ 一个带租约的推进循环（engine.ts）+
七个不碰 HTTP 的纯步骤（steps/*）+ 四张 Postgres 表（权威状态）。**
用户看到的“7 步进度条”只是 run 自己上报的步骤序列在 UI 上的一次投影；关掉浏览器，
它照样跑完。
