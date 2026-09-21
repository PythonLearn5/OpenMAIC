# 课程生成端到端流程说明

<!--
  职责：详细说明 OpenMAIC 「首页一键生成课程」的端到端流程，
  包括：准备期（handleGenerate）→ Generation Preview 7 步管线 →
  首个场景入库 → 跳转课堂页 → 课堂后台渐进生成剩余场景 → 媒体
  生成/入库 全过程。每个阶段都带代码位置、API 列表、状态机、异常
  兜底与断点续跑策略。
-->

- 相关项目结构文档：`../PROJECT-STRUCTURE.md`
- 相关总体架构文档：`../ARCHITECTURE.md`
- 相关学习大纲：`../LEARNING-OUTLINE.md`

---

## 0. 阅读提示

本文档把生成流程拆成 **四个阶段**：

| 阶段 | 名称 | 发生页面 | 关键代码入口 |
|---|---|---|---|
| **P1** | 生成准备（Session 冻结 & 材料本地化） | `/` 首页 | `app/page.tsx → handleGenerate()` |
| **P2** | 生成预览 7 步管线（首屏阻塞生成） | `/generation-preview` | `app/generation-preview/page.tsx → startGeneration()` |
| **P3** | 首个场景入库 + 跳转课堂 | `/generation-preview` 尾部 | `app/generation-preview/page.tsx` store.saveToStorage() |
| **P4** | 课堂页后台渐进生成剩余 N-1 场景 + 媒体 | `/classroom/:id` | `components/classroom/ClassroomSurface.tsx` → `lib/hooks/use-scene-generator.ts → useSceneGenerator().generateRemaining()` |

每一步均会标注 `code:` 代码相对路径 + 行号锚，方便在 IDE 里跳转。

---

## 1. 阶段 P1：生成准备（首页 handleGenerate）

### 1.1 触发条件

由首页「生成课程」按钮（或 textarea 上 `⌘/Ctrl+Enter`）触发。
按钮可用条件（`canGenerate`）：

- `form.requirement.trim().length > 0`（非空教学目标）
- `hasUsableProvider === true`（已经配置好可用 LLM Provider，并且该 provider 一定带 concrete model。参见 #580 invariant，不再在 handleGenerate 内重复校验）。

代码：
- `code: ../app/page.tsx#L696-L703`（canGenerate、Ctrl/Cmd+Enter 触发）
- `code: ../app/page.tsx#L569-L682`（handleGenerate 本体）

### 1.2 生成前的「冻结快照」设计

handleGenerate 不直接读 live form state / live store state。
点击瞬间先捕获以下**不可变快照**：

| 名称 | 代码位置 | 为什么要冻结 |
|---|---|---|
| `frozenMaterials` | `page.tsx#L588` | 防止准备期间用户「删材料、加材料」导致 documentSources 数组和实际落盘的文件撕裂。 |
| `settingsSnapshot` 及其派生：`frozenPdfProviderId` / `frozenPdfProviderConfig` | `page.tsx#L589-L602` | 防止准备期间用户改 Settings 里的 PDF 解析器（本地/MinersU/AliDocMind），导致 A 文件用了 Parser A、B 文件又用了 Parser B。 |

**同时**翻转 `preparingGenerate = true`（`page.tsx#L605`），让期间 `addCourseMaterial` / `removeCourseMaterial` 变成 no-op（`page.tsx#L557-L560`），工具栏 extractor select、web-search toggle 也被禁用 — 冻结快照 + 冻结交互，双保险。

### 1.3 需求 → UserRequirements 构造

```ts
UserRequirements {
  requirement,                 // 文本教学目标
  userNickname?, userBio?,     // 用户画像，传给 LLM 生成风格化的讲授
  webSearch?,                  // 是否开联网搜索
  interactiveMode?,            // 交互式 / 普通
  taskEngineMode?,             // 职业测验模式（= vocationalTestMode）
}
```

代码：`code: ../app/page.tsx#L607-L615`

### 1.4 文档材料 Blob 本地化（IndexedDB 临时存储）

对 `frozenMaterials` 顺序遍历，每一份：

1. `await storeDocumentBlob(item.file)` — 把浏览器内存里的 File 写入 IndexedDB。
2. 产出 `storageKey`，和 `id/name/size/lastModified/mimeType/order` 一起塞进 `documentSources[]`。
3. **MIME 修正**：浏览器 `file.type` 对 `.doc/.docx/.rtf` 经常返回空字符串，用 `normalizeDocumentMimeType({ fileName })` 兜底。

#### 失败回滚

`storeDocumentBlob` 是有副作用的磁盘写入。若 M-1 份成功、第 M 份失败（例如浏览器 IndexedDB 配额爆了），会用 `Promise.allSettled(keys.map(deleteDocumentBlob))` 把前 M-1 份全部清理，**不留下孤儿 Blob 占磁盘**。

代码：
- 主流程：`code: ../app/page.tsx#L623-L653`
- Blob 存储 API：`code: ../lib/core/document-blob-storage.ts`
- MIME 修正：`code: ../lib/core/document-mime.ts`

### 1.5 构造 SessionState 并写入 sessionStorage

数据结构见「§7. 核心数据结构 · GenerationSessionState」。

关键兼容字段：
- `documentSources[]` — 新的多材料结构，**真实真源**。
- `pdfStorageKey / pdfFileName / documentMimeType` — 只写 `documentSources[0]`，向后兼容（历史版本只有一份 PDF 的旧 session 继续能跑）。

写入用 `sessionStorage.setItem('generationSession', JSON.stringify(sessionState))`。

> **为什么用 sessionStorage 而不是 Zustand / localStorage？**
> - Zustand（内存）：用户在生成页 F5 刷新 → 丢失，Bad UX。
> - localStorage：跨 Tab 持久，两个 Tab 各生成一次会互相覆盖，串台。
> - sessionStorage：**单 Tab 生命周期、刷新保留、关闭即丢** — 最贴合「一次生成一个浏览器会话」语义。

代码：
- 写入：`code: ../app/page.tsx#L655-L671`
- 结构定义：`code: ../app/generation-preview/types.ts#L11-L43`

### 1.6 路由跳转：/ → /generation-preview

```ts
router.push('/generation-preview');
```

`finally` 里 `setPreparingGenerate(false)` 释放冻结。
注意：正常路由跳转后页面即 unmount，该 setState 是 no-op；仅在异常路径（例如 `storeDocumentBlob` 抛错）会落到用户可见的解冻逻辑。

---

## 2. 阶段 P2：Generation Preview（7 步管线 + SSE + 大纲审查 + Agent 翻牌）

入口组件：`code: ../app/generation-preview/page.tsx`，组件内叫 `GenerationPreviewContent()`。

### 2.0 前置：Session 加载与自动续跑

进入页面的第一个 useEffect（`page.tsx#L219-L242`）：

1. `cleanupOldImages(24)` — 清理超过 24h 的旧临时图片，避免 IndexedDB 只增不减。
2. `sessionStorage.getItem('generationSession')` 读取，如果合法：
   - 缺 `previewPhase` 就按 `sceneOutlines ? 'outline-ready' : 'preparing'` 推断，用于**断点续跑**。
   - `review` 阶段又没 `sceneOutlines` → 意味着用户 SSE 中途点开过编辑器又刷新了，记 sticky flag `outlineReviewIntentRef=true`。
3. `sessionLoaded = true`。

第二个 useEffect（`page.tsx#L286-L302`）：
- 只在 session 存在 & 未启动过时调用 `startGeneration()`。
- 续跑判定：`needsOutlines || phase ∈ {preparing, generating-content, review-without-outlines}` → 都自动启。**已 `review` 且 outline 已落盘的**，会停在审查界面等用户确认，而不是越过用户直接往下。

### 2.1 AbortController 生命周期

每次 startGeneration 都：

1. `abortControllerRef.current?.abort()`（停掉上一轮，防止浏览器前进后退/重复点重开造成两次请求）。
2. 新建 `AbortController`，注入后续所有 `fetch(signal)`。
3. unmount useEffect（`page.tsx#L244-L250`）兜底：离开页面 → `abort()` + 清理大纲审查 Timer，保证走后不留孤儿请求烧 token。

### 2.2 Step 0：文档材料解析（可选，有材料才执行）

| 项 | 值 |
|---|---|
| 步骤 ID | `pdf-analysis` |
| 触发条件 | `documentSources.length > 0 && !session.pdfText` |
| 后端 API | `POST /api/extract-document` |
| 后端处理 | `code: ../app/api/extract-document/route.ts`（会再按 providerId 分发到本地 parse-pdf / MinerU / AliDocMind / ASR） |
| 前端并发 | `Promise.all(per-material fetch)`，**每一份材料独立调一次 API**（可分别使用不同 providerConfig；为后续「每份材料用不同解析器」设计留口子） |

每份材料请求 body（multipart/form-data）：
- `file`：从 IndexedDB `loadDocumentBlob(storageKey)` 拿回 Blob → 还原成 File。
- `providerId / apiKey / baseUrl / accessKeyId / accessKeySecret`：优先取「材料级绑定的 source.providerConfig」，兼容旧格式；否则回落到 session 全局的 `frozenPdfProviderConfig`。

解析返回字段的**双版本兼容**：
- MinerU 新版：`data.metadata.pdfImages[]`（结构化，带 id / pageNumber / width / height / description）。
- 老版本地 parse：`data.images: string[]`（只有 dataURL，不结构化）。

解析完用 `buildDocumentBundle(parsedParts)` 做跨材料合并：
- 汇总 `bundle.text`（所有材料文本按 order 顺序拼接）。
- 汇总 `bundle.images`（所有材料图，加 `sourceDocumentId/Name/Order` 标识、算 vision 优先级）。
- 计算 `bundle.textContentBudget` / `MAX_VISION_IMAGES`，**超出的会截**，弹出 truncation warnings。

然后 `await storeImages(bundle.images)` → IndexedDB，返回 `imageStorageIds[]`，和 pdfImages 写回 session（**把 Blob 解析结果持久化**，F5 不必重新解析）。

代码：`code: ../app/generation-preview/page.tsx#L334-L469`

### 2.3 Step 1：联网搜索（可选，开了 webSearch 才执行）

| 项 | 值 |
|---|---|
| 步骤 ID | `web-search` |
| 触发条件 | `session.requirements.webSearch === true` |
| 后端 API | `POST /api/web-search` |
| 后端处理 | `code: ../app/api/web-search/route.ts`（分发 SearXNG / Brave / DuckDuckGo / Baidu / Tavily + 可选 Claude 摘要改写） |

入参：`{ query, pdfText?, providerId, apiKey?, baseUrl?, baiduSubSources?, claudeModelId? }`
返回：`{ sources: [{title, url}], context: string }`

写入 session：`researchContext / researchSources`。sources 会在页面右上角实时显示引用卡片。

代码：`code: ../app/generation-preview/page.tsx#L471-L518`

### 2.4 Step 2：大纲生成（⭐ 核心，SSE 流式 + 审查/编辑）

| 项 | 值 |
|---|---|
| 步骤 ID | `outline` |
| 后端 API | `POST /api/generate/scene-outlines-stream`，响应是 **SSE（`text/event-stream`）** |
| 后端处理 | `code: ../app/api/generate/scene-outlines-stream/route.ts`（基于 Vercel AI SDK `streamText` / 流式生成，并支持服务端重试） |

#### SSE 事件协议

| type | 语义 |
|---|---|
| `languageDirective` | 语言指令，例：「Teach in Simplified Chinese」 |
| `courseTitle` | 课程标题，后面用作 Stage.name |
| `outline` | 一个 SceneOutline（增量 push 到数组，用户看到「一条条冒出来」） |
| `retry` | 服务端决定清空重来（比如 LLM 返回格式不对、重试）— 前端清空 collected、丢掉旧 directive/title |
| `done` | 结束，带 `{ outlines, languageDirective, courseTitle, /* vocational flags */ }` |
| `error` | 中断，抛错给用户 |

前端用 ReadableStream + `TextDecoder` 按行解析，**`lines.pop()` 留最后一行给下一次 `pump()` 拼 buffer**，防止 JSON chunk mid-split 解析错。

代码：
- SSE pump 循环：`code: ../app/generation-preview/page.tsx#L558-L670`

#### 大纲审查（review）两种触发模式

1. **用户设置里 `reviewOutlineEnabled=true` → 自动进入审查**。
2. **用户在 SSE 中途点击流式大纲卡片 → `handleExpandStreamingOutline()`**：sticky ref `outlineReviewIntentRef = true`，即使流还没结束，编辑器立即展开，SSE 继续填编辑器（边流边看/边改）。

审查等待模型：`waitForOutlineReviewChoice(outlines, shouldReview, signal)`（`page.tsx#L187-L216`）。

```
shouldReview=true   → 等用户手点 Confirm，没有超时，停多久都行
shouldReview=false  → OUTLINE_REVIEW_AUTO_CONTINUE_MS=2500ms 自动 resolve
                       用户有 2.5s 看到完整大纲卡片，不会刚看到就跳下一步，节奏更自然
```

用户在审查里做了什么？
- 点 Confirm（`handleConfirmOutlines`）：手动 resolve → 往下走。
- 中途 Collapse Editor（`handleCollapseEditor`）：收起卡片并重新装备 2.5s 自动计时。
- 在编辑器改了 outline 文本/顺序（`handleOutlinesChange`）：**只在 SSE 结束后生效**（流式期间编辑器是只读，即使 racy 也被 guard 忽略）。

「已提交」定义：`previewPhase = 'generating-content'`，此时会 `localStorage.removeItem('requirementDraft')` 清掉首页输入框草稿缓存（此时已不能回退到「用户原输入」）。

### 2.5 Step 3：多智能体生成（可选，AgentMode=auto 才执行）

| 项 | 值 |
|---|---|
| 步骤 ID | `agent-generation` |
| 触发条件 | `settings.agentMode === 'auto'`（否则用用户 preset 选中的 Agent 列表） |
| 后端 API | `POST /api/generate/agent-profiles` |
| 后端处理 | `code: ../app/api/generate/agent-profiles/route.ts`（根据大纲内容自动生成角色名单 1~4 名：Teacher/助教/学生/小丑提问者等） |

**前端给后端的「选择白名单」非常关键**：
- `availableAvatars[]` + `avatarDescriptions[]`：预置 12 张头像（/public/avatars/*.png），LLM 只能从里面挑，不会凭空返回不存在的图片。
- `availableVoices[]`：用 `getEnabledProvidersWithVoices(ttsProvidersConfig, voiceProfiles)` 把「当前启用的 TTS + 音色 + clone 对应的 modelId」展平成扁平列表，LLM 只能选其中之一，避免返回未配置音色导致后面 TTS 全挂。
- `narratorVoice`：用户「全局设置里选中的 TTS 音色」，传给服务器 → **服务器强制把 Teacher agent voice pin 到它**（这样用户设置的 narrator 不会被 LLM 换掉，符合用户直觉）。

返回落地：
- `applyGeneratedAgentsToRegistry(stage.id, generatedConfigs)` → 写入 in-memory Agent Registry。
- `settings.setSelectedAgentIds(savedIds)` → 让用户 UI 里它们是「选中的角色」。
- `settings.setAgentSelectionIsUserSet(false)` → 标记这是生成的、不是用户挑的，下次进别的教室不要把这批 ghost id 带过去（跨 stage 泄漏防护）。

**然后**展示 `AgentRevealModal`：一张张翻牌动画（每张是角色头像、名字、角色、一句话人设），用户看完最后一张，才 resolve Promise 继续往下 — 纯 UX 节奏控制，不是技术需求。

#### 失败兜底

网络错误 / LLM 返回畸形 → 不会中断整个生成，catch 后 fallback 到 `settings.selectedAgentIds.filter(isPreset)`（用户预置角色），保证**至少还能生成出课程**。

代码：`code: ../app/generation-preview/page.tsx#L735-L933`

### 2.6 Step 4：第 1 个场景内容生成

| 项 | 值 |
|---|---|
| 步骤 ID | `slide-content` |
| API 封装 | `fetchSceneContent({ outline: firstOutline, ... })` — `code: ../lib/hooks/use-scene-generator.ts#L146-L198` |
| 后端 API | `POST /api/generate/scene-content` → `code: ../app/api/generate/scene-content/route.ts` |
| 重试 | `withGenerationRetry()` 包装，失败自动重试 N 次（前台重试策略 `FOREGROUND_SCENE_RETRY_OPTIONS`） |

Vision 流程：如果 pdfImages[i].storageId 已经 materialized，LLM 看到的是 `imageMapping[img.id]`（dataURL 或 AssetID），多模态模型会读图（需要 provider 支持 vision，否则退回纯文本）。

### 2.7 Step 5：第 1 个场景动作脚本生成

| 项 | 值 |
|---|---|
| 步骤 ID | `actions` |
| API 封装 | `fetchSceneActions({ content, previousSpeeches: [], agents, languageDirective, ... })` — `code: ../lib/hooks/use-scene-generator.ts#L201-L247` |
| 后端 API | `POST /api/generate/scene-actions` → `code: ../app/api/generate/scene-actions/route.ts` |
| 重试 | 同样 withGenerationRetry |

返回是完整的 `Scene { title, narration, elements: SlideElement[], actions: Action[] }`，其中 actions 是 18 种（Speech/Emotion/Highlight/Zoom/Laser/Spotlight/Navigation/Media 等）动作序列，供 PlaybackEngine 播放。

### 2.8 Step 6：第 1 个场景 TTS 合成（可选，非 browser-native-tts 且 Provider 启用时）

**注意**：课堂页在播放时才跑「browser native TTS（浏览器本地 speechSynthesis）」，它不需要后端。只有用户选了云端 TTS（Azure / FishAudio / Edge / Qwen TTS / CosyVoice / ElevenLabs ...）时，在生成期提前把 TTS 合成出来，**这样用户点播放不会先等 3s 才能听到**。

API：`generateTTSForScene(scene, languageDirective, signal, retry)`
- `code: ../lib/hooks/use-scene-generator.ts#L606-L671`
- 后端路由 `POST /api/generate/tts` → `code: ../app/api/generate/tts/route.ts`

TTS Fallback Chain（每个 narration action 独立跑，voice 不可用最多换 1 次，避免循环死）：
```
Agent's explicit voice binding →
  (unavailable? fallback) → 用户全局 narrator →
    (still unavailable? fallback) → deterministic enabled provider pick
  最多 1 hop（MAX_NARRATOR_VOICE_FALLBACK_HOPS = 1）
```

---

## 3. 阶段 P3：首个场景入库 + 跳转课堂

### 3.1 落库

P2 的 4-6 步产出 firstScene 后：

```ts
store.setStage(stage);            // 写 stage（含 generatedAgentConfigs、agentIds）
store.setOutlines(outlines);      // 持久化 outlines（F5 续跑用）
store.addScene(firstScene);       // 真正把 Scene 推入 scenes[]，
                                  // 同时自动从 generatingOutlines[] 里删掉匹配 order
store.setCurrentSceneId(firstScene.id);
store.setGeneratingOutlines(remaining);   // 剩下 N-1 outlines 标记为"骨架"
await store.saveToStorage();              // Dexie(IndexedDB) ←→ server（如有 server 后端）持久化
```

### 3.2 移交参数给课堂

写 sessionStorage 的**第二把钥匙**：`generationParams`。只包含课堂继续生成其余场景时必须具备的东西：

```ts
{ pdfImages, agents, userProfile, languageDirective }
```

> 为什么又写 sessionStorage，而不塞 Zustand？
> 因为 `router.push('/classroom/:id')` 是 SPA 软导航，store 内状态当然保留。但用户在课堂页 F5 刷新（很常见），内存里 pdfImages / agents 就丢了，后台续生成会全部因缺 imageMapping 报 vision 图找不到。**用 sessionStorage 保证刷新可用**。

清理：`sessionStorage.removeItem('generationSession')`（P2 结束，不再走 generation-preview 的 resume 分支）。

### 3.3 跳转

`router.push('/classroom/${stage.id}')`。此时用户**立刻可看到第一个场景，点播放即可**，不会在生成预览页等所有 N 个场景做完 — 这就是「首屏优先 + 后台渐进生成」设计的核心收益。

代码：`code: ../app/generation-preview/page.tsx#L941-L1052`

---

## 4. 阶段 P4：课堂后台渐进生成剩余 N-1 场景

挂载组件：`code: ../components/classroom/ClassroomSurface.tsx`

### 4.1 启动判定

进入课堂 → 加载 stage/outlines/scenes 完毕 → 判断三种情况：

| 情况 | 动作 |
|---|---|
| ① `outlineProducer === 'server-job'` | 服务器端 Durable Agent 已接管生成，浏览器**不启动任何生成器**（避免两端并行改同文档）。 |
| ② `!generationComplete && hasPendingOutlines` | **后台 resume 剩余场景** — 走 P4.2。 |
| ③ 所有 outlines 已有对应 scene，但可能 media 没跑完 | 仅启动 `generateMediaForOutlines(materializedOutlines)` 补媒体；同时 `markGenerationCompleteIfDone()` 立 flag，避免将来用户删某张 slide 被误判为「还缺场景要续生成」而无意义浪费 API。 |

代码：`code: ../components/classroom/ClassroomSurface.tsx#L406-L488`

### 4.2 Resume：从 sessionStorage 重建 imageMapping 并启动 generateRemaining

```ts
const params = JSON.parse(sessionStorage.getItem('generationParams'));
// pdfImages 可能混合 assetId（已落 server S3）和 storageId（仍在本机 IndexedDB）
// 按 img.id → assetId / storageId 合并重建真正的 imageMapping
finishResume(imageMapping);
```

合并策略是**两路 merge**，而不是假定「所有图要么全在 S3 要么全在 IndexedDB」——防止半路上传失败时，部分成功 dataURL 化被另一半全量 select 的 transport 策略 silent drop。

### 4.3 generateRemaining：串行 + 可选并行 Content 的 Pipelined Loop

实现：`code: ../lib/hooks/use-scene-generator.ts#L766-L999`（useSceneGenerator → generateRemaining）。

核心不变式：

```
- Scene *Content* 无跨场景依赖，可并行（用户开了 parallelSceneConcurrency 才启用）
- Scene *Actions* / *TTS* 依赖 previousSpeeches（为了跨场景语义不重复、前后连贯），
  必须严格串行。Actions 串行同时保证了「一张失败 → 暂停，留 Resume 按钮给用户手点重试」
  的 UX 不变（否则并行全炸，用户根本不知道继续从哪里开始）。
```

#### #572 Pipeline 机制（并发控制）

```
parallelConcurrency = settings.parallelSceneConcurrency
 （服务器 env: GENERATE_PARALLEL_SCENE_CONCURRENCY，经 settings-parallel-scene-concurrency →
   settings store → server clamp → settings store 的 clamp；这里又再 floor 一次，belt-and-suspenders）
```

- **=0 或 1**：纯串行，和最初设计行为一致。
- **>1**：启动 `lazyBoundedMap(pending, parallelConcurrency, fetchContent, shouldContinue)` 同时最多跑 N 个 content fetch，结果存 `Map<outlineId, Promise>`。主串行循环跑到 scene K 时，**内容通常已算完**（前 K-1 个的 actions/TTS 用了很多秒），actions/TTS 照串 — 不破坏 previousSpeeches + pause-on-failure UX。

每个 outline 的串行内步骤：

```
（可能已并行预热完成） Content fetch
                    ↓ success?
                    +-Yes → Actions fetch → success?
                    |                      +-Yes → TTS generate（可选） → addScene → mark done
                    |                      +-No  → 失败：serial 模式 pause / parallel 跳过
                    +-No  → 失败：同上分支
```

失败登记：
- `store.addFailedOutline(outline)`（用于用户看到「某张失败，单独重试」按钮）。
- Serial 模式：立刻 `setGenerationStatus('paused')` break 整个循环。
- Parallel 模式：当前 outline 从 generatingOutlines 移除，继续其他；最后 `hadContentFailure = true` 整体 `paused` 让用户 UI 上能看到「有失败」。

#### generationEpoch：用户点 Stop / Resume 的竞态防护

生成器启动时记录 `startEpoch = state.generationEpoch`。用户每次 Stop → Resume，store 会 `generationEpoch++`。循环里的每一次 outline 开始都会校验：

```ts
if (abortRef.current || store.getState().generationEpoch !== startEpoch) break;
```

这就避免「用户点 Stop 后 200ms 又点 Resume，旧的循环还在跑和新的循环同时 addScene」。

#### 媒体：并行，不阻塞内容/动作

媒体生成（图像生成 / 视频生成）在循环开头就 kick off：

```ts
generateMediaForOutlines(outlines, stage.id, mediaAbortSignal).catch(..warn..)
```

它不阻塞「内容/动作/TTS」的用户可见进度，用户点播放立刻能看没图的纯内容版本，图/视频生成完通过 reconciliation 自动补到 SlideElement.imageStorageId / videoStorageId 上，渲染层动态刷新。

在 server-backed 媒体持久化模式下，每次 overlap 会先 `mediaAbortRef.current?.abort()` 上一轮，防止跑着旧课程图的孤儿请求花真金白银。

### 4.4 完成

```ts
store.setGenerationStatus('completed');
store.setGeneratingOutlines([]);
store.setGenerationComplete(true);  // 这个 flag 持久化，将来删 slide 不会误触发「重新生成被删的那张」
```

---

## 5. 阶段 P5：媒体生成（补图/补视频）— P2 + P4 都会涉及

文件：`code: ../lib/media/media-orchestrator.ts` 的 `generateMediaForOutlines(...)`。

职责：遍历所有 outlines → 对应的 scene → slide elements → 每个需要 `image/video` 的占位（slot）：
- 如果已存在已落盘的 assetId / storageId → 跳过。
- 如果没有 → 调 `/api/generate/image` 或 `/api/generate/video`（对应 `code: ../app/api/generate/image/route.ts` 和 `code: ../app/api/generate/video/route.ts`）。
- 完成后 `commitToPool` → 写入 IndexedDB / server S3 → 触发 scene 字段 reconciliation，播放器热刷新。

和内容生成「解耦 + 可 resume」，这就是用户看到「幻灯片先出现占位符，几秒后图自动补全」的实现。

---

## 6. API 清单汇总

生成流程用到的所有后端 handler（相对 `app/api/`）：

| 路由 | 在哪个阶段被调 | 调用点 |
|---|---|---|
| `POST /extract-document` | P2 Step 0 | `generation-preview/page.tsx#L373` |
| `POST /web-search` | P2 Step 1 | `generation-preview/page.tsx#L480` |
| `POST /generate/scene-outlines-stream`（SSE） | P2 Step 2 | `generation-preview/page.tsx#L568` |
| `POST /generate/agent-profiles` | P2 Step 3 | `generation-preview/page.tsx#L841` |
| `POST /generate/scene-content` | P2 Step 4 + P4 loop | `hooks/use-scene-generator.ts#L169` |
| `POST /generate/scene-actions` | P2 Step 5 + P4 loop | `hooks/use-scene-generator.ts#L218` |
| `POST /generate/tts` | P2 Step 6 + P4 loop（cloud-tts only） | `hooks/use-scene-generator.ts → generateAndStoreTTS` |
| `POST /generate/image` | P2 / P4 媒体阶段后台 | `media/media-orchestrator.ts` |
| `POST /generate/video` | P2 / P4 媒体阶段后台 | `media/media-orchestrator.ts` |
| `POST /generate/voice` | （音色克隆等辅助功能，主流程外） | `api/generate/voice/route.ts` |

Provider 信息如何从前端传到后端：
- 用户在 Settings 里亲手填的 Key → 由前端 `getApiHeaders()` 打包成 `x-*` 请求头传给对应 route。
- 服务器 `*.env` 里配置的托管 Key → 后端 route 优先读 env，env 空才 fallback 到 header。
- SSRF Guard：`code: ../lib/server/ssrf-guard.ts` — 无论 Key 从哪来，**baseUrl 指向内网/loopback/非 http(s)** 一律拦截，防止 SSRF。

代码：
- `generation-preview` 侧 headers 打包：`code: ../app/generation-preview/page.tsx#L253-L278`
- `use-scene-generator` 侧 headers 打包：`code: ../lib/hooks/use-scene-generator.ts#L74-L100`（内容/动作/TTS 的 headers）

---

## 7. 核心数据结构

### 7.1 GenerationSessionState（P1 + P2 的真源）

定义：`code: ../app/generation-preview/types.ts#L11-L43`

```ts
interface GenerationSessionState {
  sessionId: string;
  requirements: UserRequirements;
  // 材料解析产物（逐步填上）
  pdfText: string;
  documentSources?: SessionDocumentSource[];
  pdfImages?: PdfImage[];
  imageStorageIds?: string[];
  imageMapping?: ImageMapping;           // old-format 兼容
  sceneOutlines?: SceneOutline[] | null; // outline 流完才有
  currentStep: 'generating' | 'complete';

  // ⭐ 断点续跑状态机（非常重要）
  previewPhase?:
    | 'preparing'          // 刚进页，什么都没跑
    | 'outline-ready'      // outline 已落盘，不需要重新 SSE
    | 'review'             // 用户在审查（或审查中刷新）
    | 'generating-content';// 已提交审查，开始内容

  // 旧单文档兼容
  pdfStorageKey?; pdfFileName?; documentMimeType?;
  pdfProviderId?; pdfProviderConfig?;

  // 逐步填上的上下文
  researchContext?; researchSources?;
  languageDirective?; courseTitle?;
  taskEngineMode?: boolean;
}
```

### 7.2 previewPhase 状态机（F5 可恢复的调度依据）

```
  [用户刚进预览页]
        │
        ▼
 'preparing' ──outline SSE 完成──▶ 'outline-ready'
        │                              │
        │ reviewOutlineEnabled          │ 用户点开流式卡片
        │ 或用户中途展开                ▼
        └─────────────────────────▶ 'review'
                                         │  用户 Confirm（或 2.5s 自动）
                                         ▼
                                  'generating-content' ──▶ P3 跳课堂
```

规则（`getActiveSteps` + 自动续跑判断实现）：
- `outline-ready`：SSE 已成功 → 跳过 Step 0/1/2（材料/搜索/SSE 都已做完），如果用户是审查模式下刷新则停在 review，否则直接进入 agent/content。
- `review` 且已有 `sceneOutlines[]`：停审查页等用户，**不自动越过用户决策**。
- `review` 且 sceneOutlines 为空：（用户 SSE 中途展开编辑器后刷新）→ 重新启 SSE，因为之前的 outline 没存。

### 7.3 ALL_STEPS 与 getActiveSteps 过滤

用户/会话不同，顶部步骤进度条只显示「这次真正会执行的步骤」：

| Step ID | 何时显示 |
|---|---|
| `pdf-analysis` | `有上传材料 && 还没解析完（没 pdfText）` |
| `web-search` | `requirements.webSearch === true` |
| `outline` | 总是显示 |
| `agent-generation` | `settings.agentMode === 'auto'` |
| `slide-content` | 总是显示 |
| `actions` | 总是显示 |

定义：`code: ../app/generation-preview/types.ts#L90-L147`

---

## 8. 失败兜底、回滚、防御性设计速查

| # | 问题 | 兜底实现 |
|---|---|---|
| 1 | 材料上传中途失败，IndexedDB 里留下孤儿 Blob | `Promise.allSettled(keys.delete)` 全清 `code: page.tsx#L649-L651` |
| 2 | 用户 F5 刷新生成预览页 | `previewPhase` + sessionStorage 断点续跑 |
| 3 | 生成中途关 Tab / 返回首页 / 前进后退 | AbortController 贯穿所有 fetch；unmount 一定 abort |
| 4 | 大纲 SSE 中途服务端返回格式非法 | 后端发 `type=retry` 清空重来；前端支持 retry event 重置 collected + directive/title 防上次脏值遗留 |
| 5 | Agent 生成失败 | Fallback 到用户 preset agents，不中断课程 |
| 6 | Narrator voice 绑定死了（克隆音色被服务器删） | 最多 1 hop fallback：`Agent binding → global narrator → enabled provider deterministic` |
| 7 | 用户在串行模式下点 Stop 又迅速 Resume | `generationEpoch` 作为启动令牌，老循环检测到 epoch 改变立即 break |
| 8 | 用户删 slide 后被误判为"还要生成这张" | `generationComplete` flag + resume 判定里 `completedOrders` 取 scenes 集合差集过滤 |
| 9 | 旧 session（单 PDF 时代）导入继续用 | `pdfStorageKey/pdfFileName/documentMimeType` 兼容字段写回 session |
| 10 | 图片缓存 24h 前的垃圾 | `cleanupOldImages(24)` 在预览页 mount 时扫一遍 |
| 11 | Settings 里 PDF 解析器/个人 Key 被异步改了 | 点击瞬间 snapshot，整个 session 用 snapshot，不读 live store |
| 12 | SSRF（用户把 baseUrl 填成 http://169.254.169.254） | 服务端 SSRF Guard（见 §6 最后） |
| 13 | IndexedDB 里 pdfImages 部分 assetId、部分 storageId，resume 选一种 transport 会丢一半 | imageMapping 双路 merge（ClassroomSurface L455-L467） |

---

## 9. 时序一览（简版 ASCII）

```
[User at /]
  │ 填 requirements + 上传材料
  │ 点 生成
  ▼
handleGenerate()
  │  (1) 冻结材料+PDF provider snapshot
  │  (2) Blobs → IndexedDB (失败回滚删除)
  │  (3) 构造 SessionState → sessionStorage.generationSession
  ▼
router.push → /generation-preview

[GenerationPreview]
  │ resume from sessionStorage
  ▼
startGeneration()
  │
  ├─ Step 0 (可选) → /api/extract-document × M 份并发 → buildDocumentBundle →
  │       storeImages → pdfText/pdfImages/imageStorageIds persist to sessionStorage
  │
  ├─ Step 1 (可选) → /api/web-search → researchContext/researchSources
  │
  ├─ Step 2        → /api/generate/scene-outlines-stream
  │                   SSE pump: languageDirective / courseTitle / outline* / retry / done
  │                   → optionally waitForOutlineReviewChoice (用户审查或 2.5s auto)
  │
  ├─ Step 3 (可选) → /api/generate/agent-profiles → AgentRevealModal 翻牌
  │
  ├─ Step 4        → fetchSceneContent(firstOutline)   [/api/generate/scene-content]
  ├─ Step 5        → fetchSceneActions                 [/api/generate/scene-actions]
  ├─ Step 6 (可选) → generateTTSForScene               [/api/generate/tts]
  │
  ├─ store.setStage() + setOutlines() + addScene(firstScene)
  │   + setGeneratingOutlines(remaining)
  │   + saveToStorage
  │   + sessionStorage.generationParams  (移交后续生成参数)
  ▼
router.push → /classroom/:stageId

[ClassroomSurface]
  │ load stage/outlines/scenes from persistence
  │ hasPendingOutlines → resume
  │   → load generationParams → rebuild imageMapping (assetId ∪ IndexedDB)
  │   → useSceneGenerator.generateRemaining():
  │       ├─ generateMediaForOutlines() (后台跑，不阻塞)
  │       ├─ serial loop over pending outlines
  │       │    ├─ Content:  串行/可选并行预热 (#572 pipeline bounded)
  │       │    ├─ Actions:  严格串行（previousSpeeches）
  │       │    ├─ TTS:      严格串行（云端 TTS）
  │       │    └─ 成功 → addScene(); 失败 → (serial) paused or (parallel) skip + failedOutline
  │       └─ all done → generationComplete = true
  ▼
用户看到全课程逐张填完；可随时播放、编辑、导出、分享。
```
