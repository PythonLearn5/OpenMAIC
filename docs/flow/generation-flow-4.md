# 课程生成管线 · 第 4 册：Agent 画像（agents）

<!--
  职责：详解生成管线**第 4 步「Agent 画像」**——大纲确认以后，这一步是
  `generating` 阶段的第一个步。它要么让 `course.agents` 槽的模型按课程标题 +
  场景大纲"造一支教师队伍"（恰好 1 个 teacher、若干 assistant/student，
  每人配 avatar / color / priority / voiceDesign / voice），要么直接按用户
  选的预设 Agent id 从注册表取人；然后把结果变成 `GenerationRunAgentsResult`
  写进 run 的 `agents` 列，并**在这里第一次构造出 `Stage`**（课程文档的骨架），
  供第 5/6/7 步与最终入库使用。
  覆盖：`case 'agents'` 的提交语义、`resolveAgents` 两条分支与 presets 回落、
  `withDeadline` 的 120 s 预算、`agent-profiles` 步骤 prompt 的三种形态、
  advertised voice token 与教师声线固定、三项校验、未知 Agent id 的前置拒绝、
  `agents` 事件与花名册揭示动画、失败链、常量、坑与验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 3 册 大纲生成与审查](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-3.md)｜下一步：[第 5 册 场景内容](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md)
- 源文件：[steps/agent-profiles.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts)
  · [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)
  · [agents/registry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/agents/registry.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `agents`（[plan.ts#L36-L37](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L36-L37)：与 4 个前置步一样，id 就是 kind） |
| 是否必经 | **必经**，且是 `generationSteps()` 返回数组的**第 0 项**——大纲确认以后跑的第一个步，见 [plan.ts#L56-L64](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L56-L64) |
| 所处 run 状态 | `generating`（[plan.ts#L67-L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L71) 的 `phaseOfStep`：除 material/research/outline 以外一律 `generating`） |
| 检查点写在哪 | `generation_run_steps(run_id, 'agents')`，值是 `AgentsOutput`（`{ agents, stage }`）；**同时**把 `GenerationRunAgentsResult` 抄进 `generation_runs.agents` 列 |
| 时间预算 | 120 s（[deadline.ts#L18](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L18) 的 `STEP_DEADLINES_MS.agentProfiles`）——但传给 `withDeadline` 的 label 是 `'agents'`，超时错误文案是 `agents did not finish within 120 s` |
| 重试 | **一层都没有**：步骤内部零重试，外层也不套 `withRouteRetry`。失败就地回落成预设队伍 |
| 失败后果 | 几乎不会让 run `paused`：只有"run 被中止"和"宿主判定不可重试"两种情况会往上抛（[engine.ts#L942-L951](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L942-L951)） |
| UI 显示 | `agent-generation`，而且**只有 `mode === 'auto'` 时这一位才存在**，见 [preview-steps.ts#L22-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L22-L33) 与 [page.tsx#L95-L101](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L95-L101) |
| 产出消费方 | 第 5 步内容（`agents().agents.agents` 进 prompt，[engine.ts#L517](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L517)）、第 6 步动作（[#L727-L728](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L727-L728)）、第 7 步旁白（选 narrator、取 `stageId`，[#L994-L999](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L994-L999)），以及后面所有用到 `agents().stage.id` 的地方 |
| 独有之处 | **课程文档 `Stage` 在这里出生**：`stageId` 与 `videoManifest` 都在本步定下来，之后只有第 7 步往里填场景 |

### 0.2 一句话本质

> **第 4 步 = 一道"选人"题，而不是一次生成。**
> 它有两个答案：让模型按大纲现编一支 3~5 人的队伍（含每人一副嗓子），
> 或者从注册表里把用户点名的人取过来。两个答案都会变成一个 `Stage` 骨架
> 和一份 `GenerationRunAgentsResult`；而模型那条路**一旦编不出来就立刻换第二条路**，
> 所以这一步是全管线里唯一"失败也不算失败"的步。

### 0.3 执行时序

```
run.state = generating, run.step = 'agents'
        │
        ├─ advanceRun() → generationSteps(N)[0] = agents   ← 大纲确认后的第一个步
        ▼
runStep(step) 的 switch → case 'agents'
        │
        ├─ outline() ─→ { outlines, languageDirective, courseTitle, taskEngineMode }
        ├─ name = courseTitle || topicFromRequirement(input.requirement)
        │
        ├─ input.agents.mode === 'auto' ?
        │      ├─ 是 → services.narrationTarget(owner)
        │      │        advertisedVoices(target) / narratorVoiceForGeneration(target, input.voice)
        │      │        withDeadline('agents', 120s, signal,
        │      │             services.agentProfiles → generateAgentProfiles)
        │      │              ├─ 成功 → { agents, agentIds, generatedAgentConfigs }
        │      │              └─ 抛错 → isAbortError? ─→ rethrow（run 中止）
        │      │                       isNonRetryableHostFailure? ─→ rethrow（run paused）
        │      │                       否则 log.warn → presets(选过的 id ?? 默认 3 人)
        │      └─ 否 → presets(input.agents.agentIds ?? 默认 3 人)
        │              └─ services.presetAgents = resolveAgentsForOwner(owner, ids)
        │                  （未知 id 在 start 时就被 400 拒掉了）
        │
        ├─ 构造 Stage：id = 材料分析给过的 stageId ?? generateClassroomId()
        │              agentIds / generatedAgentConfigs? / videoManifest
        │
        └─ done(value) → 检查点 { id:'agents', output }
                        patch { agents }
                        events [ step_completed('agents'), agents({agents}) ]
```

---

## 1. 引擎入口：`case 'agents'` 只有 14 行

[engine.ts#L662-L675](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L662-L675)

```ts
      case 'agents': {
        const resolved = await resolveAgents();
        return done(resolved, {
          patch: { agents: resolved.agents },
          events: [
            {
              type: 'agents',
              data: {
                agents: resolved.agents.generatedAgentConfigs ?? resolved.agents.agents,
              },
            },
          ],
        });
      }
```

四个值得停下来的点：

1. **没有 `withRouteRetry`，也没有 `commit` 之外的中间事件。** 第 3 步为了流式一路发 `outline_item`，第 5/6 步为并行场景做了不少编排，而这一步从进入 `case` 到 `return done(...)` 中间不发任何事件——它是原子的：要么没有 Agent，要么一整支。
2. **`done()` 是"提交"的缩写。** [engine.ts#L592-L596](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L592-L596)：

   ```ts
    const done = (value: unknown, extra: Partial<StepCommit> = {}): StepCommit => ({
      step: { id: stepId, output: value },
      patch: extra.patch,
      events: [{ type: 'step_completed', data: { step: stepId } }, ...(extra.events ?? [])],
    });
   ```

   所以 `agents` 事件**永远排在** `step_completed{step:'agents'}` 之后；检查点（`value`）、run 的 `agents` 列（`patch`）与事件由同一次 `commit` 原子落库。
3. **`patch.agents` 与事件 data 不是同一个东西。** 写进数据库的是完整的 `GenerationRunAgentsResult`（含 `customAgents`、`generatedAgentConfigs`、`stage` 需要的 `agentIds`）；广播给前端的只是**一张花名册**，而且是 `generatedAgentConfigs ?? agents` 二选一——见 §10。
4. **返回值 `resolved` 就是 `AgentsOutput`。** [engine.ts#L254-L257](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L254-L257)：

   ```ts
   interface AgentsOutput {
     agents: GenerationRunAgentsResult;
     stage: Stage;
   }
   ```

   下游用 [engine.ts#L459](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L459) 的 `const agents = () => output<AgentsOutput>('agents')!;` 读回它——**注意那个 `!`**：它假定本步已完成，因为它永远是 `generating` 阶段的第一个步。

---

## 2. `resolveAgents`：数据来源与两条分支

主体在 [engine.ts#L874-L976](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L874-L976)。

### 2.1 输入全部来自第 3 步的确认结果

```ts
  const resolveAgents = async (): Promise<AgentsOutput> => {
    const { outlines, languageDirective, courseTitle, taskEngineMode } = outline();
    const name = courseTitle || topicFromRequirement(input.requirement);
```

| 取用的字段 | 来自哪里 | 本步怎么用 |
|---|---|---|
| `outlines` | `run.outline`（确认过的那份，不是检查点） | 每条的 `title` + `description` 拼成 prompt 里的 "Scene outlines"；同时喂给 `buildVideoManifestFromOutlines` |
| `languageDirective` | 同上（第 3 步归一化产物） | 原样进 prompt，要求 Agent 名字/persona 遵循它 |
| `courseTitle` | 同上（模型给的或用户指定的课程名） | `name` 的首选 |
| `taskEngineMode` | 同上 | 写进 `Stage.taskEngineMode` |

`outline()`（[engine.ts#L455-L458](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L455-L458)）在 run 没有确认过的大纲时**抛错**：`Run ${run.id} has no confirmed outline`。这条路径在本步正常不可达（`generating` 状态本身就意味着大纲已确认），但 takeover 到一份被外部改坏的 run 时会走到。

课程名的兜底函数 `topicFromRequirement`（[engine.ts#L210-L213](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L210-L213)）做的是最朴素的事：trim 用户的 `requirement`，超过 500 字符就截成前 500 个再加 `'...'`。**它不看大纲**，所以只有第 3 步没给 `courseTitle` 时用户才会看到这个"需求原文当标题"的效果。

### 2.2 `presets` 闭包：把注册表 Agent 降形成 run 的结果

[engine.ts#L877-L905](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L877-L905)

```ts
    const presets = async (agentIds: readonly string[]): Promise<GenerationRunAgentsResult> => {
      const configs = await services.presetAgents(owner, agentIds);
      return {
        agents: configs.map((agent) => ({
          id: agent.id, name: agent.name, role: agent.role, persona: agent.persona,
        })),
        agentIds: configs.map((agent) => agent.id),
        ...(configs.some((agent) => !agent.isDefault) ? { customAgents: /* 9 字段 */ } : {}),
      };
    };
```

三个要点：

- `services.presetAgents` 就是 `resolveAgentsForOwner` 本身（[services.ts#L462](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L462)），见 §9。
- **`agents` 字段只保留 4 个 key**（`id/name/role/persona`），这正是 `AgentInfo` 的形状（[pipeline-types.ts#L8-L13](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/pipeline-types.ts#L8-L13)）——它是给第 5/6 步的 prompt 用的"最小可用画像"。
- **`customAgents` 只在队伍里有非内置成员时才出现**（`configs.some(a => !a.isDefault)`），且带 9 个字段（多 `avatar/color/priority/voiceConfig?/voiceDesign?`）。原因写在类型注释里（[types.ts#L136-L140](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L136-L140)）：*a custom teacher narrates with its own voice, as in the browser*——第 7 步要拿它去配声音，而内置 Agent 的声音由注册表自己决定，不需要抄进 run。

### 2.3 `GenerationRunAgentsResult` 全文

[types.ts#L128-L141](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L128-L141)

```ts
/** The agents a run teaches with, as the agents step resolved them. */
export interface GenerationRunAgentsResult {
  /** What the content and actions steps receive. */
  agents: AgentInfo[];
  /** The stage's `agentIds`. */
  agentIds: string[];
  /** The generated roster, embedded in the stage; absent for preset agents. */
  generatedAgentConfigs?: GeneratedAgentConfig[];
  /** The owner's custom agents among the presets, ... */
  customAgents?: GenerationRunCustomAgent[];
}
```

`generatedAgentConfigs` 与 `customAgents` 互斥地出现：auto 成功只有前者，preset 有自定义成员只有后者。`GenerationRunCustomAgent` 是 `AgentConfig` 的 `Pick<>`（[types.ts#L114-L126](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L114-L126)），而 `GeneratedAgentConfig` 定义在 DSL 包里（[stage.ts#L117-L129](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/dsl/src/stage.ts#L117-L129)）——两者的字段几乎一样，差别在 `GeneratedAgentConfig` 的字段都是必填（`avatar`/`color`/`priority`），而它落在 `Stage.generatedAgentConfigs` 里，属于课程文档的一部分。

---

## 3. auto 分支：一次带预算的调用，加一段"三层筛子"的 catch

### 3.1 调用现场

[engine.ts#L907-L941](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L907-L941)

```ts
    if (input.agents.mode === 'auto') {
      try {
        const target = await services.narrationTarget(owner);
        const profiles = await withDeadline(
          'agents',
          STEP_DEADLINES_MS.agentProfiles,
          signal,
          (callSignal) =>
            services.agentProfiles(
              owner,
              {
                stageInfo: { name, description: '' },
                sceneOutlines: outlines.map((o) => ({ title: o.title, description: o.description })),
                languageDirective,
                availableAvatars: AGENT_AVATARS.map((a) => a.path),
                avatarDescriptions: AGENT_AVATARS.map((a) => ({ path: a.path, desc: a.desc })),
                availableVoices: target ? advertisedVoices(target) : [],
                narratorVoice: target ? narratorVoiceForGeneration(target, input.voice) : undefined,
              },
              { log, signal: callSignal },
            ),
        );
        result = {
          agents: profiles.map((agent) => ({ id, name, role, persona })),
          agentIds: profiles.map((agent) => agent.id),
          generatedAgentConfigs: profiles,
        };
```

注意 `stageInfo.description` 被**写死成空串**：本步的 prompt 永远拿不到课程描述（第 3 步的大纲里有 `description`，但那给的是场景，不是课程）。这也是为什么 `agent-profiles.ts` 里 `stageInfo.description ? ... : ''` 那一行在管线里恒为假。

`withDeadline`（[deadline.ts#L43-L70](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L43-L70)）做的是"两个信号取其一"：`AbortSignal.any([runSignal, timeout.signal])`，超时抛 `StepTimeoutError`（`statusCode = 504`），run 自身中止则抛 run 的 reason。它的注释解释了预算的来源——*"the budget the step's 1.1.x API route declared, so a hung provider fails the step the way a route timeout failed the browser's request, as a retryable failure"*。

`target` 是 TTS 侧的东西，[services.ts#L491-L499](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L491-L499) 里 `narrationTarget` 在 `SlotDisabledError` / `SlotUnassignedError` 时返回 **null**。于是有一个重要结论：**没配 TTS 槽也能生成画像**，只是 `availableVoices = []`、`narratorVoice = undefined`，产出的 Agent 全部没有 `voiceConfig`（第 7 步会退化成用 `voiceDesign` 现场合成或默认声线）。

### 3.2 catch 的三层筛子

[engine.ts#L942-L951](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L942-L951)

```ts
      } catch (error) {
        if (isAbortError(error)) throw error;
        // A host failure no retry helps pauses the run here instead.
        if (isNonRetryableHostFailure(error)) throw error;
        // As the browser does: the learner's selected preset agents teach.
        log.warn(`run ${run.id}: agent generation failed, falling back to presets:`, error);
        // Never an empty roster: without a selection, the browser's default one.
        const selected = input.agents.presetAgentIds ?? [];
        result = await presets(selected.length > 0 ? selected : DEFAULT_PRESET_AGENT_IDS);
      }
```

| 层 | 判据 | 结果 |
|---|---|---|
| 1 | `isAbortError`（`name === 'AbortError'`，含 `DOMException`，见 [generation-retry.ts#L64-L75](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L64-L75)） | 往上抛——这是 run 被取消/租约丢失，**不是**画像失败，不该被吞 |
| 2 | `isNonRetryableHostFailure`（宿主 `classifyFailure` 说 `retryable: false`，[runtime.ts#L208-L210](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-run-hooks/runtime.ts#L208-L210)） | 往上抛——run `paused`，注释解释了为什么：*"A host failure no retry helps pauses the run here instead."*（换一支队伍也解决不了密钥/配额问题） |
| 3 | 其他一切（JSON 解析失败、teacher 数量不对、模型报错、**120 s 超时**） | `log.warn` + 回落 `presets` |

三个易被忽略的事实：

- **`StepTimeoutError` 落在第 3 层。** 它的 `name` 是 `'StepTimeoutError'` 而不是 `'AbortError'`，所以 120 s 超时**不会**中止 run，只会悄悄换成预设队伍——用户看到的效果是"AI 没编出角色，用了默认三人组"。
- **回落用的是 `input.agents.presetAgentIds`**，而不是本次生成出来的东西。也就是说：`auto` 模式**允许**带一组预设 id（§8 的输入形状），它同时充当"若模型失败该用谁"的答案。
- **`DEFAULT_PRESET_AGENT_IDS = ['default-1', 'default-2', 'default-3']`**（[engine.ts#L207](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L207)）：AI 老师 + AI 助教 + 显眼包。注释 *"Never an empty roster"* 是硬要求——第 5/6/7 步都假定了至少有一个 `role === 'teacher'`。

---

## 4. preset 分支与 `Stage` 的诞生

### 4.1 preset 分支

[engine.ts#L952-L957](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L952-L957)

```ts
    } else {
      // No agents selected: the default presets, as the learner's selection
      // starts out.
      const selected = input.agents.agentIds;
      result = await presets(selected.length > 0 ? selected : DEFAULT_PRESET_AGENT_IDS);
    }
```

这条路**一次 LLM 调用都没有**，所以也不受 120 s 预算约束、不会超时、不会回落（除了注册表抛错——而那在 start 阶段就被挡住了，见 §8）。`mode: 'preset'` 的 run 在 UI 上连 `agent-generation` 这一位都不显示。

### 4.2 `Stage`：本步真正的第二个产出

[engine.ts#L958-L975](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L958-L975)

```ts
    const now = Date.now();
    const stage: Stage = {
      id: output<MaterialOutput>('material-analysis')?.stageId ?? generateClassroomId(),
      name,
      description: '',
      style: 'professional',
      createdAt: now,
      updatedAt: now,
      interactiveMode: input.interactive,
      taskEngineMode,
      languageDirective,
      agentIds: result.agentIds,
      ...(result.generatedAgentConfigs ? { generatedAgentConfigs: result.generatedAgentConfigs } : {}),
      videoManifest: buildVideoManifestFromOutlines(outlines),
    };
    return { agents: result, stage };
```

| 字段 | 来源 | 备注 |
|---|---|---|
| `id` | 材料分析检查点里已经定过的 `stageId`，否则 `generateClassroomId()` | **id 在本步之前就可能存在**（用户上传材料时已生成），这样素材资产能提前归属到"将要存在的课程"；`agents` 步复用它是为了避免两个 id 并存 |
| `name` | `courseTitle \|\| topicFromRequirement` | 与 §2.1 一致 |
| `description` | `''` | 管线**从不填它** |
| `style` | `'professional'` | 写死 |
| `interactiveMode` | `input.interactive` | 影响第 5 步能生成哪些场景类型 |
| `taskEngineMode` | 大纲归一化后的值 | 见第 3 册 |
| `languageDirective` | 同上 |  classroom 运行时也要用 |
| `agentIds` | 本步的 `result.agentIds` | 注意：这是**注册表 id**（`default-N` / owner 自定义 id）或 `gen-xxxx`（生成花名册） |
| `generatedAgentConfigs` | 仅 auto 成功时 | 编辑器/运行时靠它把 `gen-*` 这类"不在注册表里的 id"解析成人 |
| `videoManifest` | `buildVideoManifestFromOutlines(outlines)` | 媒体泳道（第 7 册）读它 |

这一步之后，`stage.id` 就成了整门课的身份证：[#L727](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L727)、[#L999](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L999)、[#L1265](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1265)、[#L1295](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1295)、[#L1508](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1508)、[#L1676](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1676) 全都取 `agents().stage.id`。**课程文档本身要到第 7 步第一个场景落库时才写出去并发 `course_created`**，本步只是在内存/检查点里把骨架搭好。

### 4.3 `AGENT_AVATARS`：可选头像只有 12 个

[engine.ts#L155-L204](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L155-L204) 是一张写死的表，12 项成对出现（每个形象有一个常规版和一个 `-2` 变体）：

| # | path | 语义 |
|---|---|---|
| 1 / 2 | `teacher`、`teacher-2` | 权威教师 |
| 3 / 4 | `assist`、`assist-2` | 助教 |
| 5 / 6 | `clown`、`clown-2` | 显眼包 |
| 7 / 8 | `curious`、`curious-2` | 好奇宝宝 |
| 9 / 10 | `note-taker`、`note-taker-2` | 笔记员 |
| 11 / 12 | `thinker`、`thinker-2` | 思考者 |

传给 prompt 的是 `{ path, desc }`（desc 是英文），而模型必须**回填 `path`**（prompt 里明说 *Use the "path" value as the avatar field in the output*，[agent-profiles.ts#L239](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L239)）。12 项也解释了"3~5 人队伍"这个建议值：模型不会撞上"头像不够分"的硬约束。

---

## 5. `services.agentProfiles` 与它用的模型

[services.ts#L455-L460](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L455-L460)

```ts
  async agentProfiles(ownerId, input, ctx) {
    return generateAgentProfiles(
      { ...input, model: await stageModel(ownerId, 'agent-profiles') },
      ctx,
    );
  },
```

| 项 | 值 |
|---|---|
| LLM stage 名 | `'agent-profiles'` |
| 映射到的槽 | `'course.agents'`（[model-slots.ts#L103](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L103)） |
| 槽定义 | `{ id: 'course.agents', capability: 'chat', parent: 'llm' }`（[model-slots.ts#L61](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L61)） |
| 与第 3 步的区别 | 第 3 步是 `course.outline`；两者都继承全局 `llm`，可分别覆盖 |

`stageModel` 出来的 `StepLanguageModel` 里带 `model / modelString / thinkingConfig / serverManaged` 四件套，`generateAgentProfiles` 一次性解构使用（[agent-profiles.ts#L156](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L156)）。`serverManaged` 决定这次调用是走服务端托管的 provider 解析还是直接实例——细节见总览册的 LLM 层。

---

## 6. `generateAgentProfiles`：同一个 prompt 的三种形态

文件只有 369 行，一个导出函数，但它的 prompt 是**条件拼装**的：有无声音目录、教师声线是否已固定，会得到三段完全不同的文本。头注释（[agent-profiles.ts#L1-L7](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L1-L7)）把职责说得很清楚：

> Agent profiles: generate the classroom's agents (one teacher, plus assistants and students) for a course from its name and scene outlines, **binding each to an advertised voice**. **The teacher speaks with the user's narrator voice whenever that voice is usable.** The logic of 1.1.x's POST /api/generate/agent-profiles.

### 6.1 场景摘要与系统提示

[agent-profiles.ts#L160-L172](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L160-L172)

```ts
  const sceneSummary = sceneOutlines?.length
    ? sceneOutlines.map((s, i) => `${i + 1}. ${s.title}${s.description ? ` — ${s.description}` : ''}`).join('\n')
    : null;

  // The teacher must speak with the user's explicitly selected global voice
  // whenever it is usable; otherwise keep the old behavior (LLM-assigned).
  const narratorBinding = narratorVoice ? resolveNarratorVoice(narratorVoice, availableVoices) : undefined;

  const systemPrompt = `You are an expert instructional designer. Generate agent profiles for a multi-agent classroom simulation. Decide the appropriate number of agents (typically 3-5) based on the course content and complexity. Return ONLY valid JSON, no markdown or explanation.`;
```

`systemPrompt` 只提"数量 3~5"和"只回 JSON"两件事；**硬约束全在 userPrompt 里**（下一节）。

### 6.2 gender 规则：目录里没有男女就不拼这段

[agent-profiles.ts#L174-L187](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L174-L187)

```ts
  const binaryGenderListed = availableVoices?.some(
    (voice) => voice.gender === 'male' || voice.gender === 'female',
  );
  // Neutral-only catalogues (google-tts, VoxCPM voxcpm:auto) have nothing to
  // match. Asking the model to align identity, name, and persona would collapse
  // them onto "neutral".
  const genderMatchRules = binaryGenderListed ? `  - The voice's gender must match ...` : '';
```

四条规则（有 male/female 时才加）：声音性别必须匹配 `voiceDesign.gender`；必须匹配 `voiceDesign.identity`；名字与 persona 要跟该性别一致；**只有标了 male/female 的声音参与匹配，neutral 或未标性别的声音不算错配，也不得因此改动 Agent 的名字/persona/identity**。这段注释是全套 prompt 里最容易被忽略的一处设计——它防止 google-tts 这类目录把所有角色压成"中性"。

### 6.3 声音列表与 voicePrompt 的三形态

[agent-profiles.ts#L189-L213](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L189-L213)

```ts
  const voiceListStr = availableVoices && availableVoices.length > 0
    ? JSON.stringify(availableVoices.map((v) => ({
        id: advertisedVoiceToken(v), name: v.voiceName,
        language: v.voiceLanguage || 'unknown', ...(v.gender ? { gender: v.gender } : {}),
      })))
    : '';

  const voicePrompt = voiceListStr
    ? narratorBinding
      ? `- The teacher agent's voice is FIXED to the narrator voice "..." — ... omit the "voice" field for the teacher\n  - Every OTHER agent must still be assigned a voice ...`
      : `- Each agent should be assigned a voice that matches their persona from this list: ...`
    : '';
```

| 形态 | 条件 | 关键句 |
|---|---|---|
| A（无目录） | `availableVoices` 为空/未传（没配 TTS 槽） | `voicePrompt = ''`，`voiceJsonField = ''`——**schema 里连 `voice` 字段都不出现** |
| B（教师声线已钉死） | 有目录且 `resolveNarratorVoice` 成功 | 教师 **FIXED**、**省略** `voice` 字段；其余每人从目录里挑，语言匹配 directive，尽量互不重复，*Never assign the fixed teacher narrator voice to any other agent* |
| C（无钉死） | 有目录但 narrator 声线不可用 | 每个 Agent 自己挑，教师也一起挑（*authoritative voice for teacher, lively for energetic student*） |

形态 B 存在的意义：**用户在全局设置里选的旁白声音必须出现在教师身上**。如果让模型自己挑，它会给教师另配一副嗓子，第 7 步的旁白就和用户在设置页听到的试听声不一致了。

### 6.4 schema 示例必须用 advertised token

[agent-profiles.ts#L215-L222](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L215-L222)

```ts
  // The schema example must always be an ADVERTISED token (findAdvertisedVoice
  // only resolves advertised voices), never the narrator token — a ghost clone
  // not in the advertised list would poison the example for non-teacher agents.
  const voiceJsonField = voiceListStr
    ? narratorBinding
      ? `,\n      "voice": "string (voice id from available list, e.g. '${advertisedVoiceToken(availableVoices![0])}'; omit for the teacher — its voice is fixed)"`
      : `,\n      "voice": "string (voice id from available list, e.g. '${advertisedVoiceToken(availableVoices![0])}')"`
    : '';
```

即：**例子里的 id 永远取目录第一项**，哪怕教师已被钉成某个 qwen 克隆。"ghost clone"指 §7.5 说的、不在 advertised 列表里的克隆声——它虽然合法（教师能用），但拿它当例子会诱导其他 Agent 也去用，而那些用法会被 `findAdvertisedVoice` 丢弃。

### 6.5 userPrompt 的硬约束清单

[agent-profiles.ts#L224-L262](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L224-L262)（节选）

```
Course name: ${stageInfo.name}
${stageInfo.description ? `Course description: ...` : ''}
${sceneSummary ? `\nScene outlines:\n${sceneSummary}\n` : ''}
Requirements:
- Decide the appropriate number of agents based on the course content (typically 3-5)
- Exactly 1 agent must have role "teacher", the rest can be "assistant" or "student"
- Priority values: teacher=10 (highest), assistant=7, student=4-6
- Each agent needs: name, role, persona (2-3 sentences ...)
- Language directive for this course: ${languageDirective}
  Agent names and personas must follow this language directive.
- Each agent must be assigned one avatar from this list: ...（带 description 时用 {path, description}）
  - Use the "path" value as the avatar field in the output
- Each agent must be assigned one color from this list: ${JSON.stringify(AGENT_COLOR_PALETTE)}
  - Each agent must have a different color
- Each agent needs a "voiceDesign" object describing their VOCAL identity (not personality), ... as short comma-free phrases:
  - "identity": gender + age + role as one short comma-free phrase in the course language
  - "gender": "male" or "female" only. These two English words, never a translation ...
  - "texture": pitch + vocal quality ...
  - "delivery": emotion + pace ...
${voicePrompt}
```

最后给出**结构模板**（`{"agents": [ {...} ]}`）——注意它是个对象包数组，而第 3 步的大纲是裸数组，两步的解析策略因此不同（这里只需一次 `JSON.parse`）。

`voiceDesign.gender` 那条"只准英文 male/female、绝不翻译"的规则值得单独记住：`voiceDesign` 其他三个字段都要求**用课程语言写**，唯独 gender 是语言无关枚举——因为下游要用它跟目录里的 `voice.gender` 比对。

---

## 7. 调用、解析与三项校验

### 7.1 一次非流式 `callLLM`

[agent-profiles.ts#L264-L279](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L264-L279)

```ts
  log.info(`Generating agent profiles for "${stageInfo.name}" [model=${modelString}]`);

  const rawResult = (
    await callLLM(
      { model: languageModel, abortSignal: ctx.signal, system: systemPrompt, prompt: userPrompt },
      'agent-profiles',
      undefined,
      thinkingConfig,
      { serverManaged },
    )
  ).text;
```

| 特征 | 本步 | 对照第 3 步 |
|---|---|---|
| 流式 | **否**（取 `.text`，整段回来） | 是（`streamObjectArray`） |
| 步骤内重试 | **无** | `MAX_STREAM_RETRIES` + 回退模型 |
| 增量事件 | **无** | `outline_item` |
| 第 4 个参数 | `undefined`（不传 responseFormat） | — |

### 7.2 去围栏 + 解析

[agent-profiles.ts#L134-L141](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L134-L141)、[#L281-L304](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L281-L304)

```ts
function stripCodeFences(text: string): string {
  let cleaned = text.trim();
  if (cleaned.startsWith('```')) {
    cleaned = cleaned.replace(/^```(?:json)?\s*\n?/, '').replace(/\n?```\s*$/, '');
  }
  return cleaned.trim();
}
```

`JSON.parse` 失败 → 记 `rawText.substring(0, 500)` 到 error 日志 + 抛 `StepRefusal('unparseable', ...)`。围栏剥离只处理**整体被 ``` 包住**的情形，不做"从夹杂解释的文本里挖 JSON"的补救——这跟 system prompt 里那句 *no markdown or explanation* 是配对的：宁可失败回落，也不猜。

### 7.3 三项校验

[agent-profiles.ts#L306-L322](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L306-L322)

| refusal | 触发条件 | 日志 |
|---|---|---|
| `'unparseable'` | `JSON.parse` 抛 | `Failed to parse LLM response as JSON: <前 500 字符>` |
| `'too-few-agents'` | `!parsed.agents` / 非数组 / **`length < 2`** | `Expected at least 2 agents, got N` |
| `'teacher-count'` | `filter(a => a.role === 'teacher').length !== 1` | `Expected exactly 1 teacher, got N` |

三者定义在 [#L42](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L42)：`type AgentProfilesRefusal = 'unparseable' | 'too-few-agents' | 'teacher-count';`。

**只有这三项。** 名字重复、color/avatar 重复、priority 越界、persona 长度、`voice` 非法——统统不校验。加上 `StepRefusal` 在引擎里等价于"普通失败"，因此**任何一项不满足都换来同一件事：换成预设三人组**，而不是"重试一次"或"部分接受"。

另外注意 prompt 说 3~5，校验只要求 ≥2：模型给 2 个人也放行（一个 teacher + 一个 student 是合法课堂）。

### 7.4 落地映射与缺省值

[agent-profiles.ts#L324-L365](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L324-L365)

```ts
  const agents = parsed.agents.map((agent, index): GeneratedAgentProfile => {
    ...
    return {
      id: `gen-${nanoid(8)}`,
      name: agent.name,
      role: agent.role,
      persona: agent.persona,
      avatar: agent.avatar || availableAvatars[index % availableAvatars.length],
      color: agent.color || AGENT_COLOR_PALETTE[index % AGENT_COLOR_PALETTE.length],
      priority: agent.priority ?? (agent.role === 'teacher' ? 10 : agent.role === 'assistant' ? 7 : 5),
      ...(voiceConfig ? { voiceConfig } : {}),
      ...(voiceDesign ? { voiceDesign } : {}),
    };
  });
```

| 字段 | 缺省策略 | 坑 |
|---|---|---|
| `id` | `gen-` + `nanoid(8)` | **每次都不同**：retry 本步会得到全新 id，旧花名册不会被"复用" |
| `avatar` / `color` | `|| 按 index 轮转`（**空串也会走缺省**） | 只在缺失时兜底，模型给了重复值不会纠正 → 两人同头像/同色是可能的 |
| `priority` | `?? 10 / 7 / 5`（注意是 **nullish**，`0` 会被保留） | 学生按 prompt 是 4~6，缺省给 5 |
| `voiceDesign` | `normalizeVoiceDesign(agent.voiceDesign)`，无则整个字段消失 | 见 §10 的第 7 步影响 |
| `voiceConfig` | 见 §7.5 | 无声音的 Agent 干脆不带该字段 |

### 7.5 声音绑定的两条路径与 advertised token

[agent-profiles.ts#L65-L132](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L65-L132)、[#L325-L349](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L325-L349)

token 形状：`providerId::[modelId::]voiceId`（`advertisedVoiceToken`）。解析 `findAdvertisedVoice(token)` 的策略是**先精确匹配 token，再退回 `(providerId, voiceId)` 两段匹配**，中间那段 model **只兼容、不覆盖**：

```ts
  // The voice is authoritative for its model. The optional model segment is
  // accepted for compatibility but never allowed to override the advertised binding.
```

于是模型就算把 model 段写错，也只会拿到目录里那条记录自己的 model（`resolveTTSModelForVoice` 的结果），不会产生"某 provider 配某 model 根本没有这个 voice"的组合。段数不是 2 或 3、或 3 段但中间为空，都直接算未命中。

教师的路径完全不同（`resolveNarratorVoice`）：

1. `providerId` 或 `voiceId` 缺失 → `undefined`（回落到形态 C，让模型自己给教师挑）。
2. 在 advertised 目录里找同 provider + 同 voice → 命中就用**目录那条的 model**。
3. 没命中，但 `providerId === 'qwen-tts'` 且 `isQwenCloneVoice(voiceId)` → **仍然接受**，并用 `resolveTTSModelForVoice` 补 model。注释给了理由：*Qwen clone IDs are account-scoped and self-contained: local profile storage is not authoritative*——克隆音色存在用户账号侧，不在服务端目录里，所以它是唯一合法的"幽灵"。
4. 其他一切 → `undefined`，宁可不钉。

非教师的路径：

```ts
    } else if (agent.voice !== undefined && agent.voice !== null) {
      const advertised = typeof agent.voice === 'string'
        ? findAdvertisedVoice(agent.voice, availableVoices) : undefined;
      if (advertised) { voiceConfig = { providerId, ...(modelId ? {modelId} : {}), voiceId }; }
      else { console.warn('[AgentProfiles] Dropped voice token not present in the advertised list:', agent.voice); }
    }
```

即：**非教师只能绑 advertised 的音色**，编造的 token 一律丢弃（只 warn，不失败）。这里是整个文件唯一一处直接用 `console.warn` 而不是 `ctx.log` 的地方——意味着这条丢弃记录**不会**带上 run/step 上下文，排查时要按 stdout 找。

---

## 8. 输入形状与"在 start 就拒掉"的前置校验

### 8.1 `input.agents` 只有两种合法形状

[input.ts#L129-L149](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L129-L149)

```ts
  if (raw.agents === undefined) agents = { mode: 'auto' };
  else if (isRecord(raw.agents) && raw.agents.mode === 'auto') {
    agents = raw.agents.presetAgentIds === undefined
      ? { mode: 'auto' }
      : { mode: 'auto', presetAgentIds: agentIdList(raw.agents.presetAgentIds, 'agents.presetAgentIds') };
  } else if (isRecord(raw.agents) && raw.agents.mode === 'preset') {
    if (raw.agents.agentIds === undefined) throw new GenerationRunRefusal(INVALID_REQUEST, '...');
    agents = { mode: 'preset', agentIds: agentIdList(raw.agents.agentIds, 'agents.agentIds') };
  } else {
    throw new GenerationRunRefusal(INVALID_REQUEST,
      'agents must be { "mode": "auto", "presetAgentIds"?: [...] } or { "mode": "preset", "agentIds": [...] }');
  }
```

| 形状 | 含义 | 缺省 |
|---|---|---|
| 字段整个省略 | `auto`（模型编队伍） | — |
| `{ mode: 'auto' }` | auto，且**不带**回落名单 | 失败回落 `DEFAULT_PRESET_AGENT_IDS` |
| `{ mode: 'auto', presetAgentIds: [...] }` | auto，失败回落这些人 | `[]` 合法（等价于用默认三人组） |
| `{ mode: 'preset', agentIds: [...] }` | 完全不调模型 | **`agentIds` 必填**，但允许 `[]`（→ 默认三人组） |

`agentIdList`（[input.ts#L80-L93](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L80-L93)）做的三件事：`MAX_PRESET_AGENTS = 20` 上限、`AGENT_ID_PATTERN = /^[A-Za-z0-9_.:-]{1,128}$/`（[#L23-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L23-L24)）、`[...new Set(ids)]` 去重（顺序保留）。

### 8.2 未知 id 在 `startGenerationRun` 就抛，而不是在本步抛

[start.ts#L95-L111](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L95-L111)

```ts
  const agentIds = agents.mode === 'preset' ? agents.agentIds : agents.presetAgentIds ?? [];
  if (agentIds.length > 0) {
    // Resolved now (built-in ids read the registry, custom ones the owner's
    // table): checked up front so a run never fails late for these reasons.
    await resolveAgentsForOwner(owner.id, agentIds);
  }
```

要点：

- **校验的是"回落名单"而不只是"名单"**。`auto` 模式的 `presetAgentIds` 同样要能被解析——它可能是这个 run 唯一真正的花名册。
- 抛出的 `UnknownAgentsError` 由 `startRefusal` 映射成 **400 `INVALID_REQUEST`**（[start.ts#L52-L57](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L52-L57)），所以创建 run 的调用方**永远看不到**一个"跑到第 4 步才炸"的 Agent id 问题。
- 这次解析的**结果被丢弃**：它只是一次校验。真正的取人发生在 `presets()` 里（会再解析一次）。这是有意的取舍——多一次便宜的读，换"绝不晚失败"。
- 顺带一提，`startGenerationRun` 还在 [#L34-L40](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L34-L40) 用 `GENERATION_MODEL_SLOTS`（全部 `course.*` + image/video/tts/webSearch）检查模型槽可用性；`course.agents` 槽被禁用同样在创建时就被拒。

### 8.3 于是本步的 `presets()` 可以假定 id 全合法

只有两种例外会让它抛：数据库读失败（自定义 Agent 表），或 run 创建后 owner 把某个自定义 Agent 删了。后者是 takeover/长跑场景里唯一现实的"晚失败"，此时 `presets` 抛出的错会沿 `runStep` 往上——见 §11。

---

## 9. 注册表：`presets()` 到底从哪里取人

[registry.ts#L50-L73](file:///D:/GITHUB_chat/OpenMAIC/lib/server/agents/registry.ts#L50-L73)

```ts
/**
 * The agents `agentIds` name, in that order: a built-in id from code, any
 * other from the owner's custom agents. Registry ids only: the generated
 * agents of a course roster (`stage.generatedAgentConfigs`) are not registry
 * agents and are not resolved here. Throws {@link UnknownAgentsError} listing
 * every id that names neither. Only built-in ids never touch the database.
 */
export async function resolveAgentsForOwner(ownerId: string, agentIds: string[]): Promise<AgentConfig[]> {
  const customIds = [...new Set(agentIds.filter((id) => !getBuiltInAgent(id)))];
  const custom = new Map<string, AgentConfig>();
  if (customIds.length > 0) {
    const queryable = await pool();
    const owner = await canonicalizeOwner(queryable, ownerId);
    for (const stored of await readOwnerAgents(queryable, owner, customIds)) {
      custom.set(stored.agent.id, customAgentConfig(stored));
    }
  }
  const unknown = customIds.filter((id) => !custom.has(id));
  if (unknown.length > 0) throw new UnknownAgentsError(unknown);
  return agentIds.map((id) => getBuiltInAgent(id) ?? custom.get(id)!);
}
```

| 行为 | 说明 |
|---|---|
| 顺序 | **严格按传入 `agentIds` 的顺序**返回（最后一行是 `agentIds.map(...)`，不是 `customIds`），所以"用户选的第一个"就是队伍的第一个 |
| 内置优先 | `getBuiltInAgent(id)` 先查，查不到才当作自定义；因此 owner **不能**用一个已占用的内置 id 覆盖内置 Agent（`/api/agents` 明确拒绝 shadow，见 `agents-route.test.ts` 的 *refuses to change, delete or shadow a built-in agent*） |
| 全内置零 IO | `customIds.length === 0` 时**完全不碰数据库**——`DEFAULT_PRESET_AGENT_IDS` 那条回落路径因此几乎不可能失败 |
| owner 归一 | `canonicalizeOwner`：匿名 owner 被认领以后读账号的 Agent（`owner-agents.pg.test.ts` 的 *moves an anonymous owner's agents on a claim*） |
| 报错形状 | `UnknownAgentsError`，`code = 'UNKNOWN_AGENTS'`，message 列出**所有**未知 id（不是第一个） |
| 生成花名册 | **不在这里解析**。`gen-*` 只活在 `Stage.generatedAgentConfigs` 里，运行时由课堂的 roster 逻辑读 |

六个内置 Agent（[built-in.ts#L13-L137](file:///D:/GITHUB_chat/OpenMAIC/lib/orchestration/registry/built-in.ts#L13-L137)，`isDefault: true`）：

| id | name | role |
|---|---|---|
| `default-1` | AI teacher | teacher |
| `default-2` | AI助教 | assistant |
| `default-3` | 显眼包 | student |
| `default-4` | 好奇宝宝 | student |
| `default-5` | 笔记员 | student |
| `default-6` | 思考者 | student |

`customAgentConfig`（[#L28-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/agents/registry.ts#L28-L33)）把存储里的开放字符串 provider 当作 `AgentConfig['voiceConfig']`，注释补了后半句：*the TTS path treats an unknown one as unbound*——也就是说自定义 Agent 绑了一个已被停用的 provider，不会在本步报错，而是在第 7 步当作"没有声音"。

---

## 10. `agents` 事件、快照字段与前端的花名册

### 10.1 事件 data 是"两张脸"的

```ts
agents: resolved.agents.generatedAgentConfigs ?? resolved.agents.agents,
```

| 分支 | data.agents 的元素形状 | 有 `avatar` 吗 |
|---|---|---|
| auto 成功 | `GeneratedAgentConfig`（11 字段，含 avatar/color/priority/voiceDesign） | ✅ |
| auto 回落 / preset | `AgentInfo`（`id/name/role/persona`） | ❌ |

### 10.2 reducer 只承认"带头像的那张脸"

[reducer.ts#L175-L183](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L175-L183)

```ts
    case 'agents': {
      const agents = Array.isArray(data.agents) ? (data.agents as GeneratedAgentConfig[]) : [];
      // Generated rosters carry an avatar; preset agents are not revealed.
      next.generatedAgents =
        agents.length > 0 && agents.every((agent) => typeof agent.avatar === 'string')
          ? agents
          : null;
      return next;
    }
```

判定用的是"**每一个**元素都有字符串 avatar"，而不是"有一个"。所以一条混着两种形状的脏事件会得到 `null`（清空卡片），而不是部分显示。对应测试：`reducer.test.ts` 的 *keeps generated rosters for the agent cards and not preset agents*（[#L146-L158](file:///D:/GITHUB_chat/OpenMAIC/tests/generation-run-client/reducer.test.ts#L146-L158)）。

### 10.3 刷新以后还在（跟 research 来源不一样）

[reducer.ts#L22-L36](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L22-L36) 从快照重建视图：

```ts
  const generated = snapshot.agents?.generatedAgentConfigs;
  ...
    researchSources: [],
    generatedAgents: generated && generated.length > 0 ? generated : null,
```

| 字段 | 快照里有吗 | 刷新后 |
|---|---|---|
| `researchSources` | ❌（只在 `research_sources` 事件里） | 丢 |
| `generatedAgents` | ✅（`generation_runs.agents` 列 → `snapshot.agents`） | **保留** |

另外断线重连的合并逻辑 [#L93](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts#L93) 写的是 `next.generatedAgents ?? current.generatedAgents`：新值为 `null` 时**保留旧值**，避免一次 preset 分支的事件把已经显示的花名册抹掉。

### 10.4 UI：这一位可能整列不存在，也可能什么都不显示

| 事实 | 出处 |
|---|---|
| `agent-generation` 位**仅在 `view.input.agents.mode === 'auto'` 时列出** | [preview-steps.ts#L22-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L22-L33) + [page.tsx#L95-L101](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L95-L101) |
| `generating` 状态下无论 step 是什么都归到 `agent-generation`（含 step 为 null 的空档） | [preview-steps.ts#L35-L48](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L35-L48) |
| 动画是 3 张"带问号的卡"来回摆动，**不吃任何数据**（纯装饰） | [visualizers.tsx#L344-L368](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L344-L368) |
| 花名册真正出现是在 `AgentRevealModal`：延迟 400 ms 开第一张，之后每 500 ms 开一张，全开完 600 ms 后回调 | [agent-reveal-modal.tsx#L55-L96](file:///D:/GITHUB_chat/OpenMAIC/components/agent/agent-reveal-modal.tsx#L55-L96) |
| 自动弹出条件：有 `generatedAgents`、`mode === 'auto'`、且 `stageId` 还不存在（课程未落地） | [page.tsx#L166-L173](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L166-L173) |
| 弹出期间**不许跳到教室**（`agentRevealPending` 挡住 `router.replace`） | [page.tsx#L178-L182](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L178-L182) |
| 错过自动弹出后，还能在"AI 工作中"那行旁边点 `viewAgents` 按钮手动打开 | [page.tsx#L562-L570](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L562-L570) |

注意最后一行与第一行的组合：如果 run 很快完成（`stageId` 已写入），自动弹出条件不成立，用户只能通过按钮看到花名册。

### 10.5 与第 7 步的耦合：谁来讲旁白

[engine.ts#L990-L999](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L990-L999)

```ts
    // The narrator as the browser's registry finds it: the built-in agents
    // first, then the owner's custom agents, then the course's generated roster.
    const teacher = pickNarratorAgent([
      ...builtInAgents(),
      ...(agents().agents.customAgents ?? []).map((agent) => rosterAgent(agent, false)),
      ...(agents().agents.generatedAgentConfigs ?? []).map((agent) => rosterAgent(agent, true)),
    ]);
    const bound = teacher?.voiceConfig;
```

`pickNarratorAgent`（[agent-voice-options.ts#L47-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/agent-voice-options.ts#L47-L53)）三级优先：**带 `voiceConfig` 的 teacher > 带 `voiceDesign` 的 teacher > 任意 teacher**。因此：

| 本步的结果 | 第 7 步拿到的 narrator |
|---|---|
| auto 成功 + 教师钉住了 narrator 声 | 生成花名册里的 teacher（`bound` 就是用户全局声音） |
| auto 成功 + 无 TTS（形态 A） | 生成花名册里 role=teacher 的那个，`bound` 为 undefined → 只有 `voiceDesign` 可用来合成 |
| preset `default-1` | **内置 `default-1`**（列表第一项就命中）——生成花名册根本不参与 |
| preset 只选了 student（如 `default-3`） | 内置列表里**仍然**有 `default-1` 这个 teacher，于是旁白用它，而不是用户选的队伍 |

第四行是本步与第 7 步之间最不显眼的一处语义：**旁白教师不受 `agentIds` 约束**，它永远能在内置表里找到一个 teacher。`rosterAgent`（[#L348-L360](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L348-L360)）只是把花名册条目补成 `AgentConfig` 的形状（`allowedActions: []`、时间戳 0、`isDefault: false`，生成的再加 `isGenerated: true`），不改变 role/voiceConfig。

---

## 11. 失败链：这一步为什么几乎从不让 run 暂停

### 11.1 四层

| 层 | 发生什么 |
|---|---|
| **prompt 层** | 模型不听话（带解释、给 1 个 teacher、给 2 个 teacher、color 重复……） |
| **步骤层** | `stripCodeFences` + `JSON.parse` + 三项校验 → `StepRefusal('unparseable'/'too-few-agents'/'teacher-count')`；provider 报错 / 120 s 超时 → 普通 Error / `StepTimeoutError(504)` |
| **引擎层** | `resolveAgents` 的 catch **把它们全吃掉**（除 abort 与宿主不可重试），换成 `presets(...)`；`presets` 自己抛的错才继续往上 |
| **run 层** | 出 `runStep` 的错被包成 `StepFailedError('agents', cause)`（[engine.ts#L1827-L1839](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1827-L1839)），最终 `pause('agents', message, failureCodeOf(cause))` + `log.warn(... pausing)`（[#L1860-L1868](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1860-L1868)），run 停在 `paused` |

### 11.2 真能走到 run 层只剩三条路

| 路 | 错误 | 失败码 |
|---|---|---|
| A | run 被取消 / 租约丢失（`isAbortError` 或 run signal 的 reason） | **不 paused**，返回 `'interrupted'`，等 takeover |
| B | 宿主 `classifyFailure` 说 `retryable: false`（例如平台配额、密钥失效） | 宿主给的 `errorCode` |
| C | `presets()` 抛：owner 自定义 Agent 被删（`UnknownAgentsError`）、数据库故障、`stageModel` 之外的服务故障 | `runFailureCode` 走完所有分支 → **`INTERNAL_ERROR`**（`UnknownAgentsError` 没有 HTTP status） |

对照第 3 步（`StepRefusal` 一定会让 run `paused`），这是本步最重要的差别：**同一个 `StepRefusal`，在第 4 步只是日志里一行 warn**。

### 11.3 暂停以后 Retry 会怎样

- `stateForRetry('agents')` = `phaseOfStep(agents)` = **`generating`**（[plan.ts#L106-L110](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L106-L110)）。
- 本步**没有** `retryOptions`（不弹"换模型"面板），也没有像 `outline` 那样的 `outline_reset` 补发事件——`restarted` 分支只针对 `step.kind === 'outline'`（[engine.ts#L1820-L1822](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1820-L1822)）。
- Retry 会重新生成一批 `gen-${nanoid(8)}` id：`run.agents` 列被 patch **整体覆盖**，`Stage.agentIds` 也换一套。已经落库的场景里如果引用了旧 id，就靠课堂侧的 fallback 兜住（第 7 册）。
- 若第 4 步已 checkpoint 成功，`advanceRun` 会跳过它——**本步不会为一门课跑两次**（除非 Retry 显式回到它）。

---

## 12. 常量与配置速查

| 名称 | 值 / 位置 | 作用 |
|---|---|---|
| `STEP_DEADLINES_MS.agentProfiles` | 120 000 ms，[deadline.ts#L18](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L18) | auto 分支那一次调用的预算（label 传的是 `'agents'`） |
| LLM stage | `'agent-profiles'` | [services.ts#L457](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L457) |
| 模型槽 | `'course.agents'`（capability `chat`，parent `llm`） | [model-slots.ts#L61](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L61)、[#L103](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L103) |
| `AGENT_AVATARS` | 12 项 `{path, desc}`，[engine.ts#L155-L204](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L155-L204) | 同时是"可选清单"和"缺省轮转表" |
| `DEFAULT_PRESET_AGENT_IDS` | `['default-1','default-2','default-3']`，[engine.ts#L207](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L207) | preset 空选择 / auto 失败回落的答案 |
| 内置 Agent 总数 | 6（`default-1`…`default-6`） | [built-in.ts#L13-L137](file:///D:/GITHUB_chat/OpenMAIC/lib/orchestration/registry/built-in.ts#L13-L137) |
| `MAX_PRESET_AGENTS` | 20，[input.ts#L23](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L23) | `agentIds` / `presetAgentIds` 长度上限 |
| `AGENT_ID_PATTERN` | `/^[A-Za-z0-9_.:-]{1,128}$/`，[input.ts#L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L24) | 不允许空格、CJK（自定义 Agent 的 id 由服务端生成，故实际安全） |
| 生成 id 前缀 | `gen-` + `nanoid(8)`，[agent-profiles.ts#L354](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L354) | 8 位随机，无冲突检查 |
| voice token 形状 | `provider::[model::]voice`，[#L65-L69](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L65-L69) | prompt 里给模型的 id、回填时的匹配键 |
| priority 缺省 | teacher 10 / assistant 7 / 其他 5，[#L361](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L361) | prompt 建议 student 用 4~6 |
| 花名册数量 | prompt 建议 3~5，校验下限 2 | [#L172](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L172)、[#L307](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts#L307) |
| reveal 节奏 | 400 ms 起、每张 500 ms、收束 600 ms、切平铺 800 ms | [agent-reveal-modal.tsx#L55-L103](file:///D:/GITHUB_chat/OpenMAIC/components/agent/agent-reveal-modal.tsx#L55-L103) |

---

## 13. 易踩坑清单

1. **这一步不是"必然调模型"**。`mode: 'preset'` 时零 LLM 调用，UI 上连那一位都没有。
2. **`agent-generation` 这一位只在 `mode === 'auto'` 出现**，用 `previewStepIds` 的返回值断言进度时别把它当固定第 3 位。
3. **模型超时不会暂停 run**：`StepTimeoutError`（504）走的是回落分支，只有 `log.warn`。想看到它必须去看日志，而不是 run 的 `error` 字段。
4. **回落用的是 `presetAgentIds`**，不是本次生成结果的一部分；`auto` 也允许带它。带不带决定了失败后用户看到的是"我选的这几位"还是"默认三人组"。
5. **`presetAgentIds: []` 等价于不带**（`selected.length > 0` 才用它），不会出现空队伍。
6. **`{mode:'preset'}` 不带 `agentIds` 是 400**，但 `agentIds: []` 是合法的默认三人组。
7. **未知 id 在 `POST /api/generation-runs` 就 400**，不会跑到第 4 步；但**创建以后被删除**的自定义 Agent 会让第 4 步 `paused` + `INTERNAL_ERROR`。
8. **`Stage.description` 永远是空串**，`stageInfo.description` 那行 prompt 在本管线里从不生效——别指望课程简介影响队伍编制。
9. **prompt 说 3~5，代码只查 ≥2**；2 人队伍合法。同时**只查 teacher 恰好 1 个**，不查 assistant/student 数量。
10. **`color`/`avatar` 重复不会被纠正**，缺省只在字段缺失（或空串）时按 `index % 12` 轮转。
11. **`priority: 0` 会保留**（用的是 `??` 而不是 `||`），别把 0 当"没填"。
12. **非教师的 voice token 只有 advertised 命中才绑**；模型编造的 token 只 `console.warn`（**注意是 `console` 不是 `ctx.log`**，没有 run 上下文）。
13. **教师的 voice 是"钉死"的**：形态 B 下 prompt 让模型**省略** `voice` 字段，代码里也**先判 role 再看 token**，模型即便给教师塞了别的音色也被忽略。
14. **qwen-tts 克隆是唯一合法的"幽灵音色"**：不在目录里也能钉给教师，因为它是账号自包含的（`isQwenCloneVoice`）。任何其他 provider 的未advertised 音色 → 教师不钉死，回落到形态 C。
15. **`voiceDesign.gender` 只准英文 male/female**，其他三个字段用课程语言——改 prompt 时别把这条"不翻译"的规则顺手翻译掉。
16. **neutral-only 的目录（google-tts、`voxcpm:auto`）会整段丢掉 gender 匹配规则**，这是刻意的，防止所有角色塌成"中性"。
17. **schema 示例的 token 永远取目录第一项**，不是教师那个固定 token——写测试时别断言示例里出现 narrator token。
18. **没配 TTS 槽也能出画像**（`narrationTarget` 返回 null → `availableVoices = []`），产物没有 `voiceConfig`；这不是错误状态。
19. **`gen-${nanoid(8)}` 每次 Retry 都换**：花名册 id 不是稳定标识，别把它当外键存到别处。
20. **`agents` 事件的 data 与 `run.agents` 列内容不同**：前者只是花名册（或 4 字段名单），后者含 `customAgents`/`generatedAgentConfigs`/`agentIds`。前端拿不到 `customAgents`（它在快照里）。
21. **preset 分支的 `agents` 事件仍然会发**（带 4 字段、无 avatar），reducer 会把它判成 `null`；也就是说**事件照发，卡片不显**。
22. **旁白教师与用户选的队伍可以无关**：`pickNarratorAgent` 的候选列表第一个就是全量内置表，preset 只选 student 时仍用 `default-1` 念旁白。
23. **`Stage` 在本步才诞生**：`stageId` 在此之前只在材料分析检查点里；本步以后所有地方都读 `agents().stage.id`。
24. **本步是 `generating` 的第一个步**：`advanceRun` 从 `generationSteps(N)[0]` 起算，所以一个还没确认大纲的 run 永远不会有 `agents` 检查点。
25. **`resolveAgents` 里 `outline()` 带 `!`**：`agents()` 用的是 `output<AgentsOutput>('agents')!`，任何在第 5/6/7 步之前的 takeover 若丢了 `agents` 检查点都会以 `undefined` 崩在这里（而不是优雅报错）。

---

## 14. 代码地图与动手验证

### 14.1 代码地图

**本步主干**

| 文件 | 看什么 |
|---|---|
| [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) | `case 'agents'` #L662-L675、`resolveAgents` #L874-L976、`done` #L592-L596、`AGENT_AVATARS` #L155-L204、`DEFAULT_PRESET_AGENT_IDS` #L207、`topicFromRequirement` #L210-L213、`AgentsOutput` #L254-L257、`agents()` #L459、`rosterAgent` #L348-L360、narrator #L990-L999 |
| [steps/agent-profiles.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/agent-profiles.ts) | 整个文件 369 行：输入 #L16-L39、token/查找 #L65-L89、narrator 解析 #L98-L132、prompt #L160-L262、调用与解析 #L266-L322、映射 #L324-L365 |
| [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) | `agentProfiles` #L455-L460、`presetAgents` #L462、`narrationTarget` #L491-L499、接口声明 #L210-L212 |
| [run/narration-voice.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts) | `advertisedVoices` #L127-L152、`narratorVoiceForGeneration` #L155-L159、`slotVoice` #L84 |
| [run/types.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts) | `GenerationRunAgentsResult` #L128-L141、`GenerationRunCustomAgent` #L114-L126、快照 `agents` #L244 |
| [run/deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | `STEP_DEADLINES_MS` #L8-L25、`StepTimeoutError` #L28-L35、`withDeadline` #L43-L70 |

**取人与入参**

| 文件 | 看什么 |
|---|---|
| [agents/registry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/agents/registry.ts) | `UnknownAgentsError` #L15-L21、`customAgentConfig` #L28-L33、`listAgentsForOwner` #L43-L48、`resolveAgentsForOwner` #L57-L73 |
| [orchestration/registry/built-in.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/orchestration/registry/built-in.ts) | 六个内置 Agent #L13-L137 |
| [run/input.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts) | 常量 #L23-L24、`agentIdList` #L80-L93、agents 形状 #L129-L149 |
| [run/start.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts) | `GENERATION_MODEL_SLOTS` #L34-L40、`startRefusal` #L52-L75、前置 agents 校验 #L95-L111 |
| [run/plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) | `generationSteps` #L56-L64、`phaseOfStep` #L67-L71、`advanceRun` #L85-L103、`stateForRetry` #L106-L110 |
| [run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts) | `runFailureCode` #L41-L65 |

**消费方与前端**

| 文件 | 看什么 |
|---|---|
| [generation-run-client/reducer.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts) | 快照 #L22-L36、`agents` 事件 #L175-L183 |
| [generation-run-client/follower.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts) | 重连时 `generatedAgents` 的保留 #L93 |
| [generation-run-client/preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) | #L22-L48 |
| [app/generation-preview/page.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx) | steps #L95-L101、reveal 条件 #L166-L182、按钮 #L562-L570 |
| [components/agent/agent-reveal-modal.tsx](file:///D:/GITHUB_chat/OpenMAIC/components/agent/agent-reveal-modal.tsx) | 揭示节奏 #L55-L103 |
| [app/generation-preview/components/visualizers.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx) | `AgentGenerationVisualizer` #L344-L368 |
| [lib/audio/agent-voice-options.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/agent-voice-options.ts) | `pickNarratorAgent` #L47-L53、`effectiveVoiceDesign` #L62-L66 |

### 14.2 可运行的验证

| 想证明的事实 | 跑哪个测试 |
|---|---|
| 教师钉住 narrator 声、其他人绑 advertised 声 | `tests/server/generation-steps/agent-profiles.test.ts` → *binds the teacher to the narrator voice and the others to advertised voices* |
| catalogue 默认音色一路传到旁白 | 同文件 → *preserves a catalogue-backed provider default through profile generation and narration* |
| gender 规则何时进 prompt、何时丢弃 | `tests/server/generation-steps/agent-profiles-voice-gender.test.ts`（7 例，含 *does not add gender-match rules when no voice is male or female*） |
| token 三段/两段解析、model 不被覆盖、示例不用幽灵 clone | `tests/server/generation-steps/agent-profiles-voice-design.test.ts`（16 例，含 *keeps the schema example token from the advertised list when the narrator is a ghost clone*） |
| 教师声线固定但其他人照样分配 | 同上 → *still assigns LLM-picked voices to non-teacher agents when the narrator voice is pinned* |
| 未知 id 的顺序/命名、全内置不碰数据库 | `tests/server/agents/agents-route.test.ts` → *resolves built-in and custom ids in order, and names every unknown one*、*resolves built-in ids without the database* |
| 输入形状与 refusal 文案 | `tests/server/generation-run/input.test.ts`（`{mode:'preset'}` 缺 `agentIds`、`['']`、`{mode:'random'}` 三种都命中 `/agentIds/` 或 `/agents must be/`） |
| auto 失败 → 回落默认三人组（run **不** paused） | `tests/server/generation-runs.pg.test.ts` → *falls back to the default presets when agent generation fails and none were selected*（#L1849） |
| preset + 编辑过的大纲 + 端到端到底 | 同文件 → *uses preset agents, confirms an edited outline, and confirms itself for headless callers*（#L547） |
| 自定义 teacher 用自己的声音念旁白 | 同文件 → *narrates a custom preset teacher with its own voice, from the owner's registry*（#L1909） |
| 创建 run 时未知 id 被 400 | 同文件 #L1340（`agentIds: ['nope']`） |
| `agents` 事件只让"带 avatar 的"进花名册 | `tests/generation-run-client/reducer.test.ts` → *keeps generated rosters for the agent cards and not preset agents* |
| `autoAgents` 决定预览位是否存在 | `tests/generation-run-client/cards-and-steps.test.ts` → *list the classic steps the run takes*（#L124-L131） |

最小手工复现（不开模型也能看完整个分支）：

1. `POST /api/generation-runs` 带 `agents: { mode: 'preset', agentIds: ['default-3'] }` + `outlineReview: 'auto'`；
2. SSE 里会看到 `outline_ready` → `outline_confirmed` → `step_started(agents)` → `step_completed(agents)` → `agents`，且 `agents` 事件**没有** avatar；
3. `GET /api/generation-runs/:id` 的 `agents` 字段里 `generatedAgentConfigs` 缺省、`agentIds = ['default-3']`、`customAgents` 缺省（因为是内置）；
4. 把 `mode` 换成 `'auto'` 并关掉 `course.agents` 槽：第 2、3 步的结果**完全一样**（槽不可用被吞成回落），但 UI 上会多出一个 `agent-generation` 位。

---

## 15. 一句话总结

> **第 4 步 = 一次"要么现编、要么点名"的选人，外加课程文档的第一块骨架。**

它把第 3 步的大纲压成一段 prompt（课程名 + 场景标题列表 + 语言指令 + 头像色板 + 声音目录），要求模型恰好给出一个 teacher；给出的是 `GenerationRunAgentsResult` + 一个 `Stage`；写库的只有 `run.agents` 列和 `agents` 检查点，广播的只有一张可能带头像的花名册。要改这一层的行为，记住三个不变量：

**任何模型侧的失败都必须被换成一支能用的队伍**——`presets` 是这条步的兜底，也是它唯一的兜底，改它等于让 run 在第 4 步开始暂停；
**教师的声音不属于模型的决定**——`narratorBinding` 在 prompt 里就要求"省略"，在映射里就先判 role，两处必须同时改；
**队伍与教室不是同一个概念**——`agentIds` 是注册表 id 或 `gen-*`，而 `Stage.generatedAgentConfigs` 不是注册表 Agent，`resolveAgentsForOwner` 明确不管它们。

