# 课程生成管线 · 第 2 册：联网检索（research）

<!--
  职责：详解生成管线**第 2 步「联网检索」**——把课程需求（必要时加上材料节选）改写成
  一条搜索词，向搜索引擎问一次，把返回压成一段可进 prompt 的上下文，并把"这次到底搜
  没搜"作为布尔位传给下游。
  覆盖：run 步骤分支逐行解读、为什么它是**无条件步**却可以"空跑并检查点化自己什么都没
  做"、`services.research` 的三级门控、`buildSearchQuery` 的触发条件与五级回落、
  12 家 provider 的派发与 Tavily 默认路径、两级查询截断（350/400）、
  `formatSearchResultsAsContext` 的压缩规则、配置层（`webSearch` 槽 + `course.research`
  槽 + legacy 请求路径 + 三类错误码）、产出如何被第 3 步大纲与第 5 步首场景消费、
  事件/检查点/快照三者为何不一致（重连丢来源列表）、失败链与 Retry 语义、
  相关常量与配置、动手验证方式。
-->

- 总览：[generation-flow.md](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow.md)
- 上一步：[第 1 册 材料分析](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-1.md)｜下一步：[第 3 册 大纲生成](file:///D:/GITHUB_chat/OpenMAIC/docs/flow/generation-flow-3.md)
- 源文件：[steps/research.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts)
  · [search-query-builder.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts)
  · [web-search/index.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/index.ts)
  · [web-search-config.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts)

---

## 0. 阅读提示

### 0.1 本步在全局中的坐标

| 项 | 值 |
|---|---|
| step kind / step id | `research`（id 与 kind 同名，无序号） |
| 是否必经 | **无条件步**：`preparationSteps` 里写死，见 [plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54) |
| 能否"什么也不做" | 能。能力不满足时 `services.research` 返回 `null`，本步照样检查点化 `webSearch: false` |
| 所处 run 状态 | `preparing`（[plan.ts#L67-L72](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L67-L72) 的 `phaseOfStep`） |
| 检查点写在哪 | `generation_run_steps(run_id, 'research')`，值为 `ResearchOutput`（[engine.ts#L249-L253](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L249-L253)） |
| 时间预算 | 300 s（[deadline.ts#L14-L15](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L14-L15)） |
| 进程内重试 | **无**。本步不套 `withRouteRetry`，因此**不会**出现 `step_retry` 事件 |
| 用到的模型槽 | `course.research`（改写查询的对话模型，父槽 `llm`）；搜索本身用 `webSearch` 能力槽 |
| UI 显示 | 预览步 `web-search`——但**只有 `capabilities.webSearch` 为真时才列出**（[preview-steps.ts#L22-L33](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L22-L33)）；第 1 步 `material-analysis` 也折叠在这一位上 |
| 产出消费方 | 第 3 步 `outline`（`webSearch` 布尔 + `context` 文本）、第 5 步 `scene:0:content`（`webSearch` 布尔） |

### 0.2 一句话本质

> **这一步只做三件事：把需求变成一条搜索词、问一次搜索引擎、把结果压成一段文本。**
> 它产出的"知识"只有两种下游形态——一个 `webSearch: true` 的开关位，
> 和一段进大纲 prompt 的 markdown 上下文；其余（`answer`、`content`、`score`）都在这一步内被丢弃。

所以理解本步的关键不是"搜了什么"，而是**"没配搜索时会发生什么"**：整步空跑、
检查点写 `webSearch: false`、不发任何事件、下游按"未联网"生成。这与
[plan.ts#L1-L12](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L1-L12)
头注释的承诺一致——"计划永远不依赖在另一个时刻读到的配置"。

### 0.3 三个"现场"，同一个 `searchWeb`

```
                     lib/web-search/index.ts  searchWeb(params)   ← 12 家 provider 的纯派发
                                ▲                    ▲                    ▲
        ┌───────────────────────┘                    │                    └───────────────────────┐
        │ 现场 1：run 第 2 步（本册）        现场 2：agent 运行时工具          现场 3：课堂内聊天工具
        │ steps/research.ts                server/agent-runtime/web-search.ts  chat/pi/tools/web-search.ts
        │ 生成前一次性检索                 `web_search` 工具（能力注册才存在）  模型自己决定何时搜
        ▼                                  ▼                                  ▼
   能力门控：resolveServerGenerationCapabilities(workspaceId).webSearch
             ——false 就 return null，本步空跑
   配置解析：resolveWebSearchConnection(...)  ——抛错型
             （现场 2/3 用 resolveClassroomWebSearchConfig(...)，把"没配"吞成 undefined）
```

三处**共用同一份 provider 注册表与同一套解析规则**，但对"配置不完整"的态度不同：
run 侧只做了能力探针那一层"温柔"处理，`resolveWebSearchConnection` 的抛错会直接让本步失败（§10）。

---

## 1. run 侧逐段：`case 'research'`

入口在 [engine.ts#L634-L658](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L634-L658)，整段只有 20 余行：

```ts
case 'research': {
  const pdfText = output<MaterialOutput>('material-analysis')?.pdfText;
  const result = await withDeadline(
    stepId,
    STEP_DEADLINES_MS.research,
    signal,
    (callSignal) =>
      services.research(
        owner,
        { query: input.requirement, ...(pdfText ? { pdfText } : {}) },
        { log, signal: callSignal },
      ),
  );
  const sources = (result?.sources ?? []).map((source) => ({
    title: source.title,
    url: source.url,
  }));
  return done(
    {
      webSearch: result !== null,
      ...(result ? { context: result.context || '', sources } : {}),
    } satisfies ResearchOutput,
    result ? { events: [{ type: 'research_sources', data: { sources } }] } : {},
  );
}
```

逐点解读：

| 行 | 事实 | 含义 |
|---|---|---|
| `output<MaterialOutput>('material-analysis')?.pdfText` | 读第 1 步的检查点 | 材料是**可选输入**：没上传材料时 `pdfText` 为 `undefined`，改写就只看需求本身 |
| `{ query: input.requirement, ... }` | 送进检索的是**原始需求**，不是改写后的词 | 改写发生在 `services.research` 里面（§4），engine 不参与 |
| `withDeadline(stepId, 300_000, signal, ...)` | 只有 deadline，**没有** `withRouteRetry` | 一次机会：provider 抖动 500 直接失败并暂停 run（对比第 5/6/7 步都套了 route retry，见 [engine.ts#L509](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L509)、[#L717](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L717)、[#L1028](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L1028)） |
| `result?.sources ?? []` + `map({title,url})` | **降形**：`content` / `score` 在这里被丢掉 | 检查点和事件里只有标题与 URL；正文只活在本步的 `context` 里 |
| `webSearch: result !== null` | 布尔位来自"能力是否解析成功"，而非"搜到了几条" | 搜到 0 条也记 `webSearch: true`，下游仍按"已联网"生成 |
| `...(result ? { context: result.context \|\| '', sources } : {})` | 空跑时 `context`/`sources` **字段都不存在** | 下游用 `research?.context ?` 判断，见 §8 |
| `result ? { events: [...] } : {}` | 空跑**不发事件** | UI 的来源列表只在真搜过时才被填 |

`done(...)` 是 engine 内部提交器：写 `generation_run_steps` 检查点 + 附带事件在同一事务里落库（详见总览册的 run 存储一节）。

`ResearchOutput` 的形状在 [engine.ts#L249-L253](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L249-L253)：

```ts
interface ResearchOutput {
  webSearch: boolean;
  context?: string;
  sources?: Array<{ title: string; url: string }>;
}
```

---

## 2. 为什么它是"无条件步"

### 2.1 计划里写死

[plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54)：

```ts
export function preparationSteps(input: Pick<GenerationRunInput, 'materialIds'>): RunStep[] {
  return [
    ...(input.materialIds.length > 0
      ? [{ id: 'material-analysis', kind: 'material-analysis' } as const]
      : []),
    { id: 'research', kind: 'research' },
    { id: 'outline', kind: 'outline' },
  ];
}
```

对比第 1 册的 `material-analysis`（条件步）：**`research` 前面没有 `...` 三元**，任何 run 都有这一步。
`advanceRun` 是纯函数（[#L86-L104](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L86-L104)），只看"状态 + 已完成集合"，
所以计划本身绝不可能因为"这一刻搜索槽配没配"而变化——这是 [头注释 #L8-L11](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L8-L11) 明写的取舍：

> a step whose capability does not resolve (no web search, no server TTS) still runs and
> checkpoints that it did nothing, so the plan never depends on configuration read at another moment.

### 2.2 三条相关推论

| 推论 | 依据 |
|---|---|
| 没配搜索槽的 run 也会**多花一次 worker 时间片**（只是极短：一次能力探针） | [plan.ts#L93-L97](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L93-L97) 仍把它排为下一步 |
| `advanceRun` 不会"跳过"它；`completed` 集合里一定要有 `research` 才能进入 `await-outline-confirmation` | 同上（`find(step => !completed.has(step.id))`） |
| 接管（takeover）后重放也一样：读检查点即知"上次到底搜没搜" | 检查点里的 `webSearch` 布尔位 |

### 2.3 启动期的槽位预检拦不住它

`webSearch` 在启动预检的槽位列表里（[start.ts#L34-L40](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L34-L40)）：

```ts
export const GENERATION_MODEL_SLOTS: readonly SlotId[] = [
  ...MODEL_SLOTS.filter((slot) => slot.id.startsWith('course.')).map((slot) => slot.id as SlotId),
  'image',
  'video',
  'tts',
  'webSearch',
];
```

但 [#L97](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L97) 调的 `assertWorkspaceModelsAuthorized` **只对 `ModelAuthorizationError` 上抛**，
"槽位没分配 / 被关闭"不在此列。结论：**没配搜索槽不会导致启动被拒**，只会让本步空跑。
这与总览册 §9.2 表里"联网检索：无 refusal（能力没配就返回 null）"一行是同一件事。

---

## 3. `services.research`：三级门控

契约在 [services.ts#L182-L187](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L182-L187)，注释即语义：

```ts
/** Research the requirement; null when the webSearch slot resolves to nothing. */
research(ownerId, input: { query: string; pdfText?: string }, ctx: StepContext): Promise<ResearchResult | null>;
```

实现在 [services.ts#L419-L442](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L419-L442)：

```ts
async research(ownerId, input, ctx) {
  const workspaceId = await backgroundWorkspaceId(ownerId);
  const capabilities = await resolveServerGenerationCapabilities(workspaceId);
  if (!capabilities.webSearch) return null;                       // ① 能力探针：布尔，永不抛
  const config = await resolveWebSearchConnection(                // ② 配置解析：会抛
    workspaceId,
    {},
    { refuseDisabled: true, preferServerProvider: true, fallbackProviderId: DEFAULT_WEB_SEARCH_PROVIDER_ID },
  );
  let rewriteModel: ResolvedModel | undefined;
  try {
    rewriteModel = await stageModel(ownerId, 'web-search-query-rewrite');
  } catch (error) {
    ctx.log.warn('Search query rewrite model unavailable, falling back to raw requirement:', error);
  }
  return research({ ...input, config, rewriteModel }, ctx);       // ③ 真正干活
}
```

| 级 | 失败时的行为 | 为什么这样设计 |
|---|---|---|
| ① `resolveServerGenerationCapabilities` | **返回 `null`**，本步空跑 | 探针把 `resolveMediaSlot` 的抛错 `try/catch` 成布尔（[generation-capabilities.ts#L11-L19](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-capabilities.ts#L11-L19)），且 health / capabilities / 管线三处共用同一函数，"告诉用户能用的"与"实际用的"必然一致（[#L21-L30](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-capabilities.ts#L21-L30)） |
| ② `resolveWebSearchConnection` | **抛错 → 本步失败 → run 暂停** | 走到这里说明槽已 assigned，配置却拼不出可用 provider（缺 key、缺 baseUrl、被强制关闭）——这是"运维配错了"，不该静默跳过 |
| ③ `stageModel(... 'web-search-query-rewrite')` | **warn + `rewriteModel` 留空**，继续搜原文 | 改写是锦上添花：[research.ts#L41-L60](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L41-L60) 里没有模型就 `aiCall = undefined`，`buildSearchQuery` 第一级回落即可 |

注意 ① 与 ② 之间存在一个**窄缝**：探针用的是不带 `legacyRequest` 的 `resolveMediaSlot(slot, { workspaceId })`
（[generation-capabilities.ts#L13](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-capabilities.ts#L13)），
而 ② 用的是带 legacy 规则的版本（§7.3）。探针判"能用"、② 却抛错的场景是真实存在的（§12 坑 5）。

`rewriteModel` 走的是 **`course.research` 槽**：`stage → slot` 映射在
[model-slots.ts#L111](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L111)（`'web-search-query-rewrite': 'course.research'`），
槽定义在 [#L59](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L59)（`capability: 'chat'`, `parent: 'llm'`）——
所以没单独配 `course.research` 时会继承 `llm`。`stageModel` 本身在 [services.ts#L246-L249](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L246-L249)，
它每次都按"当前 owner"重解析（注释：a claim may have moved it），这与接管语义配套。

---

## 4. 查询改写：`buildSearchQuery`

源文件 99 行：[search-query-builder.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts)。

### 4.1 先归一化

```ts
// #L22-L24 需求：所有连续空白压成一个空格再 trim
function normalizeSearchRequirement(requirement: string): string {
  return requirement.replace(/\s+/g, ' ').trim();
}

// #L26-L32 材料节选：同样压空白，然后取前 7000 字符
function normalizePdfExcerpt(pdfText?: string): string {
  if (!pdfText) return '';
  return pdfText.replace(/\s+/g, ' ').trim().slice(0, SEARCH_QUERY_REWRITE_EXCERPT_LENGTH);
}
```

`research()` 里还有一层同样的截断（[#L38](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L38)），
两处都引用 `SEARCH_QUERY_REWRITE_EXCERPT_LENGTH = 7000`
（[search-query-builder.ts#L8](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L8)）——即"再长的材料，也只有前 7000 字符能影响搜索词"。

### 4.2 要不要改写

```ts
// #L34-L39
function shouldRewriteSearchQuery(normalizedRequirement, normalizedPdfExcerpt): boolean {
  return normalizedRequirement.length > 400 || Boolean(normalizedPdfExcerpt);
}
```

| 情况 | 是否尝试改写 |
|---|---|
| 需求 ≤ 400 字符且无材料 | **不改写**，直接用归一化后的原文 |
| 需求 > 400 字符 | 改写（短词搜索引擎更吃得住） |
| 有材料文本（哪怕需求很短） | 改写（让模型从材料里推断标题/作者/术语） |

### 4.3 五级回落

[buildSearchQuery #L41-L99](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L41-L99) 按顺序检查，任何一级不满足都**返回同一个 `fallback` 对象**（原文 + 元信息），并 `log.warn`：

| 级 | 条件 | 日志 |
|---|---|---|
| 1 | `!normalizedRequirement \|\| !rewriteAttempted` | 无（正常路径，不算回落） |
| 2 | `!aiCall`（没有改写模型） | `Query rewrite AI call unavailable, falling back to raw requirement` |
| 3 | `!prompts`（模板没找到） | `Query rewrite prompt not found, falling back to raw requirement` |
| 4 | 解析后为空串 | `Query rewrite returned empty output, falling back to raw requirement` |
| 5 | 抛异常（JSON 解析失败、模型超时等） | `Query rewrite failed, falling back to raw requirement: ...` |

**关键设计：改写永不失败整步。** 五级的出口都是"用原文搜"。

### 4.4 prompt 与软截断

```ts
// #L68-L71
const prompts = buildPrompt(PROMPT_IDS.WEB_SEARCH_QUERY_REWRITE, {
  requirement: normalizedRequirement,
  pdfExcerpt: pdfExcerpt || 'None',
});
```

- prompt id：`'web-search-query-rewrite'`（[prompts/index.ts#L29](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/index.ts#L29)）
- 模板：[templates/web-search-query-rewrite/system.md](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/templates/web-search-query-rewrite/system.md)
  与 [user.md](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/templates/web-search-query-rewrite/user.md)。
  system 里要求"只返回 `{"query": ...}`""**Keep the query under 320 characters**"，
  并明确"忽略样板文字/版权页/页码""优先具体术语而非 'this paper'"。
- 解析：`parseJsonResponse<SearchQueryRewriteResponse>`（来自 `@openmaic/generation`）
- 软截：**350** 字符（[#L7 `TAVILY_SOFT_MAX_QUERY_LENGTH`](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L7)、[#L81-L84](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L81-L84)）

于是有三层"长度约束"，数字各不相同，很容易记混：

| 层 | 值 | 位置 | 性质 |
|---|---|---|---|
| prompt 要求模型 | 320 | system.md #L14 | 软约束（靠模型自觉） |
| 代码软截 | 350 | search-query-builder.ts #L7 | 兜底截断 |
| Tavily 硬截 | 400 | tavily.ts #L14/#L34 | **超过就 400 报错**，注释原话：*"Tavily rejects queries over 400 characters with a 400 error"* |

### 4.5 改写这次 LLM 调用长什么样

[research.ts#L41-L60](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L41-L60)：

```ts
const aiCall: AICallFn | undefined = rewriteModel
  ? async (systemPrompt, userPrompt) => {
      const result = await callLLM(
        {
          model: rewriteModel.model,
          abortSignal: ctx.signal,
          messages: [
            { role: 'system', content: systemPrompt },
            { role: 'user', content: userPrompt },
          ],
          maxOutputTokens: 256,
        },
        'web-search-query-rewrite',
        undefined,
        rewriteModel.thinkingConfig,
        { serverManaged: rewriteModel.serverManaged },
      );
      return result.text;
    }
  : undefined;
```

要点：`maxOutputTokens: 256`（一条查询词用不完，省 token）；带 `ctx.signal`（**能被 abort**）；
带槽位解析出的 `thinkingConfig` 与 `serverManaged` 标记；请求里**永远不带用户 key**。

### 4.6 本步唯一的 info 日志

```ts
// research.ts #L64-L69
ctx.log.info('Running web search API request', {
  hasPdfContext, rawRequirementLength, rewriteAttempted, finalQueryLength,
});
```

排查"为什么搜出来的东西不对"时，这四个字段就够判断走没走改写、材料有没有参与。

---

## 5. provider 派发：`searchWeb` 与 Tavily 默认路径

### 5.1 派发函数

[web-search/index.ts#L18-L91](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/index.ts#L18-L91) 是一个纯 switch，12 家 provider 各配一个 `searchWith*`：

| providerId | 传输特点（本册只关心形状） |
|---|---|
| `tavily` | REST `POST /search`，Bearer key（默认，§5.3） |
| `exa` | REST `POST /search` |
| `firecrawl` | `POST /v2/search`（与 web fetch 共用一份 key/base URL） |
| `bocha` | `POST /v1/web-search` |
| `brave` | **`requiresApiKey: false`**，调用时 `apiKey: apiKey \|\| undefined` |
| `baidu` | `POST /v2/ai_search/web_search`，带 `subSources` |
| `claude` | 走 `/messages` + 服务端搜索工具，需要 `claudeModelId` |
| `kimi` | `/tools/search_pro`，受 `WEB_SEARCH_KIMI_ENABLED` 开关 |
| `minimax` | `/v1/coding_plan/search` |
| `doubao` | `open.feedcoopapi.com` `/search_api/web_search` |
| `searxng` | **不收 apiKey**（`#L83` 只传 `query/maxResults/baseUrl`），`requiresBaseUrl: true` |
| `deepseek` | `/tools/search`，受 `WEB_SEARCH_DEEPSEEK_ENABLED` 开关 |

兜底分支用 `never` 保证编译期穷尽（[#L86-L89](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/index.ts#L86-L89)）。
`signal` 的透传方式是 `const abortOptions = signal ? { signal } : {}`（[#L39](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/index.ts#L39)），
即不传时**不给各家塞一个 `signal: undefined`**。

注册表本体在 [web-search/constants.ts#L23-L133](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L23-L133)，
每项声明 `requiresApiKey` / `requiresBaseUrl` / `defaultBaseUrl` / `endpointPath` / `icon`，
部分还有 `officialBaseUrls`、`enableEnvVar`、`sharedChatProviderId`、`models`。

### 5.2 本步**没有**把 abort 信号接到搜索上

[research.ts#L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L71)：

```ts
const result = await searchWeb({ ...input.config, query: searchQuery.query });
```

`input.config` 是 `WebSearchConfig`，其类型里**有**可选 `signal`
（[#L126](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L126)：`Omit<Parameters<typeof searchWeb>[0], 'query' | 'apiKey'> & { apiKey: string }`），
但 `resolveWebSearchConnection` 从不填它，run 也没填。后果：

- 改写调用（`callLLM`）**会**被 `ctx.signal` 打断；
- 搜索引擎的 HTTP 请求**不会**被打断——`withDeadline` 只是让本步先 reject（[deadline.ts#L52-L63](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L52-L63)），
  provider 侧请求仍在飞。

### 5.3 Tavily 细节

[tavily.ts#L24-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/tavily.ts#L24-L54)：

```ts
const { query, apiKey, maxResults = 5, baseUrl, signal } = params;
// Tavily rejects queries over 400 characters with a 400 error
const truncatedQuery = query.slice(0, TAVILY_MAX_QUERY_LENGTH);   // 400

const res = await proxyFetch(buildTavilySearchUrl(baseUrl), {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${apiKey}` },
  body: JSON.stringify({
    query: truncatedQuery,
    search_depth: 'basic',
    max_results: maxResults,
    include_answer: 'basic',
  }),
  ...(signal ? { signal } : {}),
});
```

| 事实 | 影响 |
|---|---|
| `maxResults = 5` 默认（run 不传 `maxResults`） | 一次 run 大约拿 5 条来源 |
| `search_depth: 'basic'` | 不用 advanced 档（更贵、更慢） |
| `include_answer: 'basic'` | 有 `answer`，它成为 `context` 的第一段（§6） |
| `buildTavilySearchUrl`：结尾去 `/`，非 `/search` 则补 | 自定义 base URL 只填域名也能用 |
| 非 2xx → `throw new Error('Tavily API error (500): ...')` | **普通 Error**，不带 statusCode → 失败码归 `INTERNAL_ERROR`（§10） |
| 走 `proxyFetch` | 尊重部署的代理设置 |

返回类型统一为 `WebSearchResult`（[types/web-search.ts#L8-L13](file:///D:/GITHUB_chat/OpenMAIC/lib/types/web-search.ts#L8-L13)），
每条来源是 `{ title, url, content, score }`（[#L1-L6](file:///D:/GITHUB_chat/OpenMAIC/lib/types/web-search.ts#L1-L6)）。
`ResearchResult` 就是它加一个 `context`
（[research.ts#L31-L34](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L31-L34)）。

---

## 6. 结果压缩：`formatSearchResultsAsContext`

[web-search/format.ts#L6-L26](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/format.ts#L6-L26)（全文 26 行）：

```ts
export function formatSearchResultsAsContext(result: WebSearchResult): string {
  if (!result.answer && result.sources.length === 0) return '';

  const lines: string[] = [];
  if (result.answer) { lines.push(result.answer); lines.push(''); }
  if (result.sources.length > 0) {
    lines.push('Sources:');
    for (const src of result.sources) {
      lines.push(`- [${src.title}](${src.url}): ${src.content.slice(0, 200)}`);
    }
  }
  return lines.join('\n');
}
```

产出的形状（假设 2 条结果）：

```
Tavily 给出的 answer 文本（一段综述）

Sources:
- [标题 A](https://a.example): 正文前 200 字符
- [标题 B](https://b.example): 正文前 200 字符
```

| 规则 | 预算 | 备注 |
|---|---|---|
| `answer` | 不截断 | provider 自己控制长度（Tavily `include_answer: 'basic'`） |
| 每条来源正文 | `content.slice(0, 200)` | 硬截字符，不看词边界 |
| 条数 | provider 的 `maxResults`（Tavily 默认 5） | 无额外上限 |
| 空结果 | `''`（空串） | 于是 `webSearch: true` 但 `context: ''` 是合法组合 |

粗算 `context` 上界：`answer` + 5 × (标题 + URL + 200) —— 通常几百到两三千字符，**没有**全局截断，
这也意味着 provider 若返回超长 `answer`，它会整段进大纲 prompt（§12 坑 7）。

注意 `''` 之后在 engine 里的写法是 `context: result.context || ''`（[engine.ts#L654](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L654)），
下游判定用**真值**，因此"搜到了但一条来源也没有"= 大纲按"无检索上下文"走（见 §8）。

---

## 7. 配置层：两个槽、一套 legacy 规则、三类错误

本步同时受两个槽影响：`webSearch`（搜不搜、怎么搜）与 `course.research`（改写词用的模型）。
设置界面的拓扑图里它们被画在同一站："research" 站包含 `course.research` 的 rewrite 行和 `webSearch` 的 search 行
（[model-settings/diagram.ts#L55-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/model-settings/diagram.ts#L55-L65)）。

### 7.1 类型与错误

```ts
// web-search-config.ts #L126-L128
export type WebSearchConfig = Omit<Parameters<typeof searchWeb>[0], 'query' | 'apiKey'> & {
  apiKey: string;
};

// #L130-L140
export class WebSearchConfigError extends Error {
  constructor(
    readonly code: 'MISSING_API_KEY' | 'MISSING_REQUIRED_FIELD' | 'INVALID_REQUEST',
    message: string,
    readonly providerId?: WebSearchProviderId,
  ) { ... }
}
```

`WebSearchConfig` 用 `Omit` 从 `searchWeb` 的参数反推，**新增一个搜索参数不会漏改配置类型**——
这是这套类型设计的用意，也是 §5.2 里"它其实有 `signal` 字段"的原因。

### 7.2 `webSearchConfigFromConnection`：唯一会抛 `WebSearchConfigError` 的地方

[web-search-config.ts#L220-L269](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L220-L269)：

| 抛点 | 码 | 条件 |
|---|---|---|
| [#L224-L226](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L224-L226) | `INVALID_REQUEST` | providerId 不在注册表里 |
| [#L230-L236](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L230-L236) | `MISSING_API_KEY` | `provider.requiresApiKey && !apiKey` |
| [#L248-L254](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L248-L254) | `MISSING_REQUIRED_FIELD` | `provider.requiresBaseUrl && !baseUrl`（只有 searxng 是这个形态） |

另外两处不抛但改变结果：

- **workspace provider 的用户自定义 base URL 被收编**：`origin === 'configuration' && !managed && baseUrl && !requiresBaseUrl` 时走 `resolveSafeClientWebSearchBaseUrl`（[#L237-L247](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L237-L247)），即必须落在该 provider 的官方域名上——**searxng 例外**（它本来就靠运维指定 URL）。
- **模型授权**：`authorizeMediaModel(connection, 'webSearch', model)`（[#L261](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L261)）；`claude` 这类带 `models` 的 provider 会校验选中模型是否被允许。

### 7.3 `resolveWebSearchConnection`：槽优先，legacy 兜底

[#L277-L287](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L277-L287)：

```ts
const connection = await resolveMediaSlot('webSearch', {
  workspaceId,
  legacyRequest: async () => requestedWebSearchConnection(requested, rules),
});
return webSearchConfigFromConnection(connection, requested);
```

`resolveMediaSlot` 的判定在 [model-config/media.ts#L227-L244](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/media.ts#L227-L244)：

```ts
const resolution = await lookupSlot(slot, workspaceId);
if (requestMayChoose(resolution) && requestProvidersAllowed()) {
  const requested = await legacyRequest?.();
  if (requested) return requested;
}
if (resolution.status === 'assigned') { ...return fromTarget(...) }
if (resolution.status === 'disabled') throw new SlotDisabledError(slot);
throw new SlotUnassignedError(slot, ...);
```

| 判定 | 位置 | 语义 |
|---|---|---|
| `requestMayChoose(resolution)` | [runtime.ts#L81-L87](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/runtime.ts#L81-L87) | 槽**未分配且未被锁定**，或"部署仍是 legacy 变量配的 + 槽命中的是 legacy 翻译出来的 default" |
| `requestProvidersAllowed()` | [#L69-L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/runtime.ts#L69-L71) | 部署层允许用户使用自己的 key（`allowUserKeys: false` 时请求里点什么都没用） |
| `SlotDisabledError` / `SlotUnassignedError` | media.ts #L242-L243 | 由 `available()` 捕获成"能力 false" |

### 7.4 run 传的三条 legacy 规则

`services.research` 传的是 `{ refuseDisabled: true, preferServerProvider: true, fallbackProviderId: 'tavily' }`，`requested` 是 `{}`。逐条对应 [LegacySearchRules #L142-L156](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L142-L156) 与实现 [#L172-L213](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L172-L213)：

| 规则 | 效果 |
|---|---|
| `fallbackProviderId: 'tavily'` | 请求没点名 provider 时，先取运维配的 server provider，再取 `DEFAULT_WEB_SEARCH_PROVIDER_ID`（[#L176-L180](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L176-L180) + [constants.ts#L8-L9](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L8-L9)） |
| `preferServerProvider: true` | 请求点了一个"服务端没配"的 provider 时，改回运维自己的（[#L182-L190](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L182-L190)）——复刻 1.1.x `/api/web-search` 的行为 |
| `refuseDisabled: true` | provider 被强制关闭时**抛 `RequestedProviderRefusedError`（403 `PROVIDER_DISABLED`）**，而不是静默跳过（[#L191-L198](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L191-L198)） |
| searxng 特判 | `managed || providerId === 'searxng'` 时**丢弃请求里的 baseUrl**（[#L200-L202](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L200-L202)），"SearXNG base URLs are operator-managed only; never trust client input" |
| 托管即丢 key | `apiKey: resolveWebSearchApiKey(providerId, managed ? undefined : input.webSearchApiKey)`（[#L206](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L206)）：托管 provider 永远用服务端 key |

对比另一条路：[#L294-L306](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts#L294-L306) 的 `resolveClassroomWebSearchConfig`
会把 `SlotDisabled` / `SlotUnassigned` / 非 `INVALID_REQUEST` 的 `WebSearchConfigError` **全部吞成 `undefined`**，
只留非法 base URL 上抛。现场 2/3 用它，run 不用——这就是"聊天里搜索会静默消失，生成里搜索会失败暂停"的根因。

### 7.5 谁决定"能不能搜"

```
resolveServerGenerationCapabilities(workspaceId)   ← run 的唯一门（§3 ①）
        └─ available('webSearch', workspaceId)
              └─ resolveMediaSlot('webSearch', { workspaceId })   ← 不带 legacyRequest
                    ├─ lookupSlot: openmaic.yml / env / workspace 设置
                    └─ 抛 SlotDisabled / SlotUnassigned → catch → false
```

`isWebSearchProviderConfigured`（[constants.ts#L138-L154](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L138-L154)）是**面向设置界面**的另一套"看起来配置了吗"判断（`serverDisabled` 优先、searxng 只认运维），run 不读它——
这解释了"设置里看着灰灰的但仍能搜"或反之的困惑（§12 坑 3）。

---

## 8. 产出如何被下游消费（两条独立通道）

`ResearchOutput` 有两位信息，走**两条完全不同的路**进 prompt：

```
检查点 research = { webSearch, context, sources }
        │
        ├── 通道 A：布尔位 webSearch ──► runRequirements(input, research?.webSearch === true)
        │                                   └─► UserRequirements.webSearch（只在为 true 时写入字段）
        │                                       ├─ 第 3 步 outline（engine #L819）
        │                                       └─ 第 5 步 scene:0:content（engine #L523）
        │
        └── 通道 B：文本 context ────► OutlineInput.researchContext
                                        └─ 仅第 3 步 outline（engine #L822）
                                            （prompt 变量，缺省字符串 'None'）
```

| 消费点 | 代码 | 传的是什么 |
|---|---|---|
| 大纲 | [engine.ts#L815-L826](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L815-L826) | `requirements: runRequirements(input, research?.webSearch === true)` + `...(research?.context ? { researchContext: research.context } : {})` |
| 首个场景内容 | [engine.ts#L506-L530](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L506-L530)（`#L508` 读检查点，`#L523` 用法） | 只有通道 A |
| 第 2 个及以后场景 | [engine.ts#L519-L526](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L519-L526) 注释 | **不读** research：其余场景只带 task-engine 标记 |
| Agent 画像 / 动作 / 旁白 | — | 完全不消费 research |

`runRequirements` 本体在 [engine.ts#L222-L232](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L222-L232)：

```ts
export function runRequirements(input: GenerationRunInput, webSearch: boolean): UserRequirements {
  return {
    requirement: input.requirement,
    ...(input.learnerProfile?.nickname ? { userNickname: input.learnerProfile.nickname } : {}),
    ...(input.learnerProfile?.bio ? { userBio: input.learnerProfile.bio } : {}),
    ...(webSearch ? { webSearch: true } : {}),
    ...(input.interactive ? { interactiveMode: true } : {}),
    ...(input.taskEngine ? { taskEngineMode: true } : {}),
  };
}
```

含义：`webSearch` 是**存在性开关**（`{ webSearch: true }` 或不带该字段），而不是 `false`。
大纲/内容 prompt 因此能区分"这次检索过"和"没检索"，从而决定要不要写"结合检索到的资料"这类指令。
通道 B 的字段缺省时大纲侧用 `'None'` 占位——所以"搜到了但没有正文可拼"（`context === ''`）与"根本没搜"
在 prompt 里最终是同一个形态。

---

## 9. 事件、检查点与快照：三者不一致

| 载体 | 有没有 research 数据 | 出处 |
|---|---|---|
| 事件 `research_sources`（`{ sources }`） | 有（仅当搜过） | 发出点 [engine.ts#L656](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L656)；事件清单声明 [run/types.ts#L283-L284](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/types.ts#L283-L284) |
| 检查点 `generation_run_steps('research')` | 有（含 `context` 全文） | engine `done(...)` |
| **run 快照 JSON** | **没有** | `GET /api/generation-runs/[id]` 只回灌 `materialKinds` / `materialTruncated`，见 [app/api/generation-runs/[id]/route.ts](file:///D:/GITHUB_chat/OpenMAIC/app/api/generation-runs/[id]/route.ts) |
| 客户端视图 `view.researchSources` | 只从事件累加 | [reducer.ts#L121-L135](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L121-L135) |

由此产生一个**必须知道的断链**：

```ts
// reducer.ts #L35：由快照构造视图时，来源列表永远是空的
researchSources: [],

// follower.ts #L88-L94：重连拿到新快照时，靠内存旧值把它保回来
return {
  ...next,
  researchSources: current.researchSources,
  ...
};
```

| 场景 | 结果 |
|---|---|
| 正常连着的 SSE | 看到来源列表逐条出现 |
| 刷新页面 / 换设备打开同一个 run | **来源列表为空**（`context` 仍在检查点里，生成不受影响） |
| 纯 `getSnapshot` 驱动（无历史事件回放） | 同样为空 |

客户端还做了两处防御（[reducer.ts#L123-L131](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts#L123-L131)）：
`url` 必须是字符串才收；`title` 不是字符串时**用 `url` 顶替**。

### 9.1 UI 步骤位的两种"折叠"

[preview-steps.ts#L35-L48](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts#L35-L48)：

```ts
if (step === 'material-analysis' || step === 'research') return 'web-search';
...
if (view.state === 'preparing') return 'web-search';
```

| 事实 | 影响 |
|---|---|
| 第 1 步和第 2 步共用 UI 上的 `web-search` 位 | 预览页里看不到"材料分析"独立进度 |
| `preparing` 且步间空档也归到 `web-search` | run 一启动就先高亮这一位 |
| `previewStepIds({ webSearch: !!capabilities.webSearch })`（[page.tsx#L95-L102](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx#L95-L102)） | 没配搜索槽时这一位**整列不出现**，run 明明有 `research` 步但 UI 无感知 |

动画组件：[visualizers.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx) 的 `WebSearchVisualizer`（[#L50-L107](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L50-L107)）
——一个迷你搜索结果页：4 行骨架（`ROW_H = 38`），有来源时每 **1400 ms** 轮换高亮前 `min(sources.length, 4)` 条（teal 光晕，spring 动画）。
它只吃 `{ title, url }`，与本步的降形正好对齐。

---

## 10. 失败链与 Retry 语义

### 10.1 四层

| 层 | 本步的形态 |
|---|---|
| ① 业务实现 | [steps/research.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts)：改写失败=回落；provider 非 2xx=抛 `Error` |
| ② service 适配 | [services.ts#L419-L442](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts#L419-L442)：能力不满足=`null`（不是失败）；配置解析失败=原样上抛 |
| ③ 步骤包装 | [engine.ts#L634-L658](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts#L634-L658)：只有 `withDeadline`（300 s → `StepTimeoutError`，`statusCode = 504`），**无 route retry** |
| ④ 失败分类 | [failure-code.ts#L41-L65](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L41-L65)：`runFailureCode(error)` → `errorCode` |

### 10.2 常见失败 → 码

| 触发 | 异常 | `errorCode` | 可 Retry？ |
|---|---|---|---|
| 槽 assigned 但缺 key | `WebSearchConfigError('MISSING_API_KEY')`（config.ts #L230） | `MISSING_API_KEY`（failure-code #L46-L48） | 是（配好 key 再 Retry） |
| searxng 没配 base URL | `WebSearchConfigError('MISSING_REQUIRED_FIELD')`（#L248） | `MISSING_REQUIRED_FIELD` | 是 |
| providerId 不在注册表 | `WebSearchConfigError('INVALID_REQUEST')`（#L224） | `INVALID_REQUEST` | 需改配置 |
| provider 被运维强关 | `RequestedProviderRefusedError`（403，#L193-L197） | `PROVIDER_DISABLED`（[#L18](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L18) 内置码，经 `apiError` 的状态分类） | 需运维放开 |
| provider 返回 429 | 带 HTTP 状态的错误 | `RATE_LIMITED`（[#L61-L63](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts#L61-L63)） | 是 |
| provider 返回 5xx | 带 HTTP 状态的错误 | `UPSTREAM_ERROR` | 是 |
| Tavily 抛的普通 `Error('Tavily API error (500)')` | `upstreamHttpStatus` 取不到状态 | `INTERNAL_ERROR` | 是（但看不到上游码） |
| 改写模型超时/返回坏 JSON | **不失败**（§4.3 第 5 级） | — | — |
| 搜索超过 300 s | `StepTimeoutError`（deadline.ts #L28-L35） | 按 504 分类，`UPSTREAM_ERROR` 形态 | 是 |
| 能力没配 | `null` | 不产生失败 | 不需要 |

### 10.3 Retry 回到哪个状态

```ts
// plan.ts #L107-L111
export function stateForRetry(stepId: string): ExecutableRunState {
  const step = parseStepId(stepId);
  if (!step) throw new Error(`Unknown run step ${JSON.stringify(stepId)}`);
  return phaseOfStep(step);      // research → 'preparing'
}
```

即：**卡在 `research` 的 run，Retry 后回到 `preparing`**，重跑本步（会**重新**读一次能力与配置，
所以运维改完配置直接 Retry 就能生效，不必重启进程）。`material-analysis` 失败时 UI 也显示在 `web-search` 位（§9.1），
所以"停在 web-search 位的失败"要看 `view.step` 才知道是哪一步。

### 10.4 本步特有的三点"看起来不对但确实是设计"

| 现象 | 原因 |
|---|---|
| provider 抖动没有自动重试，直接暂停 run | 本步无 `withRouteRetry`；检索是增强项，不值得为它做进程内退避（第 5/6/7 步才有） |
| 从没配搜索的 run 里，检查点仍有一条 `research` 记录 | 无条件步 + "空跑也检查点化"（§2） |
| 空跑时没有任何 `research_*` 事件 | `result ? { events: [...] } : {}`（engine #L656） |

---

## 11. 常量与配置速查

| 常量 / 配置 | 值 | 位置 |
|---|---|---|
| `STEP_DEADLINES_MS.research` | 300 000 ms | [deadline.ts#L14-L15](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts#L14-L15) |
| `TAVILY_SOFT_MAX_QUERY_LENGTH` | 350 | [search-query-builder.ts#L7](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L7) |
| `SEARCH_QUERY_REWRITE_EXCERPT_LENGTH` | 7000 | [#L8](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L8) |
| 改写触发阈值 | 需求 > 400 字符 **或** 有材料节选 | [#L34-L39](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts#L34-L39) |
| 改写 `maxOutputTokens` | 256 | [research.ts#L51](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L51) |
| `TAVILY_MAX_QUERY_LENGTH` | 400（硬截） | [tavily.ts#L14](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/tavily.ts#L14) |
| Tavily `max_results` 默认 | 5 | [#L31](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/tavily.ts#L31) |
| Tavily 请求档 | `search_depth: 'basic'`, `include_answer: 'basic'` | [#L42-L47](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/tavily.ts#L42-L47) |
| context 每条正文截断 | 200 字符 | [format.ts#L21](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/format.ts#L21) |
| `DEFAULT_WEB_SEARCH_PROVIDER_ID` | `'tavily'` | [constants.ts#L8-L9](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L8-L9) |
| `CLAUDE_WEB_SEARCH_DEFAULT_MODEL` | `claude-sonnet-5` | [#L136](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L136) |
| Claude 可选模型表 | 5 个（opus-5 / sonnet-5 / opus-4-6 / sonnet-4-6 / haiku-4-5） | [#L11-L18](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L11-L18) |
| 需要 key 的 provider | 除 `brave`、`searxng` 外全部 | [#L23-L133](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L23-L133) |
| 环境开关 | `WEB_SEARCH_KIMI_ENABLED`、`WEB_SEARCH_DEEPSEEK_ENABLED` | [#L85](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L85)、[#L126](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts#L126) |
| 改写模型槽 | `course.research`（父 `llm`） | [model-slots.ts#L59](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L59)、[#L111](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L111) |
| 搜索能力槽 | `webSearch` | [media.ts#L227-L244](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/media.ts#L227-L244) |
| 启动预检槽表 | 含 `webSearch` | [start.ts#L34-L40](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/start.ts#L34-L40) |
| UI 轮播间隔 / 行高 | 1400 ms / 38 px | [visualizers.tsx#L58](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L58)、[#L70](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx#L70) |

---

## 12. 易踩坑清单

1. **"UI 上没有这一步" ≠ "run 没执行它"**。步骤列表按 `capabilities.webSearch` 裁剪（page.tsx #L98），
   而计划是无条件的（§2）。诊断要看 run 的步骤表或 `view.step`，不要看 UI。
2. **刷新后来源列表消失**是已知断链：快照里没有 research 字段，`viewFromSnapshot` 置空数组，
   只有同一次会话的内存视图能保住（§9）。别把它当成"搜索结果丢了"——`context` 仍在检查点里。
3. **设置页的灰/亮状态和 run 能不能搜是两套判断**。前者 `isWebSearchProviderConfigured`
   （constants.ts #L138-L154，含 searxng 只认运维、`serverDisabled` 优先），
   后者 `resolveServerGenerationCapabilities` → `resolveMediaSlot`。出现"看着能用但空跑"先查后者。
4. **`webSearch: true` 不代表搜到了东西**。它等于"能力解析成功且请求发出了"。
   搜到 0 条时仍是 `true`，且 `context` 是 `''`——下游 prompt 与"没搜"同形（§6/§8）。
5. **能力探针过得了，配置解析仍可能抛**。探针不带 `legacyRequest`，
   run 的解析带（§3 的窄缝、§7.3）。典型：legacy 部署下 server provider 被强制关闭 → 探针 true、
   `refuseDisabled: true` → `PROVIDER_DISABLED` 失败暂停。这**是**故意的：宁停不猜。
6. **改写失败绝不失败整步**，所以"搜索词看起来很蠢"通常是 `course.research` 槽不可用（日志里有
   `falling back to raw requirement`）而不是搜索坏了。
7. **`context` 没有全局长度上限**。provider 若返回超长 `answer`，会整段进大纲 prompt（§6）；
   换 provider 时这是主要的 prompt 体积变量。
8. **搜索引擎请求不吃 abort 信号**（§5.2）：run 被取消/租约丢失时本步会 reject，但上游请求仍在飞。
   做取消相关的性能分析时不要指望 provider 侧立即断开。
9. **只有前 7000 字符材料能影响搜索词**（§4.1）。上传了大文档却发现搜索词跟文档无关，先看材料前 7000 字符是什么。
10. **第 2 个及以后的场景内容步不读 research**（§8）。所以"检索结果没影响到后面几页"不是 bug。
11. **`research_sources` 事件在空跑时不发**（engine #L656），别把它当"步骤开始"的信号用；
    步骤开始信号是 `step_started`。

---

## 13. 代码地图

| 文件 | 看什么 |
|---|---|
| [steps/research.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts) | 本步业务实现（81 行）：节选截断、`aiCall`、`searchWeb`、`context` |
| [search-query-builder.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/search-query-builder.ts) | 归一化、触发条件、五级回落、350 软截 |
| [web-search/index.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/index.ts) | 12 家 provider 派发 + `never` 兜底 |
| [web-search/tavily.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/tavily.ts) | 默认 provider 的 REST 细节与 400 硬截 |
| [web-search/format.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/format.ts) | `context` 的构造（answer + Sources + 200 截断） |
| [web-search/constants.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/web-search/constants.ts) | provider 注册表、默认 id、回退顺序、`isWebSearchProviderConfigured` |
| [types/web-search.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/types/web-search.ts) | `WebSearchResult` / `WebSearchSource` 形状 |
| [server/web-search-config.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/web-search-config.ts) | `WebSearchConfig`、`WebSearchConfigError`、legacy 规则、两个 resolve 入口 |
| [server/generation-capabilities.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation-capabilities.ts) | 能力探针（run 的唯一门） |
| [server/model-config/media.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/media.ts) | `resolveMediaSlot`：槽优先、legacy 何时被问 |
| [server/model-config/runtime.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/model-config/runtime.ts) | `requestMayChoose` / `requestProvidersAllowed` |
| [run/services.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/services.ts) | `research` 三级门控、`stageModel` |
| [run/engine.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/engine.ts) | `case 'research'`、`ResearchOutput`、`runRequirements`、两处消费点 |
| [run/plan.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts) | 无条件步、`phaseOfStep`、`stateForRetry` |
| [run/deadline.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/deadline.ts) | `research: 300_000`、`StepTimeoutError` |
| [run/failure-code.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/failure-code.ts) | `WebSearchConfigError` → 码；429/5xx 分类 |
| [lib/prompts/templates/web-search-query-rewrite/](file:///D:/GITHUB_chat/OpenMAIC/lib/prompts/templates/web-search-query-rewrite/system.md) | 改写 prompt（system + user） |
| [server/agent-runtime/web-search.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/server/agent-runtime/web-search.ts) | 现场 2：能力注册式 `web_search` 工具（对照组） |
| [lib/chat/pi/tools/web-search.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/chat/pi/tools/web-search.ts) | 现场 3：课堂聊天里直连 `searchWeb` |
| [generation-run-client/preview-steps.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/preview-steps.ts) | `web-search` 位、两步折叠 |
| [generation-run-client/reducer.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/reducer.ts) / [follower.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/generation-run-client/follower.ts) | `researchSources` 的填充与保住 |
| [app/generation-preview/page.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/page.tsx) / [components/visualizers.tsx](file:///D:/GITHUB_chat/OpenMAIC/app/generation-preview/components/visualizers.tsx) | 步骤裁剪与 `WebSearchVisualizer` |

---

## 14. 动手验证

| 想验证什么 | 怎么做 |
|---|---|
| 空跑语义 | 不配 `webSearch` 槽起一个 run：查 `generation_run_steps` 里 `research` 行存在且值为 `{"webSearch":false}`；SSE 流里**没有** `research_sources`；UI **没有** `web-search` 这一步 |
| 无条件入计划 | 读 [plan.ts#L46-L54](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/run/plan.ts#L46-L54)：`preparationSteps` 里 `research` 不受任何 `if` 保护，只要有 run 就存在这一行步骤 |
| 空跑不报错 | 配好 `webSearch` 槽后启动，再把槽关掉并 `retry`：`step_retry` 不会为本步出现（它不在 `withRouteRetry` 里），但 `stateForRetry('research')` 会把 phase 拉回 `preparing` |
| 改写回落 | 在 [tests/server/generation-steps/research.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/server/generation-steps/research.test.ts) 里把 `aiCall` 换成抛异常 / 返回 `{"query":""}`，断言最终搜索词 == 原始 requirement，且日志是 warn 而非 error |
| 触发条件 | 同文件喂两种输入：`requirement` 长度为 401（应改写）、长度为 100 且无 `pdfText`（应跳过改写直接返回原文） |
| 350 / 400 两级截断 | 构造一条 600 字符的改写输出，看 `searchWeb` 收到的 query 长度 == 350；把 `TAVILY_SOFT_MAX_QUERY_LENGTH` 临时改到 500，看 Tavily 侧 400 硬截生效 |
| provider 派发 | [tests/web-search/index.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/web-search/index.test.ts)：逐个 providerId 断言走到对应实现、`brave`/`searxng` 无 key 也能通过 |
| 配置解析与错误码 | [tests/server/web-search-config.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/server/web-search-config.test.ts)：验 `refuseDisabled` 抛 `RequestedProviderRefusedError`、searxng 丢 `apiKey`、managed 丢 `baseUrl` |
| 能力探针吞错 | [tests/agent-runtime/web-search.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/agent-runtime/web-search.test.ts) 与 [tests/agent-runtime/runner-web-search-registration.test.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/agent-runtime/runner-web-search-registration.test.ts)：槽未配置时工具**不注册**而不是注册后报错 |
| 刷新丢来源 | 起一个带联网的 run，等 `research_sources` 出现在 SSE 里，然后刷新页面：`WebSearchVisualizer` 只剩骨架行，因为快照没有该字段 |
| 本步不吃 abort | 在 [steps/research.ts#L71](file:///D:/GITHUB_chat/OpenMAIC/lib/server/generation/steps/research.ts#L71) 打断点，取消 run 后观察 provider 请求仍然发出 |

---

## 15. 一句话总结

**第 2 步 = 一条改写后的搜索词 + 一次 provider 调用 + 一段压缩过的 `context`。**
它名义上是 7 步里的第 2 步，实际上是**无条件计划里的条件能力**：
`webSearch` 槽没配就整步空跑，只把 `{"webSearch":false}` 写进检查点；
下游拿它的方式也分两条——大纲和首场景读**布尔**决定是否"允许联网"，
只有大纲额外读**文本** `context`。
所以排查"为什么没联网"时，顺序永远是：能力探针 → 连接解析 → 改写模型 → provider 响应，
而不是去看模型有没有把搜索结果用起来。