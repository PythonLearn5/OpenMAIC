# OpenMAIC 项目配置说明文档

> 职责说明：本文档系统说明 OpenMAIC 的配置体系——两份核心配置文件（`.env.local` 与 `openmaic.yml`）的定位与字段、槽位（slot）模型、构建期与运行期开关、持久化与所有者身份、启动校验与失败行为、Docker Compose 默认值，以及配置优先级。

> 相关文档：
> - 本地运行：[LOCAL-RUN.md](file:///D:/GITHUB_chat/OpenMAIC/docs/LOCAL-RUN.md)
> - 项目结构：[PROJECT-STRUCTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/PROJECT-STRUCTURE.md)
> - 总体架构：[ARCHITECTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/ARCHITECTURE.md)

---

## 1. 配置体系总览

OpenMAIC 的配置分为**三层**，职责不重叠：

| 层 | 载体 | 放什么 | 读取时机 |
| --- | --- | --- | --- |
| **密钥与实例选项** | `.env.local` | API Key、`DATABASE_URL`、`ACCESS_CODE`、能力总开关、运行器调优 | 进程启动时（`NEXT_PUBLIC_*` 例外，见 §5） |
| **模型配置** | `openmaic.yml` | 服务器可调用的 providers、每个 slot 用哪个模型、锁定与用户权限 | 启动时读取一次，**改动需重启** |
| **运行时数据** | PostgreSQL | 课程、聊天、学习者进度、媒体；网页设置里保存的 provider 密钥（加密存储） | 请求时 |

### 1.1 配置相关文件清单

| 文件 | 角色 |
| --- | --- |
| [.env.example](file:///D:/GITHUB_chat/OpenMAIC/.env.example) | 环境变量模板，复制为 `.env.local` |
| [openmaic.example.yml](file:///D:/GITHUB_chat/OpenMAIC/openmaic.example.yml) | 模型配置模板，复制为 `openmaic.yml` |
| [docker-compose.defaults.env](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.defaults.env) | Compose 部署默认值（内置 `DATABASE_URL`、单用户模式），先于 `.env.local` 加载 |
| [docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml) | 容器拓扑、profile、端口发布、构建参数 |
| [docker-compose.db.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.db.yml) | `pnpm db:up` / `db:down` 使用的独立开发库 |
| [next.config.ts](file:///D:/GITHUB_chat/OpenMAIC/next.config.ts) | 构建期配置：`output`、构建参数、CSP frame-ancestors 头 |
| [instrumentation.ts](file:///D:/GITHUB_chat/OpenMAIC/instrumentation.ts) | 启动校验与后台任务注册（校验失败即退出进程） |

> ⚠️ 变量名与枚举值（`SSRF`、`JSON`、`mode`、`effort`、`openai-completions` 等）保持英文原样，不要翻译或改写。

---

## 2. 配置一：`.env.local`（密钥与实例选项）

```bash
cp .env.example .env.local
```

`DATABASE_URL` 是唯一**必需**项（Compose 部署由 [docker-compose.defaults.env](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.defaults.env#L12) 预置）。其余按区块说明如下。

### 2.1 实例与安全

| 变量 | 作用 | 要点 |
| --- | --- | --- |
| `OPENMAIC_CONFIG` | 模型配置文件路径 | 默认 `openmaic.yml`（仓库根 / 镜像内 `/app`）；设置后文件必须存在 |
| `OPENMAIC_SECRET_KEY` | 加密「网页设置中保存的 provider 密钥」 | 未设置时首启动生成 `data/instance-secret.key`；多实例、`data/` 不持久化或只读文件系统必须显式设置，且需与数据库一起保管 |
| `ACCESS_CODE` | 站点级访问密码 | 运行期读取；未设置时中间件放行一切（fail-open）。≥ 16 位随机值，签发 7 天有效期的签名 Cookie |
| `COOKIE_SECURE` | 覆盖 Cookie 的 `Secure` 标记 | 生产构建默认开启；纯 HTTP（非 localhost 特例，如 Safari 访问 `http://localhost`）需设为精确值 `0` |
| `TRUST_PROXY_HEADERS` | 是否信任 `x-forwarded-for` / `x-real-ip` | 仅在可信反向代理后开启；开启后访问码校验才有按客户端限流（10 次 / 60 秒） |
| `ALLOW_LOCAL_NETWORKS` | 放行私网 / 回环 / 链路本地 / CGNAT 目标 | 自托管本地模型（Ollama、Lemonade 等）必需；**公网部署禁止开启**。实例元数据地址恒被拦截 |
| `ALLOWED_FRAME_ANCESTORS` | 额外 CSP `frame-ancestors` 来源（空格分隔） | **构建期**生效，改后需重建；设置后会移除 `X-Frame-Options` |
| `INTERACTIVE_ASSET_BASE_URL` | 交互式场景静态资源基址 | 默认 `/vendor/interactive`；**构建期** 生效 |

### 2.2 LLM Provider 凭据

统一格式：`{PROVIDER}_API_KEY` / `{PROVIDER}_BASE_URL`（可选）/ `{PROVIDER}_MODELS`（逗号分隔）。

- **`*_MODELS` 是授权白名单，而非选择器列表**：服务器托管的凭据只授权显式列出的模型，以及该能力的部署槽位 / fallback 引用；目录中的模型不会隐式获得授权。为每个持有凭据的 provider 设置 `*_MODELS` 后重启。
- **没有 Key 的 provider 是不受限制的**：不要把「自带上游凭据的网关」配成无 Key 的 provider；应给它一个 Key 与 `*_MODELS`。
- 用户自带凭据与无 Key 的本地 provider 不受此限制。

模板中已列出的前缀包括：`OPENAI`、`AZURE_OPENAI`、`ATLASCLOUD`、`ANTHROPIC`、`GOOGLE`、`DEEPSEEK`、`QWEN`、`KIMI`、`MINIMAX`、`GLM`、`SILICONFLOW`、`DOUBAO`、`OPENROUTER`、`GROK`、`TENCENT`（别名 `TENCENT_HUNYUAN_*`）、`XIAOMI`（别名 `MIMO_*`）、`TOKENDANCE`，以及本地的 `OLLAMA_BASE_URL` / `LEMONADE_BASE_URL`，和 Bedrock 系列（`BEDROCK_REGION` + AWS 凭据链）。

**遗留配置说明**：在没有 `openmaic.yml` 时，这些 `*_API_KEY` 变量仍会直接配置 provider（并配合 `DEFAULT_MODEL` / `MODEL_FALLBACK` 生效），但在存在 `openmaic.yml` 后它们不再自动生效，只能通过 `${VAR}` 被引用。`MODEL_ROUTES` 已不再被读取——若在没有 `openmaic.yml` 的情况下仍设置它，服务器**拒绝启动**。

### 2.3 TTS / ASR

| 能力 | 前缀 | 备注 |
| --- | --- | --- |
| TTS | `TTS_OPENAI` / `TTS_AZURE` / `TTS_GLM` / `TTS_QWEN` / `TTS_DOUBAO` / `TTS_MINIMAX` / `TTS_ELEVENLABS` / `TTS_GOOGLE` | `TTS_QWEN_VOICE_CLONE_MODEL` 可选，覆盖音色克隆目标模型 |
| TTS（本地） | `TTS_VOXCPM_*` / `TTS_LEMONADE_BASE_URL` | 无需 Key |
| ASR | `ASR_OPENAI` / `ASR_QWEN` / `ASR_AZURE` | — |
| ASR（本地） | `ASR_FUNASR_BASE_URL` / `ASR_LEMONADE_BASE_URL` | 仅接受 WAV 输入，无需 Key |

**强制关闭开关**：任何内置 provider 都可用 `<能力>_<供应商>_ENABLED=false` 关掉，例如 `TTS_OPENAI_ENABLED=false`、`ASR_BROWSER_NATIVE_ENABLED=false`。

> 本地音视频材料抽取依赖**服务端第一个启用的 ASR provider** 与系统 `ffmpeg` / `ffprobe`；二者缺一即跳过本地抽取，转为已配置的 AliDocMind 云端抽取，两者都没有时材料以可操作提示失败。

### 2.4 PDF / 图像 / 视频 / 联网搜索

| 能力 | 前缀 | 备注 |
| --- | --- | --- |
| PDF 解析 | `PDF_UNPDF_*`、`PDF_MINERU_*`（自托管）、`PDF_MINERU_CLOUD_*`、`ALIDOCMIND_*`（AccessKey 对） | 自托管 MinerU 不会隐式回退到云端，除非显式设置 `ALLOW_MINERU_CLOUD_FALLBACK=true`（文档会离开你的基础设施） |
| 图像生成 | `IMAGE_OPENAI` / `IMAGE_SEEDREAM` / `IMAGE_QWEN_IMAGE` / `IMAGE_NANO_BANANA` / `IMAGE_MINIMAX` / `IMAGE_GROK` / `IMAGE_GLM` / `IMAGE_OPENROUTER` / `IMAGE_LEMONADE_BASE_URL` | 同样支持 `IMAGE_*_ENABLED=false` |
| 视频生成 | `VIDEO_SEEDANCE` / `VIDEO_KLING` / `VIDEO_SORA` / `VIDEO_MINIMAX` / `VIDEO_GROK` / `VIDEO_HAPPYHORSE` / `VIDEO_OPENROUTER` | 同样支持 `VIDEO_*_ENABLED=false` |
| 联网搜索 | `TAVILY` / `EXA` / `FIRECRAWL` / `BOCHA` / `BRAVE` / `BAIDU` / `SEARXNG_BASE_URL` / `WEB_SEARCH_MINIMAX_*` / `WEB_SEARCH_DOUBAO_*` / `WEB_SEARCH_CLAUDE_*` | 同样支持 `<PREFIX>_ENABLED=false` |
| 搜索旁路开关 | `WEB_SEARCH_DEEPSEEK_ENABLED` / `WEB_SEARCH_KIMI_ENABLED` | 未设置 / `false` 时，聊天凭据绝不自动激活搜索；`false` 可强制关闭（即便存在显式 slot） |

### 2.5 代理

```env
HTTP_PROXY=
HTTPS_PROXY=
NO_PROXY=localhost,127.0.0.1,.internal.example.com
```

`NO_PROXY` 支持域名后缀与 `*`。

### 2.6 运行器与生成调优（默认值见模板）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `OPENMAIC_AGENT_RUNTIME_SCAN_INTERVAL_MS` | `1000` | 扫描间隔，生成 run 共用同一套租约时序 |
| `OPENMAIC_AGENT_RUNTIME_HEARTBEAT_MS` | `2000` | 心跳 |
| `OPENMAIC_AGENT_RUNTIME_LEASE_TTL_MS` | `10000` | 租约 TTL |
| `OPENMAIC_AGENT_RUNTIME_MAX_CONCURRENT` | `2` | |
| `OPENMAIC_AGENT_RUNTIME_MAX_ATTEMPTS` | `5` | |
| `OPENMAIC_AGENT_TOOL_TIMEOUT_MS` | `600000` | 单次工具调用全局上限（10 分钟） |
| `OPENMAIC_MAX_ACTIVE_RUNS_PER_OWNER` | `2` | 超限返回 `429 ACTIVE_RUN_LIMIT` |
| `OPENMAIC_MAX_WAITING_RUNS_PER_OWNER` | `10` | |
| `OPENMAIC_GENERATION_RUN_MAX_CONCURRENT` | `4` | |
| `OPENMAIC_MATERIAL_EXTRACTION_CONCURRENCY` | `2` | |
| `OPENMAIC_MATERIAL_EXTRACTION_PER_OWNER` | `2` | |
| `OPENMAIC_MATERIAL_EXTRACTION_MAX_RESULT_MB` | `100` | 超限 `EXTRACTION_RESULT_TOO_LARGE` |
| `OPENMAIC_UNUSED_MATERIAL_TTL_HOURS` | `24` | |
| `OPENMAIC_GENERATION_RUN_STREAMS_PER_OWNER` | `16` | |
| `OPENMAIC_GENERATION_RUN_RETENTION_HOURS` | `24` | |
| `PARALLEL_SCENE_CONCURRENCY` | 未设置（串行） | > 1 时并发拉取场景内容（上限 10）；动作与 TTS 仍串行 |
| `LOG_LEVEL` / `LOG_FORMAT` / `LLM_THINKING_DISABLED` | — | 日志与思考开关 |

> 对话压缩相关变量（`OPENMAIC_AGENT_COMPACTION_*`）当前为**保留占位**，运行时尚未实现压缩，设置了也不生效。会话提示词与后续消息在服务端固定上限 100,000 字符，不可配置。

---

## 3. 配置二：`openmaic.yml`（模型配置）

```bash
cp openmaic.example.yml openmaic.yml
```

该文件在启动时读取一次，**改动后需重启服务器**。它包含四类顶层字段：`providers`、`slots`、`lock`、`allowUserKeys`。

### 3.1 providers（服务器可调用的账号）

```yaml
providers:
  openai:
    preset: openai
    apiKey: ${OPENAI_API_KEY}
    models: [gpt-5.5]
```

- provider id（键名）为**小写字母、数字与短横线**；
- `preset` 决定适配器与默认端点（`openai` / `anthropic` / `minimax` / `bedrock` / `glm` / `gemini` / `openai-image` / `openai-tts` / `voxcpm-tts` / `funasr-asr` / `mineru-cloud` / `alidocmind` / `tavily` / `openai-compatible` / `lemonade*` …）；
- `apiKey` / `baseUrl` / `models` / `options` / `voices` / `credentials` 为常用字段；密钥一律写 `${VAR}` 引用环境变量，**不落文件**；
- `models` 是**授权白名单**，不是选择器列表（与 §2.2 规则一致）；多数媒体类 preset 只覆盖单一能力，因此图像需要单独的 `openai-image` provider（可复用同一 Key）。

### 3.2 slots（每类 AI 用途的默认模型）

槽位（slot）是「一种需要 AI 的产品用途」，构成一棵**每能力一棵树**的森林；未赋值的槽位在解析时**沿父链继承**。

| 槽位 | 能力 | 父槽位 | 说明 |
| --- | --- | --- | --- |
| `llm` | chat | —（根） | 默认对话模型；所有未单独赋值的 chat 槽位继承它 |
| `course.research` | chat | `llm` | 搜索查询改写 |
| `course.outline` | chat | `llm` | 大纲生成 |
| `course.agents` | chat | `llm` | 智能体画像生成 |
| `course.content` | chat | `llm` | 场景内容生成（总） |
| `course.content.slide` | chat | `course.content` | 幻灯片 |
| `course.content.quiz` | chat | `course.content` | 测验 |
| `course.content.interactive` | chat | `course.content` | 交互式 |
| `course.content.pbl` | chat | `course.content` | 项目式学习 |
| `course.actions` | chat | `llm` | 动作生成 |
| `classroom` | chat | `llm` | 课堂讨论 / 评分 / PBL 运行时；需支持工具调用 |
| `agent` | chat | `llm` | Pro 工作台 Agent；`requires: toolCalling`，且**禁止 `thinking.effort`** |
| `agent.title` | chat | `agent` | 会话标题；仅可在 `openmaic.yml` 配置（`configOnly`） |
| `tts` | tts | —（根） | 语音合成 |
| `asr` | asr | —（根） | 语音识别 |
| `image` | image | —（根） | 图像生成 |
| `video` | video | —（根） | 视频生成 |
| `webFetch` | webFetch | —（根） | 网页正文读取 |
| `webSearch` | webSearch | —（根） | 联网搜索 |
| `document` | document | —（根） | 文档解析 |

槽位定义见 [model-slots.ts](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L57-L84)。

**引用格式**：`<provider id>:<model id>`；只写 provider id 表示使用该 provider 的默认模型（搜索 / 文档类服务通常无可选项）。

```yaml
slots:
  llm: openai:gpt-5.5

  # 对象形式：附加调用参数与一次性 fallback
  course.outline:
    model: anthropic:claude-sonnet-4-6
    fallback: openai:gpt-5.4-mini

  # null 表示该能力默认关闭（用户仍可自行开启，除非被 lock 锁定）
  video: null
```

- `fallback` 仅在语言模型类槽位可用；主模型在**可重试失败**（SDK 判定为瞬时错误、超时、网络错误、配额 429、容量 503）或返回空输出时，`callLLM` 会**重试一次** fallback。内容安全拒答与其他 4xx **不**回退。
- `agent` 槽可设置 `api`（`openai-completions` 默认 / `openai-responses`）与 `contextWindow`（钉住窗口压缩假设的上限）。
- 阶段键（stage key，如 `scene-content:slide`）是内部调用点标签，**不会出现在配置或设置界面**；每个阶段固定映射到一个槽位，映射表见 [STAGE_SLOTS](file:///D:/GITHUB_chat/OpenMAIC/lib/config/model-slots.ts#L95-L114)。

### 3.3 lock 与 allowUserKeys

```yaml
lock: [video]          # 锁定 video 及其整棵子树；lock: all 锁定全部
allowUserKeys: false   # 用户不能添加 provider / Key，只能在已声明 provider 中选择
```

- `lock` 中的槽位必须在 `slots` 下声明过（`null` 也算）；
- `allowUserKeys: false` 时用户仅能在 `lock` 未覆盖的槽位中，于已声明的 provider 之间选择；
- 模板中的 `slots` 即服务器的**默认值**：用户可在应用内的模型设置中修改，或在 `allowUserKeys: true` 时接入自己的服务。

### 3.4 校验行为

`openmaic.yml` 在启动时被完整校验：**未设置的 `${VAR}`、未知 preset 或槽位、引用了未声明的 provider**，都会让服务器以退出码非 0 **拒绝启动**，并逐项列出问题位置（YAML 语法错误则给出行号）。

---

## 4. 模型授权与能力开关

| 机制 | 位置 | 效果 |
| --- | --- | --- |
| `models` 白名单 | `openmaic.yml` provider | 服务器托管凭据只授权显式列出的模型 + 槽位 / fallback 引用 |
| `*_MODELS` | `.env.local` | 遗留配置下的等价白名单；有 `openmaic.yml` 时仅作为 `${VAR}` 被引用 |
| `<CAP>_<PREFIX>_ENABLED=false` | `.env.local` | 强制关闭任何由服务端提供的能力（TTS/ASR/图像/视频/搜索） |
| `allowUserKeys: false` | `openmaic.yml` | 全面禁止用户自带凭据 |
| 客户端凭据 | 自定义 `x-*` 请求头 | 后端配合 SSRF Guard 校验；服务器托管凭据**永不**下发浏览器 |

---

## 5. 构建期开关 vs 运行期开关

| 类型 | 代表变量 | 生效方式 |
| --- | --- | --- |
| **构建期** | `NEXT_PUBLIC_*`、`ALLOWED_FRAME_ANCESTORS`、`INTERACTIVE_ASSET_BASE_URL`、`ALPINE_MIRROR` / `NPM_REGISTRY` | 编译进浏览器产物或镜像；**改动必须重新构建** |
| **运行期** | `DATABASE_URL`、`ACCESS_CODE`、`TRUST_PROXY_HEADERS`、能力 `*_ENABLED`、运行器调优、`ALLOW_LOCAL_NETWORKS`、`RENDER_SERVICE_URL` | 读取于进程启动或请求时；**改动后重启**（多数启动时读取） |

常用构建期功能开关（详见 [.env.example](file:///D:/GITHUB_chat/OpenMAIC/.env.example#L404-L448)）：

| 变量 | 作用 |
| --- | --- |
| `NEXT_PUBLIC_PRO_WORKBENCH_ENABLED` | Pro 工作台入口（隐含 MAIC Editor 门控） |
| `NEXT_PUBLIC_MAIC_EDITOR_ENABLED` | 课堂编辑器（Pro 模式）总门控 |
| `NEXT_PUBLIC_MAIC_EDITOR_RENDERER_ENABLED` | Pro 模式下选用 `@openmaic/editor` |
| `NEXT_PUBLIC_MAIC_PLAYBACK_RENDERER_ENABLED` | 播放画布选用 `@openmaic/renderer` |
| `NEXT_PUBLIC_COURSEWARE_REFERENCE_ENABLED` | Pi 播放中的课件引用入口 |
| `NEXT_PUBLIC_ENABLE_VIDEO_EXPORT` / `NEXT_PUBLIC_ENABLE_PPTX_IMPORT` | 视频导出 / PPTX 导入入口 |
| `NEXT_PUBLIC_VIDEO_EXPORT_CTA_DESTINATION` | 导出封面卡上的信息性跳转地址；`off` 表示省略 |
| `NEXT_PUBLIC_SHOW_VOCATIONAL_TEST_UI` | 职业任务引擎测试 UI（配合 `OPENMAIC_ENABLE_VOCATIONAL`） |

> ⚠️ 布尔值只接受 `true` 或 `1`。`NEXT_PUBLIC_PI_CHAT_ENABLED` 已废弃：生产构建若其取值不为 `''` / `true` / `1`，构建**直接失败**并给出迁移提示（见 [next.config.ts](file:///D:/GITHUB_chat/OpenMAIC/next.config.ts#L96-L119)）。
> Docker Compose 的 `env_file` 提供的是**运行期**值，不等于构建参数；构建参数需在 [docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml#L5-L22) 的 `build.args` 中显式传入。

---

## 6. 持久化与所有者身份

### 6.1 数据库与资产

```env
DATABASE_URL=postgres://...     # 必需
```

- 课程、文件夹、聊天历史、学习者会话与生成媒体存于 PostgreSQL，通过内置 `/api/persistence` 提供；
- **无服务端持久化回退**：缺少 `DATABASE_URL` 服务器拒绝启动；
- 浏览器仅保留设备级数据（设置、播放位置、缓存），「设置 → 清除本地缓存」不会触碰服务端。

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `ASSET_QUOTA_BYTES` | 10 GiB | 每个所有者的活跃资产上限；`0` 表示不限制 |
| `ASSET_S3_BUCKET` | 未设置 | 资产字节改存 S3（区域 / 端点 / 凭据走 AWS SDK 链） |
| `AWS_S3_FORCE_PATH_STYLE` | 未设置 | S3 兼容存储（MinIO、Ceph）需路径寻址时设 `1` |
| `ASSET_BYTE_EGRESS` | `direct` | 设为 `redirect` 时以短期签名 URL 302 响应字节 |
| `ASSET_COLLECTION_ENABLED` / `_INTERVAL_MS` / `_GRACE_MS` | 启用 / 900000 / 3600000 | 资产回收器 |
| `ASSET_PENDING_TTL_MS` | 86400000 | 已分配但未被文档引用的资产存活窗口 |
| `OPENMAIC_CLASSROOMS_DIR` | `<cwd>/data/classrooms` | 早期版本的文件式课堂目录，启动时一次性导入 |
| `MATERIALS_POOL_BACKFILL` | 未设置 | 设为 `1` 时执行一次旧上传迁移（会删除旧对象，需所有实例都已升级） |

### 6.2 所有者身份（谁能读写库）

**每个请求都会解析出一个所有者；每个请求都归属其所有者。**必须只选一种模式：

| 模式 | 设置 | 库的可见范围 |
| --- | --- | --- |
| 单用户（Compose 默认） | `OWNER_SINGLE_USER=true` | 能访问服务器的人共享一个库；需用 `ACCESS_CODE` 或回环地址保护 |
| 共享团队 | `PERSISTENCE_SHARED_OWNER_ID=<id>` + `ACCESS_CODE` | 访问码之后的一个团队库 |
| 匿名（两者都不设时的默认） | 不设 | 每个浏览器 Cookie 一个库，换浏览器即空库 |
| 自有账号 | 在 `instrumentation.ts` 注册 owner 认证方法 | 每个登录账号一个库 |

- `PERSISTENCE_SHARED_OWNER_ID` 与 `OWNER_SINGLE_USER` **互斥**，同时设置会启动失败；
- `OWNER_SINGLE_USER_ID` 默认 `local`，仅在单用户模式下可设；
- `OWNER_CLAIM_TRIGGER` 默认 `explicit`；设为 `auto` 会在每个浏览器首次请求时声明匿名工作，**不可逆地合并**所有匿名用户的库；
- `OWNER_ANONYMOUS_PREMINT` 默认开启，首个页面响应即签发匿名所有者 Cookie；
- 所有者 id 是**永久**的：更换 `OWNER_SINGLE_USER_ID`（或从共享团队模式切换）会使旧库失联。

### 6.3 Agent 运行时

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `OPENMAIC_AGENT_RUNTIME_ENABLED` | 关闭 | 关闭时 `/api/agent/sessions*` 与 `/api/agent/owner-events` 全部返回 404 |
| `OPENMAIC_ENABLE_PI_NATIVE_CHILD_RUNTIME` | 关闭 | 服务端 Native Child 执行器 |
| `OPENMAIC_ENABLE_PI_NATIVE_CHILD_SPOTLIGHT` | 关闭 | Native Spotlight 能力（不选择运行时） |
| `OPENMAIC_ENABLE_VOCATIONAL` | 关闭 | 职业任务引擎生成路径 |

运行时复用 `DATABASE_URL`；`agent` 槽必须是支持工具调用的模型（见 §3.2）。

### 6.4 渲染服务

| 变量 | 说明 |
| --- | --- |
| `RENDER_SERVICE_URL` | 设置后「导出视频」提供一键 MP4；未设置则降级为下载工程 ZIP。该地址是**运维提供的可信配置**，转发不经 SSRF Guard |
| `RENDER_PREVIEW_MAX_CONCURRENCY` | 仅预览并发，默认 1（标准 2 / 低内存 1），超限启动失败 |
| `RENDER_CHUNK_*` / `RENDER_MAX_PARALLEL_CHUNKS` / `RENDER_TARGET_CHUNK_FRAMES` | render-service 内部的有界分块执行（默认关闭） |
| `RENDER_RESOURCE_PROFILE` / `RENDER_SERVICE_MEMORY_LIMIT` | Compose 下资源档位与内存上限（标准需 8 GiB） |

---

## 7. 启动校验与失败行为

启动校验集中在 [instrumentation.ts](file:///D:/GITHUB_chat/OpenMAIC/instrumentation.ts#L243-L318) 的 `validateBootConfiguration()`，由 `register()` 在服务器就绪前执行。

### 7.1 致命校验（不通过则退出进程）

| 校验项 | 触发条件 |
| --- | --- |
| `requireDatabaseUrl` | 缺少 `DATABASE_URL` |
| `deploymentConfig` | `openmaic.yml` 结构 / 字段 / `${VAR}` / 槽位引用错误；或存在 `MODEL_ROUTES` 却没有 `openmaic.yml` |
| `resolveAssetQuotaBytes` | `ASSET_QUOTA_BYTES` 非法 |
| `resolveAssetPendingTtlMs` | `ASSET_PENDING_TTL_MS` 非正整数 |
| `validateOwnerIdentityConfiguration` | owner 身份变量非法、两种模式并存、或设置会被宿主注册忽略 |
| `validatePersistenceHooksConfiguration` | 宿主注册的钩子与配置不匹配（如 `ASSET_BYTE_EGRESS=redirect` 但字节存储无法签名） |

失败时 `exitOnBootFailure` 会打印原因并以非 0 退出（而非留一个「每个请求都 500」的僵尸进程）：**被拒的配置**以一行原始信息输出，其余以堆栈输出。

### 7.2 仅告警（不阻断启动）

- 未设置 `ACCESS_CODE`（单用户模式下改为单用户警告，二者只出一条）；
- `OPENMAIC_SECRET_KEY` 与已存储密钥不一致；
- 模型路由配置问题（`[config]` 前缀警告）；
- 以默认 PostgreSQL 密码对外发布。

### 7.3 后台任务（`register()` 不阻塞）

生成 run worker（**不依赖** `OPENMAIC_AGENT_RUNTIME_ENABLED`，始终运行）、材料抽取器、Agent 运行时、事件通知总线、资产回收器、旧课堂导入等均以定时器 / 后台任务形式启动；`SIGTERM` / `SIGINT` 时按顺序排水关闭。**因此生成流程要求进程常驻，无服务器平台不受支持。**

---

## 8. Docker Compose 配置

| 项 | 值 / 说明 |
| --- | --- |
| 服务 | `openmaic`（应用）+ `postgres`（PostgreSQL 16，默认启动，不对外发布端口） |
| 额外 profile | `video-export` 启动 `render-service`（独立内网 `render`，`internal: true`，带 iptables 出网封锁） |
| 端口发布 | `${OPENMAIC_PUBLISH_ADDRESS:-127.0.0.1}:${OPENMAIC_PORT:-3000}:3000` |
| 数据卷 | `openmaic-data`（`/app/data`，上传材料与实例密钥）、`openmaic-postgres` |
| env 加载顺序 | `docker-compose.defaults.env` → `.env.local`（后者覆盖前者） |
| Compose 默认值 | `DATABASE_URL` 指向内置 PostgreSQL；`OWNER_SINGLE_USER=true` |
| 应用环境注入 | `RENDER_SERVICE_URL=http://render-service:9000`（覆盖 `.env.local` 中的同名值） |
| 构建参数 | `ALPINE_MIRROR`、`NPM_REGISTRY`、`ALLOWED_FRAME_ANCESTORS` 及各 `NEXT_PUBLIC_*` |

| 变量（来自 shell 或同级 `.env`，**不是** `.env.local`） | 作用 |
| --- | --- |
| `OPENMAIC_PUBLISH_ADDRESS` | 发布地址，`0.0.0.0` 表示对外；对外前必须先设 `ACCESS_CODE` |
| `OPENMAIC_PORT` | 宿主机端口，默认 `3000` |
| `PERSISTENCE_POSTGRES_PASSWORD` | 数据库密码；未编码进 URL，**只能用字母与数字** |
| `OPENMAIC_DB_PORT` | 开发库端口（`pnpm db:up`），默认 `5432` |
| `RENDER_RESOURCE_PROFILE` / `RENDER_SERVICE_MEMORY_LIMIT` / `RENDER_PREVIEW_*` | 渲染服务资源与并发 |

> ⚠️ 升级 Compose 部署时：PostgreSQL 现在总是启动、应用仅发布在 `127.0.0.1`、所有访问者是同一个所有者。若 `.env.local` 设置了 `PERSISTENCE_SHARED_OWNER_ID`，必须同时设置 `OWNER_SINGLE_USER=false`，否则应用拒绝启动。

---

## 9. 配置优先级

```
同名的运行时变量（后者覆盖前者）：
  docker-compose.defaults.env  <  .env.local  <  docker-compose.yml 的 environment:
```

- `NEXT_PUBLIC_*` 与 `ALLOWED_FRAME_ANCESTORS`、`INTERACTIVE_ASSET_BASE_URL` 在**构建期**冻结，运行时环境变量无法覆盖；
- 模型解析优先级：槽位自身的赋值 > 沿父链继承 > （无 `openmaic.yml` 时的遗留）`DEFAULT_MODEL`；调用失败时按 `fallback` 重试一次；
- 用户设置中的模型选择覆盖服务器的槽位默认值，**除非**该槽位被 `lock` 或 `allowUserKeys: false` 约束。

---

## 10. 配置速查表

| 我想…… | 改哪里 |
| --- | --- |
| 换默认对话模型 | `openmaic.yml` → `slots.llm` |
| 只让大纲用另一个模型 | `openmaic.yml` → `slots.course.outline` |
| 关闭视频生成 | `openmaic.yml` → `slots.video: null`，或 `VIDEO_*_ENABLED=false` |
| 固定所有人都不能改的槽位 | `openmaic.yml` → `lock: [video]` |
| 禁止用户自带 Key | `openmaic.yml` → `allowUserKeys: false` |
| 启用 Pro 工作台 | `.env.local` → `NEXT_PUBLIC_PRO_WORKBENCH_ENABLED=true` + `OPENMAIC_AGENT_RUNTIME_ENABLED=true`（需重建） |
| 接入本地 Ollama / Lemonade | `.env.local` → `OLLAMA_BASE_URL` / `LEMONADE_BASE_URL`（+ `ALLOW_LOCAL_NETWORKS=true`） |
| 给站点加访问密码 | `.env.local` → `ACCESS_CODE=<≥16 位随机值>` |
| 团队共享一个课程库 | `.env.local` → `PERSISTENCE_SHARED_OWNER_ID=<id>` + `ACCESS_CODE` |
| 媒体存到 S3 / MinIO | `.env.local` → `ASSET_S3_BUCKET`（+ `AWS_S3_FORCE_PATH_STYLE=1`） |
| 开启一键 MP4 导出 | `RENDER_SERVICE_URL` 或 `docker compose --profile video-export up` |
| 换开发数据库端口 | `OPENMAIC_DB_PORT`（同时改 `DATABASE_URL`） |
