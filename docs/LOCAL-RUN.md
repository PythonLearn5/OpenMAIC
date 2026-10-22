# OpenMAIC 本地运行文档

> 职责说明：本文档说明如何在本地机器上从零启动 OpenMAIC（开发模式与生产构建），涵盖前置条件、依赖安装、数据库、最小可用配置、常用脚本、可选的本地能力接入与常见问题排查。所有命令与配置均以仓库当前代码为准。

> 相关文档：
> - 项目配置说明：[CONFIGURATION.md](file:///D:/GITHUB_chat/OpenMAIC/docs/CONFIGURATION.md)
> - 项目结构：[PROJECT-STRUCTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/PROJECT-STRUCTURE.md)
> - 总体架构：[ARCHITECTURE.md](file:///D:/GITHUB_chat/OpenMAIC/docs/ARCHITECTURE.md)
> - 测试与评估：[TESTING.md](file:///D:/GITHUB_chat/OpenMAIC/docs/TESTING.md)

---

## 1. 版本与前置条件

| 组件 | 版本要求 | 必需 | 依据 |
| --- | --- | --- | --- |
| Node.js | `>= 22.19.0` | 是 | [package.json](file:///D:/GITHUB_chat/OpenMAIC/package.json#L6-L8) `engines.node`；[.nvmrc](file:///D:/GITHUB_chat/OpenMAIC/.nvmrc) 锁定主版本 22 |
| pnpm | `10.28.0`（声明版本） | 是 | [package.json](file:///D:/GITHUB_chat/OpenMAIC/package.json#L220) `packageManager` |
| PostgreSQL | 16 | 是 | 服务器启动强制校验 `DATABASE_URL`，缺失即退出（见 §3、§5） |
| Docker | 任意近期版本 | 否（仅 `pnpm db:up` 与 Compose 部署需要） | [docker-compose.db.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.db.yml)、[docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml) |
| ffmpeg / ffprobe | 系统包，需在 `PATH` 中 | 否（仅本地音视频材料抽取需要） | [.env.example](file:///D:/GITHUB_chat/OpenMAIC/.env.example#L242-L247) |

安装依赖（仅需一次）：

```bash
pnpm install
```

`postinstall` 会自动执行 [build:packages](file:///D:/GITHUB_chat/OpenMAIC/package.json#L10-L11)，按顺序构建 `packages/` 下的 `mathml2omml → pptxgenjs → @openmaic/dsl → geometry → generation → storage → importer → renderer → editor`，随后运行 `sync-maic-importer` 与 `sync-interactive-assets` 两个同步脚本。**因此首次 `pnpm install` 较慢属正常现象；若跳过它，后续 `dev`/`build`/`test` 会因 workspace 包未编译而失败。**

---

## 2. 快速启动（5 步）

### 步骤 1：克隆并安装

```bash
git clone https://github.com/THU-MAIC/OpenMAIC.git
cd OpenMAIC
pnpm install
```

### 步骤 2：生成两份配置

```bash
cp .env.example .env.local            # 密钥与实例选项
cp openmaic.example.yml openmaic.yml  # 服务器使用哪些模型
```

两份文件的分工与字段含义见 [CONFIGURATION.md](file:///D:/GITHUB_chat/OpenMAIC/docs/CONFIGURATION.md)。

### 步骤 3：填入最小必需的密钥

编辑 `.env.local`，至少设置一个可用提供商的 Key（示例模板默认只需要 OpenAI）：

```env
OPENAI_API_KEY=sk-...
```

`openmaic.yml` 默认把 `llm` 槽（默认对话模型）指向 `openai:gpt-5.5`；若改用其他提供商，同时修改 `providers` 与 `slots` 即可（模板中已给出 Anthropic、MiniMax、Bedrock、本地 OpenAI 兼容服务等注释示例）。

### 步骤 4：启动本地数据库

```bash
pnpm db:up
```

然后在 `.env.local` 中取消注释 `DATABASE_URL` 一行：

```env
DATABASE_URL=postgres://openmaic:openmaic-dev@127.0.0.1:5432/openmaic
```

### 步骤 5：运行

```bash
pnpm dev
```

浏览器打开 **http://localhost:3000**（Next.js 默认端口）即可开始生成课堂。

> ⚠️ 没有 `DATABASE_URL` 时服务器**拒绝启动**并直接退出（退出码非 0），启动日志会打印修复方式。校验逻辑见 [instrumentation.ts](file:///D:/GITHUB_chat/OpenMAIC/instrumentation.ts#L249-L255) 的 `requireDatabaseUrl`。

---

## 3. 数据库（PostgreSQL）

课程文档、文件夹、聊天历史、学习者会话与生成媒体都存放在 PostgreSQL，由应用自身的 `/api/persistence` 提供服务；浏览器只保留设备级数据（设置、播放进度、缓存）。

### 3.1 使用内置开发数据库

```bash
pnpm db:up     # 启动
pnpm db:down   # 停止（数据卷保留）
```

对应脚本见 [package.json](file:///D:/GITHUB_chat/OpenMAIC/package.json#L20-L21)，实际执行的是 [docker-compose.db.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.db.yml)：

- 复用 [docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml#L68-L88) 中的 `postgres:16` 服务定义；
- 独立的 Compose 项目名 `openmaic-dev-db`，因此有独立容器与数据卷 `openmaic-dev-db_openmaic-postgres`，**不会影响** `docker compose up` 整套栈的数据库；
- 仅发布在回环地址 `127.0.0.1`，端口由 `OPENMAIC_DB_PORT` 控制（默认 `5432`）；
- 该项目名是固定的，所以同一台机器上的所有 checkout / worktree 共享这一个开发库：在其中一个目录执行 `pnpm db:down` 会对所有目录生效。

端口冲突时：

```bash
# 指定另一个端口后，DATABASE_URL 中的端口也要同步修改
OPENMAIC_DB_PORT=55432 pnpm db:up
```

### 3.2 使用已有 PostgreSQL

任意可访问的 PostgreSQL 16 均可，只要把连接串写入 `.env.local` 的 `DATABASE_URL`。

### 3.3 所有者身份（谁会看到哪些课程）

每个请求都会解析出一个「所有者」，每个所有者拥有独立的课程库。开发默认（不设置任何身份变量）是**匿名模式**：每个浏览器 Cookie 一个库，换浏览器就看到空库。三种模式对照见 [CONFIGURATION.md](file:///D:/GITHUB_chat/OpenMAIC/docs/CONFIGURATION.md#6-持久化与所有者身份)。

---

## 4. 环境变量与模型配置（最小可用）

| 文件 | 作用 | 最小必需内容 |
| --- | --- | --- |
| `.env.local` | 密钥、实例级选项（含 `DATABASE_URL`） | `OPENAI_API_KEY` + `DATABASE_URL` |
| `openmaic.yml` | 声明服务器可调用的 **providers** 与每个 **slot**（用途）用哪个模型 | 至少 `providers.openai` 与 `slots.llm` |

`.env.local` 中的密钥以 `${VAR}` 形式被 `openmaic.yml` 引用：

```yaml
providers:
  openai:
    preset: openai
    apiKey: ${OPENAI_API_KEY}
    models: [gpt-5.5]

slots:
  llm: openai:gpt-5.5
```

其他可选提供商、能力开关、持久化与安全选项均在 `.env.example` 中按区块注释说明，逐项含义见 [CONFIGURATION.md](file:///D:/GITHUB_chat/OpenMAIC/docs/CONFIGURATION.md)。

> ℹ️ `OPENMAIC_SECRET_KEY` 未设置时会在首次启动生成 `data/instance-secret.key`，用于加密「网页设置里保存的 provider 密钥」。本地开发可保持默认；多实例或 `data/` 不持久化时必须显式设置，且要与数据库一起保管。

---

## 5. 常用脚本命令

以下均来自 [package.json](file:///D:/GITHUB_chat/OpenMAIC/package.json#L9-L38)。

### 5.1 开发与构建

| 命令 | 实际执行 | 说明 |
| --- | --- | --- |
| `pnpm dev` | `sync-interactive-assets` → `build-standalone-player` → `next dev` | 开发服务器，默认 http://localhost:3000 |
| `pnpm build` | `sync-interactive-assets` → `assert-vendor-maic-importer` → `build-standalone-player` → `next build` | 生产构建，产物为 standalone（[next.config.ts](file:///D:/GITHUB_chat/OpenMAIC/next.config.ts#L7)） |
| `pnpm start` | `next start` | 启动已构建的生产服务器 |
| `pnpm build:packages` | 顺序构建 8 个 workspace 子包 | 单独重建内部 SDK |
| `pnpm build:standalone-player` | 构建离线播放器 | 导出离线 HTML 需要 |

### 5.2 数据库

| 命令 | 说明 |
| --- | --- |
| `pnpm db:up` | 启动开发 PostgreSQL（`openmaic-dev-db` 项目） |
| `pnpm db:down` | 停止开发 PostgreSQL（保留数据卷） |

### 5.3 测试与质量检查

| 命令 | 说明 |
| --- | --- |
| `pnpm test` | Vitest 单元 / 集成测试（`tests/**/*.test.ts`） |
| `pnpm test:e2e` | Playwright 端到端测试 |
| `pnpm test:e2e:ui` | Playwright UI 模式 |
| `pnpm lint` | ESLint |
| `pnpm check` | Prettier 格式检查 |
| `pnpm format` | Prettier 写入格式化 |
| `pnpm check:i18n-keys` | 校验 i18n 文案键一致性 |
| `pnpm check:node-engine` | 校验 Node 版本契约 |
| `pnpm check:package-versions` | 校验子包版本递增 |
| `pnpm eval:*` | LLM 质量评估集（需真实 Key） |

测试细节（E2E 端口、`.env.local` 加载规则等）见 [TESTING.md](file:///D:/GITHUB_chat/OpenMAIC/docs/TESTING.md)。

---

## 6. 生产构建与运行

```bash
pnpm build
DATABASE_URL=postgres://... pnpm start
```

关键约束：

- **必须常驻进程**。课程生成在**服务器进程内**执行，浏览器关掉页面后仍会继续，因此需要进程管理器或容器承载；**无服务器（serverless）平台自 1.2.0 起不再支持**。
- `next.config.ts` 使用 `output: 'standalone'`，生产构建会走 [tsconfig.build.json](file:///D:/GITHUB_chat/OpenMAIC/tsconfig.build.json)（见 [next.config.ts](file:///D:/GITHUB_chat/OpenMAIC/next.config.ts#L32-L34)）。
- `NEXT_PUBLIC_*` 变量在构建时被编译进浏览器产物，**改动后必须重新构建**。

---

## 7. Docker Compose 一键运行（可选）

```bash
cp .env.example .env.local
# 编辑 .env.local 填入 API Key
docker compose up --build
```

- 两个容器：应用 + PostgreSQL；
- 数据卷 `openmaic-postgres`（数据库）与 `openmaic-data`（上传材料、运行时数据）在 `docker compose down` 与重建后依然保留；`docker compose down -v` 会删除它们；
- 默认以 **单用户模式**运行（`OWNER_SINGLE_USER=true`，见 [docker-compose.defaults.env](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.defaults.env)），且只发布在 `127.0.0.1:3000`；
- 需要在局域网访问时，先设置 `ACCESS_CODE`，再：

```bash
OPENMAIC_PUBLISH_ADDRESS=0.0.0.0 docker compose up -d --build
```

- 可选用 `openmaic.yml` 配置模型：从模板创建后，取消 [docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml#L56-L58) 中的挂载注释；
- 可选 MP4 视频导出（独立的 Chromium + FFmpeg 容器）：

```bash
docker compose --profile video-export up --build
```

`OPENMAIC_PUBLISH_ADDRESS`、`OPENMAIC_PORT`、`PERSISTENCE_POSTGRES_PASSWORD` 等来自 shell 或与 `docker-compose.yml` 同级的 `.env`，**不是** `.env.local`。

### 慢网络 / 国内构建加速

```bash
ALPINE_MIRROR=mirrors.tuna.tsinghua.edu.cn \
NPM_REGISTRY=https://registry.npmmirror.com \
docker compose up --build
```

这两个构建参数默认空值；仅加速 Alpine 与 npm 源，**不加速 Docker Hub 拉取**（如 `node:22-alpine` 基础镜像）。

---

## 8. 可选能力接入（本地）

以下能力默认关闭，按需启用。**改动 `.env.local` 后需重启服务器；改动 `NEXT_PUBLIC_*` 需重新构建。**

### 8.1 Pro 工作台与 Agent 运行时

```env
NEXT_PUBLIC_PRO_WORKBENCH_ENABLED=true
OPENMAIC_AGENT_RUNTIME_ENABLED=true
DATABASE_URL=postgres://openmaic:openmaic-dev@127.0.0.1:5432/openmaic
```

`agent` 槽需要**支持工具调用**的模型：

```yaml
slots:
  llm: openai:gpt-5.5
  agent:
    model: openai:gpt-5.5
    api: openai-completions        # 或 openai-responses
```

`agent` 槽不得设置 `thinking.effort`（工具调用与 reasoning effort 在该传输上无法共存）。

### 8.2 本地模型（Ollama / Lemonade）

本地服务需要放行私网地址（SSRF 防护默认拦截 private/loopback），设置：

```env
ALLOW_LOCAL_NETWORKS=true
```

> 💡 `OLLAMA_BASE_URL` 在**服务端**配置时可自动绕过 SSRF 防护；但客户端传入的 localhost 地址在生产环境一律被拦截。

### 8.3 FunASR（本地语音识别）

```bash
python -m pip install torch torchaudio
python -m pip install "funasr==1.4.0" fastapi uvicorn python-multipart
python -m pip install vllm            # NVIDIA GPU 上跑 Fun-ASR-Nano 时加装
funasr-server --device cuda --model fun-asr-nano
```

```yaml
providers:
  funasr:
    preset: funasr-asr
    baseUrl: http://localhost:8000/v1
slots:
  asr: funasr
```

CPU 环境用 `funasr-server --device cpu --model sensevoice`。

### 8.4 VoxCPM2（自托管 TTS，支持音色克隆）

```yaml
providers:
  voxcpm:
    preset: voxcpm-tts
    baseUrl: http://localhost:8000/v1
    options:
      backend: vllm-omni          # vllm-omni（默认）| python-api | nano-vllm
slots:
  tts: voxcpm
```

只有 `vllm-omni` 后端支持音色注册；另两种后端每次请求携带音色提示。

### 8.5 本地音视频材料抽取

安装系统 `ffmpeg`（同时提供 `ffprobe`）并把它放进 `PATH`，再按 §8.3 等配置好 `asr` 槽即可。应用在抽取时解析可执行文件；`ffmpeg` **不是 npm 依赖**，缺失时本地抽取器被跳过，转而使用已配置的云端抽取（如 AliDocMind）。

### 8.6 MP4 视频导出（render-service）

```bash
docker compose --profile video-export up --build
```

应用通过 `RENDER_SERVICE_URL` 自动探测（[docker-compose.yml](file:///D:/GITHUB_chat/OpenMAIC/docker-compose.yml#L45-L51) 已预置 `http://render-service:9000`）。未启用该 profile 或未设置该变量时，导出降级为下载工程 ZIP 供本地 CLI 渲染。

### 8.7 MinerU（高级文档解析）

```yaml
providers:
  mineru:
    preset: mineru-cloud          # 或 mineru + 自托管 baseUrl
    apiKey: ${PDF_MINERU_CLOUD_API_KEY}
slots:
  document: mineru
```

### 8.8 站点访问密码（共享部署）

```env
ACCESS_CODE=your-secret-code
```

未设置时不做任何网关校验（fail-open），所有路由（含 API）均可直达。**对外暴露前务必设置**，并使用 ≥ 16 位随机值。

---

## 9. 常见问题排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 启动即退出，日志提示需要数据库 | 未设置 `DATABASE_URL` | 执行 `pnpm db:up` 并在 `.env.local` 取消注释 `DATABASE_URL` |
| 启动打印一行 `[boot]` 原因后退出 | 配置被拒绝（`openmaic.yml` 字段错误 / `${VAR}` 未设置 / owner 身份冲突 / 资产配额非法等） | 按提示修正；校验清单见 [instrumentation.ts](file:///D:/GITHUB_chat/OpenMAIC/instrumentation.ts#L243-L318) 与 [CONFIGURATION.md](file:///D:/GITHUB_chat/OpenMAIC/docs/CONFIGURATION.md#7-启动校验与失败行为) |
| `pnpm test` 报 workspace 包导入错误 | 内部子包未构建 | 先执行 `pnpm build:packages` |
| 测到外部 API 的用例全红，但浏览器正常 | Vitest 默认**不加载** `.env.local`（保持 hermetic） | 单次复现时加 `TEST_LOAD_LOCAL_ENV=1`，见 [setup-env.ts](file:///D:/GITHUB_chat/OpenMAIC/tests/setup-env.ts#L24-L42) |
| E2E 起不来 / 端口被占 | E2E 固定使用 3002 端口并强制注入编辑器开关 | 见 [playwright.config.ts](file:///D:/GITHUB_chat/OpenMAIC/playwright.config.ts#L12-L45)；本地需先 `pnpm db:up` 并配置 `DATABASE_URL` |
| Safari 下每次请求都变成新所有者 / 写入 403 | 生产构建的 Cookie 带 `Secure`，Safari 不为纯 `http://localhost` 存储 | 纯 HTTP 部署设置 `COOKIE_SECURE=0`（仅限可信网络/本机） |
| 本地模型（Ollama 等）调用被拦截 | SSRF 防护拦截私网地址 | 设置 `ALLOW_LOCAL_NETWORKS=true`（**不要**在公网部署上开启） |
| 改了 `NEXT_PUBLIC_*` 没生效 | 这类变量在构建期编译进产物 | 重新执行 `pnpm build`（或重启 `pnpm dev`） |
| 依赖安装很慢 | `postinstall` 会构建 8 个 workspace 子包 | 属预期；使用镜像源可加速拉取 |
