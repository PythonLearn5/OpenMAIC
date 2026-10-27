# 课程生成管线 · 第 6 册：场景动作与装配（scene:N:actions）

<!--
  职责：详解生成管线**第 6 步「场景动作」**——每个场景三连步的第二步。它把第 5 步
  产出的 `SceneContent`（一屏元素 / 一组题目 / 一页 HTML / 一个项目）配上"老师怎么
  把这屏讲完"的动作序列：`speech` 旁白、`spotlight` / `laser` 聚焦、`play_video`、
  `discussion`、四个 `widget_*`，并把内容 + 动作 + stageId + order 装配成一个**完整
  场景**，在校验通过后才写入检查点。它同时产出本场景的 `previousSpeeches`，交给后续
  场景做同一次课的衔接。
  覆盖：`case 'actions'` 的输入装配、60 s 单尝试预算与三层重试（refusalStatus=500）、
  `previousSpeechesFor` 的首场景特例、services 层的单一 `scene-actions` stage、
  步骤文件六段、package 层四类型分派、四份 prompt 模板、action-parser 七步解析管线、
  processActions 修补、buildCompleteScene 装配、`assertValidScene` → `validateAppScene`
  → DSL 的三层校验链、事件与前端、失败链、常量、坑与验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 5 册 场景内容](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md)｜下一步：[第 7 册 旁白与入库](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-7.md)
- 源文件：[steps/scene-actions.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts)
  · [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)
  · [packages/@openmaic/generation/src/scene-generator.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts)
  · [packages/@openmaic/generation/src/action-parser.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts)
  · [packages/@openmaic/generation/src/scene-builder.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `actions` / `scene:<index>:actions`（[plan.ts#L31-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L31-L33)） |
| 是否必经 | **必经**，每个场景三连步的**第 2 项**：`SCENE_STEP_KINDS = ['content', 'actions', 'narration']`（[plan.ts#L29](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L29)） |
| 一共跑几次 | 场景数 N 次（[plan.ts#L57-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L57-L65)），**严格按 index 顺序**，不参与 prewarm |
| 所处 run 状态 | `generating`（[plan.ts#L67-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L72)） |
| 检查点写在哪 | `generation_run_steps(run_id, 'scene:<n>:actions')`，值是 `SceneActionsResult = { scene, previousSpeeches }`（[scene-actions.ts#L54-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L54-L58)）——**整棵场景树都存在 run 的步骤表里** |
| 时间预算 | **60 s，按"每次尝试"计时**（[deadline.ts#L21-L22](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L21-L22) 的 `sceneActions`；`withDeadline` 在 `withRouteRetry` 内侧） |
| 重试 | **三层**同第 5 步：run 层指数退避（[retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts)）→ provider 状态码判定（[generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L158)）→ AI SDK 内部重试关闭（`maxRetries: 0`，[scene-actions.ts#L114](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L114)） |
| 重试上限 | 第 1 个场景 **2 次**、其余 **5 次**（[retry.ts#L21-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L21-L24)）；本步的 `refusalStatus` 是 **500**（[engine.ts#L742](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L742)），即步骤文件的拒绝**可重试** |
| 失败后果 | **直接 `paused`**——本步不在"软失败跳过"机制内。跳过（`skippedScenes`）只由 content 失败产生（[engine.ts#L577-L583](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L577-L583)），一旦产生，该场景的 actions 与 narration 一并视为完成、**根本不会执行** |
| UI 显示 | `actions` 位——**actions 与 narration 共用这一个位**（[preview-steps.ts#L42-L43](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L42-L43)） |
| 产出消费方 | 第 7 步 `narrateScene` 从本步检查点 `structuredClone(actions.scene)` 起步（[engine.ts#L978-L986](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L978-L986)）；`previousSpeeches` 被下一个场景的 actions 读走（[engine.ts#L760-L770](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L760-L770)） |
| 独有之处 | **唯一"装配 + 校验"的步**：`assertValidScene` 在 commit **之前**执行，坏场景重生成而不是带病进旁白；也是**唯一产出 `sceneId`** 的步（`buildCompleteScene` 里 `nanoid()`） |

### 0.2 一句话本质

> 第 6 步是**"给一屏东西配上台词与手势，并把它装订成书"**：它拿第 5 步的内容，按
> `outline.type` 分派到四套 prompt（slide / quiz / interactive / pbl），**只调用一次
> 大模型**，把返回的 JSON 数组经过**七步容错解析**变成 `Action[]`，再做一轮**确定性
> 修补**（补 id、救 spotlight、救 discussion 的 agentId），然后与内容一起装配成完整
> 场景，**先通过三层校验才允许落检查点**。它决定的是"这一屏怎么讲"，而第 7 步决定
> "这段话怎么念出来"。

### 0.3 时序图

```
advanceRun ─▶ step = scene:1:actions（scene:1:content 已 checkpoint）
                │ commit { state:'generating', step, step_started }
                ▼
   runStep(case 'actions')
                │  content = output('scene:1:content')!          ← 依赖第 5 步
                │  previousSpeeches = previousSpeechesFor(1)      ← 依赖 scene:0:actions
                ▼
      withRouteRetry( maxRetries=5, refusalStatus=500 )
        └─ withDeadline(60 s, signal)
             └─ services.sceneActions(owner, input, ctx)
                  └─ stageModel(owner, 'scene-actions')          ← 槽 course.actions
                       └─ steps/scene-actions.ts  ── aiCall ──► callLLM(maxRetries:0)
                            │
                            ├─ generateSceneActions(package)  按 outline.type 四选一
                            │    ├─ buildPrompt(<type>-actions)  缺模板 → default 生成器
                            │    ├─ aiCall(system, user)        ← 本步唯一一次模型调用
                            │    ├─ parseActionsFromStructuredOutput  七步解析
                            │    └─ processActions              补 id / 救 spotlight / 救 discussion
                            ├─ buildCompleteScene(...)          content+actions+stageId → scene
                            │    └─ null → StepRefusal('assembly-failed')  ← 500，可重试
                            └─ 提取 previousSpeeches（成品 scene 里的 speech.text）
                  ▼
             assertValidScene(generated.scene, stepId)   ← 在 commit 之前！
                └─ 不通过 → InvalidSceneError → GENERATION_FAILED → 重试（重生成）
                ▼
      done(result) ─▶ commit { step:{id:'scene:1:actions', output:scene+speeches},
                               events:[step_completed] }
                ▼
      下一步 = scene:1:narration（同一场景的第 3 项）
```

**注意三个反直觉之处**（本册会逐条展开）：

1. **本步没有 prewarm**。第 5 册里那个"提前并行生成后续场景内容"的泳道只预热
   `scene:*:content`（[engine.ts#L541-L576](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L541-L576)），
   因为 actions 依赖**前一个场景的 speech 成品**，无法提前算。
2. **首场景的 `previousSpeeches` 来自旁白之后**，不是来自 `scene:0:actions` 的检查点。
3. **run 管线里的 actions 永远不带视觉图**——步骤文件保留了视觉分支只为与浏览器路由一致。

---

## 1. 本步在计划中的位置

### 1.1 三连步的第二项

[plan.ts#L57-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L57-L65) 在大纲确认后把计划展开成：

```ts
export function generationSteps(sceneCount: number): RunStep[] {
  const steps: RunStep[] = [{ id: 'agents', kind: 'agents' }];
  for (let sceneIndex = 0; sceneIndex < sceneCount; sceneIndex += 1) {
    for (const kind of SCENE_STEP_KINDS) {
      steps.push({ id: sceneStepId(sceneIndex, kind), kind, sceneIndex });
    }
  }
  return steps;
}
```

所以一个 3 场景课程的步骤序列是：

```
agents
scene:0:content → scene:0:actions → scene:0:narration
scene:1:content → scene:1:actions → scene:1:narration
scene:2:content → scene:2:actions → scene:2:narration
```

`advanceRun`（[plan.ts#L98-L102](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L98-L102)）取**第一个不在 `completed` 集里**的步骤，`completed` 由"已 checkpoint 的步骤 id"与 `skippedStepIds()` 并集而成（[engine.ts#L1767-L1772](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1767-L1772)）。这意味着：

| 情况 | 结果 |
|---|---|
| `scene:1:content` 未 checkpoint | `scene:1:actions` 永远轮不到（顺序在前） |
| `scene:1` 因 content 失败被跳过 | `skippedStepIds()` 把三连步一起算完成，actions **不执行**，`output('scene:1:content')` 也不会被读 |
| 接管（换进程续跑） | actions 从步骤表读出 `scene:1:content` 与 `scene:0:actions` 就能继续，无需重算 |

### 1.2 为什么 actions 不能提前算

`generateSceneActions` 的输入里有两样**跨场景**的东西：

- `allOutlines`：为了算 `pageIndex / totalPages / allTitles`，即"我在第几页"；
- `previousSpeeches`：上一页最后一段话，为了衔接。

两者都只依赖**大纲**和**前面场景的产物**。后者使 actions 必须在 `scene:index-1` 的
actions（或 index-1 === 0 时其 narration）之后才能定稿，因此第 5 册的并行泳道把
actions 留在主循环里按序消费。并行模式下你会看到 `scene:3:content` 的
`step_completed` 早于 `scene:2:actions` 到达——**这是正常现象**，不是 bug。

### 1.3 与第 5 步的边界

| | 第 5 步 content | 第 6 步 actions |
|---|---|---|
| 决定 | 这一屏**有什么**（元素/题目/HTML/项目） | 这一屏**怎么讲**（说话与聚焦的顺序） |
| 输出 | `SceneContent` + `effectiveOutline` | `CompleteScene`（含 id / order / stageId / actions） |
| 是否产生场景 id | 否 | **是**（[scene-builder.ts#L33](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L33)） |
| 是否被跳过机制保护 | 是（软失败） | 否（硬失败 → paused） |
| 是否预热 | 是 | **否** |
| 模型调用次数 | 1（内容）+ 可能的视觉图解析 | 1 |
| 校验 | 由 `parseContent` 系列与 fallback 兜 | `assertValidScene` **整棵场景** |

---

## 2. `case 'actions'`：输入装配

完整分支只有 34 行，但它是本册的"总谱"（[engine.ts#L712-L745](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L712-L745)）：

```ts
case 'actions': {
  const index = step.sceneIndex;
  const { outlines, languageDirective } = outline();
  const content = output<SceneContentResult>(sceneStepId(index, 'content'))!;
  const userProfile = learnerProfileText(input);
  const result = await withRouteRetry(
    () =>
      withDeadline(stepId, STEP_DEADLINES_MS.sceneActions, signal, async (callSignal) => {
        const generated = await services.sceneActions(
          owner,
          {
            outline: content.effectiveOutline || outlines[index]!,
            allOutlines: outlines,
            // The route receives the content as JSON; so does this step.
            content: content.content as SceneActionsInput['content'],
            stageId: agents().stage.id,
            agents: agents().agents.agents,
            previousSpeeches: previousSpeechesFor(index),
            ...(userProfile ? { userProfile } : {}),
            languageDirective,
          },
          { log, signal: callSignal },
        );
        // The complete scene must be one the course can hold before it is
        // checkpointed: an invalid one (an action the model left without a
        // type, say) fails this step, which regenerates it, rather than
        // reaching narration and failing there for good.
        assertValidScene(generated.scene, stepId);
        return generated;
      }),
    retryOptions(stepId, index, 500),
  );
  return done(result);
}
```

### 2.1 八个输入字段逐个说明

| 字段 | 来源 | 说明 |
|---|---|---|
| `outline` | `content.effectiveOutline ?? outlines[index]` | **优先用第 5 步可能改过的那条大纲**：fallback 换类型、剥掉无法解析的图片都记录在 `effectiveOutline` 上。用原始大纲会让 actions 与内容"讲的不是同一屏" |
| `allOutlines` | `outline().outlines` | 全部大纲（确认后定稿），只用来算页码与标题列表 |
| `content` | 第 5 步检查点 | 注释原文："The route receives the content as JSON; so does this step."——run 步骤与 1.1.x 的 `POST /api/generate/scene-actions` 传的是同一份 JSON |
| `stageId` | `agents().stage.id` | 第 4 步已铸造的 stage id；装配时写进场景。场景 id 反而**不是**外部传入的 |
| `agents` | `agents().agents.agents` | 第 4 步画像产出的班级智能体列表，进入 prompt 的 `Classroom Agents`，也是 `discussion.agentId` 的合法域 |
| `previousSpeeches` | `previousSpeechesFor(index)` | 见 §4 |
| `userProfile` | `learnerProfileText(input)` | 条件展开：**没有学习者画像时这个键根本不出现**，于是 prompt 里 `{{userProfile}}` 保留占位符原样（见 §8.4） |
| `languageDirective` | `outline().languageDirective` | 第 3 步语言判定产物，强制讲课语言 |

> `stageModel` 由 services 层补上（§5），`model` 是 `SceneActionsInput` 的必填第 9 项
> （[scene-actions.ts#L46-L47](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L46-L47)）。

### 2.2 那个 `!` 为什么是安全的

`output<SceneContentResult>(sceneStepId(index, 'content'))!` 是**非空断言**。它成立的
前提是 §1.1 的顺序保证：`advanceRun` 只会把 actions 排在其 content 之后。三条支撑：

1. 正常推进：content 步骤返回 `done(...)`，同一次 commit 写入检查点，`steps` Map 更新；
2. 并行 prewarm：`startContentPrewarm` 在提交预热结果时也写 `steps`（[engine.ts#L555-L560](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L555-L560)）；
3. 跳过场景：整条三连步被 `skippedStepIds()` 视为完成，actions 不执行。

因此**唯一的崩溃可能是计划函数被改坏**，不是运行时状态问题——这也是文档把它列为
"不变量"而非"错误处理缺失"的原因。

---

## 3. 三层重试与 60 秒预算

### 3.1 结构与第 5 步相同，参数不同

| 参数 | content | actions |
|---|---|---|
| `STEP_DEADLINES_MS` | `sceneContent: 300_000` | **`sceneActions: 60_000`** |
| `refusalStatus` | 500 | **500** |
| `maxRetries` | 首场景 2 / 其余 5 | 同 |
| `retryOptions` 调用 | `retryOptions(stepId, index, 500)` | 同 |

deadline 的注释说明了 60 s 从何而来（[deadline.ts#L1-L7](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L1-L7)）：*"the budget the step's 1.1.x API route declared"*——这是旧版 `POST /api/generate/scene-actions` 路由自己的超时，run 只是把它搬到进程内。动作生成只需要一次模型调用且输出比内容短得多（5–10 个 JSON 对象 vs 整屏元素），所以 60 s 够用。

超时会抛 `StepTimeoutError`，`statusCode = 504`（[deadline.ts#L27-L35](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L27-L35)）：54 落在 `>= 500` 分支 → **可重试**。`withDeadline` 用 `AbortSignal.any([runSignal, timeout.signal])` 合成信号，因此 run 被夺锁 / 课程被删 / 进程关闭时**优先表现为 abort**，不会被误记为超时（[deadline.ts#L43-L68](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L43-L68)）。

### 3.2 退避公式

`withRouteRetry` → `withGenerationRetry`（[retry.ts#L71-L108](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L71-L108)），第 n 次重试前睡眠：

```
min(16000, 1000 × 2^(n-1)) + floor(该项 × random() × 0.2)
```

即约 1 s、2 s、4 s、8 s、16 s（上限 16 s，抖动最多 20%）
（[generation-retry.ts#L167-L174](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L167-L174)，`DEFAULT_MAX_DELAY_MS = 16000` 在 [L24](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L24)）。
首场景最多 2 次，因此最坏耗时约为 `60×3 + 1 + 2 ≈ 183 s`；后续场景最多 5 次，最坏约 `60×6 + 31 ≈ 391 s`。

### 3.3 哪些失败会被重试

判定顺序（[generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L158)）：

| 失败 | 映射状态 | 是否重试 |
|---|---|---|
| `StepRefusal('assembly-failed')` | `refusalStatus` = **500** | ✅ 重试（装配失败值得再来一次） |
| `InvalidSceneError`（§12） | 无 `httpStatus` → `upstreamHttpStatus` 兜底 500 | ✅ 重试（模型再抽一次可能就好了） |
| provider 429 / 408 / 409 / 425 | 原状态 | ✅ |
| provider 5xx | 原状态 | ✅ |
| provider 400 / 401 / 403 / 404 / 422 | 原状态 | ❌ 立刻暂停 |
| `StepTimeoutError` | 504 | ✅ |
| 明确的配额拒绝 `isProviderQuotaRefusal` | — | ❌（[retry.ts#L91-L93](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L91-L93) 强制 `isRetryable: false`） |
| `ModelAuthorizationError` / 主机自判失败 | `classifyHostFailure` | 由 host 的 `retryable` 决定 |
| abort | — | ❌ 直接向上抛（不算失败，算中断） |

### 3.4 `onRetry` 会发什么事件

`retryOptions` 的 `onRetry`（[engine.ts#L434-L448](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L434-L448)）每次重试都：

1. `log.warn`：`run <id>: retrying scene:1:actions (2/6): InvalidSceneError: ...`；
2. commit 一个 **`step_retry`** 事件，字段 `step / attempt / maxAttempts / reason / cause`。

`cause` 是被重试的那次失败原文（`Name: message`），前端时间线显示它。**注意 `step_retry`
不改变 run.step**，所以预览页一直停在 `actions` 位上。

### 3.5 与 content 的软失败对比（重要）

第 5 步在并行模式下失败 → 该场景进 `skippedScenes`，其余场景继续。
**本步失败没有任何这种缓冲**：`runStep` 抛出后被主循环的 `try` 包装成 `StepFailedError`
（[engine.ts#L1826-L1839](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1826-L1839)），
最终 `pause(stepId, message, code)`（[engine.ts#L1656-L1673](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1656-L1673)）——
它会 abort prewarm 泳道、停媒体泳道，然后写 `state:'paused'` + `step_failed` + `state` 两个事件。

为什么这样设计：内容可以缺屏（课程还能讲完），**装配失败的场景无法进入课堂**，
没有"少一屏"的余地——少一屏会让 order 出现空洞、旁白链断在一处。

---

## 4. `previousSpeechesFor`：跨场景的旁白接续

### 4.1 定义（[engine.ts#L754-L770](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L754-L770)）

```ts
/**
 * The speeches of the scene before, as the browser threads them: none for
 * the first scene; the stored first scene's (after narration split its long
 * lines) for the scene the classroom starts with; else the actions result of
 * the last scene generated before this one.
 */
const previousSpeechesFor = (index: number): string[] => {
  for (let before = index - 1; before >= 0; before -= 1) {
    if (before === 0) {
      const first = output<NarrationOutput>(sceneStepId(0, 'narration'));
      return first ? speechesOf(first.scene) : [];
    }
    const actions = output<SceneActionsResult>(sceneStepId(before, 'actions'));
    if (actions) return actions.previousSpeeches ?? [];
  }
  return [];
};
```

以及 `speechesOf`（[engine.ts#L234-L238](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L234-L238)）：

```ts
function speechesOf(scene: Scene): string[] {
  return (scene.actions ?? [])
    .filter((action): action is SpeechAction => action.type === 'speech')
    .map((action) => action.text);
}
```

### 4.2 三个分支的实际行为

| 当前 index | 倒扫过程 | 取到的东西 |
|---|---|---|
| `0` | 循环体一次都不执行 | `[]` —— 首页没有前情，prompt 里不会出现 "Previous page speech" |
| `1` | `before = 0` 命中特例 | **第 0 场景 narration 之后**的场景里的 speech；若还没旁白（`scene:0:narration` 无检查点）→ `[]` |
| `2` | `before = 1`：读 `scene:1:actions` | 有 → 它的 `previousSpeeches`（本场景装配时的 speech，未拆分）；无 → 继续倒扫到 `before = 0` 走特例 |
| `5`（scene:4 被跳过） | `before = 4` 无检查点 → 继续 | 一路回退到最近一个真正跑过 actions 的场景 |

### 4.3 首场景为什么要特取"旁白之后"

第 7 步会把过长的 `speech` **拆分**成多个 clip（`splitLongSpeechActions`，
[engine.ts#L986](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L986)）。
首场景是课堂的开场，它的**实际播出文本**就是拆分后的版本；第 2 页衔接时应引用真实说过的
话，而不是拆分前的长句。其余场景不做这个处理，是因为**它们的 narration 还没跑**——
在串行顺序里 `scene:n:actions` 之前只有 `scene:n-1:actions` 一定已完成，
`scene:n-1:narration` 不一定。

### 4.4 这个数组如何进入 prompt

步骤文件把 `previousSpeeches` 放进 `generationCtx`（[scene-actions.ts#L127-L132](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L127-L132)），
package 层再用 `buildCourseContext` 渲染成文本，**只取最后一条的末尾 150 字**
（[prompt-formatters.ts#L42-L47](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompt-formatters.ts#L42-L47)）：

```ts
if (ctx.previousSpeeches.length > 0) {
  lines.push('Previous page speech (for transition reference):');
  const lastSpeech = ctx.previousSpeeches[ctx.previousSpeeches.length - 1];
  lines.push(`  "...${lastSpeech.slice(-150)}"`);
}
```

所以传多个是为了**取最后一条**，同时保留"上一页真的说过话"这个信号；首页与"上一页失败被跳过"
都会退化成空数组，模型于是收到"无衔接可参考"。

---

## 5. services 层：单一 stage，无类型细分

### 5.1 全部实现（[services.ts#L484-L489](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L484-L489)）

```ts
async sceneActions(ownerId, input, ctx) {
  return generateSceneActions(
    { ...input, model: await stageModel(ownerId, 'scene-actions') },
    ctx,
  );
},
```

对照第 5 步的 `sceneContent`（[services.ts#L460-L482](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L460-L482)），差别有两处：

| | content | actions |
|---|---|---|
| stage 选择 | **按类型**取 `scene-content:<type>`，取不到回落 `'scene-content'` | **固定** `'scene-actions'` |
| 素材 | 需要把 `imageMapping` 换成存好的 asset（`storedImageMapping`） | 不需要 |
| ctx | 注入 `resolveVisionImages` | 原样透传 `{ log, signal }` |

**这是本册第一个"与直觉不同"的点**：内容生成给七种内容形态留了七个可选槽位
（[第 5 册 §5](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md)），
动作生成只占**一个**槽。原因在 `LLM_STAGES` 的注册表里看得最清楚——见下。

### 5.2 槽位与标签

| 层 | 值 | 位置 |
|---|---|---|
| stage id | `'scene-actions'` | [lib/constants/llm-stages.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/llm-stages.ts) |
| 模型槽 | `course.actions` | [lib/config/model-slots.ts#L102](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L102) |
| 路由表 | `course.actions` → stage `scene-actions` | [lib/config/model-routes.ts#L134](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-routes.ts#L134) |
| 测试钉桩 | `expect(...).toContain('scene-actions')` | [tests/config/model-slots.test.ts#L27](file:///D:/GITHUB_chat/OpenMAIC/tests/config/model-slots.test.ts#L27) |

`stageModel(ownerId, 'scene-actions')` 走的是通用解析：owner 的显式覆盖 → 槽位配置 →
平台默认。若槽被禁用或未分配，抛出的错误由 `classifyHostFailure` 归类，可能带
`errorCode` 且 `retryable: false` —— **此时 5 次重试也不会发生，直接暂停**。

### 5.3 调用侧的模型元数据

步骤文件解构 `input.model`（[scene-actions.ts#L65-L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L65-L71)）：

| 字段 | 用途 |
|---|---|
| `model` → `languageModel` | 真正传给 `callLLM` 的 SDK 实例 |
| `modelInfo` | `capabilities.vision`（§6.2）、`outputWindow`（作为 `maxOutputTokens`） |
| `modelString` | 日志里打印 `[model=...]` |
| `thinkingConfig` | 思考模式配置，原样透传给 `callLLM` |
| `serverManaged` | 传给 `callLLM` 的最后一个参数，标记"服务端计费/托管" |

---

## 6. 步骤文件六段（steps/scene-actions.ts 全文走读）

文件 175 行，头注释已经把身份说清了（[L1-L7](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L1-L7)）：

```ts
/**
 * Scene actions: generate the playback actions (speech, spotlights, ...) for
 * one scene from its outline and content, with the course context (the
 * scene's place among all outlines and the speeches of the scenes before it),
 * and assemble the complete scene. The logic of 1.1.x's
 * POST /api/generate/scene-actions.
 */
```

### 6.1 第一段：类型契约（[L30-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L30-L58)）

```ts
export interface SceneActionsInput {
  outline: SceneOutline;
  /** Every outline of the course, for the scene's page index and the titles around it. */
  allOutlines: SceneOutline[];
  content: GeneratedSlideContent | GeneratedQuizContent
         | GeneratedInteractiveContent | GeneratedPBLContent | PBLContent;
  stageId: string;
  agents?: AgentInfo[];
  /** What the scenes before this one said, for continuity. */
  previousSpeeches?: string[];
  userProfile?: string;
  languageDirective?: string;
  /** The scene-actions stage's model. */
  model: StepLanguageModel;
}

export type SceneActionsRefusal = 'assembly-failed';
export type GeneratedScene = NonNullable<ReturnType<typeof buildCompleteScene>>;
export interface SceneActionsResult {
  scene: GeneratedScene;
  /** This scene's speeches, for the scenes after it. */
  previousSpeeches: string[];
}
```

三个值得留意的声明方式：

- `content` 联合里**多出一个 `PBLContent`**（已存储形态），而 `GeneratedPBLContent` 是生成形态——这是 §6.4 遗留数据升级的原因；
- `GeneratedScene` **不自己定义类型**，而是"取 `buildCompleteScene` 返回值的非空版"。装配函数的形状就是场景的形状，二者不可能漂移；
- `SceneActionsRefusal` 只有一个成员，写进 `StepRefusal<T>` 的泛型，使拒绝码可枚举。

### 6.2 第二段：`aiCall` 双形态（[L74-L122](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L74-L122)）

```ts
// Detect vision capability
const hasVision = !!modelInfo?.capabilities?.vision;

// AI call function (actions typically don't use vision, but kept for consistency)
const aiCall = async (systemPrompt, userPrompt, images?) => {
  if (images?.length && hasVision) { /* 多模态形态：prepareVisionImages + buildVisionUserContent */ }
  /* 纯文本形态：prompt: userPrompt */
};
```

注释写得很克制：*"typically don't use vision, but kept for consistency"*。**实际情况更强**：
package 层四处调用 `aiCall(prompts.system, prompts.user)` **从不传第三个参数**
（见 §7），所以在 run 管线里**视觉分支永远不会触发**。它存在的唯一理由是这段代码
"1.1.x `POST /api/generate/scene-actions` 的逻辑"——浏览器直接调用该路由时可以带上图片。

两个形态共同点：

| 选项 | 值 | 意义 |
|---|---|---|
| `maxRetries` | **0** | 关掉 AI SDK 内部重试，把重试权完全交给 run 层的 `withRouteRetry`（否则预算会相乘） |
| `maxOutputTokens` | `modelInfo?.outputWindow` | 用槽位声明的输出窗口，不写死 |
| `abortSignal` | `ctx.signal` | 夺锁 / 删课 / 关闭时立刻取消在途请求 |
| stage 标签 | `'scene-actions'` | `callLLM` 第二参数，用于计费与日志归类 |

视觉分支还多一步 `prepareVisionImages(images, log)`（[L84](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L84)），
它把无法解码的图丢掉、限制张数与体积——第 5 册 §10 详述过；本册只需知道它在此路径上
**从不执行**。有一条单测专门钉这个分支的行为（见 §14.5）。

### 6.3 第三段：跨场景上下文（[L124-L132](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L124-L132)）

```ts
const allTitles = allOutlines.map((o) => o.title);
const pageIndex = allOutlines.findIndex((o) => o.id === outline.id);
const generationCtx: SceneGenerationContext = {
  pageIndex: (pageIndex >= 0 ? pageIndex : 0) + 1,
  totalPages: allOutlines.length,
  allTitles,
  previousSpeeches: input.previousSpeeches ?? [],
};
```

**`pageIndex` 按 id 查找而不是用引擎传下来的 index**，且找不到时退化为 1。
为什么可能找不到：`outline` 传入的可能是 `content.effectiveOutline`，它是第 5 步
fallback 时**克隆改过**的对象——id 通常保留，所以正常都能命中；但一旦某个 fallback
造了新 id，本步不会崩，只会把这一屏当成首页（表现为"多了一句开场问候"）。

`+1` 是因为 prompt 里给人看的是"Page 2 of 5"这种 1 基编号，而 `buildCourseContext`
标记当前页时又用 `i === ctx.pageIndex - 1` 换回 0 基（[prompt-formatters.ts#L17](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompt-formatters.ts#L17)）。

### 6.4 第四段：PBL 遗留内容升级（[L137-L143](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L137-L143)）

```ts
const generationContent = (
  'type' in content && content.type === 'pbl' ? normalizeLegacyPBLContent(content) : content
) as GeneratedSlideContent | GeneratedQuizContent
   | GeneratedInteractiveContent | GeneratedPBLContent;
```

`normalizeLegacyPBLContent`（[lib/pbl/legacy/read.ts#L225-L237](file:///D:/GITHUB_chat/OpenMAIC/lib/pbl/legacy/read.ts#L225-L237)）把只有
`projectConfig`（v1）的 PBL 内容升级成带 `projectV2` 的形状。**这一步必须在装配前**，
因为 `buildCompleteScene` 的 pbl 分支判据是 `'projectV2' in content`（§11.4）。
注意 `generationContent` 之后**同时**用于生成动作和装配场景，因此模型看到的与写进文档的
是同一份升级后内容。

### 6.5 第五段：生成 + 装配 + 拒绝（[L145-L163](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L145-L163)）

```ts
const actions = await generateActions(outline, generationContent, aiCall, {
  ctx: generationCtx, agents, userProfile, languageDirective,
});
log.info(`Generated ${actions.length} actions for: "${outline.title}"`);

const scene = buildCompleteScene(outline, generationContent, actions, stageId);

if (!scene) {
  log.error(`Failed to build scene: "${outline.title}"`);
  throw new StepRefusal<SceneActionsRefusal>(
    'assembly-failed',
    `Failed to build scene: ${outline.title}`,
  );
}
```

`buildCompleteScene` 的第四个位置参数就是 `stageId`，第五个 `options`（`{ sceneId }`）**这里没传**——
所以每次尝试都会 `nanoid()` 造**新的**场景 id。含义见 §11.1 的坑。

日志两条都是 `info`：`Generating actions: "标题" (type) [model=...]`（[L135](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L135)）
与 `Scene assembled successfully: ... — N actions`（[L170-L172](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L170-L172)）。
排查"动作数不对"时先看这两行。

### 6.6 第六段：交接给下一个场景（[L165-L174](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L165-L174)）

```ts
const previousSpeeches = (scene.actions || [])
  .filter((a): a is SpeechAction => a.type === 'speech')
  .map((a) => a.text);

return { scene, previousSpeeches };
```

**从成品 `scene.actions` 提取，而不是从 `generateActions` 的返回值**：这样若将来在装配中
改动了动作数组（今天没有），交接出去的仍是场景真正会说的内容。与 `speechesOf`（§4.1）
逻辑相同，只是一个在步骤文件里、一个在引擎里。

### 6.7 六段一览

| 段 | 行 | 产物 | 失败模式 |
|---|---|---|---|
| 1 契约 | 30–58 | 类型 | 编译期 |
| 2 aiCall | 74–122 | 一次调用闭包 | provider 状态 → §3.3 |
| 3 上下文 | 124–132 | `generationCtx` | 无（退化） |
| 4 兼容 | 137–143 | `generationContent` | 无 |
| 5 生成+装配 | 145–163 | `scene` | `StepRefusal('assembly-failed')` → 500 → 重试 |
| 6 交接 | 165–174 | `previousSpeeches` | 无 |

---

## 7. package 层：`generateSceneActions` 的四类型分派

[scene-generator.ts#L1771-L1924](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1771-L1924)。

```ts
export async function generateSceneActions(
  outline: SceneOutline,
  content: GeneratedSlideContent | GeneratedQuizContent
           | GeneratedInteractiveContent | GeneratedPBLContent,
  aiCall: AICallFn,                          // ← 第三个**位置参数**，不在 options 里
  options: SceneActionsOptions = {},
): Promise<Action[]>
```

`aiCall` 由调用方注入，package 只负责 prompt 组装与结果加工。`SceneActionsOptions`
只有五个键（[L126-L132](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L126-L132)）：
`ctx`、`agents`、`userProfile`、`languageDirective`、`logger`——**全部可选**，
其中 `logger` 缺省为 `noopGenerationLogger`（第 6 节会看到 run 管线并没有传它）。

### 7.1 四个分支的共同骨架

每个分支都是同一段五步曲（slide 分支原文，[L1793-L1821](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1793-L1821)）：

```ts
if (outline.type === 'slide' && 'elements' in content) {
  const elementsText = formatElementsForPrompt(content.elements);
  const prompts = buildPrompt(PROMPT_IDS.SLIDE_ACTIONS, { /* 8 个变量 */ });
  if (!prompts) return generateDefaultSlideActions(outline, content.elements);  // 模板缺失 → 兜底
  const response = await aiCall(prompts.system, prompts.user);                  // ← 唯一一次模型调用
  const actions = parseActionsFromStructuredOutput(response, outline.type, undefined, log);
  if (actions.length > 0) return processActions(actions, content.elements, agents, log);  // ← 修补
  return generateDefaultSlideActions(outline, content.elements);                // 解析为空 → 兜底
}
```

三个出口（模板 null / 解析空 / 类型与内容形状不符）都落到**同类型的 default 生成器**，
因此**本步几乎不可能返回空数组**——空数组只在函数末尾 `return [];`（[L1923](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1923)）
出现，即 `outline.type` 与 `content` 形状四种组合一个都不匹配时。

| 类型 | 分支行 | 判据 | `buildPrompt` 独有变量 | `parse` 白名单 | `processActions` 的 elements | default 定义 |
|---|---|---|---|---|---|---|
| `slide` | [L1793-L1821](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1793-L1821) | `type==='slide' && 'elements' in content` | `elements`、`userProfile` | `undefined` | **真实** `content.elements` | [L2039](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2039) |
| `quiz` | [L1823-L1849](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1823-L1849) | `type==='quiz' && 'questions' in content` | `questions` | `undefined` | `[]` | [L2070](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2070) |
| `interactive` | [L1851-L1891](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1851-L1891) | `type==='interactive' && 'html' in content` | `elementInventory`、`widgetType`、`widgetConfig`、`conceptName`、`designIdea` | **`INTERACTIVE_WIDGET_ACTIONS`** | `[]` | [L2084](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2084) |
| `pbl` | [L1893-L1921](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1893-L1921) | **只看 `type==='pbl'`** | `projectTopic`、`projectDescription`、`projectSummary` | `undefined` | `[]` | [L1929](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1929) |

**判据是"大纲类型 AND 内容形状"两件事**，这一点比看起来重要：只要第 5 步交回的
内容形状与大纲类型不符（例如 fallback 把 quiz 的内容包成了 slide 形状），本分支
就整体不进，一路掉到 `return []`，再被装配层判成 `assembly-failed`。**类型不匹配
在本步不报错，而是表现为"装配失败"**。

`userProfile` 只在 slide 分支出现是**模板差异**而非设计偏好：quiz / interactive / pbl
的 `user.md` 里**没有** `{{userProfile}}` 占位符（§8.2），传了也不会渲染。

### 7.2 interactive 类型只有一种 outline.type

`simulation` / `diagram` / `code` / `game` / `visualization3d` / `procedural-skill`
这些**不是** `outline.type`，而是 `content.widgetType`（大纲上是
`outline.widgetType` 兜底，[L1866](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1866)）。
它们的 `outline.type` 统统是 `interactive`，因此**共享同一份
`interactive-actions` 模板**——这就是"一个 prompt 教模型操作六种 widget"的原因，
也是为什么模板里必须让它从 Element Inventory 取 selector（§8.4）。

分支开头还留了一条**专门给排障用的日志**（[L1786-L1791](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1786-L1791)）：

```
[Actions Gen] Interactive "标题": hasHtml=true, widgetType=simulation
```

> 但请注意 §14.3：这条 log 在 run 管线里**当前不会输出**（package 层的 logger 没被传入）。

### 7.3 interactive 的 Element Inventory：每次现算

[L1854-L1859](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1854-L1859) 有本册最重要的一条注释：

```ts
// Always recompute the inventory from the current html so it matches what
// the tool actually reads — persisting the field would go stale relative
// to `postProcessInteractiveHtml` output and to any in-turn html edits.
const inventory =
  (content.html ? extractInteractiveElements(content.html) : '') ||
  '(no interactive elements detected)';
```

即：**清单是从手里这份 HTML 现场解析的**，不读第 5 步存下来的任何字段。原因写在
注释里——第 5 步之后还有 `postProcessInteractiveHtml`（注入库、改写 URL、补 shim），
场景里最终存的 HTML 与生成时的中间态不同；另外用户在回合内改过 HTML 也会让持久化
字段过期。若用过期清单，模型会指向 post-process 之后已不存在的 selector，
`widget_highlight` 打空。

空 HTML 或解析不出元素时，传给 prompt 的是那句
**`'(no interactive elements detected)'`**——模板里"清单非空才优先"的条件于是
自动落到"按命名惯例猜"这一支（§8.4），这是**故意把退化状态告诉模型**的写法。

`extractInteractiveElements`（[L1452 起](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1452)）
从 HTML 里抓 id / class / data-* 三类选择器，各有上限（id 60、class 30、data 30），
防止清单本身把 prompt 撑爆。

### 7.4 interactive 白名单

[L70-L75](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L70-L75)：

```ts
const INTERACTIVE_WIDGET_ACTIONS = [
  'widget_highlight',
  'widget_setState',
  'widget_annotation',
  'widget_reveal',
];
```

作为第三个实参传进解析器（[L1879-L1884](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1879-L1884)），
于是模型即便"好心"输出 `spotlight`，也会在 §9 的第 7 步被丢掉
（`speech` 例外，永远放行）。这是本步**三层防御**中的一层，见 §9.5。

### 7.5 pbl 的 projectSummary

`buildPBLProjectSummary(outline, projectV2)`（[L1729-L1766](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1729-L1766)）
把里程碑按 `order` 排序后压成文本喂给模型，让开场白说得出"我们要先做 A，再做 B"。
注意它取的是 `(content as Partial<GeneratedPBLContent>).projectV2`（[L1896](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1896)）——
pbl 分支**不检查内容形状**，所以若第 6.4 节的遗留升级没跑（或内容既无 v1 也无 v2），
`projectV2` 是 `undefined`，summary 退化成"只有大纲"的一段话；随后
`buildCompleteScene` 因 `'projectV2' in content` 不成立而返回 null，本步以
`assembly-failed` 重试。`PROMPT_VARIABLE_DEFAULTS['pbl-actions'].projectSummary`
（[loader.ts#L16-L21](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L16-L21)）
提供的是**明确禁止编造**的占位文案：

```
'(No generated milestones are available; introduce the project topic without inventing any.)'
```


---

## 8. 四份动作模板与注入的上下文

### 8.1 模板机制（先记住这条链）

[loader.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts)：

| 步骤 | 函数 | 行为 |
|---|---|---|
| 定位 | `DEFAULT_PROMPTS_DIR`（[L14](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L14)） | 从本模块 URL 上溯两级 = **包根**，于是模板在 `templates/<promptId>/system.md`、片段在 `snippets/<id>.md`；用路径运算而非 import 资源，避免 app 打包器把 Markdown 当模块 |
| 读取 | `loadPrompt`（[L68-L103](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L68-L103)） | `system.md` **缺失 → 返回 null**（ENOENT 才 null，其他 IO 错误照抛）；`user.md` 是**可选**的，缺失就是空串 |
| 片段 | `processSnippets`（[L45-L52](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L45-L52)） | `{{snippet:name}}` → 片段文件内容；**片段缺失直接抛错**（"Fail loud rather than silently shipping `{{snippet:foo}}` to the model"，[L39](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L39)） |
| 条件块 | `processConditionalBlocks`（[L55-L65](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L55-L65)） | `{{#if x}}...{{/if}}` 按真假**整块保留或删除** |
| 变量 | `interpolateVariables`（[L106-L114](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L106-L114)） | `{{name}}`；对象 → `JSON.stringify(v, null, 2)`；**`undefined` → 原样保留占位符** |

`PromptId` 联合里四个动作模板并列在内容模板之后
（[types.ts#L16-L19](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/types.ts#L16-L19)）。

> **机制性的隐患**：`undefined` 保留占位符意味着——调用方漏传一个变量，模型会收到
> 字面的 `{{elementInventory}}`。它不报错，只是让模型表现变差且极难归因。四份动作模板
> 目前**没有**这种漏传：`userProfile` / `languageDirective` 在 slide 分支恒被传入，
> 值为空时用 `|| ''` 兜成空串（[L1798-L1806](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1798-L1806)），
> quiz/interactive/pbl 分支干脆不传这两个键，而它们的模板也**不引用**；
> `pbl-actions` 的 `projectSummary` 甚至有两层网：`buildPBLProjectSummary` 在里程碑为
> 空时自己就返回占位文案（[scene-generator.ts#L1762](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1762)），
> 而模板层还有全项目唯一一条 `PROMPT_VARIABLE_DEFAULTS`
> （[loader.ts#L16-L21](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompts/loader.ts#L16-L21)，
> 仅在变量为 `undefined` 时生效，因此对 run 管线来说是**永远用不到的第二层保险**）。
> 换句话说：这套模板系统把"补默认值"的责任放在**每个调用点**，而不是渲染器里；
> 新增变量时若忘了改调用点，就会静默退化。另一处有意的"不替换"是 kebab-case 名字——
> 变量正则只认 `\w+`，所以 `{{some-name}}` 永远原样保留，用作可读性标记。

### 8.2 四份 `user.md` 的变量对照

| | slide | quiz | interactive | pbl |
|---|---|---|---|---|
| `{{elements}}` | ✅ | — | — | — |
| `{{questions}}` | — | ✅ | — | — |
| `{{elementInventory}}` | — | — | ✅ | — |
| `{{widgetType}}` / `{{widgetConfig}}` | — | — | ✅ | — |
| `{{projectTopic}}` / `{{projectDescription}}` / `{{projectSummary}}` | — | — | — | ✅（summary 由 system.md 使用） |
| `{{title}}` / `{{keyPoints}}` / `{{description}}` | ✅ | ✅ | ✅ | ✅ |
| `{{conceptName}}` / `{{designIdea}}` | — | — | ✅ | — |
| `{{courseContext}}` | ✅ | ✅ | ✅ | ✅ |
| `{{agents}}` | ✅ | ✅ | ✅ | ✅ |
| `{{userProfile}}` | ✅（恒传，空时为 `''`） | — | — | — |
| `{{languageDirective}}` | ✅（恒传，空时为 `''`） | ✅ | ✅ | ✅ |

四份模板末尾都写了**"Output as a JSON array directly (no explanation, no code
fences, …)"** 并附一行示例 JSON——但"不要代码围栏"这个要求仍然常被违反，
所以解析器第一件事就是剥围栏（§9.1）。

要求的动作条数也各不同：**slide 5–10**（[slide-actions/user.md#L11](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/slide-actions/user.md#L11)）、
**quiz 1–2**（[quiz-actions/user.md#L10](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/quiz-actions/user.md#L10)，
且明写 "text only"）、**interactive 3–8**（[interactive-actions/user.md#L17](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/interactive-actions/user.md#L17)）、
**pbl "Typically just 1-2 speech segments"**（这条只出现在 system.md
[L53](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/pbl-actions/system.md#L53)，
user.md 未写数字）。这些数字同时是"输出长度预算"，与 60 s deadline 相匹配。

### 8.3 `buildCourseContext`：同一次课，不是两次

[prompt-formatters.ts#L9-L50](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompt-formatters.ts#L9-L50)
把 `generationCtx` 渲染成这段（**注意全大写强调词是原文**）：

```
## Course Context
This course contains 5 pages:
  1. 课程导入与学习目标
  2. 牛顿第二定律 ← current
  ...

All pages belong to the SAME class session with the SAME students.
NEVER say "last class", "previous session", "welcome back", or imply students studied prior material outside this course.
You may reference earlier pages in THIS course as "我们刚才讲了..." and later pages as "后面还会学到...".

This is the FIRST page. Open with a greeting and introduce the course.
Previous page speech (for transition reference — do NOT repeat it):
  "...所以我们把公式整理成 a = F/m。"
```

四条规则各自的动机：

| 规则 | 防的是哪种真实故障 |
|---|---|
| "SAME class session" + NEVER 列表 | 模型把每页当独立一节课，于是每一页都说"上节课我们学了…"，学生听到五遍开场白 |
| 允许引用本课页（"我们刚才讲了"） | 完全禁止回指会让衔接生硬；只禁**课与课之间**的虚构连续性 |
| `FIRST page` / `FINAL page` / 中间页三种话术 | 首页要开场+介绍课程，末页要总结+预告下次，中间页只需 1–2 句承上启下 |
| "do NOT repeat it" | 抄前情——模型会把上一段话原样再说一遍 |

页码是 **1 基**，`← current` 用 `i === ctx.pageIndex - 1` 标出（[L17](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompt-formatters.ts#L17)）。
**首页判断是 `pageIndex === 1`**，因此 §6.3 里"`findIndex` 找不到退化为 1"会被模型
理解成"这是开场页"——表现是莫名多了一句问候，而不是崩溃。

`formatAgentsForPrompt`（[L53-L63](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/prompt-formatters.ts#L53-L63)）
渲染 `Classroom agents: - role: ... — name: ... persona: ...`；没有 agents 时返回
空串。注意模板里 `{{agents}}` 独占一行，空串会留一个空行——无害。

### 8.4 四份 `system.md` 的约束差异（本步质量差异的真正来源）

**slide**（[templates/slide-actions/system.md](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/slide-actions/system.md)，180 行，四份里最长）：

- 输出**两种条目交错**：`{"type":"action","name":"spotlight","params":{...}}` 与 `{"type":"text","content":"..."}`；
- **`spotlight` 必须先于对应文本**（先聚焦再讲，否则观众看着标题听内容）；
- 可用动作：`speech`(由 text 转换)、`spotlight`、`laser`、`play_video`、`discussion`；
- **单声道教师独白禁则**：不许出现 `（AI助教）：` 这类角色前缀、不许写舞台指示；
- 引入 `{{snippet:speech-tts-readability}}`——同一份片段也被内容模板用，保证
  "写进旁白的句子"天然适配 TTS（无括号旁批、无长定语从句、数字用读法）。

**quiz**（[quiz-actions/system.md](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/quiz-actions/system.md)）：

- **只许 `type:"text"`**（模板里重复了两次，user.md 也再强调一遍）；
- **绝不泄露答案**，也不许暗示"这题考的是…"式的答案线索；
- 1–2 条，反馈由学生提交后的 post-quiz 流程触发，**不属于本步产物**。

**interactive**（[interactive-actions/system.md](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/interactive-actions/system.md)，118 行）：

- 四个 `widget_*` 动作各有语义：`widget_highlight`（高亮某 selector 并配 subtitle）、
  `widget_setState`（改控件状态，需要 `state` 对象）、`widget_annotation`（贴批注）、
  `widget_reveal`（揭示隐藏块）；
- **明令禁止 `spotlight` / `laser`**——交互组件不是幻灯片，没有 element 坐标系；
- "**target MUST come from the Element Inventory when it is non-empty**"：优先用真实
  selector，清单为空才允许按命名惯例猜；
- 3–8 条。

**pbl**（[pbl-actions/system.md](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/templates/pbl-actions/system.md)）：最短，54 行，
围绕 `{{projectSummary}}` 介绍项目与里程碑，1–2 条纯 `text`。

---

## 9. `action-parser.ts`：七步容错解析管线

全文 162 行（[action-parser.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts)）。
它是本步"为什么不崩"的核心。入口（[L41-L46](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L41-L46)）：

```ts
export function parseActionsFromStructuredOutput(
  response: string,
  sceneType?: string,
  allowedActions?: string[],
  logger: GenerationLogger = noopGenerationLogger,
): Action[]
```

### 9.1 管线逐段

| # | 行 | 做什么 | 失败时 |
|---|---|---|---|
| 1 | [L48-L49](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L48-L49) + [`stripCodeFences` L22-L25](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L22-L25) | trim 后剥首尾 ```json / ``` 围栏 | — |
| 2 | [L51-L60](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L51-L60) | 取**第一个 `[`**；有闭合 `]` 就切到它，否则从 `[` 切到结尾 | 无 `[` → `warn('No JSON array found in response')` + `[]` |
| 3 | [L62-L66](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L62-L66) | `JSON.parse` | 抛错进第 4 步 |
| 4 | [L67-L71](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L67-L71) | `jsonrepair` 再 parse（中文里未转义的引号、尾逗号…） | `info('Recovered malformed JSON via jsonrepair')`；仍失败进第 5 步 |
| 5 | [L72-L82](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L72-L82) | `partial-json`（`Allow.ARR\|OBJ\|STR\|NUM\|BOOL\|NULL`）解析**被截断**的数组 | `warn('Failed to parse JSON array:', …)` + `[]` |
| — | [L84-L87](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L84-L87) | 三条路径汇合后统一 `Array.isArray` 检查 | `warn('Parsed result is not an array')` + `[]` |
| 6 | [L89-L130](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L89-L130) | 逐条转换成 `Action` | 无 `type` 键的条目 `continue`；`action` 条目抛错才 `warn(…skipping…)` |
| 7 | [L132-L159](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L132-L159) | 语义裁剪：discussion 截尾、剥 SLIDE_ONLY、白名单 | — |

三层解析（3→4→5）的顺序很重要：`jsonrepair` 会**破坏**"截断"这一信息（它可能补出
一个语义错误但语法合法的数组），所以只在严格 parse 失败时才试它，而 partial 是
**最后**的兜底。日志措辞因此能直接区分故障种类——`Recovered malformed JSON` 是
"输出坏了但修好了"，`Failed to parse JSON array` 是"彻底救不回来"，而
`No JSON array found` 意味着模型根本没按格式回答（往往是被安全策略拒答或答成了
自然语言）。第 5 层成功时**不打任何日志**：截断在数量上是最常见的"部分可用"，
逐次刷屏没有价值。

### 9.2 第 6 步的两种合法形状

```ts
for (const item of items) {
  if (!item || typeof item !== 'object' || !('type' in item)) continue;
  const typedItem = item as Record<string, unknown>;

  if (typedItem.type === 'text') {
    const text = ((typedItem.content as string) || '').trim();
    if (text) actions.push({ id: `action_${nanoid(8)}`, type: 'speech', text });
  } else if (typedItem.type === 'action') {
    try {
      const actionName = typedItem.name || typedItem.tool_name;
      const actionParams = (typedItem.params || typedItem.parameters || {}) as Record<string, unknown>;
      const action = {
        id: (typedItem.action_id || typedItem.tool_id || `action_${nanoid(8)}`) as string,
        type: actionName as Action['type'],
        ...actionParams,
      } as Action;
      if (action.type === 'widget_setState' && action.state == null) action.state = {};
      actions.push(action);
    } catch (_e) {
      log.warn('Invalid action item, skipping:', JSON.stringify(typedItem).slice(0, 100));
    }
  }
}
```

要点：

1. **`text` → `speech` 只带 `text`**，不带 `agentId`（[L96-L104](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L96-L104)）。
   动作序列里说话的人由第 7 步的"旁白"统一绑定 narrator，解析层不猜角色；
   `content` 为空或全是空白 → 该条**丢弃**（不产生空 `speech`，否则第 7 步会为它调一次 TTS）。
2. **id 一律是 `action_${nanoid(8)}` 形态**，两种来源都可能被模型自带值覆盖：
   `speech` 的 id 总是新铸造，`action` 的 id 优先取 `action_id` / `tool_id`（[L114](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L114)）。
   保留自带 id 是因为音频按 `tts_s<order>_<actionId>` 关联（第 7 册），模型给稳定 id 时
   重试之间可以对上。
3. `params` / `parameters` 都认，`name` / `tool_name` 都认——后者是 legacy 格式，
   文件头注释（[L30-L35](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L30-L35)）把两种写法都列为受支持。
4. **`widget_setState` 的 `state` 缺失补 `{}`**（[L118-L124](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L118-L124)）：
   注释写明了原因——旧的 `TeacherAction→Action` 转换器总是默认成 `{}`，若不补，
   `state: undefined` 会一路进到 iframe，而 `SET_WIDGET_STATE` 的处理器会直接解引用它。
   用的是 `== null`，所以 `state: null` 也会被补成 `{}`。

> **反直觉点**：`type` 未知的动作**不会**在解析期被拒——这一层没有"类型必须在
> `ACTION_TYPES` 里"的检查，`actionName` 为 `undefined`（`name` 和 `tool_name` 都缺）
> 时它照样 `push` 进数组，只是 `type: undefined`。真正拦住它的是**后面的
> `assertValidScene`**（§12），消息为 `unknown action type: "undefined"`。
> 设计意图：解析层负责"尽量拿出东西"，合法性由校验层单点裁决。

### 9.3 discussion 之后全部截断（[L132-L136](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L132-L136)）

```ts
const discussionIdx = actions.findIndex((a) => a.type === 'discussion');
if (discussionIdx !== -1 && discussionIdx < actions.length - 1) {
  actions.splice(discussionIdx + 1);
}
```

`discussion` 是**把麦克风交给学生**的动作，之后老师再"自言自语"毫无意义，而且播放
时会与学生输入冲突。所以它是硬结束点——**即使模型在后面还写了 5 条，也一律丢掉**。
L132 的注释声明了两条不变量（"must be the last action, and at most one"），
实现只用一次 `splice` 就同时满足：截在**第一个** discussion 之后，后面的 discussion
自然也一起没了。原地 `splice` 而不是重新赋值，是因为此时 `actions` 还是 `const`。

### 9.4 剥掉不属于本类型的动作（[L138-L146](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L138-L146)）

```ts
let result = actions;
if (sceneType && sceneType !== 'slide') {
  const before = result.length;
  result = result.filter((a) => !SLIDE_ONLY_ACTIONS.includes(a.type as ActionType));
  if (result.length < before) {
    log.info(`Stripped ${before - result.length} slide-only action(s) from ${sceneType} scene`);
  }
}
```

`SLIDE_ONLY_ACTIONS` 来自 DSL 包（[action.ts#L264](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/action.ts#L264)），
是**跨层共享的常量**而不是本文件里的私有列表——渲染器、prompt、解析器三处用的是
同一份事实。quiz / interactive / pbl 里出现 `spotlight` 会被滤掉并记一条
`info('Stripped N slide-only action(s) from <type> scene')`，**即使 prompt 已经禁过**
（interactive 的 system.md 明令禁止）。注意守卫是 `sceneType && sceneType !== 'slide'`：
`sceneType` 传 `undefined` 时**一段都不过滤**。本步四个分支都传 `outline.type`，
但在线对话的调用方不一定传——这是一个"调用方负责"的开关。

### 9.5 三层防御纵深

同一个"别在测验页放 spotlight"的意图被实现了三次，任何一层单独失效都不至于
让非法动作进入课堂：

| 层 | 位置 | 手段 | 失效会怎样 |
|---|---|---|---|
| 1 说服 | `*-actions/system.md` | 自然语言禁令（"Do NOT use spotlight"） | 模型偶尔违反 |
| 2 过滤 | 解析器 §9.4 + 白名单 [L148-L159](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L148-L159) | 确定性地删 | `allowedActions` 忘了传 |
| 3 拒绝 | `assertValidScene`（§12） | 整场景不合格 → 本步重试 | 课程 paused |

第 7 段的注释是全文最需要读的一句（[L148-L150](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L148-L150)）：

```ts
// Step 7: Filter by allowedActions whitelist (defense in depth for role-based isolation)
// Catches hallucinated actions not in the agent's permitted set, e.g. a student agent
// mimicking spotlight/laser after seeing widget actions in chat history.
if (allowedActions && allowedActions.length > 0) {
  const before = result.length;
  result = result.filter((a) => a.type === 'speech' || allowedActions.includes(a.type));
  if (result.length < before) {
    log.info(`Stripped ${before - result.length} disallowed action(s) by allowedActions whitelist`);
  }
}
```

三点值得停一下：

1. 这一层的动机**不是**"测验页不该有 spotlight"（那是 §9.4 的职责），而是**角色隔离**：
   同一个解析器被在线对话（teacher/student agent 的 tool call 流）复用，学生 agent
   看了聊天记录后会模仿出 `spotlight`，而它没有聚光灯权限。注释里那句
   "after seeing widget actions in chat history"就是这个场景。
2. **`speech` 是隐式放行的**（`a.type === 'speech' ||`）。若没有这条例外，interactive
   的白名单会把旁白全部删光，场景只剩 widget 动作、没有人说话。
3. 生效条件是 `allowedActions && allowedActions.length > 0`——**空数组等于不限**，
   不是"什么都不许"。调用方想收紧时必须传非空列表，传 `[]` 会静默地什么都不过滤
   （本步只有 interactive 传，见 §7.4）。

---

## 10. `processActions`：确定性修补与四套兜底

### 10.1 修补（[scene-generator.ts#L1983-L2034](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1983-L2034)）

签名 `processActions(actions, elements, agents?, log = noopGenerationLogger)`。
函数头先建三张表（[L1989-L1992](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1989-L1992)）：
`elementIds`、`agentIds`、`studentAgents`（`role === 'student'`）、`nonTeacherAgents`
（`role !== 'teacher'`）。然后 `actions.map`，只做三件事，全部是**不改变讲述结构**的救火：

| 问题 | 处理 | 真实日志文本 |
|---|---|---|
| 动作缺 `id` | `action.id \|\| \`action_${nanoid(8)}\`` | 无 |
| `spotlight.elementId` 缺失或不在本屏元素里 | **回退到 `elements[0].id`**（仅当 `elements.length > 0`） | `Invalid elementId, falling back to first element: <新 id>` |
| `discussion.agentId` 缺失或不在班级画像里 | 从 **student 池**随机挑一个；池空则从**非教师池**挑；**`agents` 本身为空则整段不执行** | `Discussion agentId "<旧值或 (none)>" invalid, assigned: <id> (<name>)` |

三点值得注意：

- **spotlight 用"回退第一个元素"而不是"删掉动作"**：保留动作保住了讲述节奏（老师
  仍然"指了点什么"，只是指错了）。删掉会让后面的 `text` 变成没有配图的独白。
  quiz / interactive / pbl 传的是 `[]`（§7.1 表），`elements.length > 0` 不成立 →
  这段整体跳过，**非 slide 场景绝不能指望这里纠正 selector**（好在 §9.4 已经把它们
  的 `spotlight` 滤掉了）。
- **兜底日志会误导排障**：`log.warn` 打的是**赋值之后**的 `spotlightAction.elementId`，
  即"回退到的那个 id"，而不是"模型写错的那个 id"（[L2007-L2010](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2007-L2010)）。
  看日志时会以为是"第一个元素本来就不存在"。同理 discussion 那条把旧值放在
  `"<old>" invalid` 的位置，才带上 `assigned: <new>`。
- **discussion 的兜底可能压根不跑**：条件写作 `processedAction.type === 'discussion' && agents && agents.length > 0`
  （[L2016](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2016)）。
  第 4 步没产出 agent（或被裁剪成空数组）时，非法/缺失的 `agentId` **原样留下**。
  这不会让场景校验失败——`discussion` 的必填只有 `topic`（[validate.ts#L67](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L67)），
  `agentId` 是**可选**字段（[action.ts#L196-L201](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/action.ts#L196-L201)），
  所以**非法 agentId 能通过校验**，到播放器里表现为"这条讨论没有主持人"（用默认画像）。
  这是**已知的设计选择**，不是 bug。

### 10.2 四套 default 生成器

模型不可用（模板缺失 / 解析为空）时课堂仍要有话说，所以每个类型都有一个**零模型**
的兜底：

| 类型 | 兜底动作（全部硬编码） | 位置 |
|---|---|---|
| slide | `spotlight{title:'聚焦重点', elementId: 第一个 text 元素}`（**仅当存在 text**）+ `speech{title:'场景讲解', text: keyPoints.join('。')+'。' \|\| description \|\| title}` | [L2039-L2065](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2039-L2065) |
| quiz | 单条 `speech{title:'测验引导', text:'现在让我们来做一个小测验，检验一下学习成果。'}` | [L2070-L2079](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2070-L2079) |
| interactive | 单条 `speech{title:'交互引导', text:'现在让我们通过交互式可视化来探索这个概念。请尝试操作页面中的元素，观察变化。'}` | [L2084-L2093](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2084-L2093) |
| pbl | 单条 `speech{title:'PBL 项目介绍', text:'现在让我们开始一个项目式学习活动，了解项目的驱动问题，并在项目工作区中逐步探索和实践。'}` | [L1929-L1938](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1929-L1938) |

三件事必须知道：

1. **兜底文案是硬编码中文，且不含大纲标题**：quiz / interactive / pbl 三个兜底的形参
   写作 `_outline`（下划线前缀 = 压根不用），文案一个字都不会随场景变化；只有 slide
   兜底真的读了大纲。它们也**不受 `languageDirective` 影响**——英文课上会出现一句中文
   "测验引导"。这是可接受的代价（宁可说一句母语提示，也不要一屏死寂），但排查
   "为什么这句旁白是中文"时答案就在这里。
2. **slide 兜底只用第一个 `text` 元素**当 spotlight 目标（[L2043-L2051](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L2043-L2051)）：
   过滤条件是 `el.type === 'text'`，若本屏**没有任何 text 元素**（纯图 / 纯公式），
   兜底就只有一条 `speech`，一条动作都不画——这是合法结果，不是失败。
   旁白的文本另有优先级：`keyPoints` 拼接 → `description` → `title`。
3. 兜底**不经过 `processActions`**（分支里直接 `return`），因此它的 id 由
   `action_${nanoid(8)}` 现造、也不会触发任何 warn / info 日志。换句话说：
   日志里**一条 `Invalid elementId` 都没有**并不能证明模型输出是好的，
   也可能压根没走模型（模板缺失或解析为空 → 兜底静默）。要区分只能看
   `[Actions Gen]` / `Stripped …` 这类 info 是否出现，或直接查场景的动作条数
   （slide 兜底恒为 1~2 条）。

---

## 11. `buildCompleteScene`：装配

全文 115 行（[scene-builder.ts](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts)），
一个纯函数：输入大纲 + 内容 + 动作 + stageId，输出一个**与存储无关**的完整场景，
或者 `null`。

```ts
export interface BuildCompleteSceneOptions {
  /**
   * Stable identity supplied by retrying/upserting consumers. Reusing it turns
   * a replay into the same logical scene instead of appending a duplicate.
   * The default remains a random `nanoid()` for drop-in compatibility.
   */
  sceneId?: string;
}

/** Build a complete, store-independent scene from generated primitives. */
export function buildCompleteScene(
  outline: SceneOutline,
  content: GeneratedSlideContent | GeneratedQuizContent
           | GeneratedInteractiveContent | GeneratedPBLContent,
  actions: Action[],
  stageId: string,
  options: BuildCompleteSceneOptions = {},
): CompleteScene | null
```

### 11.1 骨架：四个分支返回的是同一组字段（[L33-L34](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L33-L34)）

```ts
const sceneId = options.sceneId ?? nanoid();
const timestamps = { createdAt: Date.now(), updatedAt: Date.now() };
```

然后每个分支各写一遍返回对象（没有 `baseScene` 变量，字段是逐分支重复的），形状恒为：

```ts
{
  id: sceneId,
  outlineId: outline.id,
  stageId,
  type: 'slide',                 // 字面量，与各分支的判据一致
  title: outline.title,
  order: outline.order,
  content: { type: 'slide', canvas },   // ← 唯一因类型而异的部分
  actions,
  ...timestamps,
}
```

| 字段 | 来源 | 备注 |
|---|---|---|
| `id` | `options.sceneId ?? nanoid()` | **run 管线不传 `sceneId`**（§6.5 只传 4 个实参），于是每次尝试一个新 id |
| `outlineId` | `outline.id` | 场景反向指回大纲；`CompleteScene` 类型把它列为**必填**（[scene-types.ts#L57](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-types.ts#L57)） |
| `stageId` | 第 4 步铸造 | 场景归属课程（不是归属大纲） |
| `type` | 分支里的**字面量** | 不是 `outline.type`——`'slide'` 四个字写死在 [L57](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L57)，所以类型收窄对 TS 是有效的 |
| `title` | `outline.title` | 字段就叫 `title`，与大纲同名（第 5 册里的 `name` 说法是错的，`Scene.name` 不存在） |
| `order` | **`outline.order` 原样** | 装配层**不做** `index + 1`；1 基序号在大纲生成时就已铸造（[outline-generator.ts#L174](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/outline-generator.ts#L174)），编辑大纲时会重排（[run/input.ts#L259](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L259)） |
| `actions` | §7–§10 的产物 | 直接挂上，装配层不再校验 |
| `createdAt` / `updatedAt` | 两次 `Date.now()` | 同一个函数体内相邻两行，实践中相等 |

> **坑（场景 id 不稳定）**：因为不传 `sceneId`，一个场景在第 5 次重试时与第 1 次
> 尝试拥有**不同的 id**。这不是 bug——检查点里存的是"最后一次成功尝试"的整个对象，
> 从未被采用的 id 不会留下痕迹。但它意味着：**不要**在任何地方缓存场景 id 去等
> actions 步；`audioId` 用的是场景 order（`tts_s<order>_<actionId>`，第 7 册）
> 正是为了避开 id 易变。`BuildCompleteSceneOptions` 的注释也点明了这个契约的
> 另一端：**只有"重试后要 upsert 同一逻辑场景"的调用方才需要传 `sceneId`**。

### 11.2 四条判据：`outline.type` AND 内容形状

与 §7 完全对称——分派**同时**要求大纲类型和内容形状匹配，四条都不中才 `return null`：

| 分支 | 判据 | 产出的 `content` | 行 |
|---|---|---|---|
| `slide` | `outline.type === 'slide' && 'elements' in content` | `{ type: 'slide', canvas }` | [L36-L64](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L36-L64) |
| `quiz` | `outline.type === 'quiz' && 'questions' in content` | `{ type: 'quiz', questions }` | [L66-L78](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L66-L78) |
| `interactive` | `outline.type === 'interactive' && 'html' in content` | `{ type: 'interactive', url: '', html, widgetType, widgetConfig }` | [L80-L98](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L80-L98) |
| `pbl` | `outline.type === 'pbl' && 'projectV2' in content` | `{ type: 'pbl', projectV2 }` | [L100-L112](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L100-L112) |
| 都不匹配 | — | **`return null`** → 上层 `StepRefusal('assembly-failed')` | [L114](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L114) |

这里有一处**与 §7 不对称的关键差异**：`generateSceneActions` 的 pbl 分支只看
`outline.type === 'pbl'`（[L1893](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1893)），
而 `buildCompleteScene` 的 pbl 分支**要求 `projectV2`**。于是 pbl 会出现"动作生成成功、
装配却返回 null"的组合——这正是 §6.4 遗留内容升级必须发生在两次调用**之前**的原因：
若内容里只有 v1 的 `projectConfig`，动作照样能生成（浪费一次模型调用），但装配返回
null，整步以 `assembly-failed` 失败重试。测试
[scene-step-retry-boundary.test.ts#L248-L276](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/scene-step-retry-boundary.test.ts#L248-L276)
钉住的正是这条链（"title-only legacy shell" → 拒绝，消息
`Failed to build scene: Community Garden Data Project`）。

`url: ''` 是历史字段：1.0 时代交互场景走外链，现在 HTML 在 `content.html` 里，
但字段必须保留才能被旧渲染器接受。

### 11.3 slide 的 canvas：每次全新铸造（[L37-L52](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L37-L52)）

只有 slide 需要画布，而且**不管内容里有没有 canvas，装配层总是新建一个**：

```ts
const defaultTheme: SlideTheme = {
  backgroundColor: '#ffffff',
  themeColors: ['#5b9bd5', '#ed7d31', '#a5a5a5', '#ffc000', '#4472c4'],
  fontColor: '#333333',
  fontName: 'Microsoft YaHei',
  outline: { color: '#d14424', width: 2, style: 'solid' },
  shadow: { h: 0, v: 0, blur: 10, color: '#000000' },
};
const canvas: Slide = {
  id: nanoid(),
  viewportSize: 1000,
  viewportRatio: 0.5625,          // 16:9
  theme: defaultTheme,
  elements: content.elements,
  background: content.background,
};
```

三点含义：

- 第 5 步**只交 `elements`（和可选 `background`）**，不需要关心画布基础设施；画布 id、
  视口、主题由第 6 步负责。这是"内容生成器与显示层解耦"的落点。
- `theme` 恒为这份硬编码默认值，因此**同一次课程里所有 slide 场景的初始主题一致**；
  课程级主题（若有）是之后由其他机制覆盖的，不在本步。
- `canvas.id` 也是 `nanoid()`。`viewportSize: 1000` + `viewportRatio: 0.5625` 定义了
  **逻辑坐标系**：视口宽 1000、高 562.5 个单位，元素的 `left/top/width/height` 就活在这
  套单位里（渲染时整体缩放，见 [slides.ts#L990-L1000](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/slides.ts#L990-L1000)
  的注释——这三个字段在 DSL 契约里是**必填**，只有 importer 允许分片解析）。
  因此 `spotlight` 的目标几何与屏幕像素无关，模型只需按第 5 步交回的元素坐标指哪打哪。

### 11.4 `CompleteScene` 类型

`CompleteScene` 来自 [scene-types.ts#L54-L57](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-types.ts#L54-L57)：

```ts
export type CompleteSceneContent = SlideContent | QuizContent | InteractiveContent | PBLContent;
export type CompleteScene = Scene<Action, CompleteSceneContent> & { outlineId: string };
```

而 `SceneActionsResult['scene']` 是它的**派生**（`NonNullable<ReturnType<...>>`，§6.1）
——因此本步骤文件的类型永远与装配函数同步，不需要人工维护两份定义；装配函数一旦
多返回 `null`，步骤层的 `scene` 类型仍是"非空场景"，编译器会逼调用点显式处理 null。

---

## 12. 校验链：`assertValidScene` 之后才允许 commit

### 12.1 引擎侧（[engine.ts#L325-L336](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L325-L336)）

```ts
/**
 * The scene a generation step is about to store must be one the course can
 * hold: the same check the document store applies on write, applied here so an
 * invalid scene fails the step and regenerates rather than reaching narration
 * and failing there for good.
 */
export function assertValidScene(scene: unknown, stepId: string): void {
  const result = validateAppScene(scene);
  if (result.valid) return;
  const detail = result.errors.slice(0, 5).map((error) => `${error.path}: ${error.message}`).join('; ');
  throw new InvalidSceneError(
    `${stepId} produced a scene the course cannot hold: ${detail}`,
    scene as Scene,
  );
}
```

"the same check the document store applies **on write**"是关键：校验规则只有一份，
run 里提前跑一次是为了**争取重生成机会**，而不是为了新增限制。也就是说：
任何能被本函数拒掉的对象，也必然会被第 7 步的 `appendScene` 拒掉——区别只在
前者可以重试 5 次，后者会把已经旁白好的音频变成孤儿。

只列前 5 条错误：模型输出常见"一处错误引发全树报错"，5 条足够定位且能防日志爆炸。
`InvalidSceneError` 是个只带消息的私有类（[engine.ts#L317-L323](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L317-L323)），
它的 `name` 是 `'InvalidSceneError'`，所以这条原因会作为 `cause` 出现在 `step_retry`
事件里；`failureCodeOf` 又把它单独映射成 `GENERATION_FAILED`
（[engine.ts#L312-L314](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L312-L314)）。
pg 测试正是用 `expect(String(retries[0]!.data.cause)).toMatch(/cannot hold/)` 钉住这条
消息（[generation-runs.pg.test.ts#L1993](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L1993)）。

### 12.2 存储侧（[lib/document-store/validators.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts)）

```ts
export const validateAppScene: SceneValidator = (scene) => {
  const value = objectValue(scene);
  if (!value) return { valid: false, errors: [{ path: '/', message: 'scene must be an object' }] };
  if (value.type === 'slide' || value.type === 'quiz') return validateScene(scene);  // 严格
  …  // 宽松分支
};
```

| 场景类型 | 校验强度 | 实际检查的东西 |
|---|---|---|
| `slide` / `quiz` | **严格**：整把交给 DSL `validateScene` | 顶层 `id`/`stageId`/`title`/`order` + `content` 形状 + 每条 action（[L32](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L32)） |
| `interactive` / `pbl` | **宽松**：app 自己写的一份 | 顶层四项（下详）+ `content.type` 必须等于场景 `type` + 类型特有字段 + **`actions` 逐条** |
| 其他 `type` | 走宽松分支但立刻报错 | `unknown app scene type: "<type>"`（[L43-L47](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L43-L47)） |
| 不是对象 | 直接失败 | `scene must be an object` |

宽松分支的顶层检查（[L34-L40](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L34-L40)）是：
`id` / `stageId` / `title` 三个**非空字符串**（`requiredString` 的消息为
`expected non-empty string \`title\``），以及 `order` 必须是**有限数**
（`expected finite number \`order\``——注意这里比 DSL 更严，DSL 只要求 `typeof === 'number'`，
所以 `NaN` 会在 app 写边界被拦下）。类型特有字段：

- interactive：`html` 与 `url` **至少要有一个是字符串**；两者若存在必须都是字符串；
  `widgetConfig` 若存在**必须是对象**（注释解释：基本类型会让 hydration 的
  `migrateInteractiveContent` 在 `'in'` 上抛异常——"Primitive widgetConfig values crash
  hydration"，[L68-L76](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L68-L76)；
  数组和无类型的对象作为历史形状**继续容忍**）。
- pbl：`projectConfig` 若存在必须是对象；`projectV2` 的处理最绕（[L86-L108](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L86-L108)）——
  `null` 当作"没有"（让 v2 校验上线之前存下来的文档仍能保存），而**同时**带着
  "结构完好且非空的 `projectConfig`"的场景被认定为 cutover 前的 hybrid 队列，
  其中损坏的 `projectV2` 作为**惰性字节**保留；但空壳 `{}` **不算**合法 legacy 数据，
  不能反过来关掉 v2 校验。最终要求是 `milestones` / `roles` / `threads` 三个数组都在
  （`hasPBLProjectV2Containers`）。

对 actions 的处理前有一段本仓最长的注释之一
（[L111-L124](file:///D:/GITHUB_chat/OpenMAIC/lib/document-store/validators.ts#L111-L124)），要点：

1. 历史上这个分支**完全没看过 `actions`**，于是 `patch_stage set /actions = "not-an-array"`
   会把一个字符串存进去，播放时 `(scene.actions ?? []).map is not a function` 直接崩——
   注释给了事故编号 **R5-P2-2**；
2. 现在每条动作**必须有字符串 `id` 和字符串 `type`**。缺 `type` 就**不再往下查**
   （`return`），因为无法为未知类型选变体规则；
3. `type` **在 DSL 注册表内**（`isActionType`）的动作，额外跑 `validateAction` 的变体必填校验——
   注释点名：`{id, type:'speech'}` 而没有 string `text` 会让播放引擎的
   `speechAction.text.trim()` 崩（**R6-P2-3**），"所以存它等于存腐败"；
4. `type` **不在注册表内**的，除 id/type 形状外保持宽容——"the write barrier is for
   corruption, not canon"，历史语料里有几百个自造类型必须还能保存；
5. 只缺 `id` 的历史动作在到达这道屏障**之前**已由 `canonicalizeLegacyScene` 补齐（**R6-P2-2**）。

这段是理解本步"什么会失败"的最短路径（消息取自源码原文）：

| 模型输出的动作 | 在 slide / quiz 场景（严格） | 在 interactive / pbl 场景（宽松） |
|---|---|---|
| 缺 `type`（`{"id":"a1"}`） | `unknown action type: undefined` | `expected string \`type\`` |
| 缺 `id` | `expected string \`id\`` | `expected string \`id\`` |
| `type:'speech'` 缺 `text` | `speech action requires \`text\`` | **同样失败**（注册表内 → 跑 `validateAction`） |
| `type:'brand_new_action'` | `unknown action type: "brand_new_action"` | ✅ 放过（只有 id/type 形状要求） |
| `actions` 不是数组 | `` `actions` must be an array `` | `` `actions` must be an array `` |

> 所以两种强度的**真正差别只有三处**：场景顶层字段的严格度（`NaN` order、canvas/questions
> 形状等）、`content` 的类型特有规则、以及**注册表外动作类型**是否放过。注册表内类型缺字段
> 在两边都会失败——这也是为什么 §10 的 `processActions` 必须给每条动作补 `id` 与 `type`：
> 那是唯一**两边都查**的两件事。

### 12.3 DSL 必填表（[validate.ts#L43-L72](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L43-L72)）

`ACTION_REQUIRED_FIELDS` 是 `Record<ActionType, Readonly<Record<string, FieldKind>>>`（编译期
强制 22 个类型全覆盖），除公共 `id` 外逐类型的额外必填（`FieldKind` 决定消息里的类型词）：

| 动作 | 额外必填 |
|---|---|
| `spotlight` / `laser` / `play_video` | `elementId` (string) |
| `speech` | `text` (string) |
| `wb_open` / `wb_clear` / `wb_close` | 无（`{}`，只查 `id`） |
| `wb_draw_text` | `content` (string) + `x` `y` (number) |
| `wb_draw_shape` | `shape` (string) + `x` `y` `width` `height` (number) |
| `wb_draw_chart` | `chartType` (string) + `x` `y` `width` `height` (number) + `data` (object) |
| `wb_draw_latex` | `latex` (string) + `x` `y` (number) |
| `wb_draw_table` | `x` `y` `width` `height` (number) + `data` (array) |
| `wb_draw_line` | `startX` `startY` `endX` `endY` (number) |
| `wb_draw_code` | `language` `code` (string) + `x` `y` (number) |
| `wb_edit_code` | `elementId` `operation` (string) |
| `wb_delete` | `elementId` (string) |
| `discussion` | `topic` (string)——**注意 `agentId` 不在表里**（§10.1） |
| `widget_highlight` / `widget_annotation` / `widget_reveal` | `target` (string) |
| `widget_setState` | `state` (object) |

三个容易记错的点：`laser` 要的是 **`elementId`** 而不是坐标；`discussion` 要的是
**`topic`** 而不是 `question`；表里**没有** `position` / `question` / `videoId` / `url` /
`widgetId` / `path` / `project` 这些字段名，动作类型也只有 `wb_*` / `widget_*` 前缀，
**没有** `open_*` / `code_*` 系列。

`checkAction`（[L151-L178](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L151-L178)）
只可能产生四种消息，照抄原文最有用：

```
action must be an object
unknown action type: "<type>"
<type> action requires `<field>`
<type> action field `<field>` must be <kind>
```

再加 `reqString` 的 `expected string \`id\``（[L84-L92](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L84-L92)）。
注意 L120-L126 的注释：变体字段循环**故意不走**通用的 `checkFields`，因为它要用
嵌套路径和不同的 "requires / must be" 消息形状。

纯函数入口是 `validateScene`（[L311](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L311)）、
`validateAction`（[L332](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L332)），
两者都返回 `{ valid, errors }`（`done()` 统一封装，[L104-L106](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/validate.ts#L104-L106)）；
`validateAppScene` 返回的是**同一个形状**，因此 `assertValidScene` 里的
`result.errors.slice(0, 5)` 拿到的 `path` 已经是 `/actions/3/text` 这种完整嵌套路径——
引擎消息里 `path: message` 的拼接（§12.1）可直接定位到第几条动作的哪个字段。

## 13. 事件与前端可见性

本步产生的事件全部由引擎统一发（步骤实现自己**不发事件**，只 `return` 结果 + `log`），
三类：

| 事件 | 发送点 | data | 与 actions 的关系 |
|---|---|---|---|
| `step_started` | 每个 `runStep` 前 | `{ step }` | `step = 'scene:<n>:actions'` |
| `step_completed` | `done()` | `{ step, output }` | 本步 `output = { scene, actions }` |
| `step_retry` | [engine.ts#L438-L449](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L438-L449) | `{ step, attempt, maxAttempts, reason, cause }` | 解析为空 / 装配失败 / 校验不过都在这里 |
| `scene_ready` | [engine.ts#L1178](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1178) | `{ index, sceneId, order }` | **不由本步发**，在第 7 步旁白落库时才发 |
| `course_created` | [engine.ts#L1175-L1177](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1175-L1177) | `{ stageId }` | 仅 `sceneIndex === 0`，同样在旁白侧 |

两条容易踩的认知：

1. **`step_retry` 的 `cause` 是排障第一现场**，但类型定义的注释里只写了
   `{ step, attempt, maxAttempts, reason }`（[types.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts)），
   引擎实际多发了一个 `cause`——它是 `InvalidSceneError.message` 或 refusal 文本。
   pg 测试正是断言这一项：
   [generation-runs.pg.test.ts#L1993](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L1993) 的
   `expect(String(retries[0]!.data.cause)).toMatch(/cannot hold/)`。
   看前端 UI 时也要知道：`cause` 只在 `log.warn` 那行（L435-L437）进服务端日志，
   UI 上通常只显示 `reason` + 第几次。
2. **动作内容对前端是不可见的**。`GET /api/generation-runs/[id]` 返回的是
   `runSnapshot`（[app/api/generation-runs/[id]/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/%5Bid%5D/route.ts)），
   只有 run/step 的**状态**，不含 `output` 里的场景 JSON；SSE 也只推事件类型 + 少量
   data。所以本步 `step_completed` 的 `{ scene, actions }` **留在服务端的 step 表里**，
   用户要看到动作，必须等第 7 步把场景写进课程文档、前端再拉课程。

预览页的动画组件同样不含数据（[visualizers.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx)）：

```
StepVisualizer@22  →  case 'actions' → ActionsVisualizer@673
```

`ActionsVisualizer` 里写死了 5 行装饰项（`Speech / Spotlight / Speech / Interact / Speech`），
每 **1600 ms** 轮播高亮一行，`ROW_H = 34`（[L676-L723](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L676-L723)）——
**它不读任何 step 输出**。所以"预览页动画卡在某一行"不代表后端卡在那条动作，
它只是个进度指示器；真正的进度信号是 SSE 事件序列。

还有一处**命名陷阱**：预览槽位映射（[preview-steps.ts#L42-L43](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L42-L43)）

```ts
const scene = step ? /^scene:\d+:(content|actions|narration)$/.exec(step) : null;
if (scene) return scene[1] === 'content' ? 'slide-content' : 'actions';
```

即 **`actions` 与 `narration` 共用同一个 `'actions'` 预览位**。UI 上那一格
"教学动作"在旁白阶段仍然是活跃状态，看起来像"动作步骤又跑了一次"，其实已经是第 7 步。
反过来说：想在 UI 上区分第 6/7 步，只能看 `step_started` 事件里的 step id 后缀。

## 14. 常量速查、坑清单与代码地图

### 14.1 常量速查

| 名称 / 值 | 位置 | 作用 |
|---|---|---|
| `'scene:<n>:actions'` | [engine.ts#L712](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L712) | step id，重试与幂等的键 |
| `retryOptions(stepId, index, 500)` | [engine.ts#L713](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L713) | 500 ms 基础退避；首场景 `FIRST_SCENE_MAX_RETRIES`，其余 `SCENE_MAX_RETRIES` |
| `refusalStatus: 500` | [engine.ts#L714](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L714) | 装配类拒绝按 5xx 记（4xx 留给输入侧） |
| `maxRetries: 0` | [scene-actions.ts#L78-L122](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/scene-actions.ts#L78-L122) | **禁止 provider 内建重试**，把重试权收给 run 引擎 |
| `stage: 'scene-actions'` | 同上 | 计量/限流/日志的分组键 |
| `action_${nanoid(8)}` | [action-parser.ts#L96-L130](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/action-parser.ts#L96-L130) + [scene-generator.ts#L1997](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L1997) | 动作 id 形态（第 7 步音频按它关联） |
| `viewportSize: 1000` / `viewportRatio: 0.5625` | [scene-builder.ts#L44-L47](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-builder.ts#L44-L47) | slide 的逻辑画布（1000×562.5 单位） |
| `INTERACTIVE_WIDGET_ACTIONS` | [scene-generator.ts#L70-L75](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/scene-generator.ts#L70-L75) | interactive 唯一允许的动作类型（4 项） |
| `SLIDE_ONLY_ACTIONS = ['spotlight','laser']` | [action.ts#L264](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/dsl/src/action.ts#L264) | 跨层共享的非 slide 禁令 |
| `errors.slice(0, 5)` | engine.ts `assertValidScene` | 引擎消息里最多列 5 个问题 |

### 14.2 五个最常踩的坑

1. **场景 id 每次尝试都可能变**。`buildCompleteScene` 默认 `nanoid()`，而 actions 步骤
   的重试会**重新装配**一次；只有第 7 步落库后的 id 才是稳定身份。因此不要在
   actions 步骤的输出上做任何跨重试缓存（`scene-step-retry-boundary.test.ts` 里
   `buildCompleteScene.mock.calls[0][1] === generateSceneActions.mock.calls[0][1]`
   就是在钉住"同一次尝试内 content 必须是同一个对象"）。
2. **未知动作名不会在解析期报错**（§9.2），会一路走到 `assertValidScene` 才炸成
   `unknown action type: "..."`。所以看到整场景重试，先看 `cause` 里的动作 type，
   别去翻解析日志——那一段是静默的。
3. **`processActions` 的 spotlight 回退只对 slide 生效**。quiz / interactive / pbl
   传的是 `elements = []`（§7.1），`if (elements.length === 0) return processed` 直接短路。
   且 warn 文本 `Invalid elementId, falling back to first element: <新 id>` 打的是
   **回退之后**的 id——按日志去查那个 id 会一无所获，要查的是原始输出。
4. **非法 `agentId` 能通过全部校验**。`DiscussionAction` 只必填 `topic`（§12.3），
   `agentId` 是可选；`processActions` 只在 `agents && agents.length > 0` 时纠正它。
   若某次调用没传 `ctx/agents`（run 管线里确实可能），一条指向不存在角色的
   discussion 会存进课程，播放时才表现为"没人接话"。
5. **run 管线不传 `logger`**，因此 `noopGenerationLogger` 让 `[Actions Gen]` 的
   prompt/响应调试日志全程静默（§7.1）；同时它也不走视觉分支——只有
   `hasVision` 为真且有截图时才调 `buildVisionUserContent`。排查"动作很奇怪"却
   什么都看不到时，要么在本地传 logger，要么直接读 checkpoint 里的 step output。

> 补充一条关于兜底的：`generateDefaultSlideActions` 在无 text 元素时**只有一条
> speech**（§10.2），这是合法状态；而 quiz / interactive / pbl 的兜底文案是硬编码中文
> 且**不受 `languageDirective` 影响**，英文课里出现中文旁白时先怀疑这里。

### 14.3 代码地图（本步全部相关文件）

```
lib/server/generation/steps/scene-actions.ts      步骤实现：读上一步 output → 选模型 → callLLM
  ├─ lib/server/generation/run/engine.ts          编排：step id / 重试 / 校验 / 落 checkpoint
  ├─ lib/server/generation/run/services.ts        模型解析、TTS/ASR 可用性、bindings
  └─ lib/server/generation/run/retry.ts           retryOptions / 退避 / StepRefusal
packages/@openmaic/generation/src/
  ├─ scene-generator.ts        generateSceneActions / processActions / 四个 default 生成器
  ├─ action-parser.ts          四级容错解析 + 三层裁剪
  ├─ scene-builder.ts          buildCompleteScene（纯装配，四判据 + null）
  ├─ scene-types.ts            Generated*Content / CompleteScene
  └─ prompts/loader.ts         buildPrompt / interpolateVariables / PROMPT_VARIABLE_DEFAULTS
prompts/templates/{slide,quiz,interactive,pbl}-actions/{system,user}.md
packages/@openmaic/dsl/src/
  ├─ action.ts                 22 个动作类型 + ACTION_TYPES + 常量集合
  ├─ slides.ts                 Slide / Canvas 必填契约
  └─ validate.ts               ACTION_REQUIRED_FIELDS / checkAction / checkScene
lib/document-store/validators.ts                  写屏障 validateAppScene
app/api/generation-runs/[id]/route.ts             runSnapshot（不含 step output）
lib/generation-run-client/preview-steps.ts        step id → 预览槽位
app/generation-preview/components/visualizers.tsx ActionsVisualizer（纯装饰动画）
tests/server/generation-steps/scene-actions.test.ts
tests/generation/scene-step-retry-boundary.test.ts
tests/server/generation-runs.pg.test.ts
```

### 14.4 动手验证

| 想验证什么 | 命令 / 操作 | 预期 |
|---|---|---|
| 解析器的四级容错 | 在 `action-parser.ts` 里喂 `'这里没有任何数组'` | 返回 `[]`，无异常 |
| 同上（截断） | 喂 `'[{"type":"action","name":"speech","params":{"text":"a"'`（不闭合） | 走 partial-json，得到 1 条 `speech` |
| 同上（坏引号） | 喂 `'[{"type":"action","name":"speech","params":{"text":"他说"你好""}}]'` | 走 `jsonrepair`，日志含 `Recovered malformed JSON` |
| quiz 剥 spotlight | `parseActionsFromStructuredOutput(x, 'quiz')`，x 含 spotlight | 该条消失，日志 `Stripped 1 slide-only action(s) from quiz scene` |
| 白名单 | 第 3 参传 `['widget_highlight']`，输入含 `spotlight` + `speech` | 只剩 `speech`（§9.5 第 2 点） |
| 装配 null | `buildCompleteScene({type:'pbl',...}, {projectId:'x'}, [], 'stage')` | 返回 `null` → `StepRefusal('assembly-failed')` |
| order 来源 | 断言 `buildCompleteScene` 结果 `order === outline.order` | 相等（**不是** `index + 1`） |
| 步骤单测 | `bun test tests/server/generation-steps/scene-actions.test.ts` | 全绿 |
| 重试边界 | `bun test tests/generation/scene-step-retry-boundary.test.ts` | 全绿（含 pbl refusal） |
| 引擎校验链 | pg 测试 `tests/server/generation-runs.pg.test.ts` 的 L1960 / L1997 两个用例 | `cause` 命中 `/cannot hold/` |
| 前端不可见性 | 生成中打开 `/api/generation-runs/<id>` | JSON 里没有动作内容 |

## 15. 一句话总结

> 第 6 步是**"把模型吐出的字符串数组变成引擎认可的合法动作序列"**：
> `generateSceneActions` 按类型组装 prompt（并**恒传** `userProfile` / `languageDirective`，
> 哪怕空串），`parseActionsFromStructuredOutput` 用四级解析 + 三层裁剪
> 保证"绝不为空手崩掉"，`processActions` 只在 slide 场景补 `elementId` / `agentId`，
> `buildCompleteScene` 纯装配成 `CompleteScene`，最后 `assertValidScene` 一票否决——
> 校验不过不是"降级保存"，而是整场景重试。

三条不变量（后面每一步都建立在此之上）：

1. **每条动作都有字符串 `id` 和字符串 `type`**——这是严格与宽松两种校验**唯一共同**
   要求的东西，也是第 7 步音频关联键 `tts_s<order>_<actionId>` 的来源；
2. **动作序列是纯数据**：不含 React 节点、不含 DOM 引用、不含 Promise，
   因此可以安全地存 checkpoint、可以原样重放、可以在重试间比较；
3. **一个场景要么完整合法、要么不存在**：本步没有"部分成功"的中间态，
   `buildCompleteScene` 返回 `null` 就是失败，`assertValidScene` 抛错就是失败。

下一步：[第 7 册 旁白拆分与入库](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-7.md)——
把 `speech` 文本切成带时间轴的旁白、逐条合成 TTS、写进课程文档，并在同一事务里
落 checkpoint；这是唯一会真正**改动课程**的一步。


