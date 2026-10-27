# 课程生成管线 · 第 7 册：旁白合成与场景入库（scene:N:narration）

<!--
  职责：详解生成管线**第 7 步（也是最后一步）「旁白」**——每个场景三连步的第三步，
  同时是**整条管线里唯一真正改动课程文档的步**。它把第 6 步装配好的完整场景拿出来，
  按 provider 的字数上限切分过长的 `speech`，逐条挑声音、调 TTS 合成音频、把字节
  存成课程资源，然后把"带 audioId 的场景"与"记录这一件事的检查点"**放进同一个数据库
  事务**写入课程；顺带把之前已经存好但还没落位的图片/视频占位符一起补齐。它不返回
  任何给前端的输出内容，只发事件。
  覆盖：`case 'narration'` 的自我提交语义、`narrationTarget` 的三种缺席情形、
  `splitLongSpeechActions` 三级切分、narrator 挑选与 `slotVoice` / `clipVoice` 的
  四层语音优先级、一条 clip 的完整调用栈（`synthesizeNarration` → `generateTTS` →
  `storeGeneratedAsset`）、无声（unvoiced）的产生与记账、`QWEN_VC_VOICE_NOT_FOUND`
  的一次换声、clip 级并发与 `releaseAssets` 回滚、`appendScene` / `placeHeldMedia` /
  `writeScene` 的事务与事件、`document.ts` 的五种写法、媒体落位与完成收尾
  （`placeMedia` / `placeIntoCurrentCourse` / `unplacedMedia` / `complete` /
  `MediaWorkPendingError`）、事件与前端、常量、坑与验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 6 册 场景动作与装配](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md)｜本册是全管线最后一册
- 源文件：[run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [steps/narration.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts)
  · [run/narration-voice.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts)
  · [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)
  · [run/document.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts)
  · [lib/audio/tts-utils.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `narration` / `scene:<index>:narration`（[plan.ts#L27-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L27-L33)） |
| 是否必经 | **必经**，三连步的**第 3 项**：`SCENE_STEP_KINDS = ['content', 'actions', 'narration']`（[plan.ts#L29](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L29)） |
| 一共跑几次 | 场景数 N 次，严格按 index 顺序（[plan.ts#L57-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L57-L65)） |
| 所处 run 状态 | `generating`；**它是把 run 从 `generating` 推进到 `completed` 的那一步** |
| 检查点写在哪 | `generation_run_steps(run_id, 'scene:<n>:narration')`，值是 `NarrationOutput = { scene }`（[engine.ts#L258-L260](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L258-L260)、[#L1163](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1163)） |
| 时间预算 | **30 s，按"每条 clip 的每次尝试"计时**（[deadline.ts#L23-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L23-L24) 的 `narrationClip`）。注意这是**全表最小**的预算，因为一个场景里有十几条 clip，每条各自计时 |
| 重试 | 与第 5/6 步同一套三层：`withRouteRetry` → `RouteStatusError.isRetryable` → `isRetryableGenerationError`（[generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L158)）；`refusalStatus` 传 **400**（[engine.ts#L1046](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1046)），即 `StepRefusal` **不重试** |
| 重试上限 | 第 1 个场景 **2 次**、其余 **5 次**（`maxAttempts = maxRetries + 1`，[retry.ts#L21-L23](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L21-L23)、[generation-retry.ts#L182-L189](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L182-L189)），且是**每条 clip 各自**这么多 |
| 失败后果 | **直接 `paused`**（同第 6 步；`skippedScenes` 只由 content 失败产生，[engine.ts#L577-L583](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L577-L583)） |
| UI 显示 | 与 actions **共用** `'actions'` 位：`stepOfRun` 里非 `content` 的场景小步一律映射到 `'actions'`（[preview-steps.ts#L42-L43](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L42-L43)） |
| 产出消费方 | 引擎自己（`previousSpeechesFor` 会读第 0 个场景的 narration 输出，[engine.ts#L760-L765](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L760-L765)）、`unplacedMedia`（[#L1636-L1640](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1636-L1640)），以及**课程文档**——本步的输出就是数据库里的场景 |
| 独有之处 | **唯一不改模型输出内容、只给它加声音的步**；唯一**自己做 commit**（`runStep` 返回 `{}`）的步；唯一**写用户可见文档**的步；唯一会**顺带把媒体占位符落位**的步；也是唯一"预算用完但场景照样入库"（unvoiced 不失败）的步 |

### 0.2 一句话本质

> 第 7 步 = **把第 6 步那串已经合法的 `speech` 文本，逐条换成能播的音频资源，然后在一个事务里把"这个场景属于这门课"这件事写进数据库。**

模型一次也没有被调用（除了 TTS）。这一步的全部风险都不在"生成得对不对"，而在**两件事必须同时成立**：字节存下来了 + 场景写进文档了。所以整册的主线是**事务边界**与**回滚半径**：

- 字节存在、场景没写 → 悬空的 pending 资源，需要 `releaseAssets` 回收；
- 场景写了、字节没了 → 一句永远播不出来的旁白，需要 `unvoiced` 记账并如实告诉前端；
- 两者都写了但没在同一个事务里 → 崩溃后重放会写出两份，或写出一个课程里没有的场景却被检查点标记为完成。

### 0.3 一个场景的旁白时序

```
runStep(case 'narration')                       engine.ts L747-L750
  └─ narrateScene(sceneIndex, stepId)           L978
       ├─ structuredClone(actions.scene)        L979-L980   ← 第 6 步的检查点
       ├─ services.narrationTarget(owner)       L981
       │     null ─────────────────────────► appendScene(无音频) L982-L985  ★短路
       ├─ splitLongSpeechActions(actions,pid)   L986
       ├─ filter speech && text                 L987-L989
       ├─ pickNarratorAgent(内建+自定义+生成)   L992-L996   → bound = teacher.voiceConfig
       ├─ slotVoice(target, input.voice)        L998        → speed
       ├─ for/of 每条 clip（或并发）：
       │     narrate(action)                    L1006-L1068
       │       ├─ clipVoice(...) → null?  unvoiced++        L1011-L1019
       │       ├─ resolveServerAgentVoiceOptions            L1021-L1026
       │       └─ withRouteRetry(                           L1028
       │            withDeadline(narrationClip = 30 s,     L1030
       │              services.narrateClip(...)           L1031-L1044
       │                └─ synthesizeNarration → generateTTS → storeGeneratedAsset
       │            , retryOptions(stepId, idx, 400))      L1046
       │          catch QWEN_VC_VOICE_NOT_FOUND → 换声重 hop≤1  L1050-L1065
       ├─ unvoiced>0 → log.warn + patch          L1093-L1095
       └─ appendScene(...)                       L1096
             exclusive(() =>                    L1114  ★写链串行
               placeHeldMedia(scene)            L1126-L1152
               writeScene(...)                  L1154-L1210
                 └─ 一个事务：commitGenerationRunIn + (createRunCourse | appendRunScene)
       catch → services.releaseAssets(allocated) L1097-L1101  ★回滚
```

---

## 1. 入口：一个"自己提交"的 case

`runStep` 的四个前置步骤都走统一出口 `done(value)`——由 `runStep` 组装 `{ step, patch, events }` 再交给调用方 commit。旁白不是。

```ts
// lib/server/generation/run/engine.ts L747-L751
      case 'narration':
        // Narration commits itself, with the scene's document write.
        await narrateScene(step.sceneIndex, stepId);
        return {};
```

这是第 7 步与前六步在结构上最大的不同，原因写在注释里：**旁白的提交动作必须和文档写入在同一个事务里**，而通用 `done()` 路径做不到——`done()` 产出的 `StepCommit` 会被外面的 `commit()` 单独执行一次事务，文档写入则是另一回事。于是旁白把 commit 的权力收进自己肚子里（见 §10 的 `writeScene`）。

两个直接后果值得记住：

1. **`runStep` 返回 `{}`**，即"本步没有 `step` 字段可写"。外层 `commit()`（[engine.ts#L412-L421](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L412-L421)）看到 `change.step` 为空就不会往 `steps` Map 里塞东西——真正塞的是 `writeScene` 自己（[#L1207-L1208](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1207-L1208)）。
2. **失败时不会有"半个检查点"**。既然提交在内部，内部事务没成就等于什么都没发生；外层的失败处理只面对"这一步没完成"，不需要考虑"文档写了但检查点没写"。

## 2. `narrateScene` 的开头：拿第 6 步的树，先问有没有嘴

```ts
// lib/server/generation/run/engine.ts L978-L986
  const narrateScene = async (sceneIndex: number, stepId: string): Promise<void> => {
    const actions = output<SceneActionsResult>(sceneStepId(sceneIndex, 'actions'))!;
    const scene: Scene = structuredClone(actions.scene) as Scene;
    const target = await services.narrationTarget(owner);
    if (!target) {
      await appendScene(sceneIndex, stepId, scene);
      return;
    }
    scene.actions = splitLongSpeechActions(scene.actions || [], target.providerId);
```

四个细节：

- `output(...)!` 带非空断言：能跑到旁白，说明 actions 检查点必然存在（三连步是严格顺序的，且 actions 失败会让整个 run `paused`）。这是引擎的**结构性不变量**，不是防御。
- `structuredClone` 是**必须的**而不是谨慎性的：下面会原地改 `action.audioId`（[#L1073](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1073)）。第 6 步的检查点树要留给 Retry 用——Retry 一个旁白步骤时，起点必须是"还没有任何音频"的那棵树。若原地改，第二次尝试就会给已经有 assetId 的动作重新分配 id，甚至把已释放的资源 id 再写回文档。
- `scene.actions || []`：`actions` 在类型上可选。空数组是合法场景（第 6 册 §10.2 提过：slide 无 text 元素时至少会有一条 speech，但 `pbl` 之类的场景理论上也可能动作很少）。
- **没有 target 就直接 `appendScene`**，`unvoiced` 保持默认 0。也就是说：**没配 TTS 的部署照样能产出一门完整课程**，只是全片静音，且**不会**被记成"无声"。无声计数只统计"有嘴但没能发声"的情形（§7.4）。

## 3. `narrationTarget`：三种"这一步没有 TTS"

```ts
// lib/server/generation/run/services.ts L490-L507
  async narrationTarget(ownerId) {
    let connection;
    try {
      connection = await resolveMediaSlot('tts', {
        workspaceId: await backgroundWorkspaceId(ownerId),
      });
    } catch (error) {
      if (error instanceof SlotDisabledError || error instanceof SlotUnassignedError) return null;
      throw error;
    }
    if (connection.providerId === 'browser-native-tts') return null;
    const providerId = connection.providerId as TTSProviderId;
    const modelId =
      connection.modelId ||
      TTS_PROVIDERS[providerId as BuiltInTTSProviderId]?.defaultModelId ||
      undefined;
    return { connection, providerId, ...(modelId ? { modelId } : {}) };
  },
```

| 情形 | 结果 | 语义 |
|---|---|---|
| `SlotDisabledError`（tts slot 被关） | `null` | 运维显式关掉配音 |
| `SlotUnassignedError`（没分配 provider） | `null` | 用户没选 TTS 模型 |
| provider 是 `browser-native-tts` | `null` | **浏览器方言不能在服务端合成**，必须留给客户端 |
| 其它解析错误 | **照抛** | 配置坏了是故障，不是"没有配音"，应让 run `paused` 而不是静默出无声课 |

返回的 `RunNarrationTarget` 形状见 [narration-voice.ts#L34-L40](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L34-L40)：`{ connection, providerId, modelId? }`。`modelId` 的回退链是"slot 自己配的模型 → provider 的 `defaultModelId` → `undefined`"，注意 `...(modelId ? { modelId } : {})` 的写法：**没有模型时这个键根本不存在**，而不是 `undefined`——后面很多 `target.modelId` 的判断依赖"键缺失"与"键为 undefined"的一致语义（`JSON.stringify` 存储时也有差别）。

> `browser-native-tts` 在两个地方被拒绝：这里返回 `null`（整个 run 静音），以及 [narration.ts#L76-L81](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L76-L81) 抛 `StepRefusal('client-side-provider')`。后者是给**直接调用合成函数的其它入口**（如浏览器路由）兜底的，run 引擎这条路径永远先在前者被拦下。

## 4. 切分长句：`splitLongSpeechActions`

```ts
// lib/audio/tts-utils.ts L11-L14
/** Provider-specific max text length limits. */
export const TTS_MAX_TEXT_LENGTH: Partial<Record<TTSProviderId, number>> = {
  'glm-tts': 1024,
};

// lib/audio/tts-utils.ts L82-L106
export function splitLongSpeechActions(actions: Action[], providerId: TTSProviderId): Action[] {
  const maxLength = TTS_MAX_TEXT_LENGTH[providerId];
  if (!maxLength) return actions;
  ...
```

**第一件要知道的事：这张表里只有 `glm-tts` 一家。** 换成 `edge-tts`、`qwen-tts`、`openai` 时 `maxLength` 是 `undefined`，函数原样返回入参**同一个数组引用**（不是拷贝）。所以"旁白会被切成小段"这件事是 provider 相关的，不是管线通用行为。文档里如果假设"一个 speech 动作就是一段音频"，在 GLM 上就会错。

切分逻辑本身是三级的（`splitLongSpeechText`，[tts-utils.ts#L21-L75](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L21-L75)）：

| 级别 | 分隔符 | 代码位置 |
|---|---|---|
| 1 句子边界 | `/(?<=[。！？!?；;：:\n])/u`（后视，标点留在前面） | [#L26](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L26) |
| 2 从句边界 | `/(?<=[，,、])/u` | [#L52](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L52) |
| 3 硬切 | `unit.slice(start, start + maxLength)` | [#L61-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L61-L65) |

装配时用 `appendUnit` 贪心地往后塞，只要 `(current + unit).length <= maxLength` 就合并（[#L43-L45](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L43-L45)），所以**切出来的是尽量填满的块**，不是"一句一块"。最后 `pushChunk(current)` 收尾（[#L73](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L73)）。

对动作数组的影响（[#L87-L104](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L87-L104)）：

```ts
    const { audioId: _audioId, ...baseAction } = action as SpeechAction;
    log.info(
      `Split speech for ${providerId}: action=${action.id}, len=${action.text.length}, chunks=${chunks.length}`,
    );
    return chunks.map((chunk, i) => ({
      ...baseAction,
      id: `${action.id}_tts_${i + 1}`,
      text: chunk,
    }));
```

- **`audioId` 被剥掉**。这是关键：原动作若已带一个音频引用，切分后那份音频只覆盖前半段文本，留着就是错的。剥掉后每一块都会被重新合成。
- **新 `id` 是 `${action.id}_tts_${i+1}`，从 1 开始**。第 6 册讲过 `id` 是 DSL 校验的必填字符串，这里保证切完仍然合法，且 `audioId` 的生成（`tts_s${scene.order}_${action.id}`，[engine.ts#L1037](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1037)）会因此天然带上 `_tts_1` 后缀。
- 只有 `chunks.length > 1` 才算切分（[#L92](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L92)），否则原样返回；`didSplit` 为 false 时返回**原数组引用**（[#L105](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts#L105)）。
- 副作用是**动作条数变了**：第 6 步检查点里是 1 条，第 7 步内存里是 3 条，最终入库的也是 3 条。`previousSpeeches`（第 6 步产出的衔接文本）不受影响，因为它是**文本**列表，不是动作列表。

## 5. 谁在说话：narrator 的挑选

```ts
// lib/server/generation/run/engine.ts L987-L999
    const speechActions = scene.actions.filter(
      (action): action is SpeechAction => action.type === 'speech' && !!action.text,
    );
    // The narrator as the browser's registry finds it: the built-in agents
    // first, then the owner's custom agents, then the course's generated roster.
    const teacher = pickNarratorAgent([
      ...builtInAgents(),
      ...(agents().agents.customAgents ?? []).map((agent) => rosterAgent(agent, false)),
      ...(agents().agents.generatedAgentConfigs ?? []).map((agent) => rosterAgent(agent, true)),
    ]);
    const bound = teacher?.voiceConfig;
    const { speed } = slotVoice(target, input.voice);
    const stageId = agents().stage.id;
```

两件事叠在一起决定声音：**谁的音色**（teacher 的 `voiceConfig`）与**哪个 provider**（slot）。

`pickNarratorAgent` 的三级优先级（[agent-voice-options.ts#L47-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/agent-voice-options.ts#L47-L53)）：

```ts
export function pickNarratorAgent(agents: AgentConfig[]): AgentConfig | undefined {
  return (
    agents.find((a) => a.role === 'teacher' && a.voiceConfig) ??
    agents.find((a) => a.role === 'teacher' && a.voiceDesign) ??
    agents.find((a) => a.role === 'teacher')
  );
}
```

它的注释解释了为什么要这么绕（[#L42-L45](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/agent-voice-options.ts#L42-L45)）：registry 永远先用 `DEFAULT` agent 兜底，一个朴素的 `find(role === 'teacher')` 会拿到**默认老师**（没有 voiceConfig 也没有 voiceDesign），从而无视真正生成的那个班级 roster。所以传进来的数组顺序（内建 → 自定义 → 生成）与这里的 `??` 顺序共同起作用——**内建老师排在前，但因为通常没有 `voiceConfig`，第一级 `find` 会跳过它命中生成 roster 里的老师**。

`rosterAgent(agent, generated)`（[engine.ts#L348-L356](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L348-L356)）把 roster 条目补成 registry 形状：`allowedActions: []`、`createdAt/updatedAt = new Date(0)`。第二个参数 `generated` 区分"生成的"与"用户自定义的"。

> 旁白**只挑一个 narrator 给整个场景用**。即使一个场景里 `speech` 由不同 agent 说出（第 6 册 §9.2：`text` 条目转成的 speech **不带 agentId**），声音也是同一个老师的。这是当前实现的一个刻意简化——`bound` 在 `narrateScene` 顶层算一次，闭包里所有 clip 共用。

## 6. slotVoice：学习者的偏好能不能用

```ts
// lib/server/generation/run/narration-voice.ts L84-L113
export function slotVoice(
  target: RunNarrationTarget,
  preference: GenerationRunInput['voice'],
): { voice: string; speed: number } {
  const usable =
    preference?.providerId === target.providerId &&
    !!preference.voiceId &&
    voiceAllowedByConfiguredCatalogue(target.providerId, preference.voiceId, target.connection.voices) &&
    voiceServesModel(target.providerId, preference.voiceId, target.modelId, target.connection.voices);
  return {
    voice: usable
      ? preference!.voiceId
      : defaultVoiceFor(target.providerId, target.modelId, target.connection.options, target.connection.voices),
    speed: preference?.speed ?? 1,
  };
}
```

四个条件全真才用学习者选的音色：同 provider、有 id、在**配置允许的目录**里、且**能为 slot 的模型服务**。任一不满足就退到 `defaultVoiceFor`（[#L69-L81](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L69-L81)）：配置默认 → `DEFAULT_TTS_VOICES[provider]` 或 `'default'`（前提是该音色能服务这个模型）→ provider 目录里第一个能服务该模型的音色 → 兜底 preferred。

`speed` 独立于音色可用性：`preference?.speed ?? 1`——**即使音色被降级，倍速偏好仍然生效**。

`providersConfig(target)`（[#L43-L66](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L43-L66)）是个值得注意的适配层：它把 slot **伪造成**一个"只有一个 provider、`enabled: true`、`isServerConfigured: true`、`apiKey: ''`"的 map，好复用浏览器那套 `voice-resolver` 纯函数。文件头注释把这件事说得很清楚（[#L2-L8](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L2-L8)）：**服务端 run 的声音决策必须和浏览器逐位一致**，否则同一门课在两种生成模式下会得到不同配音。空 `apiKey` 不会造成问题，因为真正调 provider 时读的是 `connection`，不读这份伪造 map。

## 7. 一条 clip 的完整旅程

```ts
// lib/server/generation/run/engine.ts L1006-L1026
    const narrate = async (
      action: SpeechAction,
      override?: ResolvedVoice,
      hops = 0,
    ): Promise<string | null> => {
      const chosen = clipVoice({
        target,
        preference: input.voice,
        bound,
        unavailable: unavailableBindings,
        override,
      });
      // A voice of a provider the slot does not name stays unvoiced.
      if (!chosen) return null;
      const { voice, globalVoice } = chosen;
      const providerOptions = await resolveServerAgentVoiceOptions(teacher, {
        providerId: voice.providerId,
        providerConfig: clipProviderConfig(target, voice),
        voiceId: voice.voiceId,
        language: languageDirective,
      });
```

### 7.1 `clipVoice`：四层优先级与"返回 null"

`clipVoice`（[narration-voice.ts#L175-L207](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L175-L207)）的决策链：

| 顺序 | 取哪个声音 | 代码 |
|---|---|---|
| 0 | `override`：重试时显式指定的音色 | `input.override ?? ...`（[#L192](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L192)） |
| 1 | teacher 绑定的 `voiceConfig`，**除非**它的 binding key 在 `unavailable` 集合里 | `resolveNarratorVoiceBinding(bound && unavailable.has(key) ? undefined : bound, globalVoice, configs)`（[#L193-L197](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L193-L197)） |
| 2 | 若 bound 恰好等于 slot 音（"钉住的主讲"）且该 provider 没启用 → 用 `deterministicNarratorVoice(configs)` | [#L198-L204](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L198-L204) |
| 3 | 以上都失败 → **`return null`** | `if (!isTTSProviderEnabled(voice.providerId, configs[voice.providerId])) return null;`（[#L205](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L205)） |

第 3 层就是 `unvoiced` 的**主要来源**：一个 clip 决定的音色属于 slot 没启用的 provider（在伪造 map 里只有 slot 那一家，所以任何"外部音色"都会失败）。注释写得直白（[#L172-L173](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L172-L173)）："the line stays unvoiced, as in the browser"——浏览器上这句话也是不出声的，服务端保持同样行为，**不抛错、不重试、不失败**。

### 7.2 重试与截止

```ts
// lib/server/generation/run/engine.ts L1027-L1047
      try {
        return await withRouteRetry(
          () =>
            withDeadline(stepId, STEP_DEADLINES_MS.narrationClip, signal, (callSignal) =>
              services.narrateClip(
                owner,
                {
                  target,
                  stageId,
                  text: action.text,
                  audioId: `tts_s${scene.order}_${action.id}`,
                  voice: voice.voiceId,
                  speed,
                  ...(providerOptions ? { providerOptions } : {}),
                  fence,
                },
                { log, signal: callSignal },
              ),
            ),
          retryOptions(stepId, sceneIndex, 400),
        );
```

- 嵌套顺序是 **`withRouteRetry` 在外、`withDeadline` 在内** ⇒ 30 s 是**每次尝试**的预算，一个 clip 最坏可以耗 `(maxRetries+1) × 30 s` 再加退避睡眠。
- `retryOptions(stepId, sceneIndex, 400)` 的第三参是 **`refusalStatus = 400`**，与第 6 步的 500 相反（对照 [engine.ts#L714](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L714) 附近）。含义：**旁白步骤里的 `StepRefusal` 判为不可重试**（`routeStatus` 返回 400 → `NON_RETRYABLE_STATUS_CODES` 命中 → `isRetryable: false`，见 [retry.ts#L45-L50](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L45-L50) 与 [generation-retry.ts#L130-L147](file:///D:/GITHUB_chat/OpenMAIC/packages/@openmaic/generation/src/generation-retry.ts#L130-L147)）。四种拒绝（§7.3）都是"配置问题，重试一百次也一样"，所以快速失败。
- `StepTimeoutError.statusCode = 504`（[deadline.ts#L28-L34](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L28-L34)）会经 `statusFromError` 的 `error.statusCode` 分支被读出来 → `>= 500` → **可重试**。这正是注释所说的"网关超时交给重试"。
- `audioId` 的构造 `tts_s${scene.order}_${action.id}`：**用 `order` 而不是 `sceneIndex`**。这两个值在大纲里通常一致，但语义不同——`order` 是场景自身携带的序号字段（第 6 册 §11 的装配阶段写入），`sceneIndex` 是 run 计划里的位置。日志与旧数据对齐用的是 `order`。注意它只是**日志与派生键**，返回值 `assetId` 才是最终写进 `action.audioId` 的东西。
- `fence`（[#L1004](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1004)）= `(tx) => fenceGenerationRunWriteIn(tx, lease)`：**存资源必须在持有 run 写锁的事务里做**，防止别的执行者（lease 被抢走后重开的 run）同时写。

### 7.3 `synthesizeNarration`：拒绝、音色归属、模型与记账

`services.narrateClip` 先合成再落库（[services.ts#L509-L541](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L509-L541)），合成本体在 [steps/narration.ts#L53-L171](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L53-L171)。

四种拒绝（[#L38-L46](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L38-L46)）与抛出点：

| `NarrationRefusal` | 触发条件 | 行 |
|---|---|---|
| `voice-missing` | 请求没给音色且 provider 也没默认音色 → `Missing required field: ttsVoice` | [#L71-L73](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L71-L73) |
| `client-side-provider` | `browser-native-tts` | [#L76-L81](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L76-L81) |
| `voice-context-required` | VoxCPM 的 Auto Voice 且既无 `voicePrompt` 也无 `registeredVoiceId` | [#L89-L99](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L89-L99) |
| `missing-api-key` | provider `requiresApiKey` 但没 key | [#L109-L115](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L109-L115) |

音色归属的优先级（[#L62-L69](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L62-L69)）：

```ts
  const ttsVoice =
    requestedVoice && (!requestedProviderId || requestedProviderId === ttsProviderId)
      ? requestedVoice
      : configuredDefaultVoice(connection.options, connection.voices) ||
        DEFAULT_TTS_VOICES[ttsProviderId as keyof typeof DEFAULT_TTS_VOICES] ||
        requestedVoice;
```

最后一项 `|| requestedVoice` 很微妙：**前两级都为空时，仍然用一个"本不属于这个 provider"的音色**。这是为了兼容"deprecated 请求字段"路径；在 run 引擎里 `clipVoice` 已经保证音色与 provider 匹配，所以这条分支实际不会走到。

其它要点：

- `publicOnly = connection.userEndpoint`（[#L101-L105](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L101-L105)）：用户自带 endpoint 走严格公网策略，只有运维配置才能继承 `ALLOW_LOCAL_NETWORKS`。
- Qwen 克隆音色会**强制 `speed: 1`**：`speed: qwenCloneVoice ? 1 : requestedSpeed`（[#L138](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L138)）——学习者的倍速在这里被 provider 能力覆盖。
- 模型解析分两条路（[#L124-L132](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L124-L132)）：`connection.origin === 'configuration'` 用 `slotTTSModel`，否则用 `resolveTTSModel`（可能采纳请求里的 `requestedModelId`）。run 引擎只传 `requestedVoice` 和 `speed`，**不传 `requestedProviderId` / `requestedModelId`**（[services.ts#L510-L518](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L510-L518)），因此走的是配置路径。
- `config.signal = ctx.signal`（[#L146](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L146)）：**run 丢 lease / 课程被删时能真的掐断 provider 请求**。
- 记账：`void recordGenerationUsage({ kind: 'tts', unit: 'character', quantity: text.length })`（[#L162-L168](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L162-L168)）——**按字符数**计费，`void` 表示不等它完成（记账失败不影响课程）。
- 日志一行可定位问题（[#L153-L156](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts#L153-L156)）：`Generating TTS: provider=…, model=…, voice=…, registeredVoiceId=…, audioId=…, textLen=…`。

### 7.4 落库：mimeType、容量拒绝与 `assetId`

```ts
// lib/server/generation/run/services.ts L521-L541
    const mimeType = narrationMimeType(narration.format);
    if (!mimeType) {
      throw new Error(`The TTS provider returned audio in an unknown format: ${narration.format}`);
    }
    const stored = await storeGeneratedAsset({
      ownerId, stageId: input.stageId, bytes: narration.audio, mimeType,
      kind: 'audio', fence: input.fence,
    });
    if (stored.status === 'refused') {
      ctx.log.warn(`Asset storage is full; leaving ${input.audioId} unvoiced`);
      return null;
    }
    return stored.assetId;
```

- 未知格式**抛普通 Error**（不是 `StepRefusal`）。普通 Error 没有状态码，`routeStatus` 会落到 `upstreamHttpStatus(error) ?? 500`（[retry.ts#L45-L50](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L45-L50)），于是 500 → `>= 500` → **可重试**。也就是说 provider 返回一个我们不认识的容器格式时，会白重试 5 次。
- `status === 'refused'` 是 `unvoiced` 的第二个来源：**配额满了不退钱、不失败，只让这一句静音**，与浏览器行为对齐（注释 [#L535-L536](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L535-L536)）。
- 返回值是 `assetId`，一个**不透明资源 id**。`audioId`（那个 `tts_s0_…` 字符串）只用于日志与派生键；前端播放读的是被替换后的 `assetId`（[lib/action/engine.ts#L350](file:///D:/GITHUB_chat/OpenMAIC/lib/action/engine.ts#L350)：`this.audioPlayer!.play(action.audioId || '', ...)`）。

### 7.5 每条 clip 之后：记账或静音

```ts
// lib/server/generation/run/engine.ts L1069-L1077
    const narrateOne = async (action: SpeechAction) => {
      const assetId = await narrate(action);
      if (assetId) {
        allocated.push(assetId);
        action.audioId = assetId;
      } else {
        unvoiced += 1;
      }
    };
```

`allocated` 是**回滚名单**（§8 用），`action.audioId = assetId` 是**原地改克隆树**。两条语义都要记住：

1. 只有**已成功入库**的资源才进 `allocated`。
2. `unvoiced` 只数"clipVoice 返回 null"和"store refused"两种，不数"target 不存在"。

## 8. 克隆音色没了：一次换声的 hop

```ts
// lib/server/generation/run/engine.ts L1048-L1067
      } catch (error) {
        // The bound voice's clone is gone: one retry with a different voice.
        if (
          errorCode(error) === 'QWEN_VC_VOICE_NOT_FOUND' &&
          bound &&
          hops < 1 &&
          voiceBindingKey(voice) === voiceBindingKey(bound)
        ) {
          unavailableBindings.add(voiceBindingKey(bound));
          const retryVoice = clipVoiceAfterMissingClone({
            target,
            bound,
            globalVoice,
            failed: voice,
            usedFallbackVoice: !!override,
          });
          if (retryVoice) return narrate(action, retryVoice, hops + 1);
        }
        throw error;
      }
```

四个条件缺一不可：错误码是 `QWEN_VC_VOICE_NOT_FOUND`、**确实存在 bound 音色**、`hops < 1`（**最多换一次**，避免递归）、且**当前用的就是那个 bound 音色**（`voiceBindingKey` 比较）。第四条很重要：如果失败的是 slot 默认音色而不是老师绑定的克隆，那不该按"克隆丢失"处理。

副作用是 `unavailableBindings.add(...)`——这是一个**执行级**的 `Set`（[engine.ts#L585-L586](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L585-L586)：`// Voice bindings this execution found unusable (a deleted clone).`）。它被传给每一 clip 的 `clipVoice({ unavailable })`，所以：

- 本次执行里**后续所有 clip 与后续所有场景**都不再尝试这个已失效的克隆（§7.1 第 1 层会跳过 bound）；
- **它不跨执行**：进程重启、lease 被抢后重开，集合就是空的，会再撞一次。设计意图是"少花钱"，不是"记住教训"。

`clipVoiceAfterMissingClone` → `narratorVoiceAfterMissingClone`（[narration-voice.ts#L210-L224](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts#L210-L224)）可能返回 `null`（没有可用的替代），此时原样抛错 → 该 clip 走完重试 → 场景失败 → run `paused`。注意 `usedFallbackVoice: !!override`：**已经在用 fallback 音色时就不再给第二次 fallback**。

## 9. clip 的并发与"整场景回滚"

```ts
// lib/server/generation/run/engine.ts L1078-L1102
    try {
      const concurrency = services.parallelSceneConcurrency();
      // The clips and the scene that names them commit together; a failure
      // anywhere here retries both.
      if (concurrency > 1 && speechActions.length > 1) {
        const settled = await Promise.allSettled(
          lazyBoundedMap(speechActions, concurrency, narrateOne),
        );
        const rejected = settled.find(
          (result): result is PromiseRejectedResult => result.status === 'rejected',
        );
        if (rejected) throw rejected.reason;
      } else {
        for (const action of speechActions) await narrateOne(action);
      }
      if (unvoiced > 0) {
        log.warn(`run ${run.id}: ${stepId} left ${unvoiced} speech clip(s) unvoiced`);
      }
      await appendScene(sceneIndex, stepId, scene, unvoiced);
    } catch (error) {
      // Nothing of this attempt committed: its clips are released (an entry
      // the scene's write did commit is not touched by the release).
      await services.releaseAssets(owner, allocated, stepContext);
      throw error;
    }
```

读这段要抓住五件事：

1. **并发条件是两个"与"**：`parallelSceneConcurrency() > 1` **且** `speechActions.length > 1`。并发度取自 `PARALLEL_SCENE_CONCURRENCY`，`≤0 → 0`、上限 10（[provider-config.ts#L1093-L1097](file:///D:/GITHUB_chat/OpenMAIC/lib/server/provider-config.ts#L1093-L1097)）——注意 `0` 也能通过 `> 1` 的判断为 false，所以"关并发"和"设为 1"效果相同：串行。
2. **`Promise.allSettled` 而不是 `Promise.all`**：这是刻意的。用 `all` 的话第一个 reject 会让其它 promise 变成 unhandled rejection；`allSettled` 等**所有** clip 落定，再取**第一个** rejected 抛出。含义：一条 clip 失败不会打断其它 clip 的合成（钱照花），但整个场景不会提交。
3. **`lazyBoundedMap` 是"无栅栏"原语**（[concurrency.ts#L47-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/utils/concurrency.ts#L47-L58)）：一次性为每个 item 创建 promise，但用信号量限制同时在跑的数量，`limit` 被 `Math.min(Math.floor(limit), items.length || 1)` 夹紧。
4. **`unvoiced` 的 warn 在 commit 之前**：日志里出现 `left N speech clip(s) unvoiced` 意味着**这一场景仍然会成功入库**，只是有 N 句没声音。这是"部分成功"的正常路径，不是故障。
5. **`catch` 里的 `releaseAssets(allocated)` 是唯一的回滚动作**。它释放的是本次尝试已入库但场景没写成功的音频（`pending` 状态的资源）。`releaseAssets` 的实现逐条 `releasePending`，失败只 `warn`（[services.ts#L543-L554](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L543-L554)），注释说明兜底机制：**"Still pending: the collector reclaims it once its deadline passes"**——就算释放失败，回收器最终会清掉。

一个容易漏掉的边界：如果**场景写成功了但后面的东西抛错**（例如 `placeHeldMedia` 之后 `writeScene` 内部失败），`allocated` 里的资源**不会**被释放——因为它们已经属于一个已提交的场景。注释里的 "(an entry the scene's write did commit is not touched by the release)" 说的就是这个：释放用的是 `releasePending`，已随场景提交的资源早已不是 pending 状态，调也不会误删。

## 10. 入库：`appendScene` → `placeHeldMedia` → `writeScene`

### 10.1 串行闸门 `exclusive`

```ts
// lib/server/generation/run/engine.ts L1104-L1117
  const appendScene = (
    sceneIndex: number,
    stepId: string,
    scene: Scene,
    unvoiced = 0,
  ): Promise<void> =>
    exclusive(async () => {
      const media = await placeHeldMedia(scene);
      await writeScene(sceneIndex, stepId, scene, media, unvoiced);
    });

// lib/server/generation/run/engine.ts L1212-L1219
  // ── Media ──
  /** Scene writes and media placement take turns (one worker holds the run). */
  let writeChain: Promise<unknown> = Promise.resolve();
  const exclusive = <T>(body: () => Promise<T>): Promise<T> => {
    const next = writeChain.then(body, body);
    writeChain = next.catch(() => undefined);
    return next;
  };
```

`exclusive` 是一条** promise 链式的互斥队列**，注意 `writeChain.then(body, body)`——**前一个失败也照样跑下一个**（两个分支都是 `body`），`writeChain = next.catch(() => undefined)` 保证链条本身永不 reject。也就是说：某个场景写入失败**不会**卡死写链，其它场景的写入继续排队执行。

为什么需要它：场景写入与媒体落位**都改同一份课程文档**（同一 stage 的 scenes 表）。旁白 clip 之间可以并发（§9），但落盘必须串行——否则两个 `putScene` 会交错、或者媒体落位读到一个正在被改写的场景。注释说得很直白：`Scene writes and media placement take turns (one worker holds the run)`。

### 10.2 `placeHeldMedia`：写场景之前顺手补齐媒体

```ts
// lib/server/generation/run/engine.ts L1126-L1152
  const placeHeldMedia = async (scene: Scene): Promise<StepCommit> => {
    const placed: NonNullable<StepCommit['steps']> = [];
    const events: NewGenerationRunEvent[] = [];
    owner = await currentOwnerOf(run.ownerId);
    for (const { request } of mediaItems()) {
      const checkpoint = mediaCheckpoint(request.elementId);
      if (checkpoint?.status !== 'stored' && checkpoint?.status !== 'done') continue;
      if (!sceneCarriesMediaReference(scene, request.elementId)) continue;
      if (checkpoint.status === 'stored' && !(await ownerAssetExists(owner, checkpoint.assetId))) {
        const failed: GenerationRunMediaCheckpoint = {
          mediaType: request.type,
          status: 'failed',
          message: `The stored ${request.type} was gone before its scene was written`,
        };
        placed.push({ id: mediaStepId(request.elementId), output: failed });
        events.push(mediaEvent(request.elementId, failed));
        continue;
      }
      placeInScene(scene, request.elementId, checkpoint);
      if (checkpoint.status === 'stored') {
        const done = doneOf(checkpoint);
        placed.push({ id: mediaStepId(request.elementId), output: done });
        events.push(mediaEvent(request.elementId, done));
      }
    }
    return { steps: placed, events };
  };
```

| 判断 | 含义 |
|---|---|
| 只处理 `stored` / `done` | `queued`/`generating`/`failed`/`skipped` 的媒体与本次写入无关 |
| `sceneCarriesMediaReference(scene, elementId)` | 这个场景里**真的有**该占位符的引用（不跨场景乱改） |
| `stored` 且字节不见了 | 记 `failed`：`The stored <type> was gone before its scene was written`。**不抛错**——图片没了不配让一整节课旁白失败 |
| `placeInScene` | 原地改 `scene`：把占位符引用换成 asset 引用（含 poster） |
| 只有从 `stored` 变来的才追加 `done` | `done` 的（之前已落位到别的场景）只补引用，不重复发事件 |

头部注释解释了为什么"字节不见了"要大声失败而不是静默留空（[#L1119-L1125](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1119-L1125)）：run 会**主动给已存资源续命**（keep alive），所以真字节消失是**故障**，要"fails loud（带 Retry）"，而不是让场景"什么都不指"。

`owner = await currentOwnerOf(run.ownerId)` 在循环前刷新一次：owner 身份可能在长时间合成期间发生变化（多租户 owner 行的键控）。

### 10.3 `writeScene`：一个事务里三样东西

```ts
// lib/server/generation/run/engine.ts L1154-L1210（节选）
    const change: StepCommit = {
      step: { id: stepId, output: { scene } satisfies NarrationOutput },
      steps: media.steps,
      patch: {
        // The scenes in the course: every narrated scene, this one included.
        scenesCompleted:
          [...steps.keys()].filter((id) => id !== stepId && id.endsWith(':narration')).length + 1,
        ...(sceneIndex === 0 ? { stageId: stage.id } : {}),
        ...(unvoiced > 0 ? { narrationUnvoiced: unvoiced } : {}),
      },
      events: [
        { type: 'step_completed', data: { step: stepId } },
        ...(sceneIndex === 0
          ? [{ type: 'course_created' as const, data: { stageId: stage.id } }]
          : []),
        { type: 'scene_ready', data: { index: sceneIndex, sceneId: scene.id, order: scene.order } },
        ...(media.events ?? []),
      ],
    };
    // The document write and the checkpoint that records it, in one
    // transaction (a retried append rewrites the same scene id).
    // The engine's view of the run moves only once the whole write committed.
    let committed: StoredRun | undefined;
    const inTransaction = async (tx: Queryable) => {
      committed = await commitGenerationRunIn(tx, lease, change);
    };
    if (sceneIndex === 0) {
      await createRunCourse({ ownerId: owner, lease, stage, outlines: outline().outlines,
        firstScene: scene, inTransaction });
    } else {
      await appendRunScene({ ownerId: owner, lease, stageId: stage.id, scene, inTransaction });
    }
    run = committed!;
    steps.set(stepId, change.step!.output);
    for (const checkpoint of change.steps ?? []) steps.set(checkpoint.id, checkpoint.output);
    reportCommittedRunEvents(run, change.events ?? [], currentOwnerOf);
```

四个必须理解的点：

1. **`scenesCompleted` 是从内存 `steps` 现算的**，不是 `sceneIndex + 1`。因为旁白顺序并不保证递增吗？不是——是因为**并发模式下前面的场景可能失败（skipped）**，`skippedScenes` 的那几个 index 压根没有 narration 检查点。计数只数真实入课的场景，外加当前这个（`+ 1`），并且显式排除自己（`id !== stepId`）以免重试时重复计数。
2. **`step_completed` 事件不带 `output`**（只有 `{ step: stepId }`）。这是全管线的通用约定：**大对象只落库不上 SSE**。前端想要场景内容要另外走文档接口。
3. **`inTransaction` 是关键机制**：`commitGenerationRunIn(tx, ...)` 被塞进文档写入的事务回调里。于是"场景落库"和"检查点落库"**原子**。注释里那句 "a retried append rewrites the same scene id" 说明了为什么可以随便重放：`appendRunScene` 用的是 `putScene`（按 sceneId 幂等覆盖），Retry 时同一个 `sceneId` 再写一次而已，不会产生两份。
4. **内存状态在提交后才动**：`run = committed!` 与 `steps.set(...)` 都在 `await` 之后。若事务回滚，引擎内存视图保持原样，重试从干净状态开始。事件同理——`reportCommittedRunEvents` 在最后，保证**前端不会看到未提交的事件**。

`sceneId` 从哪来？第 6 步的 `buildCompleteScene` 里 `nanoid()`（见第 6 册 §11）。旁白只是把它放进 `scene_ready` 事件里，从不新建 id。

## 11. `document.ts`：课程文档的五种写法

[run/document.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts) 是旁白步骤唯一的"文档 API"，一共五个函数，全都被 lease 围栏（fence）包住。

| 函数 | 用途 | 是否 sanitize | 行 |
|---|---|---|---|
| `createRunCourse` | 第 0 个场景：建 stage + 文档 + 首场景 + outline | **是**（stage 与 firstScene 都过 `sanitizeSceneContent`） | [#L58-L84](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L58-L84) |
| `appendRunScene` | 后续场景：`putScene`（幂等覆盖同 id） | **是** | [#L90-L106](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L90-L106) |
| `mutateRunScene` | 媒体落位：读-改-写**当前**文档里的某个场景 | **故意不做** | [#L114-L131](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L114-L131) |
| `loadRunCourse` | 读整份文档（媒体落位前看现在有哪些场景引用了占位符） | — | [#L134-L139](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L134-L139) |
| `completeRunCourse` | 收尾：把 outline 标记为生成完成 | — | [#L161-L187](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L161-L187) |

**`createRunCourse` 建的 outline**（[#L66-L74](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L66-L74)）：

```ts
  const outline: AppDocumentOutline = {
    outlines: input.outlines,
    generationComplete: false,
    producer: 'server-job',
    producerRef: input.lease.runId,
    createdAt: now,
    updatedAt: now,
  };
```

`producer: 'server-job'` + `producerRef: runId` 让这份文档**能被追溯到具体哪次 run**；`generationComplete: false` 是课程"半成品"的唯一真相位，由 `completeRunCourse` 翻成 `true`。

**`mutateRunScene` 为什么不 sanitize**（[#L124-L125](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L124-L125)）：

```ts
    // Not sanitized as a whole: the scene is the author's, and the mutation
    // changes media references only.
```

因为此刻文档里的那个场景**可能已经被用户编辑过了**（课程完成后作者改过内容）， sanitize 会按生成期的规则改动作者的东西。这个函数只替换媒体引用字段。

**`completeRunCourse` 的锁顺序**（[#L168-L186](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L168-L186)）是全文件最需要小心的部分：

```ts
    await forwardOwnerWrite(tx, input.ownerId);          // ① owner 身份锁
    await fenceGenerationRunWriteIn(tx, input.lease, input.stageId); // ② 课程归属行 + run 行
    const meta = await readStageMeta(tx, input.stageId); // ③ 读元数据
    if (!meta || meta.deletedAt !== null) throw new RunCourseDeletedError(input.stageId);
    await tx.query(`UPDATE document_outlines SET data = data || jsonb_build_object('generationComplete', true, ...) WHERE stage_id = $1`, ...);
    await touchRunCourseIn(tx, input.stageId, now);      // ④ 触发 revision
    await markStageGenerationComplete(tx, input.stageId);// ⑤ 归属行完成标记
    await input.commit(tx);                              // ⑥ run 自己变 completed
```

注释（[#L169-L170](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L169-L170)）："The order every course write takes its locks in: the owner's identity lock, the course's ownership row, then the run row."——**任何**写课程的路径都必须按这个顺序取锁，否则死锁。`touchRunCourseIn` 是一条 `UPDATE document_stages SET updated_at = ..., data = jsonb_set(...)`（[#L145-L152](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L145-L152)），靠数据库上的 **revision 触发器**通知"有编辑器打开着这门课，请重新读"。

**课程被删的两种判定**（[#L38-L43](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L38-L43)）：

```ts
function courseGone(error: unknown): boolean {
  return (
    error instanceof StageAccessError &&
    (error.refusal === 'tombstoned' || error.refusal === 'unclaimed')
  );
}
```

`tombstoned` = 删了留墓碑；`unclaimed` = 归属行根本不存在。两者都转成 `RunCourseDeletedError`（消息 `The course <stageId> was deleted`，[#L31-L36](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts#L31-L36)），run 会走 `end()` 而不是 `pause()`——**用户删课不是故障，不该让人 Retry**。

## 12. 媒体：三种落位时机

旁白步骤与图片/视频的关系是"顺带"的——媒体自己不在三连步里，它们有独立的 `media:<elementId>` 检查点（[plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) 的 `mediaStepId`）。落位有三个入口：

| 函数 | 何时 | 改哪 | 是否同事务写 narration 检查点 | 行 |
|---|---|---|---|---|
| `placeHeldMedia` | **生成期**，写某场景之前 | 内存里那份**还没入库**的场景 | 是（它就是同一个事务的一部分） | [engine.ts#L1126-L1152](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1126-L1152) |
| `placeMedia` | **生成期**，媒体先于场景完成（lane 里） | **已入库**的场景（从 narration 检查点取） | 是（`appendRunScene` + `commitGenerationRunIn`） | [#L1233-L1275](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1233-L1275) |
| `placeIntoCurrentCourse` | **课程已完成后**（Retry 补媒体） | 当前**文档**里的场景（可能被作者编辑过） | 是，但用 `mutateRunScene`，不 sanitize | [#L1288-L1354](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1288-L1354) |

`placeMedia` 的要点：它遍历所有大纲 index，从 narration 输出里挑出携带该占位符的场景，逐个 `structuredClone` + `placeInScene`，然后**每个场景都重写文档 + 重写该场景的 narration 检查点**（注释："The narration checkpoint is the scene as the course holds it"，[#L1252](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1252)），**只有最后一个**才附带 `media:<elementId>` 的 done 检查点和事件。这样中途崩溃时媒体仍是 `stored`，下次执行会重放剩下的场景。

`placeIntoCurrentCourse` 的要点（注释 [#L1277-L1287](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1277-L1287)）：

- 先 `loadRunCourse` 拿到**现在的**文档，筛出现在仍引用占位符的场景；
- `placed` 标志先检查"是否已有场景引用这些字节"（**中断前就放好了**的情形）；
- 若作者把元素删了：一个场景都不匹配 → 丢弃字节 + `releaseAssets`，并记 `MEDIA_ELEMENT_REMOVED`（**final，不重试**），日志 `run <id>: <elementId> is no longer in the course; dropping its media`（[#L1339](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1339)），检查点消息是 `The <type> was generated after its element was removed from the course`（[#L1345-L1350](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1345-L1350)）。注释解释了这个"故意不复活元素"的选择：否则每次 Retry 都会给一个页面上不存在的东西重烧一遍钱。

`unplacedMedia`（[#L1632-L1654](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1632-L1654)）处理最后一类边角——**存了字节但没有任何已旁白场景引用它**：

```ts
    const held = (elementId: string) =>
      outline().outlines.some((_, index) => {
        const narrated = output<NarrationOutput>(sceneStepId(index, 'narration'));
        return narrated ? sceneCarriesMediaReference(narrated.scene, elementId) : false;
      });
```

判据是"**narration 检查点**里有没有引用"，不是"文档里有没有"。头部注释（[#L1627-L1631](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1627-L1631)）给出分流规则：没场景持有的 → 直接转 `done`（课程完整了，不需要它们）；有场景持有但落位失败的 → **保持 `stored`**，让 completed 的 run 被重新认领去放置（`media_pending`）。

## 13. 收尾：主循环里的 complete / pause / end

旁白是最后一个会被跑的步骤，所以"所有场景的 narration 都完成"就是"run 完成"。主循环（[#L1782-L1806](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1782-L1806)）的次序：

```ts
      if (run.stageId && (await isRunCourseDeleted(run.stageId))) {
        await end(run.stageId);
        return 'ended';
      }
      // The media pass runs alongside the scenes once the course exists.
      if (run.state === 'generating' && courseExists()) await ensureMediaLane();
      if (advance.kind === 'complete') {
        const [failedIndex, failure] =
          [...skippedScenes.entries()].sort(([a], [b]) => a - b)[0] ?? [];
        if (failedIndex !== undefined) {
          // Every other scene is in: pause at the first one that failed.
          const { message, ...code } = failure!;
          await pause(sceneStepId(failedIndex, 'content'), message, code);
          return 'paused';
        }
        // Every media item has an answer before the course is complete.
        await settleMedia();
        try {
          await complete();
        } catch (error) {
          if (error instanceof MediaWorkPendingError) continue;
          throw error;
        }
        return 'completed';
      }
```

| 环节 | 事实 |
|---|---|
| `courseExists()` | `steps.has(sceneStepId(0, 'narration'))`（[#L1224](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1224)）——**课程的存在与否，等价于"第 0 个场景的旁白跑完了"**。这是全步最定义性的一行：文档是在旁白里才建出来的 |
| 媒体泳道何时启动 | 一旦 `generating` 且课程存在（[#L1787](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1787)），与后续场景并跑 |
| 完成前有没有 skipped 场景 | 有 → `pause(第一个失败场景的 content stepId)`，把失败**归因到 content**（虽然卡在这里才报） |
| `settleMedia()` | 等每个媒体项都有答案，再 `placeStoredMedia()`；放置失败**不阻塞完成**（字节留 `stored`，之后重新认领 run）[#L1598-L1618](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1598-L1618) |
| `complete()` | `completeRunCourse` + `unplacedMedia`；提交前在**同一事务**里检查 `hasMediaWorkIn(tx, run.id)`，有则抛 `MediaWorkPendingError` → `continue` 主循环再来一轮 [#L1675-L1702](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1675-L1702) |
| `pause()` | 先 `prewarmAbort.abort()` 再 `await stopLane()`，然后写 `state: 'paused'` + `step_failed` + `state` 两个事件 + `releaseLease: true` [#L1656-L1673](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1656-L1673) |
| `end()` | `state: 'ended'` + `ended` 事件（`{ stageId }`），**不带 error** [#L1704-L1712](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1704-L1712) |

`complete()` 里那句 `MediaWorkPendingError` 值得多说一句：注释 `// A Retry that queued media since the pass settled is generated first.`（[#L1689](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1689)）——在 `settleMedia()` 与真正 commit 之间有窗口，用户点了 Retry 往表里插了一条 queued 媒体。检查放在**事务内**（`hasMediaWorkIn(tx, ...)`）就是为了不在竞态下把课完成掉：宁可重跑一轮循环。

## 14. 这一步对外说了什么：事件、进度与播放

第 7 步是**唯一会改变"课程是否存在"的一步**，所以它也是唯一一次让前端从"等待"切换到"可以看了"的一步。它对外只说四种话。

### 14.1 事件表（本步产生的）

| 事件 | payload | 声明处 | 什么时候发 |
|---|---|---|---|
| `step_completed` | `{ step }` | [types.ts#L271-L272](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L271-L272) | 每个场景旁白成功，**不带 output**（output 太大，它就是整棵场景树） |
| `course_created` | `{ stageId }` | [types.ts#L305-L306](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L305-L306) | 只在 `sceneIndex === 0` 时发；注释写得很准："the course document exists (its first scene is ready)" |
| `scene_ready` | `{ index, sceneId, order }` | [types.ts#L307-L308](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L307-L308) | **每个**场景入库后发一次，是"可以打开这一页了"的信号 |
| `media` | `{ elementId, mediaType, status, ... }` | [types.ts#L313-L317](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L313-L317) | `placeHeldMedia` 把 `stored` 翻成 `done` 时追加 |

另外两个属于收尾而非本步，但同样在第 7 步的代码路径上发出：`completed{ stageId }`（[#L309-L310](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L309-L310)）与 `ended{ stageId }`（[#L311-L312](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L311-L312)）。

失败时还会看到 `step_retry`，由 `retryOptions` 的 `onRetry` 提交（[engine.ts#L438-L451](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L438-L451)）：

```ts
// lib/server/generation/run/engine.ts L435-L451
      log.warn(
        `run ${run.id}: retrying ${stepId} (${event.attempt}/${event.maxAttempts}): ${event.cause}`,
      );
      await commit({
        events: [
          {
            type: 'step_retry',
            data: {
              step: stepId,
              attempt: event.attempt,
              maxAttempts: event.maxAttempts,
              reason: event.reason,
              cause: event.cause,
            },
          },
        ],
      });
```

排障时 `step_retry` 是**最值钱的事件**：它同时给出第几次尝试、总次数、以及 `cause`。旁白失败通常在这里就能看出是音色问题（400 系）还是 TTS 服务问题（5xx / 30 s 超时）。

### 14.2 前端怎么消费

```ts
// lib/generation-run-client/reducer.ts L184-L203
    case 'course_created':
      next.stageId = (data.stageId as string) ?? next.stageId;
      return next;
    case 'scene_ready': {
      const index = Number(data.index);
      const isNew = !(index in next.readyScenes);
      next.readyScenes = { ...next.readyScenes, [index]: String(data.sceneId) };
      next.skippedScenes = withoutKey(next.skippedScenes, index);
      if (isNew) {
        const completed = next.progress.scenesCompleted + 1;
        next.progress = {
          ...next.progress,
          scenesCompleted:
            next.progress.scenesTotal > 0
              ? Math.min(completed, next.progress.scenesTotal)
              : completed,
        };
      }
      return next;
    }
```

三个细节值得注意：

1. **`isNew` 守住了进度**：Retry 重写同一个场景会再次发 `scene_ready`，但 `index in next.readyScenes` 已经为真，于是**计数器不会重复 +1**。这与 `writeScene` 里服务端那句 "a retried append rewrites the same scene id" 是一对——两边都靠"同 index 幂等"来保证重复事件无害。
2. **`scenesTotal > 0` 时才夹紧**：总数未知时不做 `Math.min`，避免把进度压成 0。
3. **`skippedScenes` 按 index 清除**：一个之前失败的场景被 Retry 补上后，"跳过"标记自动消失。

预览进度条上，`actions` 与 `narration` **共用同一个格子**（[preview-steps.ts#L42-L43](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L42-L43)）：

```ts
  const scene = step ? /^scene:\d+:(content|actions|narration)$/.exec(step) : null;
  if (scene) return scene[1] === 'content' ? 'slide-content' : 'actions';
```

也就是说 UI 上根本没有"旁白"这一步——第 6、7 步对用户合成一格。这解释了为什么第 7 步（往往是最慢的一步，因为逐条等 TTS）看起来像"卡在动作那一格"。

### 14.3 `narrationUnvoiced`：一个只会被"加重"的计数

服务端 patch 是 `unvoiced > 0` 才带上（[engine.ts#L1171](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1171)），而 SQL 是**累加**而不是覆盖：

```ts
// lib/server/generation/run/store.ts L490-L493
  if (patch.narrationUnvoiced) {
    params.push(patch.narrationUnvoiced);
    sets.push(`narration_unvoiced = narration_unvoiced + $${params.length}`);
  }
```

含义：`narration_unvoiced` 是**整个 run 累计的静默 clip 数**。Retry 补上了音频，这个数字也**不会减少**（历史已发生）。它最后的出口是完成时的警告文案（[generate-classroom-job.ts#L251-L254](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generate-classroom-job.ts#L251-L254)，函数 `completionWarning` [#L233-L256](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generate-classroom-job.ts#L233-L256)）：

```ts
  if (run.narrationUnvoiced > 0) {
    parts.push(
      `${run.narrationUnvoiced} speech clip${run.narrationUnvoiced === 1 ? ' was' : 's were'} left without narration`,
    );
  }
```

所以"课程成功生成"和"课程有 3 句没声音"是可以同时成立的——API 用这段 warning 表达它，不会把 run 置为 failed。

### 14.4 播放端拿到的 `audioId` 到底是什么

第 7 步向 TTS 申请的键是 `tts_s${scene.order}_${action.id}` 这一族**派生 id**，而写进场景的是**返回的 assetId**（[engine.ts#L1069-L1077](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1069-L1077)：`narrateOne` 拿到 `assetId` 后 `allocated.push(assetId); action.audioId = assetId`）。播放侧只认这个最终值：

```ts
// lib/action/engine.ts L343-L354
  private async executeSpeech(action: SpeechAction): Promise<void> {
    if (!this.audioPlayer) return;

    return new Promise<void>((resolve) => {
      this.audioPlayer!.onEnded(() => resolve());
      // The legacy URL of an unconverted pair rides along as the fallback of
      // last resort; converted documents carry no audioUrl.
      this.audioPlayer!.play(action.audioId || '', (action as LegacySpeechAction).audioUrl)
        .then((audioStarted) => {
          if (!audioStarted) resolve();
        })
        .catch(() => resolve());
```

三点结论：`audioId` 为空时传的是 `''`，`play` 不会真正开播，`audioStarted` 为 false 就 `resolve()`——**该动作立即结束、不阻塞时间轴**（这就是 unvoiced 场景在课堂里的表现：老师说话的那段变成"无声闪过"）；`audioUrl` 只为未转换的旧文档兜底；播放失败被 `catch` 吞掉并 `resolve`——**旁白的错误从不升级成课堂的错误**。

## 15. 速查、坑清单、代码地图、动手验证

### 15.1 常量速查（本步相关）

| 常量 | 值 | 位置 | 一句话 |
|---|---|---|---|
| `STEP_DEADLINES_MS.narrationClip` | `30_000` | [deadline.ts#L23-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L23-L24) | **每条 clip 的每次尝试** 30 s，注释点名 `POST /api/generate/tts` |
| `STEP_DEADLINES_MS.sceneActions` | `60_000` | [#L21-L22](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L21-L22) | 第 6 步用；对照着看更容易记住 |
| `StepTimeoutError.statusCode` | `504` | [deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | 消息 `` `${label} did not finish within ${ms / 1000} s` `` |
| `FIRST_SCENE_MAX_RETRIES` | `2` | [retry.ts#L21-L22](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L21-L22) | 注释：`1.1.x's browser retries for the first scene` |
| `SCENE_MAX_RETRIES` | `5` | [retry.ts#L23-L24](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts#L23-L24) | 第 1 个场景之后的所有场景 |
| `maxAttempts` | `maxRetries + 1` | retry 包装 | 所以第 0 场景最多 3 次、其余最多 6 次 |
| clip 的 `refusalStatus` | `400` | [engine.ts#L1046](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1046) | `retryOptions(stepId, sceneIndex, 400)`；对照 content 步用的是 `500`（[#L531](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L531)） |
| `DEFAULT_BASE_DELAY_MS` | `1000` | [generation-retry.ts#L22](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L22) | 指数退避起点 |
| `DEFAULT_MAX_DELAY_MS` | `16000` | [#L24](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L24) | `min(max, base * 2^(attempt-1))` + 20% jitter |
| `RETRYABLE_STATUS_CODES` | `{408, 409, 425, 429}` | [#L25](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L25) | 加上 `>= 500`（[#L143](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L143)） |
| `NON_RETRYABLE_STATUS_CODES` | `{400, 401, 403, 404, 422}` | [#L26](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L26) | 以及任何其它 4xx（[#L144](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L144)） |
| `PARALLEL_SCENE_CONCURRENCY` | 夹紧到 `[0, 10]`，默认 `0` | [provider-config.ts#L1093-L1097](file:///D:/GITHUB_chat/OpenMAIC/lib/server/provider-config.ts#L1093-L1097) | 注释说明为何默认关：低配额的 key 会把突发打成 429 |
| `TTS_MAX_TEXT_LENGTH` | `{'glm-tts': 1024}` | [tts-utils.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts) | **全库只有这一个 provider 有长度限制** |
| hop 上限 | `hops < 1` | engine.ts 的 `QWEN_VC_VOICE_NOT_FOUND` 分支 | 一次执行最多换一次声 |

`audioId` / 切分 id 的命名模板，排查数据库时靠它能反推来源：

```
tts_s${scene.order}_${action.id}          // clip 的申请键
${action.id}_tts_${i + 1}                 // 长文本切分后的子动作 id（1 起）
```

### 15.2 坑清单

1. **`runStep` 对 `narration` 返回 `{}`**（[engine.ts#L747-L750](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L747-L750)）。别按其它步的模式去找"返回值写回 state"的地方——这一步自己提交，`case` 里那句注释是全管线唯一的。
2. **`narrationTarget()` 返回 `null` 是静默的**。槽位没配 / provider 是 `browser-native-tts` → 整场景**没有音频也不计 unvoiced**（连 warn 都没有）。看日志查不出"为什么这门课全哑了"，要查的是运行配置的 TTS 槽位。
3. **400 不重试。** clip 用 `refusalStatus = 400`：音色 ID 错、文本被拒这类问题会**立即**失败并把 run 打到 `paused`。等来等去等不到重试日志时，先怀疑是"配置错"而不是"服务不稳"。
4. **并发失败 = 整场景回滚，但钱照花。** `Promise.allSettled` 会等所有 clip 落定，成功的字节被 `releaseAssets` 释放、失败的抛错；一条 clip 失败不中断其它合成。Retry 会**重新合成整个场景的所有 clip**。
5. **`unvoiced` 不是失败。** 看到 `left N speech clip(s) unvoiced` 表示场景照样入库、课程照样完成；课堂上的表现是那一段无声快进（§14.4）。
6. **存储满也会变成 unvoiced。** `narrateClip` 在 `storeGeneratedAsset` 返回 `refused` 时只 warn `Asset storage is full; leaving … unvoiced` 并返回 `null`（[services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)），**不抛错**。清盘之后 Retry 才有声音。
7. **超时 30 s 是"每条 clip 每次尝试"**，不是整个场景。串行 + 长文本 + 多次重试时，一个场景花上几分钟是正常的，别误判为卡死。
8. **只有 `glm-tts` 会切分长文本。** 其它 provider 的长句原样送出去；如果供应商自己的上限更低，报错会以 4xx 的形式出现，进而撞上第 3 条。
9. **`unavailableBindings` 不跨执行。** 它是 [engine.ts#L585-L586](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L585-L586) 里的局部 Set：本次执行认定"这个克隆音色不存在"后不再重复尝试，但**下一次 Retry 会重新发现一遍**。
10. **`mutateRunScene` 故意不 sanitize**（[document.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts)，注释 `the scene is the author's, and the mutation changes media references only`）。用户编辑过的内容**不会**被媒体回填洗掉；反过来也别指望它会替你修正非法结构。
11. **完成≠无缺陷。** `completed` 事件照发，只要 `narrationUnvoiced > 0` 或媒体有 `failed`，API 里就带 `completionWarning`（§14.3）。只看 `state === 'completed'` 的监控会漏报。
12. **课程是否存在 ≡ 第 0 场景旁白是否完成**（`courseExists()`，[engine.ts#L1224](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1224)）。第 0 场景的旁白挂掉 = 数据库里**根本没有这门课**，前面五步的成果只存在于 run 的 checkpoint 里。

### 15.3 代码地图（按"我要找什么"）

| 我要找 | 去这里 |
|---|---|
| 这一步如何被调度、为何返回 `{}` | [engine.ts#L747-L750](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L747-L750) |
| 场景旁白主流程 | [engine.ts#L978-L1103](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L978-L1103) `narrateScene` |
| 入库与 checkpoint 同事务 | [#L1154-L1210](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1154-L1210) `writeScene`；串行锁 [#L1212-L1219](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1212-L1219) `exclusive` |
| 写前媒体归位 | [#L1119-L1152](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1119-L1152) `placeHeldMedia` |
| 完成后补位 / 移除释放 | [#L1233-L1275](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1233-L1275) `placeMedia`、[#L1288-L1354](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1288-L1354) `placeIntoCurrentCourse` |
| 主循环与完成/暂停/结束 | [#L1782-L1806](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1782-L1806)、`settleMedia` [#L1598-L1618](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1598-L1618)、`complete` [#L1675-L1702](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1675-L1702) |
| 选谁当旁白老师 | [agent-voice-options.ts#L47-L53](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/agent-voice-options.ts#L47-L53) `pickNarratorAgent` |
| 音色解析四层兜底 | [narration-voice.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/narration-voice.ts) `slotVoice` / `clipVoice` |
| clip 落库与释放 | [services.ts#L509-L554](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L509-L554) `narrateClip` / `releaseAssets` |
| 长文本切分 | [tts-utils.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/audio/tts-utils.ts) `splitLongSpeechActions` |
| 浏览器侧合成（非服务端管线） | [steps/narration.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/narration.ts) `synthesizeNarration` |
| 课程文档读写 | [document.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/document.ts) 五函数 |
| 重试与状态码判定 | [retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts) + [generation-retry.ts#L130-L158](file:///D:/GITHUB_chat/OpenMAIC/packages/%40openmaic/generation/src/generation-retry.ts#L130-L158) |
| 事件类型 | [types.ts#L265-L318](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L265-L318) |

### 15.4 动手验证（四个小实验）

1. **验证"课程存在 ≡ 第 0 场景旁白完成"**：把 TTS 槽位关掉（或指向 `browser-native-tts`）后生成。预期：课程照样建起来（`narrationTarget` 返回 `null` 走的是"无音频直接 append"，不是失败），`course_created` 正常发出——对照坑 2：全哑但没有 unvoiced。
2. **验证 clip 级 30 s 预算**：把旁白文本拉成一段长朗读并把 TTS 指向一个慢端点。预期日志：`retrying scene:0:narration (1/3)`，`cause` 为 `narrationClip did not finish within 30 s`。
3. **验证切分只对 `glm-tts` 生效**：写一段超过 1024 字的 `speech`，分别用 `glm-tts` 与其它 provider。前者日志出现 `Split speech for glm-tts: action=…, len=…, chunks=…` 且场景里多出 `*_tts_1/2` 子动作；后者原样送出。
4. **验证 `scene_ready` 幂等**：让第 1 场景失败后 Retry。前端 `scenesCompleted` 只增加一次，而事件流里能看到两次 `scene_ready{ index: 1 }`——这就是 §14.2 里 `isNew` 的作用。

## 16. 总结：一句话与三条不变量

**一句话**：第 7 步做两件事——把第 6 步已经合法的 `speech` 文本逐条换成能播的音频资源，然后在**一个事务里**把"这个场景属于这门课"写进数据库；两件事同时成立，课程才算存在。

贯穿本步的三条不变量：

1. **文档与 checkpoint 不可分割。** `writeScene` 把 `inTransaction = async tx => commitGenerationRunIn(tx, lease, change)` 交给 `createRunCourse` / `appendRunScene`（§10.3）：不会出现"场景写进去了但 run 不知道"，也不会出现"run 记完成了但库里没这门课"。所有"幂等重放"的设计都建立在这一条上。
2. **能播就播，不能播就沉默地继续。** 音色四层兜底、`clipVoice` 返回 `null` 只 `unvoiced += 1`、存储满只 warn、播放失败只 `resolve`。整条链上唯一会**中断课程生成**的，是"重试次数用尽的硬失败"，而不是"缺一段声音"。
3. **写只有串行口。** `exclusive()` 让场景按顺序入库、`fence` 让失租的执行写不进任何东西、`unplacedMedia` 让媒体只能落在它自己的场景里。并发的只是**合成**，不是**提交**。

七步至此走完：材料分析 → 联网检索 → 大纲 → Agent 画像 → 场景内容 → 场景动作 → **旁白与入库**。前六步产出的是"一份计划与其零件"，只有第七步把它们**变成了一门存在的课**。

---

系列导航：[总览](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md) · [第 1 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-1.md) · [第 2 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-2.md) · [第 3 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-3.md) · [第 4 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-4.md) · [第 5 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-5.md) · [第 6 册](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-6.md) · 第 7 册（本册）
