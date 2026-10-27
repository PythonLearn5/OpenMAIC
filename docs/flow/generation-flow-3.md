# 课程生成管线 · 第 3 册：大纲生成与审查（outline）

<!--
  职责：详解生成管线**第 3 步「大纲」**——把需求 + 材料 + 联网检索结果喂给
  `course.outline` 槽的模型，流式解析出 1~100 条场景大纲（SceneOutline），
  归一化成"永远可被确认"的形状，写入 run 的 `outline` 列，并按三种审查模式
  （wait / countdown / auto）决定这条 run 是自己继续、等 2.5 秒倒计时、还是停下
  等 `confirm-outline` 命令。
  覆盖：`runOutlineStep` 逐段解读、`prepareOutline` 的三套 prompt 模板与视觉图切片、
  增量 JSON 解析器与两级缓冲上限、`MAX_STREAM_RETRIES` + 回退模型的完整决策树、
  `outline-schema.ts` 归一化器的"宽松 vs 严格"边界、审查闭环的三个命令与到期结算、
  事件与前端大纲卡片、失败链、常量、坑与验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 2 册 联网检索](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-2.md)｜下一步：[第 4 册 Agent 画像](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-4.md)
- 源文件：[steps/outline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts)
  · [outline-schema.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts)
  · [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [run/store.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `outline`（id 与 kind 同名，无序号） |
| 是否必经 | **无条件步**：`preparationSteps` 里写死，见 [plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54) |
| 所处 run 状态 | `outlining`（[plan.ts#L67-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L72) 的 `phaseOfStep`，本步是唯一独占一个 phase 的步） |
| 检查点写在哪 | `generation_run_steps(run_id, 'outline')`，值为 `OutlineResult`；**同时**把大纲抄进 `generation_runs.outline` / `outline_revision` / `scenes_total` |
| 时间预算 | 300 s（[deadline.ts#L14-L16](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L14-L16) 的 `STEP_DEADLINES_MS.outline`） |
| 进程内重试 | **本步自己管**：同模型最多 3 次 + 回退模型 1 轮，见 [outline.ts#L57](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L57)；外层 `withRouteRetry` 被显式关掉 |
| UI 显示 | 预览页第 2 个位（`outline`），且流式期间就在这一位上"长出来"，见 [preview-steps.ts#L27-L32](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L27-L32) |
| 产出消费方 | 第 4 步 `agents`、第 5/6/7 步全部 `scene:n:*`（都从 `run.outline.outlines[i]` 取自己那条），以及 `advanceRun` 的 `sceneCount` |
| 独有之处 | 7 步里唯一**会改写 run 状态机**的步：它可以把 run 停在 `awaiting_outline_confirmation` 并**释放租约** |

### 0.2 一句话本质

> **第 3 步 = 一次流式 JSON 生成 + 一个增量解析器 + 一个"能不能确认"的裁判。**
> 模型写的字节一边流一边被切成一条条完整对象，每切出一条就发一个 `outline_item`；
> 流结束后统一过 `normalizeSceneOutlines`，把它变成"确认命令可以原样接受"的规范形；
> 最后按 `outlineReview` 决定 run 是**自己往下走**、**等 2.5 秒自己走**，还是**交还租约等人点确认**。

前两步（材料、检索）产出的都是"素材"，从这一步开始产出的是**计划**——
后面 4 步（角色 + 3×N 个场景步）全部按这份计划的长度展开，所以这一步也是
**唯一一个"步骤数量由它决定"的步**。

### 0.3 整体时序

```
engine: case 'outline' ──► runOutlineStep(stepId)
   │
   ├─ 读检查点: output<ResearchOutput>('research')        ← 第 2 册的两条通道
   ├─ 读检查点: output<MaterialOutput>('material-analysis')
   │
   ├─ services.outline(owner, {...}, { log, signal, emit })
   │     └─ stageModel(owner,'scene-outlines-stream')  → course.outline 槽
   │        └─ generateOutlines = prepareOutline → streamOutlines
   │              ├─ 三套模板之一（standard / interactive / task-engine）
   │              ├─ 视觉图：排序 → 取前 20 → resolveVisionImages
   │              ├─ streamLLM(...).fullStream  (route retry: enabled:false)
   │              └─ 每个 chunk: 抽 directive / title / 新 outline → emit
   │
   ├─ emit('outline'|'courseTitle'|'languageDirective'|'retry')
   │     └─ append() 串行落事件（outline_item / outline_reset + step_retry）
   │
   ├─ normalizeSceneOutlines(result.outlines)   ← 不可用就抛，run 暂停
   │
   └─ StepCommit: patch{state, outline, outlineRevision:1, scenesTotal, releaseLease?}
         auto      → state 'generating'                      继续跑
         countdown → state 'awaiting_outline_confirmation' + outline_auto_confirm_at=now+2.5s
         wait      → state 'awaiting_outline_confirmation' + 等 confirm-outline
```

---

## 1. run 侧：`case 'outline'` 只是转发

[engine.ts#L660-L661](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L660-L661) 只有一行：

```ts
case 'outline':
  return runOutlineStep(stepId);
```

真正的内容在同文件的 [runOutlineStep](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L772-L872)（约 100 行）。
它是 7 步里唯一"不是 `done(...)` 一行搞定"的分支，原因有两个：**要边流边发事件**，
**要按审查模式改 run 状态**。逐段看：

### 1.1 输入从两个上游检查点拼出来（L773-L774）

```ts
const research = output<ResearchOutput>('research');
const pdfText = output<MaterialOutput>('material-analysis')?.pdfText;
```

这两行是本册与第 1、2 册的接缝，四个细节：

| 细节 | 事实 |
|---|---|
| `research` 可缺 | 第 2 步空跑时检查点是 `{webSearch:false}`，`research?.context` 为 `undefined` → 根本不传 `researchContext`（L822） |
| 布尔通道的唯一用途 | `runRequirements(input, research?.webSearch === true)`（[engine.ts#L819](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L819)）——注意是 `=== true`，缺检查点等价于 `false` |
| 文本通道只在这一步 | 整个 run 里只有大纲读 `research.context`；第 5 步首场景只读那个**布尔**（见第 2 册 §8） |
| 材料是"可选展开" | `...(pdfText ? { pdfText } : {})`（L820）：空字符串不会被误当成"有材料" |

### 1.2 事件串行落库的 `append` 队列（L777-L785）

```ts
let appending: Promise<unknown> = Promise.resolve();
let appendFailure: unknown;
const append = (events: NewGenerationRunEvent[]) => {
  appending = appending.then(() => commit({ events })).catch((error) => { appendFailure ??= error; });
};
```

为什么要手搓一个队列：`streamOutlines` 是在 `for await` 里同步调 `emit` 的，
而 `commit` 是异步的（要写 `generation_run_events` 并 `notify`）。
如果不串起来，两条事件的 `seq` 会乱序——大纲卡片会先出现第 3 条、再补第 1 条。
所以这里把每次 `commit` 挂到上一次的 `.then` 后面，**保证 seq 单调、顺序与模型输出一致**。

配套的三条纪律：

1. `catch` 里用 `appendFailure ??= error`：**只留第一个错**，后面的错误不覆盖它；
2. 队列失败**不会**打断流（模型还在写，继续浪费它干什么）——它只是被记下来；
3. `finally { await appending; }`（L827-L829）后立刻 `if (appendFailure) throw appendFailure;`（L830），
   于是"事件写不进去"和"步骤本身失败"走同一条暂停路径。

### 1.3 `emit` 的四种事件映射（L786-L812）

| 步骤发出的 | run 事件（落库名） | 说明 |
|---|---|---|
| `languageDirective` | `outline_language_directive` | 语言基调，后续所有步都用它 |
| `courseTitle` | `outline_course_title` | 页面标题 / 后面 `agents` 步的 stage name |
| `outline` | `outline_item`（带 `index`） | **每解析出一条就发一条** |
| `retry` | `outline_reset` + `step_retry` | 一次发两条：清空 + 报告第几次尝试 |

注意 `{ data: { data: event.data } }` 这种"套一层"的写法（L789、L792）不是笔误：
run 事件的 `data` 是对象，字符串载荷就放在它的 `data` 字段里。

`outline_reset` 的两处来源要分清：
- 这里（retry 时）——同一步内重置；
- [engine.ts#L1820-L1822](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1820-L1822)——**Retry 命令重启大纲步**时才补发（`restarted && step.kind === 'outline'`）。

### 1.4 提交：一步同时改了 5 个东西（L846-L871）

```ts
const automatic = input.outlineReview === 'auto';
const next = automatic ? 'generating' : 'awaiting_outline_confirmation';
return {
  step: { id: stepId, output: result },
  patch: {
    state: next, step: null, outline: confirmed,
    outlineRevision: 1, scenesTotal: result.outlines.length,
    ...(automatic ? {} : { releaseLease: true, outlineAutoConfirm: true }),
  },
  events: [ step_completed, outline_ready, ...(automatic ? [outline_confirmed] : []), state ],
};
```

| 字段 | 为什么这么写 |
|---|---|
| `outlineRevision: 1` | 每次生成都把大纲重置到 revision 1；只有"带 edits 的确认"才 +1（见 §9.3） |
| `scenesTotal` | 进度条的分母在这里第一次被真值覆盖（创建 run 时它是 0） |
| `step: null` | 大纲结束后 run 不再"处于任何步"，等确认期间没有步在跑 |
| `releaseLease: true` | **等人的 run 不占 worker**：租约清空，别的进程可以认领别的 run |
| `outlineAutoConfirm: true` | 只是一个开关，落库时**再查一次** `input->>'outlineReview'` 才决定要不要写时间戳（[store.ts#L473-L479](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L473-L479)） |
| 非 auto 才发 `outline_confirmed` | auto 模式下这条确认事件**由本步替 run 发**；wait/countdown 模式则由命令或到期结算发 |

`outlineAutoConfirm` 这个"开关 + SQL 里再判一次"的设计是为了兑现一条注释里的承诺：
在大纲流式期间，用户可能按下了 hold——把 `countdown` 改成了 `wait`（§9.4）。
commit 时的 `input` 已经落库，所以判定必须交给 SQL，而不是在 JS 里读早先的 `input`。

---

## 2. `services.outline`：这一步的凭据从哪来

[run/services.ts#L444-L453](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L444-L453)：

```ts
async outline(ownerId, input, ctx) {
  return generateOutlines(
    { ...input, model: await stageModel(ownerId, 'scene-outlines-stream') },
    { ...ctx, workspaceId: await backgroundWorkspaceId(ownerId),
      resolveVisionImages: ownerVisionImages(ownerId, ctx.log) },
  );
},
```

| 项 | 值 / 事实 |
|---|---|
| LLM stage | `scene-outlines-stream`（[model-routes.ts#L143](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-routes.ts#L143) 的 stage 清单） |
| 落到哪个槽 | `course.outline`（[model-slots.ts#L96](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L96)） |
| 是否可缺 | **不可缺**：`stageModel` 抛 `SlotUnassignedError` / `SlotDisabledError`，本步直接失败并暂停 run（不像第 2 步会空跑） |
| `workspaceId` | 给能力探针用（图片/视频槽是否开着，见 §4.3） |
| `resolveVisionImages` | owner 的资产解析器：把大纲里引用的图片 id 变成真实字节/URL（§4.2） |

`stageModel` 在**调用模型之前**执行，所以"没配大纲槽"表现为本步立刻失败，
错误码走 [failure-code.ts#L44-L45](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L44-L45) 的 `ModelAuthorizationError`/`ModelConfigurationError` 分支。

---

## 3. `generateOutlines` = 准备 + 流式

[outline.ts#L805-L810](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L805-L810)：

```ts
export async function generateOutlines(input, ctx) {
  return streamOutlines(await prepareOutline(input, ctx), ctx);
}
```

两半都可以单独测：`prepareOutline` 只产 prompt（不发请求），`streamOutlines` 只消费
`PreparedOutline`。这也是文件头注释（[L1-L11](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L1-L11)）强调的两件事：
**prompt 与 1.1.x 的 `POST /api/generate/scene-outlines-stream` 字节等价**，
**流在同模型上重试，然后换回退模型跑一轮，每次重试都上报事件**。

### 3.1 `OutlineInput` 的 8 个字段

[L303-L319](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L303-L319)：

| 字段 | 来源 | 备注 |
|---|---|---|
| `requirements` | `runRequirements(input, webSearch)` | 需求 + 昵称/简介 + 三个模式开关 |
| `pdfText?` | 第 1 步检查点 | 进 prompt 前会被 `MAX_PDF_CONTENT_CHARS` 截 |
| `pdfImages?` / `imageMapping?` | 第 1 步的 `materialImages()` | 只有两者都有才可能走视觉模式 |
| `researchContext?` | 第 2 步的 `context` | 缺省时 prompt 里填 `'None'` |
| `agents?` | —— | run 里**不传**（角色在第 4 步才生成），故 `teacherContext` 为空 |
| `model` | `stageModel(...)` | 类型是 `StepLanguageModel`，含 `modelInfo` / `serverManaged` / `thinkingConfig` |
| `allowImageGeneration?` / `allowVideoGeneration?` | —— | **只能 opt-out**，见 §4.3 |

### 3.2 步骤自己的类型面

| 类型 | 位置 | 内容 |
|---|---|---|
| `PreparedOutline` | [L322-L329](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L322-L329) | `model` + `requirement` + `taskEngineMode` + `prompts` + `visionImages?` |
| `OutlineEvent` | [L332-L337](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L332-L337) | 4 个变体，`retry` 带 `attempt/maxAttempts/fallback?` |
| `OutlineResult` | [L339-L344](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L339-L344) | `outlines` + `languageDirective` + `courseTitle?` + `taskEngineMode` |
| `OutlineRefusal` | [L346](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L346) | 只有一个值：`'prompt-unavailable'` |
| `OutlineGenerationError` | [L349-L354](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L349-L354) | 所有尝试都没产出大纲时抛，`cause` 是最后一次真错 |

`OutlineResult` 就是本步的**检查点结构**（engine 的 `step.output = result`），
它和 `run.outline`（`GenerationRunOutline`）的区别见 §9.1。

---

## 4. `prepareOutline`：三套模板、两种视觉路径、两个媒体门

### 4.1 三套模板怎么选

[L474-L508](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L474-L508)：

```ts
const interactiveMode = requirements.interactiveMode ?? false;
const taskEngineMode = resolveVocationalActive(requirements);
let prompts = buildOutlinePrompt(requirements, { ... });     // 标准分支（packages）
if (taskEngineMode || interactiveMode) { prompts = buildPrompt(promptId, { ... }); }
if (!prompts) throw new StepRefusal('prompt-unavailable', 'Prompt template not found');
```

| 模式 | prompt 来自 | 优先级 |
|---|---|---|
| 标准 | `@openmaic/generation` 的 `buildOutlinePrompt`（**与包内路径字节等价**） | 兜底分支 |
| 交互 | app 侧模板 `interactive-outlines`（[prompts/index.ts#L27](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/index.ts#L27)） | 被 task-engine 压制 |
| 任务引擎 | app 侧模板 `task-engine-outlines`（[L28](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/index.ts#L28)） | **最高** |

两个要点：
1. `resolveVocationalActive`（[feature-flags.ts#L121-L125](file:///D:/GITHUB_chat/OpenMAIC/lib/config/feature-flags.ts#L121-L125)）= 请求里 `taskEngineMode` **且** `OPENMAIC_ENABLE_VOCATIONAL` 打开。默认关，关掉时**静默**退回标准/交互路径——不是报错。
2. 两套分支的变量名不同：标准分支由包内函数决定注入什么，app 分支手工列了 9 个变量（`pdfContent` / `availableImages` / `researchContext` / `hasSourceImages` / `imageEnabled` / `videoEnabled` / `mediaEnabled` / `teacherContext` / `userProfile`）。`userProfile` **只在 app 分支**有（[L368-L371](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L368-L371) 构造的 `## Student Profile` 段）。

模板取不到（`buildPrompt` 返回 `null`）抛 `StepRefusal`，在
[failure-code.ts#L51](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L51) 归成 `GENERATION_FAILED`。

### 4.2 视觉图片：排序 → 切 20 → 解析 → N3 回灌

[L387-L454](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L387-L454)。只有 `pdfImages.length > 0` 且**模型有 vision 能力**且**有 imageMapping** 才走视觉分支：

```ts
const hasVision = !!modelInfo?.capabilities?.vision;                 // L374
const sortedImages = sortDocumentImagesForVision(pdfImages);         // L390
const allWithSrc = sortedImages.filter((img) => imageMapping[img.id]);
const visionSlice = allWithSrc.slice(0, MAX_VISION_IMAGES);          // 前 20 张
const textOnlySlice = allWithSrc.slice(MAX_VISION_IMAGES);           // 其余只留文字描述
const noSrcImages = sortedImages.filter((img) => !imageMapping[img.id]);
const resolvedVisionImages = await prepareVisionImages(await ctx.resolveVisionImages(...), ctx.log);
```

| 概念 | 事实 |
|---|---|
| `MAX_VISION_IMAGES` | 20（[constants/generation.ts#L10](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L10)） |
| 排序 | `sortDocumentImagesForVision`：让"值得看"的图进前 20（第 1 册 §6 讲过同一函数） |
| `imageMapping` 的值 | 第 1 步 mint 出的**资产 id**，不是 base64；`resolveVisionImages` 把它换成真实字节 |
| 解析失败的图 | 在这里被**丢弃**，同时它的 `[see attached]` 文本也从 prompt 里去掉（L436-L441 的 `resolvedPdfImages` / `resolvedImageMapping`，注释里叫 N3） |
| 为什么这里不会"后移补位" | L442-L450 长注释：`visionImages` **就是**已解析的那一片，不再二次切片；而第 5 步内容生成会重切，所以那边可能出现"补位"，这边不会 |

两种描述函数分工：进视觉片的用 `formatImagePlaceholder`（带 `[see attached]`），
其余（超出 20 的 + 没有 src 的）用 `formatImageDescription`（纯文字），拼成 `availableImagesText`（L421）。

### 4.3 媒体能力门：只能 opt-out

[L457-L467](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L457-L467)：

```ts
const capabilities = await resolveServerGenerationCapabilities(ctx.workspaceId);
const imageGenerationEnabled = capabilities.imageGeneration && input.allowImageGeneration !== false;
const videoGenerationEnabled = capabilities.videoGeneration && input.allowVideoGeneration !== false;
const mediaGenerationEnabled = imageGenerationEnabled || videoGenerationEnabled;
```

- 决定权在**工作区的 image / video 槽**（与生成管线、capabilities 端点读同一份事实）；
- 调用方能做的是 `!== false` 这种"关掉"，**永远打不开槽没给的东西**；
- 这三个布尔只影响 prompt 里能不能出现 `mediaGenerations`——真正的生成在第 7 册的媒体泳道。

### 4.4 `teacherContext` 在本步永远是空

[L471](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L471) 调 `formatTeacherPersonaForPrompt(agents)`，
但 run 的 `services.outline` **不传 `agents`**（第 4 步才生成角色，顺序在大纲之后）。
这条"看起来能传但实际不传"的线是给别的调用方（agent runtime 的工具）留的——
读代码时别以为大纲知道老师是谁。

---

## 5. 增量解析器：流式的大纲为什么能"长出来"

### 5.1 `extractNewOutlines`：一次扫描、可续扫

[L133-L194](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L133-L194)。它不是 `JSON.parse` 的尝试性调用，而是**手写的花括号深度扫描器**：

| 设计 | 实现 | 为什么 |
|---|---|---|
| 只扫一次 | `scanFrom` 游标跨 chunk 传递（L140-L151、L193） | 每个 chunk 都重扫全 buffer 会退化成 O(n²) |
| 定位数组 | `buffer.indexOf('"outlines"')` → 其后的第一个 `[`；没有包装对象就退到全文第一个 `[`（L146-L148） | 同时支持 `[{...}]` 和 `{"languageDirective":..,"outlines":[..]}` |
| 字符串感知 | `inString` / `escaped` 两个状态（L155-L169） | 描述里出现的 `{` `}` 不算深度 |
| 宽容失败 | `JSON.parse` 包在 `try{}catch{}`，解析不了就**跳过**（L182-L186） | 一个畸形对象不该毁掉整轮 |
| 不剥 markdown | 靠"按内容定位数组"而不是"先去掉 ```json" | 模型加不加围栏都能吃 |
| `consumed` | 只在成功 parse 后前移（L188） | parse 失败的对象会被**再扫一次**（下一个 chunk 补齐后可能就合法了） |

### 5.2 `languageDirective` 与 `courseTitle`：只在头部找

[L68-L121](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L68-L121)：

```ts
const head = buffer.length > 8192 ? buffer.slice(0, 8192) : buffer;
const match = head.match(/"languageDirective"\s*:\s*"((?:[^"\\]|\\.)*)"/);
```

| 细节 | 事实 |
|---|---|
| 8192 窗口 | 因为这两个键是包装对象的开头键，永远出现在 buffer 头部；不设窗口就又是 O(n²) |
| 解码 | 用 `JSON.parse(`"${m[1]}"`)` 解转义，解不动就原样返回（L76-L80） |
| 标题归一 | `normalizeStreamedTitle`（L95-L104）：trim、空则视为没有、**截到 120 字符** |
| 全 buffer 兜底 | 流结束后若还没标题，跑一次 `extractCourseTitleFromComplete`（L118-L121，仅一次、不限头部） |
| 触发时机 | L713-L718：只有**已经解析出大纲**且没标题时才兜底 |

### 5.3 每条大纲落地前做三件事

[L685-L698](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L685-L698)：

```ts
const enrichedBase = { ...outline, order: parsedOutlines.length + 1 };   // 重排 order
const normalized = taskEngineMode
  ? normalizeTaskEngineOutline(enrichedBase, requirement)
  : sanitizeNonTaskEngineOutline(enrichedBase);
const enriched = ensureUniqueOutlineId(normalized, usedOutlineIds);      // 保证 id 唯一且非空
parsedOutlines.push(enriched);
emit({ type: 'outline', data: enriched, index: parsedOutlines.length - 1 });
```

| 处理 | 规则 |
|---|---|
| `order` | **无条件重写**为 1..N，模型给的 order 不作数 |
| `id` | `ensureUniqueOutlineId`（L288-L301）：非空且不重复就用，否则 `nanoid()` 循环到不重复 |
| task-engine | `normalizeTaskEngineOutline`（L243-L261）：`slide` 剥掉 widget 字段；`interactive + procedural-skill` 补默认 `procedureType/tools/steps/successCriteria/errorConsequences`（L196-L228）；5 种普通 widget 原样放行（L241）；其余**一律降级为 slide** |
| 非 task-engine | `sanitizeNonTaskEngineOutline`（L263-L286）：模型擅自写 `procedural-skill` 时，删掉那 6 个字段、改成 `interactive + diagram`，并在 `description` 末尾追加"请以流程/结构图呈现" |

最后一条是**安全阀**：`procedural-skill` 只在 `taskEngineMode` 下存在，
防止没开职业开关的工作区被模型带进未审核的 widget 形态。

### 5.4 两级熔断：字节与条数

[L659-L664、L686、L699-L704](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L659-L704)：

| 上限 | 值 | 触发行为 |
|---|---|---|
| `MAX_OUTLINE_STREAM_BYTES` | 512 KB（UTF-8 **字节**，不是字符） | `log.warn` + `break`（**停止读取并 finalize**，不是抛错） |
| `MAX_OUTLINE_SCENES` | 100（来自 outline-schema） | 内层 `break` 后外层也 `break`，同样 warn + finalize |

两者的共同点：**已解析出的大纲仍然有效**。所以"跑飞的输出"被截断后，
run 可能拿到一份少于模型本意的大纲并照常确认；这是有意为之（防止堆被撑爆），
也是 §12 的一个坑。

---

## 6. 流式循环：重试、回退与"什么算失败"

### 6.1 循环边界

[L612-L616](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L612-L616)：

```ts
for (let attempt = 1; attempt <= MAX_STREAM_RETRIES + 1 && !(fellBack && attempt > 1); attempt++)
```

`MAX_STREAM_RETRIES = 2`（L57），翻译成人话：

- 同一模型最多 **3 次**尝试（attempt 1/2/3）；
- 之后**换回退模型跑 1 轮**：`maybeFallback` 成功后把 `attempt` 重置为 0 再 `continue`（L754、L780），
  于是 `fellBack` 为真时循环条件 `!(fellBack && attempt > 1)` 只放行 attempt==1 那一轮，跑完即退出。

### 6.2 为什么外层 route retry 被关掉

[L636-L638](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L636-L638)：

```ts
const fullStream = streamLLM(streamParams, 'scene-outlines-stream', thinkingConfig, { enabled: false }).fullStream;
```

`{ enabled: false }` 显式禁用 `withRouteRetry`。因为**这一步已经自己重试并且会重发事件**，
再在外层重试就会出现"重试了但 UI 不知道"的双重语义。对比：第 5/6/7 步的内容、动作、旁白
用的是外层 `withRouteRetry`（engine L509/L717/L1028），它们**不**自己重试。

### 6.3 消费的是 `fullStream` 而不是 `textStream`

[L627-L653](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L627-L653) 有段长注释解释这点，
三条纪律对应三种 `part.type`：

| part | 处理 |
|---|---|
| `error` | 记进 `streamError` 后 `continue`（**不中断流**） |
| `finish` | 记 `finishReason`（用来识别 `content-filter`） |
| 其他非 `text-delta` | `continue` 忽略 |
| `text-delta` | 累加文本与字节，跑三个提取器 |

关键理由：401、key 无效这类"流开始前的错误"和"内容过滤"都是**以流分片的形式**出现的，
不抛异常。若只读 `textStream`，它们会伪装成"空响应"，误走空输出回退路径。

另外每个 chunk 头部都检查 `signal?.aborted`（L643），一发现调用方走了就抛 `StepAbortedError`
——注释原话："Stop doing work the moment the caller goes away"。
`abortSignal` 也一并传给了 `streamLLM`（L557），两端都接。

### 6.4 判定顺序（每轮尝试结束时）

```
streamError 且 isNonRetryableHostFailure   → throw（即使已经流出了部分大纲也不算数）
parsedOutlines.length > 0                  → 需要时补抽标题 → break（成功）
streamError 存在                           → throw streamError（交给 catch 决策）
finishReason === 'content-filter'          → 记 lastError，warn，break（不重试不回退）
否则（空输出）                              → 记 lastError + 诊断 warn
     attempt <= 2 且未回退                  → emit retry，下一轮同模型
     否则                                   → maybeFallback(undefined, fullText)；成功则重置 attempt
```

`catch` 分支（[L758-L784](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L758-L784)）的判定顺序：

```
signal.aborted                → throw StepAbortedError（不烧重试）
isNonRetryableHostFailure     → throw 原错
记录 lastError + warn
非 quota 且 attempt<=2 且未回退 → emit retry，continue
maybeFallback(error) 成功      → attempt=0; continue
否则                          → break
```

两个谓词在 [generation-run-hooks/runtime.ts#L208-L220](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-run-hooks/runtime.ts#L208-L220)：

| 谓词 | 语义 |
|---|---|
| `isNonRetryableHostFailure` | 宿主分类器（`classifyHostFailure`）说 `retryable === false` → 立即失败，重试无意义 |
| `isProviderQuotaRefusal` | 是 provider 配额耗尽，**且宿主没有自己的分类** → 同模型重试没用，但**允许换回退模型** |

`content-filter` 单列的理由也写在注释里：**安全拒绝不是"输出为空"**，
不能走空输出 → 回退那条路（换个模型再问一遍同样的东西是没用的）。

### 6.5 `maybeFallback`：五道闸门

[L570-L604](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L570-L604)：

| 顺序 | 闸门 | 不通过时 |
|---|---|---|
| 1 | `!fellBack` | 已经回退过就不再回退 |
| 2 | `model.serverManaged` | **客户端自带的 x-model 永远不消耗运维的回退 key**（注释里明写的防滥用） |
| 3 | `shouldFallbackFor(error, text)` | 错误类型/空输出不 qualifies 就不换 |
| 4 | 能拿到 fallback 模型 | `attachedModelFallback(languageModel)`（槽自带的回退）→ 否则 `resolveFallbackModel()`；抛错则**保留原始错误**返回 false |
| 5 | `fallback` 非空 | 没配回退就不换 |

通过则：换 `streamParams.model`、置 `fellBack = true`、`logFallbackFired('scene-outlines-stream', ...)`、
发 `retry` 事件（`attempt = maxAttempts = MAX_STREAM_RETRIES + 1 = 3`，带 `fallback: modelString`）。

### 6.6 全失败：`OutlineGenerationError`

[L787-L793](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L787-L793)：一条大纲都没有 → `log.error` + 抛
`OutlineGenerationError(lastError || 'Failed to generate outlines', { cause: lastCause })`。
`lastError` 的三种可能文本值得记：

| 文本 | 场景 |
|---|---|
| `LLM returned empty response` | 流里一个 `text-delta` 都没有 |
| `LLM response could not be parsed into outlines` | 有文本但解析不出对象 |
| `LLM response blocked by content filter` | `finishReason === 'content-filter'` |

成功返回时（[L795-L801](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L795-L801)）还有最后一道工序：
`uniquifyMediaElementIds`——把模型写的顺序化 `gen_img_N` / `gen_vid_N` 换成 `gen_img_<nanoid(8)>`
（[outline-media.ts#L5-L20](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-media.ts#L5-L20)），
`languageDirective` 缺省填 `DEFAULT_LANGUAGE_DIRECTIVE`。

---

## 7. `outline-schema.ts`：一个归一化器，两处使用

文件头注释（[L1-L14](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L1-L14)）把设计目标说得很清楚：
**同一个归一化器同时作用于"步骤生成的"和"用户编辑后确认的"大纲，
所以生成出来的大纲一定可以被原样确认**。

### 7.1 宽松 vs 严格：只在管线依赖处设限

| 维度 | 规则 |
|---|---|
| 条数 | 1 ~ `MAX_OUTLINE_SCENES`（100），超出整份拒绝（[L208-L213](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L208-L213)） |
| 体积 | JSON 的 UTF-8 字节 ≤ `MAX_OUTLINE_JSON_BYTES`（512 KB，与步骤的读取上限同单位）；**归一化后再量一次**（L234） |
| `id` | 必须非空字符串，且**全体唯一**（L167-L168、L226-L228） |
| `order` | 必须是 0~10000 的整数，且**全体唯一**（L175-L178、L229-L231） |
| `type` | 只要是非空字符串且 ≤ 64 字符——**不校验是否在已知类型内** |
| 未知成员 | 一律不带出（`defined()` + 逐字段白名单） |
| `null` | 视同缺失（`withoutNulls`） |
| 可选成员形状不对 | **丢掉该成员**，而不是拒绝整条（列表里形状不对的项也逐项丢） |
| 幂等 | 头注释承诺"归一化已归一化的结果不变" |

`type` 不校验的理由注释写得很直白：**和浏览器一致**——一个没有任何内容路径支持的类型，
应该在它**自己的内容步**失败（第 5 册会看到这个拒绝），而不是在大纲这里就拦下。

### 7.2 三个子归一化器

| 成员 | 严格之处 | 文件位置 |
|---|---|---|
| `mediaGenerations` | 最多 20 条（`MAX_MEDIA_GENERATIONS`），`type ∈ {image,video}`、`prompt`/`elementId` 必备，`aspectRatio ∈ {16:9,4:3,1:1,9:16}`，否则**该项被丢** | [L86-L105](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L86-L105) |
| `quizConfig` | **全有或全无**：`questionCount` 必须是 1~100 的整数、`difficulty` 合法、`questionTypes` 非空，任一不满足 → 整个 `quizConfig` 变成 `undefined`（内容生成器随后用自己的默认值） | [L112-L134](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L112-L134) |
| `widgetType` | 必须在 `WIDGET_TYPES`（`@openmaic/dsl`）内，否则丢成 `undefined` | [L195](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L195) |

`widgetOutline` 本身**不做字段校验**，只 `withoutNulls` 递归清洗——因为它的形状由各 widget 自己管。

### 7.3 两种"拒绝"的表达

| 返回 | 含义 | 谁看 |
|---|---|---|
| `{ ok: false, message }` | 人类可读的原因（`outlines[3].order must be an integer from 0 to 10000`） | 命令侧 → 400 `INVALID_REQUEST`；engine → 抛 `Error` 让 run 暂停 |
| `OutlineRefused`（内部类） | 单条场景的拒绝，被 `normalizeSceneOutlines` 的 `try/catch` 转成上一条 | 不外泄 |

`engine` 的用法在 [L833-L835](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L833-L835)：
不可用就 `throw new Error('The generated outline is unusable: ' + message)`——
**注意这已经是 step 提交阶段**，所以它同样会走 engine 的 `StepFailedError` → 暂停路径（§10）。

## 8. 审查闭环：这份大纲等谁确认

第 3 步是 7 步里唯一会**停下来等人**的步。停下来等谁、等多久、谁来叫醒，
全部由 `outlineReview` 一个字段决定。

### 8.1 三种模式

| 值 | 谁确认 | 提交后 run 的状态 | 占不占 worker |
|---|---|---|---|
| `wait` | 人必须点确认 | `awaiting_outline_confirmation` | **不占**（`releaseLease: true`） |
| `countdown`（默认） | 2.5 秒内没人按停 → run 自己确认 | `awaiting_outline_confirmation` + 一条死线 | 不占 |
| `auto` | 步骤提交的那一瞬就算确认 | `generating` | 占（继续跑第 4 步） |

取值在 [input.ts#L174-L176](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L174-L176)
（以及 L213 再导出给 `GenerationRunInput`）校验，别的字符串一律 400。
engine 只看 `auto`：[engine.ts#L846](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L846)
`const automatic = input.outlineReview === 'auto'`，`wait` 与 `countdown` 在提交这一步**完全同形**。

### 8.2 `auto` 分支的提交差别

[engine.ts#L850-L857](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L850-L857)：

```ts
patch: {
  state: next,
  step: null,
  outline: confirmed,
  outlineRevision: 1,
  scenesTotal: result.outlines.length,
  ...(automatic ? {} : { releaseLease: true, outlineAutoConfirm: true }),
},
```

`automatic` 为真时**两个开关都不设**：既不释放租约（它还要接着跑），
也不去碰 `outline_auto_confirm_at`（没有要等的东西）。
事件里也多一条 `outline_confirmed { revision: 1, edited: false, automatic: true }`（L861-L868）。

### 8.3 那 2.5 秒的判定在 SQL 里，不在 JS 里

`outlineAutoConfirm: true` **只是个开关**，真正决定"要不要设死线"的是
[store.ts#L473-L479](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L473-L479)：

```sql
outline_auto_confirm_at = CASE WHEN input->>'outlineReview' = 'countdown'
  THEN now() + make_interval(secs => $n::double precision / 1000)
  ELSE NULL END
```

为什么要再看一次 `input`？engine 那段注释（L842-L845）说得很清楚：
**大纲在流式输出的这段时间里，用户可能已经 `hold-outline` 把 run 改成 `wait` 了**。
如果 JS 侧按提交前读到的 `input` 就写死线，一个"已被按住"的 run 会被再次自动确认。
把判定下推到 SQL，读的是同一事务里锁住的行。

`OUTLINE_AUTO_CONFIRM_MS = 2500`
（[types.ts#L55-L59](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L55-L59)），
注释写明这是"1.1.x 预览页给学习者的那段暂停时间"。

反过来，`patch.outlineAutoConfirm` 为假但 `state` 有变时会显式 `outline_auto_confirm_at = NULL`
（L480-L482）——**任何状态迁移都会顺手清掉死线**，不会留下一个"状态已走、死线还挂着"的行。

### 8.4 `confirm-outline`：双重前置 + 一个配额

[store.ts#L1400-L1456](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1400-L1456)，
判定顺序本身就是文档：

| 顺序 | 检查 | 不通过 |
|---|---|---|
| 1 | `run.state === 'awaiting_outline_confirmation' && run.outline` | 409 `RunCommandConflictError('state', ...)`，文案把下划线换成空格（"The run is generating, not waiting for its outline to be confirmed"） |
| 2 | `command.outlineRevision === run.outline_revision` | 409 类别 `outline-revision`（"The outline is at revision 2, not 1"） |
| 3 | 才轮到 `refusal`（`maxActiveRunsPerOwner`） | 429 `ACTIVE_RUN_LIMIT` |

前两条在第三条之前是有意的：状态/版本冲突说明**客户端视图已经过时**，
而配额是暂时的、稍后再试就行——所以只有"确实可确认"的 run 才去消耗活跃名额。
路由侧把类别 `outline-revision` 映射成 409 `RUN_STATE_CONFLICT`
（[confirm-outline/route.ts#L57-L62](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/confirm-outline/route.ts#L57-L62)）。

确认落库的四个改动（L1431-L1447）：

| 有 edits | 无 edits |
|---|---|
| `outline = { ...run.outline, outlines: command.outlines }` | `outline` 原样 |
| `outlineRevision = revision + 1` | `outlineRevision` 不变 |
| `edited: true` | `edited: false` |
| `scenesTotal = 编辑后的条数` | 同 |

**删场景是合法的**：`scenesTotal` 按编辑后的条数重算，
而后续步骤序列是 `generationSteps(sceneCount)` 现算的——第 3 步之后，"总共有几步"仍然会变。

`order` 由服务端重编：[input.ts#L253-L262](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L253-L262)
在归一化之前把每个元素改写成 `order: index + 1`，客户端传什么 `order` 都不算数
（注释：跟浏览器里的编辑器一样，每次编辑后按位置重新编号）。
`parseConfirmOutline` 同时**复用 §7 的归一化器**（`parseOutlines` = `normalizeSceneOutlines`），
这就是"生成出来的必可原样确认"能成立的机制：两侧走同一个函数。

### 8.5 `hold-outline`：三种模式三种反应

[store.ts#L1466-L1499](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1466-L1499)：

| 当前模式 | 反应 |
|---|---|
| `auto` | **409**：`The run confirms its own outline`（自动确认的 run 按不了暂停） |
| `wait` | **无操作**：直接返回当前 `state` / `seq`，不发事件（本来就在等人） |
| `countdown` | `jsonb_set(input, '{outlineReview}', '"wait"')` + `outline_auto_confirm_at = NULL`，发 `outline_review { outlineReview: 'wait', autoConfirmAt: null }` |

它改的是 **run 的输入**，不是某个临时状态——所以 hold 之后刷新页面、换浏览器，这个 run 仍然是 `wait`。

hold 的**有效窗口比 confirm 宽**（L1472-L1474）：

```ts
run.state === 'awaiting_outline_confirmation' ||
  (!run.outline && ['preparing', 'outlining', 'paused'].includes(run.state))
```

即"还没有大纲、但正在流式输出"时就能按下去。第 1 步、第 2 步期间也允许 hold
（那是给"我知道待会儿会倒计时，我先按住"留的口子）。反过来，大纲已确认后 hold → 409
`its outline was already confirmed`。

### 8.6 谁来结算到期的倒计时

[store.ts#L1508-L1538](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1508-L1538)
`confirmDueGenerationRunOutlines(limit = 10)`：

| 设计点 | 做法 |
|---|---|
| 谁执行 | **任意进程的 runner**（注释原话：*a deadline outlives the process that set it*）——设死线的 worker 挂了也照样有人结算 |
| 并发安全 | `FOR UPDATE SKIP LOCKED`，多实例互不重复结算 |
| 挑选顺序 | `ORDER BY outline_auto_confirm_at LIMIT 10`，每轮最多 10 个 |
| 结算动作 | 与 `auto` 的提交**同形**：`{ state: 'generating', step: null, releaseLease: true }` + `resetTakeovers` |
| 发的事件 | `outline_confirmed { revision, edited: false, automatic: true }` + `state` |
| 配额 | **不受 `maxActiveRunsPerOwner` 约束**（注释：它在此之前一直算在跑的） |
| 通知 | 每个 run 结算后 `notifyOwner(tx, owner_id)`，唤醒 owner 的 SSE |

所以一个 `countdown` run 的确认事件可能来自三个不同的地方：用户的命令、步骤提交（`auto`）、
死线结算——三者发的 `outline_confirmed` 载荷形状相同，只有 `edited` / `automatic` 不同。

### 8.7 幂等：命令重放不会确认两次

[store.ts#L1337-L1373+](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1337-L1373)
的 `runCommand`：

- 同一 `commandId`（+同一 type）**只 apply 一次**，重复请求直接返回首次记录的结果；
- **只有真正 apply 的命令才被记录**——被 409/429 拒绝的命令不写幂等表，
  等 run 状态允许后可以用同一个 `commandId` 再发；
- `before`（配额检查）**先于**行锁执行，且"仅对新命令生效"（L1355-L1363）——
  重放一个已成功的确认不会因为此刻活跃数已满而被拒。

客户端 key 的构造决定了一次"意图"的边界
（[commands.ts#L31-L51](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/commands.ts#L31-L51)）：

```ts
const key = `${view.runId}:confirm:${revision}${editedOutlines ? `:${JSON.stringify(editedOutlines)}` : ''}`;
```

| 事实 | 后果 |
|---|---|
| key 含 `revision` | 大纲一变（别人改了 / 自己改了又改回来），确认意图就是新命令 |
| key 含 edits 的 JSON | 同一份编辑重复点"确认"是安全的重放；改一个字就是新命令 |
| `holdOutline` 的 key 只有 `runId:hold` | 每个 run 只有一个"按住"意图，重复点不会堆积命令 |

### 8.8 多标签页与刷新

| 场景 | 行为 |
|---|---|
| 另一个标签页点了确认 | 本标签页收到 `outline_confirmed`；若 `edited === true`，`follower` 触发 `resync()` 重新拉快照（[follower.ts#L231](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts#L231)）——本地那份流式大纲已经不能原样确认了 |
| 倒计时期间展开某条大纲 | 预览页先 `hold` 再展开（[page.tsx#L220-L241](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L220-L241)），避免"我正在看，它自己走了" |
| 刷新页面 | `reviewIntent` 存在 ref 与 localStorage（L70/L92/L122/L159/L222/L247），"我要自己看大纲"的意图跨刷新保留 |
| 确认撞 409 `outline-revision` | `setConfirmConflict(true)` 并把 phase 退回 `review`（L116-L145），让用户重看一遍再确认 |
| 流式期间编辑大纲 | 编辑器**只读**（L252-L256）——此时 `outline_ready` 还没到，编辑没有 revision 可锚 |

---

## 9. 事件、两份存储与前端三态

### 9.1 第 3 步会看到的全部事件

| 事件 | 载荷 | 谁发 | 频次 |
|---|---|---|---|
| `outline_language_directive` | `{ data }` | engine `emit('languageDirective')` | 0 或 1 次（首个正则命中） |
| `outline_course_title` | `{ data }` | engine `emit('courseTitle')` | 0 或 1 次 |
| `outline_item` | `{ index, outline }` | engine `emit('outline')` | **每条一次**，可被 reset 作废 |
| `outline_reset` + `step_retry` | `{}` / `{ step, attempt, maxAttempts, fallback? }` | engine `emit('retry')` | 每次重试**成对**出现 |
| `step_completed` | `{ step }` | 步骤提交 | 1 次 |
| `outline_ready` | `{ revision: 1, outline }` | 步骤提交 | 1 次（**全量**，与流式副本对账） |
| `outline_confirmed` | `{ revision, edited, automatic? }` | 提交 / 命令 / 死线结算 | ≥1 次 |
| `outline_review` | `{ outlineReview, autoConfirmAt }` | `hold-outline` | 0 或 1 次 |
| `state` | `{ state, step }` | 提交与各命令 | 若干 |

事件名与载荷的权威注释在
[types.ts#L285-L302](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L285-L302)。
`step_retry` 是通用事件（第 1/2/5/6/7 步的重试也用它），只有 `outline_reset` 是本步专属。

### 9.2 大纲在库里存了两份，用途完全不同

| 位置 | 内容 | 读它的代码 |
|---|---|---|
| `generation_run_steps` 的 `outline` 行（检查点） | 步骤的原始输出 `OutlineResult`（`outlines` + `languageDirective` + `courseTitle` + `taskEngineMode`） | `output<T>(stepId)`（engine L411）——**只用于 resume** |
| `generation_runs.outline` 列 | 上表整理成的 `GenerationRunOutline`（engine L836-L841） | `outline()`（engine L455-L458）——**下游一律读这份** |

差别在"用户可能改过"：第 4 步 `resolveAgents` 解构的是 `outline()`，也就是
**确认后的那一份**（[engine.ts#L874-L876](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L874-L876)）。
删掉的场景不会再生成，改过的标题会用于 Agent 命名。
`run.outline` 为空时 `outline()` 直接抛 `Run ... has no confirmed outline`——
这是个**不可能被触发的防御**：只有大纲确认后的步骤才会跑到这里。

### 9.3 `scenesTotal` 是第一次被填上

前两步结束时 `scenes_total` 是 `NULL`（run 还不知道会有几幕）。
第 3 步提交时写 `scenesTotal: result.outlines.length`（engine L855），
确认带编辑时重写（store L1443）。**进度百分比从这一刻才有意义**，
`{scenesCompleted}/{scenesTotal}` 在此之前只能显示"正在准备"。

### 9.4 reducer 与 UI 三态

| 视图字段 | 更新规则 |
|---|---|
| `streamingOutlines` | `outline_item` 按 `index` **填空位**（不是 push），乱序到达也能落对位置（[reducer.ts#L140-L147](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L140-L147)） |
| `outlineRetrying` | `outline_reset` 置真（L136-L139），`outline_ready` 置假并**整体替换**（L148-L157） |
| `outline` | `outline_ready` 写入整份 `{ ...outline, revision }`，并把 `progress.scenesTotal` 与 `streamingOutlines` 一起对齐（L148-L157）；`outline_confirmed` **只改 `revision` 并把 `outlineAutoConfirmAt` 清成 null**（L158-L164）——注释说明"编辑过的大纲条目在快照里，不在事件里" |
| `input.outlineReview` / `outlineAutoConfirmAt` | 由 `outline_review` 改（L165-L174），非字符串的 `autoConfirmAt` 一律当 null |
| `visibleOutlines` | `view.outline?.outlines ?? view.streamingOutlines`（L220-L223）——**`outline_ready` 一到就切换到权威副本** |

预览页的 phase 由一个纯函数决定
（[outline-review.ts#L32-L46](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/outline-review.ts#L32-L46)）：

| 条件 | phase |
|---|---|
| 大纲还在流 / 正在重试 | `progress` |
| 有大纲但未确认 | `outline-ready` |
| 已确认，或用户有 review 意图 | `review` |

而顶部步骤条把 `outline` 与 `awaiting_outline_confirmation` **都映射到 UI 的第 2 格**
（[preview-steps.ts#L35-L67](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L35-L67)）：
"等人确认"不算新的一步，它仍然是"生成大纲"这一步的一部分。

---

## 10. 失败链：这一步崩了会怎样

### 10.1 四层

| 层 | 位置 | 失败表现 |
|---|---|---|
| L1 准备 | `prepareOutline` | 缺 `course.outline` 槽 → `stageModel` 抛 `ModelConfigurationError`；`outline` 输入不合法 → `StepRefusal` |
| L2 模型/流 | `streamOutlines` | 3 次同模型 + 1 轮回退模型仍无大纲 → `OutlineGenerationError`；`content-filter` 直接结束循环 |
| L3 归一化 | engine L833-L835 | `normalizeSceneOutlines` 不可用 → `Error('The generated outline is unusable: ...')` |
| L4 事件写入 | engine L777-L785、L830 | `append()` 失败记进 `appendFailure`，`finally await appending` 后 **throw** |

**L4 是最容易被忽略的一条**：事件写不进去和模型答不上来是**同一路**——
都会让 run 暂停。因为大纲的价值一半在"逐条流式可见"，静默丢掉这些事件会让 UI 永远停在半截。

### 10.2 错误 → 失败码

[run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts)
对本步的归类：

| 抛出的东西 | `code` |
|---|---|
| `ModelAuthorizationError`（x-model 未授权） | `MODEL_UNAUTHORIZED` |
| `ModelConfigurationError` / `WebSearchConfigError` | `MODEL_CONFIG_MISSING` |
| `StepRefusal` | `GENERATION_FAILED` |
| `OutlineGenerationError`（包着 429） | `RATE_LIMITED` |
| `OutlineGenerationError`（包着别的上游错） | `UPSTREAM_ERROR` |
| `isUpstreamQuotaExhausted(cause)` | `PROVIDER_QUOTA_EXHAUSTED` |
| `StepAbortedError` | 不产生失败码（是取消，不是失败） |

注意 `OutlineGenerationError` 自己不在映射表里——它是**文本 + `cause`**，
码由 `cause` 决定（所以 `lastCause` 必须带上，L787-L793）。
纯"空输出/解析失败/内容过滤"这三种没有 `cause` 的情况落到兜底 `INTERNAL_ERROR`。

### 10.3 暂停与 Retry

| 事实 | 依据 |
|---|---|
| 失败后 run 停在 `outlining` | `phaseOfStep({kind:'outline'}) === 'outlining'`（[plan.ts#L68-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L68-L72)），engine 末尾 `StepFailedError → pause`（L1860-L1868） |
| Retry 回到 `outlining` | `stateForRetry('outline')` = `phaseOfStep` = `outlining`（[plan.ts#L106-L111](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L106-L111)） |
| 重启会**补发** `outline_reset` | 主循环里 `restarted && step.kind === 'outline'`（[engine.ts#L1820-L1822](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1820-L1822)）——让 UI 丢掉上次留下的半截流 |
| Retry 受活跃上限约束 | 确认与 Retry 都让 run 重新计入 `LIMITED_RUN_STATES`（[types.ts#L47-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L47-L53)） |
| Retry 后 `outlineRevision` 重新从 1 起 | 提交 patch 里写死 `outlineRevision: 1`（engine L854） |

---

## 11. 常量速查

| 常量 | 值 | 位置 | 作用 |
|---|---|---|---|
| `STEP_DEADLINES_MS.outline` | 300 000 ms | [deadline.ts#L16-L17](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L16-L17) | 整个步骤（含自管重试）的预算 |
| `MAX_STREAM_RETRIES` | 2 → 最多 3 次同模型 | [outline.ts#L57](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L57) | 同模型重试轮数 |
| `MAX_OUTLINE_STREAM_BYTES` | 512 KB | [outline.ts#L61](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L61) | 流字节熔断（warn + break） |
| `MAX_OUTLINE_SCENES` | 100 | [outline-schema.ts#L20](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L20) | 既是条数熔断也是归一化上限 |
| `MAX_OUTLINE_JSON_BYTES` | 512 KB | [outline-schema.ts#L25](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L25) | 归一化前后各量一次 |
| `MAX_TYPE_CHARS` / `MAX_ORDER` | 64 / 10 000 | [outline-schema.ts#L27-L28](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L27-L28) | 单字段上限 |
| `MAX_QUESTION_COUNT` / `MAX_MEDIA_GENERATIONS` | 100 / 20 | [outline-schema.ts#L29-L30](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts#L29-L30) | `quizConfig` / `mediaGenerations` |
| `OUTLINE_AUTO_CONFIRM_MS` | 2 500 ms | [types.ts#L59](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L59) | 倒计时长 |
| `confirmDueGenerationRunOutlines` 的 limit | 10 | [store.ts#L1508](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1508) | 每轮结算个数 |
| `MAX_COMMAND_BODY_BYTES` | 1 MB | [input.ts#L19](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L19) | 确认带编辑体的请求上限 |
| `MAX_VISION_IMAGES` | 20 | [generation.ts#L10](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L10) | 视觉切片张数 |
| `MAX_PDF_CONTENT_CHARS` | 50 000 | [generation.ts#L7](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L7) | 文本切片 |
| directive/title 的正则窗口 | 8 192 字符 | [outline.ts#L72/L91](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L68-L81) | 只在 buffer 头部找 |
| 标题长度 | 120 字符 | [outline.ts#L95-L104](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts#L95-L104) | trim + 截断 |
| 媒体 id 后缀 | `nanoid(8)` | [outline-media.ts#L5-L20](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-media.ts#L5-L20) | `gen_img_` / `gen_vid_` |
| 模型槽 | `course.outline` | [model-slots.ts#L96](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L96) | stage `scene-outlines-stream` |

## 12. 易踩坑清单

| # | 坑 | 为什么会踩 | 怎么避开 |
|---|---|---|---|
| 1 | 以为 `wait` / `countdown` 有两条不同的提交代码 | engine 只看 `auto`，另两种同形 | 差别在 `applyPatch` 的 SQL `CASE`，不在 TS 分支 |
| 2 | 在 `outlineReview` 上加第四种取值 | `input.ts` 白名单三值，别处不会报错但 run 会 400 | 只加 prompt 侧的开关，别动这个字段 |
| 3 | 改动 `normalizeSceneOutlines` 只测生成侧 | 同一个函数也是确认命令的校验器 | 生成侧 + `parseConfirmOutline` 两侧都要过 |
| 4 | 给 `type` 加"必须在已知集合内"的校验 | 归一化器**故意**不校验（注释明写与浏览器一致） | 未知类型应在它自己的**内容步**失败（第 5 册） |
| 5 | 以为大纲步套了 `withRouteRetry` | 它显式传 `{ enabled: false }` | 只有第 5/6/7 步用外层重试；本步自管 |
| 6 | 在 `maybeFallback` 里去掉 `serverManaged` 闸门 | 客户端 x-model 会消耗运维的回退 key | 这条是**防滥用**的，注释里写了 |
| 7 | 把 `content-filter` 当成"空输出"走回退 | 安全拒绝≠模型没答上，换模型再问一遍没用 | 保持"既不重试也不回退"的现状 |
| 8 | 只读 `textStream` 做解析 | 401/无效 key/内容过滤都以流分片出现，不抛异常 | 必须消费 `fullStream` 的 `error` / `finish` |
| 9 | 熔断改成抛错 | 两个上限现在是 warn + break（保留已有结果） | 改抛错会让"跑飞的长输出"从部分可用变成全失败 |
| 10 | 在 `emit` 里做同步等待 | engine 用**串行队列**保 seq 单调；`appendFailure ??=` 只留首错 | 任何新增事件都走 `append()`，别 `await` |
| 11 | 忘了 `finally { await appending }` 后再判 `appendFailure` | 事件写入失败必须与步骤失败同路 | 移动这段顺序会让 run 在"事件没写进去"时继续跑 |
| 12 | 在 `outline` 步的 prompt 里塞整篇材料 | 视觉/文本各有切片上限，且有 N3 对齐问题 | 走 `buildOutlinePrompt` 的既有切片路径 |
| 13 | 用 `output<OutlineOutput>('outline')` 读下游要的大纲 | 检查点是**原始**输出，用户编辑看不见 | 下游一律 `outline()`（`run.outline`） |
| 14 | 客户端自己算 `order` | 服务端按位置重编（`index + 1`），传了也不算 | 只传列表顺序 |
| 15 | 以为 hold 能按住 `auto` run | `auto` 直接 409 | UI 上就不该出现暂停按钮 |
| 16 | 只在设置死线的那个 worker 上等结算 | 死线由**任意进程**的 runner 结算 | 不要在 engine 里加定时器 |
| 17 | 每次点确认都生成新 `commandId` | 重放会产生新命令（`outlineRevision` 一变就该变，不该不变） | 沿用 `commandIdFor(key)` |
| 18 | 把 `outline_reset` 当成"步骤重置" | 它只作废 `streamingOutlines` | 权威副本来自 `outline_ready` |
| 19 | 在流式期间允许编辑大纲 | 没有 revision 可锚，编辑器**故意**只读 | 保持 `handleOutlinesChange` 的门 |
| 20 | 以为任务引擎关闭会报错 | `resolveVocationalActive` 关时**静默回落**到交互模板 | 查日志的 prompt id，不要查异常 |

---

## 13. 代码地图

### 13.1 后端

| 文件 | 行数 | 本册关注的职责 |
|---|---|---|
| [steps/outline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/outline.ts) | 810 | prompt 准备、增量解析、流式循环、重试与回退 |
| [outline-schema.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/outline-schema.ts) | 236 | 归一化器与全部上限 |
| [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) | 1877 | `runOutlineStep`（L772-L872）、`case 'outline'`（L660）、主循环与补发 reset（L1820） |
| [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) | 677 | `outline` 服务注入点（L444-L453） |
| [run/store.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts) | — | `applyPatch`（L458-L509）、`confirm`/`hold`/`due`（L1400-L1538）、`runCommand`（L1343+） |
| [run/plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) | 166 | `phaseOfStep`、`stateForRetry`、`generationSteps` |
| [run/types.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts) | — | 状态集合、`OUTLINE_AUTO_CONFIRM_MS`、事件名注释 |
| [run/input.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts) | — | `outlineReview` 校验、`parseConfirmOutline` |
| [run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts) | 65 | 错误 → 码 |
| [run/deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | — | `STEP_DEADLINES_MS.outline` |
| [app/api/generation-runs/[id]/confirm-outline/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/confirm-outline/route.ts) | 66 | 确认入口 + 授权前置 + 429/409 映射 |
| [app/api/generation-runs/[id]/hold-outline/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/hold-outline/route.ts) | — | 暂停入口 |
| [generation-run-hooks/runtime.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-run-hooks/runtime.ts) | — | 两个失败谓词、`stageModel`、回退解析 |
| [@openmaic/generation/src/outline-media.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-media.ts) | — | `uniquifyMediaElementIds` |

### 13.2 前端

| 文件 | 职责 |
|---|---|
| [reducer.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts) | `outline_*` → `RunView` |
| [outline-review.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/outline-review.ts) | `nextPreviewPhase` 三态 |
| [preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) | 步骤条映射（`outline` 占第 2 格） |
| [commands.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/commands.ts) | `confirmOutline` / `holdOutline` 与幂等 key |
| [follower.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts) | `edited` 确认 → `resync()` |
| [app/generation-preview/page.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx) | 倒计时、review 意图、只读门、冲突回退 |

### 13.3 Prompt

| 文件 | 何时生效 |
|---|---|
| [@openmaic/generation 的 outline prompt](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src) | 标准分支（与 app 路径字节等价） |
| [lib/prompts 的 `interactive-outlines`](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/index.ts#L27) | 交互分支 + `userProfile` |
| [lib/prompts 的 `task-engine-outlines`](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/index.ts#L28) | 任务引擎分支（优先级最高） |

---

## 14. 动手验证

| 想验证的事 | 跑哪个测试 | 关键用例 |
|---|---|---|
| 事件顺序与逐条流式 | [tests/server/generation-steps/outline.test.ts#L83](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/outline.test.ts#L83) | `reports the directive, the title and each outline as the model writes them` |
| 空响应重试与最终失败 | 同上 [#L106](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/outline.test.ts#L106) | `retries an empty answer, reporting each retry, and then fails with the last error` |
| 重试后恢复 | 同上 [#L119](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/outline.test.ts#L119) | `recovers on a retry after an empty answer` |
| abort 不烧重试 | 同上 [#L127](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/outline.test.ts#L127) | `stops without retrying once the caller goes away` |
| 媒体门 | 同上 [#L137](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/outline.test.ts#L137) | `keeps media out of the prompt when the caller opts out, whatever the slots offer` |
| 401 分片不被当空输出 | [tests/server/outline-fallback.test.ts#L92](file:///D:/GITHUB_chat/OpenMAIC/tests/server/outline-fallback.test.ts#L92) | `surfaces a pre-stream 401 error part instead of falling back` |
| 回退只发生一次 / 需要 serverManaged | 同上 [#L113](file:///D:/GITHUB_chat/OpenMAIC/tests/server/outline-fallback.test.ts#L113)、[#L146](file:///D:/GITHUB_chat/OpenMAIC/tests/server/outline-fallback.test.ts#L146) | 两条闸门 |
| `content-filter` 不重试不回退 | 同上 [#L163](file:///D:/GITHUB_chat/OpenMAIC/tests/server/outline-fallback.test.ts#L163) | 单列分支 |
| 配额错误不被早先的 429 顶替 | [tests/server/outline-quota-failure.test.ts#L73](file:///D:/GITHUB_chat/OpenMAIC/tests/server/outline-quota-failure.test.ts#L73) | `lastError` 语义 |
| 宿主不可重试 / 保留已流出的大纲 | [tests/server/generation-run-hooks/outline-host-failure.test.ts#L110-L116](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-run-hooks/outline-host-failure.test.ts#L110-L116) | 三条判定 |
| 任务引擎三模板与静默回落 | [tests/generation/task-engine-outline.test.ts#L61/L262/L316](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/task-engine-outline.test.ts#L61) | 含 `procedural-skill` 降级、id 去重（L441） |
| N3 视觉切片与文本脱钩 | [tests/generation/outline-standard-desync.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/outline-standard-desync.test.ts) | 丢掉一张图时的 prompt 对齐 |
| 媒体 id 去重 | [tests/generation/outline-media-uniquify.test.ts#L21-L63](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/outline-media-uniquify.test.ts#L21) | 四条含"畸形数组直接丢" |
| 媒体能力门 | [tests/generation/outline-media-from-slots.test.ts#L79-L92](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/outline-media-from-slots.test.ts#L79) | 槽不给就不开 |
| 审查模式（客户端侧） | [tests/generation-run-client/outline-review.test.ts#L20/L47](file:///D:/GITHUB_chat/OpenMAIC/tests/generation-run-client/outline-review.test.ts#L20) | `wait` 与 `countdown` 两组 |
| 审查模式（预览页） | [tests/generation/preview-outline-review.test.ts#L153/L230](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/preview-outline-review.test.ts#L153) | 倒计时自动确认 / 等待确认 |
| 事件序列落库 | [tests/server/generation-runs.pg.test.ts#L400-L420](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L400-L420) | `outline_*` 全序 |
| 确认与 hold 的命令语义 | [tests/server/generation-runs.pg.test.ts#L613-L745](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L613-L745) | `outline_review` / `outline_confirmed` |
| reducer 行为 | [tests/generation-run-client/reducer.test.ts#L15-L52](file:///D:/GITHUB_chat/OpenMAIC/tests/generation-run-client/reducer.test.ts#L15-L52) | 按 index 填空位 |
| 多端一致 | [tests/generation-run-client/follower.test.ts#L105-L154](file:///D:/GITHUB_chat/OpenMAIC/tests/generation-run-client/follower.test.ts#L105-L154) | `edited` 触发 resync |

一条最小复现路径（本地起服务后）：创建一个**不传** `outlineReview` 的 run，
观察事件流里应依次出现 `outline_item ×N` → `step_completed` → `outline_ready` →
（2.5 秒后）`outline_confirmed { automatic: true }`；
在 `outline_ready` 之前发一次 `hold-outline`，则最后一条不会自动出现。

---

## 15. 一句话总结

**第 3 步 = 一次自管重试的流式 JSON 数组 + 一个归一化器 + 一份可以被人改掉的计划。**

它是 7 步里唯一改写 run 状态机的步：独占 `outlining` 相位、把 `step` 置空、
把 `scenesTotal` 第一次填上，并且除了 `auto` 之外一律释放租约——**等人的 run 不占 worker**。
它也是唯一"决定后面还有多少步"的步，以及唯一**自己管重试**（外层 `withRouteRetry` 被显式关掉）的步。

要改这一层的行为，记住三个不变量：
生成的形状必须能被同一个归一化器原样确认；
逐条事件必须与步骤成败同路（`appendFailure` 也要让 run 暂停）；
`countdown` 的真假判定在 SQL 的 `CASE` 里，不在 engine 的分支里。


