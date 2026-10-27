# 课程生成管线 · 第 1 册：材料分析（material-analysis）

<!--
  职责：详解生成管线**第 1 步「材料分析」**——把用户上传的一到多份课程材料
  （文档 PDF/DOCX/PPTX/MD/TXT，或音视频）抽取成纯文本 + 图片，打包成大纲能直接
  消费的 `pdfText` / `pdfImages`，并把图片落成一个待定课程的资产。
  覆盖：这一步为什么"两个现场执行"（上传期后台抽取 vs run 步骤等待+读取）、
  run 步骤分支逐行解读、`RunStepServices` 的三个材料方法、后台抽取 worker 的
  状态机/租约/复用/配额、能力层 `analyzeMaterial` 的媒体与文档双路径与 6 种拒绝码、
  `buildDocumentBundle` 的 50k 字符 / 20 图预算算法、产出如何被第 2/3/5 步消费、
  事件与快照字段、失败链与 Retry 语义、相关常量与配置、动手验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：无（本步是管线第一步）｜下一步：[第 2 册 联网检索](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-2.md)
- 源文件：[steps/material-analysis.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts)
  · [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts)
  · [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts)
  · [materials/extraction.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `material-analysis`（id 与 kind 同名，无序号） |
| 是否必经 | **条件步**：`input.materialIds.length > 0` 才进入计划，见 [plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54) |
| 所处 run 状态 | `preparing`（[plan.ts#L67-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L72) 的 `phaseOfStep`） |
| 检查点写在哪 | `generation_run_steps(run_id, 'material-analysis')`，值为 `MaterialOutput` |
| 时间预算 | 单次抽取 300 s（[deadline.ts#L8-L13](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L8-L13)）；**排队等待不设限** |
| 进程内重试 | 无（本步不套 `withRouteRetry`，靠 Retry 命令重启失败的抽取） |
| UI 显示 | **不占预览页步骤位**（[preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) 把它折叠进 `web-search`） |
| 产出消费方 | 第 2 步 `research`、第 3 步 `outline`（文本 + 视觉图）、第 5 步 `scene:n:content`（`imageMapping`） |

### 0.2 一句话本质

> **这一步几乎不在 run 里"抽取"**：材料在上传那一刻就已经开始后台抽取，
> run 的 `material-analysis` 步做的是**等它抽完、把结果读出来、打包、把图片变成课程资产**。
> 只有上传时没抽（`extract=false`）、抽取失败后被重启、或 run 抢在 worker 前面时，
> run 才会真正驱动一次抽取。

这就是为什么 [engine.ts#L599-L632](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L599-L632)
里那个 `case 'material-analysis'` 只有 30 行，而真正的抽取逻辑在
[steps/material-analysis.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts)（467 行）
与 [materials/extraction.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts)（783 行）里。

### 0.3 三个"现场"，同一个抽取函数

```
用户上传                POST /api/materials
   │  写入 owner_material（bytes 落 byte store），extraction=idle
   │  wakeOwnerMaterialExtractor()
   ▼
后台抽取 worker（进程级、每 owner 并发 2）
   claim → extractOwnerMaterial() → analyzeMaterial()   ← 能力层，唯一真正读字节的地方
   结果 JSON 写到 byte store 的 resultKey，行状态置 ready / failed
   ▼
run 的第 1 步            engine case 'material-analysis'
   materialsReady? 否 → 发 material_kinds 事件
   awaitOwnerMaterialExtractions(...)   ← 等 ready，idle 就补发启动，failed 就抛错
   readMaterialExtractionResult(...)    ← 读结果
   buildDocumentBundle(...)             ← 打包 + 截断
   storeMaterialImages(...)             ← 图片转课程资产，mint stageId
   ▼
检查点 MaterialOutput { pdfText, pdfImages, imageMapping, stageId }
```

三处都调用同一个 [analyzeMaterial()](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L193-L196)，
所以"run 里读到的结果"和"当场抽取的结果"字节级一致——这是
[extraction.ts 头注释 L11-L14](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L11-L14)
明确写下的设计契约。

---

## 1. run 侧：`case 'material-analysis'` 逐段解读

代码位置：[engine.ts#L599-L632](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L599-L632)。

### 1.1 先问"还需要抽吗"——`material_kinds` 事件

```ts
if (!(await services.materialsReady(owner, input.materialIds))) {
  const kinds = await services.materialKinds(owner, input.materialIds);
  await commit({ events: [{ type: 'material_kinds', data: { kinds } }] });
}
```

- `materialsReady` = 每个材料行都在且 `extraction.status === 'ready'`
  （[services.ts#L360-L367](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L360-L367)）。
- 只有**还要等**时才发 `material_kinds`：让预览页能说出"正在分析音频/视频"。
  全 ready 的 run 不发这个事件，UI 也就不显示任何分析步骤
  （e2e 断言 `Analyzing documents` 计数为 0，见
  [home-to-generation.spec.ts#L126-L128](file:///D:/GITHUB_chat/OpenMAIC/e2e/tests/home-to-generation.spec.ts#L126-L128)）。
- `materialKinds` 的判定很朴素：MIME 归一后以 `audio/` 或 `video/` 开头即为 `media`，
  否则 `document`（[services.ts#L346-L358](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L346-L358)）。

### 1.2 等待 + 读取：为什么这里**没有** `withDeadline`

```ts
const analyzed = await services.analyzeMaterials(owner, input.materialIds, { log, signal });
```

注意外层既没有 `withDeadline(...)` 也没有 `withRouteRetry(...)`，这是刻意的
（注释在 [engine.ts#L608-L610](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L608-L610)）：

- 一个材料可能排在别人的抽取队列后面，**排队时间不该算进它的预算**；
- 300 s 预算是**每次抽取本身**的，由 worker 侧施加
  （[extraction.ts#L292-L299](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L292-L299)）；
- run 侧的 `signal` 就是租约信号：租约丢了、课被删了、进程在收摊，等待立刻中止。

`analyzeMaterials` 的实现见 §3。

### 1.3 截断警告：`material_truncated`

```ts
const warnings = analyzed.truncated
  ? { events: [{ type: 'material_truncated', data: { ...analyzed.truncated } }] }
  : {};
```

大纲只看 50 000 字符、视觉模型最多看 20 张图，超出的部分**被静默丢掉是不诚实的**，
所以打包时算出的截断信息会原样变成一个事件（预算算法见 §6）。
形状：`{ textChars?: number, images?: { total, max } }`
（[services.ts#L126-L131](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L126-L131)）。

### 1.4 图片变成课程资产，顺手 mint 出课程 id

```ts
if (analyzed.images.length === 0) {
  return done({ pdfText: analyzed.text } satisfies MaterialOutput, warnings);
}
const stageId = generateClassroomId();
return done({ pdfText: analyzed.text, ...(await storeMaterialImages(stageId, analyzed.images)), stageId }, warnings);
```

- `storeMaterialImages`（[engine.ts#L469-L499](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L469-L499)）
  逐张 `storeGeneratedAsset({ kind:'image', fence })`：
  - `fence` 是租约校验，写入时若围栏失败即 `GenerationRunLeaseLostError`；
  - 存储配额满（`status === 'refused'`）→ 直接抛错，本步失败（消息：
    `Asset storage is full; the material images could not be stored`）；
  - 中途任何异常都会 `releaseAssets(allocated)` 回滚已分配的资产。
- 存完后 `pdfImages` 里的 `src` 被**清空**、只留 `assetId`（L491）：
  50 MB 的 base64 绝不进检查点 JSON，后续步骤按需再解析成 data URL（§7.3）。
- 课程 id **只在这一步产生**：没有材料时，第 4 步结束时才 mint
  （[engine.ts#L960](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L960)
  `id: output<MaterialOutput>('material-analysis')?.stageId ?? generateClassroomId()`）。
  这意味着"有材料的 run"在第一步就与一门待定课程绑上了。

### 1.5 检查点结构

```ts
interface MaterialOutput {
  pdfText: string;          // 打包后的材料正文（≤50k 字符 + 段落头）
  pdfImages?: PdfImage[];   // src 为空、assetId 已定
  imageMapping?: ImageMapping; // 图片 id → 资产 id
  stageId?: string;         // 课程 id（有图片时才产生）
}
```
见 [engine.ts#L240-L248](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L240-L248)。
`pdfText` 会被原样存进 `generation_run_steps.output`，这是管线里最大的一个检查点
（可达约 50k 字符 + 图片元数据），也是 run 能"断点续跑大纲/场景"的关键：
续跑时不必重抽材料。

---

## 2. 计划与状态：这一步怎么被排进队

| 关注点 | 代码 |
|---|---|
| 条件入计划 | [plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54)：`materialIds.length > 0` 才插入首步 |
| step id 解析 | [plan.ts#L36-L39](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L36-L39) |
| 状态归属 | `preparing`（[plan.ts#L67-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L72)） |
| 推进方式 | `advanceRun` 扫描第一个**没有检查点**的步；因此重跑/接管都天然幂等 |
| 步前先提交 | 主循环在调用 `runStep` 前提交 `state` 事件 + `step_started`（[engine.ts#L1813-L1825](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1813-L1825)） |
| 步后提交 | `done()` 一次提交 `step` 检查点 + `step_completed` + 可选警告事件（[engine.ts#L592-L596](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L592-L596)） |

材料的**准入校验发生在两处**：启动 run 的路由（`startGenerationRun` 里
[start.ts#L98-L100](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L98-L100)
调用 `resolveClassroomMaterials`，让 run 不会因为材料问题晚失败），
以及 run 内的抽取（因为配置和材料库在中间可能变化）。

`resolveClassroomMaterials`（[classroom-materials.ts#L58-L87](file:///D:/GITHUB_chat/OpenMAIC/lib/server/classroom-materials.ts#L58-L87)）做四件事：
1. 每个 id 都能解析成**该 owner 的 ready 上传**（缺失/他人/未完成/已删除**故意同答**，不做存在性预言）→ `ClassroomMaterialsUnavailableError`；
2. 格式是经典抽取族（`usesClassicExtraction`）；
3. 每个 MIME 都在 `resolveExtractableMimeTypes` 集合里 → 否则 `ClassroomMaterialsRejectedError`，消息指向 `GET /api/generate-classroom/capabilities`；
4. 总字节 ≤ `MAX_CLASSROOM_MATERIAL_TOTAL_BYTES`（150 MB）。

数量与体积上限直接沿用经典浏览器生成：
**最多 5 份材料**（`MAX_CLASSROOM_MATERIALS = MAX_DOCUMENT_BUNDLE_FILES`），
因为它们的文本共享同一份提示词预算
（[classroom-materials.ts#L29-L30](file:///D:/GITHUB_chat/OpenMAIC/lib/server/classroom-materials.ts#L29-L30)、
[bundle.ts#L4-L5](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L4-L5)）。

---

## 3. `RunStepServices` 的三个材料方法

接口声明：[services.ts#L164-L181](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L164-L181)；
默认实现：`defaultRunStepServices`（[services.ts#L345-L417](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L345-L417)）。
引擎只依赖这个接口，所以测试可以完全替换它（这也是 §16 里能"假服务跑真引擎"的原因）。

### 3.1 `analyzeMaterials` 的四步

[services.ts#L369-L417](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L369-L417)：

1. `resolveClassroomMaterials(ownerId, materialIds)` → 再校验一次准入；
2. `awaitOwnerMaterialExtractions(records, ctx.signal)` → 等到全部 `ready`（§4.5）；
3. 逐个 `readMaterialExtractionResult(record, byteStore)` → 组装 `ParsedDocumentPart[]`
   （`source.order` 就是 `materialIds` 里的顺序，**顺序即语义**：大纲按它分节）；
4. `buildDocumentBundle(parts)` → `text` + `images`，再用 `bundleTruncation(bundle)` 得出警告。

图片解码在 L401-L415：结果里每张图的 `src` **必须是 data URL**，否则
`throw new Error('Material image <id> has no inline bytes')`；解码尺寸上限
`MAX_EXTRACT_DOCUMENT_FILE_SIZE_BYTES`（50 MB），MIME 不是 `image/` 开头时兜底 `image/png`。

> 抽出来的结果是"一个 JSON 对象"，长这样
> （[extraction.ts#L82-L113](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L82-L113)）：
> `{ version: 1, text, pageCount?, images: [{ id, src, pageNumber, description?, width?, height? }] }`。
> 图片字节是**内联在结果里的 data URL**，因此结果对象可能很大——它的上限见 §4.6。

---

## 4. 后台抽取 worker：`lib/server/materials/extraction.ts`

这是本步里最"工程"的一块，值得逐节读。头注释
[L1-L23](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L1-L23) 就是它的说明书。

### 4.1 状态机

材料行的 `extraction.status`：`idle → extracting → ready | failed`
（持久化在 `owner_material`，见 `lib/persistence/owner-materials.ts`）。

### 4.2 租约与心跳（与 run 同一套思路）

[runClaimedOwnerMaterialExtraction#L406-L493](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L406-L493)：

- `claimOwnerMaterialExtraction(pool, workerId, { leaseTtlMs, perOwnerLimit, exclude })` 认领；
- 每 `heartbeatIntervalMs`（默认 `agentRuntimeConfig.heartbeatIntervalMs`）心跳一次；
  **心跳发现租约没了 → `lost.abort()` → 抽取器立刻被中止，本尝试删除自己的结果对象**，绝不删别人的（L415-L429）；
- 进程在收摊（`options.signal` aborted）→ `releaseOwnerMaterialExtraction` 把租约交还，等下一个进程接管（L458-L462）；
- 真正的失败才 `settleOwnerMaterialExtraction(..., failedExtraction(error))`（L464）。

### 4.3 复用：`extractionIdentityKey`

[L160-L178](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L160-L178)：
同一 owner 重复上传同一份字节时，若**抽取服务身份不变**就复制现成结果、不再抽取。
身份 = `mimeType + documentStatus + 文档服务的(provider, model, baseUrl, origin, options) + asr 的(provider, model, baseUrl)`；
**凭据被有意排除**（key 不改变抽取结果）。
owner 换了抽取服务，身份变了，就会重抽——这是"换个更好的解析器再试一次"的实现基础。
复制失败（那份材料中途被删）则回落实抽取（L286-L289）。

### 4.4 抽取本体：`extractOwnerMaterial`

[L251-L354](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L251-L354)，注意这几个细节：

| 行 | 细节 |
|---|---|
| L260 | `normalizeDocumentMimeType({ mimeType, fileName })`：MIME 缺失时按扩展名推断 |
| L292-L299 | 预算 = `STEP_DEADLINES_MS.materialAnalysis`（300 s），用 `AbortSignal.any` 与调用方信号合成 |
| L302-L315 | 调 `analyzeMaterial`，`request: {}`、`redactCallerInput: false`（后台现场没有"回显调用方输入"的风险，消息可以带文件名） |
| L316-L324 | 被信号打断时**报信号的原因**（超时就说超时，不是"extractor broke off"）；抽取器无视信号跑完了也照样丢弃结果 |
| L326 | 结果先落 byte store（`onStored` 记下 key，好让失败路径能清自己的对象） |
| L327-L342 | 顺手对**这一份材料单独**做一次 bundle，得到"只看这一份"的截断信息，写进 `extraction.truncated` |
| L343-L353 | 结算摘要：`textChars / pageCount / imageCount / truncated / extractor / identityKey / resultKey / resultBytes` |

### 4.5 run 侧的等待：`awaitOwnerMaterialExtractions`

[L724-L783](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L724-L783)，本步的"心脏"：

```
records.length === 0 → 直接返回 []
① 有 idle 的：startOwnerMaterialExtractions(pool, owner, ids, ['idle']) + wakeOwnerMaterialExtractor()
② 循环轮询（pollMs = 500）：
   有 failed  → throw MaterialExtractionFailedError(材料 id, 该 extraction)
   全 ready   → 返回
   有 extracting 且 claimedAt 距今 > deadlineMs + 2×leaseTtlMs
              → throw MaterialExtractionTimeoutError（worker 死了，租约也过期了）
   信号 abort → 抛出 signal.reason
③ 每轮重读 getReadyOwnerMaterials；行不见了（等的时候被删）→ ClassroomMaterialsUnavailableError
```

宽限期 `graceMs = 300s + 2 × leaseTtlMs`（L746）解释得通：抽取本身最多 300 s，
再加两个租约周期容忍"worker 崩了但行还是 extracting"。

`MaterialExtractionFailedError` 继承 `StepRefusal<string>`，
**reason 直接就是那次抽取的 `errorCode`**
（[L704-L712](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L704-L712)），
于是 §10 的失败链能把它原样上报。

### 4.6 体积与配额

| 机制 | 代码 |
|---|---|
| 单份结果上限 | `maxExtractionResultBytes()`：`OPENMAIC_MATERIAL_EXTRACTION_MAX_RESULT_MB`，默认 100 MB（[L180-L191](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L180-L191)） |
| 超限拒绝 | `ExtractionResultTooLargeError`（**也是 `StepRefusal`**），消息建议"拆文件或减少图片"（[L193-L202](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L193-L202)） |
| owner 总字节配额 | 结算时带 `maxTotalBytes = agentRuntimeConfig.maxMaterialBytesPerOwner`，超了按 `MATERIAL_QUOTA_EXCEEDED` 失败并删自己的对象（[L469-L482](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L469-L482)） |
| 上传期单文件上限 | 媒体 `maxUploadBytes`，文档 `min(maxDocumentBytes, maxUploadBytes)`（[upload-limits.ts#L9-L12](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/upload-limits.ts#L9-L12)） |
| 孤儿结果清理 | `sweepOrphanExtractionResults`，年龄门槛 `2 × (300s + leaseTtlMs)`（[L651-L682](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L651-L682)） |
| 未用上传回收 | `unusedMaterialTtlMs()`（默认 24 h）→ `sweepUnusedOwnerMaterials` |

> 关于 `input.releaseMaterials`：它可以被持久化，但**成功落库的上传不会被删**
> （只有未完成的预留会过期），断言见
> [generation-runs.pg.test.ts#L811](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L811)。
> 材料的生命周期属于 owner 素材库，不属于 run。

---

## 5. 能力层：`analyzeMaterial()`（真正读字节的那 270 行）

签名与输入：[material-analysis.ts#L193-L196](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L193-L196)、
[L82-L94](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L82-L94)。

```ts
interface MaterialAnalysisInput {
  source: { fileName; fileSize; mimeType; buffer };  // 归一化后的字节（上传形式/资产形式都一样）
  services: ExtractionServices;                       // owner 的 document / asr 槽
  request: MaterialExtractorRequest;                  // deprecated 的 providerId/apiKey/baseUrl/AK/SK
  redactCallerInput: boolean;                         // 消息里能否回显调用方可控文本
  trace?: { resolvedProviderId?: string };            // 供调用方失败日志回填
}
```

`redactCallerInput` 是本文件最重要的一条安全约束：**同一句拒绝，
asset-id 形式必须通用、multipart 形式可以带文件名**
（[L87-L91](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L87-L91)、
[L364-L366](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L364-L366)）。

### 5.1 分流：媒体还是文档

第一刀在 [L236](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L236)：
`SUPPORTED_MEDIA_MIME_TYPES.includes(mimeType)`。两条路径**产出同一种形状**
（`ParsedPdfContent`），下游完全不区分——这是 L134-L138 注释强调的
"Media takes the same route + downstream path as documents; only the extraction differs"。

### 5.2 媒体路径（音频 / 视频）

| 步 | 行 | 做什么 |
|---|---|---|
| 能力校验 | [L241-L252](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L241-L252) | 请求点名了 provider 但它不在媒体注册表 / 不支持该 MIME → `provider-cannot-extract`（消息直接建议"选 AliDocMind 或本地 ffmpeg"），避免把文档型解析器（unpdf/mineru）塞进媒体链路产生玄学错误 |
| 槽服务优先 | [L255](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L255) | `slotMediaExtractorConfig`：document 槽配的就是 AliDocMind 且没点名 `local-ffmpeg` 时，媒体直接用槽的 AK/SK |
| legacy managed | [L256-L264](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L256-L264) | 非托管槽时，若服务端配了 AliDocMind 则 `resolveManagedAliDocMindCredentials()`（env **或** YAML），保证"只写 YAML 不写 env"的部署也能抽 |
| 端点规则 | [L265-L275](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L265-L275) | 客户端自带 baseUrl 必须过 `checkClientMediaExtractorBaseUrl`，否则 `endpoint-refused` |
| 抽取 | [L276-L301](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L276-L301) | `extractMedia(...)`：本地转写走 `services.asr`（asr 槽的连接），`ctx.signal` 一路透传 |
| 拍平 | [L139-L174](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L139-L174) | `## Synopsis` / `## Transcript`（`[MM:SS] 说话人: 文本`）/ `## Keyframes`（`[MM:SS] 描述或 OCR`） |
| 空内容拒绝 | [L309-L316](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L309-L316) | 三者皆空 → `no-content`，**绝不"用空文本继续生成"** |

时间戳规则：`formatTimestamp` 在超过一小时后切到 `HH:MM:SS`，
75 分钟的视频是 `01:15:03` 而不是 `75:03`
（[L176-L185](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L176-L185)）。
带时间戳的转录对第 5/6 步很有用：模型可以在讲稿里引用"视频 3 分 20 秒处"。

### 5.3 文档路径

```
documentSlotGoverns(services) ── governed 与否决定"谁说了算"
        │
        ├─ requestConfig.providerId 有 → getDocumentExtractorProvider
        │                                不存在 → unknown-provider
        ├─ provider 不支持该 MIME → 丢掉提示
        ├─ 仍无 provider → selectDocumentExtractorProvider({ mimeType, requiredCapabilities:{text:true} })
        │                    抛错 → unsupported-type
        ├─ governed && provider.requiresServiceConfig && 用的不是槽服务 → service-required
        ├─ 走槽服务 → extractorConfigFor + (非托管端点过 checkClientDocumentExtractorBaseUrl)
        │              不合规 → endpoint-refused
        └─ 走 legacy → managed 判定 / MinerU 云回退门控 / 客户端 baseUrl 校验 → extraction()
```

关键行：
- 槽 vs 请求：[L200-L203](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L200-L203)、
  [L335-L344](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L335-L344)
- `unknown-provider`：[L345-L350](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L345-L350)
- `unsupported-type`：[L354-L375](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L354-L375)
- `service-required`（槽配了但读不了这类文件）：[L378-L386](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L378-L386)
- 槽服务与端点校验：[L388-L400](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L388-L400)

### 5.4 本步最重要的一条隐私规则：MinerU 云回退门控

[L402-L433](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L402-L433)
配合 [L113-L122](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L113-L122)：

自托管 MinerU 被选中、但没配自托管 base URL 时，代码**不会**悄悄把文档送去
MinerU Cloud，而是抛 `service-required`，消息里写清"选的是自托管 MinerU、缺的是它的
base URL、没自动发到云端、要么配 URL、要么显式 `ALLOW_MINERU_CLOUD_FALLBACK=1`"。
只有运维显式打开开关（`true` 或 `1`）才允许回退。
学习时把这段当作"**私有部署的数据流向默认保守**"的范例。

### 5.5 统一的抽取包装器 `extraction()`

[L205-L231](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L205-L231)：
所有文档分支最后都汇到这里。它做三件事——
① 把 `ctx.signal` 折进 extractor 的 `config`（于是租约丢失能真的停掉 provider 请求与子进程）；
② `documentArtifactToParsedPdfContent(artifact)` 归一；
③ 回填 metadata：`pageCount ?? 0`、`fileName`、`fileSize`、`mimeType`、`parser ?? extractor.id`。
`parser` 就是后台抽取摘要里的 `extractor` 字段，便于排查"这份 PDF 到底是谁解析的"。

---

## 6. 打包预算：`buildDocumentBundle`

文件：[lib/document/bundle.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts)。
它把 N 份抽取结果拼成**一段带小节的文本 + 一组带 `visionPriority` 的图**，
且必须"多份材料共享一个提示词预算时不偏袒某一份"。

| 常量 | 值 | 位置 |
|---|---|---|
| `MAX_PDF_CONTENT_CHARS` | 50 000 | [constants/generation.ts#L7](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L7) |
| `MAX_VISION_IMAGES` | 20 | [#L10](file:///D:/GITHUB_chat/OpenMAIC/lib/constants/generation.ts#L10) |
| `MAX_DOCUMENT_BUNDLE_FILES` | 5 | [bundle.ts#L4](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L4) |
| `MAX_DOCUMENT_BUNDLE_TOTAL_SIZE_BYTES` | 150 MB | [bundle.ts#L5](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L5) |
| `BASE_BUDGET_PER_DOCUMENT` | 1 500 | [bundle.ts#L7](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L7) |
| `RESERVED_BUDGET_RATIO` | 0.4 | [bundle.ts#L8](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L8) |
| `SECTION_SEPARATOR` | `\n\n---\n\n` | [bundle.ts#L9](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L9) |

算法（[L181-L256](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L181-L256)）：

1. 按 `source.order` 排序；每份图片先换成**稳定 id** `doc_<order>_img_<n>`，并把正文里出现的旧 id 同步替换
   （`replaceImageIds` 用了 `(?<![\w-])…(?![\w-])` 的边界断言，避免误伤 `img_1` 匹配到 `img_10`）。
2. 生成小节头（[L60-L70](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L60-L70)）：
   ```
   ## Source Document 1: 讲义.pdf
   - Order: 0
   - MIME type: application/pdf
   - Pages: 24
   ```
3. `textContentBudget = 50 000 − 所有小节头长度 − 分隔符长度`。
4. 文本预算分配（[L72-L109](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L72-L109)）：
   先留出 `min(份数 × 1500, 预算 × 40%)` 作为"保底均分"，保证短文件不被长文件挤没；
   剩余预算再按各文件**未满足长度**的比例迭代分配（每轮至少 1 字符，直到分完或无缺口）。
5. 截断在**词边界**收尾（`truncateTextAtBoundary`，[L47-L58](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L47-L58)：
   往回退到非 `\p{L}\p{N}_-` 字符处，用 `Array.from` 按码点切，不劈开代理对/emoji）。
6. 图片 id 最终统一成 `img_1..img_N`（[L222-L223](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L222-L223)），正文里的引用同步替换。
7. 视觉图挑选（[L111-L163](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts#L111-L163)）：
   先**每份文档轮取一张**（按文档顺序，文档内按"有描述 > 页码 > 面积"排序），
   再轮询补足到 20 张——跨文档公平优先；被选中的图按顺序得到递减的 `visionPriority`。

`bundleTruncation`（[extraction.ts#L121-L135](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L121-L135)）
把 `totalRawTextLength > textContentBudget` 变成 `{ textChars }`、
`totalImageCount > MAX_VISION_IMAGES` 变成 `{ images: { total, max } }`——正是 §1.3 的事件数据。

---

## 7. 抽取服务从哪来：`resolveExtractionServices`

文件：[lib/server/material-extraction/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts)。
这是"槽配置 → 抽取器配置"的翻译层，也是本步**没有任何 key/model 出现在请求里**的原因。

| 概念 | 代码 | 说明 |
|---|---|---|
| `ExtractionServices` | [L32-L47](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L32-L47) | `document`（document 槽的服务或 null）、`documentStatus`（`configured/disabled/unassigned/locked/default`）、`asr`、`asrError`（延迟到真需要转写时才报） |
| 解析 | [L72-L118](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L72-L118) | 并行解析 document 槽与 asr 连接；`browser-native-tts`/浏览器 asr 在服务端不可用 → 置空 |
| `isSelfContainedExtractor` | [L125-L128](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L125-L128) | unpdf / 本地 pdfjs 之类"自己就能读"的解析器，不需要任何服务 |
| `documentSlotGoverns` | [L131-L137](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L131-L137) | 槽为 `configured/disabled/locked` 时**由槽说了算** |
| `slotGovernedRequest` | [L154-L164](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L154-L164) | 托管时只允许保留"自包含解析器选择或槽自己的 provider"，**永不保留请求里的 key/baseUrl/AK/SK** |
| `extractorConfigFor` | [L167-L188](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L167-L188) | 槽服务就是该 provider 时带上其凭据；`allowEnvFallback: false`——**key 只来自配置，不来自进程环境** |
| `slotMediaExtractorConfig` | [L204-L217](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts#L204-L217) | 媒体复用 document 槽的 AliDocMind 键对 |

`documentStatus` 的 `locked` 值得注意：`allowUserKeys: all` 之类的锁把槽**钉死为"无"**，
此时请求也不许点名任何 provider（L39-L40 注释）。

---

## 8. 产出的下游消费（为什么这一步值得单独一册）

| 消费点 | 代码 | 用法 |
|---|---|---|
| 第 2 步 research | [engine.ts#L634-L645](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L634-L645) | `pdfText` 作为检索的上下文，帮助改写查询词 |
| 第 3 步 outline | [engine.ts#L772-L780](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L772-L780) + [#L821](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L821) | 源文本 + `materialImages()`（`pdfImages` / `imageMapping`）；大纲可"建议配图" `suggestedImageIds` |
| 第 5 步 scene content | [engine.ts#L514-L516](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L514-L516) | 同一份 `materialImages()` 传进内容生成 |
| 视觉图解析 | [services.ts#L258-L287](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L258-L287) | `ownerVisionImages`：`assetId` → data URL；**资产不见了就抛 `MaterialImageMissingError` 让步骤失败**（宁可失败也不要"没有图地生成"），超尺寸只 `warn` 后丢弃 |
| 课堂复用 | [engine.ts#L960](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L960) | 材料阶段 mint 的 `stageId` 成为最终课堂 id |
| 资产保活 | [store.ts#L1283-L1290](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1283-L1290) | 回收器识别 `step_id = 'material-analysis'` 的检查点，**只要 run 还活着，材料图片就不被当孤儿删除** |

---

## 9. 事件与快照

| 通道 | 内容 |
|---|---|
| `step_started` | `{ step: 'material-analysis' }`（主循环统一发） |
| `material_kinds` | `{ kinds: ('document'\|'media')[] }`，顺序同 `materialIds`；只在还需要等时发（[types.ts#L275-L276](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L275-L276)） |
| `material_truncated` | `{ textChars?, images?: { total, max } }`（[types.ts#L277-L282](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L277-L282)） |
| `step_completed` | `{ step: 'material-analysis' }` |
| `state` | `preparing`（进入本步时）→ 下一步仍在 `preparing`，所以通常只有一次 |
| `step_failed` | `{ step, message }` + run `paused` |
| 快照 | `GET /api/generation-runs/:id` 会把这两个事件的最新值回灌成 `materialKinds` / `materialTruncated`（[[id]/route.ts#L8-L9](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/route.ts#L8-L9)、[#L69-L70](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/route.ts#L69-L70)），故 SSE 掉线后轮询快照也能恢复显示；取法见 [store.ts#L674-L679](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L674-L679) |

事件类型全表在总览册 §7。

---

## 10. 失败链：一次抽取失败会怎样被看见

四层链条（总览册 §9）在本步的具体化：

1. **能力层拒绝**：`StepRefusal<MaterialAnalysisRefusal>`，6 种 reason
   （[material-analysis.ts#L67-L80](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts#L67-L80)）：
   `provider-cannot-extract / unknown-provider / unsupported-type / endpoint-refused / no-content / service-required`。
   另有 `StepRefusal<'EXTRACTION_RESULT_TOO_LARGE'>`（结果超限）。
2. **抽取现场分类**：`failedExtraction()`（[extraction.ts#L360-L392](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts#L360-L392)）
   把失败写成行上的 `{ status:'failed', error, errorCode, retryable }`：
   模型授权拒绝保留其 code 且 `retryable:false`；host 分类优先；否则 `errorCode = StepRefusal.reason`
   或 `EXTRACTION_TIMEOUT / MATERIAL_QUOTA_EXCEEDED / EXTRACTION_FAILED`；
   `retryable` 由 `isTransientExtractionError` 判定（**只有 5xx / 超时 / 网络类错误可重试**，
   4xx 与配置类拒绝不重试，见 [errors.ts#L31-L67](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/errors.ts#L31-L67)）。
3. **run 侧再抛**：`awaitOwnerMaterialExtractions` 遇 `failed` 抛 `MaterialExtractionFailedError`（reason 即上面的 `errorCode`）；
   等待超时抛 `MaterialExtractionTimeoutError`；材料被删抛 `ClassroomMaterialsUnavailableError`；
   准入不通过抛 `ClassroomMaterialsRejectedError`；资产满员直接 `Error`。
4. **归一与暂停**：主循环包成 `StepFailedError`（[engine.ts#L1826-L1839](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1826-L1839)），
   `failureCodeOf()` 先问 host 分类器、再 `runFailureCode()`
   （[failure-code.ts#L41-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L41-L65)：
   `StepRefusal → GENERATION_FAILED`；带 HTTP 状态则 `RATE_LIMITED / UPSTREAM_ERROR`；
   配额 `PROVIDER_QUOTA_EXHAUSTED`；其余 `INTERNAL_ERROR`），
   最后 `pause(stepId, message, code)` → `state='paused'`、`resumeState='preparing'`。

**恢复路径**：`POST /api/generation-runs/:id/retry` →
[store.ts#L1578-L1586](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1578-L1586)
特判 `run.step === 'material-analysis'`：**先把该 run 材料里 `failed` 的抽取重新排队**
（`restartFailedRunMaterialExtractionsIn`，只重启 owner 现在仍持有的材料，
[#L1610-L1630](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/store.ts#L1610-L1630)），
再把状态设回 `preparing` 并清零 `takeovers`。
配置类失败（如没配 MinerU base URL）必须先改配置再 Retry，否则同样结果。
用户也可以在素材库侧单独重启：`POST /api/materials/:id/extraction`
（[[id]/extraction/route.ts#L38-L56](file:///D:/GITHUB_chat/OpenMAIC/app/api/materials/[id]/extraction/route.ts#L38-L56)）。

---

## 11. 常量与配置一览

| 名称 | 值 / 来源 | 作用 |
|---|---|---|
| `STEP_DEADLINES_MS.materialAnalysis` | 300 000 ms | 单次抽取预算 |
| `pollMs`（等待轮询） | 500 ms | `awaitOwnerMaterialExtractions` |
| `graceMs` | 300 s + 2 × `leaseTtlMs` | 判定"抽取进程已死" |
| `OPENMAIC_MATERIAL_EXTRACTION_CONCURRENCY` | 默认 2 | 单进程并发抽取数 |
| `OPENMAIC_MATERIAL_EXTRACTION_PER_OWNER` | 默认 2 | 跨进程、单 owner 并发数 |
| `OPENMAIC_MATERIAL_EXTRACTION_MAX_RESULT_MB` | 默认 100 | 单份结果对象上限 |
| `OPENMAIC_UNUSED_MATERIAL_TTL_HOURS` | 默认 24 | 未使用上传回收时限 |
| `ALLOW_MINERU_CLOUD_FALLBACK` | **默认关闭** | 自托管 MinerU → MinerU Cloud 的唯一开关 |
| `MAX_EXTRACT_DOCUMENT_FILE_SIZE_BYTES` | 50 MB | 单材料/单视觉图字节上限 |
| `MAX_CLASSROOM_MATERIALS` / 总量 | 5 份 / 150 MB | 一次生成的材料规模 |
| `MAX_VISION_IMAGES` / 尺寸 / base64 | 20 张 / 2048 px / 1 MB | 视觉预算（详见第 5 册） |
| `agentRuntimeConfig.scanIntervalMs` | 运行时配置 | 抽取 worker 的扫描周期 |

---

## 12. 容易踩的坑（学习者清单）

1. **"材料分析很慢"通常是排队，不是解析慢**：每 owner 只允许 2 个并发抽取；
   run 侧等待不设总时限，所以长时间 `preparing` 时先看 `owner_material.extraction.status` 和 `claimedAt`。
2. **run 步骤本身不会重试**：本步没套 `withRouteRetry`，`step_retry` 事件不会出现；
   重试语义完全走"重启失败抽取 + Retry 命令"。
3. **顺序有意义**：`materialIds` 去重后保序（[input.ts#L110-L121](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/input.ts#L110-L121)），
   bundle 的 `## Source Document N` 与图片稳定 id 都依赖 `order`。
4. **截断是静默的吗？** 不是——但只在 bundle（多份合并）层面发 `material_truncated`；
   单份材料的截断记在行上的 `extraction.truncated`，两者口径不同（§4.4 的 L327-L342）。
5. **图片不检查点里**：检查点存的是 `assetId`，不是 base64；如果课堂里图不见了，
   先怀疑资产被回收或 run 的 keepAlive 逻辑（§8 末行）。
6. **`analyzeMaterial` 的两种调用现场参数不同**：后台抽取传 `request: {}` +
   `redactCallerInput: false`；资产形式的 API 传 `redactCallerInput: true`，
   所以**同一种失败在两条路上消息文本不一样**，不要据此认为是两个 bug。
7. **音视频走媒体路径但仍受同一预算**：转写结果很长时同样会被 50k 字符预算裁掉。
8. **视频有音轨但服务端没配 ASR**：`resolveExtractableMimeTypes` 仍会把该视频 MIME 列为可抽
   （无音轨视频可读），**运行时才失败**（[availability.ts#L34-L36](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/availability.ts#L34-L36) 注释）。
9. **run 不删材料**：无论 `releaseMaterials` 为何，成功上传都保留；删除属于素材库操作。

---

## 13. 代码地图

| 文件 | 本册关注的职责 |
|---|---|
| [steps/material-analysis.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/material-analysis.ts) | 能力层：媒体/文档双路径、6 种拒绝、MinerU 门控、`ParsedPdfContent` 归一 |
| [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) | `case 'material-analysis'`、`storeMaterialImages`、`materialImages`、`MaterialOutput` |
| [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) | `materialKinds` / `materialsReady` / `analyzeMaterials`、`ownerVisionImages` |
| [run/plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) | 条件步入计划、`phaseOfStep` |
| [run/deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | `materialAnalysis: 300_000` |
| [materials/extraction.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/extraction.ts) | 后台抽取：租约/心跳/复用/配额/等待/清扫 |
| [materials/bytes.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/materials/bytes.ts) | 字节存储与 `materialExtractionResultKey` |
| [material-extraction/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/services.ts) | 槽 → 抽取服务配置 |
| [material-extraction/availability.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/availability.ts) | 本服务器能抽哪些 MIME |
| [material-extraction/errors.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/material-extraction/errors.ts) | `isTransientExtractionError` |
| [document/bundle.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/document/bundle.ts) | 打包与预算 |
| [classroom-materials.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/classroom-materials.ts) | 准入校验与上限 |
| [persistence/owner-materials.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/persistence/owner-materials.ts) | `owner_material` 行、抽取认领/心跳/结算 |
| [run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts) / [run/retry.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/retry.ts) | 失败码归一（本步不走 route retry） |

---

## 14. 动手验证

| 想验证什么 | 怎么做 |
|---|---|
| run 步骤的等待/事件 | [tests/generation/preview-material-step.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/generation/preview-material-step.test.ts)：喂 `materialKinds` 快照 + `material_kinds` 事件，看 UI 是否**没有**多出一步 |
| 抽取与结算 | `tests/` 下搜 `extractOwnerMaterial` / `awaitOwnerMaterialExtractions` 的用例；把 `deadlineMs` 传小即可复现 `MaterialExtractionTimeoutError` |
| 引擎真实跑通 | [tests/server/generation-runs.pg.test.ts#L761-L763](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-runs.pg.test.ts#L761-L763) 用假 `services.analyzeMaterials` 驱动带 `materialIds` 的 run，并断言事件里出现 `material_kinds` |
| 复用与身份 | 同字节上传两次、中间改 document 槽配置，观察日志 `reused the extraction of ...` 是否消失 |
| MinerU 门控 | 只配 provider 为 `mineru` 不配 base URL，抽 PDF 应失败且消息里出现 `ALLOW_MINERU_CLOUD_FALLBACK`；设 `=1` 后改走 cloud |
| 预算分配 | 直接对 `buildDocumentBundle` 写小样：两份 40k/2k 字符文本，验证短文件不被挤没、长文件在词边界收尾 |
| Retry 重启抽取 | 造一个 `failed` 材料 + 停在 `material-analysis` 的 paused run，发 `retry` 命令，检查 `owner_material.extraction.status` 由 `failed` 变 `extracting` |

---

## 15. 一句话总结

**第 1 步 = 一份"上传即抽取"的后台产物 + 一次带预算的多文档打包 + 图片转课程资产。**
run 里的 `material-analysis` 主要在做**等待、读取、打包、落库、上报截断**，
真正的解析在 `analyzeMaterial()`（媒体或文档二选一，产出同一种文本形状），
而它的一切凭据都来自 owner 的 document / asr 槽——请求里永远不带 key。
