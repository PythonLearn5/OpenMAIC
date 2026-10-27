# 课程生成管线 · 第 5 册：场景内容（scene:N:content）

<!--
  职责：详解生成管线**第 5 步「场景内容」**——每个场景三连步的第一步。它把第 3
  步定稿的一条 `SceneOutline`（slide / quiz / interactive / pbl）变成真正可渲染
  的 `SceneContent`：slide 是一组绝对定位元素 + 背景，quiz 是一道道题，interactive
  是一整页 HTML + widgetConfig，pbl 是一个项目定义。它同时返回一条
  `effectiveOutline`（可能被 fallback 改过类型、可能被剥掉无法解析的图片）。
  本步还负责三件事：把大纲的 suggestedImageIds 变成"看得见的图"（视觉附件）或
  "说得出话的图"（文本描述）；把媒体占位 id（gen_img_N / gen_vid_N）原样留给
  第 6/7 步之后的媒体泳道；在 `PARALLEL_SCENE_CONCURRENCY > 1` 时以 prewarm
  泳道并行预生成后续场景，并对失败的场景做"标记跳过、继续其余、最后回头暂停"
  的软失败处理。
  覆盖：`case 'content'` 三条分支、`generateContent` 闭包的输入、三层重试链
  （retryOptions → withRouteRetry → isRetryableGenerationError）、300 s 单尝试
  预算、prewarm 并行与 lazyBoundedMap、跳过语义、services 的类型化 stage 与
  素材校验、步骤文件的六段、视觉图片 15 s 预算 + 连败 3 保险丝、package 层四种
  内容形态的生成与后处理、事件与前端、失败链、常量、坑与验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 4 册 Agent 画像](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-4.md)｜下一步：[第 6 册 场景动作](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md)
- 源文件：[steps/scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts)
  · [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)
  · [run/retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts)
  · [packages/@openmaic/generation/src/scene-generator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `content` / `scene:<index>:content`（[plan.ts#L31-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L31-L33) 的 `sceneStepId`） |
| 是否必经 | **必经**，且是每个场景三连步的**第 1 项**：`SCENE_STEP_KINDS = ['content', 'actions', 'narration']`（[plan.ts#L29](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L29)） |
| 一共跑几次 | 场景数 N 次（`generationSteps()` 为每个 index 展开三连步，[plan.ts#L57-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L57-L65)）；并行模式下可能"提前"跑完若干次 |
| 所处 run 状态 | `generating`（[plan.ts#L67-L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L71)） |
| 检查点写在哪 | `generation_run_steps(run_id, 'scene:<n>:content')`，值是 `SceneContentResult = { content, effectiveOutline }`（[scene-content.ts#L70-L74](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L70-L74)） |
| 时间预算 | **300 s，且按"每次尝试"计时**（[deadline.ts#L19](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L19) 的 `sceneContent`；`withDeadline` 在 `withRouteRetry` 的内侧） |
| 重试 | **三层**：run 层指数退避（[retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts)）→ provider 状态码可重试判定（[generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L158)）→ **AI SDK 内部重试被显式关掉**（`maxRetries: 0`，[scene-content.ts#L105](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L105)） |
| 重试上限 | 第 1 个场景 **2 次**、其余场景 **5 次**（[retry.ts#L22-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L22-L24)，`retryOptions` 在 [engine.ts#L430](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L430) 选值） |
| 失败后果 | 串行：直接 `paused`。并行：本场景标记跳过、其余继续，全部跑完后在**最小序号的失败场景**处 `paused`（[engine.ts#L1788-L1796](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1788-L1796)） |
| UI 显示 | `slide-content` 位——**只有 `scene:N:content` 点亮它**，actions 与 narration 都归到 `actions` 位（[preview-steps.ts#L40-L46](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L40-L46)） |
| 产出消费方 | 第 6 步动作直接读 `output<SceneContentResult>('scene:<n>:content').content`（[engine.ts#L715](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L715)）；第 7 步与入库消费的是动作 + 旁白，内容元素是它们的底 |
| 独有之处 | **唯一被"提前生成"（prewarm）的步**；也是唯一会**只跳过单个场景而不暂停整条 run** 的步；进度条**不**由本步推进（见 §7.3） |

### 0.2 一句话本质

> 第 5 步是**"把一条大纲变成一屏东西"**：它按 `outline.type` 分派到四种生成器
> （slide / quiz / interactive / pbl），每种都只调用一次大模型（AI SDK 内部重试
> 关闭），拿到 JSON 后做一长串**确定性的修复与裁剪**（默认值、LaTeX 渲染、图片 id
> 解析、旋转白名单、背景兜底），最终产出 `SceneContent` 与一条可能被改过的
> `effectiveOutline`。它同时是整条管线里**唯一允许"局部失败"**的步——并行模式下
> 单个场景生成失败不会打断课程，只会让那个场景缺席，其余照常入库。

### 0.3 时序图（两种形态）

**串行（默认，`PARALLEL_SCENE_CONCURRENCY` 未设或 ≤ 1）**

```
advanceRun ─▶ step = scene:0:content
                │ commit { state:'generating', step, step_started }
                ▼
      runStep(case 'content')  ── ahead.prewarm 为空 ──▶ done(generateContent(0, signal))
                │
                │  withRouteRetry( 300s 预算, maxRetries=2 )
                │    └─ withDeadline ─▶ services.sceneContent ─▶ 步骤文件 ─▶ package ─▶ callLLM(maxRetries:0)
                ▼
      commit { step:{id:'scene:0:content', output:{content, effectiveOutline}},
               events:[ step_completed ] }
                ▼
      advanceRun ─▶ scene:0:actions ─▶ scene:0:narration ─▶ scene:1:content …
```

**并行（`PARALLEL_SCENE_CONCURRENCY = 3`）**

```
主循环走到 scene:1:content ─▶ startContentPrewarm(1)
        │  pending = [1,2,3,…] 中未 checkpoint 的
        │  concurrency<=1 或 pending<=1 ⇒ 直接 return（保持串行）
        ▼
   lazyBoundedMap(pending, 3, fn, { shouldContinue: !prewarmSignal.aborted })
        ├─▶ fn(1): generateContent(1, prewarmSignal) ─成功─▶ commit{step, step_completed}
        ├─▶ fn(2): …                        ─失败─▶ resolve {failed}（不抛）
        └─▶ fn(3): …
   ahead.prewarm = Map{ 1→Promise, 2→Promise, 3→Promise }

主循环照常一轮轮走：
  scene:1:content → case 'content' 发现 pending → await → 成功 ⇒ return {}（不重复记账）
  scene:1:actions → 用 content 输出
  scene:1:narration → 进度 +1
  scene:2:content → await 得到 {failed} → 四道筛子都没命中
                    ⇒ skippedScenes.set(2) + commit{ step_failed, continuing:true } + return null
  scene:2:actions/narration → 因为 content 未 checkpoint，advanceRun 会回头重跑？
                    不会：skippedStepIds() 把 2 的三个步都算作"已完成"，主循环跳过
  … 直到 advanceRun.kind === 'complete'
        ▼
   skippedScenes 非空 ⇒ 取最小 index=2 ⇒ pause('scene:2:content', msg, code) ⇒ 'paused'
   （文档里已经有 scene 1、3；Retry 从 2 续跑补齐）
```

---

## 1. 步骤坐标：`scene:<n>:content` 在计划中的位置

### 1.1 三连步的展开

[plan.ts#L29-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L29-L33) 定义了场景步的命名规则：

```ts
const SCENE_STEP_KINDS = ['content', 'actions', 'narration'] as const;

export const sceneStepId = (sceneIndex: number, kind: SceneStepKind) =>
  `scene:${sceneIndex}:${kind}`;
```

`generationSteps(sceneCount)`（[#L57-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L57-L65)）返回 `['agents', …每场景三连步]`。所以一条 4 场景课程的完整步骤序列是：

```
agents
scene:0:content  scene:0:actions  scene:0:narration
scene:1:content  scene:1:actions  scene:1:narration
scene:2:content  …
```

`advanceRun`（[#L73+](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L73-L110)）做的事只有一句：**按顺序找第一个不在 `completed` 集合里的步**。`completed` 由主循环传入，等于"已 checkpoint 的步 id ∪ `skippedStepIds()`"（[engine.ts#L1767-L1772](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1767-L1772)）。

这条规则解释了 prewarm 为什么能"白拿"：**prewarm 在自己的 Promise 里就把 checkpoint 提交了**，主循环走到那一步时 `advanceRun` 甚至不会再选出它——不对，还是会选（因为 prewarm 提交发生在主循环推进之后），所以 `case 'content'` 仍会被走到，只是它发现 `ahead.prewarm` 里有这个 index，于是**只 await、不重算、不再记账**。

### 1.2 从 `paused` 恢复时落在哪

`stateForRetry`（[plan.ts#L112-L166](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L112-L166)）对 `scene:*:content` 的处理与其它场景步相同：把 run 状态改回 `generating`，`step` 保留为失败的那个步 id，于是 Retry 从**那一场景的 content** 重新起步。并行模式的端到端例子见 [generation-runs.pg.test.ts#L1451-L1491](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L1451-L1491)：`o2` 的 content 抛 `{statusCode:400}` → run `paused`、`run.step === 'scene:1:content'`、error 里 `errorCode:'UPSTREAM_ERROR'`；Retry 后三场景齐全。

### 1.3 输出对象

```ts
// lib/server/generation/steps/scene-content.ts#L70-L74
export interface SceneContentResult {
  content: GeneratedSceneContent;
  /** 生成实际使用的大纲：类型可能被 fallback 改写，图片 id 可能被剥离。 */
  effectiveOutline: SceneOutline;
}
```

`effectiveOutline` 是本步**特有的第二个返回值**——第 6 步用的正是它而不是原始 outline（[engine.ts#L715-L727](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L715-L727) 附近），因为大纲的 `type` 可能在生成时被降级（见 §9.2），存下来才能保持文档一致。

---

## 2. 引擎入口：`case 'content'` 的三条分支

[engine.ts#L678-L710](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L678-L710) 整段只有 33 行，但它是本册最重要的 33 行：

```ts
case 'content': {
  const pending = ahead.prewarm?.get(step.sceneIndex);
  if (pending) {
    const result = await pending;
    if ('failed' in result) {
      if (isAbortError(result.failed) || signal.aborted) throw result.failed;
      if (isGenerationRunLeaseLostError(result.failed)) throw result.failed;
      // A host failure no retry helps (the other scenes would meet it too)
      // pauses the run at this scene now.
      if (isNonRetryableHostFailure(result.failed)) throw result.failed;
      // Mark the scene and go on with the others.
      const message = result.failed instanceof Error
        ? result.failed.message || 'The step failed'
        : String(result.failed);
      log.warn(`run ${run.id}: ${stepId} failed; continuing with the other scenes`);
      const code = failureCodeOf(result.failed);
      skippedScenes.set(step.sceneIndex, { message, ...code });
      await commit({
        events: [{ type: 'step_failed',
          data: { step: stepId, message, ...code, continuing: true } }],
      });
      return null;
    }
    // Committed when it completed; nothing more to record.
    return {};
  }
  return done(await generateContent(step.sceneIndex, signal));
}
```

三条分支：

| 分支 | 条件 | 行为 |
|---|---|---|
| A. 命中 prewarm 且成功 | `ahead.prewarm.get(i)` 存在且结果不含 `failed` | `return {}`——**空 commit**，因为 prewarm 的 fn 在成功那一刻已经自己提交过 `step` + `step_completed`（[#L555-L560](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L555-L560)） |
| B. 命中 prewarm 且失败 | 结果 `{failed}` | 四道筛子：① abort / run 被中止 ⇒ 原样抛；② lease 丢失 ⇒ 原样抛；③ **宿主级不可重试失败**（模型没配、鉴权挂了）⇒ 原样抛，注释说得很直白"其他场景也会遇到，重试没用"；④ 都不是 ⇒ **跳过该场景** |
| C. 无 prewarm（串行/首场景） | `ahead.prewarm` 为空或不含该 index | `done(generateContent(...))`——`done()` 负责包成 `{ step: {id, output}, events: [step_completed] }` 交给主循环提交 |

`return null` 是**唯一的"什么都不提交"出口**：主循环 [#L1840](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1840) 的判断是 `if (change && (change.step || change.patch || change.events?.length))`，`null` 直接跳过；分支 B 里的 `step_failed` 事件是在 `runStep` 内部**自己 commit 掉的**（那次 commit 只有 `events`，没有 `step`/`patch`），所以它不会推进 checkpoint，只留一条事件。

> 注意分支 B 的**顺序**：四道筛子必须在"跳过"之前。跳过是**兜底**，不是默认。

---

## 3. `generateContent`：喂给步骤的输入

[engine.ts#L502-L533](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L502-L533)：

```ts
const generateContent = async (sceneIndex: number, contentSignal: AbortSignal) => {
  const { outlines, languageDirective, taskEngineMode } = outline();
  const stepId = sceneStepId(sceneIndex, 'content');
  const research = output<ResearchOutput>('research');
  return withRouteRetry(
    () => withDeadline(stepId, STEP_DEADLINES_MS.sceneContent, contentSignal, (callSignal) =>
      services.sceneContent(owner, {
        outline: outlines[sceneIndex]!,
        ...materialImages(),
        agents: agents().agents.agents,
        languageDirective,
        // The first scene is generated with the session's requirements;
        // the classroom generates the rest with the task-engine flag only.
        requirements:
          sceneIndex === 0
            ? runRequirements(input, research?.webSearch === true)
            : taskEngineMode ? { taskEngineMode: true } : undefined,
      }, { log, signal: callSignal })),
    retryOptions(stepId, sceneIndex, 500, contentSignal),
  );
};
```

四个输入要点：

1. **`outlines[sceneIndex]!`**——非空断言。并行 prewarm 会一次性把 `[fromIndex, sceneCount)` 都排进去，索引越界会直接崩，但 `pending` 的构造循环本身受 `sceneCount` 约束（[#L545](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L545)），所以这里的安全由 `outline()` 的 `outlines.length` 保证。

2. **`...materialImages()`**（[#L462-L467](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L462-L467)）：从 `material-analysis` 检查点取 `pdfImages` / `imageMapping`；**没有图时整段展开成 `{}`**，即两个字段都不传，而不是传空数组。步骤文件里 `input.pdfImages ?? []` 兜住这条路径。

3. **`agents().agents.agents`**：第 4 册产出的**教师画像数组**（`AgentInfo`：name/role/persona/voice 之类 4 字段），**不是**给对话用的完整花名册。第 5 步只需要"谁来念这页"的身份感，不需要 agent 的头像、优先级和花名册揭示信息。

4. **`requirements` 的场景差异**（最容易踩的一条，见 §14 坑 3）：只有 `sceneIndex === 0` 携带完整会话需求（`runRequirements`，[#L223-L232](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L223-L232)：requirement + userNickname? + userBio? + webSearch? + interactiveMode? + taskEngineMode?）；后续场景**只有 `taskEngineMode` 一个旗标**，其余一律 `undefined`。这正是第 3 册讲过的"需求只喂一次"的延续。

`retryOptions(stepId, sceneIndex, 500, contentSignal)` 里的 **`500`** 是 `refusalStatus`：本步的 `StepRefusal` 会被映射成 HTTP 500，因而**可重试**。对比第 7 册旁白步传的是 `400`（不可重试）——同一个 `StepRefusal` 类，在不同步里命运不同。

`contentSignal` 的来源也分两种：串行分支传的是主循环的 `signal`；prewarm 分支传的是 `prewarmSignal = AbortSignal.any([signal, prewarmAbort.signal])`（[#L540](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L540)）。

---

## 4. 三层重试链

### 4.1 第一层：run 层的 `withRouteRetry`

[retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts) 全文只有 108 行，两个常量 + 一个包装：

```ts
export const FIRST_SCENE_MAX_RETRIES = 2;   // L22
export const SCENE_MAX_RETRIES = 5;          // L24

export const withRouteRetry = async <T>(
  call: () => Promise<T>,
  options: RouteRetryOptions,
): Promise<T> => {
  // 内层：给错误贴上 HTTP 状态与可重试标记
  // 外层：unwrap 回原始错误
};
```

内层 `catch` 的逻辑（[#L60-L92](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L60-L92)）：

| 情况 | 处理 |
|---|---|
| `isAbortError(error)` | 直接 `throw`——**取消永不重试** |
| 其它 | 包成 `RouteStatusError(error, statusCode, isRetryable?)` |

`statusCode` 由 `routeStatus`（[#L40-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L40-L58)）决定，优先级：

1. `error instanceof StepRefusal` ⇒ `options.refusalStatus`（content 步 = **500**）
2. error 自带 `httpStatus` 且在 400–599 ⇒ 用它
3. 否则 ⇒ `upstreamHttpStatus ?? 500`

`isRetryable` 的覆盖逻辑：命中 `rethrow` 谓词或 `isProviderQuotaRefusal` ⇒ 强制 `false`；否则取 `classifyHostFailure(error)?.retryable`。

外层在成功路径之外 `throw error.original`——**调用方看到的是原始错误**，`RouteStatusError` 只是"运输途中贴的标签"。但 `RouteStatusError.statusCode` 仍会被 `failureCodeOf` 读取用于分类，所以标签并非无意义。

### 4.2 `retryOptions`：退避与事件

[engine.ts#L423-L453](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L423-L453)：

```ts
maxRetries: sceneIndex === 0 ? FIRST_SCENE_MAX_RETRIES : SCENE_MAX_RETRIES,
refusalStatus,
sleep: services.sleep,
signal: retrySignal,
onRetry: async (event) => {
  log.warn(`run ${run.id}: retrying ${stepId} (${event.attempt}/${event.maxAttempts}): ${event.cause}`);
  await commit({ events: [{ type: 'step_retry',
    data: { step: stepId, attempt, maxAttempts, reason, cause } }] });
},
```

三点：

- **第 1 个场景 2 次、后续 5 次**。文件头注释（[engine.ts#L1-L60](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1-L60)）解释了这个不对称：首场景失败通常意味着配置层面的问题（模型不可用、prompt 不合法），多试只是浪费时间；后续场景偶发 429/超时更常见，值得多给预算。
- `onRetry` 只提交**事件**，不提交 step，所以 `step_retry` 是可观测的但**不影响 checkpoint**。
- 退避曲线在 packages 层（下一节），`services.sleep` 是可注入的实现（测试里换成同步执行）。

### 4.3 第二层：什么错误算"可重试"

[packages/@openmaic/generation/src/generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L158) 的 `isRetryableGenerationError`，判定优先级从高到低：

| 顺序 | 判定 | 结果 |
|---|---|---|
| 1 | 错误带显式 `isRetryable` 布尔 | 直接采用 |
| 2 | 状态码 ∈ {408, 409, 425, 429} 或 ≥ 500 | 可重试 |
| 3 | 状态码 ∈ {400, 401, 403, 404, 422} 或其它 4xx | **不可** |
| 4 | 递归 `cause` / `lastError` / `errors` 数组 | 沿用第一个命中 |
| 5 | 错误名 `TimeoutError` | 可重试 |
| 6 | 消息正则命中 | 可重试 |

第 6 条的正则（[#L100-L128](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L100-L128)）覆盖：`rate limit`、`too many requests`、`timeout`、`timed out`、`fetch failed`、`network`、`ECONNRESET/ECONNREFUSED/ECONNABORTED`、`ETIMEDOUT`、`ENOTFOUND`、`EPIPE`、`socket hang up`。

退避公式（同文件）：

```
delay = min(16_000, 1_000 × 2^(attempt-1) + jitter)
jitter ≤ 20% × 基数
```

即 1 s → 2 s → 4 s → 8 s → 16 s（封顶），最坏情况 5 次重试的累计等待量级在 30 s 出头——这也是为什么"300 s 每次尝试"配合 5 次重试理论上能占满约 25 分钟：`withDeadline` 在 `withRouteRetry` 的**内侧**，每次尝试重新计时。

### 4.4 第三层：AI SDK 内部重试 = 关闭

[scene-content.ts#L99-L143](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L99-L143) 的 `aiCall` 闭包，两种形态（文本 / 视觉）的 `callLLM` 都传：

```ts
maxRetries: 0,
abortSignal: ctx.signal,
maxOutputTokens: modelInfo?.outputWindow,
```

这条**有专门的测试钉住**：[scene-step-retry-boundary.test.ts#L76](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/scene-step-retry-boundary.test.ts#L76) `'disables AI SDK retries for scene-content model calls'`。理由是重试边界必须唯一：如果 AI SDK 自己重试 3 次、run 层再重试 5 次，最坏情况就是 15 次调用 × 300 s，既烧配额又让 `step_retry` 事件失真。

---

## 5. 时间预算：300 s 按每次尝试

[deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) 全文 68 行：

```ts
export const STEP_DEADLINES_MS = {
  …
  sceneContent: 300_000,   // L19
  sceneActions: 60_000,
  narrationClip: 30_000,
} as const;

export const withDeadline = async <T>(label, ms, outer, run) => {
  const timer = setTimeout(() => controller.abort(new StepTimeoutError(`${label} did not finish within ${ms} s`)), ms);
  …
};
```

`withDeadline` 在 `withRouteRetry` 的 lambda **内部**（[engine.ts#L509-L511](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L509-L511)），因此：

| 嵌套顺序 | 语义 |
|---|---|
| `withRouteRetry( () => withDeadline(...) )` | **每次尝试都有独立的 300 s**；超时抛 `StepTimeoutError`（状态 504），命中"名字含 Timeout"规则 ⇒ 可重试 |
| 假如反过来 | 整条重试链共享 300 s，第二次尝试就没预算了——代码没有这么做 |

这是第 5 步与第 6 步（60 s）与第 7 步（30 s/条）在预算尺度上的最大差异：一屏 slide 元素的 JSON 可能有几十 KB，视觉模式下还要附带最多 20 张图，300 s 是**留给多模态大输出的**。

---

## 6. prewarm：内容生成的并行泳道

### 6.1 触发点

[engine.ts#L1808-L1812](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1808-L1812)：

```ts
// The scenes after the first are generated ahead once the course exists,
// as the classroom does when it takes over from the preview.
if (!ahead.prewarm && step.kind === 'content' && step.sceneIndex >= 1) {
  startContentPrewarm(step.sceneIndex);
}
```

三个条件同时成立才开：**还没开过**（`!ahead.prewarm`）、**当前步是 content**、**场景号 ≥ 1**。合起来的含义是：

- 场景 0 永远是串行生成的（它要带着完整 requirements，也是"配置是否正确"的探针）；
- 课程文档（`Stage`）在第 4 步已经建好，所以从场景 1 起可以像真实课堂接管预览那样提前跑；
- Retry 续跑时也一样：只要走到某个 `content` 且 index ≥ 1 就会尝试开泳道（此时 pending 只包含未 checkpoint 的）。

### 6.2 `startContentPrewarm` 全文

[engine.ts#L541-L576](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L541-L576)：

```ts
const startContentPrewarm = (fromIndex: number) => {
  const concurrency = services.parallelSceneConcurrency();
  const sceneCount = outline().outlines.length;
  const pending: number[] = [];
  for (let index = fromIndex; index < sceneCount; index += 1) {
    if (!steps.has(sceneStepId(index, 'content'))) pending.push(index);
  }
  if (concurrency <= 1 || pending.length <= 1) return;
  const promises = lazyBoundedMap(pending, concurrency, async (sceneIndex) => {
    try {
      const result = await generateContent(sceneIndex, prewarmSignal);
      await commit({ step: { id: sceneStepId(sceneIndex, 'content'), output: result },
                    events: [{ type: 'step_completed', data: { step: … } }] });
      return result;
    } catch (error) {
      return { failed: error };
    }
  }, { shouldContinue: () => !prewarmSignal.aborted });
  ahead.prewarm = new Map(pending.map((sceneIndex, i) => [sceneIndex,
    promises[i]!.then((r) => r ?? { failed: new Error('Content generation was not started') })]));
};
```

四道门槛与两个细节：

| 项 | 说明 |
|---|---|
| `concurrency <= 1` | **不开泳道**，完全保持旧串行行为。默认就是 1 以下（见 §8.5） |
| `pending.length <= 1` | 只剩一个场景时并行没有意义（也不会更快），不开 |
| `!steps.has(...)` | 已经 checkpoint 的场景不再重复生成——这是 Retry 续跑幂等的关键 |
| fn 内 `commit` | 成功即提交，**主循环之后走到该场景时不再记账** |
| fn `catch` 返回 `{failed}` | **绝不抛**：一条 Promise 拒绝会让 `Promise.all` 语义混乱，且失败场景不应打断其它场景 |
| `r ?? {failed: …}` | `lazyBoundedMap` 在 `shouldContinue` 返回 false 时把**剩余项 resolve 成 `undefined`**，所以这里必须把 `undefined` 翻译成一个失败占位，否则 `case 'content'` 会以为"成功但结果是 undefined" |

### 6.3 `lazyBoundedMap`：无屏障的信号量

[lib/utils/concurrency.ts#L47-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/utils/concurrency.ts#L47-L58)：

```ts
export const lazyBoundedMap = <T, R>(items, limit, fn, options) => {
  const active = new Set();
  const bounded = Math.max(1, Math.min(limit, items.length));
  let cursor = 0;
  const take = () => {
    if (options?.shouldContinue?.() === false) return undefined;  // 队列头检查
    …
  };
  …
};
```

与普通 `mapWithConcurrency` 的差别：它**为每项返回一个可单独 await 的 Promise**，同时全局只维持 `limit` 个在飞；一旦 `shouldContinue` 变 false，**尚未启动的项直接以 `undefined` 落定**，已启动的不受影响。`prewarmAbort.abort()` 正是通过这条 `shouldContinue` 掐掉还没开始的场景。

### 6.4 收尾：三处 `prewarmAbort.abort()`

| 位置 | 场景 |
|---|---|
| [engine.ts#L1659](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1659) | `pause()` 的第一行——run 要暂停了，提前生成的内容必须停 |
| [engine.ts#L1844](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1844) | 主 `try` 的 `catch` 入口——任何抛出的步失败都先停泳道 |
| [engine.ts#L1872](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1872) | `finally`——正常完成、被中止、异常退出都要兜底 |

`finally` 里紧跟 [#L1874](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1874) 的 `await Promise.allSettled(ahead.prewarm.values())`：不 await 的话进程可能在 in-flight 请求上被硬切，`commit` 会撞上已释放的 lease 并产生噪声错误。

---

## 7. 跳过语义：一个场景失败不等于整条 run 失败

### 7.1 `skippedScenes` 与 `skippedStepIds`

[engine.ts#L577-L583](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L577-L583)：

```ts
// In parallel mode the browser marks a scene whose content failed and goes
// on with the others, pausing once they are done; these are those scenes.
const skippedScenes = new Map<number, { message: string } & RunFailureCode>();
const skippedStepIds = () =>
  [...skippedScenes.keys()].flatMap((index) =>
    SCENE_STEP_KINDS.map((kind) => sceneStepId(index, kind)));
```

`skippedStepIds()` 的消费点只有一处：[engine.ts#L1770](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1770) 把它并进 `advanceRun` 的 `completed` 集合。所以一个被跳过的场景，它的**三连步全都被视为"已完成"**——主循环不会为它重跑 actions，也不会为它跑 narration（没有内容也没法跑）。

### 7.2 何时回头暂停

[engine.ts#L1788-L1796](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1788-L1796)：

```ts
if (advance.kind === 'complete') {
  const [failedIndex, failure] = [...skippedScenes.entries()].sort(([a], [b]) => a - b)[0] ?? [];
  if (failedIndex !== undefined) {
    // Every other scene is in: pause at the first one that failed.
    const { message, ...code } = failure!;
    await pause(sceneStepId(failedIndex, 'content'), message, code);
    return 'paused';
  }
  …
```

`pause()`（[#L1656-L1673](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1656-L1673)）做四件事：`prewarmAbort.abort()` → `stopLane()`（媒体泳道优雅停）→ commit `state:'paused'` + `error` + `releaseLease:true` → 发 `step_failed`（**不带 `continuing`**）与 `state` 事件。

于是"跳过"和"暂停"在事件流上是**两种不同形状**：

| 事件 | 携带 | 前端表现 |
|---|---|---|
| `step_failed {continuing:true}` | 每个被跳过的场景各一条 | 写入 `skippedScenes[index] = message`，**不设置 `view.error`**（[reducer.ts#L104-L111](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L104-L111)）——课程还在做，不是错误终态 |
| `step_failed`（无 continuing）+ `state {paused}` | 最小 index 的那一个 | 设置 `view.error`，界面显示"已暂停 + Retry" |

### 7.3 进度不由 content 推进

[engine.ts#L1162-L1172](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1162-L1172)（narration 步的 commit）里可以看到 `scenesCompleted` 的计算方式：**统计以 `:narration` 结尾的 checkpoint 数量**。也就是说：

- 内容生成完 ⇒ 进度条**不动**
- 动作装配完 ⇒ 进度条**不动**
- 旁白（TTS）落库 ⇒ 进度 **+1**，并发 `scene_ready` 事件（[reducer.ts#L187-L203](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L187-L203)）

这条事实在并行模式下尤其容易看错：prewarm 让 4 个场景的内容几乎同时完成，但进度条仍然按 narration 的节奏一格一格走。

### 7.4 文档里只有成功的场景

端到端证据：[generation-runs.pg.test.ts#L1451-L1491](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L1451-L1491)。`o2` 的 content 失败被跳过后：

| 断言 | 值 |
|---|---|
| `run.state` | `paused` |
| `run.step` | `scene:1:content`（0-based，即 `o2`） |
| `run.error` | `{ errorCode:'UPSTREAM_ERROR', statusCode:400 }` |
| `scenesCompleted` | **2**（o1、o3 的 narration 都完成了） |
| 场景文档 | `[scene-o1, scene-o3]`——**o2 缺席，不是占位** |
| 后续场景的 `previousSpeeches` | 用的是"生成它之前最后一个**成功**场景"的旁白，即 o1 的 |

Retry 之后 o2 补齐，三场景齐全。

## 8. services 层：模型路由与素材校验

### 8.1 `services.sceneContent`

[services.ts#L464-L482](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L464-L482)：

```ts
async sceneContent(ownerId, input, ctx) {
  // A type with a content slot of its own resolves through it; any other
  // (a type no content path supports) through plain course.content, so it
  // reaches the content step's own refusal.
  const typed = `scene-content:${input.outline.type}`;
  const stage = (LLM_STAGES as readonly string[]).includes(typed) ? typed : 'scene-content';
  return generateSceneContent(
    {
      ...input,
      ...(input.imageMapping
        ? { imageMapping: await storedImageMapping(ownerId, input.outline, input.imageMapping) }
        : {}),
      model: await stageModel(ownerId, stage),
    },
    { ...ctx, resolveVisionImages: ownerVisionImages(ownerId, ctx.log) },
  );
},
```

三个动作：

| 动作 | 说明 |
|---|---|
| **类型化 stage 解析模型** | `scene-content:<type>` 若注册过就用它，否则回落裸 `scene-content`。注释点明意图：**不受支持的类型也要走到步骤自己的 refusal**，而不是在"找不到模型"处炸掉——这样错误码是 `generation-failed` 而不是配置错误 |
| **素材存在性校验** | `storedImageMapping`，只在 `imageMapping` 存在时执行 |
| **注入视觉解析器** | `ownerVisionImages(ownerId, log)`（[#L258](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L258)）——把"asset id → 可内联的 data URL"这件事留在应用层，package 不知道存储后端长什么样 |

### 8.2 `LLM_STAGES` 里的本步成员

[model-routes.ts#L129-L133](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-routes.ts#L129-L133)：

```
'scene-content', 'scene-content:slide', 'scene-content:quiz',
'scene-content:interactive', 'scene-content:pbl'
```

对应模型槽（[model-slots.ts#L97-L101](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L97-L101)）：

| stage | 槽位 |
|---|---|
| `scene-content` | `course.content` |
| `scene-content:slide` | `course.content.slide` |
| `scene-content:quiz` | `course.content.quiz` |
| `scene-content:interactive` | `course.content.interactive` |
| `scene-content:pbl` | `course.content.pbl` |

细粒度键会向上回落（[model-routes.ts#L276](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-routes.ts#L276) 的注释：`scene-content:quiz → scene-content`）。`generate-classroom-job.ts#L85` 用 `LLM_STAGES.filter(s => s.startsWith('scene-content:'))` 收集所有课件模型，说明**这五个 stage 可以配成五个不同的模型**（例如 slide 用长输出强的、interactive 用代码强的）。

### 8.3 `storedImageMapping`：宁失败也不没有图

[services.ts#L660-L677](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L660-L677)：

```ts
for (const imageId of outline.suggestedImageIds ?? []) {
  const assetId = imageMapping[imageId];
  if (assetId && !(await ownerAssetExists(ownerId, assetId))) {
    throw new MaterialImageMissingError(imageId);
  }
}
```

`MaterialImageMissingError`（[#L652-L658](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L652-L658)）的文案是 `The material image ${imageId} is no longer stored`。设计意图写在注释里：**image 元素绝不能指向已经不存在的字节**；run 自己会保持素材存活，所以缺一张意味着真的出了问题，**失败比"生成一张没有图的页"更有价值**。

这条错误发生在**任何模型调用之前**（`stageModel` 之后、`generateSceneContent` 之内才组装 prompt），因此它是最便宜的一类失败——不烧 token。

### 8.4 `parallelSceneConcurrency`

[services.ts#L648](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L648) 直接把 `getParallelSceneConcurrency` 接进来，后者在 [provider-config.ts#L1084-L1097](file:///D:/GITHUB_chat/OpenMAIC/lib/server/provider-config.ts#L1084-L1097)：

```
PARALLEL_SCENE_CONCURRENCY:
  parseInt → 非有限数或 ≤ 0 ⇒ 0（默认串行）
  否则 ⇒ Math.min(raw, 10)
```

注释解释了默认串行的原因：**很多部署的 key 并发配额低，一上来就打并发突刺会撞 429**。上限 10 是硬夹。测试里的默认 fake 是 `parallelSceneConcurrency: () => 0`（[generation-runs.pg.test.ts#L255](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L255)），所以并行分支必须显式改写这个 fake 才会被测到。

---

## 9. 步骤文件解剖：[scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts)（334 行）

文件头注释（[#L1-L7](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L1-L7)）说清了血统：这一步等价于 1.1.x 时代的 `POST /api/generate/scene-content`，但现在它是 run 的**一个步**，不再有独立路由。

六段结构：

| # | 段落 | 行 | 做什么 |
|---|---|---|---|
| 1 | 解构与能力探测 | [#L83-L92](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L83-L92) | `hasVision = !!modelInfo?.capabilities?.vision`——**唯一的 vision 开关来源** |
| 2 | `aiCall` 闭包 | [#L99-L143](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L99-L143) | 两形态 `callLLM`，都 `maxRetries: 0`、都传 `abortSignal: ctx.signal` 与 `maxOutputTokens` |
| 3 | fallback + 图片筛选 | [#L145-L163](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L145-L163) | `applyOutlineFallbacks` → `effectiveOutline`；`suggestedImageIds ∩ pdfImages` 再排序 |
| 4 | N3 resolve-then-slice | [#L165-L286](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L165-L286) | 视觉候选图的解析、预算、保险丝与双剥离（本册 §10） |
| 5 | 调用 package | [#L288-L321](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L288-L321) | `generatedMediaMapping = {}`；只有 pbl 注入 `pblLoopFallback` |
| 6 | refusal | [#L323-L334](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L323-L334) | `!content ⇒ StepRefusal('generation-failed', …)` |

### 9.1 `aiCall` 的两种形态

[#L104-L127](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L104-L127) 是视觉分支：`images?.length && hasVision` 同时成立才走，内容用 `buildVisionUserContent(userPrompt, resolvedImages)` 拼成多模态数组；否则 [#L128-L142](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L128-L142) 是纯文本分支。两者 stage 名都是 `'scene-content'`（**注意：stage 名这里用于日志/token 统计，模型早在 services 层就选好了**）。

视觉分支里还有一句重要的注释（[#L94-L98](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L94-L98)）：这里的 `ctx.resolveVisionImages(images)` 对 run 管线是**防御性 no-op**——因为第 4 段已经预解析过了，传进来的 `src` 已是 data URL；保留它只是为了让"不经预解析直接使用 package 的调用方"永远不会把 asset id 当图片发出去。

### 9.2 `applyOutlineFallbacks`：三种降级

[#L146-L149](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L146-L149) 调用，实现在 [outline-generator.ts#L208-L236](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-generator.ts#L208-L236)：

| 条件 | 降级 |
|---|---|
| procedural-skill 且 `allowProceduralSkill` 为假（职教闸门未开） | 转成 diagram 型（`sanitizeProceduralSkillOutline`） |
| interactive 但 `widgetType` 与 `interactiveConfig` 都缺 | 转 slide |
| pbl 但缺 `pblConfig`，**或没有可用模型** | 转 slide |

`allowProceduralSkill` 来自 `resolveVocationalActive(requirements)`——**这就是为什么第 1 场景和后续场景的 requirements 不对称会有实际后果**：只有 `sceneIndex === 0` 带着完整需求，但 `taskEngineMode` 是从 `outline()` 里读的（[engine.ts#L506](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L506)），所以职教闸门对所有场景都有效。

**降级了类型，产出的 `content` 形状就跟着变了**——所以返回值才必须带 `effectiveOutline`：第 6 步要知道"这页其实是 slide"才能选对动作生成器。

### 9.3 `generatedMediaMapping` 为什么是空的

[#L288-L291](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L288-L291)：

```ts
// ── Media generation is handled client-side in parallel (media-orchestrator.ts) ──
// The content generator receives placeholder IDs (gen_img_1, gen_vid_1) as-is.
const generatedMediaMapping: ImageMapping = {};
```

空映射意味着 [scene-generator.ts#L386-L394](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L386-L394) 的 `resolveImageIds` 对 `gen_img_N` / `gen_vid_N` 走的是"**保留占位**"分支，元素原样留下，由**媒体泳道**（第 7 册）稍后回填真实 URL。占位 id 是内容步与媒体步之间的**契约**。

---

## 10. 视觉图片：15 s 预算、连败 3 保险丝与双剥离

这一段（[scene-content.ts#L165-L286](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L165-L286)）是全文件注释最密的地方，也是**"只降级不失败"哲学的最纯粹体现**。

### 10.1 进入条件

```ts
if (assignedImages && assignedImages.length > 0 && hasVision && imageMapping) {
```

四个都要真。任一为假 ⇒ 完全不解析，`resolvedVisionImages` 留 `undefined`，交给 package 的文本描述路径。

### 10.2 候选集来自共享 helper

```ts
const { withSrc } = partitionImagesForVision(assignedImages, imageMapping, MAX_VISION_IMAGES);
```

[outline-formatters.ts#L40-L79](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-formatters.ts#L40-L79) 返回五元组 `{ sorted, visionSlice, textOnlySlice, noSrcImages, withSrc }`。**步骤用 `withSrc` 决定"探测谁"，生成器用同一 helper 决定"附件是谁"**——两边共用一个排序与切片函数，就不会漂移（注释里点名的 review P2）。

排序规则 `sortDocumentImagesForVision`：`visionPriority` 降序 → `pageNumber` 升序 → `img_(\d+)` 数值升序 → `localeCompare`。它在应用侧（[lib/document/bundle.ts#L165-L179](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L165-L179)）和 package 侧（[#L24-L38](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-formatters.ts#L24-L38)）各有一份**逐字同构**的实现——这是刻意避免 package 依赖应用模块，代价是两份必须同步改。

### 10.3 循环的三种停止

[#L204-L259](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L204-L259)，每轮开头三道检查：

| 检查 | 命中后 |
|---|---|
| `resolvedById.size >= MAX_VISION_IMAGES`（20） | `break`，**不算停止原因**（正常满额） |
| `consecutiveUnresolvable >= 3` | `stopReason = 'fuse'`，break |
| `resolutionMsRemaining <= 0` | `stopReason = 'budget'`，break |

循环体里：

```ts
attempted = await Promise.race([
  ctx.resolveVisionImages([{ id, src: imageMapping[id], width?, height? }]),
  new Promise(resolve => setTimeout(() => resolve('__vision-resolution-budget-expired__'), resolutionMsRemaining)),
]);
} catch { log.error(...); attempted = []; }
  finally { clearTimeout(probeTimer); resolutionMsRemaining -= Date.now() - probeStartedAt; }
```

- **一张卡住的探测也逃不过预算**：`Promise.race` 与剩余时间赛跑，而不是等它自然结束。
- **探测抛错 ⇒ 当作不可解析**（`attempted = []`），并计入连败——"存储挂了必须降级，绝不能失败这一步"。
- `finally` 里扣时间：计时口径是**解析器耗时**，`prepareVisionImages` 的解码重编码**不计入预算**（注释：不能让慢转换饿死探测）。
- 探测成功但图不可用（`prepareVisionImages` 返回空）⇒ `consecutiveUnresolvable = 0` **保险丝复位**，因为存储其实是活的。

两个常量：

| 常量 | 值 | 位置 |
|---|---|---|
| `VISION_RESOLUTION_BUDGET_MS` | 15_000 | [#L43](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L43)，复用 ingest-drain 的 15 s 常量 |
| `MAX_CONSECUTIVE_UNRESOLVABLE_VISION_IMAGES` | 3 | [#L51](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L51) |

### 10.4 双剥离：防止悬空引用

[#L266-L276](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L266-L276)：

```ts
const droppedIds = new Set(withSrc.filter((c) => !resolvedById.has(c.id)).map((c) => c.id));
if (droppedIds.size > 0) {
  assignedImages = assignedImages.filter((img) => !droppedIds.has(img.id));
  visionImageMapping = Object.fromEntries(
    Object.entries(imageMapping).filter(([id]) => !droppedIds.has(id)));
}
```

**未解析的 id 同时从"分配图集合"和"映射表"里删掉。** 为什么这么狠？因为模型若幻觉引用了一个被丢弃的 id，`resolveImageIds` 会命中"没有映射 ⇒ 删除元素"这条**已有的干净路径**（[#L377-L380](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L377-L380)），而不是把一条已经不存在的 asset id 写进 `src` 落库（注释点名的 review P3）。

最终只发**一条汇总 warn**（[#L277-L285](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L277-L285)），文案里带上是 fuse 还是 budget 触发的、还剩几张图、其余降级为文本。

### 10.5 `prepareVisionImages`：附件本身的瘦身

[vision-images.ts#L24-L117](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts#L24-L117)。**只压缩内联附件，不动源资产。** 免处理的过审条件（[#L34-L52](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts#L34-L52)）：格式 ∈ 允许集 ∧ base64 ≤ 1 MiB ∧ 单页 ∧ 色彩空间 srgb 或黑白 ∧ 深度 uchar ∧ 最长边 ≤ 2048。

任一条不过 ⇒ 走 sharp 流水线（[#L63-L105](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts#L63-L105)）：`rotate()` 自动转向 → `resize(inside, withoutEnlargement)` 到 2048 → `flatten('#ffffff')` 去透明 → `toColourspace('srgb')` → `raw({depth:'uchar'})` 解码一次；然后按 JPEG 质量梯队 **[82, 65, 45, 30, 20]** 逐档编码，`ceil(bytes/3)*4`（估算 base64 长度）≤ 1 MiB 即采用并 `log.info('Transformed vision image')`；五档全超 ⇒ 抛 `'Encoded image exceeds the vision base64 payload cap'`，被外层 `catch` 收成 `log.warn('Dropped vision image')` —— **仍然只是丢图，不失败**。

免处理路径还额外做一次 `sharp(buffer, {failOn:'error'}).raw().toBuffer()`（[#L54-L55](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts#L54-L55)）：元数据看不出截断的像素数据，必须真解一遍。

常量（[lib/constants/generation.ts#L10-L14](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L10-L14)）：

| 常量 | 值 |
|---|---|
| `MAX_VISION_IMAGES` | 20 |
| `MAX_VISION_IMAGE_BASE64_BYTES` | 1 MiB（1024×1024） |
| `MAX_VISION_IMAGE_DIMENSION` | 2048 |

### 10.6 两条断言

[scene-content.test.ts#L87-L100](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/scene-content.test.ts#L87-L100) 把整套语义钉在一起：fake 的解析器对 `src === 'asset-2'` 返回空（模拟一张解析不出来的图），断言结果是

| 断言 | 值 | 含义 |
|---|---|---|
| `options.visionEnabled` | `true` | 走视觉分支 |
| `options.assignedImages.map(id)` | `['img_1']` | `img_2` **被剥离** |
| `options.imageMapping` | `{img_1:'asset-1', img_3:'asset-3'}` | `img_2` **同时被剥离**，注意 `img_3` 不在候选集所以保留 |
| `options.resolvedVisionImages` | `[{id:'img_1', src:<data url>}]` | 只有解析成功的一张真成了附件 |

[#L102-L113](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/scene-content.test.ts#L102-L113) 钉 refusal：package 返回 `null` ⇒ `StepRefusal{ reason:'generation-failed', message:'Failed to generate content: Leaves' }`。

---

## 11. package 层分派：一个函数四条路

[scene-generator.ts#L232-L319](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L232-L319)：

```ts
export async function generateSceneContent(outline, aiCall, options = {}): Promise<… | null>
```

| 输入 `outline.type` | 去向 | 行 |
|---|---|---|
| `interactive` | 先 `convertInteractiveConfigToWidget`（兼容老 `interactiveConfig`）；仍无 `widgetType` ⇒ 兜底成 `simulation` + `widgetOutline:{concept:title}`；然后 `generateWidgetContent` | [#L260-L285](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L260-L285) |
| `slide` | `generateSlideContent(...)`（12 个位置参数，含 `editDirective`/`baselineContent`） | [#L288-L303](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L288-L303) |
| `quiz` | `generateQuizContent` | [#L304-L305](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L304-L305) |
| `pbl` | `generatePBLSceneContent` | [#L306-L315](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L306-L315) |
| **其它任何值** | `default: return null` | [#L316-L317](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L316-L317) |

`null` 是 package 与步骤之间唯一的"我做不出来"信号，步骤把它翻译成 `StepRefusal`（§9 第 6 段）。**未知类型必须经过这里**——这也是 §8.1 里"回落裸 `scene-content` stage"的原因：让模型能解析出来，好让失败以 refusal 的形式而不是配置错误的形式呈现。

失败还有一条**旁路**：`SceneContentFailureCode = 'prompt-unavailable' | 'invalid-model-output'`（[#L79](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L79)），通过 `options.onFailure` 上报。**run 管线不传 `onFailure`**——它是给 MAIC Editor 的 regenerate 通道和测试用的；run 只看 `null`。

`aiCall` 是一个注入点：package **完全不知道模型是谁**，只知道"给我一个函数，我传 system + user（+ 图片），还我一段字符串"。这让四种生成器在单元测试里都能用假函数跑。

---

## 12. slide 深潜：一屏元素的诞生

### 12.1 图片怎么进 prompt

[scene-generator.ts#L650-L698](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L650-L698)：

| 情况 | `assignedImagesText` |
|---|---|
| 一张都没有 | 写死的 **`'无可用图片，禁止插入任何 image 元素'`**（[#L651](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L651)） |
| 视觉模式（`visionEnabled && imageMapping`） | `visionSlice` 用 `formatImagePlaceholder`（尾缀 `[see attached]`），`textOnlySlice + noSrcImages` 用 `formatImageDescription`（完整文字描述）；两者拼接 |
| 非视觉模式 | `sorted` **整份**都走 `formatImageDescription`——不做切片，顺序也是分区重构之前的顺序（注释专门解释了这点，避免"分区重构悄悄改变了非视觉模型的文本顺序"） |

附件实体在 [#L681-L694](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L681-L694) 决定：优先用步骤预解析的 `resolvedVisionImages`（按 id 匹配），否则回落 `imageMapping[id]`（此时它是个待调用方解析的 asset id）。

### 12.2 媒体占位怎么进 prompt

[#L700-L734](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L700-L734)：把 `outline.mediaGenerations` 按 `type` 拆成图/视频两组，渲染成

```
AI-Generated Images (use these IDs as image element src):
- <elementId>: "<prompt>" (aspect ratio: 16:9)

AI-Generated Videos (use these IDs as video element mediaRef):
- …
```

三个开关由此算出：`generatedImageEnabled`、`generatedVideoEnabled`、`imageElementEnabled = hasAssignedImages || generatedImageEnabled`。若原本写的是"禁止插入任何 image 元素"，媒体文本会**替换**掉那句而不是追加（[#L728-L730](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L728-L730)）——否则模型会同时读到"禁止插 image"和"把这些 id 当 image src 用"两条矛盾指令。

### 12.3 提示词构造与 EDIT MODE

[#L738-L754](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L738-L754) `buildSlideContentPrompt(flags, vars)`，`vars` 里 `elements: '（根据要点自动生成）'` 是固定占位。`prompts` 为 `null` ⇒ `onFailure('prompt-unavailable')` + 返回 null（该类型的模板在该语言下没配时才会发生）。

[#L764-L797](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L764-L797) 是 **EDIT MODE**：仅当 `editDirective` 或 `baselineContent` 存在时，把 `## EDIT MODE` 段落追加到 user prompt——这是 MAIC Editor 的 `regenerate_scene` 工具通道。**run 管线从不传这两个参数**（services 层没有它们的入口），所以对生成管线而言这段代码永远不触发；注释明确写了"不传时 prompt 与默认课程生成**逐字节相同**"。

### 12.4 解析与两道硬校验

| 步骤 | 行为 |
|---|---|
| `parseJsonResponse` | json-repair 系：剥 `think` 前缀、修 `"height: 76"` 型碎片、取最后一个可解析载荷 |
| 结构检查 | `!generatedData.elements || !Array.isArray(...)` ⇒ `invalid-model-output` + **null** |
| `findInvalidShapeRotation` | shape 的 rotate 非法 ⇒ 记 error + `invalid-model-output` + **null**（[#L808-L813](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L808-L813)） |

后一条值得注意：**旋转非法不是修复而是整页作废**。因为非法角度会一路传到 PPTX 导出与播放器，而一张没有内容的页比一张歪掉的页更容易被发现和重试。

### 12.5 后处理流水线（顺序即语义）

[#L834-L888](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L834-L888)：

```
fixElementDefaults      → processLatexElements → resolveImageIds
→ normalizeGeneratedVideoRefs → 补 id + rotate 白名单 → background → { elements, background, remark }
```

| 阶段 | 关键行为 |
|---|---|
| **`fixElementDefaults`**（[#L488-L572](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L488-L572)） | 先 `stripNulls`（JSON `null` 视为缺失——模型用 null 表示"没有值"，契约把"存在但为 null"视为畸形；**数组不动**，元组里的 null 是真畸形）；再 `normalizeElement`（来自 `@openmaic/dsl`，fail-loud）。异常时：**shape ⇒ `shapeToTextFallback` 抢救标签文字；其它类型 ⇒ 删元素**。理由写在注释里：留着畸形元素会让 `getElementRange` / `BaseLineElement` / PPTX 导出直接崩，"丢一个元素只是这页差一点，留一个畸形元素可能毁掉整场" |
| ↳ shape 几何二次校验 | `validateShapeGeometry` 不通过 ⇒ 同样走 `shapeToTextFallback`（[#L527-L536](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L527-L536)） |
| ↳ text 归正 | `normalized.type === 'text' ⇒ rotate: 0`（模型写的文字一律水平；shape 抢救出的文字框保留原角度） |
| ↳ 图片盒贴合真实宽高比 | **此刻 `src` 还是 `img_N`**，用 `assignedImages` 的真实宽高算比例；偏差 > 10% 才动；用 **contain-fit**（模型盒内能放下的最大真实比例盒、居中）**只缩不涨**，避免盖住周围已排好的元素 |
| **`processLatexElements`** | KaTeX 把 `latex` 渲成 `html`；渲染失败 ⇒ 删元素 |
| **`resolveImageIds`**（[#L360-L418](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L360-L418)） | `isImageIdReference`（`/^img_\d+$/`）⇒ 有映射就换成真 src，**没映射就删元素**；无 `src` 的 image ⇒ 删；`gen_img_N`/`gen_vid_N` 占位 ⇒ **原样保留**；video 元素要求 `src` 或 `mediaRef` 至少有一个 |
| **`normalizeGeneratedVideoRefs`** | 把视频引用统一成媒体泳道认得的形状 |
| **id 与 rotate 白名单** | `id = ${type}_${nanoid(8)}`；`rotate` 只在 `shape` / `text` 且是有限数时保留，其余置 0 |
| **background** | 只认 `solid + color` 或 `gradient + gradient`，其它（包括畸形值）**静默丢弃**，返回 `undefined`，渲染层用自己的默认背景 |
| **remark** | `generatedData.remark || outline.description`——没有讲解备注时退回大纲描述，保证第 7 步总有文本可念 |

### 12.6 slide 的"元素数"没有任何下限

流水线不会因为"只有 2 个元素"或"没有标题"而拒绝——它只保证**每个元素都合法**。这类结构性校验是第 6 步 `assertValidScene` 的职责（第 6 册）。

---

## 13. quiz / interactive / pbl 三型

### 13.1 quiz：唯一有"契约校验"的类型

[scene-generator.ts#L914-L978](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L914-L978)。

| 环节 | 行为 |
|---|---|
| 配置缺省 | `outline.quizConfig \|\| { questionCount: 3, difficulty: 'medium', questionTypes: ['single'] }`——大纲没给就默认 **3 道单选** |
| prompt | `buildPrompt(PROMPT_IDS.QUIZ_CONTENT, { title, description, keyPoints, questionCount, difficulty, questionTypes: types.join(', '), languageDirective })` |
| 调用 | `aiCall(prompts.system, prompts.user)`——**不传第三参**，所以 quiz **永远不走视觉附件**，即便模型支持 |
| 解析 | `parseJsonResponse<QuizQuestion[]>`；不是数组 ⇒ `invalid-model-output` + null |
| 归一 | `id = q.id \|\| 'q_' + nanoid(8)`；`short_answer` ⇒ `options = undefined`、`answer = undefined`、`hasAnswer = false`；其它题型 ⇒ `normalizeQuizOptions`（字符串数组 → `{value,label}` 对）+ `normalizeQuizAnswer` + `hasAnswer = true` |
| 契约 | `findQuizOptionsContractFailure(questions)`——**违反即整场景 null** |

契约（[#L989-L1029](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L989-L1029)）逐题检查，`short_answer` 跳过：

| 规则 | 失败文案 |
|---|---|
| options 非空 | `choice question has no options` |
| 每个 `option.value` 匹配 `/^[A-Z]$/` | `option N value X is not a single letter A-Z` |
| value 不重复 | `option N repeats value X` |
| `answer` 非空 | `answer key does not reference an option value` |
| answer 每项都命中某个 option value | `answer "X" does not match an option value` |

这是四种类型里**唯一在生成阶段做跨字段一致性校验**的：一道答案指向不存在选项的题，在播放器里是"点了永远错"，比排版错误严重得多，所以宁可 refusal 让 run 层重试（重试预算 5 次，模型下一次大概率能写对）。

### 13.2 interactive（widget）：六型 HTML 页

[scene-generator.ts#L1270-L1422](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1270-L1422)。

前置：`!widgetType || !widgetOutline` ⇒ warn + **null**（[#L1284-L1287](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1284-L1287)）。注意 §11 的分派已经先兜过一次 `simulation`，所以走到这里的 null 通常意味着 `widgetOutline` 缺失。

| `widgetType` | PROMPT_ID | 关键变量（都有默认值） |
|---|---|---|
| `simulation` | `SIMULATION_CONTENT` | `conceptName=concept\|title`、`variables=keyVariables.join(', ')`、`designIdea: ''` |
| `diagram` | `DIAGRAM_CONTENT` | `diagramType='flowchart'`、`nodeCount ?? prescribedNodes.length`、两个开关 `hasNodeCount`（是正数才 true）/ `hasPrescribedNodes`（有节点才 true） |
| `code` | `CODE_CONTENT` | `programmingLanguage='python'`；`starterCode`/`testCases`/`hints` **都传空串**，让模型自己出题与测试 |
| `game` | `GAME_CONTENT` | `gameType='quiz'`、`scoring={correctPoints:10, speedBonus:5}` |
| `visualization3d` | `VISUALIZATION3D_CONTENT` | `visualizationType='custom'`、`objects`/`interactions` 默认空数组 |
| `procedural-skill` | `PROCEDURAL_SKILL_CONTENT` | **双闸门**：`!options.allowProceduralSkill` ⇒ warn + null（[#L1362-L1366](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1362-L1366)）；`task = task\|concept\|title`，另有 `tools/steps/successCriteria/errorConsequences` |
| 其它 | — | `default:` warn `Unknown widget type` + null |

`procedural-skill` 的**双闸门**是有意的：`applyOutlineFallbacks`（§9.2）会在更早把这种大纲改成 diagram，但如果调用方绕过 fallback 直接进来（例如 package 消费者），这里再拦一次。测试 `procedural-skill-content-gates` 钉的就是第二道。

产出三步：`extractHtml`（[#L1229+](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1229-L1260)：优先取完整 `<!DOCTYPE` 文档，退化为剥 code fence）⇒ `findInteractiveScriptSyntaxFailure`（[interactive-script-validator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/interactive-script-validator.ts)：内联 JS 解析不了就拒绝）⇒ `extractWidgetConfig(html, widgetType)` ⇒ 返回 `{ html: postProcessInteractiveHtml(html), widgetType, widgetConfig }`。

脚本校验这条特别值得记住，注释写的是 **"Reject visually valid but inert widgets"**：渲染出来很好看但点下去没反应的交互，比一张丑页面更糟——因为它要到学生真正操作时才暴露。`postProcessInteractiveHtml`（[interactive-post-processor.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/interactive-post-processor.ts)）做落地前的规范化，第 6 步与编辑器都在它的输出上工作。

### 13.3 pbl：单调用优先，循环规划器兜底

[scene-generator.ts#L1141-L1223](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1141-L1223)。

| 环节 | 行为 |
|---|---|
| 前置 | `!outline.pblConfig` ⇒ error + **null**（[#L1150-L1154](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1150-L1154)）；§9.2 的 fallback 本该已把它改成 slide，能走到这里说明是 package 直连调用 |
| 输入 | `PBLPlannerV2Input { outline, courseContext:{ allOutlines:[outline], languageDirective ?? DEFAULT }, user:{nickname,bio,requirement}?, targetLanguage }`——注意 **`allOutlines` 只放了当前这一条**，单场景规划器看不到整门课 |
| 首选 | `generatePBLV2ProjectSingleCall(plannerInput, aiCall, {logger})`——**一次调用**产出 `projectV2`，成功即 `{ projectV2 }` |
| 失败判定 | `skipLoopFallback = plannerErrorStatus(error) !== undefined \|\| isAbortError(error)` |
| 兜底 | 仅当**注入了** `pblLoopFallback` 且 `!skipLoopFallback` ⇒ 走应用层循环规划器 |
| 终局 | 抛 `PBLGenerationError`，`cause` 保留终端原因，`statusCode = plannerErrorStatus(fallbackError) ?? plannerErrorStatus(singleCallError)` |

`skipLoopFallback` 的逻辑（注释 [#L1189-L1194](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1189-L1194)）：**provider/HTTP 层失败或用户已取消 ⇒ 不再打循环规划器**——循环要多打几十次同一个坏掉的 provider，只会更糟。只有 `PlannerV2Error`（schema/解析类）这类"模型写法不对"的失败才值得换一种规划方式。

`pblLoopFallback` **只有 run 管线会注入**，注入点在 [scene-content.ts#L309-L320](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L309-L320)，实现是应用层的 `generatePBLV2Project`（`lib/pbl/v2/agents/planner`）。package 自身不依赖它，所以裸用 package 时 pbl 只有单调用一条路。

`plannerErrorStatus`（[#L1125](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1125-L1139)）递归 `cause` / `lastError` 找 `statusCode`，带 `seen` 集合防环。

六个测试全部在 [pbl-v2-scene-routing.test.ts#L97-L170](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/pbl-v2-scene-routing.test.ts#L97-L170)：单调用先于应用 loop、校验失败才走 loop、loop 失败保留终端 cause、provider 失败跳过 loop、取消跳过 loop、scenario 失败走 no-degradation。

**pbl 是四种类型里唯一可能抛非 `StepRefusal` 错误的**（`PBLGenerationError` 带 statusCode），它照样进入 §4 的重试链——`statusCode ≥ 500` 或 429 就重试，400/401/403 就立刻放弃。

---

## 14. 事件、失败链、常量与坑

### 14.1 本步发出的事件

| 事件 | 发出者 | 携带 |
|---|---|---|
| `step_started {step:'scene:N:content'}` | 主循环 [#L1814-L1825](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1814-L1825) | 顺带让 reducer **清掉该 index 的 skip 标记** |
| `step_retry {step, attempt, maxAttempts, reason, cause}` | `retryOptions.onRetry` [#L434-L451](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L434-L451) | 串行与 prewarm 都会发 |
| `step_completed {step}` | `done()`（串行）或 prewarm fn（并行 [#L555-L560](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L555-L560)） | 二者互斥，不会重复 |
| `step_failed {step, message, …code, continuing:true}` | `case 'content'` 跳过分支 [#L696-L703](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L696-L703) | **只此一处带 `continuing`** |
| `step_failed` + `state {paused}` | `pause()` [#L1661-L1672](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1661-L1672) | 回头暂停时 |

reducer 侧（[reducer.ts#L96-L111](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L96-L111)）：

```ts
case 'step_started':  // 清掉 skippedScenes[该 index] —— "重跑的场景不再是越过它继续做的场景"
case 'step_failed':   // continuing ⇒ skippedScenes[index] = message，不动 view.error
```

`scene_ready`（真正让进度条走的）**不是本步发的**，而是 narration；见 §7.3。

### 14.2 前端看到什么

| 层 | 表现 |
|---|---|
| 步进位 | `stepOfRun`：`/^scene:\d+:(content\|actions\|narration)$/` 命中后 **`content` → `'slide-content'`，`actions` 与 `narration` 都 → `'actions'`**（[preview-steps.ts#L41-L42](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L41-L42)）——所以并行模式下 `slide-content` 位会长时间亮着，它代表"有场景还在生成内容"，不代表第几个 |
| 可视化组件 | `case 'slide-content': return <ContentVisualizer />`（[visualizers.tsx#L40-L41](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L40-L41)） |
| 组件内容 | [#L370-L392](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L370-L392)：4 张类型卡（Slide 蓝 / Quiz 紫 / PBL 琥珀 / Interactive 翠）**每 3200 ms 轮播的纯装饰动画**，`useState(0)` + `setInterval`，**不吃任何事件数据** |

结论：预览页对"第 5 步"的全部真实反馈只有**进度数字**（来自 `scene_ready`）和**被跳过场景的文案**（`skippedScenes`）。动画与场景内容毫无关系。

### 14.3 失败链四层

| 层 | 触发条件 | 结果 |
|---|---|---|
| **L1 类型内降级（不失败）** | 元素畸形、latex 失败、图片无映射、quiz 之外的形状问题、背景非法、视觉图解析不出来 | 删元素 / 丢弃字段 / 降级文本，**继续生成** |
| **L2 package 返回 null** | 解析不出 JSON、quiz 契约失败、widget 脚本不可解析、procedural-skill 闸门、类型未知、`prompt-unavailable` | 步骤抛 `StepRefusal('generation-failed')`，`retryOptions` 把它映射成 **500 ⇒ 可重试** |
| **L3 抛异常** | `MaterialImageMissingError`、provider 429/5xx、`StepTimeoutError`（300 s）、`PBLGenerationError` | 交 `withRouteRetry` + `isRetryableGenerationError`：可重试就退避重来（第 1 场景最多 2 次、其它 5 次），否则上抛 |
| **L4 run 结局** | 上抛到达 `case 'content'` | 串行 ⇒ `StepFailedError` ⇒ `pause()` ⇒ `paused`；并行 ⇒ 四道筛子后**跳过该场景**，全部跑完再在最小 index 处 `pause()`；lease 丢失 / abort ⇒ `interrupted`，**不改状态** |

`isNonRetryableHostFailure`（L4 筛子③）是"宿主级配置问题"的判定：模型没分配、槽位禁用之类。它的存在让并行模式不会把 10 个场景全跑失败一遍才发现"你根本没配模型"。

### 14.4 常量速查

| 常量 | 值 | 位置 |
|---|---|---|
| `STEP_DEADLINES_MS.sceneContent` | 300_000（每次尝试） | [deadline.ts#L19](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L19) |
| `FIRST_SCENE_MAX_RETRIES` | 2 | [retry.ts#L22](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L22) |
| `SCENE_MAX_RETRIES` | 5 | [retry.ts#L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L24) |
| `refusalStatus`（content） | 500（可重试） | [engine.ts#L531](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L531) |
| 退避基数 / 封顶 | 1000 ms × 2^(n-1) / 16000 ms | [generation-retry.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L30-L40) |
| `PARALLEL_SCENE_CONCURRENCY` | 默认 0（串行），上限 10 | [provider-config.ts#L1093-L1097](file:///D:/GITHUB_chat/OpenMAIC/lib/server/provider-config.ts#L1093-L1097) |
| `MAX_VISION_IMAGES` | 20 | [constants/generation.ts#L10](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L10) |
| `MAX_VISION_IMAGE_BASE64_BYTES` | 1 MiB | [#L13](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L13) |
| `MAX_VISION_IMAGE_DIMENSION` | 2048 | [#L14](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L14) |
| `VISION_RESOLUTION_BUDGET_MS` | 15_000 | [scene-content.ts#L43](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L43) |
| `MAX_CONSECUTIVE_UNRESOLVABLE_VISION_IMAGES` | 3 | [#L51](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts#L51) |
| JPEG 质量梯队 | 82, 65, 45, 30, 20 | [vision-images.ts#L20](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts#L20) |
| quiz 默认配置 | 3 题 / medium / single | [scene-generator.ts#L921-L925](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L921-L925) |
| 图片宽高比容差 | 10% | [scene-generator.ts#L551](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L551) |
| 元素 id 形式 | `${type}_${nanoid(8)}` | [#L861](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L861) |

### 14.5 易踩坑清单

1. **只有场景 0 带完整 requirements**——想给每页都加"用户昵称/需求"必须改 `generateContent`，光改 `runRequirements` 无效。
2. **`maxRetries: 0` 是刻意的**，别在步骤里"顺手给模型加点重试"，那会让 run 层的 `step_retry` 事件与配额消耗双双失真（测试会红）。
3. **300 s 是每次尝试**，不是整步；5 次重试最坏接近 25 分钟。改预算时想清楚 `withDeadline` 在 `withRouteRetry` 的内侧还是外侧。
4. **`refusalStatus` content=500、narration=400**——同一个 `StepRefusal` 类在不同步里命运不同，别照抄邻居步的写法。
5. **首场景 2 次、后续 5 次**：调试时"为什么这个场景只试了两次"多半是这个不对称，不是 bug。
6. **prewarm 只在 `sceneIndex >= 1` 且课程已存在时开**——场景 0 永远串行，别指望并行能加速首屏。
7. **`concurrency <= 1 || pending.length <= 1` 直接不开泳道**：单场景课程、或 Retry 只剩最后一场景时，行为与串行完全一致。
8. **默认并发是 0**：不设 `PARALLEL_SCENE_CONCURRENCY` 就没有并行；测试 fake 默认也返回 0，测并行分支必须显式改写。
9. **prewarm 成功即自行 commit**，`case 'content'` 返回 `{}` 是**空提交**——不要在这里"补一次 checkpoint"，那会写两遍。
10. **prewarm fn 的 catch 返回 `{failed}` 而非抛出**：改成抛出会污染 `Promise.allSettled` 之前的兄弟项，也会让"跳过"语义消失。
11. **`lazyBoundedMap` 停止后返回 `undefined`**，靠 `promises[i].then(r => r ?? {failed: …})` 兜住；去掉这个兜底会伪装成"成功但内容为 undefined"。
12. **被跳过场景的三个步都算完成**（`skippedStepIds`），所以 actions/narration 不会为它跑——想"只重试内容"必须靠 Retry 从 content 起步并**先清 skip 标记**（reducer 的 `step_started` 分支负责这件事）。
13. **跳过 ≠ 错误终态**：`step_failed {continuing:true}` 不设置 `view.error`；只有最后那次不带 `continuing` 的 `pause` 才是。
14. **文档里没有占位**：被跳过场景直接缺席，不是"空场景"。前端/导出/播放器都要按"场景数可能少于大纲数"处理。
15. **进度条不由本步推进**：`scenesCompleted` 只数 `:narration` checkpoint，并行时内容 4 页同时完成而进度仍是 0/4，这不是卡住。
16. **`effectiveOutline` 才是真相**：类型可能被降级（interactive→slide、pbl→slide、procedural-skill→diagram），存原始 outline 会让第 6 步选错生成器。
17. **视觉图解析失败绝不让步骤失败**——但 `MaterialImageMissingError` 会。区别在于前者"图看不清"，后者"图已经不在了"。
18. **双剥离**：未解析 id 同时从 `assignedImages` 与 `imageMapping` 删除。只删一边 = 模型仍可能引用到一个不存在的映射，写出悬空 `src`。
19. **15 s 预算只计解析器时间**，`prepareVisionImages` 的转码不计——所以真实耗时可以远超 15 s，别把它当硬 deadline 断言。
20. **20 张是"解析后"的上限**，不是候选上限：候选可以更多，落选者降级为文本描述。
21. **quiz 不走视觉**（`aiCall` 两参调用），即便配了多模态模型也不会带附件——题目不需要看图。
22. **quiz 契约违反是整场景作废**，不是删掉那一题：`findQuizOptionsContractFailure` 返回任何原因都 ⇒ null ⇒ refusal ⇒ 重试。
23. **procedural-skill 两道闸门**：`applyOutlineFallbacks`（改类型）与 `generateWidgetContent`（返 null）。绕过前者不会绕过后者。
24. **pbl 的 `allOutlines` 只含当前 outline**——单场景规划器看不到全课上下文，这是设计而非疏漏。
25. **provider 失败不跑 loop fallback**：`plannerErrorStatus(error) !== undefined` 即跳过。看到"PBL 只试了一次就失败"多半是这个闸门。
26. **`gen_img_1` / `gen_vid_1` 占位是留给媒体泳道的**，`generatedMediaMapping = {}` 是刻意的空——把它填上反而会让内容步与媒体步重复生成。
27. **EDIT MODE 参数在 run 管线永远为空**：读 `scene-generator.ts` 时别被 `editDirective`/`baselineContent` 误导成"生成管线也有编辑通道"。
28. **`onFailure` 在 run 管线没人听**：错误码信息在这里是丢弃的，只有 `null` 传得出去。

### 14.6 代码地图

| 关注点 | 文件 |
|---|---|
| 计划与步骤 id | [lib/server/generation/run/plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) |
| 入口分支 / prewarm / 暂停 | [lib/server/generation/run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) |
| 重试与状态码包装 | [lib/server/generation/run/retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts) |
| 可重试判定与退避 | [packages/@openmaic/generation/src/generation-retry.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts) |
| 单尝试超时 | [lib/server/generation/run/deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) |
| 模型/素材/并发接线 | [lib/server/generation/run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) |
| 步骤实现 | [lib/server/generation/steps/scene-content.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-content.ts) |
| 视觉附件瘦身 | [lib/server/generation/vision-images.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/vision-images.ts) |
| 图片分区与格式 | [packages/@openmaic/generation/src/outline-formatters.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-formatters.ts) |
| 大纲降级 | [packages/@openmaic/generation/src/outline-generator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-generator.ts) |
| 四种内容生成 | [packages/@openmaic/generation/src/scene-generator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts) |
| 交互 HTML 后处理/校验 | [interactive-post-processor.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/interactive-post-processor.ts) · [interactive-script-validator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/interactive-script-validator.ts) |
| PBL 单调用规划器 | [pbl/planner-single-call.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/pbl/planner-single-call.ts) |
| 并发原语 | [lib/utils/concurrency.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/utils/concurrency.ts) |
| 前端事件归约 | [lib/generation-run-client/reducer.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts) · [preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) |
| 预览动画 | [app/generation-preview/components/visualizers.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx) |

### 14.7 动手验证

| 想验证什么 | 怎么做 |
|---|---|
| 跳过语义端到端 | `pnpm vitest run tests/server/generation-runs.pg.test.ts -t "skipped"`：断言 `paused` + `run.step='scene:1:content'` + `scenesCompleted=2` + 文档 `[o1,o3]` + Retry 补齐 |
| 并行顺序 | 同文件并发 3 的用例：`content:o3` **早于** `actions:o2`，但 `actions` 严格 o1→o2→o3 |
| 视觉剥离 | `pnpm vitest run tests/server/generation-steps/scene-content.test.ts`：一条附件解析失败 ⇒ `assignedImages` 与 `imageMapping` 同时少一个 |
| AI SDK 不重试 | `pnpm vitest run tests/generation/scene-step-retry-boundary.test.ts` |
| PBL 路由六条 | `pnpm vitest run tests/generation/pbl-v2-scene-routing.test.ts` |
| quiz 契约 | `pnpm vitest run tests/generation/`（含 quiz 选项契约与 `json-repair` 相关用例） |
| 大纲掉图后的提示词一致性 | `pnpm vitest run tests/generation/outline-standard-desync.test.ts` |
| 真开并行 | `PARALLEL_SCENE_CONCURRENCY=3 pnpm dev`，做一门 4 场景课，观察日志里多条 `scene:N:content` 交叠、但 `scene_ready` 仍按序 |
| 看降级不失败 | 把某张素材图对应资产删掉后重试 → 应看到 `Dropped vision image` 或 `Stopped probing …`，而不是 run 失败；删掉 `storedImageMapping` 认得的资产则本场景**必须**失败 |

---

## 15. 一句话总结

> **第 5 步把一条大纲变成一屏真内容，并且是整条管线里唯一"允许局部失败"的步。**

三条不变量：

1. **模型只被叫一次（每个尝试）**：`maxRetries: 0` 关掉了 AI SDK 的内部重试，所有重试都活在 run 层的退避循环里——第 1 场景 2 次、其余 5 次、每个尝试 300 s、`StepRefusal` 映射成可重试的 500。
2. **能降级就绝不失败**：畸形元素被修好或删掉，latex 渲不出就丢元素，图片解析不出来就退回文本描述（15 s 预算 + 连败 3 保险丝，且未解析 id 双剥离）；只有"整页做不出来"（解析失败、quiz 契约违反、脚本不可执行、类型未知、素材已不存在）才抛 `StepRefusal`。
3. **内容完成不等于场景完成**：checkpoint 写的是 `{content, effectiveOutline}`，进度条 `scenesCompleted` 只由 narration 推进；并行 prewarm 让内容提前完成、让失败的场景被跳过，但**文档与进度仍然按 narration 的节奏落定**。

下一步：[第 6 册 场景动作](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md)——把这一屏元素配上"老师点开它们"的动作序列。
